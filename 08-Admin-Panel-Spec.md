# 08 — Admin Panel Spec

The **platform-owner console** (`kukkuone-admin-web`) is the operate-and-support
surface for KukkuOne. It is a **separate React app** from the actor apps and is
**not** part of actor onboarding. Its single privileged user, the naptrixlabs
super-admin, sees **all data across every organization** in read-first mode and
performs support/operations tasks.

**App:** `kukkuone-admin-web` — React + Vite + TypeScript + **Ant Design**,
TanStack Query, Zustand, React Hook Form + Zod, React Router.
**Backend:** the same NestJS gateway (`/api` prefix) + services; the console reads
the **farmer** surface (`/api/farmer/*`, [07](07-Farmer-Module-Spec.md)) and, as
they land, the supplier/distributor surfaces — with `Principal.isSuperAdmin`
lifting the org-scope filter to **cross-org**.

Related docs: [Product Overview](01-Product-Overview.md) ·
[Auth & RBAC](05-Auth-And-RBAC.md) · [Data Model](06-Data-Model.md) ·
[Farmer Module Spec](07-Farmer-Module-Spec.md) ·
[Client Apps Spec](09-Client-Apps-Spec.md) ·
[API Conventions](12-API-Conventions.md) ·
[Roadmap & Milestones](13-Roadmap-And-Milestones.md)

---

## 1. Purpose & Audience

| Aspect | Detail |
|--------|--------|
| **Who** | Platform owner `naptrixlabs@gmail.com` (super-admin). Matched against `ADMIN_SUPERUSER_EMAIL` at token resolution → `Principal.isSuperAdmin = true` (see [05](05-Auth-And-RBAC.md) §8). |
| **Scope** | **Cross-org read** across every organization; not bound to a single `orgId`. Org isolation filter is lifted for this principal only. |
| **Purpose** | Operate & support: inspect any org's farms/sheds/batches/logs/finance, triage tickets, verify data integrity, watch platform health, manage org/user provisioning. |
| **NOT** | Not an actor. Skips org creation and capability selection entirely. Does not run production; it observes and supports the actors who do. |
| **Writes** | Governed by the console's **own** admin permissions and fully audited (before/after `AuditLog`, see [06](06-Data-Model.md)). Cross-org reads are logged. MVP is **read-heavy**; the few write actions are listed per-page in §5. |
| **Auth** | Google sign-in → gateway verifies token → super-admin allowlist match → console. No onboarding, no capability picker. Web session token held **in memory** with silent refresh (see [05](05-Auth-And-RBAC.md) §9). |

> The admin console **never** invents business rules. Every state transition,
> guard, and computed metric it displays is owned by the services in
> [07](07-Farmer-Module-Spec.md)/[06](06-Data-Model.md). Admin write actions call
> the **same** action sub-resources actors call; RBAC and audit still apply.

---

## 2. Layout

Fixed **left sidebar** (module nav) + **top header** (org switcher / global search /
user menu / page actions) + **filter bar** + **content area**. Standard Ant Design
`Layout` shell.

```
┌───────────────────────────────────────────────────────────────────────────────┐
│ TOP HEADER  (Layout.Header, fixed)                                              │
│ ┌──────────┐  ┌───────── Org switcher ─────────┐  ┌── global search ──┐ ┌─────┐ │
│ │ KukkuOne │  │ [All organizations ▾]          │  │ 🔍 batch/farm/user │ │ 👤 ▾ │ │
│ │  ADMIN   │  │  (Select, cross-org scope)     │  │  (global Search)   │ │user │ │
│ └──────────┘  └────────────────────────────────┘  └───────────────────┘ └─────┘ │
├───────────────┬───────────────────────────────────────────────────────────────┤
│ SIDEBAR       │  FILTER BAR  (sticky under header)                              │
│ Layout.Sider  │  ┌───────────────────────────────────────────────────────────┐ │
│ + Menu        │  │ Org ▾ · Farm ▾ · Status ▾ · DateRange 📅 · [Search] [Clear]│ │
│ (collapsible) │  └───────────────────────────────────────────────────────────┘ │
│               │                                                                 │
│ ▸ Dashboard   │  CONTENT AREA  (Layout.Content)                                 │
│ ▸ Organizations│ ┌───────────────────────────────────────────────────────────┐ │
│ ▸ Users&Roles │  │  antd Table  (columns · sort · pagination)                 │ │
│ ▸ Farms       │  │  ┌─────┬──────────┬────────┬─────────┬──────────┐          │ │
│ ▸ Sheds       │  │  │ Code│ Org      │ Status │ Age     │ …        │  ──▶ row  │ │
│ ▸ Batches     │  │  ├─────┼──────────┼────────┼─────────┼──────────┤   opens   │ │
│ ▸ Daily Logs  │  │  │ …   │ …        │ [chip] │ 31d     │ …        │   Drawer  │ │
│ ▸ Feed        │  │  └─────┴──────────┴────────┴─────────┴──────────┘          │ │
│ ▸ Health      │  │                                                            │ │
│ ▸ Growth      │  └───────────────────────────────────────────────────────────┘ │
│ ▸ Finance     │                                                                 │
│ ▸ Harvest     │   ┌───────────── Drawer (right, detail) ──────────────────────┐ │
│ ▸ Suppliers ▾ │   │  Header: B-2026-014 · [status] · [actions ▾]              │ │
│ ▸ Distributors▾│  │  Tabs: Overview | Daily | Feed | Health | Growth | Finance│ │
│ ▸ Notifications│  │  …read-only detail + audited action buttons…             │ │
│ ▸ Audit Log   │   └───────────────────────────────────────────────────────────┘ │
│ ▸ Settings    │                                                                 │
└───────────────┴───────────────────────────────────────────────────────────────┘
```

### 2.1 Ant Design components used

| Region | Component(s) |
|--------|--------------|
| Shell | `Layout`, `Layout.Header`, `Layout.Sider` (collapsible), `Layout.Content` |
| Sidebar nav | `Menu` (mode `inline`, `SubMenu` for Suppliers/Distributors), route-driven `selectedKeys` |
| Org switcher | `Select` (showSearch, `All organizations` default option) → sets cross-org scope in a Zustand store |
| Global search | `Input.Search` / `AutoComplete` (jump to org/farm/batch/user) |
| User menu | `Dropdown` + `Avatar` (profile, sign out) |
| Page actions | `Button` / `Dropdown.Button` in header or table toolbar |
| Filter bar | `Space` of `Select`, `DatePicker.RangePicker`, `Input.Search`, `Button` (Clear) |
| List | `Table` (server pagination, sort, column filters), `Tag` status chips, `Badge` counts |
| Detail | `Drawer` (right, size `large`) with `Tabs`, `Descriptions`, `Timeline`, `Statistic` |
| Create/edit | `Modal` + `Form` (React Hook Form + Zod resolver), `Select`, `DatePicker`, `InputNumber` |
| Feedback | `Skeleton`/`Spin` (loading), `Empty` (empty state), `message`/`notification` (toasts), `Popconfirm` (destructive) |

---

## 3. Navigation / Module Map (sidebar)

MVP ships **org/users provisioning + the full farmer data surface first**
(mirrors the build order in [07](07-Farmer-Module-Spec.md) and
[13](13-Roadmap-And-Milestones.md)). Supplier/Distributor modules follow their
service specs.

| # | Sidebar item | Reads | MVP | Notes |
|---|--------------|-------|:---:|-------|
| 1 | **Dashboard** | aggregate (§6) | ✅ | Cross-org KPIs + three guiding questions + alerts |
| 2 | **Organizations** | core `Organization` ([06](06-Data-Model.md) §3) | ✅ | The org registry; entry point for cross-org scope |
| 3 | **Users & Roles** | core `User`/`Membership`/roles ([05](05-Auth-And-RBAC.md) §6) | ✅ | Provisioning + role/capability view |
| 4 | **Farms** | `/api/farmer/farms` ([07](07-Farmer-Module-Spec.md) §1) | ✅ | |
| 5 | **Sheds** | `/api/farmer/.../sheds` ([07](07-Farmer-Module-Spec.md) §2) | ✅ | Lifecycle status chips |
| 6 | **Batches** | `/api/farmer/batches` ([07](07-Farmer-Module-Spec.md) §3) | ✅ | Central object; per-batch console |
| 7 | **Daily Logs** | `/api/farmer/.../daily-logs` ([07](07-Farmer-Module-Spec.md) §5) | ✅ | Read-only timeline with lock indicators |
| 8 | **Feed** | `/api/farmer/feed/*` ([07](07-Farmer-Module-Spec.md) §6) | ✅ | Purchases, inventory, consumption, FCR |
| 9 | **Health / Medicine** | `/api/farmer/medicine/*`, vaccinations, health events ([07](07-Farmer-Module-Spec.md) §7) | ✅ | |
| 10 | **Growth & Performance** | `/api/farmer/.../performance`, `.../growth-trend` ([07](07-Farmer-Module-Spec.md) §8) | ✅ | Charts, estimate badges |
| 11 | **Finance** | `/api/farmer/.../finance-summary`, expenses/revenue ([07](07-Farmer-Module-Spec.md) §9) | ✅ | P&L + unit economics |
| 12 | **Harvest** | `/api/farmer/harvest/*` ([07](07-Farmer-Module-Spec.md) §10) | ✅ | Pipeline + visibility flags |
| 13 | **Suppliers ▾** → Products · Orders · Dispatch · Invoices | `/api/supplier/*` ([06](06-Data-Model.md) §5) | ⏳ | Post-farmer; per supplier-api spec |
| 14 | **Distributors ▾** → Purchases · Settlements | `/api/distributor/*` ([06](06-Data-Model.md) §6) | ⏳ | Post-farmer; per distributor-api spec |
| 15 | **Notifications / Alerts** | `/api/farmer/alerts` (+ supplier/distributor alerts later) ([07](07-Farmer-Module-Spec.md) §12) | ✅ (farmer slice) | Cross-org alert triage |
| 16 | **Audit Log** | core `AuditLog` ([06](06-Data-Model.md), [05](05-Auth-And-RBAC.md) §9) | ✅ | Every Approve/Dispatch/Settle + admin writes |
| 17 | **Settings** | console/platform config | ✅ (minimal) | Admin allowlist view, thresholds, feature flags |

Legend: ✅ MVP-first · ⏳ ships with its service module.

Sidebar order deliberately follows the **farmer batch lifecycle** top-to-bottom
(Farm → Shed → Batch → daily ops → growth → finance → harvest), so the console
reads like the production loop it supports.

---

## 4. Settlement of Scope: Cross-Org Reads

Every farmer endpoint is org-scoped ([07](07-Farmer-Module-Spec.md) §0.1). In the
admin console the super-admin `Principal` carries `isSuperAdmin = true`, so:

- The **Org switcher** default `All organizations` sends no `orgId` filter and the
  gateway returns rows across all orgs (each row carries its `orgId`/org name).
- Selecting a specific org in the switcher pins an `orgId` filter for the session,
  scoping every page to that org (equivalent to "impersonate view", read-only).
- Cross-org reads are **logged**; write actions require explicit admin permission
  and write an `AuditLog` row (see [05](05-Auth-And-RBAC.md) §9).

Every list page therefore adds an **Org** column (hidden when a single org is
pinned) and an **Org** filter in the filter bar.

---

## 5. Per-Page Spec (MVP-first pages)

Columns are the antd `Table` columns; Filters are the filter-bar controls; Detail
is the right `Drawer`; Actions are audited buttons (RBAC/guards enforced by the
service). Endpoints reference [07](07-Farmer-Module-Spec.md) unless noted.

### 5.1 Dashboard

See §6.

### 5.2 Organizations

| | |
|---|---|
| **Columns** | Name · Org code · Capabilities (`Tag`s: Farmer/Distributor/Supplier) · Owner (name/email) · #Users · #Farms · #Active batches · Currency · Status · Created |
| **Filters** | Capability · Status (active/suspended) · Created `RangePicker` · Search (name/code/owner email) |
| **Detail (Drawer)** | `Descriptions`: profile, currency, capabilities, created; tabs → **Users** (memberships + roles), **Farms** (drill to Farms page pinned to org), **Activity** (recent audit rows), **Alerts** (open alerts for the org) |
| **Actions** | Pin org scope (jump into org) · Suspend / reactivate org · Edit profile (`Modal`+`Form`) · Manage capabilities |
| **RBAC** | Read: super-admin. Suspend/edit/capabilities: admin write permission, audited. Source of truth: core `Organization` ([06](06-Data-Model.md) §3), capability model [05](05-Auth-And-RBAC.md) §4. |

### 5.3 Users & Roles

| | |
|---|---|
| **Columns** | Name · Email · Org(s) · Role(s) per capability (`Tag`s, e.g. `FARMER_OWNER`) · Provider (`ExternalIdentity.provider`) · Last active · Status |
| **Filters** | Org · Capability · Role · Status (active/disabled) · Search (name/email) |
| **Detail (Drawer)** | `Descriptions`: identity, linked `ExternalIdentity` rows, memberships; **Roles** tab (role + resolved permission keys, read-only, `resource:Action`); **Audit** tab (actions by this user) |
| **Actions** | Invite/provision user · Assign/revoke role within an org · Disable/enable user · Resend invite |
| **RBAC** | Read: super-admin. Role/user mutations: admin write, audited. Roles per capability from [05](05-Auth-And-RBAC.md) §6.1; permission keys §6.2. |

### 5.4 Farms

| | |
|---|---|
| **Columns** | Code · Name · Org · City/State · Status (`ACTIVE`/`INACTIVE`/`ARCHIVED`) · #Sheds · #Active batches · Area (m²) · Updated |
| **Filters** | Org · Status · State/city · Search (name/code) |
| **Detail (Drawer)** | `Descriptions`: profile, location (map link), area, metadata; **Sheds** tab (grid, status chips), **Batches** tab (active first), **Performance** tab (`/api/farmer/farms/:farmId/performance`) |
| **Actions** | View only (read-first). Optional admin support: archive/activate — `POST /api/farmer/farms/:farmId/archive` \| `/activate` (guarded, §1.1 of [07](07-Farmer-Module-Spec.md)) with `Popconfirm` |
| **RBAC** | Read: super-admin. Archive/activate: admin write, service guard `409 FARM_HAS_ACTIVE_WORK`. |
| **Endpoints** | `GET /api/farmer/farms`, `GET /api/farmer/farms/:farmId` ([07](07-Farmer-Module-Spec.md) §1.1) |

### 5.5 Sheds

| | |
|---|---|
| **Columns** | Code · Name · Farm · Org · Capacity (birds) · Status (`EMPTY`…`READY`, chip) · Current batch · Occupancy % · Last cycle (ready/cleaned) |
| **Filters** | Org · Farm · Status · Occupied? · Search (code/name) |
| **Detail (Drawer)** | Shed profile + equipment (`Descriptions`), lifecycle `Timeline` (last cycle: cleanedAt/disinfectedAt/readyAt), current batch card, **Historical batches** tab (`GET /api/farmer/sheds/:shedId/batches`) |
| **Actions** | View only in MVP. Lifecycle actions are operator-driven ([07](07-Farmer-Module-Spec.md) §2.2); admin surfaces them read-only with state-machine diagram. |
| **RBAC** | Read: super-admin. |
| **Endpoints** | `GET /api/farmer/farms/:farmId/sheds`, `GET /api/farmer/sheds/:shedId` ([07](07-Farmer-Module-Spec.md) §2.2) |

### 5.6 Batches

| | |
|---|---|
| **Columns** | Code · Org · Farm · Shed · Breed · Status (`PLANNED`…`CLOSED`, chip) · Age (days) · Opening → current birds · Cum. mortality % · FCR · Target harvest date · Visibility (`HIDDEN`/`NETWORK`) |
| **Filters** | Org · Farm · Shed · Status · Breed · Visibility · Placement `RangePicker` · Search (code) |
| **Detail (Drawer)** | Per-batch console with `Tabs` mirroring [07](07-Farmer-Module-Spec.md) §3.4: **Overview** (header + lifecycle diagram + derived counts), **Daily** (log timeline, §5.7), **Feed** (§5.8), **Health** (§5.9), **Growth** (§5.10), **Finance** (§5.11), **Harvest** (§5.12) |
| **Actions** | Read-first. Support-only, audited, service-guarded: `POST /api/farmer/batches/:batchId/cancel` (Owner-equiv), visibility override `PATCH …/visibility`. Lifecycle advances stay actor-driven. |
| **RBAC** | Read: super-admin. Writes: admin write + service transition guards (`409 BATCH_INVALID_TRANSITION`). |
| **Endpoints** | `GET /api/farmer/batches`, `GET /api/farmer/batches/:batchId` ([07](07-Farmer-Module-Spec.md) §3.3) |

### 5.7 Daily Logs

| | |
|---|---|
| **Columns** | Log date · Org · Batch · Opening → closing birds · Mortality (+culls) · Daily mortality % · Avg weight (g) · Feed (kg) · Locked? (`Tag`) · Actor |
| **Filters** | Org · Farm · Batch · Date `RangePicker` · Locked/editable · Has-attachment |
| **Detail (Drawer)** | Full log `Descriptions` (population/weight/feed/water/health/environment), attachments gallery, continuity check, lock status |
| **Actions** | **Read-only** in admin (logs are actor-entered). Manager force-lock (`POST /api/farmer/daily-logs/:logId/lock`) is an actor action; admin views lock state only. |
| **RBAC** | Read: super-admin. |
| **Endpoints** | `GET /api/farmer/batches/:batchId/daily-logs`, `.../daily-logs/:date` ([07](07-Farmer-Module-Spec.md) §5.3) |

### 5.8 Feed

| | |
|---|---|
| **Columns** | Purchases: Date · Org · Supplier · Feed type · Bags · Kg · Unit cost · Total. Inventory: Org · Farm · Feed type · Bags balance · Kg balance. Consumption: Batch · Date · Feed type · Kg · Derived cost |
| **Filters** | Org · Farm · Feed type (`STARTER`/`GROWER`/`FINISHER`) · Date range · (Inventory) low-balance only |
| **Detail (Drawer)** | Per-feed-type ledger; batch feed usage + FCR panel (`GET /api/farmer/batches/:batchId/feed`) with estimate badge when arrival weight unknown |
| **Actions** | View only in admin (purchases/receipts/consumption are actor-entered). |
| **RBAC** | Read: super-admin. |
| **Endpoints** | `GET /api/farmer/feed/purchases`, `/api/farmer/feed/inventory`, `GET /api/farmer/batches/:batchId/feed` ([07](07-Farmer-Module-Spec.md) §6.2) |

### 5.9 Health / Medicine

| | |
|---|---|
| **Columns** | Medicine inventory: Org · Item · Unit · Balance · Reorder level · Below-reorder? Vaccinations: Batch · Vaccine · Scheduled · Due · Status (`Scheduled`/`Due`/`Completed`) · Administered. Health events: Batch · Date · Type · Severity · Affected · Outcome |
| **Filters** | Org · Farm · Batch · Vaccination status · Health event type/severity · Date range · Below-reorder |
| **Detail (Drawer)** | Batch vaccination schedule (`Timeline` with due badges), medicine usage list, health-event log with treatments/outcomes |
| **Actions** | View only in admin. |
| **RBAC** | Read: super-admin. |
| **Endpoints** | `GET /api/farmer/medicine/inventory`, `GET /api/farmer/batches/:batchId/vaccinations`, `.../health-events` ([07](07-Farmer-Module-Spec.md) §7.2) |

### 5.10 Growth & Performance

| | |
|---|---|
| **Columns** | Batch · Org · Age · Avg weight (g) · Total live weight (kg) · Cum. mortality % · Livability % · FCR · ADG (g) · Projected harvest wt (estimate) |
| **Filters** | Org · Farm · Status · Breed · Underperformers (below benchmark) |
| **Detail (Drawer)** | Growth charts (weight/FCR/mortality vs age) from `growth-trend`; every projected value carries an **estimate `Tag`** (actual vs estimate flags, [07](07-Farmer-Module-Spec.md) §8) |
| **Actions** | View only; "flag as underperformer" is a console-local annotation (audited). |
| **RBAC** | Read: super-admin. |
| **Endpoints** | `GET /api/farmer/batches/:batchId/performance`, `.../growth-trend`, `GET /api/farmer/farms/:farmId/performance` ([07](07-Farmer-Module-Spec.md) §8.1) |

### 5.11 Finance

| | |
|---|---|
| **Columns** | Batch · Org · Total revenue · Total cost · Profit · Cost/bird · Cost/kg · Profit/kg · Frozen? (`SETTLED`) |
| **Filters** | Org · Farm · Batch status · Date range · Profitable/loss-making · Currency |
| **Detail (Drawer)** | Full P&L: expense table (by category `CHICK`…`OTHER`), revenue table (`BIRD_SALE`/`MANURE`/`OTHER`), unit-economics `Statistic` tiles; auto-generated lines marked read-only; frozen badge when `SETTLED` |
| **Actions** | **View only** in admin (finance is Accountant-owned; frozen once `SETTLED`, `423 BATCH_FINANCE_LOCKED`). |
| **RBAC** | Read: super-admin. All money is integer minor units + currency ([07](07-Farmer-Module-Spec.md) §0.3). |
| **Endpoints** | `GET /api/farmer/batches/:batchId/finance-summary`, `.../expenses`, `.../revenue` ([07](07-Farmer-Module-Spec.md) §9.2) |

### 5.12 Harvest

| | |
|---|---|
| **Columns** | Batch · Org · Farm · Age · Current birds · Avg weight · Est. live weight (kg) · Harvest window (earliest–latest) · Status (`READY_FOR_HARVEST`) · Visibility (`HIDDEN`/`NETWORK`) |
| **Filters** | Org · Farm · Near-harvest only · Visibility · Status |
| **Detail (Drawer)** | Harvest snapshot (`harvest-info`), eligibility signal, what distributors would see when `NETWORK`; settlement status if progressing |
| **Actions** | View pipeline. Support-only visibility override `PATCH /api/farmer/batches/:batchId/visibility` (audited, guard: `ACTIVE`/`READY_FOR_HARVEST` only). Mark-ready stays actor-driven. |
| **RBAC** | Read: super-admin. Visibility override: admin write, audited. |
| **Endpoints** | `GET /api/farmer/harvest/eligible`, `GET /api/farmer/batches/:batchId/harvest-info` ([07](07-Farmer-Module-Spec.md) §10.1) |

---

## 6. Cross-Org Dashboard

The landing page answers the three guiding questions ([01](01-Product-Overview.md)
§5) **at platform scale**, aggregating across all orgs (or the pinned org).

```
┌──────────────────────────── KPI ROW (Statistic cards) ─────────────────────────┐
│  Orgs: 42  │  Active farmers: 38  │  Active batches: 214  │  Birds on platform:  │
│            │                      │                       │  1.72M │ Avg FCR 1.68│
└────────────────────────────────────────────────────────────────────────────────┘
┌── What happened? ──────────┐ ┌── What needs attention today? ─┐ ┌── What next? ──┐
│ Logs recorded today: 601   │ │ 🔴 Mortality spike — Org A, B3 │ │ 12 batches near │
│ New batches placed: 7      │ │ 🟠 Low feed — Org C (FINISHER) │ │  harvest window │
│ Harvests completed: 3      │ │ 🟠 Vaccination due — 9 batches │ │ 5 settlements   │
│ Settlements closed: 4      │ │ 🔴 Settlement overdue — Org D  │ │  pending SLA    │
│ (per-org sparklines)       │ │  (click → org-scoped Batches)  │ │ 3 sheds to reset│
└────────────────────────────┘ └────────────────────────────────┘ └────────────────┘
┌──────────────── Alert feed (cross-org, filter type/severity/org) ──────────────┐
│  time · org · type · batch/resource · severity · [Acknowledge] [Open]           │
└────────────────────────────────────────────────────────────────────────────────┘
```

| Panel | Content | Source |
|-------|---------|--------|
| **KPI row** | #Orgs, #active farmers, #active batches, birds on platform, avg FCR, alerts open | Cross-org aggregate of `/api/farmer/dashboard` slices |
| **What happened?** | Logs recorded today, batches placed, harvests completed, settlements closed (with per-org sparklines) | `dashboard.whatHappened` merged across orgs |
| **What needs attention today?** | Farmer alerts (§6.1) ranked by severity, each linking to the org-scoped page | `dashboard.attentionToday` + `/api/farmer/alerts` |
| **What next?** | Batches near harvest, settlements pending, sheds awaiting reset | `dashboard.whatNext` |
| **Alert feed** | Chronological cross-org alerts with acknowledge/open | `GET /api/farmer/alerts`, `POST /api/farmer/alerts/:id/ack` |

### 6.1 Alerts surfaced (farmer slice — MVP)

Mirror [07](07-Farmer-Module-Spec.md) §12.1: **mortality increase**, **low feed**,
**low medicine stock**, **vaccination due**, **harvest approaching**, **settlement
pending**. Rendered as `Badge`/`Tag` severity chips; clicking an alert deep-links
to the relevant org-scoped list page with filters pre-applied.

> Supplier alerts (new order, dispatch due, delivery confirmed, overdue invoice,
> payment received) and distributor alerts (new ready batch, purchase accepted,
> collection due, settlement pending/overdue) join the feed when those modules ship
> (§3, rows 13–15).

---

## 7. Component Patterns

### 7.1 Standard antd list page

Every MVP list page (§5) is the **same** composition — build it once as a generic
`<AdminListPage>` and configure per resource:

```
FilterBar (Org ▾ · resource filters · RangePicker · Search · Clear)
    │  filters → TanStack Query key
    ▼
Table  (server pagination + sort + column filters)
    │  row click → open Drawer
    ▼
Drawer (detail: Descriptions + Tabs)   ── action buttons ──▶ Modal/Form or Popconfirm
    │                                                              │
    └── mutation via TanStack Query ◀──── invalidate query keys ◀──┘
```

| Concern | Pattern |
|---------|---------|
| **Data** | TanStack Query per resource; query key = `[resource, orgScope, filters, page, sort]`. Server pagination reads `meta.page`/`meta.total` ([07](07-Farmer-Module-Spec.md) §0.2). |
| **Pagination/sorting** | antd `Table` `pagination` + `onChange` → query params `?page&pageSize&sort`; sort maps to `sort=field:dir`. |
| **Filtering** | Filter bar drives query params (`status`, `farmId`, `type`, date range). URL-synced (React Router search params) so views are shareable/bookmarkable. |
| **Create/edit** | `Modal` + `Form` (React Hook Form + Zod resolver). Zod schemas shared with services where available; on submit → mutation → `invalidateQueries` → `message.success`. |
| **Destructive/transition** | `Popconfirm` (or confirm `Modal`) before any audited action; disabled with tooltip when the service guard would reject (e.g. archive with active work). |
| **Empty state** | antd `Empty` with a resource-specific hint ("No batches for this filter — clear filters or pick another org"). |
| **Loading** | `Skeleton` for first load, `Spin`/table `loading` for refetch; keep prior data on filter change (TanStack `keepPreviousData`). |
| **Error** | Render the standard error envelope (`error.code`/`message`) in a `Result`/`Alert`; `403` → "read-only / not permitted" copy; retry button. |
| **Cross-org column** | Show **Org** column + filter when scope is `All organizations`; hide when a single org is pinned. |
| **Money/units** | Format from integer minor units + currency; weights in g/kg per [07](07-Farmer-Module-Spec.md) §0.3; never parse floats. |
| **Estimate badges** | Any value flagged `source: "estimate"` renders an inline `Tag` "est." ([07](07-Farmer-Module-Spec.md) §8). |
| **Audit surfacing** | Detail Drawers include an **Audit** tab reading `AuditLog` (actor, action, before/after) for that resource. |

### 7.2 Read-first discipline

MVP admin pages are **read-optimized**. The only write actions are the audited
support operations listed per page (org suspend/edit, user/role provisioning, farm
archive/activate, batch cancel, harvest visibility override). Everything else is
observation — the actors own their data through [09](09-Client-Apps-Spec.md), and
the console never bypasses a service state-machine guard.

---

## 8. Cross-References

- **Auth, super-admin, RBAC, roles/permissions:** [05-Auth-And-RBAC.md](05-Auth-And-RBAC.md)
- **Entities & derived metrics:** [06-Data-Model.md](06-Data-Model.md)
- **Farmer endpoints, state machines, guards (all data shown here):** [07-Farmer-Module-Spec.md](07-Farmer-Module-Spec.md)
- **Actor web/mobile apps (data producers):** [09-Client-Apps-Spec.md](09-Client-Apps-Spec.md)
- **Envelope, pagination, error shapes:** [12-API-Conventions.md](12-API-Conventions.md)

*End of 08 — Admin Panel Spec.*
