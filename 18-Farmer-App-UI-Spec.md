# 18 · Farmer App UI Specification

Status: written 4 November 2026, from the source of `kukkuone-farmer-mobile` at
commit `c0f1ac2` plus one uncommitted change that trims the batch hero and
three explanatory notes. Every token, component and screen below was read out
of the code, not designed on paper — where the code and this document disagree,
the code is right and this document is stale.

Audience: a reviewer, human or agent, evaluating the interface. §10 says what
to evaluate and §9 lists the deviations already known, so a review that only
rediscovers §9 has found nothing new.

Companion documents: `.claude/skills/mobile-ui-quality/SKILL.md` is the
enforceable rulebook this app is held to; `15-Batch-Centric-Redesign.md` is
why the batch hub is shaped the way it is.

---

## 1. What the product is, in one screen

A farmer runs sheds of broiler chickens in batches of three to five thousand
birds, for 40–50 days each. The app is how a batch is recorded and judged: how
many are alive, what they weigh, what they have eaten, what it has cost, and
whether anything is going wrong. Half of its users read Telugu, most of them
read it outdoors in direct sunlight, and many are tapping with dusty or gloved
hands at arm's length while standing in a shed.

Those three facts — **Telugu, sunlight, gloves** — are the reason behind most
of the rules below, and are the right lens for any review.

---

## 2. Foundations

### 2.1 Two palettes, two moods

The app runs **two complete design languages**, and a screen belongs to one of
them, never to both.

| | **night** | **canvas** |
|---|---|---|
| Namespace | `N.*` colour, `NT.*` type | `C.*` colour, `T.*` type |
| Scaffold | `NightScreen` | `CanvasScreen`, `SheetScreen` |
| Feel | dark ground, big figures, a bloom behind the header | iOS inset-grouped lists, quieter |
| Used for | list and summary screens: Overview, Batches, Money, Farms, Store | detail and form screens: a batch, a shed, a form |
| Source | `lib/core/design/night.dart` | `lib/core/design/tokens.dart` |

Both resolve through one active theme, so **light mode is a real mode**. Every
colour is a *getter*, never a `const`, because a baked constant keeps whichever
palette was in force when the widget was first built.

> **Rule.** A literal `Color(0xFF…)` in a feature file is a defect. It renders
> correctly in dark and becomes unreadable in light.

### 2.2 Colour tokens

Semantic names only. Every one carries a single meaning, and **never carries it
alone** — a status colour always ships with a word or a glyph, because roughly
8% of men have a colour vision deficiency and a red dot on its own says nothing
to them.

**night — dark (default)**

| Token | Hex | Meaning |
|---|---|---|
| `N.bg` | `#000000` | the ground |
| `N.surface` | `#121212` | a card |
| `N.surfaceHi` | `#1E1E1E` | a pressed or raised card |
| `N.line` | `#2F2F33` | hairline |
| `N.ink` | `#F7F7F8` | primary text |
| `N.ink2` | `#B5B5BC` | secondary text |
| `N.ink3` | `#89898F` | tertiary text, figure captions |
| `N.accent` | `#A78BFA` | brand, links, the one primary action |
| `N.accentPressed` | `#7C3AED` | its pressed state |
| `N.onAccent` | `#1E1E1E` | text on an accent fill |
| `N.good` | `#279B63` | healthy, done, paid |
| `N.warn` | `#D99516` | due, attention, break-even |
| `N.bad` | `#D94350` | dead, overdue, loss |
| `N.data2` | `#287FC1` | a second data series |

**night — light**

| Token | Hex |
|---|---|
| `N.bg` | `#F8F7FA` |
| `N.surface` | `#FFFFFF` |
| `N.surfaceHi` | `#FFFFFF` |
| `N.line` | `#E9E4F0` |
| `N.ink` / `ink2` / `ink3` | `#241E30` / `#6F687B` / `#938CA1` |
| `N.accent` / `accentPressed` / `onAccent` | `#7C3AED` / `#6D28D9` / `#FFFFFF` |
| `N.good` / `warn` / `bad` / `data2` | `#279B63` / `#9A6A00` / `#D94350` / `#287FC1` |

`warn` darkens from `#D99516` to `#9A6A00` in light mode — amber on white fails
contrast at the lighter value. `good` and `bad` are deliberately identical
across themes: they are the two colours a farmer learns, and moving them would
teach the app twice.

The canvas layer (`C.*`) is a semantic alias over the same palette —
`C.tint → N.accent`, `C.ok → N.good`, `C.due → N.warn`, `C.bad → N.bad`,
`C.label/label2/label3 → N.ink/ink2/ink3`, `C.card → N.surface`,
`C.separator → N.line`. There is **one** source of colour.

### 2.3 Type

Two scales, one per palette. Nothing is below 13pt on the canvas scale and
nothing below 12pt on the night scale — this is read in sunlight.

**night (`NT.*`)** — the display scale, deliberately heavier than its paper
equivalent because light text on a dark ground optically thins.

| Token | Size / weight | Tracking | For |
|---|---|---|---|
| `NT.mega` | 52 / w800 | −2.2 | the one number a screen is about |
| `NT.big` | 29 / w700 | −1.0 | a card's headline figure |
| `NT.title` | 21 / w700 | −0.5 | a card or screen title |
| `NT.name` | 16 / w600 | −0.2 | a row's name |
| `NT.body` | 14.5 / w400 | — | running text |
| `NT.quiet` | 13.5 / w400 | — | secondary text (`N.ink2`) |
| `NT.label` | 12 / w700 | **+1.4** | ALL-CAPS figure captions (`N.ink3`) |

`NT.mega`, `NT.big` and every `T.number*` style carry
`FontFeature.tabularFigures()`, so digits do not jitter in a column.

`NT.label` is 12, not 10.5. It captions every figure in the app — ALIVE,
WEIGHT, FCR, DIED — and at 10.5 grey on black it disappeared in daylight.

**canvas (`T.*`)** — Apple's SF scale, names matching Apple's so the intent is
unambiguous: `largeTitle` 34/w700 · `title1` 28 · `title2` 22 · `title3` 20 ·
`headline` 17/w600 · `body` 17/w400 · `callout` 16 · `subhead` 15 ·
`footnote` 13 · `caption` 13/w500. Figures: `numberXL` 34/w700,
`numberL` 24/w700, `number` 17/w600, all tabular.

### 2.4 Space, shape, targets

An **8-pt grid**, with 4 where 8 is too loose.

| | Values |
|---|---|
| Spacing (`DS.s*`) | 2, 4, 6, 8, 12, 16, 20, 24, 32, 40 |
| Night rhythm | `N.gutter` 16, `N.gap` 12, `N.block` 26 |
| Side gutter | **16** on every screen, both palettes |
| Radii (canvas) | `rCard` 10, `rControl` 8, `rSheet` 12, `rPill` 100 |
| Radii (night) | `N.r` 18, `N.rSmall` 10 |
| Hit target | `DS.tapMin` **44** minimum; `DS.rowHeight` **52** for list rows |

Rows are 52, not 44: Apple's minimum assumes a bare fingertip, and this is
tapped with gloves on.

> **The gutter contract.** `CanvasScreen` pads its children; `NightScreen` and
> `SheetScreen` do not. A shared component therefore asks
> `Guttered.inset(context)` before adding its own margin, or it lands 32pt in
> while the card below it sits at 16. The batch screen shipped exactly that
> bug — hero and segment control at 32, cards at 16 —
> and `test/alignment_test.dart` now measures the edges of nine screens.

### 2.5 Motion

Tokens only: `DS.fast` 180ms, `DS.normal` 260ms, `DS.slow` 420ms, easing
`Curves.easeOutCubic` in and `easeInCubic` out. A screen may not invent its own
237ms.

The contract that matters:

- **Reduced motion means no motion.** `Entrance` puts no animation widget in
  the tree at all and `context.motion(d)` returns `Duration.zero`. A shortened
  animation is still an animation; for a vestibular disorder "faster" is not
  what was asked for.
- Therefore **everything animated is a reveal of something already legible
  without it.** Motion is never the only way a value appears.
- Opacity and transform only. Animating width, height or padding makes every
  frame a layout pass.
- One or two moving things per screen. A stagger caps at six
  (`context.stagger`); past that it stops being a flourish and becomes a
  loading screen.
- iOS does not ripple, it dims and shrinks slightly: `Pressable` is the one
  press behaviour, `NoSplash` is set globally.

---

## 3. Components

All in `lib/core/design/`. A screen composes these; it does not draw its own.

### 3.1 Scaffolds (page templates)

| Component | Anatomy | Pads children? |
|---|---|---|
| `NightScreen` | dark ground, ambient bloom behind the header, eyebrow + title, optional leading/trailing, pull-to-refresh | **no** |
| `CanvasScreen` | canvas ground, back line, title, circle buttons at top right | **yes** |
| `SheetScreen` | a form as a page of its own; **draws its own Cancel**; commit action in the bar | **no** |
| `NavScreen` | large title collapsing to a centred title on scroll | — |

`SheetScreen` rules: do not add a second Cancel; a picker inside it closes with
`closeForm`, **never** a bare `Navigator.pop` — popping the sheet's own
navigator leaves an invisible sheet swallowing every tap on the screen beneath.

### 3.2 Surfaces

| Component | What it is |
|---|---|
| `Raised` | a card on the canvas with light under it — a black shadow below, a lit top edge on the surface itself |
| `Tile` | the night equivalent |
| `Field` | the gradient field, one per screen, carrying the subject |
| `CanvasSection` | a headed group of rows as a raised card |
| `Section` | a titled group of rows, hairlines inset from the left |
| `Heading` | a quiet canvas heading (not a grouped-list header) |
| `SectionHeader` | a standalone group heading |

### 3.3 Rows and controls

| Component | Notes |
|---|---|
| `Cell` | leading glyph, title, optional subtitle, value or chevron on the right. **The title takes the space and the trailing takes its own width**, capped so a long Telugu label ellipsizes — equal flex on both split the row in half and floated every count mid-row |
| `Pressable` | the one press behaviour; bare text inside it measures only the text, so wrap it to reach 44 |
| `PrimaryButton` | the single filled action on a screen |
| `SecondaryButton` | tinted text on a soft fill |
| `CardAction` | a thumb-sized button sitting on a card rather than in a bar |
| `CircleButton` / `BackCircle` / `BackLine` | canvas header controls |
| `StatusPill` | a state in **a word and a colour**, never a colour alone |
| `QuickBar` / `QuickAction` | a row of shortcuts: glyph, word, and a count when the number is the point |
| `Segments` | a segmented control, two or three mutually exclusive choices |
| `Chips` | wrapping selectable chips for a short flat option set |

### 3.4 Fields

`HeroNumberField` (the one number a screen is about — thumb-sized, centred,
autofocused), `NumberCell`, `TextCell`, `PickerCell`, `SwitchCell`.

### 3.5 Figures

| Component | Contract |
|---|---|
| `Figures([...])` | a row of figures. **Three at most — it asserts.** Never a bare `Row` of `Fig`: a bare row does not flex and overflows at 320pt |
| `Fig(value, label)` | one figure and its ALL-CAPS name |
| `Stat` | one number and what it is — the unit of every dashboard here |
| `BigFigure` | an unboxed number that is the point of its card |
| `TrendStat` / `FigureWithTrend` | a figure with its recent shape beside it |
| `ComputedValue` | a label and a figure on one line, outside a `Section` |
| `ProgressRing` | how far through its cycle a batch is, drawn rather than written |
| `AttentionRow` | one thing needing attention, on one line |

### 3.6 Charts

Charts live in **two** places, which is itself worth a reviewer's attention:
`core/design/plots.dart` holds `Plot`, `ComparePlot`, `RunTimeline`,
`RankBars`, `StackBar`, `DayStackBars`, `AgeingBars`, `Donut` and `Measures`;
`lib/widgets/charts.dart` holds `TrendLine` and a second `Donut` (the night one
is the one in use, and `batch_detail_screen.dart` imports the other with
`hide Donut` to say so).

Chart rules, each earned from a shipped defect:

- **One series, one chart.** The Feed segment drew kg/day and bags/day from the
  same logs; keep one and put the other unit in the readout.
- **Never put quantities of different magnitude on one scale when the small one
  is the point.** Costs-by-category stacked beside income turned every cost bar
  into a sliver. Give each its own chart on a shared day axis.
- **A scrolling chart never clips its first or last axis label.**
- A reference line (e.g. the breed standard) is drawn **dashed**, not as a
  second series — it is not a measurement of this batch and must not read as
  one.
- `Donut` sizes its ring to `min(size, width × 0.42)` so the legend always wins
  the space. The ring totals the birds placed; the segments are where each one
  is now — alive, sold, lost. Nothing in the hole.

### 3.7 States

`Skeleton` (shimmerless — a spinner says nothing, a shape says what is coming),
`Empty`, `Failed` (what went wrong and one way out), `InlineError` (said once,
where the thing went wrong).

- Design **loading, empty, sparse, dense and error** for every screen, and say
  which of "nothing yet" and "nothing matches" an empty state means.
- **No dead ends.** An empty state that cannot be acted on from where it stands
  is a cul-de-sac.
- **Never a button that does nothing.** A lifecycle stage that declared itself
  drivable with no branch to run rendered a primary button that silently fell
  through. This shipped.

---

## 4. Content rules

These are as much a part of the interface as the pixels.

**Localisation.** Every user-facing string lives in `lib/l10n/app_en.arb` and
`app_te.arb`, at exact parity, enforced by `telugu_test`. Telugu runs
materially longer than English and is the single largest source of shipped
overflow. Therefore: every `Text` inside a `Row` is in an `Expanded` or
`Flexible` with `maxLines` and `TextOverflow.ellipsis`; commit labels and nav
actions are **one or two words** ("Send", not "Send to partner" — who it goes
to is the subtitle's job); and an `options[]` **key** is never translated, only
its label, because a translated key falls through its own switch silently.

**Money.** Paise end to end, divided only where printed: `money(paise / 100)`.
**Indian digit grouping** — `₹1,71,500`, never `₹171,500`. Never
`toStringAsFixed` into a currency string.

**Figures.** Null is `—`, never `0`: a figure with no data and a figure that is
genuinely zero are different facts, and showing a bill as `₹0` before it exists
is the worse of the two errors. A figure and its unit never break apart — join
with ` `.

**One number, one computation.** A farm's feed rate and days left once came out
as 6 days on the farm screen and 23 on the Store tab from the same stock. Every
screen that states a derived figure calls the same function. A corollary this
app treats as a layout rule: *what is already printed elsewhere on the screen
is not a second finding.*

---

## 5. Screen inventory

62 screen classes. Scaffold in brackets: **N**ight, **C**anvas, **S**heet,
**V** = NavScreen.

| Area | Screens |
|---|---|
| auth | SignIn *(raw `Scaffold` — see §9)* |
| onboarding | AccountSetup **V**, FarmSetup **V** |
| overview | Overview **N**, Attention **N** |
| dashboard | Dashboard **N** |
| batches | Batches **N**, **BatchDetail C**, PlaceBatch **S**, PlaceChicks **S**, CloseBatch **S**, CorrectPlacementDate **S** |
| farms | Farms **N**, FarmDetail **C**, ShedDetail **C**, FarmForm **S**, ShedForm **S** |
| logs | DailyLog **S**, WeightSample **S**, LogDetail **C** |
| health | MedicineStock **C**, AddMedication **S**, MedicinePurchase **S** |
| vet | VetCases **N**, VetCase **N**, AskVet **S** |
| harvest | Offer **S** |
| money | Money **N**, Accounts **N**, AccountStatement **N**, Costs **N**, Reports **N**, Expenses **C**, FeedStock **C**, Allocation **C**, SellBirds **S**, RecordPayment **S**, SupplierPayment **S**, FeedPurchase **S**, AddExpense **S**, AccountEntry **S**, AccountForm **S**, AccountTransfer **S**, PartyForm **S**, Wages **S** |
| inventory | Inventory **N**, StoreLedger **N**, SellFeed **S** |
| insight | Alerts **C**, AlertSettings **C**, History **C**, Standards **C**, StandardForm **S** |
| partner | PartnerOrders **N**, OrderDetail **N**, OrderChicks **S**, OrderFeed **S** |
| more | Account **N**, Profile **N**, Settings **N**, People **V**, InviteMember **S** |

Five tabs: Overview · Farms · Batches · Store · Money.

---

## 6. The batch hub — the screen the product turns on

`BatchDetailScreen` (canvas). A batch is a shed full of birds from placement to
close, and **everything a batch owns lives here**, in one header, one hero and
five segments.

```
┌─────────────────────────────────────────┐
│ ‹ Batches                               │  back line
│ CHELGAL FARM · COBB 430          (+) (⋯)│  eyebrow + circle actions
│ Shed 1                                  │  title
├─────────────────────────────────────────┤
│  ╭───╮   ● Alive            3,353       │  HERO (Tile)
│  │ ◯ │   ● Lost               147       │  Donut, ring = birds placed,
│  ╰───╯                                  │  segments = alive / sold / lost
│  ─────────────────────────────────────  │
│  Day 57      2.00 kg      1.36          │  Figures(3)  — the assert
│  AGE         WEIGHT       FCR           │  NT.label captions
│  4.2% mortality · sells around day 42 · │  a fourth..sixth value as a
│  9,125 kg fed                           │  sentence, never a 4th Fig
├─────────────────────────────────────────┤
│ [ Today | Feed | Money | Health |Growth]│  Segments
└─────────────────────────────────────────┘
```

| Segment | Carries |
|---|---|
| **Today** | stage timeline (`Stage 2 of 5`, name, primary action, "All five stages"), quick actions (Log today / Mortality / Feed used), the batch arc, the facts table, every logged day newest first with gaps shown as gaps |
| **Feed** | inventory, issued to this batch, "will it last" (kg/day, g/bird/day, days of cover), feed added, bags-a-day chart |
| **Money** | committed or profit headline with a `running`/`closed` pill, spent / earned / break-even, cost breakdown by category, costs by day, lots sold, costs, pending to collect and to pay |
| **Health** | withdrawal banner (the one thing that can stop a sale), vaccination checklist against days, medication courses, medicine inventory, add medication, ask a vet |
| **Growth** | weight against the breed standard (dashed reference), deaths a day, weigh a sample |

Design decisions worth a reviewer's attention:

- The hero repeats on all five segments. That is deliberate — the question
  "how many are alive" outranks whichever tab you are on — but it is also the
  largest single block of vertical space in the app.
- The quick actions appear **only while birds are in the shed**. A closed batch
  once still offered Log today and Mortality, which write a day against a run
  that ended.
- A log row's date turns red on a heavy-mortality day; the count is **not**
  printed twice in two colours.
- Daily-log subtitles show whole kilos. A run of days reading
  "426.6 · 416.2 · 405.8" against a 50 kg bag is noise beside the day and the
  deaths; the entered figure is a tap away in the day's own record.

---

## 7. Accessibility and field conditions

| Constraint | What it forces |
|---|---|
| Sunlight | nothing below 12pt; `NT.label` raised from 10.5 to 12; amber darkened in light mode |
| Gloves | 52pt rows, 44pt absolute minimum |
| Colour vision | a status colour never travels alone — always a word or a glyph |
| Vestibular | reduced motion means **no** motion, and nothing appears only by animating |
| Telugu | every row flexes; labels are one or two words; ARB parity is a test |
| 320pt phones | `screens_test` mounts each screen at 320×568 in **both** locales, where an overflow is a framework exception rather than a farmer's complaint |

---

## 8. What enforces this

| Test | Enforces |
|---|---|
| `screens_test` | every screen mounts at 320×568 in English and Telugu |
| `alignment_test` | nine screens' cards share the 16pt gutter |
| `telugu_test` | en/te ARB parity |
| `no_hardcoded_strings_test` | user-facing English in the source |
| `motion_test` | reduced motion is honoured |
| `nav_bar_test`, `cell_test`, `batch_ring_test`, `money_figures_test`, … | component contracts |

911 tests pass at the commit this document describes.

---

## 9. Known deviations

Found by audit on 4 November 2026. Listed so a review can look past them.

**9.1 — Seven figure labels are English on a Telugu phone.** `Fig`'s label is
its *second* argument and `no_hardcoded_strings_test` only inspects `Heading(`
and `Text(`, only matches a **capitalised** first letter, and only single
quotes. Figure captions are rendered upper-case by `NT.label`, so they are
naturally written lower-case in source — precisely the shape the lint cannot
see.

| File | Label |
|---|---|
| `money_screen.dart:477,479,482` | `'owed'`, `'past terms'`, `'buyers'` |
| `money_screen.dart:542,543` | `'owing'`, `'suppliers'` |
| `reports_screen.dart:392,393` | `'sold'`, `'batches'` |

Fix: move the seven into both ARBs, then widen the lint to `Fig`, `Cell`,
`Empty`, `StatusPill`, `PrimaryButton` and to lower-case and double-quoted
literals. The lint gap is the real defect; the seven strings are its symptom.

**9.2 — Loose English in interpolations.** `'${…} days'`
(`close_batch_screen.dart:138`, `sell_birds_screen.dart:391`) and
`'of $target'` (`overview_charts.dart:386`) are untranslated words inside
otherwise localised strings.

**9.3 — Eleven screens never mounted by `screens_test`.** They have never been
checked for Telugu overflow at 320pt: `Accounts`, `AccountEntry`,
`AccountForm`, `AccountStatement`, `AccountTransfer`, `Attention`, `Offer`,
`Reports`, `SellFeed`, `StandardForm`, `StoreLedger`. Note that `Reports` is
also in 9.1.

**9.4 — Currency built by hand, without Indian grouping.**
`medicine_purchase_screen.dart:256`, `place_batch_screen.dart:287`,
`place_chicks_screen.dart:257`, `sell_birds_screen.dart:454` and
`feed_purchase_screen.dart:247` interpolate `toStringAsFixed(2)` after a `₹`.
A five-figure rate prints `₹100000.00` where the rest of the app prints
`₹1,00,000`. `feed_purchase_screen.dart:247` additionally carries a hardcoded
English sentence.

**9.5 — Chart colours baked into feature files.**
`overview/dashboard_charts.dart:56-61` holds six literal `Color(0xFF…)` values
and `widgets/charts.dart:33-36` four more. Both sit outside `core/design/`, so
light mode cannot re-resolve them.

**9.6 — Bare `Row` of `Fig`.** `money_screen.dart:475` and `:540` build figure
rows by hand instead of `Figures([...])`, bypassing the three-item assert; the
second carries only two figures.

**9.7 — `SignInScreen` uses a raw Material `Scaffold`.** The first screen every
user meets is the one screen that does not use a house scaffold.

**9.8 — No breed standard for Cobb 430.** The Growth segment's dashed reference
line is the point of the chart and does not draw, because no `BreedStandard`
row matches the seeded breed. This is a reference-data gap, not a layout one,
and per the project's own rule a threshold no vet has confirmed must be marked
`TODO(vet): confirm` rather than invented.

**9.9 — Density, not correctness.** The hero occupies roughly 240pt on all five
segments, and the Today segment's stage card spends about 290pt on a progress
bar, "Stage 2 of 5", a stage name, a caption that repeats the three buttons
directly beneath it, a full-width button and a disclosure link. "Mark ready for
harvest" is also a four-word commit label against a one-or-two-word rule.

---

## 10. What to evaluate

A review is most useful on the things a test cannot assert:

1. **Does the hero earn its space on all five segments**, or should the lower
   four collapse it to a single line?
2. **Is the stage card worth 290pt**, and does its caption survive sitting
   directly above the three buttons that say the same thing?
3. **Is five segments the right cut of a batch**, or is Growth really part of
   Today?
4. **Does the night/canvas split read as one product**, or as two apps — the
   batch hub is canvas and the list it was opened from is night.
5. **Does every status colour genuinely travel with a word**, screen by screen.
6. **Telugu at 320pt**: mount the eleven screens of 9.3 and look.
7. **Light mode**, which is far less exercised than dark — the deviations in
   9.5 are visible there and nowhere else.
8. **Does any screen show a figure a second time** in a different shape? That
   is this codebase's most repeated defect and the fastest thing to find.

---

## 11. Cross-references

- `.claude/skills/mobile-ui-quality/SKILL.md` — the enforceable rulebook
- `.claude/skills/web-ui-quality/references/rules.md` — the web half, so the
  portal and the app stay recognisably one product
- `15-Batch-Centric-Redesign.md` — why the batch is the hub
- `16-Health-Intelligence.md` §3.3 — the vet app's screens, not yet built
- `09-Client-Apps-Spec.md` — predates this and describes an Expo structure that
  does not exist; trust this document and the code over it
