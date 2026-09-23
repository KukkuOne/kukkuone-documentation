# 03 — Tech Stack

This document is the authoritative inventory of **every** technology used in KukkuOne, why it was chosen, and what was considered instead. It is descriptive of the _locked_ stack — for the reasoning trail and trade-offs, see [00-Decision-Record.md](./00-Decision-Record.md). For how these pieces are wired together at runtime, see [02-System-Architecture.md](./02-System-Architecture.md).

> Version policy: KukkuOne pins **current-stable** major versions and tracks patch/minor releases within them. Where a range is shown (`5.x`), the lockfile (`pnpm-lock.yaml`) is the source of truth for the exact resolved version.

---

## 1. Full technology matrix

### 1.1 Platform & tooling

| Layer | Technology | Version | Role | Why chosen | Alternative considered |
|---|---|---|---|---|---|
| Runtime | Node.js | 20 LTS | JS/TS runtime for all backend services, web build tooling, and Expo tooling | Single runtime across the whole monorepo; LTS = long support window; native `fetch`, test runner, and stable ESM | Node 18 LTS (older, shorter support), Bun (fast but immature ecosystem for NestJS/Prisma) |
| Language | TypeScript | 5.x (`strict`) | One language, frontend to backend; shared types via `@kukkuone/types` | End-to-end type safety across gateway, services, web, mobile, and shared packages; `strict` catches nulls/anys early | Plain JavaScript (no safety), Flow (dead ecosystem) |
| Package manager | pnpm | 9.x | Dependency install + workspace linking | Content-addressable store = fast, disk-efficient; first-class workspaces; strict `node_modules` prevents phantom deps | npm workspaces (slower, looser hoisting), Yarn Berry (PnP friction with tooling) |
| Monorepo orchestration | Turborepo | 2.x | Task graph, caching, parallel `build`/`dev`/`lint`/`test` across apps & packages | Incremental caching + remote-cache-ready; simple `turbo.json` pipeline; pairs naturally with pnpm workspaces | Nx (heavier, more opinionated), Lerna (legacy), raw npm scripts (no caching) |

### 1.2 Backend

| Layer | Technology | Version | Role | Why chosen | Alternative considered |
|---|---|---|---|---|---|
| Backend framework | NestJS | 10 | API Gateway + per-actor services (farmer/supplier/distributor) | Opinionated modular DI, guards/interceptors for RBAC, first-class OpenAPI + testing; scales from monolith-ish to microservices | Express (too unstructured), Fastify raw (less batteries), Spring/Go (off the TS path) |
| API style | REST + OpenAPI/Swagger | OpenAPI 3.1 (`@nestjs/swagger` 7.x) | HTTP contract between clients and gateway; auto-generated docs | Universally consumable by web + mobile; Swagger UI is free documentation and a test console; codegen-ready | GraphQL (overkill for CRUD-first ERP; caching/RBAC complexity), gRPC (poor browser story) |
| Validation (shared) | Zod | 3.x | Single source of truth for DTO shapes + types in `@kukkuone/types` | Schema **is** the type (`z.infer`); reused verbatim on web, mobile, and service edges | Yup (weaker inference), io-ts (verbose), JSON Schema by hand |
| Validation (service edge) | class-validator + class-transformer | 0.14.x / 0.5.x | Runtime validation of incoming DTOs at the NestJS boundary | Native NestJS `ValidationPipe` integration; decorator DTOs map cleanly to Swagger | Zod-only pipe (viable; we keep both — Zod for shared contracts, class-validator at the HTTP edge) |
| Database | PostgreSQL | 16 | Single shared relational store, one instance, one `DATABASE_URL` | ACID, rich types (JSONB, arrays, enums), battle-tested for ERP/money data; one DB keeps ops simple | MySQL (weaker types), MongoDB (relational ERP wants joins/constraints) |
| ORM | Prisma | 5.x | Typed schema, migrations, and query layer in `@kukkuone/db` | Type-safe client generated from schema; declarative migrations; introspection; great DX | TypeORM (runtime-heavy, flaky migrations), Knex (no types), Drizzle (younger) |
| Auth SDK (server) | firebase-admin | 12.x | Verifies Firebase ID tokens at the gateway; mints custom claims for roles | Server-side token verification without rolling our own crypto; pairs with client Firebase | Auth0 SDK, custom JWT issuer (more to secure/maintain) |
| Auth abstraction | `AuthProvider` (in-house, `@kukkuone/auth`) | 1.0 | Pluggable interface; default Firebase adapter, swap via `AUTH_PROVIDER` env | Vendor independence — swap Firebase for Auth0/Cognito/custom by writing one adapter | Hard-coding Firebase everywhere (lock-in) |

### 1.3 Web (kukkuone-web + kukkuone-admin-web)

| Layer | Technology | Version | Role | Why chosen | Alternative considered |
|---|---|---|---|---|---|
| UI library | React | 18 | Component model for both web apps (kukkuone-web + kukkuone-admin-web) | Largest ecosystem; concurrent features; shared knowledge with React Native | Vue, Svelte, Angular (smaller overlap with RN mobile) |
| Build tool / dev server | Vite | 5 | Dev server (HMR) + production bundler for web apps | Instant startup, fast HMR, first-class TS + env handling (`VITE_` prefix); simpler than webpack | Create React App (deprecated), webpack (slow config), Next.js (SSR we don't need here) |
| Component kit | Ant Design (antd) | 5 | Enterprise UI components — tables, forms, layouts | Dense data-grid + form components suit ERP admin surfaces; theming via design tokens | MUI (less table-dense), Chakra (fewer enterprise widgets), hand-rolled |
| Server state / data fetching | TanStack Query | 5.x | Caching, background refetch, mutations against the gateway | Removes hand-written loading/error/cache logic; pairs with `@kukkuone/api-client` | SWR (fewer features), Redux Toolkit Query (heavier), raw `fetch` |
| Client state | Zustand | 4.x | Lightweight UI/session state (current org, role, theme) | Minimal boilerplate; no context churn; easy to test | Redux (ceremony), Jotai/Recoil (atoms overkill), Context-only |
| Forms | React Hook Form | 7.x | Form state + validation binding | Performant uncontrolled inputs; integrates with Zod resolver + antd | Formik (slower, heavier), controlled-by-hand |
| Form schema | Zod + `@hookform/resolvers` | 3.x | Validate forms with the **same** schemas the API uses | One schema, client and server; instant parity | Duplicated client-side rules (drift risk) |
| Routing | React Router | 6.x | Client-side routing per web app | Standard SPA routing; nested routes + loaders; framework-agnostic | TanStack Router (younger), Next.js routing (SSR we don't need) |

### 1.4 Mobile (single Expo app, role-aware)

| Layer | Technology | Version | Role | Why chosen | Alternative considered |
|---|---|---|---|---|---|
| App platform | Expo | SDK 51 | Managed React Native app `kukkuone-mobile` (tab layout, role-aware) | OTA updates, managed native modules, EAS build/submit — no Xcode/Gradle babysitting | Bare React Native (more native maintenance), Flutter (different language/stack) |
| Mobile framework | React Native | 0.74 | Native UI runtime (ships inside Expo SDK 51) | Shares React mental model + TS + Zod with web; one team, two targets | Native iOS/Android (2× the work) |
| Mobile routing | Expo Router | 3.x | File-based tab/stack navigation, role-aware layouts | File routing mirrors web structure; deep-linking baked in | React Navigation raw (more wiring) |
| Mobile components | React Native Paper | 5.x | Material component kit for RN | Themable, accessible, batteries-included widget set | NativeBase (heavier), Tamagui (younger), hand-rolled |
| Mobile server state | TanStack Query | 5.x | Same data layer as web, against the gateway | One data-fetching mental model across web + mobile | Apollo (GraphQL only), raw fetch |
| Mobile client state | Zustand | 4.x | Session/role/UI state | Same store library as web — shared patterns | Redux, MobX |
| Secure storage | expo-secure-store | (SDK 51) | Encrypted storage of the Firebase session/token on device | OS keychain/keystore-backed; never store tokens in plain AsyncStorage | AsyncStorage (unencrypted), MMKV+crypto (more setup) |

### 1.5 Shared packages (`packages/`)

| Package | Technology | Role | Consumed by |
|---|---|---|---|
| `@kukkuone/types` | TypeScript + Zod | Shared DTOs, enums, and `z.infer` types — the contract | Every app & service |
| `@kukkuone/db` | Prisma | Prisma schema, generated client, migration scripts | Gateway + all `*-api` services |
| `@kukkuone/auth` | TS + firebase-admin | `AuthProvider` interface + Firebase adapter, token verification, role claims | Gateway, services |
| `@kukkuone/api-client` | TS + `fetch`/TanStack-friendly | Typed client for the gateway REST API | All web apps + mobile |
| `@kukkuone/config` | TS + Zod | Zod-validated env loader (fail-fast) | Every app & service (see [10-Environment-And-Config.md](./10-Environment-And-Config.md)) |
| `@kukkuone/ui` | React + antd tokens | Shared web components/theme | All web apps |

### 1.6 Testing, quality, and delivery

| Layer | Technology | Version | Role | Why chosen | Alternative considered |
|---|---|---|---|---|---|
| Service tests | Jest + Supertest | Jest 29.x / Supertest 7.x | Unit + HTTP integration tests for NestJS services | NestJS default; Supertest drives real HTTP against the app | Vitest on backend (fine, but Jest is the Nest default) |
| Web tests | Vitest + Testing Library | Vitest 2.x / RTL 16.x | Component + hook unit tests for web apps | Vite-native (shares config/transform), fast; RTL for user-centric tests | Jest for web (slower with Vite), Enzyme (dead) |
| E2E (optional) | Playwright (web) / Detox (mobile) | Playwright 1.4x / Detox 20.x | Optional end-to-end flows | Playwright = reliable cross-browser; Detox = RN-aware device E2E | Cypress (weaker multi-tab), Appium (flakier) |
| Lint | ESLint | 9.x | Static analysis, TS + import + a11y rules | Ecosystem standard; shared config across packages | Biome (younger; not yet standard here) |
| Format | Prettier | 3.x | Deterministic formatting | Ends style debates; ESLint-integrated | dprint, editorconfig-only |
| Containers | Docker + Docker Compose | Engine 27.x / Compose v2 | Local orchestration + reproducible images; lives in `kukkuone-infrastructure` | Same image runs local → any host; one-command local stack | Podman (fine), Vagrant/VMs (heavy) — see [11-Local-Setup-And-Deploy.md](./11-Local-Setup-And-Deploy.md) |
| CI | (CI-ready, **not wired**) | — | Turbo pipeline + Docker builds are CI-ready; no provider connected yet | Git not connected yet; nothing structural depends on it | GitHub Actions / GitLab CI / Railway build hooks — pluggable later |

---

## 2. Notes on notable choices

- **One database, one `DATABASE_URL`.** All services share a single PostgreSQL instance and the Prisma schema in `@kukkuone/db`. This keeps migrations, backups, and referential integrity in one place. See [06-Data-Model.md](./06-Data-Model.md).
- **Money is never a float.** Amounts are stored and transported as **integer minor units + a currency code** (e.g. `{ amount: 1500, currency: "INR" }` = ₹15.00). No `float`/`double` anywhere in the money path.
- **Zod _and_ class-validator, on purpose.** Zod owns the shared cross-package contract (`@kukkuone/types`), reused by web/mobile. class-validator runs at the NestJS HTTP edge for DTO validation and Swagger generation. They are complementary, not redundant.
- **Auth is abstracted from day one.** Everything talks to the `AuthProvider` interface in `@kukkuone/auth`; Firebase is merely the default adapter selected by `AUTH_PROVIDER=firebase`. Swapping providers is one adapter + env change, no call-site edits. Super-user is `naptrixlabs@gmail.com`. See [05-Auth-And-RBAC.md](./05-Auth-And-RBAC.md).
- **Host-anywhere by env.** No URLs, keys, or secrets live in code — only in env. The same build runs locally or on Railway/Render/Fly/VPS/K8s by supplying the same variables. See [10-Environment-And-Config.md](./10-Environment-And-Config.md).

---

## 3. Evaluation of original suggestions

The initial stack ideas were reviewed and resolved as follows (full rationale in [00-Decision-Record.md](./00-Decision-Record.md)):

| Original suggestion | Verdict | Resolution |
|---|---|---|
| React Native for mobile | ✅ Accepted | Adopted **as Expo (SDK 51 / RN 0.74)** — managed workflow, one app `kukkuone-mobile`, role-aware tabs |
| React for web | ✅ Accepted | React 18 + Vite 5 + Ant Design 5, two web apps (kukkuone-web + kukkuone-admin-web) |
| A backend + database (unspecified) | ➡️ Resolved | **NestJS 10 + PostgreSQL 16 + Prisma 5**, API Gateway + per-actor services |
| Firebase for auth | ✅ Accepted, **abstracted** | Firebase Auth + `firebase-admin`, but behind the pluggable `AuthProvider` (swap via `AUTH_PROVIDER`) |

**See also:** [02-System-Architecture.md](./02-System-Architecture.md) · [04-Repository-Structure.md](./04-Repository-Structure.md) · [00-Decision-Record.md](./00-Decision-Record.md)
