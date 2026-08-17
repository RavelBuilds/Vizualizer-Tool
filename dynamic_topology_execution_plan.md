# Execution Plan: Edge-Driven Dynamic Topology

**Outcome:** GearboxSim reads a **From/To edge table** (exported natively from PlantSim as a TableFile→HTML) and renders the flow map from it — no hardcoded topology. Falls back to the current hardcoded map when no edge table is present, so the pitch build never breaks.

**Hard constraints (every agent must honour):**
- **Never break the existing behaviour.** All new rendering is opt-in via `?dynamic=1`. With no flag and/or no edge table, the tool behaves exactly as today.
- **Single file** `gearboxsim.html` is large and currently working. Stay inside your assigned region.
- **Match surrounding code style** (vanilla JS, D3 v7, no build step, no new deps).
- **Commit your work** on your worktree branch with a clear message and report the branch name + commit hash in your final message.

---

## The interface contract (shared by all agents)

### Edge-table HTML shape (producer ↔ consumer agreement)
A native PlantSim TableFile HTML export styled like every other report table:

```html
<table class="Auswertungen">
<thead><tr><th>From</th><th>To</th></tr></thead>
<tbody>
  <tr><td>MI1_Milling_GearBoxA_Housing1</td><td>Buffer_GearboxA_HousingPart_MI1_A1</td></tr>
  <tr><td>Buffer_GearboxA_HousingPart_MI1_A1</td><td>A1_PreAssembly_GearBoxA</td></tr>
  ...
</tbody></table>
```
- Detection: a `table.Auswertungen` whose first two `<thead>` cells match `/^\s*from\s*$/i` and `/^\s*to\s*$/i`.
- `From`/`To` cells hold **full PlantSim object names** exactly as they appear in the resource-stats rows (minus the `.Models.Model.` prefix). Full names are globally unique → no `A1`/`A11` ambiguity.
- The table may be its own dropped file OR embedded in the stats file — the parser must handle both.

### `state.edges` (the in-memory contract)
After parsing, the loader populates:
```js
state.edges = [
  { fromRaw: "MI1_Milling_GearBoxA_Housing1",
    toRaw:   "Buffer_GearboxA_HousingPart_MI1_A1",
    isFeedback: false }   // isFeedback filled later by the layout pass; parser sets false
];
```
- `[]` when no edge table is found → triggers legacy (hardcoded) rendering.
- `fromRaw`/`toRaw` are the raw names; matching to nodes is done by exact string equality against `parsedData` keys / `station.idRaw`.

---

## Workstream 1 — SimTalk edge export (PlantSim side) · **independent files**
Deliverable: a verified `dumpEdges` SimTalk method + a sample fixture + docs. **Does not touch `gearboxsim.html`.**

1. `dumpEdges` walks `root.Model`; for each object with successors, append `(obj.name, obj.succ(i).name)` rows to an `EdgeTable` TableFile, then export it to HTML matching the shape above.
   - Ground every attribute in verified PlantSim patterns; explicitly flag `numSucc`/`succ(i)` as the two names to confirm via **What's This?**, and give the **Connector `.From`/`.To` enumeration** as the fallback.
   - Read-only: no flow logic, no exit controls, no `@.move`.
2. Produce `edges_sample.htm` — a realistic fixture (20–30 edges covering the current A/B topology incl. return loops) in the exact HTML shape, for the HTML agents to test against.
3. Write a short how-to into `plantsim-simtalk` skill (or `dynamic_topology_plan.md` §3) and a one-paragraph operator note: run after each sim, drop both files.

## Workstream 2 — Edge parser in `gearboxsim.html` · **parser region (~L2150–2340)**
Deliverable: populate `state.edges` from any dropped file. Additive, legacy-safe (changes no rendering).

1. Add `parseEdgeTable(doc)` near the existing `table.Auswertungen` scanning: detect the From/To table, read tbody rows → `state.edges` (per contract).
2. Strip any `.Models.Model.` prefix and trim, to match resource-row keys.
3. Multi-file: when a second file is dropped, merge its edges into `state.edges` (don't clobber stats).
4. Add a dev affordance: `console.info('[edges] parsed', state.edges.length)` and a count badge is optional.
5. Create/extend a tiny fixture if Workstream 1's `edges_sample.htm` isn't present yet, so this is self-testable.
6. **Do not** modify `renderMap`, `TOPOLOGY`, or the node-build pass.

## Workstream 3 — Dynamic build + auto-layout in `gearboxsim.html` · **topology + renderMap regions (~L1958–2090, ~L3158+)**
Deliverable: when `?dynamic=1` **and** `state.edges.length > 0`, build and render the graph from edges. Otherwise untouched legacy path.

1. `buildDynamicGraph()`:
   - **Nodes** = unique names from `state.stations` (entries>0 or buffer) ∪ all edge endpoints.
   - **Classify** by name + stats: Source (`/source|quelle/i` or ~100% blocked, 0 entries-in), Drain (`/drain|senke/i`), Buffer (`/buffer|puffer/i`), Measuring (`/ME\d|measuring/i`), else Station.
   - **Feedback edges:** topological sort; mark back-edges `isFeedback`, exclude from ranking (set on the `state.edges` objects).
   - **Column rank** = longest path from any Source over forward edges.
   - **Line** = `Gearbox([AB])` token (default both → 'AB'); Line A lanes on top, Line B below.
   - **Row** = within (column, line): stack nodes sorted by name, vertically centred. Reuse the existing column-centering / A–B gap-normalisation passes if they can operate on the computed positions; otherwise a clean equivalent.
2. Render through the **existing** D3 node/edge drawing by feeding it the computed `visibleStations` + computed edges (respect `isFeedback` styling already in the code).
3. Gate strictly: `const DYN = new URLSearchParams(location.search).has('dynamic') && state.edges.length > 0;` — legacy path 100% unchanged when false.
4. Build a **dev mock**: derive a `state.edges` array from the current `TOPOLOGY.edges` (map ids→idRaw) so the layout can be validated to reproduce a sane left→right A/B map before any real export exists.
5. Buffers render as normal nodes for v1 (collapsing Station→Buffer→Station into an edge ring is a later optimisation — out of scope).

---

## Integration & verification (owner: me, after agents return)
1. Merge Workstream 1 (docs/SimTalk) — no conflict.
2. Merge Workstream 2 (parser region) then Workstream 3 (topology+render region) — disjoint regions → clean merge; resolve if not.
3. Smoke test:
   - No flag, existing result file → map identical to today.
   - `?dynamic=1` + `edges_sample.htm` (or mock) → all nodes incl. new housing buffers placed, left→right, A/B stacked, return loops dashed, no console errors.
4. Keep `?dynamic=1` opt-in until validated against a real exported edge file; flip default later.

## Out of scope (this pass)
Buffer-as-edge-ring collapse; deleting `TOPOLOGY`; removing the 2026-06-19 dynamic-capture fallback; auto-detecting parallel groups for rightsizing (layout-only for now).
