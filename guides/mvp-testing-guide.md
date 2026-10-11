# Fase 1 (MVP) — Guía de pruebas

**Estado:** lista para ejecutar (dueños en unidades; hora en cuartos; categoría automática; lista de quejas; login de cliente/empleado a solicitudes)  
**Audiencia:** QA / desarrollador que verifica la Fase 1 antes de demo con cliente o UAT  
**Fuente de alcance:** [plan/2. Requerimientos.md](../plan/2.%20Requerimientos.md) §6 MVP  

Ruta corta de demo en vivo: [entregas/MVP-demo-checklist.md](../entregas/MVP-demo-checklist.md).  
Entorno: [mvp-dev-guide.md](mvp-dev-guide.md).

Sigue las secciones **en orden**. Al terminar A–J + permisos + quejas habrás recorrido el MVP completo.

---

## Alcance

Esta guía debe demostrar:

- El cliente (u Ops) puede crear un SR con snapshot de Rate y número `SR-######` (unidad existente o captura de dirección).
- Operations edita datos del SR antes de asignar, ve conteos en la cola, asigna/reasigna, cierra tras completar y exporta CSV desde **Informes**.
- El empleado ejecuta la jornada (aceptar → clock in/out → **foto** → completar) solo en trabajos asignados.
- La tabla de servicios de cada rol está acotada (sin fugas entre dueños ni entre asignados).
- La visibilidad de precio cliente vs beneficio empleado cumple las reglas por rol.
- Maestros ops: categorías, servicios, tarifas, propiedades y unidades. El dueño vive en la unidad. La categoría de la unidad se asigna sola por área y baños (no se elige a mano).
- El toggle ES/EN funciona en el copy de ops (nav, pantallas, Perfil). El español usa la bandera de Colombia.
- Cliente y ops crean una queja eligiendo primero la propiedad y luego la unidad, y quedan en la lista de quejas. El empleado no ve quejas.

**Fuera de esta fase (no reportar como bugs del MVP):**

- App nativa, instalación PWA, geofencing en segundo plano  
- Sugerencias inteligentes de empleado / mapa de rastro GPS (Fase 2)  
- Visitas de supervisor (Fase 2)  
- QuickBooks / inventario (Fase 3)  
- Pasarela de pago / depósito bancario de nómina  
- Logo corporativo final (hoy: favicon SVG Castivas en `public/favicon.svg`)  

---

## Entorno

| App | URL |
|-----|-----|
| Frontend (`npm run dev`) | http://localhost:8080 |
| Frontend (Docker, `front/local.yml`) | http://localhost:8180 |
| API | http://localhost:8010 |
| Admin (opcional) | http://localhost:8010/admin/ |
| Playwright (lo levanta el e2e) | http://localhost:8090 |

El navegador llama a la API en el host (`:8010`) tanto desde `:8080` como desde `:8180`. CORS local permite esos tres orígenes. Postgres del compose está en el host `:5440` (contenedor `5432`); no hace falta abrirlo para esta guía.

**Credenciales** (contraseña `demo1234`):

| Email | Rol |
|-------|-----|
| `ops@castivas.test` | Operations |
| `maria@abcpm.test` | Cliente (dueña de ABC Apartments y ABC Riverside) |
| `james@harborhomes.test` | Cliente (dueño de Pier View Condos y Harbor Townhomes) |
| `ana@castivas.test` | Empleado |
| `carlos@castivas.test` | Empleado |

Unidad demo para crear: **ABC Apartments / Unit 215** (dueña Maria López)  
(dirección seed: 215 West 95th St, NY 10025).

### Rutas útiles (SPA)

| Rol | Ruta | Nombre en nav (ES) |
|-----|------|--------------------|
| Ops | `/queue` | Cola de operaciones (conteos + pendientes) |
| Todos | `/services` | Solicitudes |
| Cliente / Ops | `/services/new` | Nueva solicitud |
| Ops | `/masters` | Catálogo (propiedades y unidades) |
| Ops | `/reports` | Informes (CSV horas/montos) |
| Cliente / Ops | `/complaints` | Quejas (lista) |
| Cliente / Ops | `/complaints/new` | Nueva queja |

Tras el login, Operations abre `/queue`. Cliente y empleado abren `/services`. Si el `?next=` apunta a una ruta que ese rol no puede ver, también van a su inicio (no a Acceso denegado).
| Empleado | `/services/:id/journey` | Jornada (desde detalle) |
| Todos | `/profile` | Perfil |

### Precondiciones (detener la corrida si alguna falla)

- [ ] `docker compose -f local.yml up -d postgres django` en `back/` termina bien
- [ ] Migraciones aplicadas (incluye `ops.0011_owners_on_units`): `python manage.py migrate`
- [ ] Seed aplicado: `python manage.py seed_mvp_demo` (sin error; se puede re-ejecutar)
- [ ] El `.env` del frontend tiene `VITE_API_URL=http://localhost:8010`
- [ ] El navegador puede iniciar sesión sin fallos de CORS
- [ ] Chrome (o Chromium) disponible; para móvil basta el modo responsive de DevTools
- [ ] Una imagen pequeña (JPG/PNG) lista para subir en la jornada

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
# Esperado: 126+ passed (el conteo puede crecer)
```

Suite completa:

```bash
docker compose -f local.yml run --rm django pytest
```

### Frontend unitario (Vitest)

```bash
cd front
# Node 20 (nvm use 20)
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

Opcional contra un SPA ya en marcha (elige el puerto que esté sirviendo la UI):

```bash
E2E_BASE_URL=http://localhost:8080 npm run test:e2e
# Docker SPA:
E2E_BASE_URL=http://localhost:8180 npm run test:e2e
```

Cubre: cliente crea SR → ops asigna a Ana → jornada **con foto** → listas acotadas → el empleado nunca ve precio cliente / trabajos ajenos → ops cierra.

| Suite | Comando | Resultado (fecha / tester) |
|-------|---------|----------------------------|
| pytest ops | `… pytest cleaning/ops` | |
| vitest | `npm test` | |
| playwright | `npm run test:e2e` | |

---

## Recorrido manual del MVP

Usa un perfil limpio o ventana privada. Preferible crear un **SR nuevo** para no depender solo de las muestras del seed.

### A. Ops — maestros y jerarquía de sitios

1. Inicia sesión como `ops@castivas.test`.
2. Abre **Catálogo** (`/masters`).
3. Revisa pestañas: Categorías, Servicios, Tarifas, Propiedades, Unidades.
4. Confirma que existen categorías A/B/C y rates (p. ej. regular × B).
5. En **Unidades**, confirma la unidad 215 de ABC Apartments, dueña Maria López, categoría **B** (o crea una de prueba bajo una propiedad). El formulario de unidad no pide categoría.

**Esperado:** tablas cargan; CRUD básico operable solo para Ops. Al guardar una unidad, la categoría queda asignada por área y baños (banda más ajustada; un mínimo o máximo vacío no filtra).

- [ ] Pass / Fail — ________

### B. Ops — cola y dashboard (RF-REP-1)

1. Abre **Cola de operaciones** (`/queue`).
2. Mira el bloque **Resumen** (chips por estado, urgentes/emergencias abiertas, total).

**Esperado:** conteos visibles; tabla con pendientes de asignar y/o por cerrar.

- [ ] Pass / Fail — ________

### C. Cliente crea SR (unidad existente)

1. Inicia sesión como `maria@abcpm.test`.
2. Abre **Solicitudes** → **Nueva solicitud** (`/services/new`).
3. Modo **Unidad existente**: elige propiedad ABC / unidad **215**, un servicio activo con Rate.
4. Completa fecha y hora. Los minutos solo ofrecen **00, 15, 30 o 45**. Completa flexibilidad y acceso; envía.

**Esperado:** redirige a `/services`; toast con `SR-######`; precio cliente visible para Maria; la lista incluye la fila nueva. Anota el número: **SR-______**.

- [ ] Pass / Fail — ________

### D. Ops crea SR por captura (casa unifamiliar) — opcional pero recomendado

1. Como Ops, `/services/new` → **Registrar dirección nueva**.
2. Activa **Casa unifamiliar** (oculta la etiqueta de unidad y fuerza la unidad `1`).
3. Elige propietaria **Maria López**, completa una dirección distinta a la del seed, área/baños, servicio y agenda; envía.

**Esperado:** SR creado; en maestros aparece la propiedad y la unidad `1`, dueña Maria López. No hay edificio.

- [ ] Pass / Fail — ________ · N/A si se omite — ________

### E. Ops edita SR antes de asignar (RF-OPS-1)

1. Como Ops, abre el SR de la sección C (estado `pending_review`).
2. En **Editar solicitud**, cambia p. ej. instrucciones de acceso y/o prioridad a **Urgente**; guarda.

**Esperado:** toast/aviso de guardado; detalle refleja los cambios; si priorizaste urgente, el chip/conteo de urgentes puede subir al refrescar la cola.

- [ ] Pass / Fail — ________

### F. Operations asigna (RF-OPS-3)

1. En el mismo detalle, asigna a **Ana García** (`ana@castivas.test`); guarda.

**Esperado:** estado asignado; redirige a la cola; Ana deja de aparecer como pendiente de asignación para ese SR.

- [ ] Pass / Fail — ________

### G. Notificación de asignación (RF-PLAT-3)

1. Inicia sesión como `ana@castivas.test`.
2. Abre las notificaciones in-app.

**Esperado:** notificación de la nueva asignación.

- [ ] Pass / Fail — ________

### H. Jornada del empleado (RF-EMP-*)

1. Como Ana, abre el SR asignado → **Iniciar jornada**.
2. **Aceptar** → **Marcar entrada** (permite geolocalización si el navegador lo pide) → **Marcar salida**.
3. En **Fotos de evidencia**, sube al menos **una** imagen (máx. 10). Notas opcionales.
4. **Completar servicio** (el botón permanece deshabilitado sin foto).

**Esperado:** cada paso tiene éxito; estado `completed`. Sin foto, Complete falla o no se habilita. Opcional: repetir a ~390px de ancho.

- [ ] Pass / Fail — ________  
- [ ] Pass / Fail (foto obligatoria) — ________  
- [ ] Pass / Fail (ancho móvil) — ________

### I. Ops cierra + evidencia + CSV (RF-OPS-5, RF-REP-2)

1. Como Ops, abre el SR completed.
2. Confirma en detalle: conteo de **Fotos** ≥ 1 y flag GPS (con/sin según permisos del navegador).
3. **Cerrar solicitud**.
4. Abre **Informes** (`/reports`) → elige rango de fechas que cubra el SR → **Descargar CSV**.

**Esperado:** SR `closed`; CSV con columnas `number, employee, unit, service, hours_net, client_price, employee_benefit, status` y la fila del SR.

- [ ] Pass / Fail (cierre) — ________  
- [ ] Pass / Fail (CSV UI) — ________

### J. i18n (RF-PLAT-2)

1. Con cualquier rol autenticado, usa las banderas en la **barra superior** (también están en login). Español es la bandera de Colombia; inglés, la de Estados Unidos.
2. Navega a **Perfil**, Cola/Solicitudes e Informes (si ops); cambia de idioma sin recargar.
3. Recarga la página.

**Esperado:** nav, pantallas ops y Perfil cambian al instante y persisten (`localStorage` `cleaning-auth-locale`).

- [ ] Pass / Fail — ________

---

## Permisos / aislamiento

| # | Caso | Pasos | Esperado | Resultado |
|---|------|-------|----------|-----------|
| P1 | Empleado no puede crear SR | Login Ana → abrir `/services/new` o POST create | UI bloqueada / API 403 | |
| P2 | Trabajos empleado A ≠ B | Login `carlos@castivas.test` tras asignar un SR nuevo a Ana | Carlos **no** ve el SR de Ana | |
| P3 | Empleado oculta precio cliente | Como Ana en un SR asignado | Sin precio `$` del cliente; el beneficio puede verse | |
| P4 | Cliente solo ve sus unidades | Como Maria, lista de servicios | Solo solicitudes de sus unidades; sin las de James | |
| P5 | Cliente no puede asignar/cerrar/editar | Como Maria en detalle de SR | Sin edit/assign/close (o API 403) | |
| P6 | Ops ve cola + informes | Como Ops `/queue` y `/reports` | Conteos y descarga CSV | |
| P7 | Maestros solo ops | Como Maria o Ana abrir `/masters` | Van a `/services`, no a Acceso denegado | |
| P8 | Informes solo ops | Como Maria o Ana abrir `/reports` | Van a `/services`, no a Acceso denegado | |
| P9 | Reasignar jornada | Ops asigna Ana → Ana acepta → Ops reasigna a Carlos | Carlos puede aceptar; Ana no continúa la jornada anterior | |
| P10 | Complete sin foto | Ana en jornada sin subir imagen → Complete | 400 / botón deshabilitado; SR no completed | |

---

## Bordes y huecos conocidos

| Caso | Comportamiento Fase 1 | Seguimiento |
|------|----------------------|-------------|
| Servicio sin Rate | Alta bloqueada (400 / validación) | Ops mantiene Rates |
| Servicio `requires_notes` | Instrucciones especiales vacías → 400 | — |
| Servicio inactivo | No se puede elegir / 400 | — |
| GPS denegado por el navegador | La jornada igual registra; `inside_geofence` puede ser null | Calidad de lat/radio del predio |
| Seed sin DEBUG | El comando falla salvo `--force` | Nunca `--force` en prod compartida |
| Editar SR ya en curso / completed | PATCH rechazado (solo `pending_review` / `assigned`) | — |
| Hora fuera de 00/15/30/45 | Alta o edición de SR → 400 | — |
| Más de 10 fotos | API 400 | — |
| Sugerencia de empleado (RF-OPS-2) | Solo picker manual | Fase 2 |
| Filtros avanzados de la tabla de servicios (RF-REP-3) | Listado por rol; filtros ricos pueden estar delgados | Fase 1.x / 2 |
| Logo final Castivas | Favicon SVG actual | Branding |

---

## Quejas (smoke) (RF-QUE-1)

1. Login como Maria → **Quejas** (`/complaints`) → **Nueva queja**.
2. Elige la propiedad **ABC Apartments** (autocompletar). La lista de unidades solo muestra las suyas; elige **215**.
3. Solicitud relacionada opcional (si eliges una, que sea de esa unidad). Envía.

**Esperado:** vuelve a `/complaints` y la queja aparece (propiedad, unidad, tipo, prioridad, estado). Ops, en la misma lista, ve también las quejas de otros dueños. Ana no tiene Quejas en el menú; `/complaints` la lleva a solicitudes. James no ve la queja de Maria.

- [ ] Pass / Fail — ________

---

## Resumen de corrida

| Área | Pass | Fail | Blocked |
|------|-----:|-----:|--------:|
| Automatizado (pytest / vitest / e2e) | | | |
| Happy path A–J | | | |
| Permisos P1–P10 | | | |
| Quejas | | | |

| Fecha | Tester | Build / commit (back · front) | Resultado global |
|-------|--------|-------------------------------|------------------|
| | | | Pass / Fail |

Pega esta tabla en la nota de entrega o el PR al cerrar la Fase 1.
