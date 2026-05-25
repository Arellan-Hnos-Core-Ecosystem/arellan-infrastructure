# arellan-logging-centralizado — Logs Centralizados del Ecosistema

Centralización, estructura y retención de todos los logs del ecosistema Arellan Hnos. Separa los logs de auditoría del negocio (inmutables en PostgreSQL) de los logs de aplicación (operativos en CloudWatch/Axiom).

## Tipos de Logs

| Tipo | Fuente | Destino | Retención | Mutable |
|------|--------|---------|-----------|---------|
| Audit logs (negocio) | NestJS AuditInterceptor | PostgreSQL `audit_logs` | Indefinida | NO |
| Application logs | NestJS Winston | Axiom / CloudWatch | 30 días | Sí |
| Access logs | API Gateway | Axiom / CloudWatch | 90 días | Sí |
| Error logs | Sentry | Sentry Cloud | 90 días | Sí |
| Security alerts | NestJS Security Module | Sentry + Axiom | 1 año | Sí |

## Formato de Log de Aplicación (Estructurado JSON)

```json
{
  "timestamp": "2026-05-24T14:30:00.000Z",
  "level": "info",
  "service": "arellan-platform",
  "environment": "production",
  "requestId": "uuid-request-id",
  "userId": "uuid-user-id",
  "role": "FINANCE",
  "method": "POST",
  "path": "/api/v1/finance/expenses",
  "statusCode": 201,
  "durationMs": 145,
  "ip": "190.123.x.x",
  "message": "Expense created successfully",
  "context": {
    "expenseId": "uuid",
    "amount": 250,
    "category": "LOCAL_PARTS"
  }
}
```

## Configuración Winston (NestJS)

```typescript
import { createLogger, transports, format } from 'winston'

const logger = createLogger({
  format: format.combine(
    format.timestamp(),
    format.errors({ stack: true }),
    format.json()
  ),
  transports: [
    // Logs locales/staging
    new transports.Console({
      level: process.env.LOG_LEVEL || 'info',
    }),
    // Producción: enviar a CloudWatch
    ...(process.env.NODE_ENV === 'production' ? [
      new CloudWatchTransport({
        logGroupName: '/arellan/platform/production',
        logStreamName: `${hostname()}-${Date.now()}`,
        awsRegion: 'us-east-1',
      })
    ] : []),
  ],
})
```

## Audit Logs — Regla de Inmutabilidad

La tabla `audit_logs` en PostgreSQL tiene las siguientes garantías técnicas:

```sql
-- 1. Regla de base de datos: prohibir DELETE y UPDATE
CREATE OR REPLACE RULE no_delete_audit AS
  ON DELETE TO audit_logs DO INSTEAD NOTHING;

CREATE OR REPLACE RULE no_update_audit AS
  ON UPDATE TO audit_logs DO INSTEAD NOTHING;

-- 2. Row-level security: ningún role puede DELETE/UPDATE
ALTER TABLE audit_logs ENABLE ROW LEVEL SECURITY;
CREATE POLICY audit_insert_only ON audit_logs
  FOR INSERT TO arellan_app_role WITH CHECK (true);
CREATE POLICY audit_select_owners ON audit_logs
  FOR SELECT USING (true);  -- Owners ven todo en la aplicación

-- 3. El servicio de BD nunca recibe permisos de DELETE/UPDATE sobre esta tabla
REVOKE DELETE, UPDATE, TRUNCATE ON audit_logs FROM arellan_app_role;
```

## Log de Descuadres de Caja (Forensics)

Cuando se detecta un descuadre al cierre de caja, además del flag `CLOSED_WITH_DISCREPANCY`, el sistema genera automáticamente una entrada de log forense especial con contexto expandido:

```json
{
  "type": "FORENSIC_CASHBOX_DISCREPANCY",
  "sessionId": "uuid",
  "reportedBy": "uuid-finance-user",
  "expectedTotal": 1250.00,
  "actualTotal": 1180.00,
  "difference": -70.00,
  "lastTransactions": [/* últimas 10 transacciones del día */],
  "cameraFootageTimestamp": "2026-05-24T18:00:00.000Z",
  "automaticActionTaken": "OWNERS_NOTIFIED_WITH_CAMERA_REFERENCE"
}
```

## Alertas Operativas de Logs

Axiom y CloudWatch tienen alertas configuradas para:
- Error rate >5% en cualquier 5 minutos → alerta a Sentry
- Más de 10 logs de `SECURITY_VIOLATION` en 1 hora → push a owners
- Ausencia de logs de heartbeat por más de 5 minutos → backend posiblemente caído
- Log `CRITICAL_OPERATIONAL_ANOMALY` (ZKTeco checkout con OT activa) → push inmediato a owners
