# 04 — Repository Structure

KukkuOne is a single monorepo managed with **pnpm workspaces + Turborepo**. The
workspace root is `KukkuOne/`. Applications live at the root as `kukkuone-*`
folders; shared code lives under `packages/` as `@kukkuone/*`.

Related docs: [Decision Record](00-Decision-Record.md) ·
[System Architecture](02-System-Architecture.md) ·
[Tech Stack](03-Tech-Stack.md)

---

## 1. Monorepo Tree

```
KukkuOne/                          # workspace root
├── pnpm-workspace.yaml
├── turbo.json
├── package.json                   # root scripts, devDeps, pnpm config
├── tsconfig.base.json
│
├── kukkuone-api-gateway/          # NestJS — auth verify, coarse RBAC, route, aggregate
├── kukkuone-farmer-api/           # NestJS — batches, daily logs, growth, shed (BUILT FIRST)
├── kukkuone-supplier-api/         # NestJS — orders, dispatch, invoices
├── kukkuone-distributor-api/      # NestJS — discovery, purchase, collection, settlement
│
├── kukkuone-database/             # hosts @kukkuone/db — Prisma schema, migrations, seed
│
├── kukkuone-admin-web/            # React + Vite + TS + AntD — primary admin panel
├── kukkuone-web/                   # ONE role-aware actor app (Farmer/Distributor/Supplier) — React + Vite + TS + AntD
│
├── kukkuone-mobile/               # ONE Expo/React Native app (replaces 3 mobile folders)
│
├── kukkuone-infrastructure/       # Docker Compose, local + deploy config
├── kukkuone-documentation/        # these docs
│
└── packages/
    ├── types/                     # @kukkuone/types      — DTOs, enums, Principal, envelope
    ├── db/                        # @kukkuone/db         — Prisma client (lives in kukkuone-database)
    ├── auth/                      # @kukkuone/auth       — AuthProvider iface + Firebase adapter
    ├── api-client/                # @kukkuone/api-client — typed client for web/mobile
    ├── config/                    # @kukkuone/config     — env load/validate
    └── ui/                        # @kukkuone/ui         — shared React/AntD components
```

> **`kukkuone-mobile` replaces the three scaffolded mobile folders.** The old
> per-role mobile scaffolds are removed; there is exactly one role-aware Expo app.

> **`kukkuone-database` hosts `@kukkuone/db`** — the single Prisma schema,
> migrations, and seed. All services import the client from here; there is one
> `DATABASE_URL`.

---

## 2. `pnpm-workspace.yaml`

```yaml
packages:
  - "kukkuone-api-gateway"
  - "kukkuone-farmer-api"
  - "kukkuone-supplier-api"
  - "kukkuone-distributor-api"
  - "kukkuone-database"
  - "kukkuone-admin-web"
  - "kukkuone-web"
  - "kukkuone-mobile"
  - "packages/*"
```

## 3. `turbo.json`

```json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["tsconfig.base.json", ".env"],
  "pipeline": {
    "dev": {
      "cache": false,
      "persistent": true,
      "dependsOn": ["^build"]
    },
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**", "build/**"]
    },
    "lint": {
      "outputs": []
    },
    "test": {
      "dependsOn": ["^build"],
      "outputs": ["coverage/**"]
    }
  }
}
```

Root scripts delegate to Turbo:

```jsonc
// package.json (root)
{
  "scripts": {
    "dev":   "turbo run dev",
    "build": "turbo run build",
    "lint":  "turbo run lint",
    "test":  "turbo run test"
  }
}
```

---

## 4. Apps & Packages

| Name | Kind | Purpose | Tech |
|------|------|---------|------|
| `kukkuone-api-gateway` | app | Auth verify, coarse RBAC, routing, dashboard aggregation | NestJS |
| `kukkuone-farmer-api` | app | Batches, daily logs, growth, harvest, shed lifecycle (built first) | NestJS + Prisma |
| `kukkuone-supplier-api` | app | Orders, accept, dispatch, invoices | NestJS + Prisma |
| `kukkuone-distributor-api` | app | Discovery, purchase, collection, settlement | NestJS + Prisma |
| `kukkuone-database` | app/pkg | Prisma schema, migrations, seed; hosts `@kukkuone/db` | Prisma + PostgreSQL |
| `kukkuone-admin-web` | app | Primary admin panel | React + Vite + TS + AntD |
| `kukkuone-web` | app | ONE role-aware actor app (Farmer/Distributor/Supplier) | React + Vite + TS + AntD |
| `kukkuone-mobile` | app | One role-aware mobile app (tabs by capability) | Expo / React Native |
| `kukkuone-infrastructure` | infra | Docker Compose, deploy config | Docker |
| `kukkuone-documentation` | docs | Developer docs | Markdown |
| `@kukkuone/types` | pkg | DTOs, enums, Principal, envelope | TypeScript |
| `@kukkuone/db` | pkg | Prisma client, schema access | Prisma |
| `@kukkuone/auth` | pkg | `AuthProvider` interface + Firebase adapter | TypeScript |
| `@kukkuone/api-client` | pkg | Typed client against the gateway | TypeScript |
| `@kukkuone/config` | pkg | Env loading/validation | TypeScript |
| `@kukkuone/ui` | pkg | Shared React/AntD components | React + AntD |

---

## 5. Import / Dependency Rules

- **Business code depends only on `@kukkuone/*` abstractions — never on vendor SDKs
  directly.** In particular, **Firebase is only ever imported inside `@kukkuone/auth`**;
  no service or app imports the Firebase SDK. Swapping the auth provider is an
  adapter + `AUTH_PROVIDER` env change, nothing more.
- Apps depend on packages; **packages never depend on apps.**
- Cross-service data access goes **through the gateway**, not service-to-service.
- All DB access goes through `@kukkuone/db`; no app instantiates its own Prisma
  schema.
- Shared DTOs/enums live in `@kukkuone/types`; do not redefine them per app.

```
apps  ─────depend on────▶  packages (@kukkuone/*)  ─────▶  vendor SDKs
(gateway, *-api, *-web, mobile)                    (Firebase only via @kukkuone/auth,
                                                    Prisma only via @kukkuone/db)
```

---

## 6. Farmer-First Layout, Supplier/Distributor Slot In

- The **farmer** vertical is built first: `kukkuone-farmer-api` +
  the farmer experience in `kukkuone-web` + farmer tabs in `kukkuone-mobile`, against the batch
  tables in `@kukkuone/db`.
- **Supplier** and **distributor** slot in **without refactor**: each is a sibling
  NestJS app fronted by the same gateway, sharing `@kukkuone/types`,
  `@kukkuone/auth`, `@kukkuone/db`, and `@kukkuone/api-client`. Adding them means
  adding a service folder + routes, not reworking the farmer core.
- New extension capabilities (buyers/processors, logistics, finance, marketplace,
  intelligence) follow the same slot-in pattern.

---

## 7. Naming Conventions

| Thing | Convention | Example |
|-------|-----------|---------|
| App / infra / docs folders | `kebab-case`, `kukkuone-` prefix | `kukkuone-farmer-api` |
| Shared packages | `@kukkuone/<name>` scope | `@kukkuone/auth` |
| Package folder under `packages/` | `kebab-case`, unprefixed | `packages/api-client` |
| Env vars | `UPPER_SNAKE_CASE` | `DATABASE_URL`, `AUTH_PROVIDER` |

See [Decision Record](00-Decision-Record.md),
[System Architecture](02-System-Architecture.md), and [Tech Stack](03-Tech-Stack.md).
