# 17 · The AI Engine Boundary

Status: decided, 1 November 2026. Replaces the handover `CLAUDE.md` written for
the Poultry AI Engine and supersedes its §3 architecture, §4 repository layout
and §5 data model. Point a new session at this document, not at that file.
Read with 16, which is the plan this boundary serves.

## 1. The decision

**The engine is read-side only.** KukkuOne is the system of record for every
farm fact. The engine computes and returns; it never writes farm data, and
until phase 3 it stores nothing at all.

## 2. Why

The engine was designed without the application. Its data model re-creates
what the Prisma schema already holds:

| The engine designed | KukkuOne already has |
|---|---|
| `observations`, with a generic envelope | `DailyLog`, `WeightRecord`, `FeedConsumption`, `MedicineUsage`, `VaccinationSchedule` |
| `cases`, `case_reviews`, `labels` | `VetCase`, `CaseMedia`, `CaseFinding`, `CaseDiagnosis`, `VetVisit`, `trainingConsent` |
| `reference_curves` | `BreedStandard`, `BreedStandardPoint` |
| `prescriptions` and withdrawal | `MedicineUsage` |
| `daily_unit_stats` | the rollups in `reports.service.ts` and `pnl.service.ts` |
| `insights` | `Alert`, `AlertSetting`, `AlertMute`, with the rules in `alerts.service.ts:7-12` |
| `units`, `batches`, `attachments`, `case_events` | `Shed`, `Batch`, `CaseMedia`, and the audit log |

Built as designed, the farmer app would write every fact twice and every
figure would be computed twice in two languages. Doc 15 phase 1 exists because
this codebase already had three definitions of batch profit; a second store
would reintroduce that across a network boundary, where it is far harder to
find and fix. One number, one computation, is the rule the boundary protects.

Three facts decided the shape:

- **The live stack is already the engine's stack.** `wrangler.toml` deploys
  farmer-api as a Worker against one shared D1 named `kukkuone` and one R2
  bucket, with Prisma resolving a per-request client inside the Worker. There
  is no platform to adopt.
- **One org-scoped database, not one per farm.** Farm onboarding must not be a
  deploy, and the queries that make this a platform — the vet's queue across
  farms, the area-outbreak signal of 16 §3.7, consensus, dataset builds, the
  admin console — all span farms. Manual entry produces kilobytes a day, so
  size is not the constraint.
- **Isolation is a scope rule, not a second database.** `vet.service.ts:213`
  already admits exactly the farm that owns a case and the vet it was given
  to, and no one else.

## 3. The boundary

### 3.1 The engine gets no database binding

Not a read-only one — none. Its `wrangler.toml` declares no `d1_databases`
and no `r2_buckets`; it holds one service binding to farmer-api, the way the
gateway reaches its services in `proxy/upstream.ts`. "It never writes farm
data" is then structurally true rather than a rule someone has to remember,
and it survives the next person to join.

### 3.2 Who owns what

| | Owner |
|---|---|
| Farm facts: batches, logs, weights, feed, medicine, cases, photos | farmer-api, in the one database. The only writer. |
| Derived figures: FCR, mortality, cost by head, feed cover | farmer-api computes them once; the engine **consumes** them |
| New analysis: baselines, expected water, health score, lift prediction, risk patterns, summaries | the engine |

The middle row is the point. The engine never recomputes a figure the
application already defines.

### 3.3 What crosses, going in

One internal read endpoint on farmer-api, reached only over the service
binding, returning a batch's facts as the Health and Growth segments already
know them: age in days, placed and alive, mortality by day and cumulative,
feed by day and type, water by day, weight samples with uniformity, shed
temperature and humidity, the outside weather of 16 §3.6, vaccinations given,
medicine courses with their withdrawal dates, the breed's standard curve, and
the farm's closed-batch history for its own baseline. For a case: the
complaint, the birds affected, and the photos by media id.

Identifiers only — org, farm, shed, batch, case, media, vet. No names, no
phone numbers, no money. The engine cannot ask for them, because the endpoint
does not serve them.

### 3.4 What comes back, and where it lands

Every analyzer returns its result with `confidence` and `missing_inputs`, and
runs on whatever inputs exist. farmer-api persists what is worth keeping as
`Insight` rows beside `Alert` — the application has to read them anyway — and
raises an alert when an analyzer says so.

So in phases 1 and 2 **the engine is stateless**: it computes and returns. The
first thing that genuinely needs a store of its own is the model registry and
the dataset manifests of phase 3.

### 3.5 Not built

| Cancelled | Because |
|---|---|
| `FarmDbResolver`, per-farm D1, the farms-to-binding registry | one org-scoped database (§2) |
| `observations`, `units`, `batches`, `attachments`, `daily_unit_stats`, `prescriptions`, `case_events` | they exist (§2) |
| `cases` and `case_reviews` as engine tables | `VetCase` exists; blind review and consensus become columns and one table in the one database |
| `reference_curves` | `BreedStandard` exists. The Vencobb tables are still owed (§6) |
| The ingest Worker, `POST /v1/observations`, `POST /v1/attachments` | farmer-api has these paths, including the raw-body photo upload and the private read-back |
| A second auth layer, service keys, per-request farm scope | the gateway resolves the principal and gates capabilities already |

### 3.6 Kept from the handover, unchanged

`packages/domain` pure and fully unit-tested; contracts as the single source
of truth at the boundary; `confidence` and `missing_inputs` on every analyzer;
rules that keep working with the AI server offline; the three-layer baseline
(breed standard, then farm history, then the live batch's rolling trend); the
expected-water formula with its temperature correction; EPEF; the weighted
health score with rescaling weights; the Gompertz fit for lift date; the
withdrawal lock; labels versioned and never overwritten; `TODO(vet): confirm`
on every threshold; ULIDs with a prefix; forward-only migrations; ISO 8601
with offset; the model registry with shadow, approval and rollback; R2 for
datasets and model files; queues for analyzer runs; and the Python service for
summaries and training.

The case state machine is kept too, as a richer `VetCaseStatus` rather than a
table of its own.

### 3.7 The sensor envelope, later

The generic observation envelope earns its place when sensors arrive, as a
**new ingest path into farmer-api** — not as a second write path beside the
daily log. `DailyLog` has fixed columns; that is the right trade while every
input is typed by a farmer.

### 3.8 What stays as it is

The farmer app and its offline write queue, the gateway and its capability
gating, the vet case flow of doc 16, the alerts and their settings, the daily
log form, auth and onboarding, the one database.

## 4. Defaults chosen, and what to say to change them

- **The engine is stateless until phase 3.** Say "give the engine its own D1"
  to change it.
- **It consumes computed figures, never raw rows.** Say "let the engine
  compute from raw" and §2's rule goes with it.
- **Blind review and consensus land in KukkuOne**, not in an engine store.
- **A hosted model writes the summaries in phases 1–2**; the GPU box arrives
  with the perception models in phase 4, behind the same `/summaries/*`
  interface. Say "buy the box now" to bring it forward. Run the 20-case Telugu
  comparison either way before committing to a model.
- **The AI never names a disease to the farmer**, and names a ranked shortlist
  to the vet alone (16 §3.5). Say "pattern only everywhere" to drop the
  shortlist, knowing it costs the agreement metric and the labels.
- **One org-scoped database.** Say "shard per farm" to revisit.

## 5. Phases

These are doc 16's phases 5 to 7, with the engine's own work named. Nothing
here starts before 16 phase 1 and doc 15 phase 1: an analyzer reasoning over a
wrong FCR is worse than no analyzer.

| # | Lands | Where the code goes | Proof |
|---|---|---|---|
| A | The facts bundle: one internal read endpoint, contracts for it | `kukkuone-farmer-api`, `packages/contracts` | A fixture batch's bundle matches what the Health and Growth segments show |
| B | The analyzers as pure functions: three-layer baselines, expected water, health score, risk patterns, EPEF | `packages/domain` | A unit test per formula and per rule, thresholds marked `TODO(vet)` |
| C | `Insight` rows and the alerts they raise; water-to-feed and heat stress reach the farmer | `kukkuone-farmer-api` | 16 phase 5's tests; the July batch's figures unchanged |
| D | The engine Worker: no database binding, one service binding, queues and cron | a new app, or farmer-api if B and C are enough on their own | The Worker runs with no `d1_databases` in its config |
| E | Summaries in Telugu and English from computed numbers, with the disease-and-drug output filter | the Python service | The filter rejects a disease name in farmer-facing text |
| F | Lift prediction, batch report, model registry, datasets, shadow mode, vet approval | `packages/domain` plus the Python service | Back-tested against closed batches; a model reaches live only by approval |

Phase D is worth a pause: if B and C deliver the analysis inside farmer-api
and nothing is straining, the separate Worker can wait. A service that exists
because a design said so is the kind of code this project does not need.

## 6. Open items

Owner is Prakash unless named otherwise.

- The Vencobb 430Y / 400Y reference tables from the chick supplier. Build the
  loader and the CSV format; leave the data files empty with a README rather
  than inventing numbers.
- Vet-confirmed alert thresholds and the vaccination template. The live
  thresholds in `alerts.service.ts:7-12` are unvalidated placeholders and
  carry the same obligation.
- **Who pays a vet who reviews a case and does not treat it.** Blind review
  across a panel is a labelling programme, and unpaid review stops after about
  two weeks, which starves phase F. The treating vet's fee is already modelled
  as `VetVisit.feePaise`; reviewers are not.
- Review deadline per severity.
- A lab partner for confirmations.
- The weather provider (16 §4 proposes Open-Meteo).
- AI server hardware, deferred to phase 4 by §4.

## 7. Cross-references

- 16 — the vet loop and the four layers this boundary serves; phases 5–7 are A–F here
- 15 §2 — the accuracy faults that must be fixed before any analyzer reads the figures
- 06 — the models §2 says already exist
- 02, 12 — the gateway, its domains and the service bindings of §3.1
- 14 LOGIC-09 — a spending cap on any hosted model before §5 phase E ships
