# LuCash — Frontend

SPA de control de finanzas personales. Consume la API REST del backend en **`/api/v1`**.

**Stack:** Vite 5 · JavaScript ES modules (vanilla) · Tailwind CDN · Chart.js · Lucide Icons

---

## Arranque

```bash
npm install
npm run dev      # http://localhost:5173  (strictPort)
npm run build    # salida en dist/
npm run preview  # previsualizar build
```

Requisito: backend en `http://127.0.0.1:8000` (Docker Compose en `../backend`).

### Proxy de desarrollo

[`vite.config.js`](vite.config.js) reenvía `/api` → `http://127.0.0.1:8000`.  
El cliente usa base relativa `API_BASE_URL = '/api/v1'`, así que **no hay problemas de CORS** entre `localhost` y `127.0.0.1`.

---

## Estructura

```
frontend/
├── index.html              # Shell: auth, MFA, sidebar, modales, viewport
├── vite.config.js          # puerto 5173 + proxy /api
├── package.json
└── src/
    ├── main.js             # Estado, auth, handlers, carga de datos
    ├── style.css           # Tema glass / tipografía
    ├── services/
    │   └── api.js          # Cliente HTTP (JWT, refresh, páginas)
    ├── router/
    │   └── router.js       # Rutas SPA + gate de Catálogo (admin+MFA)
    └── views/
        ├── DashboardView.js
        ├── TransactionsView.js
        ├── TransfersView.js
        ├── AccountsView.js
        ├── CounterpartiesView.js
        ├── BudgetsView.js
        ├── CatalogView.js      # solo admin con MFA
        └── SettingsView.js
```

Patrón de vista: cada módulo exporta `{ render(), init(state, utils) }`. El router inyecta HTML en `#router-viewport` y llama a `init`.

---

## Pantallas

| Ruta (`switchTab`) | Vista | Descripción |
|--------------------|-------|-------------|
| `dashboard` | Panel | KPIs, comparativa de periodo, gráficos, desgloses de `/reports/summary` |
| `transactions` | Transacciones | Filtros **en servidor**, export CSV/JSON, CRUD gasto/ingreso |
| `transfers` | Transferencias | `POST /transactions/transfers` (misma moneda) |
| `accounts` | Cuentas | Listado con saldo; crear/editar; desactivar/reactivar |
| `counterparties` | Contrapartes | CRUD + soft-delete |
| `budgets` | Presupuestos | Límites + `/budgets/status`; desactivar/reactivar |
| `catalog` | Catálogo | Categorías/subcategorías (escribe solo **admin + MFA**) |
| `settings` | Ajustes | Perfil, setup MFA TOTP, desactivar cuenta propia |

---

## Autenticación

1. Registro: `POST /auth/register` (JSON).
2. Login: `POST /auth/login` (**form-urlencoded** `username` / `password`).
3. Tokens en `localStorage`: `ff_access_token`, `ff_refresh_token`.
4. Si la respuesta trae `mfa_required`, se muestra la pantalla MFA → `POST /auth/mfa/verify`.
5. En `401` con JWT, el cliente intenta `POST /auth/refresh` una vez y reintenta.
6. Logout: `POST /auth/logout` + limpia tokens.

Usuarios demo del seed backend: `demo001` / `Password123!` (ver [`../backend/docs/FRONTEND.md`](../backend/docs/FRONTEND.md)).

---

## Cliente API (`src/services/api.js`)

Responsabilidades:

- Headers `Authorization: Bearer …`
- Errores FastAPI (`detail` string o lista de validación)
- `fetchAllPages` para recursos paginados (`limit`/`offset`, máx. 100)
- Métodos alineados a los contratos del backend (campos en español: `monto`, `tipo`, `fecha`, `limite`, …)

### Recursos cubiertos

| Dominio | Operaciones principales |
|---------|-------------------------|
| Auth | register, login, mfaVerify/setup/confirm, logout, me, refresh |
| Users | get / update / deactivate |
| Accounts | list, create, update, deactivate, reactivate |
| Counterparties | CRUD + reactivate |
| Categories / Subcategories | list + escritura admin |
| Transactions | list (filtros), CRUD, transfers, export |
| Budgets | list, status, create, update, deactivate, reactivate |
| Reports | `getReportSummary({ account_id, date_from, date_to })` |
| Health | `GET /health` (indicador de conexión) |

---

## Estado central (`main.js`)

```js
state = {
  user, transactions, budgets, budgetStatus,
  categories, subcategories, accounts, counterparties,
  report, txFilters, reportFilters, ui, pendingMfaToken
}
```

Al iniciar sesión se ejecuta `Router.init` y luego `loadAllData()` en paralelo (cuentas, catálogo, movimientos, presupuestos, contrapartes, reporte, `/auth/me`).

**Contratos importantes (evitar regresiones):**

- Payloads usan nombres del backend (`monto`, no `amount`; `tipo: gasto|ingreso`).
- `medio_pago=cuenta` → `account_id` (sin wallets `tipo=efectivo` en el selector).
- `medio_pago=efectivo` → `moneda` + `account_id: null`.
- Soft-delete HTTP = desactivar; la UI ofrece reactivar donde el API lo permite.
- **Saldo neto** del panel = `total_ingresos − total_gastos` del summary (sin transferencias).

---

## UI / UX

- Shell en `index.html`: login/registro, challenge MFA, sidebar, modales globales.
- Estilos glass en `style.css` + utilidades Tailwind (CDN).
- Iconos Lucide; gráficos Chart.js en el dashboard.
- Indicador de API (ping a `/health` cada 5 s).

---

## Convenciones para contribuir

1. No modificar el backend desde este cliente: adaptar el frontend a `/api/v1`.
2. Mantener el puerto **5173** (`strictPort`) o el proxy deja de ser el entorno esperado.
3. Nuevas pantallas: vista + ruta en `router.js` + enlace en el sidebar de `index.html` + handlers en `main.js` si hace falta.
4. Preferir pantallas delgadas (`render`/`init`) y lógica de API/sesión en `api.js` / `main.js`.

---

## Resolución de problemas

| Síntoma | Qué revisar |
|---------|-------------|
| Login “falla” sin 401 en Network | Estás fuera de `:5173` o sin proxy (evita puertos alternos) |
| 401 tras login | Token no guardado / backend caído / rate limit auth (~10/min) |
| Catálogo no aparece | Usuario no es `admin` o MFA no está activo |
| 422 al crear movimiento | Falta subcategoría en el catálogo o IDs inválidos |
| Transferencia rechazada | Cuentas distintas, misma moneda, ≥ 2 cuentas activas |

Más detalle de contrato HTTP: [`../backend/docs/API.md`](../backend/docs/API.md) y [`../backend/docs/FRONTEND.md`](../backend/docs/FRONTEND.md).
