# 16 · Health Intelligence and the Vet Loop

Status: proposed, 1 November 2026. Extends the Health and Growth segments of
15 §3.2, and builds the Intelligence extension point named in 01 §3.2.

## 1. The ask

A farmer photographs what worries them in a batch. A vet looks, marks the
lesion on the photo, names the disease, prescribes, and visits when the photos
are not enough — photographing what they find for the record. Everything a
batch carries feeds prediction: disease, growth, and FCR.

## 2. What the audit found

Read-only passes over the four APIs, the schema, the gateway and the farmer
app. Three quarters of this is built or modelled already.

**The data is there.** `DailyLog` holds deaths, culls, feed, water,
temperature, humidity, a weight sample and uniformity per day;
`WeightRecord`, `FeedConsumption`, `MedicineUsage` and `VaccinationSchedule`
hang off the batch; `BreedStandard` gives the curve to read against. Alerts
already compute mortality spike, cumulative mortality, weight behind standard
and FCR drift (`alerts.service.ts:7-12`).

**The case is modelled in full.** `VetCase` with its six statuses,
`CaseMedia` with six photo kinds, `CaseFinding` with organ, lesion, severity
and a `region` box on a named photo, `CaseDiagnosis` with a ranked disease, an
evidence grade and advice, `VetVisit` with birds examined and a fee, and
`trainingConsent` with its own timestamp on the case.

**Only the farm's half is built.** Raise, consent, attach a photo, read one
back (`kukkuone-farmer-api/src/vet`). There is no endpoint that writes a
finding, a diagnosis, a visit or a status, and no surface a vet can open.
`Capability.VET` and `VET_PRACTITIONER` exist in `rbac.ts` and are wired to
nothing.

**Five faults sit in the way.**

| Fault | Where |
|---|---|
| `vetOrgId` is never written by any code, so a raised case can reach no vet | `vet.service.ts:90` |
| The case queries filter on `orgId` alone, so a vet's principal reads an empty list | `vet.service.ts:33`, `:42`, `:235` |
| The gateway knows three domains and `forward()` conflates the URL prefix with the upstream name | `proxy.controller.ts:18`, `:48` |
| The catalogue is 14 entries and nothing serves it, so a vet picking a disease has no list to pick from | `prisma/diseases.ts` |
| `water`, `tempC` and `humidity` are written and read by nothing; humidity is not on the log form | `batches.service.ts:569-572` |

`readMedia` is the exception and the model for the rest: it already admits the
assigned vet as a guest on farm data (`vet.service.ts:213`).

**No AI anywhere.** No provider, no key, no inference path, no weather.

## 3. The design

### 3.1 One app for the vet, no new service

A vet works in `kukkuone-vet-mobile`, Flutter, a fifth app on the pattern of
04 §7. Vets are their own organisations — a panel of practices, not
employees — which `VetCase.vetOrgId` already assumed and which onboarding
already supports, since `VET` is a capability an org may select at signup.

The vet's half of the API does **not** get a service of its own. It goes into
the existing `kukkuone-farmer-api/src/vet` module, scoped by `vetOrgId`
instead of `orgId`, for the reason that module already states: the data is
farm data and a vet is a guest on it rather than the owner of a separate
record. The gateway gets an `api/v1/vet/*` route, `vet: ['VET']` in
`DOMAIN_CAPABILITIES`, and a domain-to-upstream lookup so the `vet` prefix
reaches the farmer service. Auth, OTP, workspaces, members and scopes are
untouched.

### 3.2 The endpoints

| Route | Does | Writes |
|---|---|---|
| `GET /vet/cases` | the queue, by `vetOrgId`, most urgent first | — |
| `GET /vet/cases/:id` | the case, its photos, and the batch's numbers | — |
| `POST /vet/cases/:id/claim` | take an unassigned case | `vetOrgId`, `vetUserId`, → `TRIAGED` |
| `POST /vet/cases/:id/findings` | organ, lesion, severity, the box on the photo | `CaseFinding` |
| `POST /vet/cases/:id/diagnoses` | the ranked disease, evidence, advice, medicine | `CaseDiagnosis` |
| `POST /vet/cases/:id/visit` | a visit is needed, or happened | `VetVisit`, → `VISIT_NEEDED` / `VISITED` |
| `POST /vet/cases/:id/media` | the vet's own post-mortem photos | `CaseMedia` |
| `POST /vet/cases/:id/close` | close it | → `CLOSED`, `closedAt` |
| `GET /vet/diseases` | the catalog to pick from | — |

### 3.3 The app

| Screen | Shows | Enters |
|---|---|---|
| **Queue** | Cases by urgency: dying now, raised today, waiting on a visit. Farm, batch age, birds affected, first photo | Claim |
| **Case** | The photos large, the complaint, and the batch beside them: age, mortality today and cumulative, weight against standard, feed and water trend, shed and outside temperature, vaccinations given, what is already medicated | Ask the farm a question |
| **Mark** | One photo, full screen | Draw a box, pick organ, lesion, severity |
| **Diagnose** | The ranked shortlist, the catalog | Up to three diseases, evidence grade, advice, medicine with dose, duration and withdrawal days |
| **Visit** | Schedule, then the visit itself | Post-mortem photos, birds examined, summary, fee |

A vet diagnosing from a photo alone is guessing, so the batch context on the
Case screen is not decoration — it is most of the diagnosis.

### 3.4 Routing a case

The farmer picks their vet, from the practices on the platform matched against
the farm's own `PartyKind.VET` parties. A farm with no vet of its own raises
into the pool, where any panel practice may claim it. No admin routing.

### 3.5 Four layers of analysis, and only one needs a model

| Layer | What | Needs |
|---|---|---|
| 0 | FCR, mortality curve, weight against standard, feed cover — and the two new signals in §3.6 | Arithmetic. Built, extend it |
| 1 | Weather per farm per day, and a three-day forecast | One provider, one cached table |
| 2 | Projected harvest weight, date, FCR at close, margin at close | The breed curve and the batch's own growth ratio. No ML |
| 3 | Vision triage of a case's photos against its batch context | Claude, with the catalog as the answer set |
| 4 | A model of our own, on vet-confirmed labels | A few thousand confirmed cases. Later, or never |

Layer 3 runs once when a case is raised with photos, and writes
`CaseDiagnosis` rows ranked 1–3 with `evidence: PRESUMPTIVE` and
`diagnosedBy: 'ai'`. Its job is to sort the vet's queue and narrow the
question, never to answer the farmer: a vision model is fair on gross lesions
and unreliable on the subtle ones that matter most. What the farmer sees
before a vet signs is urgency and first aid — isolate, do not medicate yet,
a vet in four hours. Prescribing on a model's guess is the one thing in this
product that can harm birds, the people who eat them, and us.

Agreement between the AI's rank 1 and the vet's confirmed diagnosis is the
metric that says whether Layer 3 earns its cost.

Layer 4 trains only on cases where `trainingConsent` is true. Disease history
is commercially damaging, and a farm must be able to show a vet without
donating the photos to a dataset.

### 3.6 Signals we already store and never read

| Signal | From | Says |
|---|---|---|
| Water to feed, ~1.8–2.0:1 and rising with heat | `DailyLog.water`, `feedConsumed` | A fall precedes visible sickness and mortality by a day or two; a spike is heat stress or a leak. The earliest signal in the whole dataset |
| Heat stress | `DailyLog.tempC`, `humidity`, and the weather feed | Above about 29 °C with high humidity, intake falls and FCR drifts — which otherwise reads as a growth fault |
| Shed against outside | the same two | Separates bad ventilation from a hot day |
| Season | the weather feed | Coccidiosis after wet litter, CRD after cold nights. A prior for Layer 3 |

Humidity joins the daily log form. `WeatherDay(farmId, date, tempMin, tempMax,
humidity, rainfall)` is pulled daily per farm location and cached; the
forecast gives a forward alert — 38 °C on Thursday, fit the foggers, feed in
the cool hours — which is the first alert in the system that arrives before
the damage.

### 3.7 Three obligations the health module carries

- **Withdrawal.** ~~`MedicineUsage` exists and nothing stops a batch reaching
  harvest inside the withdrawal window.~~ **Wrong when written, and §5g says
  what was actually true.** The lot path had been blocking it since the money
  module was built, and the clear date was already on the Health segment.
  Three *other* ways out had no check at all; they do now. Residue in meat is
  still the largest real liability here.
- **Notifiable disease.** `Disease.notifiable` is already a column. A
  confirmed notifiable case is a reporting obligation, not an alert card.
- **Area outbreak.** Cases across farms carry locations. "Three farms within
  15 km confirmed CRD this week" warns a farmer and primes Layer 3.
  Aggregate only; another farm is never named.

### 3.8 What stays as it is

The farm's half of the case and its screens, the raw-body photo upload and its
private read path, consent, the daily log form apart from humidity, the
alerts and their settings, the batch hub of doc 15, auth and onboarding.

## 4. Defaults chosen, and what to say to change them

- **Vets are a panel of their own organisations.** Say "in-house vets" and it
  becomes one org with no claim step and no pool.
- **The farmer picks, the pool catches the rest.** Say "pool only" to drop the
  picker, or "admin routes" to move it to `kukkuone-admin-web`.
- **No AI diagnosis reaches the farmer before a vet signs it.** Say "show the
  AI to the farmer" to reverse it, knowing §3.5.
- **The vet app has no offline write queue**, only a local draft of the open
  diagnosis. Say "offline vet app" to carry farmer-mobile's queue across.
- **Weather from Open-Meteo**, free and keyless, daily min, max, humidity and
  rain. Say the provider to change it.

## 5. Phases

Each phase ends green on the API tests, the Flutter suites of both apps, and a
simulator walk. Phase 5 is independent of every other and may go any time.

| # | Phase | Lands | Proof |
|---|---|---|---|
| 0 | Catalog | ~25 diseases seeded with Telugu names and the notifiable flag; `GET /vet/diseases` | Seed test; the list on a device |
| 1 | The vet's half | **API done, 4 Nov.** The `vet` domain in the gateway; findings, diagnoses, visit, media, close; the case queries admit the assigned vet | A test per endpoint; a vet principal reads a case it was given and nothing else |
| 2 | Routing | **API done, 4 Nov.** The farmer picks a vet or raises into the pool; claim | `vetOrgId` set on raise or claim; a pool case is claimable once |
| 3 | The app | **Built 5 Nov, released nowhere.** `kukkuone-vet-mobile`: sign-in, the practice, queue, case, diagnose, close | 44 tests and a clean analyze; no device run, no sign-in against the live gateway |
| 4 | Marking and visits | **Built 5 Nov, deployed nowhere.** The box on a photo, post-mortem media, the visit record and its fee into the batch's cost | `CaseFinding.region` round-trips under test; the fee books once per visit, unpaid, as VET |
| 5 | The signals we hold | **API deployed 5 Nov, app unreleased.** Water to feed, humidity on the form, the shed's conditions, `WeatherDay`, the outside line and HEAT_FORECAST | 1,089 Flutter tests and 86 API tests; both migrations applied to both databases |
| 6 | Triage | Layer 3 on raise, ranked and presumptive; the agreement metric | A fixture case returns three ranked rows; the farmer sees urgency only |
| 7 | Forecast | Projected weight, date, FCR and margin at close on the Growth segment | Back-tested against closed batches |

Phase 3 is the checkpoint. If vets will not use the app, no amount of AI
saves it. Phases 6 and 7 are worth nothing before phase 1 of doc 15 — the
figures an AI reasons over have to be right first.

## 5a. What phase 1 landed

Built 4 November, deployed the same day from a clean archive of `develop`.

- **The gateway** gained a `vet` domain. `forward()` had built the upstream
  URL out of the domain alone, so the URL prefix, the service and the upstream
  path were one word; `DOMAIN_ROUTE` separates them, which is what lets
  `/api/v1/vet/*` be served by farmer-api. That is the fault §2 records at
  `proxy.controller.ts:18` and `:48`, now closed.
- **Eight endpoints** under `api/farmer/vet-desk` — the queue, one case,
  claim, findings, diagnoses, visit, close, the catalogue — plus the vet's own
  photo upload and read. Not `api/farmer/vet`: the farm's half already answers
  there scoped by `orgId`, and one path cannot serve two queues.
- **The scoping fault is closed.** §2 records it three times: the case queries
  filter on `orgId`, and a vet's org never owns the case. The vet's half
  filters on `vetOrgId`, in one tested predicate rather than three inline
  `where` clauses.
- **A vet sees their own cases and the unclaimed pool**, and the pool stops at
  RAISED. Claiming is conditional, so two vets opening the pool together do
  not both believe they have it. Nothing may be written onto a case nobody has
  claimed — a finding with no one answerable for it is training data nobody
  signed.
- **The case carries the batch's figures**, read from `BatchesService.summary`
  rather than computed again. A batch whose numbers disagree with the farmer's
  screen would be worse than no numbers.

Deployed: farmer-api `90d4de0f`, gateway `f0b2ef4c` in production; `fdd5663e`
and `7f37fbc7` on dev. Inert in production, because no org holds `VET` there —
the dev database has a seeded practice and the catalogue to try it against.

## 5b. What phase 2 landed

`vetOrgId` is written now, on both paths: `raise` takes the vet the farm
picked, and `claim` takes a case out of the pool. Both proofs §5 asks for are
met, and the one that needed care is claiming — it is conditional on the case
still being unclaimed, so two vets opening the pool together do not both
believe they have it.

`GET /farmer/vet/practices` is the picker. §3.4 says the farmer picks from the
practices on the platform "matched against the farm's own `PartyKind.VET`
parties", and **matched** needed saying plainly: `Party` carries no link to an
`Organization`, because a farm's vet is a name and a phone number in their own
address book, written down long before that vet had an account. So the join is
the phone, compared on its last ten digits — `+91 98765 43210`, `09876543210`
and `919876543210` are one farm's one vet.

`Party.linkedOrgId` would be tidier and is not worth a forward-only migration
in both schemas for a flag on a picker. If a second thing ever needs that
link, that is the moment to add it.

The farm's own vet leads the list and everybody else follows alphabetically: a
farm whose vet has no account yet still needs somebody to send to, and a list
of one with a dead end beside it is worse than a list.

The picker is on the farmer app's Ask a vet sheet, between the batch and the
bird counts. It reads **"Any vet"** when nothing is picked rather than sitting
empty: a farm raising into the pool has answered the question, and a blank row
would read as an unfinished form on a sheet somebody is filling in with dying
birds in front of them.

Deployed 4 November: farmer-api `4c66c562` in production, `7afe41bb` on dev.
Inert in production — no org holds `VET` there, so the practices list comes
back empty and every case still goes to the pool. **The app half is built and
not released**; it goes out with the next farmer-app build.

One thing a sign-out test caught that a reading would not: `vetPracticesProvider`
looks platform-wide, but its `known` flag is computed against the farm's own
parties, so it is account data and must be dropped when somebody signs out. It
watches `sessionProvider`, like `vetCasesProvider` beside it.

---

## 5c. The first of the signals

`DailyLog.water` had been written by the log form and read by the log's own
detail screen and the export — recorded and never used, which is what §2 means
by the three signals nothing reads.

It is a third chart on Growth now, beside weight and deaths a day. Water is
stored in millilitres and feed in grams, and a millilitre of water weighs a
gram, so the ratio is the two figures divided: no conversion, and none to get
wrong.

**No threshold and no verdict.** The band is quoted near 1.8–2.0:1, no vet has
confirmed it for these birds in this climate, and the farmer is shown the shape
instead. A coloured figure would be a claim where a line is an observation. The
function names the spot a confirmed band goes into, marked `TODO(vet): confirm`
like the five already live in `alerts.service.ts`.

A day with water and no feed is skipped rather than divided — that is half a
filled-in form, not a day the birds drank and did not eat.

**Humidity is on the log form**, beside the temperature, because heat is the
two together and one without the other says very little.

That turned up a data-loss bug neither §2 nor this document had named. The
form is authoritative over the day — *a field left empty here was emptied* —
so with no humidity field it sent `humidity: null` on **every save**, and
anything ever recorded was wiped by the next ordinary log. Temperature was in
the same position by a different route: written by the form, never put back
into it on re-open, and so erased by the next save of a day somebody opened to
correct a death count. Both are prefilled now.

One thing left open rather than worked around: the form fetches the day
through `BatchRepo.logs` rather than `logsProvider`, deliberately, so it reads
fresh before overwriting a whole day. That call cannot be stubbed from the
screen harness, so the prefill path is read rather than tested. Giving the
form the provider, or the harness that repo, would close it; changing a fetch
that guards against overwriting a day with stale data was not something to do
in passing.

**The shed's own weather is drawn**, directly under the weight chart, because
that is what it explains. One series with the humidity in the readout — 31 °C
and 68% do not share a scale, and the feed chart already answers that question
the same way.

**No index and no threshold**, for the same reason as water to feed: combining
heat and humidity into one number means picking a published
temperature-humidity index *and* a cut-off, and neither is confirmed for these
birds in this climate. The farmer gets the shape; a vet decides what counts as
too hot.

So all three of the signals §2 records as "written and read by nothing" now
read: water against feed, and the shed's temperature with its humidity.

**A farm has a point on the ground now**, which it did not. `Farm.location`
was the village as the farmer says it and nothing else — no coordinates in the
schema, no location permission in either manifest, no package. A farm is often
kilometres outside the village it is named after, so the name was never going
to answer what Thursday will do.

`Farm.latitude` and `longitude`, nullable, taken from the phone on a tap while
somebody stands at the farm. **Not geocoded from the village name**: that
would attribute a position to somebody's birds that nobody chose, silently,
and be wrong by kilometres for exactly the farms that are out on their own.
Null is a working state — the farm saves without it and only the weather is
not offered.

This is the first permission the app has ever asked for, so: never on open,
never in the background, no `ACCESS_BACKGROUND_LOCATION`, and the reason sits
on the row above the button because by the time the OS prompt appears the
decision is already made. A refusal is not an error.

**Not applied anywhere.** The migration is committed in both schemas and run
against no database yet.

## 5d. The weather, and the alert that arrives first

`WeatherDay(farmId, date, tempMin, tempMax, humidity, rainfall)` exists, in
both schemas, and it is a cache rather than a record: any row may be replaced
by a later fetch of the same day, and a forecast is expected to change.

**Open-Meteo, as §4 chose** — free, keyless, and verified against the live
endpoint before anything was written, because four field names guessed from
memory is four ways for an alert to quietly never fire again. The recorded
answer is the fixture in `test/weather.test.ts`, so a renamed field fails a
test rather than a farm.

`date` is a `TEXT` of YYYY-MM-DD, as `Alert.dayKey` is, and not a timestamp.
"Thursday at this farm" is a calendar day in India, the worker reading it runs
in UTC, and an instant would drift a day at either end — at the one place a
weather row is read, which is the place it matters.

**A farm is asked about to two decimal places**, about 1.1 km. That is finer
than the provider's own grid, so the forecast loses nothing, and the exact
spot somebody stood in their own shed does not need to leave the platform to
find out what the weather will do. Two farms in one village share a cache row
into the bargain.

**Fetched on a read, not on a schedule.** There is no cron in this deployment
— the alert engine already re-evaluates when somebody opens the app — and
adding a scheduler to answer a question being asked right now is the long way
round. A fetched forecast is believed for six hours; a day that has already
happened is written once and skipped on every fetch after, which is what
keeps a refresh at four writes rather than eighteen.

Nothing in the feed throws at a caller. A farm with no coordinates and a
provider that is down are the same thing to every screen: no weather, and the
rest of the page as it was. Five seconds and the request is abandoned, because
a provider that hangs must not hang a list of alerts.

### The outside temperature

The shed chart now draws the outside maximum dotted behind the shed's own
line, and names it. That is the whole reason the table holds a fortnight of
past days rather than only yesterday: an outside line covering the last day of
a fourteen-day window is a stub, and the comparison is the point. 34 °C in a
shed is a ventilation fault when it is 29 °C outside and nothing but the
weather when it is 35, and neither line alone says which.

It is a second series on the house trend chart — dotted, unmarked, sharing the
axis, with its figure beside the shed's in the readout — rather than a new
chart. A farm that has never given its coordinates gets the shed line it
always had.

### HEAT_FORECAST

The first alert in this system that arrives before the damage rather than
reporting it. One per farm, not one per shed: the sky is the same across a
farm and a farmer with five sheds does not need the same Thursday five times.
That also means no per-shed mute, which makes it the second kind after
`PAYMENT_OVERDUE` to skip it.

Days ahead only, and the **nearest** crossing day leads rather than the
hottest — the foggers have to go up before the first one, and a farmer told
about Saturday will not act on Wednesday. It reads as *Green Valley — 41 °C on
Thursday. Humidity around 30%. Fit the foggers and feed in the cool hours.*
Amber, not red: something to do today about a day that has not arrived.

**The threshold is 38 °C and no vet has confirmed it.** This one had two
candidate numbers and they are not close. §3.6 puts the start of trouble near
29 °C, where intake falls and FCR drifts — and an alarm at 29 would fire on
most days between March and October in Telangana, which is the
`MORTALITY_CUMULATIVE` mistake again: a threshold that fires during an
ordinary week teaches a farmer to ignore it. 38 is where heat stops being a
drag on growth and starts killing birds, so that is what is live, carrying
`TODO(vet): confirm` like the five already in `alerts.service.ts`. The right
figure almost certainly moves with humidity and with the age of the birds — a
day-35 bird in feather suffers at a temperature a chick does not notice.

### What is not done

- ~~Neither migration is applied to any database.~~ **Both are applied to
  both databases and the API is deployed**, 5 November, version
  `49f09525`, from a clean archive of `develop`. The app half is still
  unreleased and goes out with the next farmer build, so the outside line and
  the "use my location" row are not on anybody's phone yet — the alert,
  however, is live, and will fire the first time a farm gives its
  coordinates.

  Applying the two to production is worth a line, because it did not go
  cleanly: `0009` came back `Authentication error` from the Cloudflare API
  while `0010`, seconds later, applied fine. Production briefly had the
  `WeatherDay` table and no `Farm.latitude` — a state no migration describes
  and `farms.service.ts` cannot write to. Re-running `0009` fixed it, and
  `prisma/d1/README.md` now carries the warning: the exit status of a batch of
  these is worth nothing, ask the database.
- **The alert text is English from the server**, as every other alert's title
  and detail already is. Only the kind label is localised, and
  `alertKindHeatForecast` joins its ten siblings reading English in
  `app_te.arb` — one Telugu label among eleven English ones would read worse
  than eleven English ones. It is in the ratchet.
- **Season as a prior** (§3.6's fourth signal) is untouched. The rainfall
  column is stored and nothing reads it.
- `istDayKey` assumes India, marked as a shortcut with the line that undoes
  it: read `utc_offset_seconds` from the provider's own answer.

So phase 5 is complete as §5 scoped it — water to feed, humidity on the form,
the shed's own conditions, `WeatherDay`, and the forward alert. The API half
is **deployed**; the app half is **unreleased** and goes out with the next
farmer build.

---

## 5e. What phase 3 landed

`kukkuone-vet-mobile`, built 5 November, released nowhere. Three screens of
§3.3 — the queue, the case and the diagnosis — and 44 tests.

### An app of its own, and why that needed arguing

The root `CLAUDE.md` says a new counterparty role belongs in
`kukkuone-partner-mobile` rather than in a new app. That is right for the roles
it means, and `kukkuone-partner-mobile/lib/core/models/roles.dart` gives the
test it is applying: the five roles there are "one kind of software — publish a
catalogue or an offer, receive an order, agree terms, record a dispatch or a
pickup, settle". A vet does none of those. A clinical queue, a photograph with
a lesion on it and a ranked differential are a different kind of software,
which is the same argument that file already makes for keeping farming out.

So §3.1 stands. Two things found while checking it:

- `Capability` in that file has six values and no `VET`, so the partner app's
  hand-maintained catalogue is already out of step with the API's enum —
  `fromWire('VET')` returns null there today.
- The root `CLAUDE.md` sentence is a generalisation that does not fit this
  role. It is left as it is rather than edited in passing.

### No Firebase, and the phone is the point

A vet signs in with a mobile number and a code. Nothing else.

The gateway composites its verifiers with the session one first
(`packages/auth/src/providers/composite.ts`), so a token this platform minted
at `/auth/otp/verify` is accepted by the deployed gateway even though it
reports `authProvider: firebase`. That was checked in the source before the
app was built on it, not assumed.

It means no Firebase project, no Google sign-in, and three fewer dependencies
than the farmer app carries. More than that, the phone is the *right*
credential rather than the cheap one: `Party` carries no link to an
`Organization`, so `practicesForFarm` joins a farm's own vet on the number
(§3.4) — and a vet signing in with the number their farms already have written
down is recognised by those farms without anybody doing anything.

### The queue, the case, the diagnosis

The queue separates cases in hand from the pool, because they are different
decisions: one is work, the other is an offer, and taking it is a commitment
another practice then cannot make. The server's order is kept exactly as sent;
a queue sorted twice is a queue whose order nobody owns.

Every figure on the case screen was read from the server and not one is
computed in Dart. A case with no batch says the figures are absent rather than
showing zeros. Photos come through the authenticated endpoint and are cached
per launch — there is no `Image.network` and there must not be, because the
bucket is private and the request needs a bearer that `Image.network` cannot
send.

A notifiable disease says so when it is picked, and says that recording it is
not reporting it (§3.7).

### A privacy fault in phase 1, found by building the consumer

`GET /vet/cases/:id` returned `BatchesService.summary` straight through, and
that is the farm's own batch screen — it carries `totalExpensesMinor`,
everything the farm has spent on those birds. A vet opening a case about
gasping birds was reading the farm's cost base.

It is the argument this module already makes about photos, pointed the other
way: a farm's disease history is commercially damaging, and so is what it pays
for feed. A vet is a clinical guest, not a commercial one, and a vet who also
trades birds is not a hypothetical person. Fixed as an allow-list rather than a
delete, so a figure added to the farm's summary later does not reach a vet
because nobody remembered the function existed.

### Localisation, from the first commit

Both ARB files, every string through `L`, and four guard tests. One of them
closes a hole the farmer app's own ratchet cannot see: a key present in
`app_en.arb` and missing from `app_te.arb` falls back to English in silence,
and the farmer app's ratchet only compares keys that exist in both.

28 strings are translated, taken verbatim from the farmer app where they have
already been read by somebody who speaks Telugu. 77 read English and are
listed in `test/untranslated_telugu.json`; a test holds them at that number.
Translating 77 at once would be 77 unreviewed translations landing in one go,
which is a worse problem than the one it fixes.

### What is not there

- **Mark and Visit**, the other two screens of §3.3. `CaseFinding.region`
  exists and nothing writes one; findings are read on the case screen and
  cannot be entered. Both are phase 4.
- **Most of what §3.3 asks the Case screen to show.** Weight against the
  breed standard, the feed and water trends, the shed and outside temperature,
  vaccinations given and what the birds are already medicated with are all
  named there and none is sent: the case endpoint returns the batch summary
  and no series. Medicine with a dose, a duration and withdrawal days has no
  field on `CaseDiagnosis` to go in at all. §3.3 describes a screen the API
  cannot yet fill, and that gap is phase 4's to close rather than something
  the app should fake.
- **No device run.** 44 tests and a clean analyze; nothing has been built for
  a simulator or signed in against the live gateway, because a real sign-in
  spends an SMS on a number somebody owns.
- **No iOS or Android release config.** The scaffold is `flutter create`'s.

## 5f. What phase 4 landed

Built 5 November, released nowhere, deployed nowhere.

### The box on a photo

`CaseFinding.region` has existed since phase 1 and nothing had ever written
one. The Mark screen writes it, on the column's own terms: fractions of the
image rather than pixels, so a resize or a thumbnail does not invalidate every
label ever drawn.

The arithmetic is the risky part of the whole phase, so it is a pure function
in `marking.dart` with sixteen tests on it. A box off by a few per cent looks
right on the screen it was drawn on and labels the wrong thing forever, and
the pair of image and region is training data (§3.5) — this is the one place
where a sloppy implementation would quietly poison a dataset rather than
visibly break a screen.

What makes the conversion exact: the image is laid out at **its own aspect
ratio**, so the widget's rectangle *is* the image's rectangle and a drag
divides straight down to a fraction. Letterboxed inside a `BoxFit.contain` it
would have needed the displayed rect worked out first, which is the kind of
sum that is wrong by a few per cent forever. Also handled: a drag up and to
the left is the same box as one down and to the right; a flick too thin in
either direction is not a box; a drag off the edge is clamped rather than
claiming part of an image that does not exist.

The organ and lesion pickers keep the enums' own order, which is anatomical
rather than alphabetical — trachea, lungs, air sacs, then the heart sac, then
down through the gut. A vet opening a bird works in roughly that order.

### Post-mortem photos

An action on the case, not a screen. The case already shows every photo on
it, the vet's included — they hang off the same case and come back through the
same private read — so a second gallery would be a second place to look at one
set of images.

A sheet rather than straight to the camera, because a vet may have taken the
picture an hour ago on the way back. Resized to 2048 on the long edge before
it goes: a modern phone's full-resolution photograph is several megabytes, the
endpoint takes twelve, and a vet on a farm road is uploading over whatever
signal there is.

### The visit, and the one thing a vet writes that is not clinical

The visit screen is one screen for needed and done, because they are the same
visit at two moments. The calendar only opens backwards for a visit that
happened and forwards for one that is coming.

`VetVisit.feePaise` has also existed since phase 1 with nothing booking it, so
a farm's vet bills were invisible in cost by head. They are not now, and this
is the only thing in the entire vet module that writes rather than reads —
onto somebody else's books — so every rule it follows sits in `visit-fee.ts`
beside the code and is pinned by a test:

- The cost is the **farm's**, against the case's `orgId`, never the vet's.
- It lands **unpaid**, attributed to the vet's own user id, with the practice
  named in the note. A vet can create a debt here; a farmer seeing a fee they
  did not expect must be able to see who put it there.
- **Only a visit that happened.** A promise is not a bill. The server applies
  that, not the client, so the rule lives in one place.
- A fee of zero or null books nothing. A free visit is a visit, not a cost of
  ₹0 taking up a line in a breakdown.
- A batch cost carries no `farmId`, so the farm-level reports cannot count it
  twice.
- The date is the day of the visit, not the day it was typed.
- Idempotent on `billKey` = `vet-visit:<id>`, so a retry on a bad line — which
  is how a phone on a farm road behaves — charges the farm once.

**`ExpenseCategory` gained `VET`.** `MEDICINE` would have taken the fee and
conflated the consultation with the drug bill, so a farm comparing medicine
spend across batches would have been reading two different things added
together. It is **not** in the farmer app's `expenseHeads` picker, for the
same reason `RENT` is not: a farmer never types it, and offering it would
invite a second manual copy of a cost already on the batch. Postgres only —
`Expense.category` is `TEXT` in the D1 schema.

### What is still not there

- **The API is deployed**, 5 November: dev `a2d3bfd3`, production
  `28f840b8`, both from a clean archive of `develop`.
  `20261105140000_vet_expense_head` needs no D1 file at all — it adds a value
  to an enum *type* and `Expense.category` is `TEXT` there, which
  `prisma/d1/README.md` now says once for every migration of that shape.
  **Neither app is released**, so nothing reaches a phone: the vet app has
  never been built for either platform, and the farmer app's next build is
  what carries the `VET` cost head onto a farmer's screen.

  So a fee booked today would land in the database and be invisible in the
  app — correctly, as an `Expense` row the farmer's current build renders
  under "Other" through `headLabel`'s fallthrough, rather than as nothing.
- **Medicine with a dose, a duration and withdrawal days.** §3.3 asks for it
  on the Diagnose screen and `CaseDiagnosis` has no field for it. The
  withdrawal obligation in §3.7 — blocking mark-ready and sale inside the
  window — is a schema change and a phase of its own, and it is the largest
  real liability this document names.
- **The rest of the Case screen §3.3 describes.** Weight against the breed
  standard, the feed and water trends, the shed and outside temperature,
  vaccinations given and current medication are still not sent by the case
  endpoint.
- **No device run, still.** 64 tests in the vet app and 1,090 in the farmer
  app; nothing has drawn a box with a finger, and the box arithmetic has never
  been checked against a real photograph on a real screen.

## 5g. Withdrawal, and a lesson about reading the code first

§3.7 said "nothing stops a batch reaching harvest inside the withdrawal
window". That was not true, and building from it rather than from the code
produced a second implementation of a figure that already existed — the one
fault this codebase's first rule exists to prevent. The work was thrown away
and done again. Writing it down because the document caused it.

### What was already there

- `MoneyService.safeSaleDate(orgId, batchId)` — last dose plus the withdrawal
  days, taking whichever course ends latest and falling back to
  `MedicineItem.withdrawalDays` when the course's own is zero.
- A gate on the **lot** path in `money.service.ts`, which refuses a lift
  inside the window *unless* the owner types a reason — and the reason is
  recorded against the sale.
- `GET money/batches/:id/safe-sale-date`, which the Sell birds screen already
  calls.
- The clear date on the Health segment, and a preview of it on the Add
  medication form.

### What was actually missing

Three other ways for birds to leave, none of which checked anything:

| Path | Judged on |
|---|---|
| `HarvestService.markReady` | the target harvest date the farmer names |
| `HarvestService.recordSale` | today — a second sale path, never guarded |
| `OffersService.create` | the offer's own `readyDate` |

All three read `safeSaleDate` rather than recomputing it. The dates differ on
purpose: marking a batch ready on the 7th for a lift on the 12th is correct
even when the birds clear on the 9th, and refusing that would refuse a correct
action — a farmer who cannot plan around a window will work around it instead.
An offer for a date inside the window is a promise the farm will break with a
distributor's lorry already sent, so it is refused on the date offered. A sale
is judged on the day it happens, because that is when meat enters the food
chain.

**None of the three takes an override.** The lot path's override is a real
clinical answer because its reason is written against the sale; an override
with nowhere to record the reason is not an override, it is a hole.

A test reads the source of all four paths rather than calling them, which
needs no database and guards the fault that produced this section: a new way
to sell birds added without a check. A test that exercised one path would have
passed while three others leaked, which is how this survived.

### The duplication that is staying

`MedicineCourse.safeFrom` in the farmer app does the same arithmetic in Dart.
It is kept, and `safeFrom` now says why in the code: the Add medication form
needs the date before the course is saved, when there is no server to ask, and
a form preview that round-trips per keystroke would be worse for no gain while
the two agree — same last dose, same days, same arithmetic. What it also says
is that the server is the authority, that four paths enforce its reading
whatever the getter returns, and that if the boundary ever moves it moves in
both places in one commit.

### Not done

- **A zero withdrawal period is ambiguous, and the ambiguity is now said
  out loud but not resolved.** The mechanism, traced since: the Add
  medication form starts its withdrawal field at `'0'`, and
  `MoneyService.safeSaleDate` does `if (days === 0) continue` — so a farmer
  who records a course and never touches that field has silently cleared the
  birds for sale, and nothing downstream asks again. There is no catalogue to
  fall back on: `MedicineItem.withdrawalDays` is documented as "the label's
  figure" and is typed by the same farmer, defaulting to 0 in two more forms.

  What is there now is a notice on that form — "No withdrawal: these birds
  may be sold the day the course ends. The number of days is on the label." —
  so the claim is stated where the label is in the farmer's hand, with a test
  that fails if it goes silent again. That makes a wrong zero noticeable. It
  does not make it impossible.

  What is still missing is the third state. `withdrawalDays` is
  `Int @default(0)` on both `MedicineItem` and `MedicineUsage`, so "none,
  confirmed" and "nobody filled it in" cannot be distinguished, and no
  report can say which courses are unrecorded. Making the column nullable is
  the honest model and the right moment is before release, while the data is
  small — but on D1 it is a SQLite table rebuild on two live tables, and
  every existing 0 stays ambiguous whatever is done. Simply making the field
  required is not the answer: a farmer who has lost the label would be unable
  to record the dose at all, and an unrecorded medication is worse than one
  with an unknown period.
- **Deployed.** Production runs the gates as of version `18ee8b06`
  (farmer-api `af31002`).
- **No test drives a real refusal.** All four gates need a database to reach,
  so what is pinned is that each path calls the check and names the date, not
  that a sale is actually refused.

## 6. Cross-references

- 15 §3.2 — the Health and Growth segments this fills
- 06 — `VetCase`, `CaseMedia`, `CaseFinding`, `CaseDiagnosis`, `Disease`, `VetVisit`
- 05 — `Capability.VET`, `VET_PRACTITIONER`, and the guest read in `readMedia`
- 12 §gateway routing — the `vet` domain
- 14 LOGIC-09 — a spending cap on the AI provider before phase 6 ships
- 17 — the engine boundary: phases 5 to 7 compute inside farmer-api, and
  anything separate reads farm data rather than keeping its own copy
