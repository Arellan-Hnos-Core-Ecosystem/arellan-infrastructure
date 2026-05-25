# arellan-infra-devops — Operaciones y DevOps

Configuraciones de red, políticas de despliegue seguro, scripts de migración de base de datos, linters de código y herramientas de operación diaria del ecosistema Arellan Hnos.

## Contenido

Este módulo contiene todo lo necesario para operar el ecosistema día a día:

```
arellan-infra-devops/
├── docker/
│   ├── docker-compose.yml          # Entorno de desarrollo local completo
│   ├── docker-compose.staging.yml  # Staging en Railway
│   └── Dockerfile.production       # Imagen optimizada para producción
├── scripts/
│   ├── migrate.sh                  # Ejecutar migraciones Prisma
│   ├── seed.sh                     # Seed de datos iniciales
│   ├── deploy.sh                   # Deploy por ambiente
│   ├── rollback.sh                 # Rollback al release anterior
│   ├── backup.sh                   # Backup manual de BD
│   ├── rotate-secrets.sh           # Rotación de JWT keys y secretos
│   └── health-check.sh             # Verificación post-deploy
├── linting/
│   ├── .eslintrc.base.js           # Config ESLint base (de @arellan/eslint-config)
│   ├── .prettierrc                 # Formato de código
│   └── .editorconfig               # Consistencia de editors
└── ci/
    └── github-actions/             # Templates de workflows reutilizables
```

## Docker Compose — Entorno Local

```yaml
# docker-compose.yml — Levantar con: docker compose up -d
version: '3.9'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: arellan_dev
      POSTGRES_USER: arellan
      POSTGRES_PASSWORD: local_dev_only
    ports: ['5432:5432']
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports: ['6379:6379']
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru

  mailhog:
    image: mailhog/mailhog
    ports:
      - '1025:1025'   # SMTP
      - '8025:8025'   # Web UI para ver emails en dev

volumes:
  postgres_data:
```

## Migraciones de Base de Datos

```bash
# Crear una nueva migración
npx prisma migrate dev --name nombre_descriptivo_del_cambio

# Aplicar migraciones pendientes en staging/producción
npx prisma migrate deploy

# Verificar estado de migraciones
npx prisma migrate status

# Reset completo (SOLO en desarrollo local)
npx prisma migrate reset
```

**Reglas de migración:**
- Las migraciones deben ser retrocompatibles (nunca eliminar columnas en el mismo deploy que las deja de usar)
- Cada migración pasa por code review antes de aplicarse a producción
- Las migraciones de producción se aplican en horario de baja actividad

## Linting y Calidad de Código

```bash
# Correr ESLint en todos los repos
npm run lint

# Auto-fix de problemas corregibles
npm run lint:fix

# Verificar tipos TypeScript (sin emit)
npm run type-check

# Formato con Prettier
npm run format

# Todo junto (pre-commit hook)
npm run check
```

## Pre-commit Hooks (Husky + lint-staged)

```json
{
  "lint-staged": {
    "*.{ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{json,md,yml}": [
      "prettier --write"
    ]
  }
}
```

## Política de Network y Firewall

**Producción (AWS):**
- El backend (ECS/EC2) está en subnet privada — no accesible directamente desde internet
- Solo el Load Balancer (en subnet pública) recibe tráfico externo en puertos 80/443
- La base de datos (RDS) solo acepta conexiones desde el security group del backend
- Redis (ElastiCache) solo acepta conexiones desde el backend
- Todos los recursos tienen salida a internet solo a través de NAT Gateway (para llamadas a APIs externas)

**Reglas de Security Groups:**
```hcl
# Backend EC2/ECS
ingress: 443 (HTTPS) desde ALB security group
egress:  5432 (PostgreSQL) a RDS security group
egress:  6379 (Redis) a ElastiCache security group
egress:  443 (HTTPS) a internet (para APIs externas: Supabase, Resend, etc.)

# RDS PostgreSQL
ingress: 5432 solo desde backend security group

# ElastiCache Redis
ingress: 6379 solo desde backend security group
```
