# Plan de despliegue: versión apuntando a PostgreSQL

Este documento describe los pasos para desplegar la versión que usa **PostgreSQL** como base de datos en los proyectos **meetzy-ai-server**, **flark-el-server** y **meetzy-jobs**.

---

## 1. Estado actual por proyecto

| Proyecto           | Base de datos actual | Soporte PostgreSQL |
|--------------------|----------------------|---------------------|
| **flark-el-server** | ClickHouse (por defecto) | ✅ Sí, vía `DB_MODE=postgres` |
| **meetzy-ai-server** | Solo ClickHouse       | ❌ No implementado |
| **meetzy-jobs**      | ClickHouse (por defecto) | ✅ Sí, vía `DB_MODE=postgres` |

- **flark-el-server** ya tiene un facade de base de datos (`database-facade.ts`) que permite `DB_MODE`: `clickhouse` | `postgres` | `both`. Solo hay que configurar variables de entorno y desplegar.
- **meetzy-jobs** tiene un facade (`database_facade.js`) que permite `DB_MODE`: `clickhouse` | `postgres` | `both`. Usa `database_clickhouse.js` y `database_pg.js`; las consultas se convierten automáticamente a PostgreSQL cuando `DB_MODE=postgres`.
- **meetzy-ai-server** sigue usando únicamente `database_clickhouse.js`; no tiene capa PostgreSQL.

---

## 2. Despliegue de flark-el-server con PostgreSQL

Este es el único servicio que hoy puede desplegarse en modo PostgreSQL sin cambios de código.

### 2.1 Requisitos previos

- PostgreSQL de producción accesible desde la red donde corre ECS (VPC, security groups).
- Migraciones de esquema aplicadas en PostgreSQL (tablas en snake_case equivalentes a las de ClickHouse).
- Secret de AWS Secrets Manager con las variables necesarias.

### 2.2 Variables de entorno (AWS Secrets Manager)

flark-el-server en producción carga la configuración desde:

- **Secret:** `prod/flark/low-latency-config`  
- **Región:** `eu-south-2`

Para que use PostgreSQL, el JSON del secret debe incluir al menos:

```json
{
  "DB_MODE": "postgres",
  "PG_HOST": "<host-postgresql>",
  "PG_PORT": "5432",
  "PG_USER": "<usuario>",
  "PG_PASSWORD": "<contraseña>",
  "PG_DATABASE": "<nombre_bd>"
}
```

El resto de variables existentes (Twilio, ElevenLabs, Resend, etc.) se mantienen.

**Pasos:**

1. En AWS Console → Secrets Manager → `prod/flark/low-latency-config`.
2. Editar el secret (nueva versión).
3. Añadir o actualizar las claves anteriores.
4. Guardar (quedará una nueva versión con `AWSCURRENT`).

Si en algún entorno (p. ej. ECS task definition) se inyectan variables de entorno además del secret, asegurar que no se sobrescriba `DB_MODE` ni las `PG_*` con valores viejos.

### 2.3 Orden recomendado del despliegue

1. **Actualizar el secret** con `DB_MODE=postgres` y `PG_*` (como arriba).
2. **Desplegar la imagen** de flark-el-server (push a `production` dispara el workflow que construye y sube a ECR).
3. **Reiniciar el servicio** que usa esa imagen (p. ej. forzar nuevo despliegue del servicio ECS para que tome la imagen `latest` y la nueva versión del secret).

### 2.4 Pipeline actual (recordatorio)

- **Repositorio:** flark-el-server  
- **Trigger:** push a rama `production`  
- **Workflow:** `.github/workflows/aws.yml`  
- **Acción:** build de la imagen Docker y push a ECR:
  - `077460552379.dkr.ecr.eu-south-2.amazonaws.com/flark-lowlatency:latest`

El workflow no actualiza ECS; quien ejecute los contenedores (p. ej. ECS) debe estar configurado para usar esta imagen y volver a desplegar para tomar cambios.

### 2.5 Comprobaciones tras el despliegue

- Logs de arranque: debe aparecer algo como `Database mode: postgres` y la conexión a PostgreSQL (mensajes del pool en `database-pg.ts`).
- `GET /health`: debe responder 200 y en `database.pool` verse estadísticas del pool de PostgreSQL (size, available, borrowed, etc.).
- Probar una llamada de prueba o un flujo crítico que toque BD para confirmar lecturas/escrituras contra PostgreSQL.

### 2.6 Rollback

- En Secrets Manager: crear una nueva versión del secret con `DB_MODE=postgres` eliminado o con `DB_MODE=clickhouse` (y que las variables de ClickHouse sigan presentes), o restaurar una versión anterior del secret que tuviera `clickhouse`.
- Volver a desplegar el servicio (o revertir a una imagen anterior si además cambiaste código) para que los nuevos tasks lean el secret actualizado.

---

## 3. Despliegue de meetzy-jobs con PostgreSQL

meetzy-jobs puede usar PostgreSQL mediante `DB_MODE=postgres` y variables `PG_*`.

### 3.1 Secret y variables

meetzy-jobs en producción carga la configuración desde:

- **Secret:** `prod/flark/jobs-config`
- **Región:** `eu-south-2`

Para usar PostgreSQL, el JSON del secret debe incluir:

```json
{
  "DB_MODE": "postgres",
  "PG_HOST": "<host-postgresql>",
  "PG_PORT": "5432",
  "PG_USER": "<usuario>",
  "PG_PASSWORD": "<contraseña>",
  "PG_DATABASE": "<nombre_bd>"
}
```

Opcional: `"PG_SSL": "true"` si la conexión usa TLS.

### 3.2 Comprobaciones

- Logs de arranque: `Database mode: postgres` y mensajes de conexión a PostgreSQL.
- `GET /ping`: responde con `dbMode`, `poolHealthy` y estadísticas del pool.

### 3.3 Rollback

- Nueva versión del secret con `DB_MODE=clickhouse` (o sin `DB_MODE`) y redesplegar.

---

## 4. meetzy-ai-server

Hoy **no tiene implementado** el uso de PostgreSQL; todo pasa por ClickHouse (`database_clickhouse.js`).

Opciones:

**A) Dejarlos en ClickHouse de momento**  
Solo se despliega la versión actual (sin cambios de BD). El “plan de despliegue a PostgreSQL” se aplica solo a flark-el-server.

**B) Que dependan de flark-el-server para datos**  
Si en el futuro toda la persistencia se hace vía APIs de flark-el-server (que ya usa PostgreSQL), meetzy-ai-server podría dejar de usar ClickHouse directamente y no necesitaría soporte PG propio.

**C) Añadir soporte PostgreSQL**  
Implementar en meetzy-ai-server una capa similar a la de flark-el-server o meetzy-jobs (facade + `database-pg` + variables `DB_MODE` y `PG_*`).

---

## 5. Resumen rápido (flark-el-server o meetzy-jobs en PostgreSQL)

1. **Secret** correspondiente: añadir/actualizar `DB_MODE=postgres` y `PG_HOST`, `PG_PORT`, `PG_USER`, `PG_PASSWORD`, `PG_DATABASE`.
   - flark-el-server: `prod/flark/low-latency-config`
   - meetzy-jobs: `prod/flark/jobs-config`
2. **Código** en rama `production` y push para que el workflow construya y suba la imagen a ECR.
3. **Redesplegar** el servicio para que arranque con el nuevo secret y la nueva imagen.
4. **Comprobar** logs (`Database mode: postgres`) y endpoint `/health` (flark-el-server) o `/ping` (meetzy-jobs).

---

## 6. Referencia de ECR y ramas

| Proyecto           | ECR Repository (eu-south-2)     | Rama de despliegue |
|--------------------|----------------------------------|---------------------|
| meetzy-ai-server   | `077460552379.dkr.ecr.../flark-api`       | `production` |
| flark-el-server    | `077460552379.dkr.ecr.../flark-lowlatency`| `production` |
| meetzy-jobs        | `077460552379.dkr.ecr.../flark-jobs`      | `production` |

Workflows: cada repo tiene `.github/workflows/aws.yml` o `aws-ecr.yml` que se ejecuta en push a `production`.
