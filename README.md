# Affine self-hosted — My Store Digital

Reemplazo de Notion. Open source MIT. Sin límite de seats (10 self-host basic, ampliable).

- **URL pública**: https://affine.mystoredigital.cloud
- **Admin panel**: https://affine.mystoredigital.cloud/admin
- **VPS**: Oracle Cloud Free Tier (ARM64) via Dokploy + Traefik + Let's Encrypt
- **Versión**: AFFiNE 0.26.6
- **Dokploy compose ID**: `QWfXVq8dOcCmO1cj7bVfj` (proyecto Affine/production)

## Stack

| Servicio | Rol |
|---|---|
| `affine` | Server principal (Next.js + NestJS), puerto interno 3010, expuesto via Traefik en HTTPS |
| `affine_migration` | Job one-shot, corre las migraciones de Prisma antes del server |
| `postgres` | pgvector/pg16, almacena docs, users, workspaces |
| `redis` | Cache + jobs de background |
| `template_proxy` | Reverse proxy con CORS para importar plantillas de affine.pro |

## Storage

**Blobs (imágenes, archivos, importaciones):** Cloudflare R2
- Bucket: `affine-storage`
- Endpoint S3: `https://f38d2420afafb61fc46018388f6f2e1f.r2.cloudflarestorage.com`
- Cero egress fees, 10 GB gratis, $0.015/GB/mes después
- Las credenciales R2 viven en la DB de Affine (configurables en `/admin → Settings → Storage`)

**Avatars:** mismo bucket R2 (`affine-storage`)

**Postgres + config:** volumes Docker locales en el VPS Oracle
- `affine_postgres` (DB)
- `affine_storage` (filesystem fallback, vacío después del switch a R2)
- `affine_config` (config del server)

## Email (SMTP)

Gmail Workspace con App Password:
- Host: `smtp.gmail.com`
- Port: 465 (SSL)
- User: `connect@mystoredigital.com`
- Verificado funcionando (auto-envíos al mismo email los filtra Gmail; a otras direcciones llegan OK).

## Plantillas (templates de affine.pro)

Las plantillas de https://affine.pro/templates se importan a través del proxy CORS de este repo
(servicio `template_proxy`).

**Cómo importar una plantilla:**
1. Andá a https://affine.pro/templates y abrí la plantilla que querés
2. Right-click sobre el botón "Use this template" → Copy link
3. Pegá el link en https://affine.mystoredigital.cloud/template-cdn/
4. Click Importar → se redirige a tu Affine con la plantilla cargada

El proxy:
- Sirve la página HTML estática `template-proxy/import.html` en `/template-cdn/`
- Reverse-proxy a `cdn.affine.pro/template-snapshots/*` agregando los CORS headers
  que el CDN oficial de Affine no devuelve

## Quotas / Plan

Affine self-hosted FOSS gratuito tiene defaults limitantes:
- 10 GB storage total
- 10 MB por archivo individual
- 3 members por workspace

Override aplicado manualmente en DB para el primer admin (`connect@mystoredigital.com`):
plan `lifetime_pro_plan_v1` → 1 TB storage / 100 MB por archivo / 30 días history.

Aplicado vía:
```sql
UPDATE user_features
SET name = 'lifetime_pro_plan_v1'
WHERE user_id = (SELECT id FROM users WHERE email = 'connect@mystoredigital.com')
  AND name = 'free_plan_v1';
```

NO requiere licencia comercial ni paga. Aprovecha un plan que existe en el código
pero que normalmente sólo se asigna en SaaS Pro.

El HTTP body máximo del server es 100 MB (hardcoded en el binario, sin env var override).
Para uploads de archivos individuales > 100 MB hay que parchar el binario.

## Gotchas conocidos

- **Tag `stable` está desactualizado** (apuntaba a 0.19.0 nightly). Usar tag semver explícito
  como `0.26.6`. El cliente web requiere server >= 0.23.0.
- **CDN de templates no tiene CORS headers**. Por eso el `template_proxy` es necesario para
  importar plantillas oficiales sin error CORS.
- **Workspaces "Local storage"** guardan datos sólo en el navegador del usuario. Activar
  "AFFiNE Cloud" en el workspace (botón en banner rojo) los migra al server self-hosted.
  Engaño visual: "AFFiNE Cloud" en self-hosted significa TU server, no el SaaS de Affine.
- **Las 100 GB que muestra la UI son nominales** del plan asignado, no significan que se
  transmita data al SaaS de Affine. Verificable: las tablas `subscriptions`, `licenses` y
  `installed_licenses` deben estar vacías para confirmar zero tráfico al SaaS.
- **`affine_migration` job debe completar exitosamente** antes de que `affine` arranque
  (definido en `depends_on` con `service_completed_successfully`). Si falla la migración,
  el server no inicia.

## Operaciones comunes

**Ver logs del server:**
```bash
ssh ubuntu@150.230.28.174 'sudo docker logs --tail 100 affine_server'
```

**Listar users:**
```bash
ssh ubuntu@150.230.28.174 'sudo docker exec affine_postgres psql -U affine -d affine -c \
  "SELECT email, name, created_at FROM users;"'
```

**Test SMTP desde el container:**
```bash
ssh ubuntu@150.230.28.174 'sudo docker exec affine_server node -e "
const nm = require(\"nodemailer\");
const t = nm.createTransport({host:process.env.MAILER_HOST,port:+process.env.MAILER_PORT,secure:true,auth:{user:process.env.MAILER_USER,pass:process.env.MAILER_PASSWORD}});
t.verify().then(()=>console.log(\"OK\"));"'
```

**Redeploy desde Dokploy API:**
```bash
curl -X POST -H "x-api-key: $DOKPLOY_ORACLE_TOKEN" \
  -d '{"composeId":"QWfXVq8dOcCmO1cj7bVfj"}' \
  https://dokploy-oci.mystoredigital.cloud/api/compose.deploy
```

**Bumpear plan de un user:**
```bash
ssh ubuntu@150.230.28.174 'sudo docker exec affine_postgres psql -U affine -d affine -c \
  "UPDATE user_features SET name = '\''lifetime_pro_plan_v1'\'' \
   WHERE user_id = (SELECT id FROM users WHERE email = '\''X@Y.com'\'') \
   AND name = '\''free_plan_v1'\'';"'
```

## Backup

Lo importante a respaldar:
1. **Postgres**: `affine_postgres` volume — contiene docs, users, workspaces, settings.
2. **R2 bucket** `affine-storage`: blobs, avatars, archivos subidos.
3. **Config**: `affine_config` volume.

Cloudflare R2 ya tiene durabilidad cross-region. Postgres se respalda via Dokploy
o pg_dump manual al storage local.
