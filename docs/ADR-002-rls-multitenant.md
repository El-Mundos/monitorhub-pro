# ADR-002: Row Level Security (RLS) para aislamiento multi-tenant

**Estado:** Aceptado  
**Fecha:** 2025-11-03  
**Autores:** Sergio Tabernero Hernández  

---

## Contexto

MonitorHub Pro es una plataforma multi-tenant: múltiples organizaciones (tenants) comparten la misma infraestructura pero sus datos deben estar completamente aislados. Un tenant nunca debe poder ver, modificar ni eliminar datos de otro.

El diseño inicial de la capa de aplicación usaba un esquema **shared-database / shared-schema**: todas las tablas incluyen una columna `tenant_id` y la API filtra por ella en cada query.

El problema de este enfoque es que el aislamiento depende completamente de que **ninguna query en el código de aplicación olvide el filtro `WHERE tenant_id = $1`**. Un bug, un endpoint nuevo sin revisar, o una query de debugging en producción pueden provocar data leakage entre tenants.

Se evaluaron dos enfoques adicionales:

| Enfoque | Descripción | Seguridad | Complejidad operacional |
|---|---|---|---|
| **Shared schema + filtro en app** | Una BD, tenant_id en cada tabla, filtro en código | Baja (depende del código) | Baja |
| **Schema por tenant** | Un schema PostgreSQL distinto por tenant | Alta | Alta (migraciones × N tenants) |
| **RLS en PostgreSQL** | Políticas de seguridad a nivel de fila en la BD | Alta (garantía a nivel de motor) | Media |

---

## Decisión

Se implementa **Row Level Security (RLS)** de PostgreSQL en todas las tablas que contienen datos de tenant.

Se define una variable de sesión `app.current_tenant_id` que la capa de API establece al inicio de cada conexión:

```sql
-- Al inicio de cada request en el connection pool
SET LOCAL app.current_tenant_id = '{{tenant_uuid}}';
```

Las políticas RLS garantizan que solo se devuelven filas del tenant activo:

```sql
-- Habilitar RLS en la tabla
ALTER TABLE metrics ENABLE ROW LEVEL SECURITY;

-- Política de lectura
CREATE POLICY tenant_isolation_select ON metrics
  FOR SELECT
  USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

-- Política de escritura
CREATE POLICY tenant_isolation_insert ON metrics
  FOR INSERT
  WITH CHECK (tenant_id = current_setting('app.current_tenant_id')::uuid);
```

Con este esquema, **aunque el código de aplicación olvide el filtro**, el motor de base de datos rechaza las filas de otros tenants. Es una segunda línea de defensa que opera independientemente de la lógica de aplicación.

---

## Consecuencias

**Positivas:**
- El aislamiento se garantiza a nivel de motor de base de datos, independientemente de bugs en el código de aplicación
- Compatible con TimescaleDB (ver ADR-001) — RLS funciona sobre hypertables sin modificaciones
- El overhead medido en benchmarks es inferior al 3% en queries típicas

**Negativas:**
- Las queries de administración (migraciones, backups por tenant, reporting cross-tenant) requieren un rol de base de datos con `BYPASSRLS`
- El connection pooling (PgBouncer, ver ADR-003) debe configurarse en modo `session` o `transaction` con cuidado para que `SET LOCAL` funcione correctamente
- El debugging de problemas de acceso es más complejo porque el filtro es invisible en los logs de aplicación

**Neutrales:**
- El esquema de tablas no cambia: `tenant_id` sigue siendo una columna estándar, pero ahora también es la columna sobre la que aplican las políticas RLS

---

## Alternativas descartadas

**Schema por tenant** fue descartado porque con 50+ tenants activos las migraciones de schema se convierten en una operación de riesgo elevado (hay que aplicar cada migración N veces, gestionar fallos parciales, etc.). El overhead operacional no está justificado para este proyecto.

**Filtro solo en aplicación** fue descartado por el riesgo de data leakage ante un bug. En una plataforma multi-tenant la seguridad de los datos es un requisito no negociable, y depender exclusivamente del código de aplicación para garantizarla es una práctica inadecuada.

---

## Referencias

- PostgreSQL Docs — Row Security Policies: https://www.postgresql.org/docs/current/ddl-rowsecurity.html
- ISO/IEC 25010:2011 — Security (confidentiality, integrity)
- IEEE 1016-2009 — Software Design Descriptions
