# Phase 1 (MVP) — Testing guide

**Status:** ready to run  
**Audience:** QA / developer verifying Phase 1 before client demo or UAT  
**Scope source:** [plan/2. Requerimientos.md](../plan/2.%20Requerimientos.md) §6 MVP  

Short live-demo path: [entregas/MVP-demo-checklist.md](../entregas/MVP-demo-checklist.md).  
Environment setup: [mvp-dev-guide.md](mvp-dev-guide.md).

---

## Scope

This guide must prove:

- Client (or Ops) can create an SR with Rate snapshot and an `SR-######` number.
- Operations can assign/reassign, close after completion, and export hours/amounts CSV.
- Employee can run the journey (accept → clock in/out → complete) on assigned jobs only.
- Each role’s services table is scoped (no cross-tenant / cross-assignee leakage).
- Client price vs employee benefit visibility matches role rules.
- ES/EN toggle works for ops UI copy.
- Basic complaint create works for client/ops.

**Out of this phase (do not file as MVP bugs):**

- Native app, PWA install, background geofencing  
- Smart employee suggestions / map of GPS trail (Fase 2)  
- Supervisor visits (Fase 2)  
- QuickBooks / inventory (Fase 3)  
- Payment gateway / payroll bank deposit  

---

## Environment

| App | URL |
|-----|-----|
| Frontend (Vite) | http://localhost:8080 |
| API | http://localhost:8010 |
| Admin (optional) | http://localhost:8010/admin/ |

**Credentials** (password `demo1234`):

| Email | Role |
|-------|------|
| `ops@castivas.test` | Operations |
| `maria@abcpm.test` | Client (ABC Property Management) |
| `ana@castivas.test` | Employee |
| `carlos@castivas.test` | Employee |

Demo unit for create: **ABC Apartments / Building 5 / Unit 215**.

### Preconditions (stop the run if false)

- [ ] `docker compose -f local.yml up -d postgres django` in `back/` succeeds
- [ ] Migrations applied: `python manage.py migrate`
- [ ] Seed applied: `python manage.py seed_mvp_demo` (no error; safe to re-run)
- [ ] Frontend `.env` has `VITE_API_URL=http://localhost:8010`
- [ ] Browser can log in without CORS failures
- [ ] Chrome (or Chromium) available; for mobile checks, DevTools responsive mode is enough

### How to record

- **Pass** — expected result holds.
- **Fail** — note step #, actual vs expected, screenshot if UI. Do not continue a chain that depended on that fail.
- Mark **Blocked** if a precondition failed.

---

## Automated suite

Run these first; they catch most regressions.

### Backend (pytest)

```bash
cd back
docker compose -f local.yml run --rm django pytest cleaning/ops -q
# Expected: 109+ passed (count may grow)
```

Full suite:

```bash
docker compose -f local.yml run --rm django pytest
```

### Frontend unit (Vitest)

```bash
cd front
npm test
# Expected: all files green (roles, opsApi, opsCopy, …)
```

### End-to-end (Playwright)

Backend must be up and seeded. Playwright starts Vite on **:8090** (CORS-whitelisted).

```bash
cd front
npm ci
npx playwright install chromium
npm run test:e2e
```

Optional against an already running SPA:

```bash
E2E_BASE_URL=http://localhost:8080 npm run test:e2e
```

Covers: client creates SR → ops assigns Ana → journey → scoped lists → employee never sees client price / other jobs.

| Suite | Command | Result (date / tester) |
|-------|---------|------------------------|
| pytest ops | `… pytest cleaning/ops` | |
| vitest | `npm test` | |
| playwright | `npm run test:e2e` | |

---

## Happy path (manual)

Use a clean browser profile or private window. Prefer creating a **new** SR so seed samples are not required.

### A. Client creates SR

1. Login as `maria@abcpm.test`.
2. Open **Services** → create new request.
3. Choose Unit **215**, an active service that has a rate (e.g. regular clean for category B).
4. Fill schedule / access fields as required; submit.

**Expect:** SR number `SR-######` shown; client **price** visible on create/detail; Maria’s services list includes the new row.

- [ ] Pass / Fail — ________

### B. Operations assigns

1. Login as `ops@castivas.test`.
2. Open **Queue** (or services list) → open the new SR.
3. Assign **Ana García** (`ana@castivas.test`); save.

**Expect:** Status assigned; Ana selectable from employees list.

- [ ] Pass / Fail — ________

### C. Assignment notification

1. Login as `ana@castivas.test`.
2. Open in-app notifications.

**Expect:** Notification for the new assignment (RF-PLAT-3).

- [ ] Pass / Fail — ________

### D. Employee journey

1. As Ana, open the assigned SR → **Journey**.
2. **Accept** → **Clock in** (allow geolocation if prompted) → **Clock out** → **Complete**.

**Expect:** Each step succeeds; status progresses to completed. Optional: repeat at phone width (~390px) — controls usable without horizontal scroll.

- [ ] Pass / Fail — ________  
- [ ] Pass / Fail (mobile width) — ________

### E. Operations closes + CSV

1. As Ops, open the completed SR → **Close**.
2. Download hours/amounts report (`Reports` UI or `GET /api/reports/hours.csv?from=YYYY-MM-DD&to=YYYY-MM-DD`).

**Expect:** SR closed; CSV downloads with hours and snapshot amounts (client price / employee benefit columns present).

- [ ] Pass / Fail (close) — ________  
- [ ] Pass / Fail (CSV) — ________

### F. i18n

1. As any role, toggle **ES / EN** in the shell.
2. Reload the page.

**Expect:** Ops/auth copy language changes and persists (`localStorage` key `cleaning-auth-locale`).

- [ ] Pass / Fail — ________

---

## Permissions / isolation

| # | Case | Steps | Expect | Result |
|---|------|-------|--------|--------|
| P1 | Employee cannot create SR | Login Ana → open `/services/new` or POST create | UI blocked / API 403 | |
| P2 | Employee A ≠ B jobs | Login `carlos@castivas.test` after Ana was assigned a new SR | Carlos does **not** see Ana’s SR | |
| P3 | Employee hides client price | As Ana on an assigned SR | No client `$` price; benefit may show | |
| P4 | Client sees only own company | As Maria, services list | Only ABC rows; no other companies | |
| P5 | Client cannot assign/close | As Maria on an SR detail | No assign/close actions (or API 403) | |
| P6 | Ops sees queue | As Ops | Pending/assigned SRs visible across clients | |
| P7 | Masters ops-only | As Maria or Ana open `/masters` | Redirect / denied | |
| P8 | Reassign journey | Ops assigns Ana → Ana accepts → Ops reassigns to Carlos | Carlos can accept; Ana cannot continue old journey | |

---

## Edges and known gaps

| Case | Phase 1 behavior | Follow-up |
|------|------------------|-----------|
| Service with no Rate | Create blocked (400 / validation) | Ops maintains Rates |
| `requires_notes` service | Blank special instructions → 400 | — |
| Inactive service | Cannot select / 400 | — |
| GPS denied by browser | Journey still records; `inside_geofence` may be null | Property lat/radius data quality |
| Seed without DEBUG | Command errors unless `--force` | Never `--force` on shared prod |
| Ops dashboard KPIs (RF-REP-1) | May be thin vs RF | Enhance later |
| Photo UX polish (RF-EMP-4) | Basic evidence path; do not block on gallery polish | Fase 1.x |
| Employee suggestion (RF-OPS-2) | Manual picker only | Fase 2 |

---

## Complaints (smoke)

1. Login as Maria → **New complaint** linked to unit 215 (optional SR).
2. Submit.

**Expect:** Complaint created; Ops can see/manage it (list or detail per current UI).

- [ ] Pass / Fail — ________

---

## Run summary

| Area | Pass | Fail | Blocked |
|------|-----:|-----:|--------:|
| Automated (pytest / vitest / e2e) | | | |
| Happy path A–F | | | |
| Permissions P1–P8 | | | |
| Complaints | | | |

| Date | Tester | Build / commit (back · front) | Overall |
|------|--------|-------------------------------|---------|
| | | | Pass / Fail |

Paste this table into the entrega note or PR when handing off Phase 1.
