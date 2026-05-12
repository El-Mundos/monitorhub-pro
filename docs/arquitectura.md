# Arquitectura de MonitorHub Pro

> Documento de diseño siguiendo IEEE 1016-2009 (Software Design Descriptions)

**Versión:** 1.0  
**Fecha:** Mayo 2026  
**Autor:** Sergio Tabernero Hernández  

---

## 1. Visión general

MonitorHub Pro sigue una arquitectura de **microservicios desacoplados** comunicados a través de Redis Pub/Sub. El sistema está diseñado para operar en un entorno containerizado (Docker) y puede escalar horizontalmente cada capa de forma independiente.

```
Cliente (navegador / agente)
         │
         ▼
   ┌─────────────┐
   │  Nginx      │  ← Reverse proxy / TLS termination
   │  (puerto 80/│
   │   443)      │
   └──────┬──────┘
          │
    ┌─────┴──────────────────┐
    │                        │
    ▼                        ▼
┌───────────┐         ┌─────────────┐
│  API      │         │  Ingesta    │
│  Node.js  │         │  FastAPI    │
│  :4000    │         │  :8000      │
└─────┬─────┘         └──────┬──────┘
      │                      │
      │              ┌───────▼──────┐
      │              │    Redis     │
      │              │  Pub/Sub     │
      │              └───────┬──────┘
      │                      │
      │              ┌───────▼──────┐
      │              │  Procesador  │
      │              │  (alertas)   │
      │              └───────┬──────┘
      │                      │
      └──────────┬───────────┘
                 │
         ┌───────▼────────┐
         │   PgBouncer    │
         │   :5432        │
         └───────┬────────┘
                 │
         ┌───────▼────────┐
         │  PostgreSQL +  │
         │  TimescaleDB   │
         │  :5433         │
         └────────────────┘
```

---

## 2. Componentes

### 2.1 Servicio de ingesta (Python / FastAPI)

**Responsabilidad:** Recibir métricas de los agentes instalados en los servidores monitorizados y publicarlas en Redis.

**Endpoints principales:**

```
POST /ingest/metrics          — Ingesta de métricas individuales o en batch
POST /ingest/metrics/batch    — Ingesta optimizada de hasta 1.000 métricas por request
GET  /health                  — Health check
```

**Flujo:**

1. El agente envía un POST con las métricas en formato JSON
2. El servicio valida el payload (Pydantic) y verifica el API key del tenant
3. Publica el evento en Redis: `PUBLISH metrics:{tenant_id} {payload}`
4. Responde 202 Accepted — no espera confirmación de escritura en BD

**Rendimiento medido:** 10.000 métricas/segundo con latencia p99 < 50ms.

---

### 2.2 Servicio de procesamiento (Node.js worker)

**Responsabilidad:** Consumir eventos de Redis, evaluar condiciones de alerta y escribir en TimescaleDB.

**Flujo:**

1. Suscripción a `metrics:*` en Redis
2. Para cada evento: comprobación de reglas de alerta configuradas por el tenant
3. Si se cumple una condición: enqueue de notificación + escritura en `alert_events`
4. Escritura de la métrica en TimescaleDB (batch insert cada 500ms)

---

### 2.3 API REST (Node.js / Express)

**Responsabilidad:** Exponer datos al frontend y a clientes externos. Gestionar autenticación, autorización y lógica de negocio.

**Autenticación:**

- JWT (HS256) con expiración de 1 hora para usuarios internos
- API Keys para agentes de ingesta
- SAML 2.0 para SSO con proveedores de identidad corporativos

**Aislamiento multi-tenant:**

Al inicio de cada request autenticado, el middleware establece el tenant activo:

```javascript
// middleware/tenant.js
app.use(async (req, res, next) => {
  const tenantId = req.user.tenantId;
  await db.query("SET LOCAL app.current_tenant_id = $1", [tenantId]);
  next();
});
```

Row Level Security en PostgreSQL garantiza que las queries solo devuelven datos del tenant activo (ver [ADR-002](ADR-002-rls-multitenant.md)).

**Documentación:** OpenAPI 3.0 disponible en `/docs` (Swagger UI). 47 endpoints documentados.

---

### 2.4 Frontend (React / TypeScript)

**Responsabilidad:** Interfaz de usuario. Dashboards en tiempo real, gestión de alertas, administración de la cuenta.

**Comunicación en tiempo real:** WebSocket conectado al API para actualizaciones de métricas sin polling.

**Widgets disponibles:**

| Tipo | Descripción |
|---|---|
| Gráfica temporal | Serie temporal con zoom y pan |
| Gauge | Valor actual con rangos configurables |
| Tabla | Grid con ordenación y filtros |
| Mapa de calor | Distribución de valores en el tiempo |
| Contador | Valor numérico grande con tendencia |
| Barra de estado | Estado OK/WARN/CRIT de servicios |
| Sparkline | Mini gráfica de tendencia |
| Pie chart | Distribución proporcional |
| Histograma | Distribución de frecuencias |
| Top N | Ranking de los N valores más altos |
| Texto | Widget de texto libre / markdown |
| Iframe | Embed de contenido externo |

---

## 3. Modelo de datos principal

### 3.1 Tabla `metrics` (hypertable TimescaleDB)

```sql
CREATE TABLE metrics (
    id          BIGSERIAL,
    tenant_id   UUID        NOT NULL,
    source      TEXT        NOT NULL,  -- hostname / servicio
    name        TEXT        NOT NULL,  -- cpu.usage, mem.used, etc.
    value       DOUBLE PRECISION NOT NULL,
    tags        JSONB,
    timestamp   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (id, timestamp)
);

SELECT create_hypertable('metrics', 'timestamp',
    chunk_time_interval => INTERVAL '1 day');

CREATE INDEX ON metrics (tenant_id, timestamp DESC);
CREATE INDEX ON metrics (tenant_id, source, name, timestamp DESC);
```

### 3.2 Tabla `alert_rules`

```sql
CREATE TABLE alert_rules (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID        NOT NULL,
    name        TEXT        NOT NULL,
    metric_name TEXT        NOT NULL,
    condition   TEXT        NOT NULL,  -- 'gt', 'lt', 'eq'
    threshold   DOUBLE PRECISION NOT NULL,
    window_secs INTEGER     NOT NULL DEFAULT 60,
    severity    TEXT        NOT NULL,  -- 'info', 'warning', 'critical'
    enabled     BOOLEAN     NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## 4. CI/CD Pipeline

```yaml
# .github/workflows/ci.yml (resumen)
on: [push, pull_request]

jobs:
  test:
    - Lint (ESLint + Ruff)
    - Unit tests (Jest + pytest)
    - Integration tests (Docker Compose)
    - Load test (k6 — falla si p95 > 2000ms)

  build:
    - Docker build multi-stage
    - Push a GitHub Container Registry

  deploy:
    - Deploy a entorno de staging (si rama main)
```

---

## 5. Decisiones de diseño

Ver carpeta `docs/` para los Architecture Decision Records completos:

- [ADR-001](ADR-001-timescaledb.md) — TimescaleDB para series temporales
- [ADR-002](ADR-002-rls-multitenant.md) — Row Level Security para multi-tenant
- [ADR-003](ADR-003-pgbouncer.md) — PgBouncer para connection pooling

---

## 6. Referencias normativas

- IEEE 1016-2009 — IEEE Standard for Information Technology: Systems Design — Software Design Descriptions
- ISO/IEC 25010:2011 — Systems and software Quality Requirements and Evaluation
