# AI Handoff Document — PlantSim Visualizer
**Last updated:** June 15, 2026 (help mode + drag-drop fix)
**Supersedes:** `Handoff_10-06-26.md`

---

## Project Overview

Single-file client-side web app (`gearboxsim.html`, **3257 lines**) that parses PlantSim HTML reports and visualizes them as an interactive D3.js topology map. No build step, no npm. The entire app — CSS, inline SVG symbols, HTML, and JS — lives in one file.

**Goal:** Replace manual spreadsheet analysis with a visual, actionable dashboard that instantly surfaces bottlenecks, starvation, WIP excess, and KPI deviations — with recommendation cards specific to Siemens Tecnomatix Plant Simulation actions.

---

## File Sections (accurate as of 15-06-26)

| Section | Lines | Notes |
|---|---|---|
| `<style>` | 8–989 | All CSS. Uses CSS variables from `:root`. **Never use raw colors.** Always `var(--color-*)`. |
| SVG `<defs>` | 990–1052 | Inline `<symbol>` icons referenced via `<use href="#icon-name">`. Icons: `#icon-drilling`, `#icon-grinding`, `#icon-milling`, `#icon-turn`, `#icon-measuring`, `#icon-assemly`, `#icon-gear`, `#icon-cylinder`, `#icon-cube`, `#icon-drain`, `#icon-database`, `#icon-fallback`. |
| HTML panels | 1053–1210 | `#tooltip` (pos:fixed, used by both D3 hover and `.more-nodes` spans), `#drop-overlay`, `#app-container` (grid: header + map + right-panel + bottom-panel) |
| `<script>` begins | 1211 | |
| `scrollToRec()` | 1281 | Scroll-highlights a rec card by ID and pans/blinks the map node. Guards `if (el)` on getElementById. |
| `TOPOLOGY` const | 1338 | Read-only node/edge definitions. **Never mutate.** `state.stations` is always a fresh `map()` copy in `parseHTM()`. |
| `CONFIG` const | 1420 | All diagnostic thresholds. Edit here only. |
| `parseHTM()` | 1459 | HTM parsing pipeline — produces `state.stations`, `state.drains`, `state.meta`. |
| `analyze()` | 1597 | KPI computation, statusRole assignment, constraints list, rec cards. |
| `moreSpan()` | 1716 | Function declaration inside `analyze()` (hoisted). Generates `<span class="more-nodes" data-tooltip="...">+N more</span>`. |
| `RULES = [` | 1721 | Array of 8 rule objects, each with `id` and `evaluate(stations, drains)`. |
| `initD3()` | 2048 | D3 setup, zoom behavior, edge/node rendering. |
| `getRoleColor()` | 2091 | Maps `statusRole` → CSS color variable. |
| `renderMap()` | 2126 | Full D3 map re-render (called from `analyze()` and on resize). |
| `.more-nodes` event delegation | 2516 | Three `document.addEventListener` calls (mouseover, mousemove, mouseout) for the `+N more` tooltip. Uses `_moreNodesRafId` to cancel stale rAF. |
| `handleMouseOver` | 2446 | D3 node hover tooltip. |
| `handleMouseOut` | 2505 | D3 node unhover. |
| `handleClick` | 2549 | D3 node click → opens detail panel. |
| Help Inspect Mode | 3024–3185 | `enterHelpMode / exitHelpMode / showHelpPopup / identifyElement` + event listeners. |

---

## CONFIG Object (line 1420)

```js
const CONFIG = {
  BOTTLENECK_THRESH:    75,   // util% → statusRole = 'Bottleneck'
  STARVED_THRESH:       15,   // waiting% → statusRole = 'Starved'
  BLOCKED_THRESH:       20,   // blocked% → statusRole = 'Blocked'
  BLOCK_CARD_BLOCKED_MIN: 25, // R_GEN_BLOCK trigger minimum (> BLOCKED_THRESH intentionally)
  BLOCK_CARD_UTIL_MAX:  20,   // R_GEN_BLOCK trigger maximum util%
  OVERPROD_MULTIPLIER:  1.5,
  BALANCE_RATIO_MIN:   0.1,
  FAILURE_THRESH:        5,   // failed% for tooltip/detail flagging; no rec card generated
  BUFFER_WIP_RATIO:    0.15,
  TARGET_OUTPUT:       300,   // pcs/mo target for KPI deviation coloring
  WORK_DAYS_PER_YEAR:  250,   // editable via UI stepper
};
```

**Threshold gap:** `BLOCKED_THRESH=20` is intentionally below `BLOCK_CARD_BLOCKED_MIN=25`. Stations blocked 20–24% appear in the Constrained tier but generate no Fix card — the Fix link is suppressed (not a fallback) when no rec card targets the station.

---

## statusRole Classification (line 1639)

Sources, drains, and buffers are always `'Healthy'` — they are never classified as Bottleneck/Starved/Blocked regardless of metrics.

For machines only (in priority order):
1. `util > 75 && waiting < 5` → `'Bottleneck'`
2. `waiting > 15` → `'Starved'`
3. `blocked > 20` → `'Blocked'`
4. else → `'Healthy'`

---

## Node Color Scheme (line 2091)

```js
function getRoleColor(role) {
  if (role === 'Bottleneck') return 'var(--color-critical)';   // red
  if (role === 'Starved' || role === 'Blocked') return 'var(--color-moderate)'; // amber
  return 'transparent';
}
```

**Two tiers only.** Blocked and Starved are collapsed into a single amber "Constrained" color. The Constraints panel still shows distinct metric labels ("Waiting X%" vs "Blocked X%") to preserve directional information without needing a third color.

---

## RULES Array (line 1721) — 8 rules in order

| ID | Line | Severity | Trigger |
|---|---|---|---|
| `R_GEN_BOTTLE` | 1724 | critical | `statusRole === 'Bottleneck'`; generates one card per line (A/B) |
| `R_GEN_BALANCE` | 1762 | critical | `min(entries)/max(entries) < 0.1` within `parallelGroup` |
| `R_PLANTSIM_GATING` | 1804 | warning | ME1 or ME2 is Bottleneck AND any buffer is near its cap ceiling (≥90% of `MODEL_CONFIG.BUFFER_CAPS`) |
| `R_GEN_STARVED` | 1829 | warning | Any machine has `statusRole === 'Starved'`; **always fires when any Starved station exists**, so `recs` is never empty when a Starved station is in the Constrained tier |
| `R_GEN_OVERPROD` | 1871 | warning | `entries > lineDrain × 1.5 && statusRole === 'Blocked'` |
| `R_GEN_BLOCK` | 1898 | warning | `blocked > 25 && util < 20 && !isSource` |
| `R_GEN_BUFFER` | 1921 | info | Buffer `maxContents` near/at cap (from HTM `stateTable`) |
| `R_GEN_NOMATCH` | 1947 | info | Stations with `idRaw === node.id` (fallback match, no real HTM row found) |

**R_GEN_FAILURE was removed** — the user cannot change failure rates in Tecnomatix Plant Simulation, so this card gave non-actionable advice and was deleted.

### R_GEN_BOTTLE specifics
- Generates one card per affected line, not one card for all bottlenecks.
- If the bottleneck is ME1 or ME2: advises duplicating the station as ME1b/ME2b with Min. Contents exit strategy on feeders and the return buffer. Cites expected +20–35% throughput gain.
- Uses `moreSpan()` to overflow station names beyond the first shown.
- Does NOT branch on failure rate (verified: a failure-dominated station naturally has low utilization and won't trip the bottleneck threshold).

### R_GEN_BALANCE specifics
- Uses `parallelGroup` node metadata from TOPOLOGY to identify which stations compete.
- Special-case for `groupId === 'B_asm'`: names D3, Buf_Gears_B, Buf_Shafts_B explicitly in the fix string.
- Uses correct PlantSim terminology: "Min. Contents" (not "Shortest Queue").

### R_PLANTSIM_GATING specifics
- `targets` contains ME1/ME2 IDs only (not buffer IDs).
- Buffers are always `statusRole = 'Healthy'` and never appear in the Constrained tier, so the missing buffer IDs in `targets` cause no Fix-link routing issue.
- Explains grinder-gating SimTalk pattern in the fix string with inline `<code>` elements.

---

## Key CSS Classes (added since 10-06-26 handoff)

```css
/* Overflow station names in rec cards and KPI pill */
.more-nodes {
  display: inline-block;
  cursor: help;
  color: var(--accent);
  font-size: 11px;
  font-weight: 500;
  border-bottom: 1px dashed currentColor;
  white-space: nowrap;
}

/* Inline code snippets in rec card fix text */
.rec-card code {
  font-family: var(--font-mono);
  font-size: 11px;
  background: rgba(255,255,255,0.06);
  border: 1px solid rgba(255,255,255,0.1);
  border-radius: 4px;
  padding: 1px 5px;
  color: var(--accent);
}
```

---

## .more-nodes Tooltip System (line 2516)

The `+N more` spans use **event delegation on `document`** (not CSS `::after`) to reuse the existing `#tooltip` div. This avoids clipping by `.rec-card { overflow: hidden }`.

Key implementation details:
- `_moreNodesRafId` (declared at line 2516, module-scoped) cancels stale rAF before queuing a new one — prevents unbounded rAF accumulation on mousemove.
- Tooltip content written via `tip.textContent` (not `innerHTML`) — safe for any text input and preserves `\n` line breaks because `#tooltip` has `white-space: pre-line`.
- Items in `data-tooltip` are joined with `\n` in `moreSpan()`.

---

## Constraints List Rendering (line 1982)

Two tiers rendered in order: Bottleneck (red), then Constrained (amber = Starved + Blocked merged).

Fix-link logic:
```js
const targetingRec = recs.find(r => r.targets && r.targets.includes(s.id));
const fixLink = targetingRec
  ? `<a class="fix-link" onclick="scrollToRec('${targetingRec.id}', '${s.id}')">→ Fix</a>`
  : '';
```

If no rec targets the station (Blocked 20–24% gap), the Fix link is **omitted**, not fallen back to an unrelated card.

Issue lines are appended with `insertAdjacentHTML('beforeend', ...)` (not `innerHTML +=`) to avoid O(N²) re-parse on each iteration.

---

## Detail Panel Color Alignment (line 2606)

The stat-bar legend swatch for "Block" uses `var(--color-moderate)` — same amber as the node dot and Constrained tier header. All three surfaces now agree: node border, constraints list dot, detail panel swatch.

---

## Data Pipeline

```
User drops results_*.htm
  → parseHTM()
      resTable   → working/waiting/blocked/failed/unplanned %
      stateTable → entries/exits/maxContents
      drainTable → total throughput per drain node

      Node matching (2-stage):
        1. Primary regex: (?=.*searchId)(?=.*GearboxA|B)(?=.*subtype?)  [case-insensitive]
        2. Fallback: regex(node.id) as substring

      state.stations = TOPOLOGY.nodes.map(node => ({...node, ...nodeMetrics[node.id]}))
      TOPOLOGY.nodes is NEVER mutated.
      node.id always comes from TOPOLOGY (hardcoded); never overwritten by HTM-parsed idRaw.

  → analyze()
      statusRole assigned per machine (sources/drains/buffers always Healthy)
      RULES.forEach → evaluate() → rec cards pushed to recs[]
      Constraints list built: Bottleneck tier, then Constrained tier
      Rec cards rendered in bottom panel

  → renderMap()
      S-bend cubic bezier edges, isFeedback arcs, label-fitted node pills
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
  parallelGroup: string?,  // used by R_GEN_BALANCE
}
```

---

## UI/UX Rules (enforce on every edit)

- **CSS variables only.** Never write `red`, `#FF0000`, `orange`. Use `var(--color-critical)`, `var(--color-moderate)`, `var(--color-healthy)`, `var(--accent)`, `var(--text-*)`, `var(--surface-*)`.
- **8-point grid.** All spacing/sizing in multiples of 8px (4px for fine details).
- **No raw colors in JS.** The detail panel stat-bar legend swatches use CSS variables — keep them aligned with `getRoleColor()`.
- **TOPOLOGY is read-only.** Never mutate `TOPOLOGY.nodes` or `TOPOLOGY.edges`.
- **PlantSim terminology.** Use "Min. Contents" (not "Shortest Queue") in all rec card text. Use "entrance control" (not "exit control"). SimTalk snippets use `exitLocked` only.
- **`tip.textContent`** for the `.more-nodes` tooltip — never `tip.innerHTML` for plain text.

---

## Help Inspect Mode (lines 3024–3185)

Press `?` (or click the `#help-btn` button) to enter help mode. The body receives `class="help-mode"`, which:

- Changes the cursor to a custom SVG pointer+blue-`?` badge via `cursor: url(data:image/svg+xml,...) !important` on `body.help-mode *`
- Makes `#help-overlay` (`position: fixed; inset: 0; z-index: 8000; background: transparent`) cover the entire viewport and intercept clicks

**Click flow:**
1. `#help-overlay` click handler fires, temporarily sets `pointer-events: none` on the overlay, calls `document.elementFromPoint()` to find the real element underneath, restores pointer-events, then calls `showHelpPopup(x, y, identifyElement(el))`.
2. `identifyElement(el)` walks the DOM with priority-ordered `.closest()` checks across all ~25 element types and returns `{category, title, body}`.
3. `showHelpPopup` positions a glassmorphism `#help-popup` card (z-index 8001) near the cursor, clamped to the viewport.

**Escape / exit:** First Escape dismisses the popup (if visible); second Escape calls `exitHelpMode()`. Clicking `#help-btn` again also toggles off.

**Drag-drop through help mode:** The `window.dragover` handler calls `e.preventDefault()`, which signals drop-allowed through the bubble chain even when `#help-overlay` is the event target. Dropping a file while in help mode loads the new run and hides the drop overlay normally.

**Bug fixed (15-06-26):** The initial help mode implementation contained 347 Unicode smart/curly quotes (`'` U+2018/19, `"` U+201C/1D) used as JS string delimiters. Chrome reported `SyntaxError: Invalid or unexpected token` at the first curly double-quote on script compile, causing the **entire `<script>` block to fail** — including all drag-drop event listeners. Fixed by replacing all curly quotes with straight equivalents in the JS block. **Never paste help text into the JS block from a word processor or rich-text editor** — always verify with a byte-level check for smart quotes before shipping.

---

## Known Gaps / Potential Next Steps

1. **Dynamic topology** — `TOPOLOGY` is hardcoded for Gearbox A/B. A different model requires manual edits.
2. **Per-line R_GEN_STARVED** — starvation card covers all starved stations in one card regardless of line.
3. **R_GEN_BOTTLE hardcodes ME1/ME2** — the duplicate-station advice is ME-specific, not generalizable. For other bottlenecks (non-ME machines), it gives generic capacity advice.
4. **D3 drag vs. pan conflict** — vertical resize handle near bottom panel has hitbox overlap with D3 pan.
5. **R_PLANTSIM_GATING `MODEL_CONFIG.BUFFER_CAPS`** — gating rule depends on a separate `MODEL_CONFIG` object that must be kept in sync with the validated caps in `documentation.md`.

---

## What Changed Since 10-06-26 Handoff

| Area | Change |
|---|---|
| Rec cards | `R_GEN_FAILURE` removed; all remaining cards give PlantSim-specific actions |
| Rec cards | `R_GEN_BOTTLE` reworked: ME1/ME2 gets "duplicate as ME1b/ME2b + Min. Contents" advice |
| Rec cards | `R_GEN_BALANCE` reworked: uses "Min. Contents" terminology; B_asm special case names D3/Buf_Gears_B/Buf_Shafts_B |
| Rec cards | NEW `R_PLANTSIM_GATING`: deadlock-risk warning when ME is bottleneck and buffers near ceiling |
| Node colors | Blocked + Starved collapsed to single amber "Constrained" tier; `getRoleColor()` updated |
| Overflow names | `moreSpan()` helper + `.more-nodes` CSS class + `#tooltip` event delegation for +N more hover |
| KPI pill | Bottleneck list uses `moreSpan()` for overflow; pill uses `innerHTML` not `innerText` |
| Fix links | Fix link suppressed (not fallen back) when no rec targets a Constrained station |
| Fix links | `insertAdjacentHTML` replaces `innerHTML +=` in constraints loop |
| Detail panel | "Block" swatch changed from `var(--accent)` (cyan) to `var(--color-moderate)` (amber) |
| Tooltip | `_moreNodesRafId` cancel guard prevents rAF accumulation on mousemove |
| Tooltip | `tip.textContent` replaces `tip.innerHTML` |
| Help mode | NEW: `?` button enters Help Inspect Mode; click any element for a glassmorphism popup describing it |
| Bug fix | 347 smart/curly quotes in JS block caused `SyntaxError` on load, silently killing all event listeners (including drag-drop); replaced with straight quotes |
