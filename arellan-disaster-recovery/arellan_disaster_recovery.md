# arellan-disaster-recovery — Plan de Recuperación ante Desastres

Procedimientos de backup, recuperación y continuidad operativa para el ecosistema digital de la Clínica Automotriz Arellan Hnos.

## Objetivos de Recuperación

| Métrica | MVP (Supabase/Railway) | Producción (AWS) |
|---------|----------------------|-----------------|
| **RPO** (Punto de recuperación) | 24 horas (backup diario Supabase) | 1 hora (backup cada hora RDS) |
| **RTO** (Tiempo de recuperación) | 4–8 horas | 30–60 minutos |

## Estrategia de Backups

### Base de Datos

**MVP con Supabase:**
- Backup automático diario a las 2 AM (hora Lima UTC-5)
- Retención: 7 días en plan Pro
- Point-in-time recovery disponible en el dashboard de Supabase

**Producción con AWS RDS:**
```bash
# Backup automático configurado en Terraform
backup_retention_period = 7      # días
backup_window           = "07:00-09:00"  # UTC (2-4 AM Lima)
delete_automated_backups = false

# Backup manual mensual (primer día del mes)
aws rds create-db-snapshot \
  --db-instance-identifier arellan-production \
  --db-snapshot-identifier "arellan-monthly-$(date +%Y%m)"
```

### Archivos (Fotos de Vehículos, Documentos)

**MVP con Supabase Storage:**
- Replicación automática incluida en plan Pro

**Producción con AWS S3:**
```hcl
resource "aws_s3_bucket_replication_configuration" "arellan_files" {
  role   = aws_iam_role.replication.arn
  bucket = aws_s3_bucket.arellan_files.id

  rule {
    status = "Enabled"
    destination {
      bucket        = aws_s3_bucket.arellan_files_replica.arn
      storage_class = "STANDARD_IA"
    }
  }
}
```

## Runbook de Recuperación — Pérdida Total de BD

**Tiempo estimado: 4–8 horas (MVP) / 1–2 horas (producción)**

```
PASO 1: Identificar el último backup válido
  → Supabase: Dashboard > Database > Backups
  → AWS: RDS Snapshots o S3 backups

PASO 2: Crear nueva instancia de BD desde el snapshot
  → Supabase: "Restore" en el backup seleccionado
  → AWS RDS: aws rds restore-db-instance-from-db-snapshot

PASO 3: Verificar integridad de los datos
  → Correr queries de verificación en tablas críticas:
    SELECT COUNT(*) FROM audit_logs;
    SELECT COUNT(*) FROM work_orders;
    SELECT COUNT(*) FROM financial_transactions;

PASO 4: Actualizar DATABASE_URL en AWS Secrets Manager
  → Si es una nueva instancia con diferente endpoint

PASO 5: Reiniciar el servicio backend
  → Railway: railway redeploy
  → AWS ECS: aws ecs update-service --force-new-deployment

PASO 6: Verificar health check de la API
  → curl https://api.arellan.pe/health

PASO 7: Notificar al equipo
  → Estado del sistema en status.arellan.pe
  → WhatsApp a owners con resumen del incidente
```

## Runbook de Recuperación — Backend Caído

```bash
# 1. Verificar estado del servicio
curl -f https://api.arellan.pe/health

# 2. Revisar logs recientes
# Railway: railway logs --tail 100
# AWS: aws logs tail /ecs/arellan-api --since 1h

# 3. Si el último deploy causó el problema → rollback
./scripts/rollback.sh production

# 4. Si es problema de recursos → escalar
# AWS: aumentar CPU/memoria del task definition
aws ecs update-service \
  --cluster arellan-production \
  --service arellan-api \
  --task-definition arellan-api:latest \
  --desired-count 2

# 5. Actualizar status.arellan.pe con el incidente
```

## Runbook de Incidente de Seguridad

Ver `arellan-security-compliance/arellan-incident-response` para el playbook completo.

Acciones inmediatas:
1. **Revocar todas las sesiones activas:** `DELETE FROM refresh_tokens WHERE revoked_at IS NULL`
2. **Rotar TODOS los secretos** inmediatamente
3. **Preservar logs forenses** antes de cualquier acción que los modifique
4. **Notificar a owners** con detalle del incidente

## Pruebas de Recuperación

Las pruebas de recuperación se ejecutan:
- **Mensualmente (primer sábado del mes):** restaurar backup en ambiente de staging y verificar integridad
- **Trimestralmente:** simulacro completo de recuperación en entorno aislado

Los resultados de las pruebas se documentan en `arellan-docs-governance/runbooks/dr-test-results/`.
