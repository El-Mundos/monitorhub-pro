# ADR-003: PgBouncer para connection pooling

**Estado:** Aceptado  
**Fecha:** 2025-12-01  
**Autores:** Sergio Tabernero Hernández  

---

## Contexto

Durante los tests de carga con k6 (Sprint 7), se detectó que la API comenzaba a degradarse significativamente al superar las 500 peticiones concurrentes. El tiempo de respuesta p95 pasaba de ~200ms a más de 4 segundos.

El análisis identificó tres cuellos de botella:

1. **Queries N+1** en el ORM para relaciones de alertas → resuelto con eager loading
2. **Índices ausentes** en columnas de filtrado temporal → resuelto añadiendo índices compuestos
3. **Sin connection pooling**: cada request de la API abría una conexión nueva a PostgreSQL

PostgreSQL tiene un límite práctico de ~100–200 conexiones simultáneas antes de degradar el rendimiento por overhead de gestión de procesos. Con 500 requests concurrentes, la BD estaba saturada de conexiones.

---

## Decisión

Se introduce **PgBouncer** como proxy de connection pooling entre la API y PostgreSQL.

Configuración adoptada:

```ini
[databases]
monitorhub = host=db port=5432 dbname=monitorhub

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
reserve_pool_size = 5
reserve_pool_timeout = 3
```

Se usa `pool_mode = transaction` (una conexión del pool se asigna a un cliente solo durante la duración de una transacción, no de toda la sesión). Esto maximiza la reutilización de conexiones.

**Consideración importante con RLS (ADR-002):** `SET LOCAL` solo afecta a la transacción actual, lo cual es compatible con `pool_mode = transaction`. Si se usara `SET` (sin LOCAL), el valor persistiría en la conexión pooled y podría contaminar requests de otros tenants. Todos los `SET` de `app.current_tenant_id` en la capa de API usan explícitamente `SET LOCAL`.

---

## Consecuencias

**Positivas:**
- El tiempo de respuesta p95 bajo carga pasó de 4.100ms a 530ms (mejora del 87%)
- PostgreSQL opera con 25 conexiones reales independientemente del número de clientes concurrentes
- PgBouncer expone métricas propias (`SHOW STATS`) integrables con el sistema de monitorización

**Negativas:**
- `pool_mode = transaction` es incompatible con `LISTEN/NOTIFY` de PostgreSQL y con cursores con `HOLD`. Ninguna de estas features se usa en MonitorHub Pro, pero hay que tenerlo en cuenta para futuras funcionalidades
- Añade un componente de infraestructura adicional que debe monitorizarse y mantenerse
- La configuración de `SET LOCAL` para RLS debe revisarse en cualquier nuevo endpoint que abra transacciones

**Neutrales:**
- Los tests de carga con k6 pasaron a ejecutarse como parte del pipeline CI/CD (ver workflow `ci.yml`) para detectar regresiones de rendimiento tempranamente

---

## Alternativas descartadas

**pgpool-II** fue considerado pero descartado por su mayor complejidad de configuración y por incluir features (replicación, load balancing) que no son necesarias en este proyecto. PgBouncer es más ligero y más adecuado para el caso de uso de pooling simple.

**Connection pool en la aplicación** (usando el pool interno de `pg` en Node.js) fue la configuración inicial. El problema es que cada instancia de la API tiene su propio pool, y al escalar horizontalmente el número total de conexiones a PostgreSQL crece linealmente con el número de instancias. PgBouncer centraliza el pooling.

---

## Referencias

- PgBouncer Documentation: https://www.pgbouncer.org/config.html
- k6 Load Testing: https://k6.io/docs/
- IEEE 1016-2009 — Software Design Descriptions
