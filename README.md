# arellan-infrastructure

Repositorio de infraestructura como código (IaC), CI/CD, observabilidad, seguridad operativa y scripts de despliegue para el ecosistema digital de la Clínica Automotriz Arellan Hnos.

## Descripción

`arellan-infrastructure` contiene todo lo necesario para desplegar, operar y monitorear el ecosistema en la nube. No contiene código de aplicación — solo configuración, scripts y definiciones de infraestructura.

## Estructura

```
arellan-infrastructure/
├── arellan-terraform-infra/        # Módulos Terraform por recurso
├── arellan-infra-devops/           # CI/CD, GitHub Actions workflows
├── arellan-config-center/          # Configuración de entornos y secretos
├── arellan-devsecops-platform/     # Security scanning, SAST, DAST
├── arellan-disaster-recovery/      # Backups, RPO/RTO, runbooks de recuperación
├── arellan-logging-centralizado/   # Centralización de logs (CloudWatch/Axiom)
└── arellan-observability-platform/ # Grafana, métricas, APM, alertas
```

## Ambientes

| Ambiente | Descripción | Deploy |
|----------|-------------|--------|
| `dev` | Local Docker Compose | Manual (`docker compose up`) |
| `staging` | Railway + Supabase | Auto en merge a `develop` |
| `production` | AWS (Fase 2+) | Manual con aprobación requerida |

## Stack de Infraestructura MVP (Meses 1–6)

| Servicio | Proveedor | Costo/mes |
|---------|-----------|-----------|
| Base de datos + Auth + Storage | Supabase Pro | $25 |
| Backend API | Railway Starter | $5–20 |
| Frontends | Vercel Free/Pro | $0–20 |
| CDN + DNS + SSL | Cloudflare Free | $0 |
| Redis | Upstash | $0–10 |
| Email | Resend Free | $0 |
| Errores | Sentry Free | $0 |
| Status | Upptime (GitHub) | $0 |

## Stack de Infraestructura Producción (Meses 7+)

| Servicio | Proveedor | Costo/mes |
|---------|-----------|-----------|
| Backend API | AWS EC2 t3.small / ECS Fargate | $15–30 |
| Base de datos | AWS RDS PostgreSQL t3.micro | $15–25 |
| Redis | AWS ElastiCache t3.micro | $20–30 |
| Storage | AWS S3 + CloudFront | $5–30 |
| DNS | AWS Route 53 | $2–5 |
| Secretos | AWS Secrets Manager | $2–5 |
| Frontends | Vercel Pro | $20 |

## CI/CD — GitHub Actions

Cada repo del ecosistema tiene su pipeline `.github/workflows/`:

```yaml
# Pipeline estándar por PR
on: pull_request
jobs:
  quality:
    - lint (ESLint)
    - type-check (TypeScript)
    - test (Jest/Vitest con cobertura mínima 80%)
    - build (verificación de build exitoso)

# Pipeline en merge a main
on: push (main)
jobs:
  deploy-staging:
    - build Docker image
    - push a registry
    - deploy a staging
    - smoke tests

# Deploy a producción (tag de release)
on: push (tag v*)
jobs:
  deploy-production:
    - requiere aprobación manual de owner
    - deploy a producción
    - health check post-deploy
    - rollback automático si health check falla
```

## Acceso

Solo 2 personas tienen acceso completo a este repo: los owners de la organización `arellan-tech`. Los secretos de producción nunca se almacenan en este repo — siempre en AWS Secrets Manager o GitHub Secrets (cifrados).

## Repos Relacionados

- `arellan-platform` — el backend que este repo despliega
- `arellan-security-compliance` — políticas de seguridad aplicadas aquí
- `arellan-docs-governance` — runbooks operativos y ADRs

## Licencia

Privado — © 2026 Arellan Hnos. Todos los derechos reservados.
