# arellan-devsecops-platform — Seguridad en el Pipeline de Desarrollo

Herramientas y prácticas de DevSecOps para el ecosistema Arellan Hnos. Seguridad integrada en el pipeline de CI/CD: análisis de código estático, escaneo de dependencias y protección de secretos.

## Herramientas Activas

| Herramienta | Propósito | Cuándo ejecuta |
|-----------|---------|---------------|
| **GitHub Advanced Security** | SAST — CodeQL | En cada PR |
| **Dependabot** | Vulnerabilidades en dependencias | Diariamente |
| **GitHub Secret Scanning** | Detecta credenciales expuestas | En cada push |
| **Semgrep** (Fase 2) | Análisis semántico de código | En cada PR |
| **OWASP ZAP** (Fase 2) | DAST — testing de seguridad dinámico | Semanal en staging |

## CodeQL — Análisis Estático

```yaml
# .github/workflows/security-scan.yml
name: Security Scan
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]
  schedule:
    - cron: '0 0 * * 1'  # Semanal los lunes

jobs:
  codeql:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
      contents: read

    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with:
          languages: javascript,typescript
          queries: security-and-quality

      - uses: github/codeql-action/analyze@v3
        with:
          category: "/language:typescript"
```

## Dependabot — Gestión de Vulnerabilidades

```yaml
# .github/dependabot.yml (en cada repo)
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    open-pull-requests-limit: 5
    reviewers:
      - "arellan-tech/owners"
    labels:
      - "security"
      - "dependencies"
    ignore:
      # Ignorar actualizaciones major (revisar manualmente)
      - dependency-name: "*"
        update-types: ["version-update:semver-major"]
```

## Reglas para Desarrollo Seguro

### SQL Injection Prevention
```typescript
// INCORRECTO — nunca concatenar SQL
const result = await db.query(`SELECT * FROM users WHERE email = '${email}'`)

// CORRECTO — Prisma previene SQL injection por diseño
const user = await prisma.user.findFirst({ where: { email } })
```

### XSS Prevention
```typescript
// Todos los inputs del usuario deben sanitizarse con class-validator
@IsString()
@MaxLength(255)
@Transform(({ value }) => sanitizeHtml(value, { allowedTags: [] }))
description: string
```

### IDOR Prevention
```typescript
// Verificar que el recurso pertenece al usuario autenticado
@Get('orders/:id')
async getOrder(
  @Param('id') id: string,
  @CurrentUser() user: AuthUser,
) {
  const order = await this.ordersService.findById(id)

  // Mecánico solo puede ver sus propias OTs
  if (user.role === UserRole.MECHANIC && order.mechanicId !== user.id) {
    throw new ForbiddenException('No tienes acceso a esta orden')
  }

  return order
}
```

## OWASP Top 10 — Cobertura

| Vulnerabilidad | Mitigación implementada |
|---------------|------------------------|
| A01 — Broken Access Control | RBAC + guards en cada endpoint, verificación de ownership |
| A02 — Cryptographic Failures | AES-256 campos sensibles, JWT RS256, HTTPS obligatorio |
| A03 — Injection | Prisma ORM (sin SQL raw), class-validator en todos los DTOs |
| A04 — Insecure Design | Rate limiting, audit logs inmutables, segregación de funciones |
| A05 — Security Misconfiguration | Helmet.js, CORS restrictivo, env vars en Secrets Manager |
| A06 — Vulnerable Components | Dependabot + CodeQL automatizados |
| A07 — Auth Failures | JWT RS256, MFA obligatorio para roles críticos, refresh tokens revocables |
| A09 — Security Logging | Audit interceptor en 100% de endpoints que mutan datos |
| A10 — SSRF | Validación de URLs en webhooks de ZKTeco y pasarelas de pago |

## Proceso de Reporte de Vulnerabilidades

Si se descubre una vulnerabilidad de seguridad:
1. NO crear un issue público en GitHub
2. Enviar email a `seguridad@arellan.pe` con detalle del hallazgo
3. Incluir: descripción, pasos para reproducir, impacto potencial
4. El equipo responde en máximo 48 horas
5. Seguir el proceso de incident response en `arellan-security-compliance/arellan-incident-response`
