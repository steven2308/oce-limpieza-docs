# Limpieza — Getting started (local)

## Prerequisites

- Docker and Docker Compose
- Git

## Clone repositories

- Clone the **backend** and **frontend** repos (sibling folders or your preferred layout).
- Replace `Limpieza` / `{{PROJECT_SLUG}}` placeholders per each repo README.

## Environment

- Copy backend env examples (e.g. `.envs.example/` → `.envs/.local/`) and adjust values.
- Copy frontend `.env.example` → `.env` and set the API base URL.

## Run backend

```bash
cd backend
docker-compose -f local.yml up
```

Create a superuser when the API is up:

```bash
docker-compose -f local.yml run django python manage.py createsuperuser
```

## Run frontend

```bash
cd frontend
docker-compose -f local.yml up
```

Open the app at **http://localhost:8080** (default local SPA port).

## Next steps

- Fill in [architecture/](architecture/) diagrams for Limpieza.
- Add planning docs under [plan/](plan/) and delivery notes under [entregas/](entregas/).
