# 20 · The AI Engine on the Mac

Status: built 13 November 2026 on `feat/ai-assistant` in kukkuone-database,
packages, kukkuone-farmer-api, kukkuone-api-gateway, kukkuone-farmer-mobile,
kukkuone-platform-web and kukkuone-vet-mobile. Not deployed, no secrets set,
no remote migration applied. Read with 16 (the vet loop and its four layers)
and 17 (the boundary this keeps).

## 1. The shape

```
farmer app / web ──► gateway ──► farmer-api ──(HTTPS, X-Engine-Key)──► Cloudflare Tunnel ──► engine on the owner's Mac
                                     │                                                     (kukkuone-ai-engine)
                                     ├── D1: AssistantConversation, AssistantMessage, CaseDiagnosis (ai rows)
                                     └── R2: assistant/{orgId}/{mediaId}, vet/{orgId}/{caseId}/…

engine ──(x-engine-pull-key)──► gateway /api/v1/engine/* ──(service binding)──► farmer-api internal/engine/training/*
```

- **The engine is read-side and stateless for farm facts** (17 §1). It gets
  photos and the batch facts bundle in a request, answers, and keeps nothing
  about the farm. The platform is the system of record for the conversation:
  every turn is stored in `AssistantMessage` before or after the engine is
  asked, never by the engine.
- **The Mac is not always on.** It sleeps, reboots and travels. Everything on
  the platform side treats it as optional (§3.3).
- **The tunnel** gives the Mac a public HTTPS hostname without opening a port.
  farmer-api reaches it by `fetch`, with a shared secret in `X-Engine-Key`.

## 2. Configuration

| Where | Name | Kind | What |
|---|---|---|---|
| farmer-api | `ENGINE_URL` | var (or secret) | The tunnel's base URL, no trailing slash |
| farmer-api | `ENGINE_KEY` | Wrangler secret | Sent as `X-Engine-Key` on every call |
| gateway | `ENGINE_PULL_KEY` | Wrangler secret | What the engine presents to pull training data |

Two keys, not one: the key farmer-api uses to call the engine and the key the
engine uses to call the platform are different secrets, so a leak of either
opens only one direction. With `ENGINE_URL` or `ENGINE_KEY` unset the
assistant answers "unavailable — connect to a vet"; with `ENGINE_PULL_KEY`
unset the training feed is closed to everybody.

None of these is set yet. Locally, put them in farmer-api's and the
gateway's `.env`.

## 3. The contract

Every engine route takes `X-Engine-Key`. Types are in
`packages/types/src/engine.ts`.

| Route | In | Out |
|---|---|---|
| `GET /v1/health` | — | `{ok, vision:{modelVersion,…}\|null, chat:{model, ready}}` |
| `POST /v1/triage` | `{locale, images:[base64 jpeg], facts?}` | `{farmer:{severity, patterns[], firstAid[], possibleConditions[], connectVet:true, text}, vet:{ranked:[{diseaseCode, class, probability}], modelVersion, quality}}` |
| `POST /v1/chat` | `{locale, messages:[{role, content}], facts?, triage?}` | `{reply, severity, possibleConditions[], connectVet:true, safety:{blocked, reasons[]}, model}` |

`facts` is farmer-api's facts bundle (17 §3.3, `FactsService.batchFacts`):
ids and computed figures, no names, no phones, no money. `triage` in a chat
call is only the `farmer` half of a triage result.

### 3.1 The farmer's assistant (farmer-api)

| Route | Does |
|---|---|
| `POST /farmer/assistant/media` | A photo as the raw body (the vet case upload's shape). Stored privately at `assistant/{orgId}/{id}`; returns `{id}` |
| `GET /farmer/assistant/media/:id` | The photo back, for the org that sent it only |
| `POST /farmer/assistant/messages` | `{conversationId?, batchId?, locale, body, mediaIds?}`. Stores the farmer's turn, builds the batch's facts, runs `/v1/triage` when there are photos and `/v1/chat`, stores the reply. Returns `{conversationId, vetCaseId, available, message, reply}` |
| `GET /farmer/assistant/conversations` | The caller's own, newest first, with the last turn |
| `GET /farmer/assistant/conversations/:id` | One, with its turns |
| `POST /farmer/assistant/conversations/:id/connect-vet` | `{vetOrgId?, trainingConsent?}`. Raises a case through `VetService.raise` — the farmer's words as the complaint, the assistant's last severity noted — copies the photos onto it through `VetService.attach`, runs layer 3, links `vetCaseId` and returns the case. A second call returns the same case |

Permissions are the health module's: `health:Create` to ask, `health:View`
to read. A conversation is scoped to its author (org and user), so a
co-worker's talk stays theirs. The gateway gives the two slow assistant
routes 28 s instead of 15 s.

### 3.2 What is stored

`AssistantConversation(id, orgId, userId, batchId?, locale, vetCaseId?,
createdAt, updatedAt)` and `AssistantMessage(id, conversationId, role
'farmer'|'assistant', body, severity?, possibleConditions JSON?, patterns
JSON?, firstAid JSON?, mediaIds JSON?, engineModel?, createdAt)`. Postgres
migration `20261115090000_ai_assistant`, D1 `0021_ai_assistant.sql`, marker
`m0021` in `prisma/d1/README.md`. The same migration adds
`CaseDiagnosis.probability` for layer 3. `patterns` and `firstAid` are
columns beyond the first sketch, because the reply shows them and a reloaded
conversation must show them again.

### 3.3 When the Mac is off

The client in `kukkuone-farmer-api/src/assistant/engine.ts` returns `null`
for every failure — unconfigured, refused, timed out (triage 10 s, chat 14 s),
a non-2xx, or a body that does not match the contract. Nothing throws, so
nothing becomes a 500:

- The farmer's turn is already stored.
- The reply is stored as a localised "The assistant is not available right
  now… connect to a vet", with `available: false` in the response, and both
  apps show their own unavailable notice beside it.
- If triage answered and chat did not, the triage text is the reply.
- Layer 3 is skipped silently; the case is a case either way.

## 4. Safety policy

Decided by the owner on 13 November 2026; 16 §4 records the change.

**The farmer sees:** severity (`low`, `watch`, `urgent`), patterns, first
aid, and the conditions the signs **may** be consistent with — labelled
"Possible — only a vet can confirm" on both apps — and every answer ends
with "Connect to a vet", which stays pinned on the screen.

**The farmer never sees:** a medicine, a dose or a withdrawal period from the
AI, nor the ranked differential or any probability.

**How that is enforced, on the platform side, independently of the engine:**

1. **No field.** The farmer-facing types (`TriageFarmer`, `ChatResult`,
   `AssistantReply`), the `AssistantMessage` table and the Flutter model
   `lib/core/models/assistant.dart` have no field that could hold a
   treatment. A Flutter test reads the model's source and fails on
   `medicine`, `dose`, `withdrawal`, `prescri` or `drug`.
2. **Allow-lists on read.** `readTriage` and `readChat` copy only the named
   fields; a `medicine` the engine grows later is dropped, not passed on.
3. **A second filter.** A reply the engine marks `safety.blocked`, or whose
   text carries a dose (mg, mcg, IU, cc, a concentration such as g/l or
   ml/kg, or "withdrawal period/days"), is replaced with "I cannot advise on
   medicines or doses. Only a vet can prescribe — connect to a vet." First-aid
   lines carrying a dose are dropped. Grams and millilitres alone are not
   treated as doses, because "1,800 g" is a weight and "250 ml" a day's water.
4. **The vet half stays with the vet.** `TriageResult.vet` is never written
   to `AssistantMessage` or returned to a farmer. It lands only as layer 3's
   `CaseDiagnosis` rows, which `farmerDiagnoses` filters out of every farm
   view.

The engine's own output filter is the first lock; these are the second, so
that a change on the Mac cannot by itself put a dose on a farmer's phone.

### 4.1 Layer 3 for the vet

When a case gets photos — each upload on the existing Ask-a-vet path, and
once after connect-vet copies a conversation's photos — farmer-api sends up
to four of the case's bird, droppings, flock and post-mortem photos to
`/v1/triage` with the batch's facts and writes up to three `CaseDiagnosis`
rows: `diagnosedBy: 'ai'`, `evidence: PRESUMPTIVE`, ranked by probability,
`probability` set, mapped to the catalogue by `Disease.code`. A class with no
catalogue code is skipped, never invented. Each run replaces the previous
run's AI rows. The vet desk now sends each diagnosis with its catalogue name.

Both vet desks show these apart from the vet's own calls, as "AI suggestion —
not a diagnosis" with the probability, and AI rows no longer take a rank
from the vet's next diagnosis (`kukkuone-vet-mobile` case and diagnose
screens; `kukkuone-platform-web/src/vet/VetCase.tsx`).

Cost of the per-upload run: each photo upload on the Ask-a-vet form waits for
triage (at most 10 s, nothing when the Mac is off). If that slows multi-photo
uploads in practice, run it once after the last photo instead.

## 5. The training feed

Machine access for the engine to build datasets. The engine is not a user:
no bearer, no org, no principal.

**Door:** `GET https://<gateway>/api/v1/engine/training/…` with header
`x-engine-pull-key: <ENGINE_PULL_KEY>`, compared in constant time. The gateway
rebuilds the upstream path from one of three patterns — never forwards what
it was sent — and calls farmer-api's `internal/engine/training/*` over the
`FARMER` service binding. Anything else is 404; a wrong or missing key is 403.

**Two locks on every case:** `VetCase.trainingConsent` is true, **and** a vet
— `diagnosedBy` set and not `'ai'` — put a catalogue disease on it. An
AI-only case is excluded: that would be the model grading itself. The label
is the vet's leading call (lowest rank, then latest).

**Nothing that identifies a farm:** ids, a disease, photos of birds. No org,
farm, shed, batch, user, name, phone, location or money field anywhere.
Shed and document photos are left out, because a shed photo can carry the
farm's sign and a document is usually a lab report with the farm's name on
it. Thread text is the farmer's and vet's own words, with the farm's name,
the organisations' names, the authors' names and anything shaped like a
phone number replaced by `[name]` and `[number]`.

| Route | Returns |
|---|---|
| `GET /api/v1/engine/training/cases?since=<ISO>` | `{data: [{caseId, diseaseCode, diseaseName, evidence, confirmedAt, photos:[{id, kind}]}]}`; `since` filters on the vet's diagnosis time. up to 500 per call, oldest confirmation first: page by passing the last `confirmedAt` (inclusive — dedupe by `caseId`) |
| `GET /api/v1/engine/training/media/:id` | The photo's bytes with its content type — only for a photo of a case in the feed; anything else is 404 |
| `GET /api/v1/engine/training/threads?since=<ISO>` | `{data: [{caseId, messages:[{role:'farmer'\|'vet', body, at}]}]}` for consented cases, the complaint first; `since` filters on the case's last change |

Consent can be withdrawn. A case that leaves the feed simply stops being
listed; an engine that has already exported it must drop it at its next
full rebuild, so exports should be rebuilt from the feed rather than only
appended to.

### 5.1 The engine's export layout

What the engine side writes from the feed (its scripts are the engine
repo's, not this one's):

```
exports/<YYYY-MM-DD>/<caseId>/
  meta.json        # the cases row as served: caseId, diseaseCode, diseaseName, evidence, confirmedAt, photos
  photos/<id>.jpg  # GET …/training/media/<id>, one per photos[] entry
  thread.json      # the threads row for this caseId, if any
```

A sketch of the pull, with `$KEY` from the engine's own secret store:

```
curl -s -H "x-engine-pull-key: $KEY" "$GATEWAY/api/v1/engine/training/cases?since=$SINCE"
curl -s -H "x-engine-pull-key: $KEY" "$GATEWAY/api/v1/engine/training/media/$ID" -o photos/$ID.jpg
curl -s -H "x-engine-pull-key: $KEY" "$GATEWAY/api/v1/engine/training/threads?since=$SINCE"
```

## 6. The apps

- **Farmer app** — `lib/features/assistant/assistant_screen.dart`, "Ask about
  your birds". Reached from the batch Health tab, the Alerts screen's header,
  and a HEALTH_RISK alert's action. Photo from camera or gallery through the
  Ask-a-vet form's resize; replies with severity, patterns, first aid and
  possible conditions; "Connect to a vet" pinned under the talk (with no talk
  yet it opens the Ask-a-vet form; once a case exists it opens the case). Not
  queued offline: an answer is the point of asking, so offline keeps the draft
  and says so, with the vet as the way forward.
- **Web portal** — `/assistant` in the farmer portal, same behaviour, the
  conversation and batch in the URL; a HEALTH_RISK alert links there.

## 7. Not done

- **Nothing deployed.** No secret set, no tunnel URL, migration 0021 applied
  to no database. Apply 0021 before deploying the farmer-api that writes it.
- **No device run, no live engine.** Unit and widget tests only; the contract
  is the one stated by the engine's builders, not one measured against it.
- **Telugu in the new strings is unreviewed**: written in the apps' existing
  vocabulary (డాక్టర్ for vet), not read by a native speaker. Same for the
  two server fallback sentences.
- **`likelihood` is shown as the engine wrote it**, in whatever language it
  wrote it.
- **The dose filter is a pattern**, not understanding. It will miss a dose
  written as words and may withhold a harmless sentence; the engine's own
  filter is the first line.
- **Agreement metric** (16 §3.5: AI rank 1 against the vet's confirmed call)
  is not computed yet; the rows it needs now exist.
- **A spending cap** (14 LOGIC-09) does not apply while the model runs on the
  owner's Mac; it does the day the engine moves to a hosted model.

## 8. Cross-references

- 16 §3.5, §4 — the four layers and the decision this changes
- 17 — the boundary: the engine stores nothing about a farm
- 14 — the security checklist; the pull key and the private media paths
