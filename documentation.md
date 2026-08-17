# Simulation Configuration Reference — Gearbox Inc. Plant Simulation

> **VALIDATED 2026-06-14.** Finite caps on every buffer, zero deadlock, output at baseline (Drain_GearboxA ≈ 2,900+, Drain_GearboxB ≈ 3,450+). The deadlock-free design is **return buffers + grinder-gating** (not the output-buffer "Option B" that earlier drafts of this doc proposed). See *Buffer Configuration* below; full methodology in `methodology_feeder_gating.md`, code in `simtalk_codebase.md`.
>
> **Visualizer tool updated 2026-06-18.** The diagnostic rules in `gearboxsim.html` now reflect this topology exactly: ME upstream of part buffers, grinder-gating SimTalk pattern, Min. Contents exit strategy at all fan-out points. See `Handoff_18-06-26.md` for the current tool architecture.
>
> **Topology correction:** the measuring station **ME is UPSTREAM of the part buffers** — `G1/G2 → ME1 → (FlowControl) → Buffer_Gears/Buffer_Shafts → A4/A5/A6`. Earlier drafts had the buffers feeding ME; they do not. Confirmed from results: `ME1 entries 27,008 = G1 gears 12,138 + G2 shafts 14,869`.

**How to read:**

- **N/A** — single downstream connector; exit strategy setting has no effect
- **Min. Contents** — sends to the parallel station with the fewest MUs currently waiting; best for load-balancing identical stations
- **FlowControl object** — conditional split by MU **class** (the existing FlowControl routes measured gears/shafts to their buffers and finished gearboxes to the Drain via its own dialog rules — no PartType attribute, no SimTalk; leave it untouched)
- **exitLocked control** — a SimTalk entrance control toggles an upstream station's `exitLocked` to gate flow (used by grinder-gating, below)

---

## Line A

| Station | Successors (validated topology) | Exit Strategy | Reason |
| --- | --- | --- | --- |
| Src Hsg1 | → MI1 | N/A | Single path |
| Src Hsg2 | → MI2 | N/A | Single path |
| Src Gears | → T1 | N/A | Single path |
| Src Shafts | → T2 | N/A | Single path |
| **MI1** | → A1, A2, A3 | **Min. Contents** | 3 identical parallel pre-assembly stations; balances housing part 1 load (see MI sync note) |
| **MI2** | → A1, A2, A3 | **Min. Contents** | Same 3 stations; both housing parts fan out independently |
| **A1** | → D1, D2 | **Min. Contents** | 2 identical drilling stations |
| **A2** | → D1, D2 | **Min. Contents** | Same |
| **A3** | → D1, D2 | **Min. Contents** | Same |
| **D1** | → Buffer_GearboxA_Housing | N/A | Drilled housings buffer before final assembly |
| **D2** | → Buffer_GearboxA_Housing | N/A | Same |
| Buffer_GearboxA_Housing | → A4, A5, A6 | **Min. Contents** | Feeds drilled housings to the 3 assembly stations |
| T1 | → G1 | N/A | Single path |
| **G1** | → ME1 | N/A (gated) | Grinds gears, feeds ME1 directly. **exitLocked gated** on `Buffer_Gears_GearBoxA.full` (grinder-gating) |
| T2 | → G2 | N/A | Single path |
| **G2** | → ME1 | N/A (gated) | Grinds shafts, feeds ME1 directly. **exitLocked gated** on `Buffer_Shafts_GearBoxA.full` |
| **ME1** | → FlowControl → Buf Gears A, Buf Shafts A, Drain A | **FlowControl object** | Measures raw gears (10 min), raw shafts (5 min), and returning finished gearboxes (35 min). FlowControl routes by MU class: measured gear → Buf Gears A, measured shaft → Buf Shafts A, finished GbxA → Drain A |
| Buf Gears A | → A4, A5, A6 | **Min. Contents** | Measured gears fan out to the 3 assembly stations |
| Buf Shafts A | → A4, A5, A6 | **Min. Contents** | Measured shafts fan out to the 3 assembly stations |
| **A4** | → Buffer_Return_A | N/A | Assembles GbxA (gear+shaft+housing), discharges to the return buffer |
| **A5** | → Buffer_Return_A | N/A | Same |
| **A6** | → Buffer_Return_A | N/A | Same |
| **Buffer_Return_A** (cap 3) | → ME1 | N/A | Feedback: finished gearboxes queue here for final measuring at ME1. Cap = 3 stations → assembly can always discharge |

---

## Line B

| Station | Successors (validated topology) | Exit Strategy | Reason |
| --- | --- | --- | --- |
| Src Hsg B | → MI3 | N/A | Single path |
| Src Gears B | → T3 | N/A | Single path |
| Src Shafts B | → T4 | N/A | Single path |
| MI3 | → D3 | N/A | Single path |
| **D3** | → A7, A8, A9, A10, A11 | **Min. Contents** | 5 identical assembly stations; **root cause of A10/A11 = 0% working** — any strategy other than Min. Contents (e.g. default "Start at successor 1") always tries A7 first and starves A10/A11 |
| T3 | → G3 | N/A | Single path |
| **G3** | → ME2 | N/A (gated) | Grinds gears B, feeds ME2. **exitLocked gated** on `Buffer_Gears_GearBoxB.full` |
| T4 | → G4 | N/A | Single path |
| **G4** | → ME2 | N/A (gated) | Grinds shafts B, feeds ME2. **exitLocked gated** on `Buffer_Shafts_GearBoxB.full` |
| **ME2** | → FlowControl → Buf Gears B, Buf Shafts B, Drain B | **FlowControl object** | Same pattern as ME1: routes measured gears/shafts to their buffers, finished GbxB → Drain B (final measure 10 min) |
| Buf Gears B | → A7–A11 | **Min. Contents** | Measured gears fan out to the 5 assembly stations |
| Buf Shafts B | → A7–A11 | **Min. Contents** | Measured shafts fan out to the 5 assembly stations |
| A7–A11 | → Buffer_Return_B | N/A | Assemble GbxB, discharge to the return buffer |
| **Buffer_Return_B** (cap 5) | → ME2 | N/A | Feedback for final measuring. Cap = 5 stations → assembly can always discharge |

---

## Post-Optimization Changes (V2 — ME1b / ME2b + Line B assembly reduction)

Two independent optimizations bring V2 to **362 pcs/mo Line A / 382 pcs/mo Line B** (V1 baseline: 212 / 252) and cut mean lead time by 70–83%:

### 1. Add ME1b / ME2b as parallel measuring stations

When ME1b and ME2b are added, the *return buffer* gains a second successor and its exit strategy must be set. The grinder-gating control needs **no change** — the gates live on G1–G4 and govern any number of MEs unchanged.

| Station | Successors (optimized) | Exit Strategy | Change from baseline |
| --- | --- | --- | --- |
| **Buffer_Return_A** | → ME1, ME1b | **Min. Contents** | Was single path to ME1; now 2 parallel measuring stations pull finished gearboxes |
| **Buffer_Return_B** | → ME2, ME2b | **Min. Contents** | Same for Line B |
| **G1 / G2** | → ME1, ME1b | **Min. Contents** | Raw gears/shafts now feed either ME; gating unchanged |
| **G3 / G4** | → ME2, ME2b | **Min. Contents** | Same for Line B |

> One subtlety with parallel MEs: two MEs pulling in the same event instant could momentarily overfill a part buffer by one. The `.full` gate self-corrects on the next event (the second ME blocks briefly); if you want zero overshoot, gate one step earlier with `numMU >= capacity - 1`.

### 2. Reduce Line B assembly from 5 stations to 1

With ME2/ME2b resolving the bottleneck, the five Line B assembly stations (A7–A11) become heavily underutilized — validated by the GearboxSim rightsizing rule and confirmed in simulation: **A8–A11 can be removed**, leaving A7 as the sole assembly station at the target throughput of 300 pcs/mo.

| What to change | V1 (baseline) | V2 (optimized) |
| --- | --- | --- |
| D3 successors | A7, A8, A9, A10, A11 | **A7 only** |
| Buf Gears B successors | A7–A11 | **A7 only** |
| Buf Shafts B successors | A7–A11 | **A7 only** |
| Buffer_Return_B cap | 5 (= #stations) | **1** (= #stations) |

Exit strategy on D3 / Buf Gears B / Buf Shafts B becomes **N/A** (single path) once A8–A11 are removed.

### V2 KPI summary (confirmed from simulation runs, 2026-06-18)

| KPI | V1 Line A | V1 Line B | V2 Line A | V2 Line B |
| --- | --- | --- | --- | --- |
| Monthly output | 212 pcs | 252 pcs | **362 pcs** | **382 pcs** |
| Mean lead time | 12h 46min | 14h 50min | **3h 48min** | **2h 29min** |
| Little's Law WIP | ~3.7 gearboxes | ~5.1 gearboxes | **~1.9** | **~1.3** |
| System MU | 21 MU | 13 MU | **10 MU** | **12 MU** |

---

## Implementation Notes

### Grinder-gating — the deadlock-free control

ME can only block on output if it just measured a gear and `Buffer_Gears` is full, or a shaft and `Buffer_Shafts` is full (finished gearboxes go to the always-accepting Drain). The control **locks the grinder feeding ME a part type whenever that type's buffer is full**, so ME stays free to measure the other type and returning gearboxes — it can never block.

`updateGatesA` (assigned to the **Entrance** of `Buffer_Gears_GearBoxA`, `Buffer_Shafts_GearBoxA`, and `A4/A5/A6`, "Front" unchecked):

```simtalk
if Buffer_Gears_GearBoxA.full
    G1_Grinding_GearBoxA_Gears.exitLocked := true
else
    G1_Grinding_GearBoxA_Gears.exitLocked := false
end
if Buffer_Shafts_GearBoxA.full
    G2_Grinding_GearBoxA_Shafts.exitLocked := true
else
    G2_Grinding_GearBoxA_Shafts.exitLocked := false
end
```

`updateGatesB` mirrors it (G3/G4, `...GearBoxB`, A7–A11). `init`/`reset` unlock G1–G4 so a Reset starts clean. Full code and rationale: `simtalk_codebase.md`. No counters, no MU attribute — the control reads live buffer state and is idempotent, so it cannot drift into a latch.

### ME1 / ME2 — FlowControl object (existing, leave untouched)

ME receives three MU kinds: raw gears (10 min), raw shafts (5 min for ME1), and returning finished gearboxes (35 min ME1 / 10 min ME2). A **FlowControl object** immediately downstream of ME splits the output:

- finished gearbox → `Drain_GearboxA` / `Drain_GearboxB`
- measured gear → `Buffer_Gears_GearBox*`
- measured shaft → `Buffer_Shafts_GearBox*`

It routes by **MU class** via its own dialog rules — there is **no `PartType` attribute** in this model (the User-defined tab is empty), and none is needed. The grinder-gating control does not touch the FlowControl. *(Earlier drafts of this doc instructed setting `PartType = "SubComponent"/"FinalGearbox"` at sources/assembly — that was incorrect and is not required.)*

### MI1 / MI2 synchronization caveat

Both MI1 and MI2 fan out to the same 3 pre-assembly stations (A1, A2, A3). If the model requires Housing Part 1 and Part 2 to meet at the **same** pre-assembly station (AND-join), Min. Contents alone is insufficient — it routes each part to whichever queue is shortest at that moment, which may differ.

Solutions:

- Set both MI1 and MI2 to **Cyclic** with a shared counter so the two parts always alternate to the same station in sync
- OR a **SimTalk method** that records where Part1 went and forces Part2 to follow
- OR model both housing parts as one combined MU, eliminating the AND-join

If MI1/MI2 are simply two parallel machines making the same part type (not two halves of one part), Min. Contents is correct and no sync is needed.

---

## Summary: Nodes Where Exit Strategy Matters

| Fan-out point | Strategy |
| --- | --- |
| MI1 → A1/A2/A3 | Min. Contents |
| MI2 → A1/A2/A3 | Min. Contents |
| A1/A2/A3 → D1/D2 | Min. Contents |
| Buffer_GearboxA_Housing → A4/A5/A6 | Min. Contents |
| **ME1 → FlowControl → Buf Gears A / Buf Shafts A / Drain_A** | **FlowControl object** (by MU class) |
| Buf Gears A / Buf Shafts A → A4/A5/A6 | Min. Contents |
| **D3 → A7–A11** | **Min. Contents** ← fix this first |
| **ME2 → FlowControl → Buf Gears B / Buf Shafts B / Drain_B** | **FlowControl object** (by MU class) |
| Buf Gears B / Buf Shafts B → A7–A11 | Min. Contents |
| Buffer_Return_A → ME1/ME1b *(post-opt)* | Min. Contents |
| Buffer_Return_B → ME2/ME2b *(post-opt)* | Min. Contents |

**Gated single paths:** G1/G2 → ME1 and G3/G4 → ME2 each have a single connector, but carry the grinder-gating `exitLocked` control. All other nodes have a single downstream connector — exit strategy is irrelevant.

---

## Buffer Configuration

### How MaxContent works in Plant Simulation

`MaxContent = N` on a Buffer creates a **hard upstream block**: when the buffer holds N MUs, the upstream machine finishes processing, holds its output part, and **freezes** — it cannot start the next part or accept inputs until a slot opens. It is a hard lock, not a soft throttle. `MaxContent = −1` means unlimited.

### Why naive caps deadlocked — and how the validated design avoids it

ME has a **feedback loop**: assembly pushes finished gearboxes back into ME for final measuring. Capping the buffers without protection caused a circular/starvation deadlock — **confirmed in `results_2020_11-06-26.htm`** (ME1 blocked 99.99%, assembly 99.96%, only 11 GbxA / 39 GbxB in 366 days) and again in `results_2020_14-06-26_17-09.htm` (a misplaced control latched the part buffers: 12,138 gears in / 0 out, Drains = 0).

The **validated** fix has two independent parts (no source-rate changes, no machine split):

1. **Return buffers** — `Buffer_Return_A` (cap 3 = #stations on A), `Buffer_Return_B` (cap 5 = #stations on B) between assembly and ME. Pure topology, no code. Guarantees assembly can always discharge → kills circular blocking.
2. **Grinder-gating** — lock G1–G4 on the matching part buffer's `.full` (see Implementation Notes). Guarantees ME never blocks on output → kills part-type starvation.

> The earlier "Option B" output buffer `Buf_Measured_A/B` after ME is **no longer used** — superseded by the two mechanisms above.

### Queue vs Stack

Use **Queue (FIFO)** on all buffers: it matches conveyor/tray reality (first in, first used), prevents parts aging at the bottom of a LIFO stack, and keeps WIP-age uniform for lead-time analysis. Stack would only suit overhead gravity-fed bins (not applicable here).

### Validated caps (Step 9 — applied to the live model)

> These values ran deadlock-free at baseline output. The original sizing was estimated with a gap formula `MaxContent = ⌈T_gap ÷ T_upstream_eff⌉ + 1`; note that formula predates the topology correction (it assumed the buffer fed ME). The values below are the **empirically validated** caps, and with grinder-gating in place the cap also bounds how many measured parts may wait before its grinder is locked.

| Buffer | Position (validated) | MaxContent | Mode | Role |
| --- | --- | --- | --- | --- |
| Buffer_Gears_GearBoxA | ME1 → A4/A5/A6 | **5** | Queue | Holds measured gears waiting for assembly; full → G1 locked |
| Buffer_Shafts_GearBoxA | ME1 → A4/A5/A6 | **8** | Queue | Holds measured shafts; full → G2 locked |
| Buffer_GearboxA_Housing | D1/D2 → A4/A5/A6 | **6** | Queue | Decouples drilling from assembly (≈ 2 per station) |
| Buffer_Gears_GearBoxB | ME2 → A7–A11 | **3** | Queue | Smaller — ME2 final measure is only 10 min vs ME1's 35 |
| Buffer_Shafts_GearBoxB | ME2 → A7–A11 | **6** | Queue | Holds measured shafts B; full → G4 locked |
| Buffer_Return_A | A4/A5/A6 → ME1 | **3** | Queue | = #stations on A; lets assembly always discharge |
| Buffer_Return_B | A7–A11 → ME2 | **5** (baseline) / **1** (V2, after removing A8–A11) | Queue | = #stations on B; reduce when assembly stations are removed |

**Rollback:** set the 5 part buffers back to `MaxContent = −1` to return to the unlimited-WIP baseline; the gating goes inert (`full` never true) and output is unchanged.

---

## Buffer Sizing Experiment & The One-Bottleneck Principle (2026-06-19)

### Experiment: validated caps vs. caps = 100

Setting all buffer capacities to 100 (effectively unlimited) produced a significant throughput jump:

| KPI | V2 (validated caps 5–8) | V2 (caps = 100) | Δ |
|---|---|---|---|
| Gearbox A pcs/mo | 362 | **420** | +16% |
| Gearbox B pcs/mo | 382 | **428** | +12% |
| ME1 effective utilisation | ~79% | **~93%** | +14 pp |
| Lead Time A | 3h 48min | 3h 33min | −6% |

The validated caps of 5–8 were sized to prevent deadlock and bound WIP — but they were **suppressing throughput** by creating artificial starvation at ME1. With caps at 100, the gear/shaft buffers fill up (Buf Shafts hit 100/100), parts are always available, and ME1 runs nearly flat out.

### The One-Bottleneck Principle

> **In a correctly buffered production system, there is always exactly one active bottleneck — the station with the highest effective utilisation. If no clear bottleneck exists, buffers are likely undersized and are suppressing throughput by starving the true constraint.**

Consequences:
- **Undersized buffers → hidden bottleneck.** Buffers at 5/8 caused starvation at ME1 (6.7% waiting). ME1 appeared comfortable at 79% u_eff, but was being artificially throttled by empty buffers.
- **Correctly sized buffers → bottleneck reveals itself.** At caps = 100, ME1 jumps to 93% u_eff and becomes the visible, single constraint. Throughput rises until ME1 saturates.
- **The bottleneck governs system output.** To increase throughput further, address the bottleneck directly (parallel capacity, faster process, reduced MTTR) — adding more buffer elsewhere does nothing.
- **One constraint at a time.** Once ME1 is addressed, the next bottleneck surfaces. Theory of Constraints (Goldratt): exploit the constraint first, then elevate it.

### New finding: missing Buffer_GearboxB_Housing (Line B)

With caps = 100, Line B revealed a structural gap: **D3_Drilling_GearBoxB is blocked 26.7%** because it feeds ME2/ME4 directly with no housing buffer. Comparison:

| | Line A | Line B |
|---|---|---|
| Housing buffer before ME | `Buffer_GearboxA_Housing` (exists, cap 6) | **missing** |
| Drilling station blocking | D1: **1.4%** | D3: **26.7%** |
| ME effective utilisation | ME1: **93%** | ME2: **86%** |

**Fix:** Add `Buffer_GearboxB_Housing` between D3 and ME2/ME4 (same pattern as Line A). Sizing: cap ≈ 5–10 is sufficient; the buffer only needs to absorb the time ME2 is busy with gears/shafts when D3 finishes a housing part.

This will reduce D3 blocking, free MI3 (which cascades from D3), and allow ME2 to approach ME1's utilisation level — further increasing Line B throughput.

---

## True-Bottleneck Run & Parallel-Pool Reading (2026-06-19)

### Method: strip variability to expose the raw constraint

To find the *true* constraint independent of failures and shift gaps, run with **Availability = 100% (no failures) and the shift calendar disabled (24/7)**. With variability removed, the binding resource shows as **100% working, ~0% waiting, ~0% blocked** — pure cycle-time saturation, nothing masking it. Results saved in `results_2020_19-06-26_13-57_buffer-test.htm`.

> Visualizer caveat: in a no-shift run the GearboxSim `Util%` reads >100% (e.g. ME2 = 209.6%) because it normalises against *planned shift time*, which no longer exists. The ratio is constant (100.00 / 209.6 = 0.477 = the old planned fraction). **For a no-shift run, trust the HTM `working` column, not the visualizer Util%.**

### Finding: read parallel stations as a pooled resource, not per-station

Every near-100% station in the run has **idle parallel partners** sitting next to it. The individual saturation is a *load-balancing artefact*, not a true cap. Read the pool:

| Resource (pool) | Primary | Idle partner(s) | True pooled load |
|---|---|---|---|
| Line A measuring (ME1+ME3) | ME1 **99.14%** | ME3 83.37% | **~91%** |
| Line A pre-assembly (A1/A2/A3) | A1 **95.39%** (blocked 4.6%) | A2, A3 = **0%** | **~32%** if balanced |
| Line A final assembly (A4/A5/A6) | A6 **94.81%** (waiting 5.2%) | A4, A5 = **0%** | **~32%** if balanced |
| Line B measuring (ME2+ME4) | ME2 **100.00%** | ME4 83.38% | **~92%** |
| Line B assembly (A7 only) | A7 **99.97%** | A8–A11 disconnected | **A7 alone — no relief** |

### The One-Bottleneck Principle, refined

The earlier principle holds **per serial line**, with two corrections proven by this run:

1. **"Resource" ≠ "station" when machines are parallel — pool the twins.** ME1 at 99% looks like a hard cap, but the ME1+ME3 *pool* is only 91%; ME3 sits idle. The model's Min. Contents fan-outs are **not actually load-balancing** — the primary station (ME1, A1, A6) wins every routing decision while its partners idle at 0–83%. This is the highest-leverage fix: balance the pools and ME1 drops 99→91%, A1/A6 drop 95→~32%.

2. **A station is only the *cause* if it has no waiting and no blocking.** A station that is **blocked** (A1, 4.6%) is held back by something downstream; a station that is **waiting** (A6, 5.2%) is starved by something upstream. Both are *symptoms* pointing at the real constraint (the measuring pool), not independent bottlenecks.

**Result: each line still has exactly one true constraint.** The factory shows "multiple bottlenecks" only because it has **two independent lines** (one constraint each) — not because a single line has several.

- **Line A** true constraint: the **ME1/ME3 measuring pool (~91%)**.
- **Line B** true constraint: **A7 (~100%, alone)**.

### A7 is a self-inflicted constraint — the V2 reduction overshot

Reducing Line B assembly to A7-only (see *Post-Optimization Changes*) left **A7 pinned at 99.97% with A8–A11 idle beside it**. A single 30-min assembly station cannot keep up with the ME2/ME4 measuring pool.

**Buffers will NOT fix A7.** A7 is at 100% *working*, 0% *waiting* — it is never starved, so a buffer feeding it has nothing to convert into throughput. (General rule: a buffer only helps a station that idles on *waiting* or *blocked*; it cannot add capacity to a station that is already busy 100% of the time.)

**Fix = capacity, not buffer:** re-activate one Line B assembly station (A8). That halves A7's load to ~50%, after which the **ME2/ME4 pool (~92%) becomes Line B's true ceiling** — the constraint hand-off you want. This corrects the V2 "reduce to 1 station" recommendation: the right number for Line B assembly is **2**, not 1.

---

## GearboxSim Visualizer — Integration & Scaling Notes (2026-06-19)

> Captured after the Siemens Advanta interest in the tool. Scope question: could the visualizer live *inside* Plant Simulation, and would it scale to **very large models (~1000 machines)**? Summary: the diagnostic *value* scales superbly; the current SVG implementation does not.

### How Plant Simulation is built (integration surfaces)

| Layer | What it is |
|---|---|
| **Core engine** | Native **C++** discrete-event kernel (lineage SiMPLE++ → eM-Plant → Siemens DI Software). Windows desktop app. |
| **Scripting** | **SimTalk 2.0** — built-in, interpreted, OO. Reads/writes every attribute, table, statistic live. |
| **3D / 2D** | OpenGL 3D view; classic 2D frame editor. |
| **Automation** | **COM / ActiveX** interface — drive it from .NET, C++, Python (`win32com`), Excel VBA; run models, push/pull tables, read results. |
| **Native hooks** | SimTalk can call external **C/C++ DLLs**; models can embed **ActiveX** controls in dialogs; HTML statistics report (what the tool already consumes). |

There is **no public rich UI-plugin SDK** for first-class dockable panels. You extend PlantSim via SimTalk, COM automation, embedded ActiveX/HTML controls, or external DLLs — not by writing a native tool window.

### Integration options, ranked by effort

1. **Auto-export + auto-launch (easy, days).** SimTalk method exports stats HTML + edge table after a run and opens the visualizer (auto-loads them). Reuses 100% of current code, zero rendering risk. The pragmatic 90% solution.
2. **Companion app via COM (medium, 1–3 weeks).** Electron/.NET app reads the live model + results over COM and renders the D3 map beside PlantSim.
3. **Embedded panel inside PlantSim (medium–hard, version-dependent).** Host the HTML/JS in a PlantSim web/ActiveX control, fed by SimTalk. **Pivotal risk:** the embedded browser engine. Legacy PlantSim used the **Internet Explorer/Trident** engine — modern D3 + ES6 will not run on it. Verify whether 2404 ships a **Chromium/WebView2** control *before* promising this.
4. **Truly native, shipped-in-product panel (hard — not a student task).** Owned by the **Plant Simulation product team at Siemens DI Software**, a different org from Advanta. Roadmap/partnership decision, not a bolt-on.

> Strategic read: Advanta can adopt the tool internally via options 1–2 immediately; getting it *into the shipped product* is not Advanta's to give.

### Scaling to ~1000 machines

**The value scales — the current build does not.** Reading tables is O(n) tedium (1000 rows = 20× the pain); the ranked-constraint view is ~O(1) cognitive load ("show me the 5 red nodes among 1000"). The bigger the model, the more valuable the tool — *if* it renders.

Where the current implementation breaks, by layer:

| Layer | ~50 machines | ~1000 machines |
|---|---|---|
| **Parsing** | instant | Multi-MB HTML parses OK (~1s), **but** O(n²) hotspots (`TOPOLOGY.nodes.forEach(node => pdKeys.find(...))`, edge→node resolution by linear `.find()`) become millions of compares. Fixable with `Map` indexing. |
| **Layout** | hand-tuned columns | Topo-rank is O(V+E) — fine. A *flat* 1000-node graph is hundreds of columns wide — needs clustering. |
| **Rendering ⚠️** | crisp SVG | **The ceiling.** D3 draws **SVG** (one DOM node per shape). 1000 nodes + ~2–5k edges + glassmorphism blur filters ≈ 15–30k DOM elements. SVG degrades past ~1–5k; blur filters and per-node animations are especially costly → multi-second renders, janky pan/zoom/hover. Effectively unusable. |
| **Station table** | 45 rows | 1000 rows need virtualized (windowed) scrolling. |

What scaling to 1000+ actually requires (a focused project, weeks–months — not a tweak):

1. **Swap SVG → Canvas/WebGL** (PixiJS / regl / sigma.js / Cytoscape WebGL). Canvas handles 10k+, WebGL 100k+. **Single most important change**; the blur/glass effects must go or be faked cheaply.
2. **Mirror PlantSim's Frame hierarchy + semantic zoom.** Real large models are nested **Frames**, not flat. Show top-level frames, drill down on zoom. Natural answer to scale; matches how engineers already think. (Edge export must capture hierarchy.)
3. **Index the parser** (`Map` lookups, not linear `.find()`); consider structured export or live COM read instead of parsing huge HTML.
4. **Lead with the diagnostic view, not the full map** — top-N constraints, filter-by-role, search. The ranked-constraint panel becomes the primary surface; the map is drill-down.
5. **Virtualize the station table.**

**Thesis-scope framing:** *"Scaling an interactive factory-simulation analysis layer to 1000+ machines via WebGL rendering and hierarchical semantic zoom."* Bounded, technically meaty, immediately useful — and it forces an answer to the WebView2-vs-IE question.
