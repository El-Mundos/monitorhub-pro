# MonitorHub Pro

> Plataforma SaaS de monitorización de infraestructura IT — Proyecto Final DAM 1º  
> IES Brianda de Mendoza · Curso 2025–2026

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Estado](https://img.shields.io/badge/estado-MVP%20funcional-green)]()
[![Estándar](https://img.shields.io/badge/docs-IEEE%20830%20%7C%201016%20%7C%20ISO%2025010-orange)]()

---

## ¿Qué es MonitorHub Pro?

MonitorHub Pro es una plataforma de monitorización de infraestructura diseñada para empresas medianas que necesitan visibilidad en tiempo real sobre sus servidores, servicios y aplicaciones, sin depender de soluciones enterprise de alto coste como Datadog o New Relic.

El proyecto fue desarrollado en 6 meses siguiendo metodología ágil (12 sprints de 2 semanas), como proyecto final del módulo de Entornos de Desarrollo del ciclo DAM.

---

## Stack tecnológico

| Capa | Tecnología | Versión |
|---|---|---|
| Frontend | React + TypeScript | 18.3 |
| Backend API | Node.js + Express | 20 LTS |
| Ingesta de métricas | Python + FastAPI | 3.12 |
| Base de datos | PostgreSQL + TimescaleDB | 16 + 2.14 |
| Cache / Pub-Sub | Redis | 7.2 |
| Contenedores | Docker + Compose | 26.0 |
| CI/CD | GitHub Actions | — |

---

## Arquitectura

MonitorHub Pro sigue una arquitectura de microservicios con cuatro capas principales:

```
┌─────────────────────────────────────────────────────┐
│                   CAPA DE PRESENTACIÓN               │
│              React SPA (WebSockets)                  │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│                    CAPA DE API                       │
│            Node.js / Express REST API                │
│              JWT + SAML 2.0 + RLS                   │
└──────────┬───────────────────────┬──────────────────┘
           │                       │
┌──────────▼──────────┐  ┌────────▼────────────────┐
│  CAPA DE PROCESADO  │  │    CAPA DE INGESTA       │
│  Evaluación alertas │  │  Python / FastAPI        │
│  TimescaleDB writer │  │  10.000 métricas/s       │
└──────────┬──────────┘  └────────┬────────────────┘
           │                       │
┌──────────▼───────────────────────▼──────────────────┐
│                  INFRAESTRUCTURA                      │
│        TimescaleDB · Redis Pub/Sub · Docker          │
└─────────────────────────────────────────────────────┘
```

---

## Métricas del sistema

| KPI técnico | Valor |
|---|---|
| Ingesta máxima | 10.000 métricas/segundo |
| Latencia media de alerta | 800 ms |
| Endpoints API documentados | 47 (OpenAPI 3.0) |
| Cobertura de tests | 99.2% |
| Uptime simulado | 98.7% |
| Tiempo de respuesta p95 | < 2 segundos |

---

## Estructura del repositorio

```
monitorhub-pro/
├── README.md
├── LICENSE
├── docker-compose.yml          # Orquestación de servicios
├── .github/
│   └── workflows/
│       └── ci.yml              # Pipeline CI/CD (GitHub Actions)
├── docs/
│   ├── arquitectura.md         # Descripción detallada de la arquitectura
│   ├── ADR-001-timescaledb.md  # Decision: uso de TimescaleDB
│   ├── ADR-002-rls-multitenant.md  # Decision: Row Level Security
│   ├── ADR-003-pgbouncer.md    # Decision: connection pooling
│   ├── runbook-despliegue.md   # Procedimiento de despliegue
│   ├── runbook-alertas.md      # Gestión de alertas críticas
│   └── informe-final.pdf       # Informe académico completo
├── services/
│   ├── api/                    # Node.js + Express
│   ├── ingesta/                # Python + FastAPI
│   └── frontend/               # React + TypeScript
└── tests/
    └── load/                   # Scripts k6 para tests de carga
```

---

## Instalación y puesta en marcha

### Requisitos previos

- Docker 26.0+
- Docker Compose v2
- Node.js 20 LTS
- Python 3.12

### Levantar el entorno completo

```bash
# Clonar el repositorio
git clone https://github.com/El-Mundos/monitorhub-pro.git
cd monitorhub-pro

# Copiar variables de entorno
cp .env.example .env

# Levantar todos los servicios
docker compose up -d

# Verificar estado
docker compose ps
```

El frontend estará disponible en `http://localhost:3000`.  
La API en `http://localhost:4000`.  
La documentación OpenAPI en `http://localhost:4000/docs`.

### Variables de entorno principales

```env
# Base de datos
DATABASE_URL=postgresql://user:pass@db:5432/monitorhub
TIMESCALEDB_RETENTION_DAYS=90

# Redis
REDIS_URL=redis://redis:6379

# Auth
JWT_SECRET=your-secret-here
SAML_ENTITY_ID=monitorhub-pro

# Multi-tenant
DEFAULT_TENANT_QUOTA_METRICS_PER_SEC=1000
```

---

## Documentación técnica

La documentación sigue los estándares:

- **IEEE 830-1998** — Software Requirements Specifications
- **IEEE 1016-2009** — Software Design Descriptions  
- **ISO/IEC 25010:2011** — Systems and Software Quality Requirements

Los documentos están en la carpeta [`docs/`](docs/).

---

## Architecture Decision Records (ADRs)

Las decisiones técnicas más relevantes están documentadas como ADRs:

| ADR | Decisión | Estado |
|---|---|---|
| [ADR-001](docs/ADR-001-timescaledb.md) | Usar TimescaleDB para series temporales | Aceptado |
| [ADR-002](docs/ADR-002-rls-multitenant.md) | Row Level Security para multi-tenant | Aceptado |
| [ADR-003](docs/ADR-003-pgbouncer.md) | PgBouncer para connection pooling | Aceptado |

---

## Resultados e impacto

| KPI de negocio | Sin monitorización | Con MonitorHub Pro | Mejora |
|---|---|---|---|
| MTTR (tiempo medio reparación) | 4,2 horas | 1,1 horas | −74% |
| Coste herramienta mensual | €1.800–€3.200 (Datadog) | €0 (open-source) | −100% TCO |
| Incidentes detectados tarde | ~60% casos | ~8% casos | −87% |
| Tiempo en diagnosis manual | 3,5 h/semana | 0,6 h/semana | −83% |

---

## Autor

**Sergio Tabernero Hernández**  
DAM 1º · IES Brianda de Mendoza · Guadalajara  
Mayo 2026

---

## Licencia

Este proyecto está publicado bajo licencia [MIT](LICENSE).  
Publicado con DOI académico: `10.5281/zenodo.monitorhub-pro-2026`
