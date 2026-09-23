# 00 — Architecture Decision Record (Locked)

> This document records the **binding** technology and architecture decisions for the KukkuOne MVP. Every other document assumes these. Any change here ripples everywhere — treat changes as deliberate.

Status: **Locked for MVP** · Owner: Prakash (naptrixlabs) · Build order: **Farmer module first**

---

## D-0 · Product framing

- **Product**: KukkuOne — B2B Poultry Management SaaS ERP.
- **Actors**: Supplier, Farmer, Distributor. A single **Organization** may hold multiple **capabilities** (it can be a chick supplier *and* a feed supplier *and* more). The data model must never hard-code one org to one business type.
- **Central engine**: The **Farmer ERP** (farms → sheds → batches → daily lifecycle → harvest → settlement → next batch). Built first; Supplier and Distributor plug into the same architecture.
- **Git**: Not connected yet. All work is **local**. The architecture is built so that adding remotes/CI later changes nothing structural.

---

## D-1 · Repository & workspace — **pnpm + Turborepo monorepo**

One workspace root (`KukkuOne/`) ties every app and service together with `pnpm-workspace.yaml` + Turborepo. This keeps the per-service and per-web-app folders you already scaffolded, while giving shared types, one install, and one command to run everything.

**Why:** shared TypeScript types/contracts across gateway ↔ services ↔ web ↔ mobile without publishing packages; atomic changes across boundaries; one dependency graph; trivial to split into true multi-repo later (each folder is already self-contained).

**Rejected:** 14 fully independent repos (your original scaffold) — 3–5× the install/build/deploy/versioning overhead for a solo/small team with zero benefit at MVP scale. We keep the *folders*, drop the *repo fragmentation*.

---

## D-2 · Backend — **NestJS gateway + per-actor service APIs** (your choice, honored)

```
Clients ──► kukkuone-api-gateway (NestJS)
                 │  verifies auth, RBAC gate, routing, aggregation, rate-limit
                 ├──► kukkuone-farmer-api      (NestJS)   ← built first
                 ├──► kukkuone-supplier-api    (NestJS)
                 └──► kukkuone-distributor-api (NestJS)
```

- **Framework**: **NestJS (TypeScript)** for the gateway and every service. Modular, DI-based, first-class for domain modules, validation, guards, OpenAPI — the industry standard for TypeScript ERPs.
- **Gateway responsibilities**: terminate auth (verify Firebase ID token → internal principal), enforce coarse RBAC, route `/api/farmer/*`, `/api/supplier/*`, `/api/distributor/*`, `/api/core/*` to the right service, aggregate cross-service reads for dashboards, rate-limit, request logging/tracing.
- **Service responsibilities**: own their domain modules and business logic; trust the gateway-provided principal; expose internal REST.
- **Shared cross-cutting concerns** (auth verification contract, org/RBAC model, notifications, audit) live in **shared packages** consumed by all services — not duplicated.

**Why services behind a gateway (vs one monolith):** you explicitly want it, and it cleanly matches the three actors and your scaffold. We keep them **light** (shared DB, shared packages) so it stays MVP-affordable.

**Note on cost:** true microservices usually pay for independent DBs + infra. For the MVP we deliberately run **one shared database** (D-4) and **one deployable stack** so the gateway/service split is an *organizational* boundary, not an operational tax. Splitting DBs/deploys later is a documented, non-breaking step.

---

## D-3 · Web — **one role-aware client app + a separate admin console** (updated)

- **Framework**: **React + Vite + TypeScript**.
- **Two web apps only** (in the monorepo):
  - **`kukkuone-web`** — the **single role-aware actor app** for Farmers, Distributors, and Suppliers. Replaces the three separate `farmer-web` / `distributor-web` / `supplier-web` folders. Same capability-driven pattern as mobile (D-6): on login it reads the org's **capabilities + role** and renders the matching experience (Farmer first). Multi-capability orgs get an **actor switcher**.
  - **`kukkuone-admin-web`** — the **platform owner console** (for **naptrixlabs@gmail.com**): sees all data across every organization, operate/support. Kept separate because its audience and layout differ from the actor app.
- **Admin layout**: fixed **left sidebar** (module nav) + **top header** (org/user, search, actions) + **filter bar** + content area. Standard data-dense ERP shell.
- **UI kit**: **Ant Design (antd)** for both — ships dense tables, filters, forms, date pickers that an ERP lives on, fastest path to a usable console. (Alternative considered: Tailwind + shadcn/ui — kept as an option for a more custom actor UI.)
- **Data layer**: **TanStack Query** (server state) + **Zustand** (light UI state). **Forms**: React Hook Form + **Zod** (schemas shared from `@kukkuone/types`). **Routing**: React Router (capability route-groups mirror the mobile stacks).

---

## D-4 · Database — **PostgreSQL + Prisma, one shared instance**

- **Engine**: **PostgreSQL**. An ERP is relational and transactional to its core — batches, inventory movements, invoices, settlements, financial rollups. Postgres gives ACID transactions, constraints, JSONB for flexible metadata, and mature hosting everywhere.
- **ORM**: **Prisma** — typed client, first-class migrations, great DX; the generated types feed the shared type packages.
- **Topology for MVP**: **one Postgres instance, one `DATABASE_URL`**, partitioned by **Prisma schema / logical module** (`core`, `farmer`, `supplier`, `distributor`). Services import a shared `@kukkuone/db` package.
- **Future**: split to database-per-service when a service genuinely needs independent scaling. The module partitioning today makes that a clean cut.

**Rejected:** MongoDB / document store — poor fit for the financial + relational integrity an ERP settlement loop demands.

---

## D-5 · Auth — **pluggable layer, Firebase Google Auth default, swappable**

This is a first-class requirement: you must be able to add/remove/replace the **entire** auth layer later.

- **Contract**: an `AuthProvider` interface (in `@kukkuone/auth`) defines `verifyToken(token) → Principal`, `getUser()`, `signIn()/signOut()` (client), and identity mapping. Business code depends **only** on the interface + the internal `Principal`, never on Firebase types.
- **Default provider**: **Firebase Authentication** with **Google sign-in**. Client SDK obtains a Firebase **ID token**; gateway verifies it via Firebase Admin SDK, maps `firebase_uid → User → Organization/roles`, and issues the internal principal for downstream services.
- **Swap strategy**: switching to Auth0 / Cognito / custom JWT = implement one new adapter + change `AUTH_PROVIDER` env var. Zero changes to business modules. RBAC and the `User`/`Organization` tables are **provider-agnostic** (`firebase_uid` is just one nullable external-identity column among possible others).
- **Admin panel default login**: **naptrixlabs@gmail.com** is the seeded super-admin (platform owner) with full access. Configured via env allowlist, not hard-coded in logic.
- **RBAC**: org-scoped roles + permission actions (`View, Create, Edit, Approve, Dispatch, Settle, Report`) per the PRD. Enforced at the gateway (coarse) and service (fine) layers.

Full design in **[05-Auth-And-RBAC](05-Auth-And-RBAC.md)**.

---

## D-6 · Mobile — **one Expo app, role-aware** (your choice + your question answered)

- **Stack**: **Expo (React Native + TypeScript)**, **Expo Router** (file-based nav), **tab layout**. Data: TanStack Query + Zustand. UI: React Native Paper (Material) — stable and fast for forms/tables. Secure token storage via `expo-secure-store`.
- **How the three actors differ inside ONE app** (your question): on sign-in the app fetches the user's **Organization capabilities** (`Supplier`/`Farmer`/`Distributor`, per PRD §5) and **role**. A capability→navigation map renders the matching **tab set**:
  - Farmer capability → tabs: *Home, Batches, Daily Log, Harvest, More* (built first).
  - Distributor capability → tabs: *Home, Discover, Purchases, Settlements, More*.
  - Supplier capability → tabs: *Home, Orders, Dispatch, Invoices, More*.
  - An org with **multiple** capabilities gets an **actor switcher** (header segmented control / drawer) to flip contexts; each actor mode is a self-contained navigation stack under `app/(farmer)`, `app/(distributor)`, `app/(supplier)`.
- Shared screens (auth, org, profile, notifications) live in a common area; only actor stacks differ. Maximizes reuse, one app to ship. Full design in **[09-Client-Apps-Spec](09-Client-Apps-Spec.md)**.

**Note:** the three scaffolded mobile folders (`kukkuone-farmer-mobile`, `-distributor-mobile`, `-supplier-mobile`) are **replaced by one** `kukkuone-mobile` app.

---

## D-6b · Signup & capability selection — **the user chooses who they are** (new)

Both `kukkuone-web` and `kukkuone-mobile` share **one identity + onboarding flow**:

1. User signs in with **Google** (Firebase).
2. First-time users complete **onboarding**: name their **Organization** and select one or more **capabilities** — **Farmer, Distributor, Supplier** — *a user can select any, several, or **all***.
3. The selected capabilities are stored on the **Organization** (`capabilities[]`) and drive the UI: the app renders the tab/nav set for each selected capability. **Multiple selected → actor switcher** to flip contexts.
4. Roles within each capability (per PRD §5) can be assigned afterward by the org owner.

Consequences:
- **One codebase per platform** (one web app, one mobile app), experience decided at runtime from `capabilities + role` — no separate builds per actor.
- The **admin console** (`kukkuone-admin-web`) is **not** part of this flow; it is gated to the platform super-admin allowlist (naptrixlabs@gmail.com).
- Capabilities are **editable later** (an org can add "Distributor" to an existing "Farmer" account) without a new account.

Full flow in **[05-Auth-And-RBAC](05-Auth-And-RBAC.md)**; UI in **[09-Client-Apps-Spec](09-Client-Apps-Spec.md)**.

---

## D-7 · Config & hosting — **env-driven, one connection, host-anywhere**

- **Everything configurable via environment variables.** No secrets, URLs, or keys in code. Each app ships a committed `.env.example`; real values live in `.env` (git-ignored) and in the host's env panel in production.
- **One connection philosophy**: a single `DATABASE_URL`, a single gateway `API_BASE_URL` consumed by every client, a single Firebase config block. Point these at local or at any cloud — nothing else changes.
- **Local**: **Docker Compose** brings up Postgres + gateway + services + web. Mobile runs via Expo against the gateway URL.
- **Host-anywhere**: the same images/build deploy to **Railway / Render / Fly.io / any VPS / any Kubernetes** by supplying the same env vars. No provider lock-in.

Full detail in **[10-Environment-And-Config](10-Environment-And-Config.md)** and **[11-Local-Setup-And-Deploy](11-Local-Setup-And-Deploy.md)**.

---

## D-8 · Shared packages

| Package | Purpose |
|---------|---------|
| `@kukkuone/types` | Shared TypeScript types + **Zod** schemas (DTOs, enums, statuses) used by services, web, mobile |
| `@kukkuone/db` | Prisma schema, client, migrations, seed |
| `@kukkuone/auth` | `AuthProvider` contract + Firebase adapter (client + server) + `Principal`/RBAC types |
| `@kukkuone/api-client` | Typed HTTP client wrapping the gateway, shared by web + mobile |
| `@kukkuone/config` | Env loading/validation (Zod-validated env) |
| `@kukkuone/ui` (optional) | Cross-web shared components/theme |

---

## Evaluation of your original suggestions

| Your suggestion | Verdict | Notes |
|-----------------|---------|-------|
| Mobile = **React Native** | ✅ Adopted (as **Expo**) | Expo speeds builds, OTA updates, and native modules. One role-aware app instead of three. |
| Web = **React** | ✅ Adopted | React + Vite + TypeScript + Ant Design. Separate app per actor (your call). |
| Backend/DB = *"you suggest"* | ➡️ **NestJS + PostgreSQL + Prisma** | Best-fit TypeScript ERP stack; shares types with your React/RN front-ends; matches your existing NestJS/Postgres experience. |
| Auth = **Firebase Google**, swappable | ✅ Adopted, abstracted | Firebase default behind an `AuthProvider` interface; swap = one adapter + one env var. |
| Gateway calling per-actor APIs | ✅ Adopted | Gateway + farmer/supplier/distributor services, kept light on a shared DB for MVP affordability. |
| Web app structure | ↩️ Updated | **One** role-aware `kukkuone-web` (Farmer/Distributor/Supplier) + separate `kukkuone-admin-web` owner console. Supersedes the earlier per-actor web plan. |
| Signup capability selection | ✅ Added | User picks capabilities (any/all) at onboarding; app decides experience. |
| Admin sidebar/header/filter layout | ✅ Adopted | Standard ERP shell in admin-web. |

---

## Decision quick-reference

- Monorepo: **pnpm + Turborepo**
- Backend: **NestJS** (gateway + farmer/supplier/distributor services)
- DB: **PostgreSQL + Prisma**, one shared instance, one `DATABASE_URL`
- Auth: **pluggable**, default **Firebase Google**, swap by adapter + env
- Admin super-user: **naptrixlabs@gmail.com**
- Web: **React + Vite + TS + Ant Design** — **one role-aware `kukkuone-web`** + separate **`kukkuone-admin-web`** console
- Mobile: **one Expo app**, tab layout, role-aware by org capability
- Onboarding: user **selects capabilities (any/all)** at signup → app decides experience (web + mobile share it)
- Config: **env-driven**, one connection, Docker Compose local, host-anywhere
- Build order: **Farmer first**, then Distributor, then Supplier
