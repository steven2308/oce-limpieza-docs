# Fase 1 (MVP) — Guía de desarrollo

Cómo levantar y extender el MVP de Castivas Cleaning en local. Alcance de producto: [plan/2. Requerimientos.md](../plan/2.%20Requerimientos.md) §6 MVP. Esbozo de arquitectura: [architecture/](../architecture/).

Repos: `back/` y `front/` están al lado de esta carpeta `docs/` en el monorepo.

---

## Stack

| Capa | Tecnología | URL local |
|------|------------|-----------|
| API | Django + DRF (paquete `cleaning`) | http://localhost:8010 |
| SPA | Vue 3 + Vuetify + Pinia + Vite | http://localhost:8080 (`npm run dev`) |
| SPA (Docker) | `front/local.yml` | http://localhost:8180 |
| Postgres | Docker | host `5440` → contenedor `5432` |
| Redis / Celery | Docker (opcional para notificaciones) | Redis host `6380` |

La autenticación del SPA es **sesión + CSRF** (cookies), no tokens bearer.

---

## Requisitos previos

- Docker + Docker Compose
- Node **20** (se recomienda nvm) para el frontend
- Git

---

## Puesta en marcha (primera vez)

### Backend

```bash
cd back
cp -r .envs.example/.local .envs/.local   # si falta .envs/
docker compose -f local.yml build
docker compose -f local.yml up -d postgres django
docker compose -f local.yml exec django python manage.py migrate
docker compose -f local.yml exec django python manage.py seed_mvp_demo
```

`seed_mvp_demo` exige `DEBUG=True` (por defecto en local) o `--force`. Es idempotente.

Stack completo opcional (Redis, Celery, scheduler):

```bash
docker compose -f local.yml up -d
```

Admin / shell de Django:

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

Si el login falla por CORS, confirma que el origen del SPA está en `back/config/settings/local.py` (`CORS_ALLOWED_ORIGINS`). Por defecto incluye `:8080`, Docker Compose `:8180` y Playwright `:8090`.

---

## Usuarios demo (seed)

Contraseña para todos: **`demo1234`**

| Email | Rol | Grupo / perfil |
|-------|-----|----------------|
| `ops@castivas.test` | Operations | `operations` |
| `maria@abcpm.test` | Cliente | contacto de ABC Property Management |
| `ana@castivas.test` | Empleado | `EmployeeProfile` activo |
| `carlos@castivas.test` | Empleado | `EmployeeProfile` activo |

Árbol demo: ABC Apartments (215 West 95th St, NY 10025 + geo) → Building 5 → **Unit 215** (categoría B). Captura unificada: `POST /api/properties/capture/` reutiliza propiedad por dirección+ZIP. El seed también crea servicios Form. 1, categorías A/B/C, Rates A/B/C y SRs de ejemplo.

---

## Mapa del repositorio

### Backend (`back/`)

| Ruta | Rol |
|------|-----|
| `cleaning/ops/models/` | Catálogo, parties, predios, SR, jornada, quejas |
| `cleaning/ops/services/` | Resolución de Rate, asignación, jornada, scoping, quejas |
| `cleaning/ops/api/` | Serializers DRF, permisos, viewsets |
| `cleaning/ops/management/commands/seed_mvp_demo.py` | Seed demo |
| `cleaning/ops/tests/` | Suite pytest de ops |
| `config/api_router.py` | Todas las rutas `/api/…` |
| `config/settings/local.py` | CORS, DEBUG, DB local |

El paquete Python se llama **`cleaning`** (ya no `limpieza`).

### Frontend (`front/`)

| Ruta | Rol |
|------|-----|
| `src/services/opsApi.ts` | Cliente HTTP de ops |
| `src/utils/roles.ts` | `canAccess` / helpers de rol |
| `src/pages/ops/*` | Maestros, cola, alta/detalle SR, jornada, quejas |
| `src/i18n/opsCopy.ts` | Copy ops ES/EN |
| `src/router/index.ts` | Rutas + `meta.roles` |
| `e2e/` | Happy path Playwright |

---

## Roles y permisos

Se resuelven en `/api/users/me/` como `role`: `operations` | `employee` | `client` (staff/superuser se tratan como capaces de operations).

| Capacidad | Operations | Cliente | Empleado |
|-----------|:----------:|:-------:|:--------:|
| Maestros (categorías, rates, empresas, …) | ✓ | | |
| Cola ops + asignar/cerrar | ✓ | | |
| Crear SR | ✓ | ✓ | |
| Listar SR (acotado) | todos | su empresa | asignados a sí |
| Ver precio cliente | ✓ | ✓ | |
| Ver beneficio empleado | ✓ | | ✓ (propio) |
| Acciones de jornada | | | solo asignado |
| Crear queja | ✓ | ✓ | |
| Reporte CSV de horas | ✓ | | |

Guard del router: `meta.roles` en rutas Vue; la API aplica las mismas reglas (no confiar solo en la UI).

---

## Superficie API principal (MVP)

Prefijo: `/api/`. Cookie de sesión tras `POST /api/auth/login/`.

| Recurso | Notas |
|---------|--------|
| `service-requests/` | Listar/crear/detalle; acciones: `assign/`, `close/`, jornada (`accept/`, `reject/`, `en_route/`, `arrive/`, `clock_in/`, `break_start/`, `break_end/`, `clock_out/`, `complete/`) |
| `employees/` | Empleados activos para el picker de asignación |
| `service-types/`, `services/`, `property-categories/`, `rates/` | CRUD de catálogo (ops) |
| `client-companies/`, `properties/`, `buildings/`, `units/` | Jerarquía |
| `complaints/` | Crear / gestionar |
| `reports/hours.csv` | Export ops (query `from`, `to`) |
| `users/me/` | Perfil + `role` |
| `notifications/` | Notificaciones in-app (asignación, etc.) |

Al crear un SR se hace snapshot de `client_price` / `employee_benefit` vía `resolve_rate`. Sin Rate → 400. Los empleados no pueden crear SRs (403).

---

## Flujo de dominio (happy path)

```text
Cliente/Ops crea SR (snapshot de Rate)
    → Ops asigna empleado (+ notifica)
    → Empleado accept → … → clock_in / breaks → clock_out → complete
    → Ops cierra el SR
    → Ops exporta CSV de horas/montos
```

Reasignar a otro empleado escribe un límite de jornada `reassigned` para que el nuevo asignado pueda aceptar.

Estados relevantes: `pending_review` → `assigned` → jornada → `completed` → `closed` (enum exacto en `ServiceRequestStatus`).

---

## Comandos del día a día

```bash
# Tests backend (solo ops — rápido)
docker compose -f local.yml run --rm django pytest cleaning/ops -q

# Suite backend completa
docker compose -f local.yml run --rm django pytest

# Migraciones tras cambios de modelo
docker compose -f local.yml run --rm django python manage.py makemigrations
docker compose -f local.yml run --rm django python manage.py migrate

# Tests unitarios frontend
cd front && npm test

# e2e frontend (API arriba + seed)
cd front && npm run test:e2e
```

**Tip Docker:** las migraciones generadas dentro del contenedor pueden quedar con dueño `root`. Corrige la propiedad antes de operaciones git:

```bash
docker compose -f local.yml run --rm --no-deps django \
  chown -R "$(id -u):$(id -g)" /app/cleaning/ops/migrations
```

---

## Añadir una feature del MVP (checklist)

1. Confirma que el RF está en **fase 1** (no Fase 2/3) en [Requerimientos](../plan/2.%20Requerimientos.md).
2. Backend: pytest en rojo en `cleaning/ops/tests/` → modelo/servicio → API → verde.
3. Frontend: Vitest para helpers puros; pantalla cableada con `opsApi` + `opsCopy` ES/EN.
4. Scoping por rol: otros roles deben recibir 403 / lista vacía según corresponda.
5. Actualiza el seed solo si el demo necesita fixtures nuevos; mantén `seed_mvp_demo` idempotente.
6. Extiende Playwright solo en caminos críticos entre roles.

---

## Huecos conocidos / diferidos del MVP

No inviertas esfuerzo de Fase 1 aquí salvo que producto reabra el alcance:

| Tema | Estado |
|------|--------|
| App nativa / PWA / geofence en segundo plano | Fuera (web + GPS al abrir) |
| Sugerencia inteligente de empleado (distancia/carga) | Fase 2 |
| Visitas de supervisor | Fase 2 |
| QuickBooks / inventario | Fase 3 |
| Contadores del dashboard ops (RF-REP-1) | Delgado / incompleto vs RF |
| Edición rica del SR antes de asignar (RF-OPS-1) | Parcial |
| UX de galería de fotos | La jornada admite evidencia; el pulido puede seguir |
| Logo final de Castivas | Favicon SVG teal temporal (`public/favicon.svg`) |

---

## Documentos relacionados

- [Guía de pruebas MVP](mvp-testing-guide.md)
- [Checklist smoke de demo MVP](../entregas/MVP-demo-checklist.md)
- [back/README.md](../../back/README.md) · [front/README.md](../../front/README.md)
- Plan de implementación: [superpowers/plans/2026-10-09-mvp-castivas-cleaning.md](../superpowers/plans/2026-10-09-mvp-castivas-cleaning.md)
