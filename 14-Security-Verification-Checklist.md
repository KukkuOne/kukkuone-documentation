# 14 — Security Verification Checklist (OWASP-based)

An agent-runnable security audit for the whole KukkuOne workspace. Every check is phrased so it can be verified from the code, the config, or a controlled runtime test, and every check is mapped to the OWASP standard it comes from.

**Standards covered**

| Standard | Version | What it gives us |
|---|---|---|
| OWASP **MASVS** (Mobile Application Security Verification Standard) | v2.1 | Mobile controls: STORAGE, CRYPTO, AUTH, NETWORK, PLATFORM, CODE, RESILIENCE, PRIVACY |
| OWASP **ASVS** (Application Security Verification Standard) | 5.0 | Backend and web controls, chapters V1–V17 |
| OWASP **API Security Top 10** | 2023 | API1–API10 |
| OWASP **Mobile Top 10** | 2024 | M1–M10 |
| OWASP **Top 10** (web) | 2025 | A01–A10 |

> **About the wording.** The check IDs (`AUTHZ-01`, `MOB-STO-03`, …) and the wording are KukkuOne's own. They put the OWASP requirements into concrete, testable terms for this stack (Flutter, NestJS on Cloudflare Workers, Prisma/PostgreSQL, Firebase Auth, React/Vite, Astro). Each check names the official category it comes from. For the official requirement text, use the sources in [Appendix C](#appendix-c--official-sources). A static code audit doesn't replace an external penetration test.

---

## 0. Instructions for the auditing agent

**Read this section first and follow it exactly.**

1. **Read-only.** Don't change code, config, secrets, databases or cloud resources. Don't commit. This is an audit, not a fix.
2. **No production traffic.** Runtime checks run only against local (`localhost`) or a staging environment the user names. Never send attack payloads to production. If a check can only be confirmed at runtime and no safe target is available, mark it `NEEDS-RUNTIME`.
3. **Evidence or it didn't happen.** Every `PASS` and `FAIL` must cite evidence: `path/to/file.ts:LINE`, a command and its output, or a config key. "Looks fine" is not evidence.
4. **Never print secret values.** If you find a secret, report its location and type, and mask the value (`sk_live_****ab12`).
5. **Status values:**
   - `PASS`: requirement met, with evidence
   - `FAIL`: requirement not met, with evidence
   - `PARTIAL`: met in some places but not others; list both
   - `N/A`: doesn't apply (say why, e.g. "no file uploads exist")
   - `NEEDS-RUNTIME`: needs a live test that couldn't be done safely
   - `NEEDS-HUMAN`: a process or organisational question the code can't answer
6. **Severity** (when a check fails, use the check's default severity unless the evidence justifies changing it, and explain why):
   - `Critical`: direct compromise of data, accounts or money, exploitable remotely with little effort
   - `High`: serious exposure that needs some precondition
   - `Medium`: defence-in-depth gap or limited exposure
   - `Low`: hardening, hygiene
7. **Check every app the item applies to.** A check with scope `[API]` must be checked in *each* API repo, and the result reported per repo.
8. **Parallelise by section.** Sections are independent. Give each section (or each repo) to a separate sub-agent, then merge the results into the report template in [Appendix B](#appendix-b--report-template).
9. **Report what you didn't check.** Unchecked items must appear in the report as unchecked, not be silently dropped.

### Scope tags

| Tag | Repos |
|---|---|
| `[MOB]` | `kukkuone-farmer-mobile`, `kukkuone-partner-mobile`, `kukkuone-hatchery-mobile`, `kukkuone-trading-partner-mobile` (Flutter) |
| `[API]` | `kukkuone-api-gateway`, `kukkuone-farmer-api`, `kukkuone-supply-api`, `kukkuone-trade-api` (NestJS, Cloudflare Workers) |
| `[WEB]` | `kukkuone-admin-web`, `kukkuone-platform-web`, `kukkuone-distributor-web`, `kukkuone-supplier-web` (React/Vite), `kukkuone-marketing-web` (Astro) |
| `[DB]` | `kukkuone-database` (Prisma/PostgreSQL) |
| `[PKG]` | `packages/*` (shared auth, types, config, api-client) |
| `[INFRA]` | `kukkuone-infrastructure`, every `wrangler.toml`, Firebase config, CI config |
| `[ADMIN]` | `kukkuone-admin-web` only |
| `[ALL]` | Every repo with source code |
| `[ORG]` | Process items, usually `NEEDS-HUMAN` |

Skip repos that only contain a README (no source yet) and list them in the report as "not yet implemented".

---

## 1. Architecture and threat model

*ASVS V15 (Secure Coding and Architecture) · Top 10 A06 (Insecure Design)*

- [ ] **ARCH-01** · High · `[ORG]` A written threat model exists covering assets (accounts, tenant data, money, PII), actors (farmer, supplier, distributor, admin, anonymous, other tenants) and trust boundaries (device → gateway → services → DB → Firebase). **Verify:** look in `kukkuone-documentation/` for a threat model or STRIDE analysis.
- [ ] **ARCH-02** · High · `[API]` Every client request goes through the gateway, and the internal services (`farmer-api`, `supply-api`, `trade-api`) **can't be reached directly** from the internet, or if they can, they authenticate and authorise each request on their own. **Verify:** check each `wrangler.toml` for `routes`, `workers_dev = true` (a public `*.workers.dev` URL) and service bindings. Check whether the services trust gateway headers (e.g. `x-user-id`) without verifying them.
- [ ] **ARCH-03** · Critical · `[API]` Services never trust identity or role from client-controlled headers or bodies. Identity comes only from a verified token, or from a signed or bound internal channel. **Verify:** grep for `headers['x-`, `req.headers`, `request.header(`, and trace where `userId`/`tenantId`/`role` come from.
- [ ] **ARCH-04** · High · `[ALL]` Security controls (auth guard, tenant scoping, validation) live in one shared place (`packages/auth`, global guards and pipes) instead of being copied into each controller. **Verify:** check for a global `APP_GUARD`, a global `ValidationPipe`, and shared tenant-scoping helpers.
- [ ] **ARCH-05** · Medium · `[ALL]` Deny by default: new routes are protected unless explicitly marked public. **Verify:** look for a global guard and an `@Public()`-style opt-out decorator, and list every route that uses the opt-out.
- [ ] **ARCH-06** · Medium · `[ORG]` Documentation matches the code (the documented stack, auth flow and environments are what is actually built). **Verify:** compare `03-Tech-Stack.md` and `05-Auth-And-RBAC.md` against the repos and report differences that affect security.
- [ ] **ARCH-07** · Medium · `[ALL]` Data in each environment (dev/staging/prod) is isolated: separate databases, Firebase projects and secrets. **Verify:** `10-Environment-And-Config.md`, `wrangler.toml` `[env.*]` blocks, Firebase project IDs per flavour.

---

## 2. Authentication

*ASVS V6 (Authentication), V10 (OAuth and OIDC) · MASVS-AUTH-1, AUTH-2, AUTH-3 · API2 · M3 · A07*

- [ ] **AUTHN-01** · Critical · `[API][PKG]` Every Firebase ID token is verified on the server with `verifyIdToken` (signature, `aud` = our project ID, `iss`, `exp`). The token is never just decoded. **Verify:** read `packages/auth`. Grep for `jwt.decode`, `atob(`, `split('.')`, `decodeJwt` used without verification. On Workers (where `firebase-admin` may not run), confirm the JWKS-based verification checks the signature, `aud`, `iss`, `exp`, `iat` and `auth_time`.
- [ ] **AUTHN-02** · High · `[API][PKG]` Token verification checks for revocation (`checkRevoked: true`, or an equivalent revocation or `tokensValidAfterTime` check) for sensitive operations at least. **Verify:** read the verification code.
- [ ] **AUTHN-03** · High · `[API]` Signing keys (JWKS) are fetched from Google's official endpoint over HTTPS and cached according to `Cache-Control`. Keys aren't hard-coded. **Verify:** read the key-fetching code.
- [ ] **AUTHN-04** · Critical · `[API]` Algorithm confusion is prevented: only `RS256` is accepted, never `none` or HS*. **Verify:** check the verification options.
- [ ] **AUTHN-05** · High · `[API]` Roles and tenant membership come from server-side data (DB) or server-set custom claims. Users can't set their own role at sign-up or in a profile update. **Verify:** trace role assignment. Grep for `setCustomUserClaims` and where its input comes from.
- [ ] **AUTHN-06** · High · `[API][MOB][WEB]` Login, sign-up, OTP and password-reset flows don't reveal whether an account exists (same response and similar timing). **Verify:** read the error handling in auth endpoints and UI messages.
- [ ] **AUTHN-07** · High · `[API][INFRA]` Brute-force and SMS-pumping protection on OTP and login: rate limits per IP and per phone or account, Firebase App Check enforced, and phone-auth country restrictions set if the product is single-country. **Verify:** check throttler config, Cloudflare rate-limit rules, and Firebase App Check enforcement (mark `NEEDS-HUMAN` for console settings).
- [ ] **AUTHN-08** · Medium · `[API]` If email/password is supported, email verification is required before access to tenant data. **Verify:** check whether `email_verified` is enforced.
- [ ] **AUTHN-09** · High · `[MOB][WEB]` Google/Apple sign-in uses the official SDKs with PKCE or native flows. No custom OAuth with an embedded client secret. **Verify:** check `finish-google-signin.sh` and the auth code. Grep for `client_secret`.
- [ ] **AUTHN-10** · High · `[MOB][WEB]` OAuth/OIDC redirect URIs and deep-link callbacks are an exact allow-list (no wildcards), and the `state`/`nonce` are checked. **Verify:** auth config and the Firebase authorised domains (`NEEDS-HUMAN` for the console).
- [ ] **AUTHN-11** · Medium · `[MOB]` (MASVS-AUTH-2) If biometric or device unlock gates the app, it is tied to a Keychain/Keystore item that requires user presence, not just a `true/false` from `local_auth`. **Verify:** grep for `local_auth`, `authenticate(` and check what it protects.
- [ ] **AUTHN-12** · High · `[API][MOB]` (MASVS-AUTH-3) Sensitive operations ask for re-authentication or a fresh token (`auth_time` within N minutes): changing phone or email, adding bank or payout details, deleting the account, changing roles, large payments or settlements. **Verify:** list the sensitive endpoints and check each one.
- [ ] **AUTHN-13** · Medium · `[ADMIN]` Admin accounts (`kukkuone-admin-web`) require MFA. **Verify:** auth flow for admin, Firebase MFA config (`NEEDS-HUMAN`).
- [ ] **AUTHN-14** · Medium · `[API]` Service-to-service calls use their own credentials (service binding, mTLS or a signed internal token), not a forwarded user token treated as a service identity, and not a shared static secret in source. **Verify:** gateway → service call code.
- [ ] **AUTHN-15** · Low · `[MOB][WEB]` After a logout the server-side refresh token or session is revoked, and local tokens, caches and secure storage are cleared. **Verify:** logout implementation.

---

## 3. Session and token management

*ASVS V7 (Session Management), V9 (Self-contained Tokens) · API2 · M3*

- [ ] **SESS-01** · High · `[MOB]` Tokens are stored only in `flutter_secure_storage` (Keychain/Keystore), never in `shared_preferences`, plain files, Hive/Isar without encryption, or logs. **Verify:** grep for `SharedPreferences`, `setString(`, `Hive.box`, `File(`, then check what is written.
- [ ] **SESS-02** · High · `[WEB]` Web apps don't keep long-lived tokens in `localStorage`/`sessionStorage`. If the Firebase JS SDK holds the session, its persistence mode is chosen on purpose, and the XSS controls (WEB-01..05) are strong. **Verify:** grep for `localStorage.setItem`, `setPersistence`.
- [ ] **SESS-03** · Medium · `[API]` Access tokens are short-lived (Firebase ID token ≈ 1h), and no custom long-lived tokens are issued. **Verify:** grep for custom JWT signing (`sign(`, `SignJWT`) and check `exp`.
- [ ] **SESS-04** · Medium · `[API][ORG]` An admin can revoke all of a user's sessions (on compromise, offboarding or role change). **Verify:** look for `revokeRefreshTokens` usage or an admin endpoint.
- [ ] **SESS-05** · Medium · `[API]` When a user's role or tenant membership changes, the change takes effect right away (claims refreshed or token revoked). Stale tokens can't keep old privileges. **Verify:** role-change flow.
- [ ] **SESS-06** · Medium · `[WEB]` If cookies are used: `Secure`, `HttpOnly`, `SameSite=Lax/Strict`, a `__Host-` prefix where possible, and CSRF protection for cookie-authenticated state changes. **Verify:** grep for `Set-Cookie`, `cookie`.
- [ ] **SESS-07** · Low · `[MOB][WEB]` Idle timeout on admin and finance screens. **Verify:** UI and session code.
- [ ] **SESS-08** · High · `[ALL]` Tokens never appear in URLs (query strings), logs, analytics events or crash reports. **Verify:** grep for `?token=`, `access_token=`, and logging of headers or request objects.

---

## 4. Authorization and multi-tenant isolation

*ASVS V8 (Authorization) · API1 (BOLA), API3 (BOPLA), API5 (BFLA) · M3 · A01*

> KukkuOne is a multi-tenant B2B ERP. A tenant isolation failure is the most damaging bug class. Treat this section as the highest priority.

- [ ] **AUTHZ-01** · Critical · `[API][DB]` **BOLA:** every endpoint that takes a resource ID (path, query or body) checks that the resource belongs to the caller's tenant and the caller is allowed to access it. **Verify:** list every route with `:id`-style params or ID fields in the body. For each one, show the Prisma query includes a tenant/owner condition (`where: { id, tenantId }`), or that an explicit check runs before data is returned or changed. Report every route individually.
- [ ] **AUTHZ-02** · Critical · `[API][DB]` List and search endpoints are always scoped to the caller's tenant. No unscoped `findMany()`. **Verify:** grep for `findMany(`, `count(`, `aggregate(`, `groupBy(` and check each has tenant scoping.
- [ ] **AUTHZ-03** · Critical · `[API][DB]` Updates and deletes include the tenant condition in the same query (`updateMany`/`deleteMany` with `tenantId`, or a find-then-check in one transaction). Checking before and writing by `id` alone is a race and a bypass risk. **Verify:** grep for `update(`, `delete(`, `upsert(` that use `where: { id }` only.
- [ ] **AUTHZ-04** · Critical · `[API]` **BFLA:** every admin, internal or privileged route has a role guard checked on the server. Hiding a button in the UI is not access control. **Verify:** list all routes with their guards and roles. Flag any admin-like path (`/admin`, `/internal`, `approve`, `verify`, `settle`, `refund`, `role`) that has no role guard.
- [ ] **AUTHZ-05** · High · `[API]` **BOPLA / mass assignment:** request bodies are never passed straight into Prisma (`data: body`, `data: { ...dto }`) unless the DTO is a strict allow-list. Protected fields (`role`, `tenantId`, `ownerId`, `status`, `verified`, `balance`, `price`, `createdBy`) can't be set by clients. **Verify:** grep for `data: body`, `data: dto`, `...body`, `...dto`, `...req.body` into Prisma calls.
- [ ] **AUTHZ-06** · High · `[API]` **BOPLA / over-exposure:** responses return only the fields the client needs. No full Prisma rows containing internal fields, other users' PII or secrets. **Verify:** check for `select`/`omit` or response DTO mapping. Flag routes that return raw `include`-heavy objects.
- [ ] **AUTHZ-07** · High · `[API]` Cross-actor flows (supplier → farmer → distributor, trading partner) check the *relationship* (e.g. this supplier actually has a contract with this farmer), not just the role. **Verify:** trade/supply flows in `supply-api` and `trade-api`.
- [ ] **AUTHZ-08** · High · `[API]` Platform/admin users who can act across tenants are explicitly marked, and every cross-tenant action is audit-logged (see LOG-03). **Verify:** admin routes and logging.
- [ ] **AUTHZ-09** · Medium · `[API][DB]` IDs exposed to clients can't be guessed (UUID/CUID), as an extra layer on top of AUTHZ-01 and never instead of it. **Verify:** `schema.prisma` `@id @default(...)`.
- [ ] **AUTHZ-10** · High · `[API]` Automated tests prove isolation: user A from tenant 1 gets 403/404 for tenant 2's resources on read, update and delete. **Verify:** search the test folders for cross-tenant tests and report the coverage.
- [ ] **AUTHZ-11** · Medium · `[DB]` Optional defence in depth: PostgreSQL Row-Level Security, or a Prisma client extension that adds `tenantId` to every query automatically. **Verify:** look for `$extends`, middleware, or `CREATE POLICY` in migrations.
- [ ] **AUTHZ-12** · High · `[INFRA]` Firebase Firestore/Storage/RTDB rules, if any of those products are used, aren't open (`allow read, write: if true` or `if request.auth != null` on tenant data). **Verify:** `firestore.rules`, `storage.rules`, `database.rules.json`, `firebase.json`.

---

## 5. Input validation, encoding and injection

*ASVS V1 (Encoding and Sanitization), V2 (Validation and Business Logic) · MASVS-CODE-4 · M4 · A05*

- [ ] **INPUT-01** · High · `[API]` A global `ValidationPipe` is set with `whitelist: true`, `forbidNonWhitelisted: true`, `transform: true`. **Verify:** `main.ts` / worker bootstrap in every API.
- [ ] **INPUT-02** · High · `[API]` Every controller input (body, query, params, headers that are used) has a DTO or Zod schema with types, lengths, ranges, formats and enums. No `any`/`object`/`Record<string, unknown>` bodies. **Verify:** list controllers whose `@Body()` has no class DTO or that use `any`.
- [ ] **INPUT-03** · Critical · `[API][DB]` **SQL injection:** no string-built SQL. `$queryRawUnsafe` / `$executeRawUnsafe` aren't used with user input. `$queryRaw` uses only tagged-template parameters. **Verify:** grep for `queryRawUnsafe`, `executeRawUnsafe`, `Prisma.raw(`, `` `SELECT `` / `` `UPDATE `` template strings.
- [ ] **INPUT-04** · High · `[API][DB]` **ORM operator injection:** users can't pass objects where scalars are expected (e.g. `{ "id": { "not": "" } }` into a `where`). **Verify:** check that IDs and filters are validated as primitives before reaching Prisma.
- [ ] **INPUT-05** · High · `[API]` Sort and filter fields from the client are checked against an allow-list before use in `orderBy`/`where`. **Verify:** grep for `orderBy: { [`, dynamic keys from the query.
- [ ] **INPUT-06** · Medium · `[API]` Numeric and money fields: range checks, no negative quantities or prices unless intended, a decimal type for money (`Decimal`, not float). **Verify:** DTOs and `schema.prisma` money columns.
- [ ] **INPUT-07** · High · `[API]` No `eval`, `new Function`, `vm`, template rendering of user input, or shell execution. **Verify:** grep for `eval(`, `new Function(`, `child_process`, `exec(`.
- [ ] **INPUT-08** · Medium · `[API]` Regexes run on user input are safe from catastrophic backtracking (ReDoS). **Verify:** list regexes that run on user input and flag nested quantifiers.
- [ ] **INPUT-09** · Medium · `[API]` Prototype pollution: no deep-merge of user objects into config or shared objects (`lodash.merge`, `Object.assign` from user input into a prototype chain). **Verify:** grep for `merge(`, `defaultsDeep`, `__proto__`.
- [ ] **INPUT-10** · High · `[WEB]` Output encoding: React's default escaping isn't bypassed. Any `dangerouslySetInnerHTML`, `innerHTML` or Astro `set:html` uses sanitised input (DOMPurify). **Verify:** grep for those three.
- [ ] **INPUT-11** · Medium · `[WEB][API]` User-supplied URLs used in `href`/`src`/redirects are restricted to `https:` (blocking `javascript:` and `data:`). **Verify:** grep for links built from data.
- [ ] **INPUT-12** · Medium · `[API]` CSV/Excel exports neutralise formula injection (cells starting with `= + - @`). **Verify:** search for export code.
- [ ] **INPUT-13** · Medium · `[API]` If HTML, email or SMS templates include user data, it is encoded for that context. **Verify:** notification templates.
- [ ] **INPUT-14** · High · `[MOB]` (MASVS-CODE-4) Data from deep links, push payloads, QR codes, the clipboard and the server is validated before use in navigation, WebViews or business logic. **Verify:** deep-link handlers, `onMessage` handlers, and QR/scan code.

---

## 6. Business logic and abuse

*ASVS V2 · API4 (Unrestricted Resource Consumption), API6 (Unrestricted Access to Sensitive Business Flows)*

- [ ] **LOGIC-01** · High · `[API]` State machines (order, batch, settlement, invoice, approval) enforce allowed transitions on the server, so a client can't jump `PENDING → PAID` or edit a closed record. **Verify:** status-change code and `status` fields in update DTOs.
- [ ] **LOGIC-02** · High · `[API]` Prices, totals, discounts, commissions and quantities are calculated on the server from trusted data. Client-sent totals are never trusted. **Verify:** order and invoice creation.
- [ ] **LOGIC-03** · High · `[API][DB]` Money and stock operations are atomic (`$transaction`, row locks or conditional updates) and safe against races and double-submits. **Verify:** stock decrement and balance update code. Look for read-then-write outside a transaction.
- [ ] **LOGIC-04** · Medium · `[API]` Idempotency keys on create-payment, create-order and settlement endpoints. **Verify:** look for an `Idempotency-Key` header or a unique request ID.
- [ ] **LOGIC-05** · High · `[API][INFRA]` Rate limiting per user and per IP on expensive and abusable endpoints: OTP, login, search, export, report generation, file upload, invite. **Verify:** `@nestjs/throttler` config, Cloudflare rate-limit rules, Workers rate-limit bindings.
- [ ] **LOGIC-06** · Medium · `[API]` Pagination is enforced with a server-side maximum page size. No unbounded `findMany`. **Verify:** list endpoints, `take` limits.
- [ ] **LOGIC-07** · Medium · `[API]` Request body size limits, query depth/complexity limits, and timeouts on outbound calls. **Verify:** body-parser limits, `fetch` timeouts (`AbortSignal.timeout`).
- [ ] **LOGIC-08** · Medium · `[API]` Invites, referrals and onboarding can't be abused (enumerating invite codes, self-invites to a privileged role, unlimited invites). **Verify:** invite flow.
- [ ] **LOGIC-09** · Medium · `[ORG][API]` Usage alerts or spending caps on paid third-party APIs (SMS, maps, AI, email) so abuse doesn't become a large bill. **Verify:** `NEEDS-HUMAN` plus code-level limits.

---

## 7. API and web service security

*ASVS V4 (API and Web Service) · API7 (SSRF), API8 (Security Misconfiguration), API9 (Improper Inventory Management), API10 (Unsafe Consumption of APIs)*

- [ ] **API-01** · High · `[API]` CORS: an explicit allow-list of origins per environment. No `*` with credentials, and the `Origin` header isn't reflected back unchecked. **Verify:** `enableCors` config and worker CORS handling.
- [ ] **API-02** · Medium · `[API]` Swagger/OpenAPI UI and JSON are off in production, or protected. **Verify:** `SwaggerModule.setup` and whether it's gated by environment.
- [ ] **API-03** · Medium · `[API]` HTTP methods are restricted. Responses send `Content-Type` correctly, and JSON endpoints reject unexpected content types. **Verify:** content-type handling.
- [ ] **API-04** · High · `[API]` **SSRF:** any feature that fetches a URL from user input (image URL, webhook URL, import from URL) uses an allow-list of hosts and blocks private, link-local and metadata ranges. **Verify:** grep for `fetch(` and `axios` where the URL comes from user data.
- [ ] **API-05** · Medium · `[API]` **Inventory:** every deployed API version, environment and Worker is known and documented. Old or unused Workers, `*.workers.dev` URLs and `/v1` routes are removed. **Verify:** list routes and Workers from the `wrangler.toml` files. Flag `workers_dev = true` in production environments.
- [ ] **API-06** · High · `[API]` **Unsafe consumption:** responses from third-party APIs (payment, SMS, maps, partner systems) are validated with a schema and treated as untrusted. They use TLS, have timeouts, and follow redirects only if needed. **Verify:** third-party client code.
- [ ] **API-07** · High · `[API]` Incoming webhooks (payment provider, SMS status) verify the signature and timestamp (replay protection) before acting. **Verify:** webhook controllers.
- [ ] **API-08** · Medium · `[API]` Errors return a generic message plus a correlation ID, never stack traces, Prisma errors, SQL or internal paths (see LOG-06). **Verify:** global exception filter. Grep for `error.message` sent back in responses.
- [ ] **API-09** · Low · `[API]` The API contract (OpenAPI) is the source of truth, and undocumented routes don't exist. **Verify:** compare registered routes with the OpenAPI spec.

---

## 8. File handling

*ASVS V5 (File Handling)*

- [ ] **FILE-01** · High · `[API]` Uploads: an allow-list of types, checked by content (magic bytes) and not just extension or `Content-Type`, plus a size limit. **Verify:** upload handlers. `N/A` if there are no uploads.
- [ ] **FILE-02** · High · `[API][INFRA]` Uploaded files are stored outside the web root (R2/S3/Firebase Storage), use server-generated names, and are served with `Content-Disposition: attachment` or from a separate domain. **Verify:** storage code.
- [ ] **FILE-03** · High · `[API][INFRA]` Access to stored files is authorised per tenant (signed, short-lived URLs). Buckets aren't public or listable. **Verify:** bucket config, signed URL TTL.
- [ ] **FILE-04** · Medium · `[API]` No path traversal: user input is never joined into file paths. **Verify:** grep for `path.join(` and `readFile(` with user input.
- [ ] **FILE-05** · Medium · `[API]` Images are re-encoded or stripped of EXIF metadata (GPS location of farms) before being shared with other users. **Verify:** image processing.
- [ ] **FILE-06** · Low · `[API]` Archive uploads (zip) are protected against zip bombs and zip-slip, if supported.

---

## 9. Web frontend security

*ASVS V3 (Web Frontend Security) · A02, A05*

- [ ] **WEB-01** · High · `[WEB]` A Content-Security-Policy is set (via `_headers` on Cloudflare Pages, a Worker or `<meta>`): no `unsafe-inline`/`unsafe-eval` for scripts where avoidable, `object-src 'none'`, `base-uri 'self'`, `frame-ancestors 'none'`. **Verify:** `public/_headers`, `wrangler.toml`, `index.html`.
- [ ] **WEB-02** · Medium · `[WEB]` Security headers: `Strict-Transport-Security` (with `includeSubDomains`), `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy`. **Verify:** header config. `NEEDS-RUNTIME` to confirm on staging.
- [ ] **WEB-03** · Medium · `[WEB]` Protection against clickjacking (`frame-ancestors` or `X-Frame-Options: DENY`), especially on admin. **Verify:** headers.
- [ ] **WEB-04** · Critical · `[WEB]` No secrets in the client bundle. Every `VITE_*` and `PUBLIC_*` variable is safe to be public. Firebase web config is acceptable (it's public by design) **only** together with App Check and restricted API keys. **Verify:** grep `.env*`, `import.meta.env`, and `dist/` for keys, `secret`, `sk_`, `private`, service-account JSON.
- [ ] **WEB-05** · Medium · `[WEB]` Production builds don't publish source maps publicly, or they're uploaded privately to error tracking only. **Verify:** `vite.config.ts` `build.sourcemap`, and look for `*.map` files in `dist/`.
- [ ] **WEB-06** · Medium · `[WEB]` Third-party scripts loaded from CDNs use Subresource Integrity (SRI) or are self-hosted. **Verify:** `index.html`, Astro layouts.
- [ ] **WEB-07** · Medium · `[WEB]` Open redirects: `?next=`/`?redirect=` parameters only accept relative, same-origin paths. **Verify:** grep for `redirect`, `returnTo`, `next=`, `window.location =`.
- [ ] **WEB-08** · Low · `[WEB]` `target="_blank"` links to external sites use `rel="noopener noreferrer"`. **Verify:** grep.
- [ ] **WEB-09** · Medium · `[WEB]` The admin app isn't indexed by search engines (`noindex`, `robots.txt`) and isn't hosted on a guessable public path without auth. **Verify:** `robots.txt`, meta tags.
- [ ] **WEB-10** · Medium · `[WEB]` Marketing site forms (contact, lead, sign-up) have bot protection (Turnstile) and server-side validation. **Verify:** `kukkuone-marketing-web` form handlers.

---

## 10. Cryptography

*ASVS V11 (Cryptography) · MASVS-CRYPTO-1, CRYPTO-2 · M10 · A04*

- [ ] **CRYPTO-01** · High · `[ALL]` No custom cryptography. Only platform or vetted libraries (WebCrypto, `crypto`, the platform Keystore/Keychain). **Verify:** grep for hand-written XOR, base64 used as "encryption", custom ciphers.
- [ ] **CRYPTO-02** · High · `[ALL]` No weak algorithms for security purposes: MD5, SHA-1, DES, 3DES, RC4, ECB mode, or RSA with <2048 bits. **Verify:** grep for `md5`, `sha1`, `createCipher(` (deprecated), `ECB`, `DES`.
- [ ] **CRYPTO-03** · High · `[ALL]` Secure randomness for tokens, OTPs, invite codes and IDs (`crypto.getRandomValues`, `crypto.randomUUID`, `Random.secure()` in Dart). Never `Math.random()` or Dart `Random()`. **Verify:** grep for `Math.random`, `Random()`.
- [ ] **CRYPTO-04** · High · `[ALL]` Encryption keys aren't hard-coded or stored next to the data. Server keys live in a secret manager (Workers secrets). Mobile keys are created in and never leave Keystore/Keychain. **Verify:** grep for `key =`, `iv =`, `AES` with literal values.
- [ ] **CRYPTO-05** · Medium · `[ALL]` Authenticated encryption (AES-GCM / ChaCha20-Poly1305) with a unique IV/nonce per encryption. **Verify:** encryption code.
- [ ] **CRYPTO-06** · Medium · `[API]` If the system stores any password or secret token, it is hashed with Argon2id/bcrypt/scrypt (passwords) or HMAC-SHA-256 (API tokens). **Verify:** grep for `hash(`, `bcrypt`, `argon2`. `N/A` if Firebase handles all passwords.
- [ ] **CRYPTO-07** · Medium · `[API]` Secrets are compared in constant time (`timingSafeEqual`) for webhooks, API keys and HMACs. **Verify:** grep for `===` used on signatures.
- [ ] **CRYPTO-08** · Medium · `[DB]` Very sensitive fields (bank account, government ID, Aadhaar/PAN if collected) use field-level encryption, or aren't stored at all. **Verify:** `schema.prisma` field names.

---

## 11. Secure communication

*ASVS V12 (Secure Communication) · MASVS-NETWORK-1, NETWORK-2 · M5*

- [ ] **NET-01** · High · `[MOB]` Android: `android:usesCleartextTraffic="false"` (or not set with targetSdk ≥ 28), and `network_security_config.xml` has no `cleartextTrafficPermitted="true"` for production domains and no `<certificates src="user"/>` in release. **Verify:** `android/app/src/main/AndroidManifest.xml`, `res/xml/network_security_config.xml`.
- [ ] **NET-02** · High · `[MOB]` iOS: App Transport Security isn't turned off (`NSAllowsArbitraryLoads` not `true`), and exceptions are narrow and justified. **Verify:** `ios/Runner/Info.plist`.
- [ ] **NET-03** · Critical · `[MOB]` TLS validation isn't bypassed: no `badCertificateCallback = (...) => true`, no custom `HttpOverrides` that accept every certificate, no `onBadCertificate` returning true in release. **Verify:** grep Dart for `badCertificateCallback`, `HttpOverrides`, `onBadCertificate`.
- [ ] **NET-04** · Medium · `[MOB]` (MASVS-NETWORK-2) Certificate or public-key pinning for our own API domains, if the risk profile requires it, with backup pins and a way to rotate them. If it isn't used, record that as an accepted decision. **Verify:** pinning config.
- [ ] **NET-05** · High · `[ALL]` Every endpoint and base URL in release builds is `https://`. **Verify:** grep for `http://` in config, flavours, `.env`, `--dart-define` and `wrangler.toml` (ignoring `localhost` in dev configs).
- [ ] **NET-06** · Medium · `[INFRA]` Cloudflare: "Always Use HTTPS", minimum TLS 1.2, HSTS on. **Verify:** `NEEDS-HUMAN` (dashboard).
- [ ] **NET-07** · High · `[DB][INFRA]` The database connection uses TLS (`sslmode=require` or `verify-full`) outside local dev. The DB isn't reachable from the public internet, or it's limited to known egress (Hyperdrive or a private network). **Verify:** `DATABASE_URL` format in example env files, Hyperdrive config.

---

## 12. Configuration and secrets

*ASVS V13 (Configuration) · M1 (Improper Credential Usage), M8 (Security Misconfiguration) · A02*

- [ ] **CONF-01** · Critical · `[ALL]` No secrets committed to git, either now or in history: service-account JSON, private keys, DB URLs with passwords, API keys for paid services, `.env` files. **Verify:** run `gitleaks detect` (or `trufflehog git`) on each repo and its history. Check `.gitignore` covers `.env*`, `*.jks`, `*.keystore`, `key.properties`, `GoogleService-Info.plist`/`google-services.json` if considered sensitive, and `serviceAccount*.json`.
- [ ] **CONF-02** · Critical · `[INFRA][API]` `wrangler.toml` `[vars]` contain only non-secret values. Secrets are set with `wrangler secret put`. **Verify:** read every `wrangler.toml`.
- [ ] **CONF-03** · Critical · `[MOB]` No secrets in the mobile app: no server API keys, no admin credentials, no signing secrets in Dart code, `--dart-define`, assets or native config. Anything in the app binary is public. **Verify:** grep `lib/`, `assets/`, `android/`, `ios/` for `key`, `secret`, `token`, `password`, `Bearer`.
- [ ] **CONF-04** · High · `[MOB][WEB][INFRA]` Firebase/Google API keys (public by design) are restricted in Google Cloud Console to our app package/bundle ID and SHA fingerprints (Android) or HTTP referrers (web), and to the APIs actually used. **Verify:** `NEEDS-HUMAN` (console). List every key found.
- [ ] **CONF-05** · High · `[MOB]` Release signing keys and `key.properties` aren't in the repo. Upload keys are stored securely, and Play App Signing is enabled. **Verify:** `android/key.properties`, `*.jks` presence, `.gitignore`.
- [ ] **CONF-06** · High · `[API]` Config is validated at startup and fails fast (`@kukkuone/config` with Zod). Production can't start with dev defaults (e.g. a default JWT secret, `AUTH_PROVIDER=mock`, debug auth bypass). **Verify:** config schema. Grep for `bypass`, `mock`, `DEV_AUTH`, `skipAuth`, `if (process.env.NODE_ENV !== 'production')` around auth.
- [ ] **CONF-07** · High · `[API]` No backdoor or test logins in production code (hard-coded test users, master OTP like `123456`, a `?debug=true` auth bypass). **Verify:** grep for `123456`, `000000`, `test@`, `debug`, `bypass`, `backdoor`.
- [ ] **CONF-08** · Medium · `[INFRA]` Local `docker-compose.yml` doesn't expose DB or Redis ports to `0.0.0.0` with default passwords on a shared network, and isn't reused for production. **Verify:** `kukkuone-infrastructure/docker-compose.yml`.
- [ ] **CONF-09** · Medium · `[ALL]` Debug features are off in release: Flutter `kDebugMode` guards, no debug banners or menus, no verbose network logging (`LogInterceptor`), NestJS logger level. **Verify:** grep for `LogInterceptor`, `debugPrint`, `print(` around network or auth, `logger: ['debug'`.
- [ ] **CONF-10** · Medium · `[ORG][INFRA]` Least privilege for cloud accounts: Cloudflare API tokens scoped per use, Firebase/GCP IAM roles minimal, no shared personal accounts for deploys. **Verify:** `NEEDS-HUMAN`.

---

## 13. Data protection and privacy

*ASVS V14 (Data Protection) · MASVS-PRIVACY-1..4, MASVS-STORAGE-2 · M6 · India DPDP Act 2023 · GDPR (if EU users)*

- [ ] **PRIV-01** · High · `[ORG][DB]` A data inventory lists every personal data field (name, phone, email, address, GPS of farms, bank details, government IDs, photos), why it's collected, where it's stored, and how long it's kept. **Verify:** `06-Data-Model.md` vs `schema.prisma`. Report PII fields with no documented purpose.
- [ ] **PRIV-02** · High · `[MOB][WEB][API]` Data minimisation: collect only what the feature needs. Flag collected fields that are never used. **Verify:** compare forms and DTOs against usage.
- [ ] **PRIV-03** · High · `[MOB][WEB]` Consent and privacy notice are shown before collecting personal data, in the user's language (DPDP requires notice in English or a scheduled Indian language on request). **Verify:** onboarding flows and `l10n`.
- [ ] **PRIV-04** · High · `[MOB][API]` Account and data deletion inside the app (required by Apple and Google Play), which really deletes or anonymises the data across services, storage and Firebase Auth. **Verify:** look for a delete-account feature and trace the backend deletion.
- [ ] **PRIV-05** · Medium · `[API]` Data export / access request is possible (DPDP right to information, GDPR access and portability). **Verify:** endpoint or documented manual process.
- [ ] **PRIV-06** · Medium · `[DB][ORG]` Retention: old data (logs, OTPs, deleted accounts, inactive tenants) is purged on a schedule. **Verify:** cron jobs and TTLs.
- [ ] **PRIV-07** · High · `[MOB]` Platform privacy declarations match reality: iOS `PrivacyInfo.xcprivacy` (privacy manifest, required-reason APIs), App Store privacy labels, Google Play Data Safety form, and permission usage strings (`NSCameraUsageDescription`, `NSLocationWhenInUseUsageDescription`…). **Verify:** `ios/Runner/PrivacyInfo.xcprivacy`, `Info.plist`. Store forms are `NEEDS-HUMAN`.
- [ ] **PRIV-08** · Medium · `[MOB]` (MASVS-PRIVACY-1) Only the permissions needed are requested, at the moment they're needed. No `ACCESS_BACKGROUND_LOCATION`, `READ_CONTACTS`, `READ_SMS` or similar unless essential. **Verify:** `AndroidManifest.xml` `<uses-permission>`, `Info.plist` usage keys. Note that plugins can add permissions, so check the merged manifest (`build/app/intermediates/merged_manifests/`) if a build exists.
- [ ] **PRIV-09** · Medium · `[MOB][WEB]` (MASVS-PRIVACY-2) Third-party SDKs (analytics, crash, ads, attribution) are listed, and personal data isn't sent to them (no phone or email as the analytics user ID, no PII in event properties). **Verify:** `pubspec.yaml`, `package.json`, and grep for `logEvent`, `setUserId`, `setUserProperty`, `Sentry.setUser`.
- [ ] **PRIV-10** · Medium · `[ORG]` Data processing agreements with processors (Firebase/Google, Cloudflare, SMS provider, hosting DB), and data residency decided if required. **Verify:** `NEEDS-HUMAN`.
- [ ] **PRIV-11** · Medium · `[ORG]` A breach response plan meeting DPDP (notify the Data Protection Board and affected users) and GDPR (72h) timelines if applicable. **Verify:** `NEEDS-HUMAN`.

---

## 14. Logging, monitoring and error handling

*ASVS V16 (Security Logging and Error Handling) · A09 (Security Logging & Alerting Failures) · A10 (Mishandling of Exceptional Conditions)*

- [ ] **LOG-01** · High · `[ALL]` Logs never contain tokens, passwords, OTPs, full phone numbers, bank details or full request bodies with PII. **Verify:** grep for `console.log(req`, `logger.log(body`, `JSON.stringify(request`, `print(` / `debugPrint(` with responses, and logging interceptors.
- [ ] **LOG-02** · High · `[API]` Security events are logged in a structured way: login success/failure, token verification failure, 401/403, rate-limit hits, role changes, permission denials, admin actions. **Verify:** guards and interceptors.
- [ ] **LOG-03** · High · `[API][DB]` An audit trail for business-critical and admin actions (who, what, when, which tenant, before/after) that can't be edited by the actors it records. **Verify:** look for an audit-log table or model and where it's written.
- [ ] **LOG-04** · Medium · `[INFRA][ORG]` Logs are centralised and kept (Workers Logs/Logpush, crash reporting), with alerts for spikes in 401/403/5xx and unusual admin activity. **Verify:** `observability` in `wrangler.toml`. Alerting is `NEEDS-HUMAN`.
- [ ] **LOG-05** · Medium · `[ALL]` Every request has a correlation/request ID that is passed to services and returned in error responses. **Verify:** middleware.
- [ ] **LOG-06** · High · `[API]` A global exception filter turns unexpected errors into a generic 500 with no stack, SQL, Prisma error or file path. Prisma `P2002`/`P2025` are mapped to safe 409/404 messages. **Verify:** exception filters.
- [ ] **LOG-07** · High · `[API]` **Fail closed:** if auth verification, the permission check, a JWKS fetch or a config load throws, the request is denied, not allowed. **Verify:** check `try/catch` blocks in guards for `return true` or `next()` in the `catch`.
- [ ] **LOG-08** · Medium · `[API][MOB]` Exceptions don't leave partial state (transactions roll back), and unhandled promise rejections or `Future` errors are caught and reported. **Verify:** `runZonedGuarded` / `FlutterError.onError` / `PlatformDispatcher.onError` in `main.dart`, and transaction use.
- [ ] **LOG-09** · Medium · `[MOB]` Crash reports (Crashlytics/Sentry) are scrubbed of PII and tokens. **Verify:** `beforeSend` hooks and custom keys.

---

## 15. Supply chain and dependencies

*Top 10 A03 (Software Supply Chain Failures) · M2 (Inadequate Supply Chain Security) · MASVS-CODE-3 · ASVS V15*

- [ ] **DEP-01** · High · `[API][WEB][PKG]` No known high or critical vulnerabilities in npm dependencies. **Verify:** run `pnpm audit --prod` at the workspace root and list high/critical findings with the affected package and path.
- [ ] **DEP-02** · High · `[MOB]` No known vulnerable Dart/Flutter packages, and no discontinued packages. **Verify:** run `flutter pub outdated` and `dart pub deps` in each Flutter app. Check packages against OSV (`osv-scanner --lockfile=pubspec.lock` if available).
- [ ] **DEP-03** · Medium · `[ALL]` Lockfiles (`pnpm-lock.yaml`, `pubspec.lock`) are committed, and CI installs with a frozen lockfile. **Verify:** git status, CI scripts.
- [ ] **DEP-04** · Medium · `[ALL]` Packages come from trusted publishers. Watch for typosquats, abandoned packages (no release in more than 2 years) and packages with install scripts. **Verify:** review direct dependencies. Flag low-reputation or unmaintained ones.
- [ ] **DEP-05** · Medium · `[MOB]` Native SDKs pulled in by plugins (Firebase, Google Sign-In, etc.) are on current versions. **Verify:** `pubspec.lock`, `android/build.gradle`, `ios/Podfile.lock`.
- [ ] **DEP-06** · Medium · `[ORG][INFRA]` Automated dependency updates and scanning (Dependabot/Renovate plus CI audit). **Verify:** `.github/dependabot.yml`, `renovate.json`. Report as missing if CI isn't wired.
- [ ] **DEP-07** · Low · `[INFRA]` Docker base images are pinned by digest or specific version, not `latest`. **Verify:** Dockerfiles, `docker-compose.yml`.

---

## 16. Database (Prisma / PostgreSQL)

*ASVS V8, V13, V14 · API1, API3*

- [ ] **DB-01** · High · `[DB][INFRA]` The application DB user has least privilege: no superuser, and no `CREATE`/`DROP` at runtime. Migrations run with a separate role. **Verify:** connection-string users in example env files and docs. Otherwise `NEEDS-HUMAN`.
- [ ] **DB-02** · High · `[DB]` Every tenant-owned model has a non-null `tenantId`/`orgId` with an index, and foreign keys keep child rows inside the same tenant. **Verify:** `schema.prisma`. List tenant-owned models missing a tenant column.
- [ ] **DB-03** · Medium · `[DB]` Unique constraints include the tenant where needed (`@@unique([tenantId, code])`), so one tenant's data can't block another's. **Verify:** `schema.prisma`.
- [ ] **DB-04** · High · `[DB]` Seed scripts don't create default admin users with known passwords in non-dev environments. **Verify:** `prisma/seed.*`, `scripts/`.
- [ ] **DB-05** · Medium · `[DB][ORG]` Backups are encrypted, restores are tested, and point-in-time recovery is on in production. **Verify:** `NEEDS-HUMAN`.
- [ ] **DB-06** · Medium · `[DB]` Soft-deleted records are excluded from normal queries and from cross-tenant leaks. **Verify:** `deletedAt` handling.
- [ ] **DB-07** · Medium · `[DB]` Production data isn't copied into dev or staging without masking. **Verify:** scripts that dump or restore. Otherwise `NEEDS-HUMAN`.

---

## 17. Mobile: data storage

*MASVS-STORAGE-1, STORAGE-2 · M9 (Insecure Data Storage)*

- [ ] **MOB-STO-01** · High · `[MOB]` Sensitive data (tokens, PII, financial data) is stored only in Keychain/Keystore (`flutter_secure_storage`) or an encrypted DB with its key in Keystore. **Verify:** list all persistence: `shared_preferences`, `sqflite`, `drift`, `hive`, `isar`, `path_provider` file writes.
- [ ] **MOB-STO-02** · Medium · `[MOB]` `flutter_secure_storage` options: Android `encryptedSharedPreferences: true` (or the current default), iOS accessibility `first_unlock_this_device` / `unlocked_this_device` (not `always`). **Verify:** construction of `FlutterSecureStorage(...)`.
- [ ] **MOB-STO-03** · High · `[MOB]` Android backups don't include sensitive data: `android:allowBackup="false"`, or `fullBackupContent`/`dataExtractionRules` exclude it. **Verify:** `AndroidManifest.xml`, `res/xml/*backup*`.
- [ ] **MOB-STO-04** · Medium · `[MOB]` Cached API responses with sensitive data (offline cache, image cache of documents) are encrypted, or cleared on logout. **Verify:** cache libraries and their storage.
- [ ] **MOB-STO-05** · Medium · `[MOB]` (STORAGE-2) No sensitive data in logs (`print`, `debugPrint`, `log`), and release builds strip or disable verbose logging. **Verify:** grep in `lib/`.
- [ ] **MOB-STO-06** · Medium · `[MOB]` Sensitive text fields turn off autocorrect, suggestions and keyboard learning (`autocorrect: false`, `enableSuggestions: false`, `obscureText` for secrets, `autofillHints` correct). **Verify:** password, OTP and bank fields.
- [ ] **MOB-STO-07** · Medium · `[MOB]` The clipboard isn't used for sensitive values, or is cleared after a short time. **Verify:** grep for `Clipboard.setData`.
- [ ] **MOB-STO-08** · Low · `[MOB]` Files shared with other apps (PDF invoices, exports) go through a scoped `FileProvider`/share sheet. Sensitive files aren't written to public external storage. **Verify:** grep for `getExternalStorageDirectory`, `Downloads`.

---

## 18. Mobile: platform interaction

*MASVS-PLATFORM-1, PLATFORM-2, PLATFORM-3 · M4, M8*

- [ ] **MOB-PLT-01** · High · `[MOB]` Android components (activities, services, receivers, providers) have `android:exported="false"` unless they really need to be public, and exported ones are protected with permissions. **Verify:** `AndroidManifest.xml` (and the merged manifest if built).
- [ ] **MOB-PLT-02** · High · `[MOB]` Deep links and App Links / Universal Links: verified domains (`assetlinks.json`, `apple-app-site-association`), and handlers validate parameters. A deep link can't trigger a sensitive action (payment, role change, logout of others) without user confirmation and auth. **Verify:** `intent-filter`s, `Associated Domains`, routing code (`go_router` redirect logic).
- [ ] **MOB-PLT-03** · High · `[MOB]` (PLATFORM-2) WebViews, if used: JavaScript only when needed, no `addJavascriptChannel` exposing sensitive actions to untrusted pages, only our own HTTPS origins loaded, file access off. **Verify:** grep for `webview_flutter`, `InAppWebView`, `JavaScriptMode.unrestricted`, `addJavaScriptChannel`.
- [ ] **MOB-PLT-04** · Medium · `[MOB]` (PLATFORM-3) Sensitive screens (bank details, payment, OTP, reports) block screenshots and screen recording on Android (`FLAG_SECURE`) and hide content in the iOS app-switcher snapshot. **Verify:** grep for `FLAG_SECURE`, `secure_application`, `screen_protector`, or lifecycle-based blur.
- [ ] **MOB-PLT-05** · Medium · `[MOB]` Push notifications don't contain sensitive data in the visible text (amounts, health or flock disease data, OTPs beyond what's needed), and notification taps are validated like deep links. **Verify:** notification handling and backend push payloads.
- [ ] **MOB-PLT-06** · Medium · `[MOB]` Custom URL schemes aren't used for auth callbacks where a verified App Link / Universal Link is possible (schemes can be hijacked). **Verify:** `CFBundleURLSchemes`, scheme intent filters.
- [ ] **MOB-PLT-07** · Low · `[MOB]` Android `PendingIntent`s use `FLAG_IMMUTABLE`. Implicit intents don't carry sensitive extras. **Verify:** native Android code, if any.
- [ ] **MOB-PLT-08** · Medium · `[MOB]` Overlay and tapjacking protection on sensitive confirm buttons (Android `filterTouchesWhenObscured`) for high-risk actions. **Verify:** native config.

---

## 19. Mobile: code quality and updates

*MASVS-CODE-1, CODE-2, CODE-3, CODE-4 · M8*

- [ ] **MOB-COD-01** · Medium · `[MOB]` (CODE-1) Reasonable minimum OS versions (Android `minSdk` ≥ 24 recommended, iOS deployment target ≥ 15) and current `targetSdk` (as required by Play). **Verify:** `android/app/build.gradle(.kts)`, `ios/Podfile`, the Xcode project.
- [ ] **MOB-COD-02** · High · `[MOB][API]` (CODE-2) **Forced update:** the backend can block outdated app versions (minimum supported version check at startup, or `426`/custom error from the API with an update screen). **Verify:** look for a version-check mechanism in the app and gateway.
- [ ] **MOB-COD-03** · Medium · `[MOB][API]` Server-side kill switches / feature flags can turn off a vulnerable feature without an app release. **Verify:** Remote Config or a feature-flag service.
- [ ] **MOB-COD-04** · High · `[MOB]` Release builds have `debuggable=false` (the default for release), and no `android:testOnly`. Debug-only code is behind `kDebugMode`/`kReleaseMode`. **Verify:** Gradle build types, manifest.
- [ ] **MOB-COD-05** · Medium · `[MOB]` Different flavours (dev/staging/prod) can't point production builds at dev backends or use dev Firebase projects. **Verify:** flavour config, `firebase_options*.dart`, `--dart-define` usage.
- [ ] **MOB-COD-06** · Medium · `[MOB]` `analysis_options.yaml` has strict lints, and the app builds with no analyzer errors. **Verify:** run `flutter analyze`.

---

## 20. Mobile: resilience (reverse engineering and tampering)

*MASVS-RESILIENCE-1..4 · M7 (Insufficient Binary Protections)*

> Required for high-risk apps (money, settlements). The server checks in sections 2–6 are always the real protection. These controls only make attacks harder.

- [ ] **MOB-RES-01** · Medium · `[MOB]` Release builds use `--obfuscate --split-debug-info=<dir>`, and Android R8/ProGuard is on (`minifyEnabled true`). Debug symbols are uploaded privately to crash reporting. **Verify:** build scripts, CI, Gradle config.
- [ ] **MOB-RES-02** · High · `[MOB][API]` **App integrity attestation:** Firebase App Check (Play Integrity on Android, App Attest/DeviceCheck on iOS) is enabled and **enforced** for Firebase services and checked by our own API on sensitive endpoints. **Verify:** `firebase_app_check` in `pubspec.yaml`, activation code, server-side token checks. Enforcement mode is `NEEDS-HUMAN`.
- [ ] **MOB-RES-03** · Low · `[MOB]` (RESILIENCE-1) Root/jailbreak detection with a sensible response (warn or limit sensitive features, not a hard crash), if the risk profile needs it. **Verify:** grep for `flutter_jailbreak_detection`, `safe_device`, `freeRASP`.
- [ ] **MOB-RES-04** · Low · `[MOB]` (RESILIENCE-4) Anti-debugging and anti-hooking (Frida) checks for the highest-risk flows, if needed. **Verify:** RASP SDKs.
- [ ] **MOB-RES-05** · Medium · `[MOB]` (RESILIENCE-2) The app checks its own signature or package integrity, or relies on attestation (MOB-RES-02). **Verify:** as above.
- [ ] **MOB-RES-06** · Medium · `[MOB]` Business rules that matter (prices, limits, entitlements, feature access) aren't only enforced in the app. **Verify:** cross-reference with LOGIC-01/02 and AUTHZ-04.

---

## 21. Cloud, edge and Firebase configuration

*ASVS V13 · API8 · A02*

- [ ] **CLOUD-01** · High · `[INFRA]` Every `wrangler.toml`: production environment has `workers_dev = false` (unless intended), routes bound to the right zone, no secrets in `[vars]` (CONF-02), and observability turned on. **Verify:** read each `wrangler.toml`.
- [ ] **CLOUD-02** · Medium · `[INFRA]` Cloudflare WAF / managed rules and rate-limiting rules cover the API hostname. Bot protection on auth endpoints. **Verify:** `NEEDS-HUMAN`.
- [ ] **CLOUD-03** · High · `[INFRA]` Firebase project: App Check enforced; unused auth providers turned off; authorised domains list is minimal; phone-auth SMS region policy set; Firestore/Storage rules locked down if those are used (AUTHZ-12). **Verify:** `firebase.json` plus `NEEDS-HUMAN` for the console.
- [ ] **CLOUD-04** · Medium · `[INFRA]` Firebase service-account keys: none downloaded and stored in repos. Workload identity or Workers secrets are used. Old keys are rotated or deleted. **Verify:** grep for `"type": "service_account"`. Console part is `NEEDS-HUMAN`.
- [ ] **CLOUD-05** · Medium · `[INFRA]` R2/S3/Storage buckets are private by default, without public listing. CORS on buckets is restricted. **Verify:** config and `NEEDS-HUMAN`.
- [ ] **CLOUD-06** · Medium · `[INFRA]` DNS: no dangling CNAMEs to deleted services (subdomain takeover). Admin and staging subdomains are protected (Cloudflare Access) or not public. **Verify:** `NEEDS-HUMAN`.

---

## 22. Secure development lifecycle and CI/CD

*ASVS V15 · OWASP SAMM · A03, A08*

- [ ] **SDLC-01** · Medium · `[INFRA]` CI runs on every PR: lint, type-check, tests, `pnpm audit`, secret scanning (gitleaks), SAST (Semgrep/CodeQL), and `flutter analyze` for mobile. High/critical findings fail the build. **Verify:** CI config. The docs say CI isn't wired yet; report as missing.
- [ ] **SDLC-02** · Medium · `[INFRA][ORG]` Protected `main` branch: reviews required, no force-push, status checks required. **Verify:** `NEEDS-HUMAN` (`gh api repos/{owner}/{repo}/branches/main/protection` if connected).
- [ ] **SDLC-03** · Medium · `[INFRA]` Deploys come from CI with scoped tokens, not from developer laptops with full-access tokens. **Verify:** deploy scripts, `11-Local-Setup-And-Deploy.md`.
- [ ] **SDLC-04** · Medium · `[INFRA]` (A08) Build artefacts and mobile releases are signed. CI actions are pinned to a commit SHA. No `curl | bash` installs from unpinned URLs in build scripts. **Verify:** `setup-kukkuone.sh`, `scripts/`, CI workflows.
- [ ] **SDLC-05** · Medium · `[ORG]` Security review is part of code review for auth, payments, permissions and file handling. **Verify:** PR template / `CONTRIBUTING`. Otherwise `NEEDS-HUMAN`.
- [ ] **SDLC-06** · Medium · `[API][MOB]` Security tests exist: auth-required tests for every protected route, cross-tenant tests (AUTHZ-10), validation tests for malicious input. **Verify:** test folders. Report coverage.
- [ ] **SDLC-07** · Low · `[ORG]` A security point of contact, `security.txt` / responsible-disclosure page on the marketing site. **Verify:** `kukkuone-marketing-web/public/.well-known/security.txt`.

---

## 23. Vulnerability management and incident response

*ASVS V16 · OWASP SAMM (Operations)*

- [ ] **VULN-01** · Medium · `[ORG]` A vulnerability process: triage with CVSS plus business impact, fix-time targets (e.g. Critical 72h, High 7d, Medium 30d, Low 90d), and tracking until closed. **Verify:** `NEEDS-HUMAN`.
- [ ] **VULN-02** · Medium · `[ORG]` An incident response runbook: who decides, how to rotate secrets, revoke all sessions (SESS-04), force-update the apps (MOB-COD-02), turn off features (MOB-COD-03), notify users and regulators (PRIV-11). **Verify:** `NEEDS-HUMAN`.
- [ ] **VULN-03** · Medium · `[ORG]` A secret rotation procedure for each secret type (DB password, Firebase service account, Cloudflare tokens, third-party API keys), tested at least once. **Verify:** `NEEDS-HUMAN`.
- [ ] **VULN-04** · Medium · `[ORG]` An external penetration test against MASVS (mobile) and ASVS (API/web) before go-live and after major changes. **Verify:** `NEEDS-HUMAN`.

---

## Appendix A — Top 10 coverage map

Use this to confirm that every Top 10 item is covered by at least one check.

### OWASP Mobile Top 10 (2024)

| ID | Risk | Checks |
|---|---|---|
| M1 | Improper Credential Usage | CONF-01, CONF-03, CONF-04, CONF-05, AUTHN-09 |
| M2 | Inadequate Supply Chain Security | DEP-01..07, PRIV-09 |
| M3 | Insecure Authentication/Authorization | AUTHN-01..15, SESS-01..08, AUTHZ-01..12 |
| M4 | Insufficient Input/Output Validation | INPUT-01..14, MOB-PLT-02, MOB-PLT-03 |
| M5 | Insecure Communication | NET-01..07 |
| M6 | Inadequate Privacy Controls | PRIV-01..11, LOG-01, LOG-09 |
| M7 | Insufficient Binary Protections | MOB-RES-01..06 |
| M8 | Security Misconfiguration | CONF-06..09, MOB-PLT-01, MOB-COD-04, MOB-STO-03 |
| M9 | Insecure Data Storage | MOB-STO-01..08, SESS-01 |
| M10 | Insufficient Cryptography | CRYPTO-01..08 |

### OWASP API Security Top 10 (2023)

| ID | Risk | Checks |
|---|---|---|
| API1 | Broken Object Level Authorization | AUTHZ-01, 02, 03, 09, 10, 11 |
| API2 | Broken Authentication | AUTHN-01..15, SESS-03..05 |
| API3 | Broken Object Property Level Authorization | AUTHZ-05, AUTHZ-06 |
| API4 | Unrestricted Resource Consumption | LOGIC-05, 06, 07, 09 |
| API5 | Broken Function Level Authorization | AUTHZ-04, AUTHZ-08, ARCH-05 |
| API6 | Unrestricted Access to Sensitive Business Flows | LOGIC-01..05, LOGIC-08, AUTHN-07 |
| API7 | Server Side Request Forgery | API-04 |
| API8 | Security Misconfiguration | API-01..03, API-08, CONF-*, CLOUD-* |
| API9 | Improper Inventory Management | API-05, API-09, ARCH-02 |
| API10 | Unsafe Consumption of APIs | API-06, API-07 |

### OWASP Top 10 (web, 2025)

| ID | Risk | Checks |
|---|---|---|
| A01 | Broken Access Control | AUTHZ-*, ARCH-03, WEB-07 |
| A02 | Security Misconfiguration | CONF-*, WEB-01..03, CLOUD-*, API-01, API-02 |
| A03 | Software Supply Chain Failures | DEP-*, SDLC-04, WEB-06 |
| A04 | Cryptographic Failures | CRYPTO-*, NET-* |
| A05 | Injection | INPUT-01..13 |
| A06 | Insecure Design | ARCH-*, LOGIC-* |
| A07 | Authentication Failures | AUTHN-*, SESS-* |
| A08 | Software or Data Integrity Failures | SDLC-04, API-07, MOB-RES-02 |
| A09 | Security Logging & Alerting Failures | LOG-01..05 |
| A10 | Mishandling of Exceptional Conditions | LOG-06..08, API-08 |

### MASVS v2 control groups

| Group | Checks |
|---|---|
| MASVS-STORAGE | MOB-STO-*, SESS-01 |
| MASVS-CRYPTO | CRYPTO-* |
| MASVS-AUTH | AUTHN-*, SESS-* |
| MASVS-NETWORK | NET-01..05 |
| MASVS-PLATFORM | MOB-PLT-* |
| MASVS-CODE | MOB-COD-*, DEP-02, DEP-05, INPUT-14 |
| MASVS-RESILIENCE | MOB-RES-* |
| MASVS-PRIVACY | PRIV-* |

### ASVS 5.0 chapters

| Chapter | Checks |
|---|---|
| V1 Encoding and Sanitization | INPUT-03, 07, 10, 11, 12, 13 |
| V2 Validation and Business Logic | INPUT-01, 02, 04..06, LOGIC-* |
| V3 Web Frontend Security | WEB-* , SESS-06 |
| V4 API and Web Service | API-* |
| V5 File Handling | FILE-* |
| V6 Authentication | AUTHN-* |
| V7 Session Management | SESS-* |
| V8 Authorization | AUTHZ-* |
| V9 Self-contained Tokens | AUTHN-01..04, SESS-03 |
| V10 OAuth and OIDC | AUTHN-09, AUTHN-10 |
| V11 Cryptography | CRYPTO-* |
| V12 Secure Communication | NET-* |
| V13 Configuration | CONF-*, CLOUD-* |
| V14 Data Protection | PRIV-*, DB-* |
| V15 Secure Coding and Architecture | ARCH-*, DEP-*, SDLC-* |
| V16 Security Logging and Error Handling | LOG-*, VULN-* |
| V17 WebRTC | N/A (no WebRTC in KukkuOne) |

---

## Appendix B — Report template

The agent writes its result to `kukkuone-documentation/security-audit/YYYY-MM-DD-report.md` (or wherever the user says) in this format.

```markdown
# KukkuOne Security Audit — YYYY-MM-DD

## Summary
- Repos audited: …
- Repos skipped (no source yet): …
- Checks: N total · PASS n · FAIL n · PARTIAL n · N/A n · NEEDS-RUNTIME n · NEEDS-HUMAN n · NOT CHECKED n
- Failures by severity: Critical n · High n · Medium n · Low n

## Top risks (fix first)
1. [CHECK-ID] One-line description — repo — severity

## Findings
### [CHECK-ID] Title — FAIL — Critical
- Repo(s): …
- Evidence: `path/file.ts:123` — what the code does (quote ≤ 5 lines, secrets masked)
- Why it matters: concrete attack scenario in 1–2 sentences
- Fix: specific change
- Verify fix by: test or command that will prove it

## Full results table
| ID | Repo | Status | Severity | Evidence |
|---|---|---|---|---|

## Needs a human
| ID | Question for the team |
|---|---|

## Tools run
| Command | Result summary |
|---|---|
```

---

## Appendix C — Official sources

For the official requirement text, current versions and test procedures:

| Standard | Link | Machine-readable |
|---|---|---|
| MASVS | https://mas.owasp.org/MASVS/ | https://github.com/OWASP/masvs (releases) |
| MASTG (testing guide) | https://mas.owasp.org/MASTG/ | https://github.com/OWASP/mastg |
| MASWE (mobile weaknesses) | https://mas.owasp.org/MASWE/ | in the MASTG repo |
| ASVS 5.0 | https://owasp.org/www-project-application-security-verification-standard/ | https://github.com/OWASP/ASVS (CSV/JSON in releases) |
| API Security Top 10 2023 | https://owasp.org/API-Security/editions/2023/en/0x11-t10/ | — |
| Mobile Top 10 2024 | https://owasp.org/www-project-mobile-top-10/ | — |
| Top 10 2025 | https://owasp.org/Top10/ | — |
| Cheat Sheet Series | https://cheatsheetseries.owasp.org/ | https://github.com/OWASP/CheatSheetSeries |

OWASP content is published under CC BY-SA 4.0.
