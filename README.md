# RestoCloud CR 🍽️

> **QR-based order management SaaS for small restaurants in Costa Rica.**
> Customers scan a QR at their table, order from their phone, and staff
> receives it in real time — no app downloads, no paper, no installation.

**Built end-to-end solo: architecture, database design, frontend, and deployment.**

<br>

[![Live Demo](https://img.shields.io/badge/Live%20Demo-▶%20Open-22c55e?style=for-the-badge)](https://restocloud-cr.vercel.app/login)
[![Stack](https://img.shields.io/badge/Stack-Vue%203%20%2B%20Supabase-4f46e5?style=for-the-badge)]()
[![DB](https://img.shields.io/badge/DB-PostgreSQL%2017%20%2B%20RLS-336791?style=for-the-badge)]()
[![Deploy](https://img.shields.io/badge/Deploy-Vercel-000000?style=for-the-badge&logo=vercel)]()

---

<!-- SCREENSHOT: Hero — captura panorámica del dashboard o del kanban lleno de pedidos.
     Que se vea que hay datos reales. Esta es la primera imagen que ve cualquiera. -->
![RestoCloud Dashboard](./screenshots/hero.png)

---

## The Problem

Small restaurants in Costa Rica lose time and money due to:

- **Handwritten orders** → transcription errors, wrong items sent to kitchen
- **No real-time visibility** → kitchen doesn't know what's pending, waitstaff doesn't know what's ready
- **No business data** → no sales tracking, no margins, no top-item analytics

RestoCloud replaces all of that with a kitchen tablet, a waiter's phone, and the customer's phone.

---

## 🔴 Live Demo

Runs against a real Supabase database with pre-loaded data.
**No installation required.**

### Admin Panel

| Role | Email | Password | Access |
|------|-------|----------|--------|
| Admin (Soda) | `demo-soda@restocloud.cr` | `demo1234` | Full panel + analytics |
| Admin (Fino) | `demo-fino@restocloud.cr` | `demo1234` | Full panel + fine dining flow |
| Waiter | `mesero@restocloud.cr` | `demo1234` | Orders, tables, availability |
| Kitchen | `cocina@restocloud.cr` | `demo1234` | Real-time kitchen view |

### Customer Flow (no login)

**→ [Open demo table](https://restocloud-cr.vercel.app/q/WUJC8F)**
*(Simulates being a customer at Table 1 of the demo restaurant)*

<!-- SCREENSHOT: Dos capturas lado a lado — celular mostrando el menú del cliente (izquierda)
     y celular mostrando el carrito o confirmación de pedido (derecha).
     Estas son las más impactantes para alguien que nunca vio el producto. -->
| Customer menu | Order confirmation |
|:---:|:---:|
| ![Customer menu](./screenshots/customer-menu.png) | ![Order confirmation](./screenshots/customer-order.png) |

---

## Business Flows

### Soda Flow — pay before kitchen prep
*(Soda: Costa Rican casual diner where customers pay upfront)*

```
Waiter opens table → Customer scans QR → Customer orders
→ Order enters as PENDING_PAYMENT
→ Waiter collects payment (cash / card / SINPE*)
→ Order moves to IN_KITCHEN
→ Kitchen marks READY_FOR_DELIVERY
→ Waiter delivers → DELIVERED
→ Table enters GRACE_PERIOD (20 min to reorder without re-scanning)
→ pg_cron releases table to AVAILABLE

* SINPE: Costa Rica's instant mobile payment system
```

### Fine Dining Flow — pay at the end

```
Waiter opens table → Customer scans QR → Customer orders
→ Order enters directly as IN_KITCHEN
→ Kitchen marks READY_FOR_DELIVERY → Waiter delivers
→ Waiter requests payment → PENDING_FINAL_PAYMENT
→ Paid → Table immediately released to AVAILABLE
```

<!-- SCREENSHOT: El kanban de comandas con columnas visibles y al menos 3-4 pedidos
     en distintos estados. Es la vista más "wow" para cualquier dev o cliente. -->
![Orders Kanban](./screenshots/kanban.png)

---

## Architecture

```
Client (mobile) ──── Vue 3 SPA (Vercel) ──── Supabase
                                               ├── PostgreSQL 17
                                               │     ├── RLS (Row Level Security)
                                               │     ├── Atomic RPCs
                                               │     └── pg_cron (automated jobs)
                                               ├── Auth (JWT + custom PG hook)
                                               ├── Realtime (WebSocket)
                                               ├── Storage (menu images)
                                               └── Edge Functions (Deno 2)
```

### Multi-tenancy
Every restaurant is an isolated tenant. All operational tables carry
`id_restaurante uuid`. RLS policies read that value from the JWT via
`fn_get_tenant_id()` — **no client query can ever cross tenant boundaries**,
regardless of the application layer.

---

## Key Technical Decisions

### RLS over application-layer filters
A `WHERE id_restaurante = X` filter in the frontend can be bypassed with
a manipulated JWT or an application bug. With RLS, isolation is enforced
**at the database engine level** — no application layer can break it.

### Atomic RPCs for critical operations
Creating an order requires inserting into `Pedidos` + N rows in
`Detalle_Pedido` + 1 row in `Historial_Estados`. If any step fails,
everything rolls back automatically. No inconsistent states possible.
Same applies to `rpc_onboarding_restaurante` and `rpc_cambiar_estado_pedido`.

### Price & cost snapshots
`precio_unitario_snapshot` and `costo_unitario_snapshot` are written
at order creation and **never modified**. Admins can update item prices
without affecting historical sales and margin calculations.

### Auth Hook in PostgreSQL (not Edge Functions)
Edge Functions have cold-start latency (500ms+). The hook that injects
`rol` and `id_restaurante` into every JWT runs as a PostgreSQL function —
**zero additional latency** on every login and token refresh.

### Optimistic locking via `updated_at`
Every UPDATE RPC includes `WHERE id = :id AND updated_at = :ts`.
If two waiters attempt to charge the same order simultaneously,
the second receives a 409 and the UI notifies them. Double-charging
is architecturally impossible.

### `SECURITY DEFINER` for soft-deletes
The RLS UPDATE policy has `WITH CHECK (deleted_at IS NULL)`.
A standard soft-delete would fail because the resulting row wouldn't
meet that condition. Soft-delete RPCs run with `SECURITY DEFINER`
and validate permissions internally via `fn_get_user_rol()`.

---

## Views & Roles

| View | Path | Roles | Purpose |
|------|------|-------|---------|
| Customer Menu | `/order/:tenant/:table` | Anonymous | Browse menu, cart, place order |
| Orders | `/admin/comandas` | Admin, Waiter, Kitchen | Real-time order Kanban |
| Tables | `/admin/mesas` | Admin, Waiter | CRUD, QR generation, live status |
| Menu | `/admin/menu` | Admin | Item CRUD with image and cost |
| Availability | `/admin/disponibilidad` | Admin, Waiter, Kitchen | Toggle available/sold out (Realtime to customer) |
| Dashboard | `/admin/dashboard` | Admin, Waiter | Analytics: sales, margins, top items, avg ticket |
| Sales | `/admin/ventas` | Admin, Waiter | Daily sales report with net profit per order |
| Staff | `/admin/personal` | Admin | User CRUD by role |
| Superadmin | `/superadmin/tenants` | Superadmin | New restaurant onboarding |

<!-- SCREENSHOT: Dos capturas lado a lado — Analytics dashboard (izquierda)
     y vista de mesas con estados en tiempo real (derecha).
     Muestran el valor de negocio más allá del flujo de pedidos. -->
| Analytics dashboard | Live table status |
|:---:|:---:|
| ![Analytics](./screenshots/dashboard.png) | ![Tables](./screenshots/tables.png) |

---

## Pricing Plans

| Plan | Price | Tables | Users | Items | Fine Dining |
|------|-------|--------|-------|-------|-------------|
| **GALLITO** | $30/mo | 5 | 2 | 50 | ✗ |
| **CASADO** | $55/mo | 15 | 5 | 200 | ✓ |
| **BANQUETE** | $90/mo | Unlimited | Unlimited | Unlimited | ✓ |

Limits enforced via PostgreSQL trigger (`fn_check_plan_limits`) BEFORE INSERT.
The frontend catches the error (`LIMIT_REACHED:MESAS:5`) and displays an upgrade modal.

---

## Full Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Vue 3, Vue Router, Pinia |
| Backend / DB | Supabase (PostgreSQL 17, RLS, Realtime, Storage) |
| Serverless | Edge Functions (Deno 2) |
| Jobs | pg_cron |
| Auth | Supabase Auth + Custom Access Token Hook (PostgreSQL) |
| Deploy | Vercel |
| Images | Supabase Storage (bucket `menu-items`, RLS per tenant) |

---

## Contact

Questions about the product or architecture?

**[LinkedIn](https://linkedin.com/in/ricardo-chaves-222868388)** ·
**[Email](mailto:ricardochavesg@gmail.com)**

---

> *Private repository — code available on request for technical evaluation.*
