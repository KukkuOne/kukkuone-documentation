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

- **Withdrawal.** `MedicineUsage` exists and nothing stops a batch reaching
  harvest inside the withdrawal window. Residue in meat is the largest real
  liability here. Mark-ready and sale are blocked, with the clear date on the
  batch's Health segment.
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
| 3 | The app | `kukkuone-vet-mobile`: queue, case, diagnose, close | The loop end to end on two simulators, no AI involved |
| 4 | Marking and visits | The box on a photo, post-mortem media, the visit record and its fee into the batch's cost | `CaseFinding.region` round-trips; the fee shows in cost by head |
| 5 | The signals we hold | **Water to feed done, 4 Nov.** Heat stress, humidity on the form, `WeatherDay` and the forward alert | Alert tests at the thresholds; the July batch's figures unchanged |
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

Still to do in phase 5: `WeatherDay` — the outside temperature, which is what
separates bad ventilation from a hot day — and the forward alert, which would
be the first alert in the system that arrives before the damage.

Everything in phase 5 is app-side and **not released**; it goes out with the
next farmer-app build.

---

## 6. Cross-references

- 15 §3.2 — the Health and Growth segments this fills
- 06 — `VetCase`, `CaseMedia`, `CaseFinding`, `CaseDiagnosis`, `Disease`, `VetVisit`
- 05 — `Capability.VET`, `VET_PRACTITIONER`, and the guest read in `readMedia`
- 12 §gateway routing — the `vet` domain
- 14 LOGIC-09 — a spending cap on the AI provider before phase 6 ships
- 17 — the engine boundary: phases 5 to 7 compute inside farmer-api, and
  anything separate reads farm data rather than keeping its own copy
