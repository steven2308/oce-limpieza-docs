# Template rename — Castivas Cleaning

**Date:** 2026-10-09  
**Approach:** Full slug + branding pass

## Decisions

| Item | Value |
|------|--------|
| Technical slug | `cleaning` |
| Display name | `Castivas Cleaning` |
| Database / Postgres user (local) | `cleaning` |
| Production hosts | Leave placeholders / localhost |
| Theme | Fresh Teal — primary `#0D9488`, accent `#14B8A6`, destructive `#DC2626` |
| Favicon | Keep template icon; replace later |
| Backend envs | Copy `.envs.example/` → `.envs/` with slug applied |

## Scope

### Backend
- Rename package folder `limpieza/` → `cleaning/` and all `limpieza.*` imports
- Docker images / Traefik labels / media paths: `limpieza_*` → `cleaning_*`
- Display strings `Limpieza` → `Castivas Cleaning` (admin, emails, templates, docs)
- `.envs.example` and initialized `.envs`: DB/user/password/URL → `cleaning`
- `known_first_party` in `.editorconfig`

### Frontend
- `appName`, page title, package name → Castivas Cleaning / `cleaning-frontend`
- Docker / Makefile image names → `cleaning_*`
- Auth locale storage key → `cleaning-auth-locale`
- Vuetify light theme → Fresh Teal
- Icon left as-is (TODO in README)

### Out of scope
- Parent workspace folder name `limpieza/`
- Product docs under `docs/plan/` (Spanish domain language)
- Real logo asset, production domains

## Verification
- Grep for remaining `limpieza` / `Limpieza` under `back/` and `front/` (excluding plan docs)
- Backend `.envs/` exists with local + production copies
- Front theme tokens match Fresh Teal
