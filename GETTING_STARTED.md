# Castivas Cleaning — Getting started (local)

Sibling repos/folders: `back/` (API) and `front/` (SPA), next to this `docs/` tree.

## Prerequisites

- Docker and Docker Compose
- Node **20** (for Vite/npm on the host)
- Git

## Backend

```bash
cd back
cp -r .envs.example/.local .envs/.local   # if missing
docker compose -f local.yml build
docker compose -f local.yml up -d postgres django
docker compose -f local.yml exec django python manage.py migrate
docker compose -f local.yml exec django python manage.py seed_mvp_demo
```

API: **http://localhost:8010**

## Frontend

```bash
cd front
cp .env.example .env     # VITE_API_URL=http://localhost:8010
npm ci
npm run dev              # http://localhost:8080
```

Docker SPA alternative: `docker compose -f local.yml up --build` in `front/` → **http://localhost:8180**.

## Demo logins

Password `demo1234`: `ops@castivas.test`, `maria@abcpm.test`, `ana@castivas.test`, `carlos@castivas.test`.

## Next steps

- [Phase 1 developer guide](guides/mvp-dev-guide.md)
- [Phase 1 testing guide](guides/mvp-testing-guide.md)
- [Architecture](architecture/) · [Plan](plan/)
