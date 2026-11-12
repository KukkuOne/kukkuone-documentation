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
with who should have acted. Push delivery is not part of this phase — the apps
have no push channel yet (no FCM/APNs setup); warnings arrive in the app and
the web portal.

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
