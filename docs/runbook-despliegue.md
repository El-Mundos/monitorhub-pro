# Runbook: Despliegue de MonitorHub Pro

**Versión:** 1.0 · **Última actualización:** Mayo 2026

---

## Despliegue estándar (rama main → staging)

El despliegue es automático vía GitHub Actions cuando hay un push a `main`. Este runbook cubre el despliegue manual y los procedimientos de rollback.

---

## Pre-requisitos

- Acceso SSH al servidor de producción
- Docker 26.0+ y Docker Compose v2 instalados
- Variables de entorno configuradas en `.env`
- Acceso de escritura al GitHub Container Registry

---

## Procedimiento de despliegue manual

### 1. Verificar estado actual

```bash
docker compose ps
docker compose logs --tail=50
```

### 2. Hacer pull de la nueva imagen

```bash
docker compose pull
```

### 3. Aplicar migraciones de base de datos

```bash
docker compose run --rm api npm run migrate
```

Verificar que las migraciones se aplicaron correctamente:

```bash
docker compose run --rm api npm run migrate:status
```

### 4. Desplegar con zero-downtime

```bash
docker compose up -d --no-deps --scale api=2 api
sleep 30
docker compose up -d --no-deps --scale api=1 api
```

### 5. Verificar health checks

```bash
curl http://localhost:4000/health
curl http://localhost:8000/health
```

Respuesta esperada: `{"status": "ok", "version": "x.y.z"}`

### 6. Verificar métricas post-despliegue

- Comprobar latencia p95 en el dashboard de MonitorHub Pro
- Verificar que no hay alertas activas de tipo `critical`
- Revisar logs de errores los primeros 5 minutos

---

## Rollback

Si se detecta un problema tras el despliegue:

```bash
# Ver historial de imágenes disponibles
docker images | grep monitorhub

# Rollback a la versión anterior
docker compose stop api
docker compose run --rm -e IMAGE_TAG=anterior api npm run migrate:rollback
docker compose up -d api
```

---

## Troubleshooting frecuente

### La API no responde tras el despliegue

```bash
docker compose logs api --tail=100
# Buscar: "Cannot connect to database" o "Redis connection refused"
docker compose restart pgbouncer redis
```

### Migración falla a mitad

```bash
# Ver estado de migraciones
docker compose run --rm api npm run migrate:status

# Revertir última migración
docker compose run --rm api npm run migrate:rollback:one

# Revisar logs de PostgreSQL
docker compose logs db --tail=50
```

### PgBouncer rechaza conexiones

```bash
# Ver estadísticas de PgBouncer
docker compose exec pgbouncer psql -p 6432 -U pgbouncer pgbouncer -c "SHOW STATS;"
docker compose exec pgbouncer psql -p 6432 -U pgbouncer pgbouncer -c "SHOW POOLS;"
```
