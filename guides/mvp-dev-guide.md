# Phase 1 (MVP) — Developer guide

How to run and extend the Castivas Cleaning MVP locally. Product scope: [plan/2. Requerimientos.md](../plan/2.%20Requerimientos.md) §6 MVP. Architecture sketch: [architecture/](../architecture/).

Repos: `back/` and `front/` sit next to this `docs/` folder in the monorepo layout.

---

## Stack

| Layer | Tech | Local URL |
|-------|------|-----------|
| API | Django + DRF (`cleaning` package) | http://localhost:8010 |
| SPA | Vue 3 + Vuetify + Pinia + Vite | http://localhost:8080 (`npm run dev`) |
| SPA (Docker) | `front/local.yml` | http://localhost:8180 |
| Postgres | Docker | host `5440` → container `5432` |
| Redis / Celery | Docker (optional for notifications) | Redis host `6380` |

Auth is **session + CSRF** (cookies), not bearer tokens for the SPA.

---

## Prerequisites

- Docker + Docker Compose
- Node **20** (nvm recommended) for the frontend
- Git

---

## First-time setup

### Backend

```bash
cd back
cp -r .envs.example/.local .envs/.local   # if .envs/ is missing
docker compose -f local.yml build
docker compose -f local.yml up -d postgres django
docker compose -f local.yml exec django python manage.py migrate
docker compose -f local.yml exec django python manage.py seed_mvp_demo
```

`seed_mvp_demo` requires `DEBUG=True` (local default) or `--force`. It is idempotent.

Optional full stack (Redis, Celery, scheduler):

```bash
docker compose -f local.yml up -d
```

Admin / Django shell:

```bash
docker compose -f local.yml exec django python manage.py createsuperuser
docker compose -f local.yml exec django python manage.py shell
```

### Frontend

```bash
cd front
cp .env.example .env   # VITE_API_URL=http://localhost:8010
# Node 20
npm ci
npm run dev            # http://localhost:8080
```

If login fails with CORS errors, confirm the SPA origin is listed in `back/config/settings/local.py` (`CORS_ALLOWED_ORIGINS`). Defaults include `:8080` and Playwright `:8090`.

---

## Demo users (seed)

Password for all: **`demo1234`**

| Email | Role | Group / profile |
|-------|------|-----------------|
| `ops@castivas.test` | Operations | `operations` |
| `maria@abcpm.test` | Client | contact of ABC Property Management |
| `ana@castivas.test` | Employee | active `EmployeeProfile` |
| `carlos@castivas.test` | Employee | active `EmployeeProfile` |

Demo tree: ABC Apartments → Building 5 → **Unit 215** (category B). Seed also creates Form. 1–style services, A/B/C categories, rates, and sample SRs.

---

## Repo map

### Backend (`back/`)

| Path | Role |
|------|------|
| `cleaning/ops/models/` | Catalog, parties, properties, SR, journey, complaints |
| `cleaning/ops/services/` | Rate resolve, assign, journey, scoping, complaints |
| `cleaning/ops/api/` | DRF serializers, permissions, viewsets |
| `cleaning/ops/management/commands/seed_mvp_demo.py` | Demo seed |
| `cleaning/ops/tests/` | Pytest suite for ops |
| `config/api_router.py` | All `/api/…` routes |
| `config/settings/local.py` | CORS, DEBUG, local DB |

Package rename: Python package is **`cleaning`** (not `limpieza`).

### Frontend (`front/`)

| Path | Role |
|------|------|
| `src/services/opsApi.ts` | Ops HTTP client |
| `src/utils/roles.ts` | `canAccess` / role helpers |
| `src/pages/ops/*` | Masters, queue, SR create/detail, journey, complaints |
| `src/i18n/opsCopy.ts` | ES/EN ops copy |
| `src/router/index.ts` | Routes + `meta.roles` |
| `e2e/` | Playwright happy path |

---

## Roles and permissions

Resolved in `/api/users/me/` as `role`: `operations` | `employee` | `client` (staff/superuser treated as operations-capable).

| Capability | Operations | Client | Employee |
|------------|:----------:|:------:|:--------:|
| Masters (categories, rates, companies, …) | ✓ | | |
| Ops queue + assign/close | ✓ | | |
| Create SR | ✓ | ✓ | |
| List SR (scoped) | all | own company | assigned to self |
| See client price | ✓ | ✓ | |
| See employee benefit | ✓ | | ✓ (own) |
| Journey actions | | | assigned only |
| Create complaint | ✓ | ✓ | |
| Hours CSV report | ✓ | | |

Router guard: `meta.roles` on Vue routes; API enforces the same rules (do not trust the UI alone).

---

## Main API surface (MVP)

Prefix: `/api/`. Session cookie required after `POST /api/auth/login/`.

| Resource | Notes |
|----------|--------|
| `service-requests/` | List/create/retrieve; actions: `assign/`, `close/`, journey (`accept/`, `reject/`, `en_route/`, `arrive/`, `clock_in/`, `break_start/`, `break_end/`, `clock_out/`, `complete/`) |
| `employees/` | Active employees for assignment picker |
| `service-types/`, `services/`, `property-categories/`, `rates/` | Catalog CRUD (ops) |
| `client-companies/`, `properties/`, `buildings/`, `units/` | Hierarchy |
| `complaints/` | Create / manage |
| `reports/hours.csv` | Ops export (`from`, `to` query params) |
| `users/me/` | Profile + `role` |
| `notifications/` | In-app notifications (assignment, etc.) |

Create SR snapshots `client_price` / `employee_benefit` from `resolve_rate`. Missing rate → 400. Employees cannot create SRs (403).

---

## Domain flow (happy path)

```text
Client/Ops creates SR (Rate snapshot)
    → Ops assigns employee (+ notify)
    → Employee accept → … → clock_in / breaks → clock_out → complete
    → Ops closes SR
    → Ops exports hours/amounts CSV
```

Reassign to a different employee writes a `reassigned` journey boundary so the new assignee can accept.

Statuses of interest: `pending_review` → `assigned` → journey → `completed` → `closed` (exact enum in `ServiceRequestStatus`).

---

## Day-to-day commands

```bash
# Backend tests (ops only — fast)
docker compose -f local.yml run --rm django pytest cleaning/ops -q

# Full backend suite
docker compose -f local.yml run --rm django pytest

# Migrations after model changes
docker compose -f local.yml run --rm django python manage.py makemigrations
docker compose -f local.yml run --rm django python manage.py migrate

# Frontend unit tests
cd front && npm test

# Frontend e2e (API up + seeded)
cd front && npm run test:e2e
```

**Docker tip:** migrations generated inside the container may be owned by `root`. Fix ownership before git operations:

```bash
docker compose -f local.yml run --rm --no-deps django \
  chown -R "$(id -u):$(id -g)" /app/cleaning/ops/migrations
```

---

## Adding an MVP feature (checklist)

1. Confirm the RF is in **fase 1** (not Fase 2/3) in [Requerimientos](../plan/2.%20Requerimientos.md).
2. Backend: failing pytest in `cleaning/ops/tests/` → model/service → API → green.
3. Frontend: Vitest for pure helpers; page wired through `opsApi` + `opsCopy` ES/EN.
4. Role scoping: assert other roles get 403/empty list as required.
5. Update seed only if demos need new fixtures; keep `seed_mvp_demo` idempotent.
6. Extend Playwright only for critical cross-role paths.

---

## Known MVP gaps / deferred

Do not spend Phase 1 effort here unless product reopens scope:

| Topic | Status |
|-------|--------|
| Native app / PWA / background geofence | Out (web + GPS on tap) |
| Smart employee suggestion (distance/load) | Fase 2 |
| Supervisor visits | Fase 2 |
| QuickBooks / inventory | Fase 3 |
| Ops dashboard counts (RF-REP-1) | Thin / incomplete vs RF |
| Rich SR edit before assign (RF-OPS-1) | Partial |
| Photo gallery limits UX | Journey supports evidence; polish may follow |
| Castivas `public/icon.png` | Still placeholder |

---

## Related docs

- [MVP testing guide](mvp-testing-guide.md)
- [MVP demo smoke checklist](../entregas/MVP-demo-checklist.md)
- [back/README.md](../../back/README.md) · [front/README.md](../../front/README.md)
- Implementation plan: [superpowers/plans/2026-10-09-mvp-castivas-cleaning.md](../superpowers/plans/2026-10-09-mvp-castivas-cleaning.md)
