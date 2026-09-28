# 12 — API Conventions

Every KukkuOne service speaks the **same HTTP dialect**. One base path, one auth
scheme, one response envelope, one error shape, one pagination model. The
**API Gateway** (`kukkuone-api-gateway`) is the only component exposed publicly;
it verifies the caller, mints an internal `Principal`, and routes to the domain
services (`kukkuone-farmer-api`, `-supplier-api`, `-distributor-api`) over the
trusted internal network. This document is the contract that clients and services
both compile against.

Related docs: [System Architecture](02-System-Architecture.md) ·
[Auth & RBAC](05-Auth-And-RBAC.md) ·
[Data Model](06-Data-Model.md) ·
[Farmer Module Spec](07-Farmer-Module-Spec.md)

---

## 1. Gateway routing

### 1.1 Path shape

All public traffic enters through the gateway at a single base, is namespaced by
API version, then by **domain prefix**. Nothing else is publicly reachable.

```
https://api.kukkuone.com / api / v1 / {domain} / {resource...}
                           └base┘ └ver┘ └──────── routed ───────┘
```

| Segment | Value | Notes |
|---------|-------|-------|
| Base | `/api` | Public entry; gateway owns it |
| Version | `/v1` | Major version only (see §9) |
| Domain | `core` \| `farmer` \| `supplier` \| `distributor` | Selects the internal service |
| Resource | `batches`, `daily-logs`, … | camelCase resource, lowercased plural path (§3) |

### 1.2 Domain → service routing table

The gateway holds no business logic for the domains — it strips the
`/api/v1/{domain}` prefix and proxies the remainder to the internal service base
URL configured by env (see [Environment & Config](10-Environment-And-Config.md)).
`core` is served by the gateway itself (auth, org, membership, capability, admin
aggregation).

| Public prefix | Handled by | Internal base URL (env) | Example rewrite |
|---------------|-----------|--------------------------|-----------------|
| `/api/v1/core/*` | Gateway (in-process) | — | `/api/v1/core/orgs` → in-process |
| `/api/v1/farmer/*` | `kukkuone-farmer-api` | `FARMER_API_URL` | `/api/v1/farmer/batches/place` → `${FARMER_API_URL}/batches/place` |
| `/api/v1/supplier/*` | `kukkuone-supplier-api` | `SUPPLIER_API_URL` | `/api/v1/supplier/orders` → `${SUPPLIER_API_URL}/orders` |
| `/api/v1/distributor/*` | `kukkuone-distributor-api` | `DISTRIBUTOR_API_URL` | `/api/v1/distributor/purchases` → `${DISTRIBUTOR_API_URL}/purchases` |

```ts
// gateway route map (illustrative)
const ROUTES: Record<string, string | null> = {
  core:        null,                      // in-process module
  farmer:      process.env.FARMER_API_URL!,
  supplier:    process.env.SUPPLIER_API_URL!,
  distributor: process.env.DISTRIBUTOR_API_URL!,
};
// GET /api/v1/farmer/batches/abc  ->  GET {FARMER_API_URL}/batches/abc
```

**Rules**

- The version segment is preserved end-to-end conceptually but stripped from the
  internal path; services are versioned in lockstep with the gateway.
- Dashboard/aggregation endpoints that fan out to several services live under
  `core` (e.g. `/api/v1/core/dashboard/farmer`) — the gateway composes them.
- A request to an unknown domain returns `404 NOT_FOUND`; a known domain whose
  service is unreachable returns `502` mapped to a `500`-class envelope (§4).

---

## 2. Authentication

### 2.1 Client → Gateway

Clients send the Firebase ID token as a bearer token on **every** request:

```http
Authorization: Bearer <Firebase ID token>
```

The gateway verifies the token with `firebase-admin` (via the `AuthServer`
adapter, see [Auth & RBAC](05-Auth-And-RBAC.md)), resolves the internal
`Principal` (`{ userId, orgId, capabilities[], roles[], permissions[] }`), and
enforces **coarse RBAC** (capability + action) before routing.

### 2.2 Gateway → Service (internal principal injection)

Domain services **never see the Firebase token and never re-verify it**. The
gateway injects the resolved principal as trusted internal headers on the
internal network:

| Header | Example | Meaning |
|--------|---------|---------|
| `X-User-Id` | `usr_9f2c…` | Authenticated user id |
| `X-Org-Id` | `org_4a1b…` | Active organization scope |
| `X-Capabilities` | `farmer,supplier` | Org capability set (CSV) |
| `X-Roles` | `owner,farm_manager` | Roles within the org (CSV) |
| `X-Permissions` | `batch:Create,batch:Dispatch` | Fine-grained `resource:Action` grants (CSV) |
| `X-Request-Id` | `req_7d3…` | Correlation id, propagated to logs |

```
Client ──Bearer Firebase ID token──▶ Gateway (verify + RBAC)
Gateway ──X-User-Id / X-Org-Id / X-Capabilities / …──▶ Service (trusts headers)
```

**Trust boundary.** These headers are authoritative **only** because services are
not publicly reachable — they bind to the internal network and reject traffic
that does not arrive from the gateway (network policy / shared internal secret
`INTERNAL_GATEWAY_SECRET`). A service must never accept these headers from the
public edge. See [System Architecture §Network](02-System-Architecture.md).

### 2.3 Token refresh

Refreshing the Firebase ID token is the **client's** responsibility. The
`@kukkuone/api-client` interceptor calls `getIdToken(forceRefresh)` on `401
AUTH_TOKEN_EXPIRED`, retries once, and surfaces the error if refresh fails. The
gateway does not issue or rotate tokens; it only verifies them.

---

## 3. Resource naming

- **Resources are nouns, plural, camelCase in code; paths are lowercase plural.**
  A resource named `dailyLog` is exposed at path `daily-logs`.
- **Actions on a resource collection** use a trailing verb segment only when the
  action is a state transition that is not a plain CRUD write (e.g.
  `/batches/place`, `/batches/{id}/mark-ready`, `/orders/{id}/dispatch`). These
  map to lifecycle transitions in [Data Model](06-Data-Model.md).
- **Nesting** expresses ownership, capped at **two levels** to keep URLs legible.
  Beyond that, filter on the child collection instead.

| Good | Avoid |
|------|-------|
| `/api/v1/farmer/batches` | `/api/v1/farmer/getBatches` |
| `/api/v1/farmer/batches/{id}/daily-logs` | `/api/v1/farmer/batches/{id}/logs/{logId}/entries/{n}` |
| `/api/v1/farmer/sheds/{id}/reset` | `/api/v1/farmer/resetShed?id=…` |
| `/api/v1/distributor/purchases?farmId=…` | `/api/v1/distributor/farms/{id}/batches/{id}/purchases` |

Path parameters are the resource id (CUID/UUID). Query parameters are for
filtering, sorting, and pagination only (§6) — never for identity or secrets.

---

## 4. Standard response envelope

Every response — success or error — uses one of two shapes.

### 4.1 Success

```jsonc
// 200 / 201 — single resource
{
  "data": { "id": "bat_01H…", "status": "PLACED", "…": "…" },
  "meta": { "requestId": "req_7d3…" }
}
```

```jsonc
// 200 — collection (see §6 for pagination meta)
{
  "data": [ { "id": "bat_01H…" }, { "id": "bat_02J…" } ],
  "meta": {
    "requestId": "req_7d3…",
    "pagination": { "limit": 20, "nextCursor": "eyJpZCI6…", "hasMore": true }
  }
}
```

`204 No Content` responses (e.g. idempotent deletes) carry no body.

### 4.2 Error

```jsonc
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "One or more fields are invalid.",
    "details": [
      { "field": "placedQty", "issue": "must be a positive integer" },
      { "field": "breed",     "issue": "required" }
    ]
  },
  "meta": { "requestId": "req_7d3…" }
}
```

`details[]` is always an array (empty when not applicable). `message` is
human-readable and safe to display; `code` is the stable, machine-branchable
value from the canonical list (§9).

### 4.3 HTTP status usage

| Status | Use for | Envelope |
|--------|---------|----------|
| `200 OK` | Successful read or non-creating write / transition | success |
| `201 Created` | Resource created (returns the new resource) | success |
| `204 No Content` | Success with no body (idempotent delete, ack) | none |
| `400 Bad Request` | Malformed request (bad JSON, missing header) | error |
| `401 Unauthorized` | Missing/invalid/expired token | error |
| `403 Forbidden` | Authenticated but lacks capability/permission | error |
| `404 Not Found` | Resource or route does not exist / not in caller's org | error |
| `409 Conflict` | Illegal state transition, or idempotency-key reuse conflict | error |
| `422 Unprocessable Entity` | Body well-formed but fails validation | error |
| `429 Too Many Requests` | Rate limit exceeded (`Retry-After` header set) | error |
| `500 Internal Server Error` | Unexpected server fault (or upstream `502/503`) | error |

Ownership rule: a resource in another org is reported as `404`, never `403`, so
existence is not leaked across org boundaries.

---

## 5. Validation

- **Schemas live in `@kukkuone/types` as Zod schemas** and are the single source
  of truth shared **client ↔ server**. Clients validate before sending; services
  validate at the edge. DTO types are inferred from the Zod schema
  (`z.infer<typeof PlaceBatchSchema>`).
- Services validate at the boundary with a **Zod validation pipe** (NestJS);
  where class-validator DTOs already exist they are equivalent, but Zod is
  canonical because it is the shared shape.
- A validation failure returns **`422 VALIDATION_FAILED`** with per-field
  `details[]` (`{ field, issue }`). Malformed transport (non-JSON body, missing
  required header) returns **`400`**, not `422`.

```ts
// @kukkuone/types
export const PlaceBatchSchema = z.object({
  shedId:   z.string().cuid(),
  breed:    z.string().min(1),
  placedQty: z.number().int().positive(),
  placedAt: z.string().datetime(),          // Day-0 timestamp
  chickSource: z.object({
    supplierOrgId: z.string().cuid().optional(),
    supplierOrderId: z.string().cuid().optional(),
  }).optional(),
});
export type PlaceBatchInput = z.infer<typeof PlaceBatchSchema>;
```

---

## 6. Pagination, filtering, sorting

### 6.1 Pagination

**Cursor-based is the default** (stable under inserts, used for feeds and large
collections). Offset paging is available for admin tables that need page numbers.

| Mode | Query | When |
|------|-------|------|
| Cursor (default) | `?limit=20&cursor=<opaque>` | Feeds, mobile lists, large sets |
| Offset (opt-in) | `?page=2&pageSize=20` | Admin grids needing page jump / totals |

`limit`/`pageSize` default to `20`, max `100`. The `cursor` is an opaque,
base64-encoded token — clients must not parse it.

```jsonc
// cursor meta
"pagination": { "limit": 20, "nextCursor": "eyJpZCI6ImJhdF8wMkoifQ==", "hasMore": true }
// offset meta
"pagination": { "page": 2, "pageSize": 20, "total": 137, "totalPages": 7 }
```

### 6.2 Filtering

Filters are explicit query params, ANDed together. Common filters:

| Param | Example | Applies to |
|-------|---------|-----------|
| `status` | `?status=ACTIVE` | Any lifecycle resource |
| `farmId` / `shedId` / `batchId` | `?farmId=frm_…` | Scoped collections |
| `from` / `to` | `?from=2026-01-01&to=2026-01-31` | Date-range on `createdAt` (override with `dateField`) |
| `q` | `?q=broiler` | Free-text search where supported |

All list endpoints are implicitly scoped to `X-Org-Id`; org is never a query
param.

### 6.3 Sorting

`?sort=` takes a comma-separated field list; a leading `-` means descending.
Default is `-createdAt`.

```
GET /api/v1/farmer/batches?status=ACTIVE&farmId=frm_1&sort=-createdAt&limit=20
```

---

## 7. Idempotency

Any **POST that creates a financial or state-changing record** — supplier order
submission, dispatch, invoice, settlement, distributor purchase, batch placement
— **must accept an `Idempotency-Key` header** (client-generated UUID v4).

```http
POST /api/v1/distributor/purchases/settle
Idempotency-Key: 6f1a2b7c-9d4e-4b21-8f0a-1c2d3e4f5a6b
```

**Guard behavior**

1. On first receipt, the service persists `(orgId, endpoint, idempotencyKey)` with
   the request hash and the resulting response, inside the same transaction as the
   write.
2. A **replay with the same key and same body** returns the **stored original
   response** (same status, same `data`) — the operation runs at most once.
3. A replay with the **same key but a different body** returns
   **`409 IDEMPOTENCY_CONFLICT`**.
4. Keys are retained ≥ 24h. Clients should reuse the same key when retrying after
   a network error — this is how "did my settlement go through?" is answered
   safely.

Money is always stored as **integer minor units + currency** (never floats); the
idempotency guard protects those records from double-writes.

---

## 8. Audit & ownership

Every business record carries the ownership/audit columns defined in
[Data Model §Common columns](06-Data-Model.md):

| Column | Source | Purpose |
|--------|--------|---------|
| `orgId` | `X-Org-Id` | Tenant/ownership scope (enforced on every query) |
| `status` | Lifecycle | Current state (see lifecycle diagrams) |
| `createdAt` / `updatedAt` | Server | Timestamps (UTC) |
| `createdBy` / `updatedBy` | `X-User-Id` | Actor attribution |

**Privileged actions** — anything in the `Approve / Dispatch / Settle` family (and
`Create` of financial records) — additionally write an **`AuditLog`** row
(`{ actorId, orgId, action, resource, resourceId, before, after, requestId, at }`).
Permission to perform them is checked against `X-Permissions` per
[Auth & RBAC §Permissions](05-Auth-And-RBAC.md). Reads are not audited; state
transitions always are.

---

## 9. Versioning & error codes

### 9.1 Versioning

- **URL major version only** (`/v1`). Additive changes (new optional fields, new
  endpoints, new enum values on output) ship **without** a version bump.
- **Breaking changes** (removing/renaming a field, changing a type, tightening
  validation) require `/v2`. `v1` and `v2` run side-by-side during migration.
- **Deprecation policy**: a deprecated endpoint returns a `Deprecation: true` and
  `Sunset: <date>` response header for **≥ 90 days** before removal, and is
  announced in [Decision Record](00-Decision-Record.md).

### 9.2 Canonical error codes

| `code` | HTTP | Meaning |
|--------|------|---------|
| `AUTH_TOKEN_MISSING` | 401 | No `Authorization` header |
| `AUTH_TOKEN_INVALID` | 401 | Token signature/issuer invalid |
| `AUTH_TOKEN_EXPIRED` | 401 | Token expired — client should refresh + retry |
| `FORBIDDEN` | 403 | Lacks required capability or permission |
| `VALIDATION_FAILED` | 422 | Body failed Zod validation (`details[]` set) |
| `BAD_REQUEST` | 400 | Malformed transport / missing required header |
| `NOT_FOUND` | 404 | Resource/route absent or outside caller's org |
| `CONFLICT_STATE` | 409 | Illegal lifecycle transition for current status |
| `IDEMPOTENCY_CONFLICT` | 409 | `Idempotency-Key` reused with a different body |
| `RATE_LIMITED` | 429 | Too many requests (`Retry-After` set) |
| `INTERNAL_ERROR` | 500 | Unexpected fault / upstream service failure |

Codes are stable API surface: clients branch on `error.code`, never on
`error.message`.

---

## 10. OpenAPI / Swagger

- **Each service** (gateway + three domain APIs) generates its own OpenAPI 3.1
  document from NestJS decorators + the Zod schemas, served at `GET /docs`
  (Swagger UI) and `GET /docs-json` (raw spec) — non-prod only, gated behind
  admin auth in staging.
- **The gateway composes** a unified spec: at boot it fetches each internal
  service's `/docs-json`, re-prefixes every path with `/api/v1/{domain}`, merges
  components (namespaced to avoid `$ref` collisions), and serves the aggregate at
  `GET /api/docs`. This gives clients one spec that mirrors the public routing
  table in §1.
- `@kukkuone/api-client` is generated from that composed spec, so client method
  names and the routing table never drift.

---

## 11. Worked example — `POST /api/v1/farmer/batches/place` (Day-0)

Creates a Day-0 batch in a prepared shed, transitioning the batch to `PLACED`
(see [Farmer Module Spec §Day-0](07-Farmer-Module-Spec.md) and the Batch
lifecycle in [Data Model](06-Data-Model.md)). It is state-changing and
optionally links a supplier order, so it takes an `Idempotency-Key`.

### Request

```http
POST /api/v1/farmer/batches/place HTTP/1.1
Host: api.kukkuone.com
Authorization: Bearer eyJhbGciOiJSUzI1NiIs…        // Firebase ID token
Content-Type: application/json
Idempotency-Key: 6f1a2b7c-9d4e-4b21-8f0a-1c2d3e4f5a6b

{
  "shedId": "shd_01HRA3F9",
  "breed": "Cobb 500",
  "placedQty": 12000,
  "placedAt": "2026-08-09T06:30:00.000Z",
  "chickSource": {
    "supplierOrgId": "org_9chickco",
    "supplierOrderId": "sord_01HRA1ZZ"
  },
  "expectedHarvestAt": "2026-09-20T00:00:00.000Z",
  "notes": "Placed 12k day-old chicks, avg 42g"
}
```

The gateway verifies the token, checks the org holds the `farmer` capability and
the caller has `batch:Create`, then proxies to
`${FARMER_API_URL}/batches/place` with the injected principal headers
(`X-User-Id`, `X-Org-Id`, `X-Capabilities`, `X-Roles`, `X-Permissions`,
`X-Request-Id`).

### Response — `201 Created`

```jsonc
{
  "data": {
    "id": "bat_01HRA4TQ7K",
    "orgId": "org_4a1b",
    "shedId": "shd_01HRA3F9",
    "breed": "Cobb 500",
    "status": "PLACED",
    "placedQty": 12000,
    "currentQty": 12000,
    "placedAt": "2026-08-09T06:30:00.000Z",
    "expectedHarvestAt": "2026-09-20T00:00:00.000Z",
    "chickSource": {
      "supplierOrgId": "org_9chickco",
      "supplierOrderId": "sord_01HRA1ZZ"
    },
    "createdAt": "2026-08-09T06:31:12.004Z",
    "updatedAt": "2026-08-09T06:31:12.004Z",
    "createdBy": "usr_9f2c"
  },
  "meta": { "requestId": "req_7d3f5a10" }
}
```

Placement also transitions the shed toward `RUNNING` and writes an `AuditLog`
row for the `Create` (§8). A retry with the **same** `Idempotency-Key` and body
returns this identical `201` payload; the same key with a different body returns
`409 IDEMPOTENCY_CONFLICT` (§7).

### Failure examples

```jsonc
// 422 — validation
{ "error": { "code": "VALIDATION_FAILED", "message": "One or more fields are invalid.",
  "details": [ { "field": "placedQty", "issue": "must be a positive integer" } ] },
  "meta": { "requestId": "req_7d3f5a11" } }

// 409 — shed not in a placeable state (EMPTY/CLEANING/…)
{ "error": { "code": "CONFLICT_STATE", "message": "Shed shd_01HRA3F9 is not READY for placement (status=PREPARING).",
  "details": [] }, "meta": { "requestId": "req_7d3f5a12" } }

// 403 — caller lacks batch:Create
{ "error": { "code": "FORBIDDEN", "message": "Missing permission batch:Create.", "details": [] },
  "meta": { "requestId": "req_7d3f5a13" } }
```

---

*Next: the phased build plan in
[13-Roadmap-And-Milestones](13-Roadmap-And-Milestones.md).*
