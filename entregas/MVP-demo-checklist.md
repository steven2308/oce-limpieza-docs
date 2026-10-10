# MVP demo — manual smoke checklist

Casos QA completos de Fase 1: [guides/mvp-testing-guide.md](../guides/mvp-testing-guide.md). Setup de desarrollo: [guides/mvp-dev-guide.md](../guides/mvp-dev-guide.md).

Use after [seed](../../back/README.md#demo-data) and with backend + frontend running locally. Automated coverage: Playwright `front/e2e/mvp-happy-path.spec.ts`.

## Run stack + seed

```bash
# Terminal A — backend (from repo `back/`)
docker compose -f local.yml up -d postgres django
docker compose -f local.yml exec django python manage.py migrate
docker compose -f local.yml exec django python manage.py seed_mvp_demo

# Terminal B — frontend (from repo `front/`)
cp .env.example .env   # VITE_API_URL → http://localhost:8010
npm ci && npm run dev  # http://localhost:8080
```

## Seed credentials

All passwords: **`demo1234`**

| Email | Role |
|-------|------|
| `ops@castivas.test` | Operations |
| `maria@abcpm.test` | Client (owns ABC Apartments units) |
| `ana@castivas.test` | Employee (Ana García) |
| `carlos@castivas.test` | Employee |

Demo property: ABC Apartments → **Unit 215** (category B), owned by Maria López. Sample SRs exist after seed (pending, assigned to Ana, closed).

## Run Playwright e2e

From `front/` (backend up + seeded):

```bash
npm ci && npx playwright install chromium
npm run test:e2e
```

See [front/README.md — End-to-end tests](../../front/README.md#end-to-end-tests-playwright) for `E2E_BASE_URL`, `E2E_PORT`, and CORS notes.

---

## Smoke checks (Chrome)

Legend: **✅ Playwright 2026-10-09** = verified in Task 17 e2e; **⬜ verify locally** = manual only.

### Environment

- [ ] Seed completes without error (`seed_mvp_demo` idempotent on second run)
- [ ] Frontend loads login; API reachable (no CORS errors on login)

### 1. Operations — masters

- [ ] ⬜ Login `ops@castivas.test` → masters show property categories A/B/C and rates (tables load, regular×B and carpet band visible)

### 2. Client — create SR

- [x] ✅ **2026-10-09** Login `maria@abcpm.test` → create SR (unit 215 + service with rate) → SR number shown
- [x] ✅ **2026-10-09** Client **services table** lists own SRs only
- [ ] ⬜ On create form, **client price** visible before submit (confirm in browser; e2e asserts employee list hides `$120`, not client UI)

### 3. Operations — assign

- [x] ✅ **2026-10-09** Ops queue → open SR → assign **Ana** → save
- [ ] ⬜ Employee receives **in-app notification** for assignment (open notifications as Ana)

### 4. Employee — journey (mobile)

- [x] ✅ **2026-10-09** Login `ana@castivas.test` → assigned SR → accept → clock in → clock out → **upload ≥1 photo** → complete
- [ ] ⬜ Same flow at **phone width** (DevTools responsive); buttons usable without horizontal scroll
- [ ] ⬜ Complete stays blocked / errors without a photo

### 5. Operations — close + report

- [x] ✅ **2026-10-09** Ops closes SR after **completed** status
- [ ] ⬜ **Export hours/amounts CSV** via nav **Reports** (`/reports`) or API `GET /api/reports/hours.csv`
- [ ] ⬜ Ops **Queue** shows status / urgent / emergency counts
- [ ] ⬜ Ops can **edit** a pending SR (access notes / priority) before assign

### 6. i18n

- [ ] ⬜ Toggle **ES / EN** in shell; copy changes and **persists** after reload (`cleaning-auth-locale` in localStorage)

### Extra (e2e-only, optional manual spot-check)

- [x] ✅ **2026-10-09** `carlos@castivas.test` does **not** see Ana’s newly assigned SR
- [x] ✅ **2026-10-09** Maria sees SR **Closed** after ops close

---

## Sign-off (delivery)

| Date | Tester | Result | Notes |
|------|--------|--------|-------|
| | | | |

Paste completed rows into the PR or entrega note when handing off MVP.
