# Affine — Self-hosted

Reemplazo de Notion sin límites de seats. Open source MIT.

URL: https://affine.mystoredigital.cloud
Infra: Oracle Cloud Free Tier (ARM64) via Dokploy + Traefik + Let's Encrypt
Repo: My-Store-Digital-Team/affine

## Stack

- `affine` — server principal (Next.js + NestJS), puerto 3010
- `affine_migration` — job one-shot que corre antes del server
- `postgres` — pgvector/pgvector:pg16
- `redis` — redis:7-alpine

Variables de entorno se configuran en Dokploy → Affine → Environment.
