# Castivas Cleaning MVP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship the MVP from [`docs/plan/2. Requerimientos.md`](../../plan/2.%20Requerimientos.md): web platform where a Service Request (`SR`) is created with Rate snapshot, assigned, executed with evidence/hours, listed by role, and closed — without native apps.

**Architecture:** New Django app `cleaning.ops` holds domain models (catalog, properties, service requests, complaints) and DRF viewsets with **role-scoped querysets**. Vue SPA reuses existing auth/shell/i18n; adds role-based nav and mobile-first employee screens. Demo/browser flows run against a **management seed** command. Automated browser tests use **Playwright** against local Docker stack.

**Tech Stack:** Django 4.x + DRF + pytest (+ factory_boy), Vue 3 + Vuetify + Pinia + Vitest, Playwright for e2e, Docker Compose (`back/local.yml`, `front/local.yml`).

**Spec:** MVP bullets in `2. Requerimientos.md` §6 (not Fase 2/3). Estimate: `3. Estimación.md`.

## Global Constraints

- Package name is **`cleaning`** (not `limpieza`); display name **Castivas Cleaning**.
- TDD: red → green → refactor → commit for every task; no feature without a failing test first.
- Backend tests: `cd back && docker-compose -f local.yml run django pytest <path> -v`
- Frontend unit tests: `cd front && npm test`
- Browser/e2e: Playwright in `front/e2e` (Task 14); manual browser checks allowed as extra, not a substitute for Playwright.
- Roles via Django **Groups**: `operations` (admin ops), `employee`, `client`. Superuser/`is_staff` treated as operations for scoping.
- No native app / PWA deliverable; GPS only on clock-in/out tap.
- Employee suggestion algorithm = Fase 2 (MVP: plain employee list).
- QuickBooks / inventory = Fase 3.
- i18n ES + EN on all new UI strings.
- Do not commit unless the user asks (or the task step says commit and user approved commits for this plan).
- Currency: **USD** for Rate amounts unless Castivas says otherwise.

---

## File map (create unless noted)

```
back/cleaning/ops/
  apps.py
  models/
    __init__.py          # re-exports
    catalog.py           # ServiceType, Service, PropertyCategory, Rate
    parties.py           # ClientCompany, ClientContact, EmployeeProfile
    properties.py        # Property, Building, Unit
    service_requests.py  # ServiceRequest, JourneyEvent
    complaints.py        # Complaint
  services/
    rates.py             # resolve_rate(), apply_snapshot()
    scoping.py           # queryset filters by role
  api/
    serializers/*.py
    views/*.py
    urls.py              # optional; prefer config/api_router.py
  management/commands/
    seed_mvp_demo.py
  tests/
    factories.py
    test_catalog_models.py
    test_rates.py
    test_properties_api.py
    test_service_requests_api.py
    test_assignment.py
    test_journey.py
    test_complaints.py
    test_scoping_list.py
    test_reports.py
    test_seed_mvp_demo.py
  admin.py
  migrations/

back/config/settings/base.py   # INSTALLED_APPS += cleaning.ops
back/config/api_router.py      # register viewsets
back/cleaning/notifications/catalog.py  # new event types

front/src/
  services/opsApi.ts           # axios wrappers
  stores/ops.ts                # optional thin store
  pages/ops/
    ServicesListView.vue       # RF-REP-3
    ServiceRequestCreateView.vue
    ServiceRequestDetailView.vue
    Masters*.vue               # catalog + properties (can split)
    OpsQueueView.vue
    EmployeeJourneyView.vue
    ComplaintCreateView.vue
    ReportsView.vue
  router/index.ts              # routes + meta.roles
  i18n/opsCopy.ts              # ES/EN strings
  components/ops/...

front/e2e/
  playwright.config.ts
  global-setup.ts              # optional login helpers
  mvp-happy-path.spec.ts
```

---

### Task 1: Scaffold `cleaning.ops` + role groups helper

**Files:**
- Create: `back/cleaning/ops/apps.py`, `back/cleaning/ops/__init__.py`, `back/cleaning/ops/models/__init__.py`, `back/cleaning/ops/services/scoping.py`, `back/cleaning/ops/tests/test_roles.py`
- Modify: `back/config/settings/base.py` (`LOCAL_APPS`)
- Create: `back/cleaning/ops/migrations/0001_roles_noop.py` only if needed after first model migration in Task 2 — for this task only register the app.

**Interfaces:**
- Produces: `ROLE_OPERATIONS = "operations"`, `ROLE_EMPLOYEE = "employee"`, `ROLE_CLIENT = "client"`
- Produces: `def user_role(user) -> str` → one of those three (`operations` if superuser/staff or in operations group)
- Produces: `def ensure_role_groups()` creates Groups if missing

- [ ] **Step 1: Write the failing test**

```python
# back/cleaning/ops/tests/test_roles.py
import pytest
from django.contrib.auth.models import Group
from cleaning.ops.services.scoping import (
    ROLE_CLIENT,
    ROLE_EMPLOYEE,
    ROLE_OPERATIONS,
    ensure_role_groups,
    user_role,
)
from cleaning.users.tests.factories import UserFactory, AdminFactory


@pytest.mark.django_db
def test_ensure_role_groups_creates_three_groups():
    ensure_role_groups()
    names = set(Group.objects.values_list("name", flat=True))
    assert {ROLE_OPERATIONS, ROLE_EMPLOYEE, ROLE_CLIENT}.issubset(names)


@pytest.mark.django_db
def test_user_role_resolves_group_and_staff():
    ensure_role_groups()
    emp = UserFactory()
    emp.groups.add(Group.objects.get(name=ROLE_EMPLOYEE))
    assert user_role(emp) == ROLE_EMPLOYEE
    assert user_role(AdminFactory()) == ROLE_OPERATIONS
```

- [ ] **Step 2: Run test to verify it fails**

Run: `docker-compose -f local.yml run django pytest cleaning/ops/tests/test_roles.py -v`  
Expected: FAIL (module `cleaning.ops` missing or import error)

- [ ] **Step 3: Write minimal implementation**

```python
# back/cleaning/ops/services/scoping.py
ROLE_OPERATIONS = "operations"
ROLE_EMPLOYEE = "employee"
ROLE_CLIENT = "client"

def ensure_role_groups():
    from django.contrib.auth.models import Group
    for name in (ROLE_OPERATIONS, ROLE_EMPLOYEE, ROLE_CLIENT):
        Group.objects.get_or_create(name=name)

def user_role(user) -> str:
    if not user or not user.is_authenticated:
        raise ValueError("anonymous")
    if user.is_superuser or user.is_staff:
        return ROLE_OPERATIONS
    names = set(user.groups.values_list("name", flat=True))
    if ROLE_OPERATIONS in names:
        return ROLE_OPERATIONS
    if ROLE_EMPLOYEE in names:
        return ROLE_EMPLOYEE
    if ROLE_CLIENT in names:
        return ROLE_CLIENT
    return ROLE_CLIENT  # safest default for linked portal users
```

Register `cleaning.ops.apps.OpsConfig` in `LOCAL_APPS`.

- [ ] **Step 4: Run tests — PASS**

- [ ] **Step 5: Commit** (only if user asked for commits)

```bash
git add back/cleaning/ops back/config/settings/base.py
git commit -m "feat(ops): scaffold app and role group helpers"
```

---

### Task 2: Catalog models (ServiceType, Service, PropertyCategory, Rate)

**Files:**
- Create: `back/cleaning/ops/models/catalog.py`
- Modify: `back/cleaning/ops/models/__init__.py`
- Create: `back/cleaning/ops/tests/factories.py`, `back/cleaning/ops/tests/test_catalog_models.py`
- Create: migrations via `makemigrations`

**Interfaces:**
- Produces models:
  - `ServiceType(name, slug, is_active)`
  - `Service(service_type FK, name, slug, is_active, requires_notes: bool)`
  - `PropertyCategory(name, slug, min_area_sqft, max_area_sqft, min_baths, max_baths)` — nullable bounds
  - `Rate(service FK, property_category FK null=True, client_price Decimal, employee_benefit Decimal, currency default "USD")` unique_together `(service, property_category)`

- [ ] **Step 1: Failing model tests**

```python
# back/cleaning/ops/tests/test_catalog_models.py
import pytest
from decimal import Decimal
from django.db import IntegrityError
from cleaning.ops.models import ServiceType, Service, PropertyCategory, Rate


@pytest.mark.django_db
def test_rate_unique_per_service_and_category():
    st = ServiceType.objects.create(name="Regular Cleaning", slug="regular")
    svc = Service.objects.create(service_type=st, name="Regular std", slug="regular-std")
    cat = PropertyCategory.objects.create(name="B", slug="b", min_baths=2, max_baths=2)
    Rate.objects.create(service=svc, property_category=cat, client_price=Decimal("200"), employee_benefit=Decimal("80"))
    with pytest.raises(IntegrityError):
        Rate.objects.create(service=svc, property_category=cat, client_price=Decimal("1"), employee_benefit=Decimal("1"))


@pytest.mark.django_db
def test_rate_allows_null_category_for_carpet_bands():
    st = ServiceType.objects.create(name="Carpet Cleaning", slug="carpet")
    svc = Service.objects.create(service_type=st, name="Carpet <=200", slug="carpet-200")
    Rate.objects.create(service=svc, property_category=None, client_price=Decimal("90"), employee_benefit=Decimal("40"))
    assert Rate.objects.filter(service=svc, property_category__isnull=True).count() == 1
```

- [ ] **Step 2: Run — FAIL**

- [ ] **Step 3: Implement models + `makemigrations` + `migrate`**

- [ ] **Step 4: Run — PASS**

- [ ] **Step 5: Commit** if allowed: `feat(ops): add catalog and rate models`

---

### Task 3: Rate resolution service (snapshot helper)

**Files:**
- Create: `back/cleaning/ops/services/rates.py`
- Create: `back/cleaning/ops/tests/test_rates.py`

**Interfaces:**
- Consumes: `Service`, `PropertyCategory | None`, `Unit` (Task 4 — for now pass category explicitly)
- Produces:
  - `class RateSnapshot: client_price, employee_benefit, currency, rate_id, service_id, property_category_id`
  - `def resolve_rate(*, service: Service, property_category: PropertyCategory | None) -> Rate`
  - Raises `Rate.DoesNotExist` if missing

- [ ] **Step 1: Failing test**

```python
@pytest.mark.django_db
def test_resolve_rate_prefers_category_match():
    # setup type/service/cat/rates...
    from cleaning.ops.services.rates import resolve_rate
    rate = resolve_rate(service=svc, property_category=cat)
    assert rate.client_price == Decimal("200")
```

- [ ] **Step 2–4:** TDD implement `resolve_rate`
- [ ] **Step 5:** Commit `feat(ops): resolve_rate for category×service`

---

### Task 4: Parties + property hierarchy models

**Files:**
- Create: `back/cleaning/ops/models/parties.py`, `properties.py`
- Extend: `factories.py`, `test_properties_models.py`

**Interfaces:**
- `ClientCompany(name)`
- `ClientContact(company FK, user OneToOne nullable, name, email, phone)`
- `EmployeeProfile(user OneToOne, is_active)`
- `Property(company FK, name)`
- `Building(property FK, name)`
- `Unit(building FK, label, area_sqft, bedrooms, bathrooms, is_vacant, category FK null)`
- `def assign_category_for_unit(unit) -> PropertyCategory | None` — first matching band by area/baths

- [ ] **Step 1: Test unit category assignment by baths/area**

```python
@pytest.mark.django_db
def test_assign_category_for_unit_matches_bath_band():
    PropertyCategory.objects.create(name="B", slug="b", min_baths=2, max_baths=2, min_area_sqft=None, max_area_sqft=None)
    # create company→property→building→unit with bathrooms=2
    from cleaning.ops.services.rates import assign_category_for_unit
    cat = assign_category_for_unit(unit)
    assert cat.slug == "b"
    unit.refresh_from_db()
    assert unit.category_id == cat.id
```

- [ ] **Step 2–4:** Implement models + `assign_category_for_unit` (put in `services/rates.py` or `services/categories.py`)
- [ ] **Step 5:** Commit

---

### Task 5: ServiceRequest model + journey events

**Files:**
- Create: `back/cleaning/ops/models/service_requests.py`
- Tests: `test_service_request_models.py`

**Interfaces:**
- `ServiceRequest` fields: `number` (SR-000001), `status` (enum: `pending_review`, `assigned`, `in_progress`, `completed`, `closed`, `cancelled`), FKs: company, contact, unit, service, assigned_employee (User null), scheduled_date, scheduled_time, schedule_rigidity, access_mode, access_instructions, special_instructions, priority (`normal|urgent|emergency`), `client_price`, `employee_benefit`, `currency`, `rate_id` nullable, timestamps
- `JourneyEvent(service_request FK, kind, at, lat, lng, inside_geofence null, payload JSON)`
- Kinds: `accepted`, `rejected`, `en_route`, `arrived`, `clock_in`, `break_start`, `break_end`, `clock_out`, `completed`
- `def generate_sr_number() -> str`

- [ ] **Step 1: Test number generation and snapshot fields required on create helper**

```python
@pytest.mark.django_db
def test_generate_sr_number_increments():
    from cleaning.ops.models import generate_sr_number
    assert generate_sr_number() == "SR-000001"
    # create one SR with number SR-000001
    assert generate_sr_number() == "SR-000002"
```

- [ ] **Step 2–4:** Implement + migrate
- [ ] **Step 5:** Commit

---

### Task 6: Create SR API with Rate snapshot (RF-SOL-1, RF-SOL-5)

**Files:**
- Create: `back/cleaning/ops/api/serializers/service_requests.py`, `views/service_requests.py`
- Modify: `back/config/api_router.py`
- Tests: `test_service_requests_api.py`

**Interfaces:**
- `POST /api/service-requests/` body includes `unit`, `service`, schedule fields…  
- Server loads unit.category (or assigns), `resolve_rate`, fills snapshot; **400** if no Rate
- `GET /api/service-requests/` scoped (full scoping Task 9 — for now auth required)
- Produces serializer fields matching Form. 1 essentials

- [ ] **Step 1: Failing API test**

```python
@pytest.mark.django_db
def test_create_sr_snapshots_rate(api_client):
    ensure_role_groups()
    # build company, client user in client group, unit cat B, service+rate...
    api_client.force_authenticate(user=client_user)
    res = api_client.post("/api/service-requests/", {...}, format="json")
    assert res.status_code == 201
    assert res.data["number"].startswith("SR-")
    assert Decimal(res.data["client_price"]) == Decimal("200.00")
    assert Decimal(res.data["employee_benefit"]) == Decimal("80.00")


@pytest.mark.django_db
def test_create_sr_without_rate_returns_400(api_client):
    ...
    assert res.status_code == 400
```

- [ ] **Step 2:** FAIL  
- [ ] **Step 3:** ViewSet + serializer + router register `service-requests`  
- [ ] **Step 4:** PASS  
- [ ] **Step 5:** Commit `feat(ops): create service request with rate snapshot`

---

### Task 7: Catalog + properties CRUD APIs (masters)

**Files:**
- `api/views/catalog.py`, `properties.py`, serializers
- Tests: `test_catalog_api.py`, `test_properties_api.py`

**Interfaces:**
- ViewSets (operations-only write; client read own properties):  
  `service-types`, `services`, `property-categories`, `rates`, `client-companies`, `properties`, `buildings`, `units`
- Client `GET` properties filtered to `company` linked via `ClientContact`

- [ ] **Step 1:** Test operations can create rate; client cannot  
- [ ] **Step 2–4:** TDD implement  
- [ ] **Step 5:** Commit

---

### Task 8: Assignment + notification event (RF-OPS-3)

**Files:**
- Modify: service request viewset (`assign` action)
- Modify: `back/cleaning/notifications/catalog.py` — add `SERVICE_ASSIGNED`, `SERVICE_REJECTED`
- Tests: `test_assignment.py`

**Interfaces:**
- `POST /api/service-requests/{id}/assign/` `{ "employee_id": N, "internal_notes": "..." }`
- Sets `assigned_employee`, `status=assigned`, creates in-app notification for employee
- MVP: no suggestion algorithm — any active employee user

- [ ] **Step 1:**

```python
@pytest.mark.django_db
def test_assign_notifies_employee(api_client, settings):
    # ops user assigns emp
    res = api_client.post(f"/api/service-requests/{sr.id}/assign/", {"employee_id": emp.id})
    assert res.status_code == 200
    sr.refresh_from_db()
    assert sr.status == "assigned"
    assert sr.assigned_employee_id == emp.id
    from cleaning.notifications.models import Notification
    assert Notification.objects.filter(recipient=emp, event_type="service_assigned").exists()
```

- [ ] **Step 2–4:** Implement action + notification dispatch using existing notifications services  
- [ ] **Step 5:** Commit

---

### Task 9: Role-scoped services list (RF-REP-3) + column visibility

**Files:**
- `services/scoping.py` — `def service_requests_for(user) -> QuerySet`
- Serializer: omit `client_price` / `employee_benefit` by role
- Tests: `test_scoping_list.py`

**Interfaces:**
- Operations: all SRs  
- Employee: `assigned_employee=user`  
- Client: `unit__building__property__company` in user’s companies  
- List includes past and future (`scheduled_date` any)

- [ ] **Step 1:**

```python
@pytest.mark.django_db
def test_employee_list_hides_others_and_client_price(api_client):
    ...
    res = api_client.get("/api/service-requests/")
    ids = {row["id"] for row in res.data["results"]}
    assert sr_mine.id in ids and sr_other.id not in ids
    assert "employee_benefit" in res.data["results"][0]
    assert "client_price" not in res.data["results"][0]


@pytest.mark.django_db
def test_client_list_only_own_properties(api_client):
    ...
```

- [ ] **Step 2–4:** Implement scoping + serializer field dropping  
- [ ] **Step 5:** Commit `feat(ops): role-scoped service request list`

---

### Task 10: Employee journey API (RF-EMP-*)

**Files:**
- Actions on ServiceRequestViewSet: `accept`, `reject`, `en_route`, `arrive`, `clock_in`, `break_start`, `break_end`, `clock_out`, `complete`
- Tests: `test_journey.py`

**Interfaces:**
- Each action appends `JourneyEvent` with optional `lat`, `lng`; compute `inside_geofence` if unit/property has lat/lng+radius (if no coordinates on unit yet, store GPS and `inside_geofence=null`)
- `clock_out` / `complete` set status `completed` when `completed=true`
- Net duration helper excludes breaks (for reports Task 12)

- [ ] **Step 1:** Test accept → clock_in → clock_out creates events and duration  
- [ ] **Step 2–4:** TDD  
- [ ] **Step 5:** Commit

---

### Task 11: Complaints API (RF-QUE-*)

**Files:**
- `models/complaints.py`, API, `test_complaints.py`

**Interfaces:**
- `Complaint`: company, reporter contact, unit, optional `service_request`, type, description, priority, notify_on_resolve, status
- Client creates; operations closes + notification if flag

- [ ] **Step 1–5:** TDD CRUD + close notify

---

### Task 12: Ops close + hours/amounts report CSV (RF-OPS-5, RF-REP-2)

**Files:**
- Action `close` on SR (only from `completed`)
- `GET /api/reports/hours.csv?from=&to=`
- Tests: `test_reports.py`

**Interfaces:**
- CSV columns: `number, employee, unit, service, hours_net, client_price, employee_benefit, status`
- Operations-only

- [ ] **Step 1–5:** TDD

---

### Task 13: Seed command `seed_mvp_demo`

**Files:**
- Create: `back/cleaning/ops/management/commands/seed_mvp_demo.py`
- Test: `test_seed_mvp_demo.py`

**Interfaces:**
- `python manage.py seed_mvp_demo` (idempotent via fixed usernames)
- Creates:
  - Groups via `ensure_role_groups()`
  - Users (password `demo1234` for all):
    - `ops@castivas.test` → operations  
    - `ana@castivas.test` → employee (Ana García)  
    - `carlos@castivas.test` → employee  
    - `maria@abcpm.test` → client (Maria López)  
  - `ClientCompany` ABC Property Management + contact linked to maria  
  - Property ABC Apartments → Building 5 → Unit 215 (3 bed, 2 bath, category B)  
  - ServiceTypes/Services seed from Form. 1 + carpet band service  
  - PropertyCategories A/B/C  
  - Rates for regular×B and carpet null-category  
  - One `pending_review` SR and one `assigned` SR for Ana (future date) and one `closed` historical SR  

- [ ] **Step 1:**

```python
@pytest.mark.django_db
def test_seed_mvp_demo_is_idempotent():
    from django.core.management import call_command
    call_command("seed_mvp_demo")
    call_command("seed_mvp_demo")
    from cleaning.users.models import User
    assert User.objects.filter(username="ops@castivas.test").count() == 1
    from cleaning.ops.models import ServiceRequest, Unit
    assert Unit.objects.filter(label="215").exists()
    assert ServiceRequest.objects.count() >= 3
```

- [ ] **Step 2–4:** Implement command  
- [ ] **Step 5:** Commit `feat(ops): seed_mvp_demo for local and e2e`

Document in `back/README.md`:

```bash
docker-compose -f local.yml run django python manage.py seed_mvp_demo
```

---

### Task 14: Frontend foundation — API client, roles, nav, i18n

**Files:**
- `front/src/services/opsApi.ts`
- `front/src/i18n/opsCopy.ts` (es/en)
- Modify: `front/src/router/index.ts`, `front/src/App.vue` nav
- Vitest: `front/src/services/opsApi.spec.ts` for URL builders / price field stripping helpers if any pure fns

**Interfaces:**
- Routes with `meta.roles: ['operations'|'employee'|'client']`
- Nav items filtered by `user_role` from `/users/me/` (extend backend User serializer to include `role` and `groups` if not already — **TDD on backend** small test in this task if missing)

- [ ] **Step 1:** Backend test `test_me_includes_role` if needed → add `role` on User serializer  
- [ ] **Step 2:** Vitest for route meta helper `canAccess(role, metaRoles)`  
- [ ] **Step 3:** Implement nav + router guards  
- [ ] **Step 4:** PASS  
- [ ] **Step 5:** Commit

---

### Task 15: Frontend — masters + create SR + services table

**Files:**
- Pages under `front/src/pages/ops/`
- Vitest for any pure formatters (SR number, money)

**Flows:**
- Operations: CRUD-ish masters (categories, services, rates) — tables + dialogs sufficient  
- Client: create SR (select unit, service, see price), services table  
- All roles: `ServicesListView` using GET `/api/service-requests/`

- [ ] **Step 1:** Component/unit tests where logic is pure; otherwise mark Playwright as gate (Task 17)  
- [ ] **Step 2–4:** Implement UI wired to API  
- [ ] **Step 5:** Commit

---

### Task 16: Frontend — ops assign/close + employee journey + complaints

**Files:**
- `OpsQueueView.vue`, `EmployeeJourneyView.vue`, `ComplaintCreateView.vue`, detail pages

**Flows:**
- Ops: open pending SR → assign Ana → close when completed  
- Employee: accept → clock in (mock geolocation in browser tests) → photos optional → complete  
- Client: create complaint linked to SR  

- [ ] **Step 1–5:** Implement against API; prefer thin views; commit

---

### Task 17: Playwright e2e (browser testing)

**Files:**
- Create: `front/e2e/playwright.config.ts`, `front/e2e/mvp-happy-path.spec.ts`
- Modify: `front/package.json` scripts `"test:e2e": "playwright test"`
- Document prerequisites in `front/README.md`

**Prerequisites (run before e2e):**
```bash
# terminal A
cd back && docker-compose -f local.yml up
# migrate + seed
docker-compose -f local.yml run django python manage.py migrate
docker-compose -f local.yml run django python manage.py seed_mvp_demo
# terminal B
cd front && npm ci && npm run dev
```

**Interfaces / credentials from seed:**
- ops@castivas.test / demo1234  
- ana@castivas.test / demo1234  
- maria@abcpm.test / demo1234  

- [ ] **Step 1: Write failing e2e spec**

```typescript
// front/e2e/mvp-happy-path.spec.ts
import { test, expect } from "@playwright/test";

const BASE = process.env.E2E_BASE_URL ?? "http://localhost:8080";

test("client creates SR, ops assigns, employee completes, all see scoped lists", async ({ browser }) => {
  const client = await browser.newPage();
  await client.goto(`${BASE}/login`);
  await client.getByLabel(/email|correo/i).fill("maria@abcpm.test");
  await client.getByLabel(/password|contraseña/i).fill("demo1234");
  await client.getByRole("button", { name: /log in|ingresar/i }).click();
  await client.goto(`${BASE}/services/new`);
  // select unit 215 + a service with rate — adapt selectors to implemented UI
  await client.getByRole("button", { name: /send|enviar/i }).click();
  await expect(client.getByText(/SR-/i)).toBeVisible();

  const ops = await browser.newPage();
  await ops.goto(`${BASE}/login`);
  await ops.getByLabel(/email|correo/i).fill("ops@castivas.test");
  await ops.getByLabel(/password|contraseña/i).fill("demo1234");
  await ops.getByRole("button", { name: /log in|ingresar/i }).click();
  await ops.goto(`${BASE}/ops/queue`);
  await ops.getByText(/SR-/i).first().click();
  await ops.getByRole("button", { name: /assign|asignar/i }).click();
  await ops.getByText(/Ana/i).click();
  await ops.getByRole("button", { name: /save|guardar|notify/i }).click();

  const emp = await browser.newPage();
  await emp.goto(`${BASE}/login`);
  await emp.getByLabel(/email|correo/i).fill("ana@castivas.test");
  await emp.getByLabel(/password|contraseña/i).fill("demo1234");
  await emp.getByRole("button", { name: /log in|ingresar/i }).click();
  await emp.goto(`${BASE}/employee/services`);
  await expect(emp.getByText(/SR-/i)).toBeVisible();
  await emp.getByText(/SR-/i).first().click();
  await emp.context().grantPermissions(["geolocation"]);
  await emp.getByRole("button", { name: /accept|aceptar/i }).click();
  await emp.getByRole("button", { name: /clock in/i }).click();
  await emp.getByRole("button", { name: /clock out/i }).click();
  await emp.getByRole("button", { name: /complete|finalizar/i }).click();

  await ops.goto(`${BASE}/services`);
  await expect(ops.getByText(/completed|completado|closed|cerrado/i)).toBeVisible();

  await client.goto(`${BASE}/services`);
  await expect(client.getByText(/SR-/i)).toBeVisible();
  // client must not see another company's data — seed only has ABC
});
```

- [ ] **Step 2: Install Playwright and run — FAIL until UI ready**

```bash
cd front && npm init playwright@latest
# or: npx playwright install
npm run test:e2e
```

Expected first run: FAIL on missing selectors/routes — fix UI or selectors until green.

- [ ] **Step 3: Stabilize selectors** (`data-testid` on key buttons: `sr-submit`, `assign-save`, `journey-clock-in`, etc.)

- [ ] **Step 4: Run e2e — PASS**

- [ ] **Step 5:** Commit `test(e2e): mvp happy path with seeded users`

**Additional e2e (same task or follow-up):**
- `employee cannot see other employee SR` (create second assigned SR to Carlos in seed; login Ana; assert Carlos’s number hidden)
- `client price hidden from employee list` (assert no `$` price cell / no `client_price` in row)

---

### Task 18: Manual browser smoke + README checklist

**Files:**
- Modify: `docs/plan/README.md` or `front/README.md` + `back/README.md` with MVP demo script

- [ ] **Step 1: Run seed + both servers**
- [ ] **Step 2: Manually verify in browser (Chrome):**
  1. Login ops → masters show categories/rates  
  2. Login maria → create SR → see price → services table  
  3. Login ops → assign Ana → notification  
  4. Login Ana → journey on phone width (devtools)  
  5. Login ops → close → export CSV  
  6. Language toggle ES/EN persists  
- [ ] **Step 3: Paste checklist results into PR/entrega note under `docs/entregas/` when delivering**
- [ ] **Step 4: No new code unless bugs found — file fixes as hotfix tasks with TDD**

---

## Spec coverage checklist (self-review)

| MVP requirement | Task(s) |
|-----------------|--------|
| Service type / Service / Property category / Rate | 2, 3, 7, 13 |
| Snapshot price/benefit on SR | 3, 6 |
| Client + Ops create SR | 6, 15 |
| Assign + notify | 8, 16 |
| Employee journey + GPS tap | 10, 16, 17 |
| Complaints | 11, 16 |
| Services table by role | 9, 15, 17 |
| Hours + amounts CSV | 12 |
| i18n ES/EN | 14 |
| Seed data | 13 |
| Browser/e2e tests | 17, 18 |
| No native/PWA / no geofence daemon | Global constraints |
| No employee suggestion algo | Explicitly out (Fase 2) |

## Out of this plan (do not implement)

- Fase 2: suggestions, map, supervision, feedback stars  
- Fase 3: QuickBooks, inventory  
- Payment gateway, Uber claim, PWA packaging  

---

## Execution handoff

Plan saved to `docs/superpowers/plans/2026-10-09-mvp-castivas-cleaning.md`.

**Two execution options:**

1. **Subagent-Driven (recommended)** — fresh subagent per task, review between tasks  
2. **Inline Execution** — execute tasks in this session with executing-plans checkpoints  

Which approach?
