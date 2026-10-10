# Fase 1 (MVP) — Guía de pruebas

**Estado:** lista para ejecutar  
**Audiencia:** QA / desarrollador que verifica la Fase 1 antes de demo con cliente o UAT  
**Fuente de alcance:** [plan/2. Requerimientos.md](../plan/2.%20Requerimientos.md) §6 MVP  

Ruta corta de demo en vivo: [entregas/MVP-demo-checklist.md](../entregas/MVP-demo-checklist.md).  
Entorno: [mvp-dev-guide.md](mvp-dev-guide.md).

---

## Alcance

Esta guía debe demostrar:

- El cliente (u Ops) puede crear un SR con snapshot de Rate y número `SR-######`.
- Operations puede asignar/reasignar, cerrar tras completar y exportar CSV de horas/montos.
- El empleado puede ejecutar la jornada (aceptar → clock in/out → completar) solo en trabajos asignados.
- La tabla de servicios de cada rol está acotada (sin fugas entre empresas ni entre asignados).
- La visibilidad de precio cliente vs beneficio empleado cumple las reglas por rol.
- El toggle ES/EN funciona en el copy de ops.
- La creación básica de queja funciona para cliente/ops.

**Fuera de esta fase (no reportar como bugs del MVP):**

- App nativa, instalación PWA, geofencing en segundo plano  
- Sugerencias inteligentes de empleado / mapa de rastro GPS (Fase 2)  
- Visitas de supervisor (Fase 2)  
- QuickBooks / inventario (Fase 3)  
- Pasarela de pago / depósito bancario de nómina  

---

## Entorno

| App | URL |
|-----|-----|
| Frontend (Vite) | http://localhost:8080 |
| API | http://localhost:8010 |
| Admin (opcional) | http://localhost:8010/admin/ |

**Credenciales** (contraseña `demo1234`):

| Email | Rol |
|-------|-----|
| `ops@castivas.test` | Operations |
| `maria@abcpm.test` | Cliente (ABC Property Management) |
| `ana@castivas.test` | Empleado |
| `carlos@castivas.test` | Empleado |

Unidad demo para crear: **ABC Apartments / Building 5 / Unit 215**.

### Precondiciones (detener la corrida si alguna falla)

- [ ] `docker compose -f local.yml up -d postgres django` en `back/` termina bien
- [ ] Migraciones aplicadas: `python manage.py migrate`
- [ ] Seed aplicado: `python manage.py seed_mvp_demo` (sin error; se puede re-ejecutar)
- [ ] El `.env` del frontend tiene `VITE_API_URL=http://localhost:8010`
- [ ] El navegador puede iniciar sesión sin fallos de CORS
- [ ] Chrome (o Chromium) disponible; para móvil basta el modo responsive de DevTools

### Cómo registrar resultados

- **Pass** — se cumple el resultado esperado.
- **Fail** — anota el paso #, actual vs esperado, captura si es UI. No continúes una cadena que dependía de ese fallo.
- Marca **Blocked** si falló una precondición.

---

## Suite automatizada

Ejecuta esto primero; atrapa la mayoría de regresiones.

### Backend (pytest)

```bash
cd back
docker compose -f local.yml run --rm django pytest cleaning/ops -q
# Esperado: 109+ passed (el conteo puede crecer)
```

Suite completa:

```bash
docker compose -f local.yml run --rm django pytest
```

### Frontend unitario (Vitest)

```bash
cd front
npm test
# Esperado: todos los archivos en verde (roles, opsApi, opsCopy, …)
```

### Extremo a extremo (Playwright)

El backend debe estar arriba y con seed. Playwright arranca Vite en **:8090** (CORS permitido).

```bash
cd front
npm ci
npx playwright install chromium
npm run test:e2e
```

Opcional contra un SPA ya en marcha:

```bash
E2E_BASE_URL=http://localhost:8080 npm run test:e2e
```

Cubre: cliente crea SR → ops asigna a Ana → jornada → listas acotadas → el empleado nunca ve precio cliente / trabajos ajenos.

| Suite | Comando | Resultado (fecha / tester) |
|-------|---------|----------------------------|
| pytest ops | `… pytest cleaning/ops` | |
| vitest | `npm test` | |
| playwright | `npm run test:e2e` | |

---

## Happy path (manual)

Usa un perfil limpio o ventana privada. Preferible crear un **SR nuevo** para no depender de las muestras del seed.

### A. Cliente crea SR

1. Inicia sesión como `maria@abcpm.test`.
2. Abre **Services** → crear nueva solicitud.
3. Elige la unidad **215**, un servicio activo con Rate (p. ej. limpieza regular categoría B).
4. Completa agenda / acceso según se pida; envía.

**Esperado:** redirige a la lista de solicitudes (`/services`); toast de éxito con `SR-######`; el **precio** cliente es visible en detalle; la lista de Maria incluye la nueva fila.

- [ ] Pass / Fail — ________

### B. Operations asigna

1. Inicia sesión como `ops@castivas.test`.
2. Abre **Queue** (o la lista de servicios) → abre el SR nuevo.
3. Asigna a **Ana García** (`ana@castivas.test`); guarda.

**Esperado:** estado asignado; Ana aparece en la lista de empleados.

- [ ] Pass / Fail — ________

### C. Notificación de asignación

1. Inicia sesión como `ana@castivas.test`.
2. Abre las notificaciones in-app.

**Esperado:** notificación de la nueva asignación (RF-PLAT-3).

- [ ] Pass / Fail — ________

### D. Jornada del empleado

1. Como Ana, abre el SR asignado → **Journey**.
2. **Accept** → **Clock in** (permite geolocalización si el navegador lo pide) → **Clock out** → **Complete**.

**Esperado:** cada paso tiene éxito; el estado avanza a completed. Opcional: repetir a ancho de teléfono (~390px) — controles usables sin scroll horizontal.

- [ ] Pass / Fail — ________  
- [ ] Pass / Fail (ancho móvil) — ________

### E. Operations cierra + CSV

1. Como Ops, abre el SR completed → **Close**.
2. Descarga el reporte de horas/montos (UI de Reports o `GET /api/reports/hours.csv?from=YYYY-MM-DD&to=YYYY-MM-DD`).

**Esperado:** SR cerrado; el CSV descarga con horas y montos del snapshot (columnas de precio cliente / beneficio empleado presentes).

- [ ] Pass / Fail (cierre) — ________  
- [ ] Pass / Fail (CSV) — ________

### F. i18n

1. Con cualquier rol autenticado, usa las banderas ES/EN en la **barra superior** (también están en login).
2. Navega a **Perfil** y a Solicitudes; cambia de idioma sin recargar.
3. Recarga la página.

**Esperado:** nav, pantallas ops y Perfil cambian de idioma al instante y persisten (`localStorage` `cleaning-auth-locale`).

- [ ] Pass / Fail — ________

---

## Permisos / aislamiento

| # | Caso | Pasos | Esperado | Resultado |
|---|------|-------|----------|-----------|
| P1 | Empleado no puede crear SR | Login Ana → abrir `/services/new` o POST create | UI bloqueada / API 403 | |
| P2 | Trabajos empleado A ≠ B | Login `carlos@castivas.test` tras asignar un SR nuevo a Ana | Carlos **no** ve el SR de Ana | |
| P3 | Empleado oculta precio cliente | Como Ana en un SR asignado | Sin precio `$` del cliente; el beneficio puede verse | |
| P4 | Cliente solo ve su empresa | Como Maria, lista de servicios | Solo filas ABC; sin otras empresas | |
| P5 | Cliente no puede asignar/cerrar | Como Maria en detalle de SR | Sin acciones assign/close (o API 403) | |
| P6 | Ops ve la cola | Como Ops | SRs pendientes/asignados visibles entre clientes | |
| P7 | Maestros solo ops | Como Maria o Ana abrir `/masters` | Redirect / denegado | |
| P8 | Reasignar jornada | Ops asigna Ana → Ana acepta → Ops reasigna a Carlos | Carlos puede aceptar; Ana no continúa la jornada anterior | |

---

## Bordes y huecos conocidos

| Caso | Comportamiento Fase 1 | Seguimiento |
|------|----------------------|-------------|
| Servicio sin Rate | Alta bloqueada (400 / validación) | Ops mantiene Rates |
| Servicio `requires_notes` | Instrucciones especiales vacías → 400 | — |
| Servicio inactivo | No se puede elegir / 400 | — |
| GPS denegado por el navegador | La jornada igual registra; `inside_geofence` puede ser null | Calidad de lat/radio del predio |
| Seed sin DEBUG | El comando falla salvo `--force` | Nunca `--force` en prod compartida |
| KPIs del dashboard ops (RF-REP-1) | Puede estar delgado vs RF | Mejorar después |
| Pulido UX de fotos (RF-EMP-4) | Camino básico de evidencia; no bloquear por galería | Fase 1.x |
| Sugerencia de empleado (RF-OPS-2) | Solo picker manual | Fase 2 |

---

## Quejas (smoke)

1. Login como Maria → **New complaint** ligada a la unidad 215 (SR opcional).
2. Enviar.

**Esperado:** queja creada; Ops puede verla/gestionarla (lista o detalle según la UI actual).

- [ ] Pass / Fail — ________

---

## Resumen de corrida

| Área | Pass | Fail | Blocked |
|------|-----:|-----:|--------:|
| Automatizado (pytest / vitest / e2e) | | | |
| Happy path A–F | | | |
| Permisos P1–P8 | | | |
| Quejas | | | |

| Fecha | Tester | Build / commit (back · front) | Resultado global |
|-------|--------|-------------------------------|------------------|
| | | | Pass / Fail |

Pega esta tabla en la nota de entrega o el PR al cerrar la Fase 1.
