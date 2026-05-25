# arellan-observability-platform — Monitoreo, Métricas y Alertas

Plataforma de observabilidad del ecosistema Arellan Hnos. Centraliza métricas de aplicación, logs, alertas de negocio y APM (Application Performance Monitoring).

## Stack de Observabilidad

| Herramienta | Propósito | MVP | Producción |
|-----------|---------|-----|-----------|
| **Sentry** | Captura de errores y excepciones | Free (5k/mes) | Team ($26/mes) |
| **Axiom** | Logs estructurados centralizados | Free tier | Starter |
| **Upptime** | Uptime y disponibilidad | Free (GitHub) | Free |
| **Railway Metrics** | CPU, RAM, requests del backend | Incluido | — |
| **Grafana** | Dashboards de negocio | Fase 2 | Grafana Cloud |
| **CloudWatch** | Métricas y alarmas AWS | — | Incluido AWS |
| **OpenTelemetry** | Instrumentación distribuida | Fase 2 | Fase 2 |

## Sentry — Captura de Errores

```typescript
// Configuración en arellan-platform/src/main.ts
import * as Sentry from '@sentry/node'
import { nodeProfilingIntegration } from '@sentry/profiling-node'

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  integrations: [
    nodeProfilingIntegration(),
  ],
  tracesSampleRate: 0.1,    // 10% de requests trazados
  profilesSampleRate: 0.1,
  beforeSend(event) {
    // No enviar errores esperados (401, 404, 422)
    if (event.exception?.values?.[0]?.type === 'HttpException') {
      const status = event.extra?.status as number
      if ([401, 403, 404, 422].includes(status)) return null
    }
    return event
  },
})
```

## Métricas de Negocio (Grafana — Fase 2)

Dashboards predefinidos para los owners:

### Dashboard Financiero
- Ingresos del día (actualización cada 5 minutos)
- Gastos pendientes de aprobación
- Descuadres de caja detectados
- Alertas de importaciones con comisiones anómalas

### Dashboard Operativo
- OTs activas en este momento por mecánico
- OTs completadas en el día vs. promedio histórico
- Vehículos en taller (incluyendo los del propio taller en uso)
- Stock por debajo del mínimo

### Dashboard de Seguridad
- Intentos de login fallidos por IP
- Accesos fuera de horario laboral
- Exportaciones de datos en las últimas 24h
- Modificaciones al módulo de auditoría

## Alertas Críticas de Negocio

```yaml
# alertas-negocio.yml (CloudWatch / Grafana Alerts)
alertas:
  - nombre: "Descuadre de Caja"
    condicion: cashbox_discrepancy_amount > 5
    severidad: CRITICA
    destino: [push_owners, email_owners, whatsapp_owners]

  - nombre: "Vehículo Fuera del Perímetro"
    condicion: vehicle_geofence_violation = true
    severidad: CRITICA
    destino: [push_owners_inmediato]

  - nombre: "Login Sospechoso"
    condicion: unusual_device_login = true
    severidad: ALTA
    destino: [push_owner_afectado, email_owner_afectado]

  - nombre: "Backend API Caído"
    condicion: health_check_failures >= 3
    severidad: CRITICA
    destino: [push_owners, email_tech, actualizar_status_dashboard]

  - nombre: "Error Rate Alto"
    condicion: error_rate_5min > 5%
    severidad: ALTA
    destino: [sentry_alert, push_tech]
```

## Health Check Endpoint

```typescript
// GET /health — público, sin autenticación
@Get('health')
async healthCheck(): Promise<HealthCheckResult> {
  return {
    status: 'ok',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    services: {
      database: await this.checkDatabase(),
      redis: await this.checkRedis(),
      storage: await this.checkStorage(),
    },
    version: process.env.APP_VERSION,
  }
}
```

## Retención de Logs

| Tipo de log | Retención | Almacenamiento |
|-------------|-----------|---------------|
| Audit logs (negocio) | Indefinido | PostgreSQL `audit_logs` |
| Application logs | 30 días | Axiom / CloudWatch |
| Error logs (Sentry) | 90 días | Sentry |
| Access logs (API Gateway) | 90 días | Axiom / CloudWatch |
| Backup verification | 1 año | S3 |
