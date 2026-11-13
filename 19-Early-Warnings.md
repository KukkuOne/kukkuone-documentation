# 19 · Early warnings and breed baselines

Status: building, 13 November 2026. Extends `16` §3.5–3.6 (layers 0–2 and the
signals we already store). Read-side only, per `17`: every warning is computed
from facts the app already holds, in `farmer-api/src/insight/alerts.service.ts`,
and stored only as an `Alert` row. Nothing here names a disease to a farmer.

## 1. What a warning is

An `Alert` row, as today, plus three things it lacked:

| Field | Type | Meaning |
|---|---|---|
| `severity` | `info` \| `watch` \| `act` | `act` = do something today; `watch` = look again tomorrow; `info` = for the record |
| `action` | code (below) | The one thing to do. The app words it; the server never sends prose for it |
| `params` | JSON object | The numbers the app needs to write the sentence in Telugu or English |
| `escalatedAt` | timestamp \| null | Set when an `act` alert stays unread for 24 h; it then shows to the owner as overdue |
| `batchId` | id \| null | The batch it is about, so it can open there |

`title` and `detail` stay, in English, as the fallback for a kind a client does
not know. Clients render known kinds from `kind` + `params`, localised.

Action codes: `RECORD_LOG`, `CHECK_WATER`, `CHECK_FEEDERS`, `CALL_VET`,
`ORDER_FEED`, `COOL_SHED`, `WARM_BROODER`, `PROTECT_LITTER`, `WEIGH_SAMPLE`,
`REVIEW_FEED`, `VACCINATE`, `PLAN_SALE`, `COLLECT_PAYMENT`, `PREPARE_SHED`.

## 2. The kinds

Every threshold is an `AlertSetting` the farm can change. A number no vet has
confirmed carries `TODO(vet): confirm` beside its default in code and
`basis: 'vetUnconfirmed'` in the settings response.

### 2.1 How a vet confirms a threshold

Three layers, the first that exists wins:

1. **The farm's own `AlertSetting`** for that kind — the farm's number, for
   that farm only.
2. **The platform value**, a `PlatformAlertThreshold` row (one per kind, set in
   the admin console's *Alert thresholds* page): `threshold`, `secondary` (the
   rule's second number, for the kinds that have one), who confirmed it
   (`confirmedByName`, `confirmedByRegistration` — the vet's council
   registration number — and `confirmedByUserId` when the vet has a KukkuOne
   account), `note` and `confirmedAt`.
3. **The code default**, `ALERT_THRESHOLD_DEFAULTS` in
   `packages/types/src/alert-thresholds.ts` — the one list of kinds, read by
   both farmer-api and the gateway. Each veterinary number there keeps its
   `TODO(vet): confirm`: it describes the fallback, which nobody confirmed.

`effectiveAlertDefault` (same file) is the one place layers 2–3 resolve. The
settings response's `defaultThreshold` is the platform value when there is one;
`basis` is `vetConfirmed` only when the row carries a vet's name, registration
and date, and then `confirmedBy: {name, registration, at}` comes with it.
Operational kinds stay `operational` and cannot be vet-confirmed — asking a vet
what hour counts as late would spend the marker. A farm cannot override a
`secondary`; the platform can. The 70 % humidity that pairs with
`HEAT_STRESS`'s second number stays code-only (`signals.ts` `SECONDARY`),
because a row holds one second number.

The super-admin records a confirmation through
`PUT /api/v1/admin/alert-thresholds/:kind` with `{threshold, secondary?,
confirmedByName, confirmedByRegistration, confirmedByUserId?, note}`:

- name **and** registration → stored as confirmed, dated now;
- neither, with a changed number → stored **unconfirmed** — the old
  confirmation was for the old number;
- neither, same numbers (a note edit) → the existing confirmation stays;
- one without the other → refused.

Every save appends a `PlatformAlertThresholdHistory` row (never updated or
deleted) and an `AuditLog` row under org `platform`
(`ADMIN_ALERT_THRESHOLD_CONFIRMED` / `_SET`). `GET /api/v1/admin/alert-thresholds`
returns each kind's code default, platform row, effective value and history.

The farmer app and the web portal show "Confirmed by Dr X (reg. no.), date"
where they used to show "Not yet confirmed by a vet". The vet confirms the
platform number; a farm that changed it is still told what the vet confirmed.

Farmer-api reads the platform rows once per settings read, which is once per
alert evaluation run. D1 migration `0019_platform_alert_thresholds.sql` must be
applied before the farmer-api and gateway that read these tables are deployed.

| Kind | Fires when (default) | Severity | Action | params |
|---|---|---|---|---|
| `NOT_LOGGED` (exists) | no daily log by evening | watch | RECORD_LOG | — |
| `MORTALITY_SPIKE` (exists) | deaths today ≥ 2 % of birds alive | act | CALL_VET | deaths, pct |
| `MORTALITY_CUMULATIVE` (exists) | cumulative ≥ 5 % | act | CALL_VET | pct |
| `MORTALITY_RISING` new | deaths rose 3 days running and today ≥ 0.1 % of alive | watch → act if ≥ 0.5 % | CALL_VET | days, deaths[] , pct |
| `FEED_DROP` new | feed a bird today < 85 % of its own 3-day average | watch; act with a water drop | CHECK_FEEDERS | todayG, avgG, dropPct |
| `WATER_DROP` new | water today < 85 % of its 3-day average, or water:feed below 1.6 | act | CHECK_WATER | todayL, avgL, ratio, dropPct |
| `FEED_BELOW_STANDARD` new | 3-day average feed a bird < 90 % of the breed baseline for that age | watch | REVIEW_FEED | age, intakeG, standardG, pct |
| `FEED_LOW` (defined, now raised) | farm feed stock covers < 3 days at the current rate | act | ORDER_FEED | days, bags |
| `FCR_DRIFT` (defined, now raised) | FCR > standard for age + 0.1 | watch | REVIEW_FEED | fcr, standard |
| `WEIGHT_BEHIND` (exists) | weight ≥ 8 % behind standard | watch | REVIEW_FEED | pct |
| `WEIGHT_NOT_SAMPLED` new | no weigh-in for 7 days after day 7 | info | WEIGH_SAMPLE | days |
| `HEAT_STRESS` new | today's shed reading ≥ 33 °C, or ≥ 30 °C with humidity ≥ 70 % | act | COOL_SHED | tempC, humidity |
| `HEAT_FORECAST` (exists) | tomorrow's maximum ≥ 38 °C | act | COOL_SHED | maxC, date |
| `COLD_FORECAST` new | a night below 18 °C in the next 3 days while the flock is ≤ 14 days old | act | WARM_BROODER | minC, date, age |
| `WET_WEATHER` new | ≥ 20 mm rain, or humidity ≥ 85 % on two days, in the next 3 | watch | PROTECT_LITTER | rainMm, humidity, date |
| `HEALTH_RISK` new | two or more of {feed drop, water drop, mortality rising or spike, weight behind} on the same batch within 48 h | act | CALL_VET | signals[] (kinds), score |
| `VACCINATION_DUE`, `BATCH_READY`, `SHED_IDLE`, `PAYMENT_OVERDUE` (exist) | unchanged | info/watch | VACCINATE / PLAN_SALE / PREPARE_SHED / COLLECT_PAYMENT | as today |

`HEALTH_RISK` is the "disease prediction" of this phase: a pattern and a
severity, never a name. It is Layer 0 arithmetic over Layer 0 signals (16
§3.5). The vet sees the same signals as context on a case; a ranked
differential stays the vet's alone.

## 3. Supervisor warnings

Managers and workers see every warning on their batches. An `act` warning
unread after 24 h gets `escalatedAt` and is shown to the owner as overdue,
with who should have acted.

### 3.1 Push (built 14 November 2026, waiting on the setup steps in §3.2)

The farmer app receives `act` warnings as push notifications through Firebase
Cloud Messaging, project `kukkuone-farm`. watch and info are never pushed;
they wait in the app.

| Event | Goes to |
|---|---|
| A new `act` alert (`Alert.pushedAt` null) | every owner, manager and worker (`FARMER_OWNER`, `FARMER_MANAGER`, `FARMER_OPERATOR`; not the accountant) who can see it: no `MemberScope` rows sees the whole org; a farm-scoped manager sees those farms; a shed-scoped worker sees those sheds, and farm-level alerts on the farms they sit in |
| An `act` alert escalated after it was pushed, still unread | owners only |

- **When.** At the end of every `AlertsService.evaluate` — the 3-hourly cron
  (§4) and the on-demand evaluate when someone opens the alerts list.
  `pushedAt` is set even when nobody has a device, so a phone registered
  tomorrow is not woken by today's backlog.
- **How often.** No quiet hours — these are farm emergencies. An alert row is
  one per kind per subject per day (`dayKey`) and today's row inherits
  `pushedAt` from yesterday's, so a condition that holds for a week is pushed
  once, and again only if it escalates.
- **Words.** Written server-side in the device's language (`en` / `te`,
  registered with the token), mirroring the app's own alert sentences:
  `farmer-api/src/push/messages.ts` copies the ARB strings and says so. Title =
  what happened; body = shed or farm · what to do (or "Overdue" for the
  owner's escalation). Data payload `{alertId, kind, batchId}`; a tap opens the
  batch, or the alerts list for a farm-level warning. Never a disease name.
- **Devices.** `DeviceToken(userId, orgId, token, platform, appFlavour,
  locale, createdAt, lastSeenAt)`, unique on `(token, orgId)` — migration 0018
  in both schemas, made per organisation by 0020. The app registers with
  `POST /api/v1/farmer/devices {token, platform, flavour, locale}` after
  sign-in, on FCM token refresh and when the language changes; the server
  writes one row for every active farm organisation the user is a member of
  (and the one they are in), so someone in two organisations is warned about
  both. Registering drops the token's rows for anyone else (a handed-on phone)
  and for organisations the user has left. `DELETE
  /api/v1/farmer/devices/:token` on sign-out removes the caller's rows for that
  token in every organisation. A token FCM reports `UNREGISTERED` (or
  `INVALID_ARGUMENT` naming the token) is deleted in every organisation — the
  verdict is on the token, not the organisation.
- **Sending.** `farmer-api/src/push/fcm.ts`: a JWT signed with WebCrypto RS256
  from the service-account key, exchanged for an OAuth2 access token cached
  until a minute before expiry, then FCM HTTP v1 `messages:send`. With no
  `FCM_SERVICE_ACCOUNT` secret it logs once and sends nothing; nothing fails.
- **In the app.** Permission is asked once, after sign-in and setup, with the
  reason on screen first (Android 13+ `POST_NOTIFICATIONS`, iOS's own prompt).
  A warning arriving while the app is open refreshes the alerts and shows the
  app's own banner — no system notification on top. Android channel
  `warnings` (high importance, created in `MainActivity`); small icon
  `ic_stat_warning`, the mark's white silhouette with the eye cut out, made
  by `tool/render_notification.swift` (run by `tool/brand.sh`), named both in
  the FCM message and as the manifest's `default_notification_icon`, tinted
  by `default_notification_color` — the app's accent, `#4B3BD6` light and
  `#A79BFF` dark. iOS: Push
  Notifications entitlement (`aps-environment`, development for Debug/Profile,
  production for Release) and Background Modes → remote notifications, for
  both flavours.
- **Rollout.** Migration 0018 marks every alert open at the time it is applied
  as already pushed, so the first run after deploy does not send the backlog.
  Apply it to D1 before deploying the farmer-api that reads `pushedAt`.

### 3.2 Setup the owner of the Firebase and Cloudflare accounts must do

None of these were done by the code change; push stays silent until they are.

1. **Service account for sending.** In the Google Cloud console for project
   `kukkuone-farm`: IAM & Admin → Service Accounts → Create service account
   (e.g. `farmer-api-push`), grant it the role **Firebase Cloud Messaging API
   Admin** (`roles/firebasecloudmessaging.admin`), then Keys → Add key → JSON.
   From `kukkuone-farmer-api/`, store the whole JSON file as the secret, once
   per environment:

   ```
   npx wrangler secret put FCM_SERVICE_ACCOUNT < farmer-api-push.json
   npx wrangler secret put FCM_SERVICE_ACCOUNT --env dev < farmer-api-push.json
   ```

   Then delete the downloaded file. Never commit it.
2. **FCM API.** In the same project, APIs & Services → make sure **Firebase
   Cloud Messaging API (V1)** is enabled (Firebase console → Project settings →
   Cloud Messaging shows it as Enabled).
3. **APNs key for iOS.** Apple Developer → Certificates, Identifiers &
   Profiles → Keys → create a key with **Apple Push Notifications service
   (APNs)**, download the `.p8` once, note its Key ID and the Team ID
   (`FL8PARABU2`). Firebase console → Project settings → Cloud Messaging →
   Apple app configuration → upload the `.p8` with both IDs for **each** iOS
   app (`com.naptrixlabs.kukkuone.farmer` and
   `com.naptrixlabs.kukkuone.farmerdev`). Both App IDs need the Push
   Notifications capability enabled; Xcode's automatic signing does this when
   it next builds with the new entitlements.
4. **Database.** Apply `kukkuone-database/prisma/d1/0018_device_tokens.sql`
   and then `0020_device_token_per_org.sql` to `kukkuone-dev` and `kukkuone`,
   confirm with the marker query in `prisma/d1/README.md` (`m0018 = 1`,
   `m0020 = 1`), then deploy `kukkuone-farmer-api`. Without 0020 a second
   organisation's registration fails on the old unique index.

## 4. When it runs

A cron trigger on `farmer-api` every 3 hours: refresh `WeatherDay` for every
farm with a location (Open-Meteo, as `weather.ts` already does), then evaluate
alerts for every organisation with an active batch. Opening the alerts list
still evaluates on demand.

## 5. Breed baselines

Platform baselines for the breeds farms actually raise (Ross 308, Cobb 500,
Cobb 430), from each breeder's own published performance objectives, kept
with their source in `kukkuone-database/prisma/reference/breed-standards/`.
Stored as `BreedStandard` rows with `orgId = 'platform'` and a `source`. A
farm's own standard for a breed wins; otherwise the platform baseline is
served, marked as such. Points gain `dailyFeedG` and `cumFeedG` beside
`weightG` and `fcr`. Nothing is interpolated: a day the source does not give
is a day without a point.
