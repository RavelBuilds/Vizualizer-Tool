# Plan: Truly Dynamic Topology for GearboxSim Visualizer

**Goal:** `gearboxsim.html` should reconstruct the *entire* flow graph — nodes, edges, lines, parallel groups, layout — from the dropped result file(s), with **zero hardcoded topology**. Drop any model's HTM and get a correct map.

**Status quo:** the topology (nodes + edges + col/row positions) is a hand-written constant `TOPOLOGY` (~130 lines in `gearboxsim.html`). New objects (e.g. housing buffers) are now *captured dynamically into the data tables* (2026-06-19 change) but are **not placed on the flow map**, because the map needs edges and positions the HTM doesn't provide.

---

## 1. The core constraint: the HTM has no edges

A PlantSim **Statistics** report exports only *object rows* with metrics:

| What the HTM gives us | What it does NOT give us |
|---|---|
| Full object name (`.Models.Model.MI1_Milling_GearBoxA_Housing1`) | Any connector / predecessor / successor data |
| working / waiting / blocked / failed / paused / unplanned % | Which station feeds which |
| entries, exits, maxContents, relativeFull | Physical layout / coordinates |
| drain throughput, lead time | Parallel-group membership |

**Everything about *connectivity* must therefore come from one of two channels:**

- **Channel A — Naming convention** (your current plan): encode the edge inside each buffer's name.
- **Channel B — A SimTalk topology export** (recommended): a tiny method dumps the real connector graph to a second file the tool reads alongside the stats.

These are not mutually exclusive — but one of them must carry the edges. Naming alone cannot be avoided unless you add Channel B.

---

## 2. Is your buffer-naming scheme enough? — Mostly yes, with caveats

Your scheme: `Buffer_<Line>_<Part>_<From>_<To>`
Example: `Buffer_GearboxA_HousingPart_MI1_A1`

If you place a buffer between **every** connected station pair, then **the set of all buffer names IS the complete edge list.** That genuinely makes the graph reconstructable from the HTM. This is a valid and clever approach. The caveats:

### 2.1 What it derives correctly
- **Edges:** `From → Buffer → To` for every buffer → collapse to logical edge `From → To`.
- **Lines:** the `GearboxA` / `GearboxB` token → line assignment. ✔ Sufficient.
- **Part lanes:** the `HousingPart` / `Gears` / `Shafts` token → vertical lane within a line (nice-to-have for layout).

### 2.2 Where it breaks without hardening

| Risk | Example | Fix |
|---|---|---|
| **Prefix ambiguity** | token `A1` also matches `A10`, `A11` | Resolve tokens against the **first underscore-segment** of full station names, matched **exactly** (`A1` ≠ `A11`). Confirmed distinct in current model. |
| **Underscores inside fields** | `HousingPart` is one token but a part could be `Housing_Part`; station IDs never have `_` but parts might | Use a **fixed field count** (exactly 5 fields) OR a **double-underscore field separator**: `Buffer__<From>__<To>`. Recommended: full station IDs + `__`. |
| **Non-buffered edges** | Source→first station, last station→Drain, and any link you *didn't* put a buffer on | Either buffer those too, or let the layout engine infer them (Source/Drain heuristics, §4.3). |
| **Feedback / return edges** | `A6 → Buffer_Return_A → ME1` points "backwards" | Detect as a back-edge in the rank graph (§4.4) — no naming needed, but a `Return` token in the name makes it explicit and robust. |
| **Direction** | is `Buffer_..._MI1_A1` MI1→A1 or A1→MI1? | **Fix field order = `From_To`** and document it. Cross-check against entries/exits if ever ambiguous. |
| **Shared / cross-line stations** | ME3 serving both A and B | Name carries both tokens (`GearboxAB`) or the edge set connects both components → flag `line: 'AB'`. |

### 2.3 The decisive recommendation on the name

Adopt a **single rigid grammar** and validate it on load:

```
Buffer__<FromID>__<ToID>[__<tag>]
```

- `<FromID>` / `<ToID>` = the exact first-segment station ID as it appears in the HTM (`MI1`, `A1`, `A11`, `ME1`, `D3`, `Drain_GearboxA`…).
- `__` (double underscore) separates the three logical fields so internal single underscores never confuse the parser.
- optional `<tag>` = `Return` (feedback), or a part label for lanes.
- Line & part are **derived from the endpoint stations**, not re-stated in the buffer — removes one source of disagreement.

Example: `Buffer__A6__ME1__Return`, `Buffer__MI1__A1`.

> If you keep your current `Buffer_<Line>_<Part>_<From>_<To>` form, the tool can still parse it via "last two segments = From,To" — but the `__` grammar is markedly more robust. **This is decision #1 for you (see §7).**

---

## 3. Recommended alternative/supplement: SimTalk topology export (Channel B)

The most robust path is to stop encoding edges in names and instead **export the real graph** once, with a small method. PlantSim objects expose their successors, so a method can walk every object and emit `from,to` rows to an HTML/CSV that you drop alongside the stats file.

Sketch (to be finalised in `plantsim-simtalk` skill):
```simtalk
-- dumpTopology: write every connector as a "from -> to" row to a TableFile,
-- then export that table to HTML next to the stats report.
-- Iterate root.Model, for each object read its successors and record edges.
```

**Why it's better:** authoritative edges (no guessing, no naming discipline), survives renames, captures non-buffered links and feedback automatically. **Cost:** one extra exported file per run, and writing/validating the method.

**Verdict:** Channel A (names) is fine for the pitch timeline and works with files you already produce. Channel B is the "correct" long-term engineering answer. The visualizer rework below is designed to accept **either** edge source through one internal interface.

---

## 4. The visualizer rework — what actually has to be built

Replace the hardcoded `TOPOLOGY` with a pipeline: **parse → classify → build edges → layer → lay out → render.**

### 4.1 Parse (mostly exists)
Reuse the current `parsedData` builder. Keep every object row (already done by the dynamic-capture pass).

### 4.2 Classify node type (by name + stats)
| Type | Detection |
|---|---|
| Source | name `Source*`/`Quelle*`; or 0 entries + ~100% blocked |
| Drain | name `Drain*`/`Senke*`; or ~100% working/waiting + 0 exits-onward |
| Buffer | name `Buffer*`/`Puffer*` |
| Measuring | name `ME\d`/`Measuring` |
| Station | everything else with entries > 0 |

### 4.3 Build the edge list
- **From buffers (Channel A):** parse each `Buffer__From__To`; resolve `From`/`To` to real node IDs by exact first-segment match; emit logical edge `From → To` and attach the buffer's fill metrics to that edge.
- **From export (Channel B):** read edge rows directly.
- **Infer the unbuffered ends:** any Source with no outgoing buffer → connect to the nearest station sharing its line+part token; any Drain with no incoming buffer → connect from the measuring/last station of its line. (Or require buffers/edges there too.)

### 4.4 Layer into columns (auto-rank)
- Build directed graph from **forward** edges only.
- **Detect feedback edges:** run a topological sort; any edge whose target would precede its source (back-edge) is marked `isFeedback` and removed from ranking. (`Return` tag, if present, confirms it.)
- **Column = longest path from any source** (rank). This reproduces the current left-to-right column layout automatically.

### 4.5 Assign rows (lanes + parallel stacking)
- **Line** = `Gearbox[AB]` token (or weakly-connected component). Lines stack vertically (A on top, B below), exactly like today.
- **Part lane** within a line = `Housing` / `Gears` / `Shafts` token → sub-rows.
- **Parallel group** = nodes in the same column with the **same predecessor-set and successor-set** (e.g. A1/A2/A3) → stack them and reuse the existing `parallelGroup` rightsizing logic.
- Reuse the current per-line column-centering and A/B gap-normalization passes (Pass 1b / Pass 2 in `renderMap`) unchanged — they already operate on computed positions.

### 4.6 Render (mostly exists, two changes)
- Feed the **computed** nodes/edges into the existing D3 render instead of the constant.
- **Buffers as edge adornments, not boxes:** since a buffer encodes an edge, draw it as the existing **`buf-ring`** fill indicator *on that edge*, not as a separate node. This keeps the map readable even with a buffer between every station, and still surfaces buffer fill/blocking. (Standalone buffer boxes become optional/toggleable.)

---

## 5. What stays, what goes

| Component | Fate |
|---|---|
| `parsedData` parser, drain/importer parsing | **Keep** |
| Dynamic-capture pass (2026-06-19) | **Keep** as fallback for anything the edge-builder misses |
| `TOPOLOGY.nodes` / `TOPOLOGY.edges` constants | **Replace** with computed graph |
| `archVariant` old/shared/parallel edge filtering | **Replace** — architecture is now whatever the edges say; no variants needed |
| `searchId` / `searchRegex` per-node matching | **Remove** — matching becomes ID-based off real names |
| Column-centering / gap-normalization / parallel rightsizing | **Keep** — operates on computed positions |
| `buf-ring`, node cards, diagnostics, KPIs | **Keep** |

---

## 6. Migration path (low-risk, incremental)

1. **Phase 0 (done):** dynamic-capture pass — unknown objects appear in tables/KPIs.
2. **Phase 1:** lock the buffer-naming grammar (§2.3) and rename existing buffers to match; add a **validator** in the tool that lists any buffer whose name doesn't parse (so you catch typos immediately).
3. **Phase 2:** build the **edge list** from buffer names; render buffers as edge rings on top of the *existing* hardcoded node positions (visual diff = sanity check against today's known-good map).
4. **Phase 3:** replace node positions with the **auto-layer/auto-layout** engine; keep the old `TOPOLOGY` behind a `?legacyLayout=1` flag for one release to compare.
5. **Phase 4:** delete `TOPOLOGY`; optionally add Channel B (SimTalk export) for authoritative edges.

Each phase is shippable and reversible.

---

## 7. Decisions needed from you

1. **Naming grammar:** adopt `Buffer__<From>__<To>[__tag]` (recommended) or keep `Buffer_<Line>_<Part>_<From>_<To>`? *(Affects the parser spec.)*
2. **Edge coverage:** will you place a buffer on **every** edge (incl. Source→first, last→Drain, and return loops), or should the tool infer the unbuffered ends heuristically?
3. **Return edges:** add a `Return` tag to feedback buffers, or rely on automatic back-edge detection?
4. **Channel B:** worth building the SimTalk topology export now, or defer until after the pitch and rely on names?
5. **Buffer rendering:** buffers as **edge rings** (recommended, clean) or as **standalone boxes** (closer to PlantSim canvas)?

---

## 8. Direct answers to your questions

- **"Is the line derivable from the names?"** — Yes. The `GearboxA`/`GearboxB` token on stations (and/or buffers) is enough to assign lines; shared stations get `AB`.
- **"Is the buffer name enough to derive the topology?"** — Yes for **edges**, *if* a buffer sits on every link and the name reliably encodes `From`/`To`. Harden the grammar (`__` separators, exact first-segment IDs) to kill the `A1`/`A11` ambiguity.
- **"Am I missing something?"** — Three things: (1) the HTM has **no connector data**, so names are load-bearing — a typo = a missing edge; (2) **layout** still has to be computed (auto-layering), it isn't in the names; (3) **non-buffered links** (sources, drains, feedback) need either buffers too or inference. The SimTalk export (Channel B) removes all three worries if you want the bulletproof version.

---

*Plan authored 2026-06-19. Companion to `documentation.md` (model topology) and the `plant-simulation` / `plantsim-simtalk` skills. Supersedes the hardcoded `TOPOLOGY` constant in `gearboxsim.html` once Phase 4 lands.*
