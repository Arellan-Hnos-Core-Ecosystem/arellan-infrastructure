# arellan-config-center — Centro de Configuración y Secretos

Documentación de la gestión de variables de entorno, secretos y configuración por ambiente para el ecosistema Arellan Hnos. No contiene valores reales — solo la estructura y las reglas de gestión.

## Principio Fundamental

**Los secretos nunca están en el código ni en git.** Esta es una regla no negociable del ecosistema. Cualquier credencial, API key o contraseña en el código es una vulnerabilidad crítica.

## Gestión por Ambiente

| Ambiente | Gestión de secretos | Gestión de config |
|----------|--------------------|--------------------|
| `dev` (local) | Archivo `.env` local (en `.gitignore`) | `.env.example` en repo |
| `staging` | Railway Environment Variables | Railway dashboard |
| `production` | AWS Secrets Manager | AWS Parameter Store |

## Variables por Servicio

### `arellan-platform` (Backend)
```env
# ===== Base de Datos =====
DATABASE_URL=                     # PostgreSQL connection string

# ===== Autenticación =====
JWT_PRIVATE_KEY=                  # RS256 private key (PEM format)
JWT_PUBLIC_KEY=                   # RS256 public key (PEM format)
JWT_EXPIRES_IN=1h
JWT_REFRESH_EXPIRES_IN=7d

# ===== Supabase =====
SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=        # Clave de servicio (solo backend)

# ===== Redis =====
REDIS_URL=

# ===== Storage =====
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=us-east-1
AWS_S3_BUCKET=arellan-files-production

# ===== Notificaciones =====
RESEND_API_KEY=
VAPID_PUBLIC_KEY=
VAPID_PRIVATE_KEY=

# ===== Integraciones =====
ZKTECO_DEVICE_SECRET=
SUNAT_CLIENT_ID=                  # Fase 2
SUNAT_CLIENT_SECRET=              # Fase 2
CULQI_PRIVATE_KEY=                # Fase 2
WHATSAPP_ACCESS_TOKEN=            # Fase 2

# ===== App =====
NODE_ENV=production
PORT=3000
API_URL=https://api.arellan.pe
ALLOWED_ORIGINS=https://app.arellan.pe,https://mobile.arellan.pe,https://taller.arellan.pe,https://cliente.arellan.pe
```

### `arellan-frontend-web` (Admin)
```env
NEXT_PUBLIC_API_URL=https://api.arellan.pe
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=    # Clave pública (seguro en frontend)
NEXT_PUBLIC_VAPID_PUBLIC_KEY=
```

### Frontends Restantes (mobile-app, mechanic-ui, client-portal)
Misma estructura que `arellan-frontend-web` con sus respectivas URLs de dominio.

## Rotación de Secretos

| Secreto | Frecuencia de rotación | Responsable |
|---------|----------------------|-------------|
| JWT RS256 keys | Cada 90 días | Automatizado (script) |
| Supabase Service Role Key | Al comprometer o cada 90 días | Manual por owner |
| AWS IAM credentials | Cada 90 días | Automatizado (IAM rotation) |
| CULQI / SUNAT keys | Al comprometer | Manual por owner |
| ZKTeco device secret | Al cambiar hardware | Manual por owner |

**Al salir un colaborador del proyecto:** rotar TODOS los secretos en un plazo máximo de 24 horas.

## Procedimiento de Rotación JWT

```bash
# 1. Generar nuevo par de claves RS256
openssl genrsa -out jwt_private_new.pem 4096
openssl rsa -in jwt_private_new.pem -pubout -out jwt_public_new.pem

# 2. Actualizar en AWS Secrets Manager
aws secretsmanager update-secret \
  --secret-id arellan/production/jwt-private-key \
  --secret-string file://jwt_private_new.pem

# 3. Reiniciar el servicio backend (deploy rolling para zero-downtime)
# Los tokens anteriores siguen válidos durante su vida útil (máx 1h)

# 4. Eliminar archivos de clave locales
rm -f jwt_private_new.pem jwt_public_new.pem
```

## `.gitignore` Global Obligatorio

Estos archivos NUNCA deben aparecer en ningún commit de ningún repo:

```gitignore
.env
.env.local
.env.*.local
*.pem
*.key
*_rsa
*credentials*
terraform.tfvars
secrets/
.aws/
```
