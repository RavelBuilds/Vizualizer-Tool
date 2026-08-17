# AI Handoff Document — PlantSim Visualizer
**Last updated:** June 18, 2026 (dynamic constraints, card sync, rightsizing fix, FAB tray; Little's WIP, MU renames, theme icon)
**Supersedes:** `Handoff_15-06-26.md`

---

## Project Overview

Single-file client-side web app (`gearboxsim.html`, **~4 629 lines**) that parses PlantSim HTML reports and visualizes them as an interactive D3.js topology map. No build step, no npm. The entire app — CSS, inline SVG symbols, HTML, and JS — lives in one file.

**Goal:** Replace manual spreadsheet analysis with a visual, actionable dashboard that instantly surfaces bottlenecks, starvation, WIP excess, and KPI deviations — with recommendation cards specific to Siemens Tecnomatix Plant Simulation actions.

---

## File Sections (accurate as of 18-06-26)

| Section | Approx. Lines | Notes |
|---|---|---|
| `<style>` | 8–1370 | All CSS. Uses CSS variables from `:root`. **Never use raw colors.** |
| SVG `<defs>` | ~1370–1430 | Inline `<symbol>` icons referenced via `<use href="#icon-name">`. |
| HTML panels | ~1430–1790 | `#tooltip`, `#drop-overlay`, `#app-container`, `#fab-tray`, `#help-popup` |
| `<script>` begins | ~1790 | |
| `TOPOLOGY` const | ~1955 | Read-only node/edge definitions. **Never mutate.** |
| `CONFIG` const | ~2081 | All diagnostic thresholds. Edit here only. |
| `MODEL_CONFIG` const | ~2097 | Buffer caps for R_PLANTSIM_GATING. |
| `parseHTM()` | ~2110 | HTM parsing pipeline — produces `state.stations`, `state.drains`, `state.meta`. |
| `analyze()` | ~2319 | KPI computation, statusRole assignment, constraints list, rec cards. |
| `RULES = [` | ~2484 | 9 rule objects (R_GEN_BOTTLE, BALANCE, GATING, STARVED, OVERPROD, BLOCK, BUFFER, RIGHTSIZE, NOMATCH). |
| rec filter | ~2884 | Strips recs targeting only Healthy stations — keeps cards in sync with panel. |
| constraints rendering | ~2897 | Three tiers: Bottleneck → Constrained (sub-grouped) → Healthy (collapsible). |
| bottom panel rendering | ~3010 | Hides `#bottom-panel` and `#horizontal-resizer` when `recs.length === 0`. |
| `initD3()` | ~3050 | D3 setup, zoom behavior, edge/node rendering. |
| `renderMap()` | ~3165 | Full D3 map re-render. Multi-pass layout: grid → per-line column centering → A→B gap → Line C. |
| Help Inspect Mode | ~4120 | `enterHelpMode / exitHelpMode / showHelpPopup / identifyElement`. |
| `identifyElement()` | ~4203 | ~25 priority-ordered `.closest()` checks; returns `{category, title, body}` with HTML formulas. |

---

## CONFIG Object (~line 2081)

```js
const CONFIG = {
  BOTTLENECK_THRESH:      75,   // util% → statusRole = 'Bottleneck'
  STARVED_THRESH:         20,   // waiting% → statusRole = 'Starved'  (was 15)
  BLOCKED_THRESH:         28,   // blocked% → statusRole = 'Blocked'  (was 20)
  BLOCK_CARD_BLOCKED_MIN: 28,   // R_GEN_BLOCK trigger minimum (aligned with BLOCKED_THRESH)
  BLOCK_CARD_UTIL_MAX:   100,   // util cap removed — any blocked station gets a Fix card
  OVERPROD_MULTIPLIER:    1.5,
  BALANCE_RATIO_MIN:      0.1,
  FAILURE_THRESH:           5,  // failed% for tooltip/detail flagging; no rec card generated
  BUFFER_WIP_RATIO:       0.15,
  TARGET_OUTPUT:          300,  // pcs/mo target; editable via UI stepper
  WORK_DAYS_PER_YEAR:     250,  // editable via UI stepper
};
```

---

## statusRole Classification (~line 2361)

Classification runs in this priority order for every machine station. Sources, drains, buffers, and **stations with zero entries** (Line C / Future) are always `'Healthy'` and short-circuit before any threshold check.

```
entries === 0           → 'Healthy'   (inactive station; 100% waiting is an artefact)
util > 75 && wait < 5  → 'Bottleneck'
waiting > 20           → 'Starved'
blocked > 28           → 'Blocked'
else                   → 'Healthy'
```

**Dynamic post-pass** (~line 2388): If no Bottleneck exists AND over 55% of active machines are still classified Blocked (system-wide takt rhythm, not a real constraint), the effective blocked threshold is raised to `min(p75 of blocked values, 40%)`. Affected stations are reclassified to Healthy. This prevents the Constrained tier from being flooded during a well-running simulation.

**`roleSeverity`** field is also set on each station:
- `'critical'` for Bottleneck
- `'high'` for Starved > 35% or Blocked > 40%
- `'moderate'` for Starved 20–35% or Blocked 28–40%
- `'none'` for Healthy

---

## RULES Array (~line 2484) — 9 rules in order

| ID | Badge Label | Severity | Trigger |
|---|---|---|---|
| `R_GEN_BOTTLE` | Bottleneck | critical/warning | `statusRole === 'Bottleneck'`; one card per affected line (A/B) + ME3 shared |
| `R_GEN_BALANCE` | Exit Strategy | critical | `min(entries)/max(entries) < 0.1` within `parallelGroup` |
| `R_PLANTSIM_GATING` | Deadlock Risk | warning | ME is Bottleneck AND any buffer ≥ 90% of its `MODEL_CONFIG.BUFFER_CAPS` ceiling |
| `R_GEN_STARVED` | Starved | warning | Any machine has `statusRole === 'Starved'` |
| `R_GEN_OVERPROD` | Overproduction | warning | `entries > lineDrain × 1.5 && statusRole === 'Blocked'` |
| `R_GEN_BLOCK` | Blocked | warning | `statusRole === 'Blocked'` (no util cap — removed) |
| `R_GEN_BUFFER` | WIP Buildup | info | Buffer `(entries - exits) / entries > 0.15` |
| `R_GEN_RIGHTSIZE` | Underutilized | info | Parallel group where surplus stations exist at target output |
| `R_GEN_NOMATCH` | Unmatched | info | Stations with `entries === 0 && exits === 0 && working === 0 && idRaw === id` |

**Every rec now carries a `label` field** used for the badge text. Severity controls color only; the label conveys the issue type. Badge colors: `badge-critical` = red, `badge-warning` = amber, `badge-info` = blue.

### Card–Panel sync (~line 2884)

After all rules run, recs are filtered in place:

```js
const constrainedIds = new Set(
  state.stations.filter(s => s.statusRole !== 'Healthy').map(s => s.id)
);
recs.splice(0, recs.length, ...recs.filter(r =>
  !r.targets || r.targets.length === 0 ||
  r.targets.some(id => constrainedIds.has(id))
));
```

Rules fire on raw data; this filter ensures no card surfaces for stations that are currently Healthy (e.g., a routing imbalance that isn't causing a visible constraint in a healthy run).

### R_GEN_BLOCK specifics (rewritten 18-06-26)

Old: fired only when `blocked > 25 && util < 20`.
New: fires for **all** `statusRole === 'Blocked'` stations (util cap removed). Walks `TOPOLOGY.edges` from the worst blocked station to name the specific downstream buffer in the Fix text.

### R_GEN_RIGHTSIZE specifics (algorithm fix 18-06-26)

The formula now distinguishes balanced from imbalanced parallel groups:

```
Balanced:   totalLoad = N × avgUtil     (each station carries 1/N of the total)
Imbalanced: totalLoad = maxUtil         (busiest station already carries ALL the load)

nMin = ceil(totalLoad × scaleRatio / 80%)
```

Previously `N × maxUtil` was used for imbalanced groups, inflating the total load by ×5 and incorrectly computing nMin=2 when the answer is 1. The fix aligns the math with observed behavior: the user confirmed removing 4 of 5 assembly stations was valid.

---

## Constraints & Causes Panel (~line 2897)

Three tiers rendered in order:

```
● Bottleneck  (red)
  └─ [station list with → Fix links]

● Constrained  (amber)
  ├─ Blocked  ←── sub-heading (only shown when both Blocked + Starved exist)
  │   └─ [station list with → Fix links]
  └─ Starved
      └─ [station list with → Fix links]

● Healthy  (green, collapsible)
  └─ "N stations ▸"  →  click to expand, shows util% per station
```

**Fix link logic** (`bestFixRec`): Searches recs by exact station ID first, then falls back to the role-matched generic rec (`R_GEN_STARVED` for Starved, `R_GEN_BLOCK` for Blocked, `R_GEN_BOTTLE` for Bottleneck). Every constrained station now always has a → Fix link.

The Constrained panel only shows stations with `entries > 0` — inactive stations are excluded from all three tiers.

---

## FAB Tray (#fab-tray, ~line 946 CSS / ~line 1634 HTML)

The three corner buttons (print, theme, help) are wrapped in a `<div id="fab-tray">` that provides the liquid-glass backdrop:

```css
#fab-tray {
  position: fixed; bottom: 24px; left: 24px;
  display: flex; flex-direction: column; align-items: center; gap: 6px;
  padding: 10px; border-radius: 36px;
  background: rgba(28, 28, 32, 0.54);
  backdrop-filter: blur(32px) saturate(180%);
  border: 1px solid rgba(255, 255, 255, 0.10);
  box-shadow: inset 0 1px 0 rgba(255,255,255,0.07), 0 16px 48px rgba(0,0,0,0.50);
  z-index: 100;
}
```

Individual buttons inside the tray are simple 44×44 px circles with no backdrop-filter of their own (the tray handles all glassing). Order top-to-bottom: Print → Theme → Help. `#help-btn.help-active` styling is preserved.

---

## Node Highlight (rec-card click)

```css
.node-card-highlight rect {
  stroke: var(--accent) !important;
  stroke-width: 2.5 !important;
  filter: brightness(0.88) drop-shadow(0 0 14px var(--accent)) !important;
}
.node-card-highlight text {
  filter: drop-shadow(0 0 3px rgba(0,0,0,0.85)) drop-shadow(0 1px 2px rgba(0,0,0,0.7)) !important;
}
```

`brightness(0.88)` **darkens** the node on highlight rather than brightening it — the previous `brightness(1.4)` washed out green Healthy nodes and made white text unreadable. The text drop-shadow layers two dark halos to maintain legibility on any node color.

---

## Help Popup Formulas (~line 4136)

`showHelpPopup` now sets `help-popup-body` via **`innerHTML`** (previously `textContent`). This enables HTML formatting in the body strings.

All eight KPI entries now include human-readable formulas, e.g.:

```
pcs / month = Drain exits ÷ Simulated weekdays × Work days/year ÷ 12
Simulated weekdays = Simulation days × 5 ÷ 7

Confidence interval ±%:
Half-width = 1.96 ÷ √(exit count) × 100
```

**Rule:** all body strings must be self-contained safe HTML. Never include `<script>` or user-derived text. `<strong>`, `<br>`, `<em>`, and HTML entities (`&divide;`, `&times;`, `&ge;`, etc.) are all in use.

---

## Map Layout — renderMap() Multi-Pass

1. **Pass 1** — Grid: `x = offsetX + col × colW`, `y = offsetY + row × rowH`
2. **Pass 1b** — Per-line column centering: finds `maxCount` (tallest column), computes `lineMidY = offsetY + (maxCount-1) × rowH / 2`, centers each column's nodes around it
3. **Pass 2** — A→B gap: shifts all Line B nodes down by `shift = aBottom + TARGET_GAP − bTop` (TARGET_GAP = 132 px)
4. **Line C box** — computed as fixed position below Line B
5. **lineBoxes** — bounding boxes for Line A/B outlines, computed from final node positions after all passes; titles at `x+28, y+38` (matches "Future / For Sale" title offsets)

---

## Data Pipeline

```
User drops results_*.htm
  → parseHTM()
      resTable   → working/waiting/blocked/failed/unplanned %
      stateTable → entries/exits/maxContents
      drainTable → total throughput + lead time per drain node

      Node matching (2-stage):
        1. Primary regex: (?=.*searchId)(?=.*GearboxA|B)(?=.*subtype?)  [case-insensitive]
        2. Fallback: regex(node.id) as substring

      state.stations = TOPOLOGY.nodes.map(node => ({...node, ...nodeMetrics[node.id]}))
      TOPOLOGY.nodes is NEVER mutated.

  → analyze()
      statusRole assigned (0-entry → Healthy, then priority thresholds)
      Dynamic post-pass adjusts blocked threshold if needed
      RULES.forEach → evaluate() → recs[]
      recs filtered to non-Healthy targets
      Constraints panel: 3 tiers with sub-headings
      Bottom bar: hidden when recs.length === 0

  → renderMap()
      S-bend cubic bezier edges, isFeedback arcs, label-fitted node pills
      Flow-proportional arrow sizes on edges
```

---

## TOPOLOGY Node Schema

```js
{
  id:            string,   // unique key; comes from TOPOLOGY, never from HTM parse
  searchId:      string,   // primary HTM row substring
  line:          'A'|'B',
  subtype:       string?,  // secondary substring
  label:         string,   // display text
  col:           number,   // layout column (×220px)
  row:           number,   // layout row (×120px)
  isSource:      bool?,
  isBuffer:      bool?,
  isDrain:       bool?,
  parallelGroup: string?,  // used by R_GEN_BALANCE and R_GEN_RIGHTSIZE
}
```

---

## UI/UX Rules (enforce on every edit)

- **CSS variables only.** Never write `red`, `#FF0000`, `orange`. Use `var(--color-critical)`, `var(--color-moderate)`, `var(--color-healthy)`, `var(--accent)`, `var(--text-*)`, `var(--surface-*)`.
- **8-point grid.** All spacing/sizing in multiples of 8px (4px for fine details).
- **No raw colors in JS.** Detail panel stat-bar swatches use CSS variables — keep aligned with `getRoleColor()`.
- **TOPOLOGY is read-only.** Never mutate `TOPOLOGY.nodes` or `TOPOLOGY.edges`.
- **PlantSim terminology.** Use "Min. Contents" (not "Shortest Queue"). Use "entrance control" (not "exit control"). SimTalk snippets use `exitLocked` only.
- **`tip.textContent`** for the `.more-nodes` tooltip — never `tip.innerHTML` for plain text.
- **Help popup body uses `innerHTML`** — keep it safe HTML with no user-derived content.
- **Smart quotes kill the app.** Never paste from a word processor into the `<script>` block. Curly quotes (`'` `"`) as JS string delimiters cause a `SyntaxError` that silently kills all event listeners.

---

## Known Gaps / Potential Next Steps

1. **Dynamic topology** — `TOPOLOGY` is hardcoded for Gearbox A/B. A different model requires manual edits to the node/edge list.
2. **Per-line R_GEN_STARVED** — starvation card covers all starved stations in one card regardless of line; could generate separate A/B cards for clarity.
3. **R_GEN_BOTTLE hardcodes ME1/ME2** — duplicate-station advice is ME-specific. For non-ME bottlenecks it falls back to generic capacity advice.
4. **R_PLANTSIM_GATING MODEL_CONFIG sync** — gating rule depends on `MODEL_CONFIG.BUFFER_CAPS` which must be kept in sync with validated caps in `documentation.md`.
5. **Resizer persistence** — the horizontal resizer height is not persisted across page reloads.

---

## WIP / MU KPI Naming (updated 18-06-26)

Three KPI rows in the Simulation Report pill relate to parts-in-system:

| Pill label | Element ID | What it measures | Formula |
|---|---|---|---|
| **System MU** | `#kpi-wip` | Net parts in the system at run end | `sum(entries − exits)` across all non-source, non-drain stations |
| **Max MU** | `#kpi-max-wip` | Peak simultaneous MU count | `sum(maxContents)` across all non-source, non-drain stations |
| **Little's WIP** | `#kpi-wip-mo` | Finished gearboxes simultaneously in system | `(pcs/month ÷ 30.42) × (leadTimeMin ÷ 1440)` |

**Why Little's WIP replaced the old "WIPs/mo":** The original formula counted total source exits per month — one MU per sub-part, so ~6× the finished-gearbox count. This was a simulation bookkeeping artefact, not an inventory measure. Little's Law gives a meaningful, sub-part-independent figure in finished gearboxes, directly comparable across lines and versions.

```js
// Little's Law WIP calculation (~line 2475)
const wipLLA = (moA && ltA && ltA.leadTimeMin) ? moA * ltA.leadTimeMin / (30.42 * 1440) : null;
const wipLLB = (moB && ltB && ltB.leadTimeMin) ? moB * ltB.leadTimeMin / (30.42 * 1440) : null;
// moA/moB = pcs/month (already work-calendar normalised)
// ltA/ltB.leadTimeMin = mean lead time in minutes from drainTable
// 30.42 * 1440 = minutes per month (calendar time)
```

Result is in **finished gearboxes** (decimal, one place). Display format: `X.X gearboxes (A: X.X / B: X.X)`.

---

## Theme Toggle Icon Convention

The theme button shows the icon for the **state you will switch TO**, not the current state:

```js
// applyTheme(light):
iconDark.style.display  = light ? ''      : 'none';  // moon shown when currently light → click goes dark
iconLight.style.display = light ? 'none'  : '';       // sun shown when currently dark → click goes light
```

`theme-icon-dark` = crescent moon SVG path. `theme-icon-light` = circle + 8 rays SVG.

---

## What Changed Since 15-06-26 Handoff

| Area | Change |
|---|---|
| FAB buttons | Wrapped in `#fab-tray` liquid-glass pill; tray provides all backdrop-filter/blur; individual buttons simplified to icon containers |
| Constraint thresholds | `STARVED_THRESH` 15 → 20; `BLOCKED_THRESH` 20 → 28; `BLOCK_CARD_UTIL_MAX` 20 → 100 (cap removed) |
| Constraint classification | 0-entry stations short-circuit to Healthy before any threshold check (prevents 100% waiting artefact triggering Starved) |
| Dynamic post-pass | When no Bottleneck exists and > 55% of machines are Blocked, effective threshold raised to p75 of blocked values (max 40%) |
| `roleSeverity` field | Added to each station alongside `statusRole` (critical / high / moderate / none) |
| Constraints panel | Three tiers: Bottleneck → Constrained (with Blocked / Starved sub-headings when both present) → Healthy (collapsible) |
| Fix links | `bestFixRec()` helper: exact ID match first, then role-based fallback; every constrained station now has a → Fix link |
| Card labels | `label` field added to every rec; badges show issue name (Bottleneck, Exit Strategy, Starved, Blocked, Deadlock Risk, Overproduction, WIP Buildup, Underutilized, Unmatched) instead of WARNING/CRITICAL/INFO |
| Card–panel sync | Recs filtered post-rules to only those targeting non-Healthy stations; `recs.splice` in-place so all downstream references work unchanged |
| Bottom bar | Hides `#bottom-panel` and `#horizontal-resizer` via `style.display = 'none'` when `recs.length === 0` |
| R_GEN_BLOCK rewrite | Now targets `statusRole === 'Blocked'` directly; walks TOPOLOGY edges to name specific downstream buffer in Fix text |
| R_GEN_RIGHTSIZE fix | Imbalanced groups use `totalLoad = maxUtil` (not `N × maxUtil`); fixes nMin=2 when answer is 1 for fully-imbalanced routing |
| Node highlight | `brightness(1.4)` → `brightness(0.88)` (darkens instead of washing out); text gets `drop-shadow` for legibility on any node color |
| Help popup | Body rendered via `innerHTML` (was `textContent`); all KPI entries include human-readable formulas with HTML entities and `<strong>` headings |
| Title alignment | Line A/B box titles changed to `x+28, y+38` — matching "Future / For Sale" offsets for consistent visual margin |
| Constraints list help | Updated to describe all three tiers and the dynamic threshold adjustment |
| README | Updated throughout to reflect three-tier classification, new card labels, FAB tray, and current line count |
| **Little's WIP** | "WIPs / mo" (source-exit artefact, ~6× inflated) replaced with Little's Law WIP in finished gearboxes: `(pcs/month ÷ 30.42) × (leadTimeMin ÷ 1440)` |
| **MU renames** | "System WIP" → "System MU"; "Max WIP" → "Max MU" in pill labels, print slide, and help popup text |
| **Theme icon** | Icons now show the *target* state — ☀️ sun while in dark mode, 🌙 moon while in light mode (display logic swapped in `applyTheme()`) |
