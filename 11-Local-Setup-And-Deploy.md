# 11 — Local Setup & Deploy

How to get KukkuOne running on your machine, and how to deploy the **same build** anywhere. This guide assumes the env model in [10-Environment-And-Config.md](./10-Environment-And-Config.md) and the runtime topology in [02-System-Architecture.md](./02-System-Architecture.md).

---

## 1. Prerequisites

| Tool | Version | Purpose | Install |
|---|---|---|---|
| Node.js | 20 LTS | Runtime for services, web build, Expo tooling | `nvm install 20 && nvm use 20` |
| pnpm | 9.x | Package manager / workspaces | `corepack enable && corepack prepare pnpm@latest --activate` |
| Docker Desktop | latest | Local Postgres + Compose stack | <https://www.docker.com/products/docker-desktop> |
| Expo CLI + Expo Go | SDK 51 | Run the mobile app on device/simulator | `npm i -g expo` (or `pnpm dlx expo`); install **Expo Go** on your phone |
| Firebase project | — | Google auth provider + admin credentials | <https://console.firebase.google.com> |

Verify: `node -v` → `v20.x`, `pnpm -v` → `9.x`, `docker --version`, `expo --version`.

---

## 2. First-time setup

```bash
# 1. Clone / open the monorepo root
cd KukkuOne

# 2. Install everything (pnpm workspaces resolves all apps + packages)
pnpm install

# 3. Create real .env files from every committed example
#    (repeat per app, or script it — see snippet below)
cp kukkuone-api-gateway/.env.example   kukkuone-api-gateway/.env
cp kukkuone-farmer-api/.env.example    kukkuone-farmer-api/.env
cp kukkuone-supplier-api/.env.example  kukkuone-supplier-api/.env
cp kukkuone-distributor-api/.env.example kukkuone-distributor-api/.env
cp kukkuone-admin-web/.env.example     kukkuone-admin-web/.env
cp kukkuone-web/.env.example    kukkuone-web/.env
cp kukkuone-mobile/.env.example        kukkuone-mobile/.env
```

```bash
# Optional one-liner: copy every example that lacks a .env sibling
find . -name '.env.example' -not -path '*/node_modules/*' | while read -r f; do
  [ -f "${f%.example}" ] || cp "$f" "${f%.example}"
done
```

### 2.1 Firebase setup (one time)

1. Create a Firebase project (or reuse the dev one).
2. **Authentication → Sign-in method → Google →** enable.
3. **Project settings → General → Your apps → Web app** → register an app → copy the web config into the `VITE_FIREBASE_*` / `EXPO_PUBLIC_FIREBASE_*` vars.
4. **Project settings → Service accounts → Generate new private key** → use its `project_id`, `client_email`, `private_key` for the backend `FIREBASE_*` vars (or mount the JSON via `GOOGLE_APPLICATION_CREDENTIALS`). See secret handling in [10-Environment-And-Config.md](./10-Environment-And-Config.md#6-secret-handling).

Fill the values into the `.env` files you just created.

---

## 3. Local run

Two supported modes. **Compose mode** runs the whole stack in containers (closest to prod). **Dev mode** runs services/web with Turbo + hot reload, with only Postgres in Docker.

### 3.1 Compose mode — full stack in Docker

The Compose file lives in `kukkuone-infrastructure`. Sketch:

```yaml
# kukkuone-infrastructure/docker-compose.yml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: kukku
      POSTGRES_PASSWORD: kukku
      POSTGRES_DB: kukkuone
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U kukku -d kukkuone"]
      interval: 5s
      timeout: 5s
      retries: 10

  gateway:
    build: ../kukkuone-api-gateway
    env_file: ../kukkuone-api-gateway/.env
    ports: ["3000:3000"]
    depends_on:
      postgres: { condition: service_healthy }
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:3000/health || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5

  farmer-api:
    build: ../kukkuone-farmer-api
    env_file: ../kukkuone-farmer-api/.env
    ports: ["3101:3101"]
    depends_on:
      postgres: { condition: service_healthy }

  supplier-api:
    build: ../kukkuone-supplier-api
    env_file: ../kukkuone-supplier-api/.env
    ports: ["3102:3102"]
    depends_on:
      postgres: { condition: service_healthy }

  distributor-api:
    build: ../kukkuone-distributor-api
    env_file: ../kukkuone-distributor-api/.env
    ports: ["3103:3103"]
    depends_on:
      postgres: { condition: service_healthy }

  admin-web:
    build: ../kukkuone-admin-web
    env_file: ../kukkuone-admin-web/.env
    ports: ["5173:80"]
    depends_on:
      gateway: { condition: service_healthy }

volumes:
  pgdata:
```

```bash
cd kukkuone-infrastructure
docker compose up --build         # add -d to detach
# gateway → http://localhost:3000, admin-web → http://localhost:5173
```

### 3.2 Dev mode — Turbo + Postgres-in-Docker

Best inner loop: HMR on web, watch-reload on services, only the DB containerised.

```bash
# 1. Just the database
cd kukkuone-infrastructure
docker compose up -d postgres

# 2. Everything else with hot reload, from the root
cd ..
pnpm turbo dev        # runs gateway + *-api + web apps in parallel
# or target a subset:
pnpm turbo dev --filter=kukkuone-api-gateway --filter=kukkuone-farmer-api --filter=kukkuone-web
```

---

## 4. Database workflow (Prisma)

Prisma schema and migrations live in `@kukkuone/db` (see [06-Data-Model.md](./06-Data-Model.md)). All commands read the shared `DATABASE_URL`.

```bash
# From the db package (or via a root script)
cd packages/db

# Create + apply a migration during development
pnpm prisma migrate dev --name init

# Regenerate the typed client after schema changes
pnpm prisma generate

# Seed: creates the super-admin + a demo org with the Farmer capability
pnpm prisma db seed

# Inspect data
pnpm prisma studio
```

The seed script provisions:

- **Super-admin** user `naptrixlabs@gmail.com` (see [05-Auth-And-RBAC.md](./05-Auth-And-RBAC.md)).
- A **demo organization** with the **Farmer** capability enabled (Farmer module is built first — [07-Farmer-Module-Spec.md](./07-Farmer-Module-Spec.md)).

> In production, run `prisma migrate deploy` (not `migrate dev`) during release — see [§6](#6-host-anywhere).

---

## 5. Running the mobile app

```bash
cd kukkuone-mobile

# Point the app at your gateway.
# Physical device: use your machine's LAN IP, NOT localhost.
#   EXPO_PUBLIC_API_BASE_URL=http://192.168.1.20:3000   (edit .env)
expo start

# Then:
#  - press i  → iOS simulator
#  - press a  → Android emulator
#  - or scan the QR code with Expo Go on a physical device
```

The app is role-aware: after Google sign-in, tabs render according to the user's role/capabilities. See [09-Client-Apps-Spec.md](./09-Client-Apps-Spec.md).

---

## 6. Verification checklist

Run these after a fresh setup to confirm the stack is healthy:

- [ ] **Gateway health** — `curl http://localhost:3000/health` returns `200`/`{"status":"ok"}`.
- [ ] **DB reachable** — `pnpm prisma studio` opens and shows seeded tables.
- [ ] **Seed present** — super-admin `naptrixlabs@gmail.com` and the demo org exist.
- [ ] **Login with Google** — open admin-web (`http://localhost:5173`), sign in with Google, land on the dashboard.
- [ ] **Create a farm** — in the Farmer web/app, create a farm; confirm it persists (reappears after refresh) and is visible in Prisma Studio.
- [ ] **Mobile connects** — `expo start`, sign in, tabs load against the gateway.

If login loops or 401s, verify `AUTH_PROVIDER`, the Firebase web config, and that the backend `FIREBASE_*` creds match the same project.

---

## 6. Host anywhere {#host-anywhere}

The build is **environment-agnostic**. Any target runs the identical images/build; you only supply env: **`DATABASE_URL` + gateway `API_BASE_URL` + clients' `*_API_BASE_URL` + the Firebase block** (full list in [10-Environment-And-Config.md](./10-Environment-And-Config.md#2-the-one-connection-variables-host-anywhere)). Nothing else changes.

### Build commands per app

| App | Build | Start (prod) |
|---|---|---|
| Gateway / `*-api` (NestJS) | `pnpm --filter <app> build` | `node dist/main.js` |
| Web apps (Vite) | `pnpm --filter <app> build` → static `dist/` | serve `dist/` (nginx/static host) |
| Mobile (Expo) | `eas build` (iOS/Android) or `expo export` (web) | store / OTA via EAS |
| DB migrations | — | `pnpm prisma migrate deploy` on release |

### Per-target steps

**Railway**
1. Create a **PostgreSQL** plugin → copy its `DATABASE_URL`.
2. Create one **service per app** (gateway, farmer-api, supplier-api, distributor-api, admin-web, …) from the monorepo, each with its root/build command.
3. Set each service's **Variables** (the one-connection set). Run `prisma migrate deploy` as a release/deploy step on the gateway service.

**Render**
1. Create a **Managed PostgreSQL** → grab its internal `DATABASE_URL`.
2. **Web Service** per NestJS app (build `pnpm --filter <app> build`, start `node dist/main.js`); **Static Site** per Vite web app (publish `dist/`).
3. Put shared vars in an **Environment Group**; attach to each service.

**Fly.io**
1. `fly launch` in each app dir → generates a `fly.toml` and app.
2. Create Postgres: `fly postgres create` → `fly postgres attach` (sets `DATABASE_URL`).
3. `fly secrets set FIREBASE_PRIVATE_KEY=... JWT_SECRET=...`; `fly deploy` per app.

**Any VPS**
1. Install Docker + Compose; copy `kukkuone-infrastructure/docker-compose.yml`.
2. Provide a production `.env` per service (from the host secret store, not committed).
3. `docker compose up -d --build`; run `prisma migrate deploy` once. Front with nginx/Caddy for TLS.

**Kubernetes**
1. Build/push each app image to a registry.
2. Create a `Secret` with the one-connection vars; reference via `envFrom`.
3. `Deployment` + `Service` per app; run migrations as a `Job` (`prisma migrate deploy`) before rollout.

### The single connection point

> To move the entire platform to a new host: set **`DATABASE_URL`**, the gateway **`API_BASE_URL`**, the clients' **`*_API_BASE_URL`**, and the **Firebase** vars. That's the whole switch — same code, same images, new environment. See [10-Environment-And-Config.md](./10-Environment-And-Config.md) and [02-System-Architecture.md](./02-System-Architecture.md).

**See also:** [10-Environment-And-Config.md](./10-Environment-And-Config.md) · [02-System-Architecture.md](./02-System-Architecture.md) · [06-Data-Model.md](./06-Data-Model.md)
