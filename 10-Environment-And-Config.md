# 10 — Environment & Config

KukkuOne is **host-anywhere by design**: the same build runs on your laptop, Railway, Render, Fly.io, a bare VPS, or Kubernetes with no code changes — you only supply a handful of environment variables. This document defines the env philosophy, the "one connection" variables, the Zod-validated `@kukkuone/config` loader, and the complete `.env.example` blocks for every app.

Related: [05-Auth-And-RBAC.md](./05-Auth-And-RBAC.md) (how auth vars are used) · [11-Local-Setup-And-Deploy.md](./11-Local-Setup-And-Deploy.md) (how to run/deploy with these vars).

---

## 1. Env philosophy

The rules, in order of importance:

1. **No secrets, URLs, or keys in code — ever.** Not in source, not in constants, not in committed config. Everything environment-specific comes from the process environment.
2. **Every app ships a committed `.env.example`.** It lists **every** variable the app reads, with safe placeholder values and inline comments. It is the contract and the onboarding doc.
3. **Real `.env` files are git-ignored.** They contain actual secrets and never leave the developer's machine. `.gitignore` covers `.env`, `.env.*`, `!*.env.example`.
4. **Production uses the host's env panel / secret store.** Railway variables, Render env groups, Fly secrets, Kubernetes Secrets, or a VPS's `docker compose` env file — never a committed file.
5. **Fail fast.** On boot, each app validates its env with Zod (via `@kukkuone/config`). A missing or malformed variable **crashes at startup with a clear message**, never a mysterious runtime error later.
6. **Prefix by platform.** Backend uses bare names; Vite web apps require the `VITE_` prefix; Expo requires the `EXPO_PUBLIC_` prefix. Only prefixed vars are exposed to browser/mobile bundles — so **never** put a server secret behind `VITE_`/`EXPO_PUBLIC_`.

---

## 2. The "one connection" variables (host-anywhere)

These are the only variables you change to move the whole system between environments. Set these correctly and nothing else needs to change.

| Variable | Scope | Purpose |
|---|---|---|
| `DATABASE_URL` | Gateway + all `*-api` services | The **single** PostgreSQL connection string. One shared instance, one URL (see [06-Data-Model.md](./06-Data-Model.md)). |
| `API_BASE_URL` | Gateway (self) | Public base URL of the API Gateway. |
| `VITE_API_BASE_URL` / `EXPO_PUBLIC_API_BASE_URL` | Web / mobile clients | The gateway URL every client consumes. Points clients at local or cloud gateway. |
| `AUTH_PROVIDER` | Gateway + services + `@kukkuone/auth` | Selects the auth adapter. Default `firebase`. |
| `FIREBASE_PROJECT_ID` / `FIREBASE_CLIENT_EMAIL` / `FIREBASE_PRIVATE_KEY` | Backend | Firebase Admin credentials for verifying ID tokens (server side). |
| `VITE_FIREBASE_*` / `EXPO_PUBLIC_FIREBASE_*` | Web / mobile | Firebase **web** config block for client sign-in (public by design). |
| `ADMIN_SUPERUSER_EMAIL` | Gateway | The bootstrap super-admin. **`naptrixlabs@gmail.com`.** |
| `JWT_SECRET` / `SESSION_SECRET` | Gateway | Signs session/JWT material issued by the gateway. |
| `PORT` | Each service (per service) | Listening port; distinct per service locally, host-assigned in cloud. |
| `NODE_ENV` | All backend | `development` / `production` / `test`. |

> **The single-connection principle:** to point the whole platform at a new environment, set `DATABASE_URL` + the gateway's `API_BASE_URL` + clients' `*_API_BASE_URL` + the Firebase block. Everything else is derived. See "point local vs cloud" in [§5](#5-pointing-local-vs-cloud).

---

## 3. The `@kukkuone/config` package (Zod-validated loader)

Every app imports its config from `@kukkuone/config` instead of touching `process.env` directly. The package parses the environment through a Zod schema and **throws on the first invalid boot**, so a misconfigured deploy never limps forward.

```ts
// packages/config/src/env.ts
import { z } from "zod";

/** Shared base every backend service needs. */
const baseSchema = z.object({
  NODE_ENV: z.enum(["development", "production", "test"]).default("development"),
  DATABASE_URL: z.string().url("DATABASE_URL must be a valid connection string"),
  AUTH_PROVIDER: z.enum(["firebase"]).default("firebase"),
  PORT: z.coerce.number().int().positive().default(3000),
});

/** Gateway adds public URL, admin bootstrap, secrets, and Firebase admin creds. */
const gatewaySchema = baseSchema.extend({
  API_BASE_URL: z.string().url(),
  ADMIN_SUPERUSER_EMAIL: z.string().email().default("naptrixlabs@gmail.com"),
  JWT_SECRET: z.string().min(32, "JWT_SECRET must be at least 32 chars"),
  SESSION_SECRET: z.string().min(32),
  FIREBASE_PROJECT_ID: z.string().min(1),
  FIREBASE_CLIENT_EMAIL: z.string().email(),
  // Private key arrives with literal "\n" — normalise to real newlines.
  FIREBASE_PRIVATE_KEY: z
    .string()
    .min(1)
    .transform((k) => k.replace(/\\n/g, "\n")),
});

export type GatewayEnv = z.infer<typeof gatewaySchema>;

/**
 * Parse once at process start. On failure, print every problem and exit —
 * fail fast, never boot half-configured.
 */
export function loadGatewayEnv(source: NodeJS.ProcessEnv = process.env): GatewayEnv {
  const parsed = gatewaySchema.safeParse(source);
  if (!parsed.success) {
    console.error("Invalid environment configuration:");
    console.error(parsed.error.flatten().fieldErrors);
    process.exit(1);
  }
  return parsed.data;
}
```

```ts
// usage at the top of the gateway bootstrap (main.ts)
import { loadGatewayEnv } from "@kukkuone/config";

const env = loadGatewayEnv();          // throws & exits if anything is missing/invalid
await app.listen(env.PORT);
```

Each `*-api` service uses a `serviceSchema` (base + its own extras); web/mobile use browser-safe schemas that read only `VITE_`/`EXPO_PUBLIC_` vars. The pattern is identical everywhere: **schema → `safeParse` → fail fast → typed object**.

---

## 4. `.env.example` blocks

Copy each block to a real `.env` in the same app directory and fill in values. These files are committed; the filled `.env` is not.

### 4.1 API Gateway — `kukkuone-api-gateway/.env.example`

```dotenv
# ── Runtime ───────────────────────────────────────────────
NODE_ENV=development
PORT=3000

# ── Public URL of this gateway (clients point here) ───────
API_BASE_URL=http://localhost:3000

# ── Database (ONE shared Postgres, ONE url) ───────────────
DATABASE_URL=postgresql://kukku:kukku@localhost:5432/kukkuone?schema=public

# ── Auth ──────────────────────────────────────────────────
AUTH_PROVIDER=firebase
ADMIN_SUPERUSER_EMAIL=naptrixlabs@gmail.com

# ── Secrets (generate long random values; min 32 chars) ───
JWT_SECRET=change-me-to-a-32+char-random-string
SESSION_SECRET=change-me-to-another-32+char-random-string

# ── Firebase Admin (server-side token verification) ───────
FIREBASE_PROJECT_ID=your-firebase-project-id
FIREBASE_CLIENT_EMAIL=firebase-adminsdk-xxxxx@your-project.iam.gserviceaccount.com
# Keep the quotes; \n stays literal here and is normalised by @kukkuone/config
FIREBASE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\nMIIEv...\n-----END PRIVATE KEY-----\n"
# Alternative to the three vars above: point at a service-account JSON file
# GOOGLE_APPLICATION_CREDENTIALS=/run/secrets/firebase-service-account.json
```

### 4.2 Farmer API — `kukkuone-farmer-api/.env.example`

```dotenv
# ── Runtime ───────────────────────────────────────────────
NODE_ENV=development
PORT=3101

# ── Database (SAME shared Postgres as the gateway) ────────
DATABASE_URL=postgresql://kukku:kukku@localhost:5432/kukkuone?schema=public

# ── Auth (services verify tokens too) ─────────────────────
AUTH_PROVIDER=firebase
FIREBASE_PROJECT_ID=your-firebase-project-id
FIREBASE_CLIENT_EMAIL=firebase-adminsdk-xxxxx@your-project.iam.gserviceaccount.com
FIREBASE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\nMIIEv...\n-----END PRIVATE KEY-----\n"
```

> **`supplier-api` and `distributor-api` mirror this file exactly** — same variables, only `PORT` differs (e.g. supplier `3102`, distributor `3103`). They all read the same `DATABASE_URL`.

### 4.3 Admin Web — `kukkuone-admin-web/.env.example`

```dotenv
# Vite exposes ONLY vars prefixed with VITE_ to the browser bundle.
# Never put a server secret here.

# ── Gateway the admin app talks to ────────────────────────
VITE_API_BASE_URL=http://localhost:3000

# ── Firebase WEB config (public by design — client sign-in)
VITE_FIREBASE_API_KEY=AIzaSy...your-web-api-key
VITE_FIREBASE_AUTH_DOMAIN=your-project.firebaseapp.com
VITE_FIREBASE_PROJECT_ID=your-firebase-project-id
VITE_FIREBASE_STORAGE_BUCKET=your-project.appspot.com
VITE_FIREBASE_MESSAGING_SENDER_ID=1234567890
VITE_FIREBASE_APP_ID=1:1234567890:web:abcdef123456
```

### 4.4 Actor Web — `kukkuone-web/.env.example`

```dotenv
# Same shape as admin-web. One app serves all actor capabilities.
VITE_API_BASE_URL=http://localhost:3000

VITE_FIREBASE_API_KEY=AIzaSy...your-web-api-key
VITE_FIREBASE_AUTH_DOMAIN=your-project.firebaseapp.com
VITE_FIREBASE_PROJECT_ID=your-firebase-project-id
VITE_FIREBASE_STORAGE_BUCKET=your-project.appspot.com
VITE_FIREBASE_MESSAGING_SENDER_ID=1234567890
VITE_FIREBASE_APP_ID=1:1234567890:web:abcdef123456
```

### 4.5 Mobile — `kukkuone-mobile/.env.example`

```dotenv
# Expo exposes ONLY vars prefixed with EXPO_PUBLIC_ to the app bundle.

# ── Gateway the mobile app talks to ───────────────────────
# Use your machine's LAN IP (not localhost) when testing on a physical device.
EXPO_PUBLIC_API_BASE_URL=http://localhost:3000

# ── Firebase WEB config (public by design — client sign-in)
EXPO_PUBLIC_FIREBASE_API_KEY=AIzaSy...your-web-api-key
EXPO_PUBLIC_FIREBASE_AUTH_DOMAIN=your-project.firebaseapp.com
EXPO_PUBLIC_FIREBASE_PROJECT_ID=your-firebase-project-id
EXPO_PUBLIC_FIREBASE_STORAGE_BUCKET=your-project.appspot.com
EXPO_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=1234567890
EXPO_PUBLIC_FIREBASE_APP_ID=1:1234567890:web:abcdef123456
```

---

## 5. Pointing local vs cloud

Because every environment-specific value is an env var, switching targets is a values-only change — **no rebuild of logic, no code edits**.

| Value | Local | Cloud (example) |
|---|---|---|
| `DATABASE_URL` | `postgresql://kukku:kukku@localhost:5432/kukkuone` | `postgresql://user:pass@db.internal:5432/kukkuone` (host-provided) |
| Gateway `API_BASE_URL` | `http://localhost:3000` | `https://api.kukkuone.com` |
| `VITE_API_BASE_URL` | `http://localhost:3000` | `https://api.kukkuone.com` |
| `EXPO_PUBLIC_API_BASE_URL` | `http://<LAN-IP>:3000` | `https://api.kukkuone.com` |
| Firebase block | dev Firebase project | prod Firebase project |

Change those, redeploy, done. The database is the same schema; the code is the same build. See [11-Local-Setup-And-Deploy.md](./11-Local-Setup-And-Deploy.md#host-anywhere) for per-target steps.

---

## 6. Secret handling

- **Never commit a real secret.** Only `.env.example` (placeholders) is tracked. `.env` and `.env.*` are git-ignored.
- **Use the host's secret store in production.** Railway Variables, Render Environment Groups, Fly Secrets (`fly secrets set`), Kubernetes `Secret` objects, or a locked-down `.env` on a VPS injected via `docker compose --env-file`.
- **Rotate `JWT_SECRET` / `SESSION_SECRET`** on any suspicion of leak; they are ≥32 random chars.
- **Firebase service account** — two supported delivery methods:
  1. **Discrete env vars** — `FIREBASE_PROJECT_ID`, `FIREBASE_CLIENT_EMAIL`, `FIREBASE_PRIVATE_KEY` (with `\n`-escaped key; `@kukkuone/config` un-escapes it). Best for platforms with a variables UI.
  2. **Service-account JSON file** — set `GOOGLE_APPLICATION_CREDENTIALS=/path/to/sa.json` (mount it as a secret file), **or** ship it as a base64 env var and decode at boot:
     ```bash
     # produce the value
     base64 -i service-account.json | tr -d '\n'   # -> FIREBASE_SA_BASE64
     ```
     ```ts
     // decode at startup when FIREBASE_SA_BASE64 is set
     const json = Buffer.from(process.env.FIREBASE_SA_BASE64!, "base64").toString("utf8");
     const serviceAccount = JSON.parse(json);
     ```
- **Client Firebase config is public.** `VITE_FIREBASE_*` / `EXPO_PUBLIC_FIREBASE_*` are meant to ship to the browser/app; access is gated by Firebase security rules and gateway RBAC ([05-Auth-And-RBAC.md](./05-Auth-And-RBAC.md)), not by hiding these keys.

**See also:** [05-Auth-And-RBAC.md](./05-Auth-And-RBAC.md) · [11-Local-Setup-And-Deploy.md](./11-Local-Setup-And-Deploy.md) · [03-Tech-Stack.md](./03-Tech-Stack.md)
