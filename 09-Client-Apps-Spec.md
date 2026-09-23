# 09 — Client Apps Spec

The **actor apps** — one web app and one mobile app — serve all three actors
(**Farmer / Distributor / Supplier**) from a **single codebase each**. The app
does not fork per actor; it renders the matching navigation, screens, and tabs at
runtime from the org's **capabilities** + the user's **role**. The **Farmer**
experience is built **first**.

**Apps:**

| App | Stack |
|-----|-------|
| `kukkuone-web` | React + Vite + TypeScript + **Ant Design**, TanStack Query, Zustand, React Hook Form + Zod, React Router |
| `kukkuone-mobile` | Expo / React Native + **Expo Router**, **React Native Paper**, **tab layout**, TanStack Query, Zustand, `expo-secure-store`, React Hook Form + Zod |

Both consume the shared packages **`@kukkuone/api-client`** (typed calls to the
gateway `/api/*`) and **`@kukkuone/types`** (`Principal`, `Capability`, entity
types) and share **Zod** validation schemas — so a screen written once validates
and calls identically on web and mobile.

> The platform-owner console `kukkuone-admin-web` is a **separate** app; see
> [08-Admin-Panel-Spec.md](08-Admin-Panel-Spec.md).

Related docs: [Product Overview](01-Product-Overview.md) ·
[Auth & RBAC](05-Auth-And-RBAC.md) · [Data Model](06-Data-Model.md) ·
[Farmer Module Spec](07-Farmer-Module-Spec.md) ·
[Admin Panel Spec](08-Admin-Panel-Spec.md) ·
[API Conventions](12-API-Conventions.md) ·
[Roadmap & Milestones](13-Roadmap-And-Milestones.md)

---

## 1. Shared Pattern — One App, Three Actors

On login the client fetches the org's **capabilities** and the user's **roles**
(the `Principal`, [05](05-Auth-And-RBAC.md) §2.1) and renders the matching nav set.
Route/tab **groups** are shared or per-capability; the active capability selects
which per-capability group is mounted.

```
                         ┌──────────────────────────────┐
   Google sign-in  ──▶   │  Resolve Principal            │
                         │  capabilities[] + roles[]     │
                         └───────────────┬──────────────┘
                                         │
              ┌──────────────────────────┼──────────────────────────┐
              │ one capability           │ many capabilities         │
              ▼                          ▼                           │
     Render that experience     Actor switcher (Farmer/Distributor/  │
     directly                   Supplier) sets active capability ────┘
                                         │
                                         ▼
        ┌──────────────── Route / Tab groups ─────────────────┐
        │ SHARED            (auth) (onboarding) (profile)      │
        │                   (notifications)                    │
        ├──────────────────────────────────────────────────────┤
        │ PER-CAPABILITY    (farmer)      ← built first         │
        │  (mounted by      (distributor)                      │
        │  active cap)      (supplier)                         │
        └──────────────────────────────────────────────────────┘
```

| Group | Purpose | Rendered when |
|-------|---------|---------------|
| `(auth)` | Google sign-in, token bootstrap | Unauthenticated |
| `(onboarding)` | Create org + capability multi-select ([05](05-Auth-And-RBAC.md) §4) | Authenticated, no org |
| `(profile)` | User profile, org settings, **manage capabilities**, actor switcher | Always available |
| `(notifications)` | Alert feed + acknowledge (all capabilities merge here) | Always available |
| `(farmer)` | Farmer ERP screens (§5) | Active capability = FARMER |
| `(distributor)` | Distributor screens (§6) | Active capability = DISTRIBUTOR |
| `(supplier)` | Supplier screens (§6) | Active capability = SUPPLIER |

**Actor switcher.** Multi-capability orgs get a switcher (mobile: header control /
"More" tab; web: header `Select` or left-nav section switch) that sets the active
capability in a Zustand store, which sets `Principal.capabilities` context for the
session and swaps the mounted per-capability group. Single-capability orgs (the
common Farmer-only case) never see it. **Role** further gates individual
actions/screens within a capability (e.g. Accountant sees Finance; Operator does
not Approve) per the RBAC matrix ([05](05-Auth-And-RBAC.md) §6.5,
[07](07-Farmer-Module-Spec.md) §0.1).

---

## 2. Onboarding UX

```mermaid
flowchart TD
  A[Google sign-in] --> B{Known identity + org?}
  B -- yes --> J[Route to matching experience]
  B -- no org --> C[Create Organization<br/>name + base currency]
  C --> D[Capability multi-select<br/>Farmer ▢  Distributor ▢  Supplier ▢<br/>any / all]
  D --> E[Persist Organization.capabilities[]<br/>assign OWNER per capability]
  E --> F{How many?}
  F -- one --> G[Land in that experience]
  F -- many --> H[Land + actor switcher]
```

- **Sign in** with Google (Firebase default; provider-agnostic, [05](05-Auth-And-RBAC.md) §3).
- **Create org** — name + base currency.
- **Capability multi-select** — pick **any/all** of Farmer / Distributor / Supplier
  (checkbox cards with a one-line description each). At least one required.
- On confirm the app persists `Organization.capabilities[]`, assigns the **Owner**
  role per selected capability, and routes: one → that experience; many → experience
  + actor switcher.
- **Editable later**: an Owner adds/removes capabilities in `(profile)` → Org
  settings; removal is soft (history retained, experience hidden) ([05](05-Auth-And-RBAC.md) §4 step 8).

Since the Farmer ERP ships first, **Farmer-only** is the primary onboarding path and
lands directly on the Farmer Home ([07](07-Farmer-Module-Spec.md) §12).

---

## 3. Mobile — `kukkuone-mobile`

**Tab layout** (React Native Paper bottom navigation), per-capability tab set. Each
capability's group has its own `_layout.tsx` declaring 5 tabs; the last tab is
always **More** (secondary screens + actor switcher + profile).

| Capability | Tabs (bottom nav) | Status |
|------------|-------------------|--------|
| **Farmer** | Home · Batches · Daily Log · Harvest · More | **Built first** |
| **Distributor** | Home · Discover · Purchases · Settlements · More | Later |
| **Supplier** | Home · Orders · Dispatch · Invoices · More | Later |

### 3.1 Expo Router structure

```
app/
  _layout.tsx                 # root: auth gate, Principal load, providers (Query/Zustand)
  (auth)/
    sign-in.tsx
  (onboarding)/
    create-org.tsx
    select-capabilities.tsx
  (common)/                   # shared, reachable from any capability
    (profile)/
      index.tsx               # profile
      org-settings.tsx        # manage capabilities, actor switcher
    (notifications)/
      index.tsx               # alert feed  → /api/farmer/alerts (+ others later)
  (farmer)/                   # ← built first
    _layout.tsx               # Tabs: Home | Batches | Daily Log | Harvest | More
    home.tsx                  # dashboard (§5.1)
    batches/
      index.tsx               # list (§5.2)
      [batchId].tsx           # detail w/ tabs (§5.3)
      place.tsx               # Day-0 placement wizard (§5.4)
    daily-log/
      index.tsx               # pick batch → today's log
      [batchId].tsx           # fast entry (§5.5)
    feed/index.tsx            # (§5.6)
    health/index.tsx          # vaccinations + medicine + events (§5.7)
    harvest/index.tsx         # near-harvest + visibility (§5.8)
    more.tsx                  # feed/health/finance/settlement entries + switcher
  (distributor)/
    _layout.tsx               # Tabs: Home | Discover | Purchases | Settlements | More
    …                         # (§6.1)
  (supplier)/
    _layout.tsx               # Tabs: Home | Orders | Dispatch | Invoices | More
    …                         # (§6.2)
```

The root `_layout` mounts exactly one per-capability group based on the active
capability; the switcher (in `(common)/(profile)/org-settings`) changes it.

---

## 4. Web — `kukkuone-web`

Same capability groups as the mobile app, rendered as an Ant Design shell. The
**per-capability nav** is a left `Menu` (or top tabs on narrow layouts); the actor
switcher is a header `Select` for multi-capability orgs.

```
┌───────────── Header: [KukkuOne] [Farm switcher ▾] [Actor: Farmer ▾] 🔔 👤 ──────┐
├───────────┬────────────────────────────────────────────────────────────────────┤
│ Left Menu │  Route group = active capability                                    │
│ (farmer)  │  /farmer/home · /farmer/batches · /farmer/batches/:id ·             │
│  Home     │  /farmer/batches/place · /farmer/daily-log · /farmer/feed ·         │
│  Batches  │  /farmer/health · /farmer/harvest · /farmer/finance                 │
│  DailyLog │                                                                     │
│  Feed     │  Shared: /profile · /notifications · /onboarding · /auth            │
│  Health   │                                                                     │
│  Harvest  │                                                                     │
│  Finance  │                                                                     │
└───────────┴────────────────────────────────────────────────────────────────────┘
```

- React Router route groups mirror the mobile groups (`(auth)`, `(onboarding)`,
  `(profile)`, `(notifications)`, `(farmer)`, `(distributor)`, `(supplier)`).
- **Farm switcher** in the header (farmer capability) scopes farm-specific views
  ([07](07-Farmer-Module-Spec.md) §1.3).
- Shares `@kukkuone/api-client` + `@kukkuone/types` + Zod schemas with mobile — the
  screen→endpoint mapping in §5 is identical across both; mobile is the **primary
  data-entry** surface, web adds bulk/desk workflows (Manager/Accountant).

---

## 5. Farmer Screens (built first) — Web + Mobile

Each screen lists purpose + primary endpoints (all `/api/farmer/*` from
[07](07-Farmer-Module-Spec.md)). RBAC per role from [07](07-Farmer-Module-Spec.md)
§0.1 / [05](05-Auth-And-RBAC.md) §6.5.

### 5.1 Home / Dashboard

**Purpose.** Answer the three guiding questions ([01](01-Product-Overview.md) §5):
**What happened / What needs attention today / What next** — three panels + alert
feed + KPI tiles. Alerts: mortality increase, low feed, low medicine, vaccination
due, harvest approaching, settlement pending. Each alert/next-action deep-links to
the relevant screen.

```
┌──── KPIs: Active batches · Birds on farm · Avg FCR ────┐
├─ What happened ─┬─ Attention today ─┬─ What next ──────┤
│ logs today: 3   │ 🔴 Vaccination due │ Mark ready B-02  │
│ mortality 0.18% │ 🟠 Low feed FINISH │ Open settlement  │
│ feed 1240kg     │  (tap → screen)    │  B-03            │
└─────────────────┴────────────────────┴──────────────────┘
   Alert feed (ack / open) ────────────────────────────────
```

| Screen → | Endpoint | Role |
|----------|----------|------|
| Dashboard aggregate | `GET /api/farmer/dashboard?farmId=` | Farm Operator |
| Alert feed | `GET /api/farmer/alerts` (filter type/severity/status) | Farm Operator |
| Acknowledge alert | `POST /api/farmer/alerts/:id/ack` | Farm Operator |

### 5.2 Batches — List

**Purpose.** All batches, **active first**, with status chip, age, current birds,
mortality %, FCR, target harvest. Filter by status/farm/shed. Tap → detail; FAB /
button → Day-0 placement.

| Screen → | Endpoint | Role |
|----------|----------|------|
| List batches | `GET /api/farmer/batches?status=&farmId=&shedId=` | Farm Operator |
| New (placement) | → §5.4 | Farm Manager |

### 5.3 Batch — Detail

**Purpose.** Per-batch console with tabs **Overview / Daily / Feed / Health /
Growth / Finance / Harvest**. Overview shows lifecycle (`PLANNED`…`CLOSED`),
performance summary, and finance summary; tabs deep-link into §5.5–§5.8/§5.9.
Lifecycle actions (activate, ready-for-harvest, start-harvest, record-sale,
settlement, close) are surfaced by role/guard ([07](07-Farmer-Module-Spec.md) §3.2,
§11).

| Screen → | Endpoint | Role |
|----------|----------|------|
| Batch header + counts | `GET /api/farmer/batches/:batchId` | Farm Operator |
| Performance summary (Overview/Growth) | `GET /api/farmer/batches/:batchId/performance` | Farm Operator |
| Growth trend charts | `GET /api/farmer/batches/:batchId/growth-trend` | Farm Operator |
| Finance summary (Finance tab) | `GET /api/farmer/batches/:batchId/finance-summary` | Accountant |
| Activate | `POST /api/farmer/batches/:batchId/activate` | Farm Operator |
| Mark ready for harvest | `POST /api/farmer/batches/:batchId/ready-for-harvest` | Farm Manager |
| Start harvest | `POST /api/farmer/batches/:batchId/start-harvest` | Farm Manager |
| Record sale | `POST /api/farmer/batches/:batchId/record-sale` | Farm Manager |
| Open / record settlement | `POST /api/farmer/batches/:batchId/settlement/open` · `POST …/settlement` | Accountant |
| Close batch | `POST /api/farmer/batches/:batchId/close` | Farm Manager |

### 5.4 Day-0 Placement Wizard

**Purpose.** Guided stepper that creates a live batch at Day 0
([07](07-Farmer-Module-Spec.md) §4). **Mobile is the primary surface** (incl.
scan/select a received supplier order). Atomic commit on Confirm.

**Ordered steps** (matches [07](07-Farmer-Module-Spec.md) §4.1):

```
1 Select farm (ACTIVE)      → 2 Select shed (READY)     → 3 Chick supplier
4 Chick product / breed     → 5 Quantity (≤ capacity)   → 6 Placement date (Day 0)
7 Arrival weight (opt)      → 8 Initial feed (opt)       → 9 Vaccination/medicine (opt)
10 Review & Confirm ─────────────────────────────────────▶ atomic create
```

| Screen → | Endpoint | Role |
|----------|----------|------|
| Farm picker (step 1) | `GET /api/farmer/farms?status=ACTIVE` | Farm Manager |
| Shed picker (step 2) | `GET /api/farmer/farms/:farmId/sheds?status=READY` | Farm Manager |
| (Optional) linked order lookup | supplier order ref (`linkedInputOrderId`), read via supplier surface | Farm Manager |
| Confirm placement (steps 1–10) | `POST /api/farmer/batches/place` | Farm Manager |

Client validates step-by-step with the shared Zod schema (qty ≤ shed capacity,
date ≤ today ≥ shed readyAt); server returns `422 SHED_NOT_READY` /
`SHED_CAPACITY_EXCEEDED` on guard failure.

### 5.5 Daily Log Entry

**Purpose.** The operator's **once-per-day** record — fast data entry. Opening
birds prefilled (prior day's closing); sample-weight calculator; feed/water; health;
environment; attachments. Closing birds & mortality % auto-computed. One log per
`(batchId, logDate)` — a second create returns `409 DAILY_LOG_EXISTS` (client edits
instead). **Offline capture on mobile**: drafts saved locally, synced when online
(§7). First log auto-advances a `PLACED` batch to `ACTIVE`.

| Screen → | Endpoint | Role |
|----------|----------|------|
| List logs (timeline) | `GET /api/farmer/batches/:batchId/daily-logs?from=&to=` | Farm Operator |
| Get a day's log | `GET /api/farmer/batches/:batchId/daily-logs/:date` | Farm Operator |
| Create today's log | `POST /api/farmer/batches/:batchId/daily-logs` | Farm Operator |
| Edit within window | `PATCH /api/farmer/batches/:batchId/daily-logs/:date` | Farm Operator |

### 5.6 Feed

**Purpose.** Assign feed consumption to a batch+date, view inventory balances and
per-batch feed cost + FCR; low-feed alert; receive stock. Quantities in **bags +
kg**.

| Screen → | Endpoint | Role |
|----------|----------|------|
| Inventory balances | `GET /api/farmer/feed/inventory?farmId=&feedType=` | Farm Operator |
| Purchases list | `GET /api/farmer/feed/purchases` | Farm Operator |
| Record purchase | `POST /api/farmer/feed/purchases` | Farm Manager |
| Receive stock | `POST /api/farmer/feed/purchases/:id/receive` | Farm Operator |
| Assign consumption | `POST /api/farmer/feed/consumption` | Farm Operator |
| Batch feed usage + FCR | `GET /api/farmer/batches/:batchId/feed` | Farm Operator |

### 5.7 Health / Vaccination

**Purpose.** Vaccination checklist with **due badges** and one-tap complete;
medicine usage entry; report a health event. Vaccination status
`Scheduled → Due → Completed` ([07](07-Farmer-Module-Spec.md) §7.1).

| Screen → | Endpoint | Role |
|----------|----------|------|
| Medicine balances | `GET /api/farmer/medicine/inventory` | Farm Operator |
| Record medicine usage | `POST /api/farmer/medicine/usage` | Farm Operator |
| Batch vaccination schedule | `GET /api/farmer/batches/:batchId/vaccinations` | Farm Operator |
| Schedule a vaccination | `POST /api/farmer/batches/:batchId/vaccinations` | Farm Manager |
| Mark vaccination completed | `POST /api/farmer/vaccinations/:id/complete` | Farm Operator |
| List health events | `GET /api/farmer/batches/:batchId/health-events` | Farm Operator |
| Record health event | `POST /api/farmer/batches/:batchId/health-events` | Farm Operator |

### 5.8 Harvest

**Purpose.** "Near harvest" list, **mark READY_FOR_HARVEST**, and a **visibility
toggle** (`HIDDEN`/`NETWORK`) with a confirmation of exactly what distributors will
see ([07](07-Farmer-Module-Spec.md) §10). `NETWORK` + `READY_FOR_HARVEST` exposes
the batch to distributor discovery.

| Screen → | Endpoint | Role |
|----------|----------|------|
| Near-harvest list | `GET /api/farmer/harvest/eligible` | Farm Manager |
| Harvest snapshot | `GET /api/farmer/batches/:batchId/harvest-info` | Farm Operator |
| Mark ready for harvest | `POST /api/farmer/batches/:batchId/ready-for-harvest` | Farm Manager |
| Set visibility HIDDEN/NETWORK | `PATCH /api/farmer/batches/:batchId/visibility` | Farm Manager |

> Sheds & settlement reset (clean → disinfect → ready) surface in the batch detail /
> More tab via the shed endpoints ([07](07-Farmer-Module-Spec.md) §2.2, §11.1).

---

## 6. Distributor & Supplier Tab Sets (outline)

Enough to slot in when the distributor/supplier services land (endpoints per those
specs; entities in [06](06-Data-Model.md) §5–§6). Alerts per the shared context.

### 6.1 Distributor — Home · Discover · Purchases · Settlements · More

| Tab | Screen (1–2 lines) |
|-----|--------------------|
| **Home** | Dashboard: three guiding questions + distributor alerts (new ready batch, purchase accepted, collection due, settlement pending/overdue). |
| **Discover** | Feed of `NETWORK` + `READY_FOR_HARVEST` batches (from farmer harvest visibility); filter by area/weight/age; open a batch to raise a purchase. |
| **Purchases** | Purchase list (`Purchase` [06](06-Data-Model.md) §6); create/track; record `PurchaseActuals` at collection (birds, rejects, live weight, adjustments, final). |
| **Settlements** | `DistributorSettlement` list: payable/paid/balance, record outcome + reference; status chips. |
| **More** | Farmer relationships, profile, actor switcher, notifications. |

### 6.2 Supplier — Home · Orders · Dispatch · Invoices · More

| Tab | Screen (1–2 lines) |
|-----|--------------------|
| **Home** | Dashboard: three guiding questions + supplier alerts (new order, dispatch due, delivery confirmed, overdue invoice, payment received). |
| **Orders** | `SupplierOrder` list from farmers; accept/manage lines (`OrderLine`); status pipeline. |
| **Dispatch** | Create/track `Dispatch` (vehicle, driver, date, status); mark shipped → feeds farmer Day-0 receipt handoff. |
| **Invoices** | `Invoice` + `Payment`: amount/paid/balance, record payments, overdue flags. |
| **More** | Products & inventory (`Product`/`Inventory`), profile, actor switcher, notifications. |

---

## 7. UX Conventions

| Concern | Convention |
|---------|-----------|
| **Fast daily entry** | Daily log (§5.5) minimizes taps: prefilled opening birds, numeric keypads, sample calculator, sensible defaults, one screen. Mobile is the primary entry surface. |
| **Secure token storage** | Mobile: ID/refresh token in `expo-secure-store` (Keychain/Keystore). Web: access token **in memory**, silent refresh; never `localStorage` ([05](05-Auth-And-RBAC.md) §9). |
| **Offline draft (daily logs)** | Mobile caches a daily-log draft locally (per `batchId`+date); create/patch is queued and retried on reconnect; `409 DAILY_LOG_EXISTS` resolves to an edit of the existing day. Attachments captured offline and uploaded on sync. |
| **Optimistic updates** | TanStack Query mutations apply optimistic cache updates for ack/complete/consume actions with rollback on error and `invalidateQueries` on settle. |
| **Form validation** | React Hook Form + **shared Zod schemas** (`@kukkuone/*`), identical on web & mobile; mirror server guards so client rejects before the round-trip (qty ≤ capacity, mortality+culls ≤ opening birds, money in minor units). |
| **Money & units** | Integer **minor units** + currency; feed in bags + kg; weights in grams — never floats ([07](07-Farmer-Module-Spec.md) §0.3). |
| **Estimate badges** | Any `source: "estimate"` value (projected weight, FCR before sold weight) shows an inline "est." badge ([07](07-Farmer-Module-Spec.md) §8). |
| **Role-gated UI** | Actions render per role/permission (Operator no Approve/Settle; Accountant owns Finance); hidden-or-disabled with a reason, and re-enforced server-side ([05](05-Auth-And-RBAC.md) §6). |
| **Error envelope** | Render `error.code`/`message` from the standard envelope ([12](12-API-Conventions.md)); map guard codes (`SHED_NOT_READY`, `DAILY_LOG_LOCKED`, `BATCH_INVALID_TRANSITION`) to friendly copy. |
| **Deep links** | Dashboard alerts / next-actions deep-link to the target screen with context (batchId, farmId). |

---

## 8. Cross-References

- **Capabilities, roles, onboarding, actor switcher, token storage:** [05-Auth-And-RBAC.md](05-Auth-And-RBAC.md)
- **Farmer endpoints, state machines, guards (all farmer screens):** [07-Farmer-Module-Spec.md](07-Farmer-Module-Spec.md)
- **Entities (supplier/distributor tab data):** [06-Data-Model.md](06-Data-Model.md)
- **Platform-owner console (separate app):** [08-Admin-Panel-Spec.md](08-Admin-Panel-Spec.md)
- **Envelope, pagination, error shapes, auth header:** [12-API-Conventions.md](12-API-Conventions.md)

*End of 09 — Client Apps Spec.*
