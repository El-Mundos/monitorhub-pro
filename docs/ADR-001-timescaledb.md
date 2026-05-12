# ADR-001: Uso de TimescaleDB para almacenamiento de series temporales

**Estado:** Aceptado  
**Fecha:** 2025-10-14  
**Autores:** Sergio Tabernero Hernández  

---

## Contexto

MonitorHub Pro necesita almacenar y consultar grandes volúmenes de métricas con marca de tiempo (CPU, memoria, latencia, etc.). Las consultas más frecuentes son:

- Agregaciones por ventana temporal (`avg`, `max`, `percentile`) en rangos de minutos a días
- Filtrado por `tenant_id` + `timestamp` combinado
- Retención configurable por tenant (30–365 días)
- Escrituras de alta frecuencia (objetivo: 10.000 métricas/segundo)

Se evaluaron tres alternativas:

| Opción | Ventajas | Inconvenientes |
|---|---|---|
| **PostgreSQL puro** | Ya en stack, sin dependencias extra | Queries temporales lentas a escala, sin particionado automático |
| **InfluxDB** | Diseñado para series temporales, buena comunidad | Lenguaje de query propio (Flux), no SQL, difícil integrar con RLS |
| **TimescaleDB** | Extensión de PostgreSQL, SQL estándar, particionado automático por tiempo | Requiere conocer el modelo de hypertables |

---

## Decisión

Se adopta **TimescaleDB** como motor de almacenamiento para todas las tablas de métricas.

Las tablas de métricas se crean como `hypertables` con chunk interval de 1 día:

```sql
SELECT create_hypertable('metrics', 'timestamp', chunk_time_interval => INTERVAL '1 day');
```

Los índices principales son compuestos `(tenant_id, timestamp DESC)` para optimizar las queries de filtrado por tenant en ventanas temporales.

La retención de datos se gestiona mediante policies de compresión y drop automático:

```sql
SELECT add_compression_policy('metrics', INTERVAL '7 days');
SELECT add_retention_policy('metrics', INTERVAL '90 days');
```

---

## Consecuencias

**Positivas:**
- Las queries de agregación temporal mejoran entre 10× y 50× respecto a PostgreSQL puro en benchmarks internos con datasets de 100M+ filas
- Se mantiene SQL estándar, compatible con el resto del stack y con Row Level Security (ver ADR-002)
- El particionado automático elimina la necesidad de gestión manual de tablas de particiones

**Negativas:**
- Añade una dependencia a la imagen Docker de PostgreSQL (imagen `timescale/timescaledb-ha`)
- El equipo necesita entender el modelo de hypertables y chunks para operar correctamente
- Las migraciones de schema en hypertables requieren precaución adicional

**Neutrales:**
- Los backups siguen funcionando con `pg_dump` estándar
- La monitorización de la propia BD usa las mismas herramientas de siempre

---

## Alternativas descartadas

**InfluxDB** fue descartado principalmente por la incompatibilidad con Row Level Security, que es un requisito de seguridad no negociable para el modelo multi-tenant (ver ADR-002). Migrar a un lenguaje de query propio (Flux) también añadiría complejidad sin beneficio claro frente a TimescaleDB.

**PostgreSQL puro** fue descartado tras medir que las queries de agregación sobre 30 días de datos con 50 tenants activos superaban los 8 segundos de tiempo de respuesta, muy por encima del SLA de 2 segundos definido en los requisitos.

---

## Referencias

- TimescaleDB Docs — Hypertables: https://docs.timescale.com/use-timescale/latest/hypertables/
- IEEE 1016-2009 — Software Design Descriptions
