# 15 · Batch-Centric Redesign of the Farmer App

Status: proposed, 14 October 2026. Supersedes the tab layout in 09 §3 and the
batch screen in 07 §3.4 where they differ.

## 1. The ask

The farmer app should work batch by batch. Feed and money belong to the batch
that used or earned them. Everything else exists to serve a batch.

## 2. What the audit found

Three read-only passes over the app, the API and the data model. Pointers are
to `kukkuone-farmer-mobile` unless named.

**The data already thinks in batches.** Sales, feed issues, medicine usage,
vaccinations, closure, rent, batch costs and allocated farm costs all carry a
batch id. Buyer receipts reach a batch through the lot they settle. Only four
things are farm or organisation level by design, and should stay so: stock on
hand and its purchase bills, monthly farm costs before allocation, the cash
accounts, and parties.

**The app does not.** The owner has five tabs: Overview, Farms, Batches, Store,
Money. One batch's money is drawn on four screens (`settlement_screen.dart`,
`money/batch_ledger_screen.dart`, the detail `_Ledger`, and
`batch_money_screens.dart`). Feed for one batch appears in five places. Feed
purchase is reachable only through Farms → Farm → Feed stock, never from the
batch that will eat it. Payments for a batch's lots are reachable only from the
Money tab's dues list. The batch screen is one 900-line scroll with link cards
at the bottom.

**Five accuracy faults sit under the figures.** Any batch-centric number is only
worth showing once these are fixed.

| Fault | Where |
|---|---|
| Period cost by head counts feed and medicine twice: the purchase bill and the issue | `pnl.service.ts:145-154` |
| Rent accrues only when someone opens the settlement sheet | `settlement.service.ts:94` |
| Batch profit has three different sums: closure summary, finance summary, and `batchCost` | `finance.service.ts:73-116`, `batches.service.ts:600-617` |
| Reports pick closed batches by placement date; P&L picks by close date | `reports.service.ts:31-33`, `pnl.service.ts:60` |
| Farm cost splits never recalculate after a placement or a close | `money.controller.ts:275` |

**Lifecycle.** Nine stages are named, the app can drive five. Sales from the app
never advance the status, so Sold, Settlement pending and Settled are
unreachable, and the Settled stage draws a Close button that does nothing
(`stage_timeline.dart`, default case). Phase-one close bypasses settlement.

**Dead weight.** `batch_card.dart` and `batch_quick_menu.dart` have no callers.
The old `DashboardScreen` survives only behind a worker's account sheet. Stale
comments in `app.dart` describe tab sets that no longer exist.

## 3. The design

### 3.1 The five tabs stay

Overview, Farms, Batches, Store and Money remain the bar. Batch-centric is a
statement about where data and actions live, not about how many tabs there
are. A farmer still reads the farm as a whole on Overview, owns farms and
sheds on Farms, and sees the godown on Store. Farm-wise views stay for the
same reason. Revised 15 October 2026: a first cut folded Overview and Store
into the other tabs and was reverted on review.

Money keeps what is not any one batch's: accounts, who owes whom, and the
period reports. A batch's own money lives on the batch.

### 3.2 The batch screen is the hub

One screen, five segments, in the order a day runs:

| Segment | Shows | Enters |
|---|---|---|
| **Today** | Stage, day, alive, lost, logged today or not, what needs doing | Log, deaths, weight sample, mark ready, record lift, close |
| **Feed** | Fed to date, feed a day, days of cover for this batch from the farm store, issues by type | Feed purchase for this farm, order feed from the partner |
| **Money** | Spent, earned, per bird, cost by head, lots, what is still to collect | Sell birds, record a payment against this batch's lots, add a cost |
| **Health** | Vaccination checklist, courses, withdrawal, vet cases | Complete a vaccination, add medication, buy medicine, ask a vet |
| **Growth** | Weight against the standard, mortality, FCR | Weight sample |

The four batch money screens collapse into the Money segment. The five feed
views collapse into the Feed segment and the farm's store. Settlement remains
as the export of the Money segment, not a separate place.

### 3.3 Lifecycle

Stages become the ones the app can drive: planned, active, ready, harvesting,
closed. Selling birds from the app marks the batch sold once the shed is empty,
and close follows. No stage ever draws a button it cannot act on.

### 3.4 What stays as it is

The daily log form, the offline write queue, the place-batch flows, the partner
order flows, alerts and their settings, people and scope, sign-in, the design
system and the rulebook.

## 4. Defaults chosen, and what to say to change them

- **Payments live on the batch's Money segment as well as per party in Money.**
  Say "payments only in Money" to keep the batch read-only on cash.
- **Nine stages become five.** Say "keep settlement as a stage" and the Sold and
  Settled stages stay, with the app made able to drive them.
- **Finished batches stay on the board, below the running ones.** Say "finished
  under Money" to move them.

## 5. Phases

Each phase ends green on the full Flutter suite, the gateway and API tests, and
a simulator check against the dev account's July batch. That batch is the
fixture: spent ₹8,15,485, earned ₹8,01,047, 12,034 kg fed, FCR 1.47. A phase
that changes those numbers without a fault to point at is wrong.

| # | Phase | Lands | Proof |
|---|---|---|---|
| 0 | Baseline | Screenshots of every tab and the batch screen on the simulator; test count recorded | Files under `.audit/baseline/` |
| 1 | Figures you can trust | The five faults in §2 fixed in the API, one profit definition, rent accrued on write paths, allocations recalculated on place and close | API tests for each; July batch figures unchanged except where a fault explains the change |
| 2 | The batch hub | Segments on the batch screen; payments, purchases and orders enterable from it; the duplicate money and feed screens deleted with their callers migrated | `screens_test` map updated; simulator walk of all five segments |
| 3 | Money tab | Per-batch sections leave the Money tab; break-even moves onto the batch's Money segment | `money_batch_first_test`; the batch segment on the simulator |
| 4 | Lifecycle | Five stages, sale advances status, dead button gone | `lifecycle_test`, `stage_timeline_test` |
| 5 | Sweep | Dead files, stale comments, the worker dashboard, the localisation leaks the audit listed | `no_hardcoded_strings_test`, analyzer count not above today's |

Phase 1 is independent of every product default above and goes first. Phase 2
is where the defaults bite, and is the checkpoint.
