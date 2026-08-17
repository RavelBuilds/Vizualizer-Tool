# Winning Strategy – Gearbox Inc. Factory Optimization
## Siemens Advanta & HS München | Plant Simulation Praktikum 2026

---

## 1. Mission & Success Criteria

**Client:** Gearbox Inc. (Californian start-up, brownfield site in Bavaria)
**Goal:** Deliver a Plant Simulation model + factory layout that achieves and exceeds the monthly output targets while meeting all boundary conditions.

**Graded deliverables (June 22, 2026 steering committee):**
- **16-minute pitch** to steering committee
- Mandatory: 3 filled-in one-pager PPTX templates (Simulation results, Layout, Key Findings) — additional slides allowed and recommended
- Working Plant Simulation model

**Hard targets to beat:**
| KPI | Target |
|-----|--------|
| Gearbox A output | ≥ 300 units / month |
| Gearbox B output | ≥ 300 units / month |
| Station count | Pre-defined (27 total; can justify changes) |
| Machine availability | 80% per station, MTTR 5 min |
| Shift model | 2 shifts × 8 h, 5 days/week, 250 days/year |

---

## 2. How Plant Simulation Works (Relevant Mechanics)

Tecnomatix Plant Simulation is a **discrete-event simulation** tool. Each "Mobile Unit" (MU – a part, housing, etc.) moves between objects that represent machines, buffers, and transports. Time advances only when events happen (part arrival, processing end, failure, etc.).

### Objects used in this model
| Object | What it does |
|--------|-------------|
| `Source` | Generates MUs at a defined rate or on demand |
| `SingleProc` | One-at-a-time machine; set cycle time + availability + failures |
| `Buffer` | Queue between stations; set `MaxContent` to cap WIP |
| `Drain` | Counts finished goods leaving the system |
| `EventController` | Master clock; "Stop at" defines simulation horizon |
| `ShiftCalendar` | Defines when machines are operational |
| `DataTable` | Maps part-type → process time for shared machines |

### Key configuration parameters per station
- **Processing time:** set in the `Times` tab (can reference a DataTable for multi-product)
- **Availability:** set in the `Failures` tab → `Availability = 80%`, `MTTR = 5 min`
- **Exit strategy:** set in `Exit` tab → **Min. Contents** is the key lever for load-balancing parallel stations
- **Bottleneck Analyzer:** Tools → Bottleneck Analyzer → highlights the station with highest sustained utilization

### Interpreting statistics output
The HTML report columns mean:
- **Working %** – fraction of total simulation time the machine was actually processing
- **Waiting %** – fraction idle with an empty input buffer (starved)
- **Blocked %** – fraction idle with a full output buffer (downstream congested)
- **Failures %** – fraction in repair (from the 80% availability setting)
- **Unplanned %** – time outside the shift calendar (≈52.3% for this 2-shift/5-day model)

Effective utilization within uptime = `Working / (1 – Unplanned – Failures)`.

---

## 3. Current Validated Baseline (model: 14-06-2026-17-44-PlantSim.spp)

> **This is the deadlock-free baseline** — buffer caps + return buffers + grinder-gating SimTalk in place (validated 2026-06-14). Not the original unlimited-WIP run. The GearboxSim Visualizer KPI pill is the authoritative source for these numbers.

### Annual throughput
| Line | Monthly output | Target | Gap | Lead Time |
|------|---------------|--------|-----|-----------|
| Gearbox A | **242 pcs/mo** | 300 | **−19%** ±3.6% | 11h 15min |
| Gearbox B | **287 pcs/mo** | 300 | **−4%** ±3.3% | 12h 59min |

### System-level WIP
| Metric | Total | Line A | Line B |
|--------|-------|--------|--------|
| Avg. WIP | 36 MUs | 18 | 18 |
| Max WIP | 61 MUs | 31 | 30 |

### Bottlenecks identified by GearboxSim Visualizer
- **ME1 (79% effective utilization)** — primary bottleneck for Line A
- **ME2 (79% effective utilization)** — latent bottleneck for Line B; currently 4% below target

### Why the output is below target

**Problem 1 – ME1 is the bottleneck for Gearbox A (−19%)**

ME1 handles gears (10 min), shafts (5 min), AND final measuring of finished gearboxes (35 min) — it is triple-loaded. At 79% effective utilization it is the rate-limiting step for the entire Line A. The cascade: ME1 saturated → assembly A4/A5/A6 cannot discharge → they block → housing sub-line backs up.

Required annual capacity for 300/mo: 3,600 gearboxes/year.
Single ME1 at 80% availability: `(250 × 2 × 480 × 0.80) / 35 = ~3,420/year` — already below target, and queuing effects amplify the loss to −19%.

**Problem 2 – Assembly exit strategy unbalanced (both lines)**

Some assembly stations show near-0% working with high blocked %. The upstream fan-out (D1/D2 → housing buffer → assembly, ME1 → Buf Gears/Shafts → assembly) is not distributing parts evenly. Min. Contents is either not configured or overridden.

**Problem 3 – ME2 at 79% utilization is a latent risk for Line B**

B is only 4% below target. But ME2 at 79% effective utilization has very little headroom — a ±20% process time variation or any availability dip will push B below 300/mo.

**Problem 4 – Lead times are long relative to processing time**

Line A: 11h 15min lead time vs. ~2h actual processing time per gearbox → roughly 5× buffering overhead. This means WIP is queueing heavily in front of ME1. Line B at 12h 59min shows similar queuing even though output is closer to target.

---

## 4. Optimization Steps – Ordered by Impact

### Step 1 (Critical): Add ME1b — parallel measuring station for Line A

Add `ME1b_Measuring_GearBoxA` as a second measuring station identical to ME1.
- Copy exact settings: 35 min (final), 10 min (gear), 5 min (shaft), 80% availability, 5 min MTTR
- Connect Buffer_Return_A → ME1 and ME1b (Min. Contents)
- Connect Buf_Gears_A and Buf_Shafts_A → ME1 and ME1b (Min. Contents)
- Expected: ME1 load drops from 79% to ~40% → assembly unblocks → Line A output reaches 330–350/mo

**In Plant Simulation:**
1. Duplicate ME1, rename `ME1b_Measuring_GearBoxA`
2. Connect Buffer_Return_A to ME1b (set exit strategy to Min. Contents)
3. Connect Buf_Gears_A and Buf_Shafts_A to ME1b (Min. Contents)
4. Connect ME1b output to Drain_GearboxA via FlowControl (same rules as ME1)
5. Update grinder-gating control: `updateGatesA` already handles any number of MEs — no change needed

### Step 2 (High): Fix assembly exit strategies (A4/A5/A6 and A7–A11)

Confirm Min. Contents is set on every fan-out point feeding assembly:
- D1/D2 → Buffer_GearboxA_Housing → A4/A5/A6: Min. Contents on buffer exit
- Buf_Gears_A / Buf_Shafts_A → A4/A5/A6: Min. Contents
- D3 → A7–A11: Min. Contents ← most critical, likely root cause of A10/A11 = 0%
- Buf_Gears_B / Buf_Shafts_B → A7–A11: Min. Contents

Expected: assembly utilization balances across all parallel stations, blocked time drops from 20–38% to <5%.

### Step 3 (Medium): Add ME2b — parallel measuring for Line B

Line B is 4% below target and ME2 is at 79% utilization. Add ME2b as a precaution using the same pattern as ME1b.
- ME2 final measuring is only 10 min (vs ME1's 35 min), so ME2b has more headroom
- Still worthwhile: makes B robust against process time variation and any future demand increase

### Step 4 (Medium): Run sensitivity scenarios for the pitch

These are not mandatory but differentiate a strong presentation:

**Q1 – Availability sensitivity:** Run at 60%, 70%, 80%, 90% and plot output vs. availability. Identifies the maintenance floor below which output drops under 300/mo.

**Q2 – ±20% process time fluctuation:** `ProcessTime × (0.8 + RANDOM() × 0.4)`, 10 replications. Report mean ± CI. Validates robustness of the optimized design.

**Q3 – 1-shift impact:** Change ShiftCalendar to 1 shift × 8 h. Output roughly halves — gives the client the cost of single-shift operation.

**Q4 – Lead time improvement:** Show avg. lead time before (11h 15min / 12h 59min) vs. after ME1b + routing fix. Expected 30–40% reduction as ME1 queue dissolves.

### Step 5 (Low): Verify source rates

All sources should use the shift calendar and produce at a rate that matches downstream demand. Unlimited sources create artificial WIP spikes at startup. Recommended:
- `Source.InterArrivalTime = 00:10:00` for housing sources (matches MI1/MI2 at 10 min each)

---

## 5. Expected Post-Optimization KPIs

| KPI | As-Is (validated) | After ME1b + routing fix | Target |
|-----|-------------------|--------------------------|--------|
| Gearbox A monthly output | 242 pcs/mo | **~330–350 pcs/mo** | 300 ✓ |
| Gearbox B monthly output | 287 pcs/mo | **~305–315 pcs/mo** | 300 ✓ |
| Lead Time A | 11h 15min | ~7h (−38%) | — |
| Lead Time B | 12h 59min | ~9h (−30%) | — |
| ME1 effective utilization | 79% | ~40% | — |
| ME2 effective utilization | 79% | ~40% (with ME2b) | — |
| Max WIP (Line A) | 31 MUs | ~18 MUs (−42%) | — |
| Max WIP (Line B) | 30 MUs | ~18 MUs (−40%) | — |
| Assembly blocking (Line A) | 20–38% | <5% | — |
| Dead stations (A5/A10/A11) | 0% working | balanced ~10–15% | — |

---

## 6. Factory Layout Strategy

### Recommended production structure: Variants – Use of Synergies

Given two product lines (A and B) with many similar process types (milling, turning, grinding, assembly, measuring), the **hybrid "Variants + Synergies"** layout is optimal:
- Shared turning cells (T1/T2 for A gears+shafts, T3/T4 for B)
- Shared grinding cells per line
- Dedicated assembly lines per product (prevents cross-contamination and simplifies routing)
- Shared measuring zone (ME1/ME1b for A, ME2/ME2b for B) placed at end of each line

### Layout principles
1. **Material flow direction:** left-to-right (incoming goods left → machining → assembly → measuring → shipping right)
2. **Product-based zones** for assembly (A-line: A1–A6, B-line: A7–A11) to minimize cross-traffic
3. **Functional clustering** for machining (all milling together, all turning together) when process synergies exist
4. **Measuring stations isolated** from vibration-generating machining (quality requirement from briefing)

### Storage area placement
- **Incoming goods (162 m²):** Located at building entrance (left/front gate), rectangular block, connected to production start
- **Shipping (90 m²):** Located at opposite end (right/rear gate), rectangular, connected to Drain areas
- Both storage areas must be connected as a rectangular block (no L-shapes)
- Minimum 1 m clearance maintained on all machine sides

### Expandability for Gearbox C
- Reserve a dedicated zone (e.g., right section of building or a dedicated bay) for future C-line
- Route utilities (power, compressed air) to reserved zone during initial build-out
- Design the shared machining zone with modular spacing so additional stations can be inserted

### Layout pros/cons
| Pros | Cons |
|------|------|
| Short transport paths between sequential steps | Requires clear aisle management between A and B zones |
| Measuring stations isolated from machining vibration | Shared machining zone requires careful scheduling |
| Expandable for Gearbox C without disrupting A/B | More complex routing logic for shared machines |
| Logical incoming → production → shipping flow | Larger footprint than pure product-based layout |

---

## 7. Optional Analyses (Extra Points)

**Q1 – Availability sensitivity threshold:**
Run scenarios with availability at 60%, 70%, 80%, 90%. Plot monthly output vs. availability. The threshold where output drops below 300 identifies the critical maintenance floor.

**Q2 – ±20% process time fluctuation:**
Modify all `SingleProc` times to `ProcessTime × (0.8 + RANDOM() × 0.4)` using SimTalk. Run 10 replications. Report mean output and confidence interval.

**Q3 – 1-shift impact:**
Change ShiftCalendar to 1 shift × 8 h/day. Expected output roughly halves. Quantifies the cost of single-shift operation.

**Q4 – WIP reduction levers:**
Before/after comparison of Max WIP and Avg. Lead Time after ME1b + buffer cap changes. Already measurable: as-is Max WIP 31/30 MUs per line.

**Q5 – Digitalization potentials:**
- Automated AGV transport (remove manual transport time)
- Real-time OEE monitoring at each station
- Predictive maintenance to reduce unplanned downtime
- MES integration for A/B sequencing in mixed-volume months

---

## 8. Simulation Build Checklist

**Model setup:**
- [ ] All 27 stations placed and named per PDF (MI1-3, T1-4, G1-4, D1-3, A1-11, ME1-2)
- [ ] Process times set per station (see section 10)
- [ ] Availability 80% + MTTR 5 min set on every `SingleProc` via Failures tab
- [ ] ShiftCalendar: 2 × 8h shifts, Mon–Fri, 250 days/year
- [ ] Sources: one per raw-material stream per line
- [ ] Drain: one per finished product line
- [ ] Min. Contents on all parallel-station forks
- [ ] Buffer caps set (see `documentation.md` Buffer Configuration table)
- [ ] Return buffers: Buffer_Return_A (cap 3), Buffer_Return_B (cap 3)
- [ ] Grinder-gating SimTalk on entrance of Buf_Gears and Buf_Shafts (see `simtalk_codebase.md`)
- [ ] Bottleneck Analyzer enabled during run

**After each scenario run — validate with GearboxSim Visualizer:**
- [ ] Export: *Results → Statistics → Save as HTML*
- [ ] Drag into `gearboxsim.html` → verify KPI pill shows correct model name and simulated days
- [ ] Gearbox A and B pcs/month vs. 300 target (green = above target)
- [ ] ME1/ME2 should show "Healthy" (not "Bottleneck") after adding ME1b/ME2b
- [ ] Assembly balance: check all stations have similar utilization
- [ ] Max WIP decreases vs. as-is baseline (31/30 MUs)
- [ ] Lead Time decreases vs. as-is (11h 15min / 12h 59min)
- [ ] Screenshot the map + KPI pill for the PPTX slides

---

## 9. Pitch Structure for Steering Committee (16 minutes)

**Format:** 3 mandatory PPTX template slides + additional slides. Aim for ~10 slides total, 1–1.5 min each.

| # | Slide | Content | Time |
|---|-------|---------|------|
| 0 | **Title / Context** | Team name, Gearbox Inc. situation, mission statement | 1 min |
| 1 | **As-Is: GearboxSim Map** | Screenshot of current model — ME1/ME2 red, KPI pill showing 242/287 vs. 300 target | 2 min |
| 2 | **Root Cause Analysis** | Why ME1 is the bottleneck (triple-role, 79% util, capacity math), why dead stations exist | 2 min |
| 3 | **Template 1** – Simulation results | KPI table as-is vs. optimized, impact on premises, activities, challenges | 2 min |
| 4 | **Optimized: GearboxSim Map** | Screenshot after ME1b + routing fix — all green, KPI ≥ 300 for both lines | 1 min |
| 5 | **Sensitivity Analysis** | ±20% process time, availability threshold, 1-shift scenario | 2 min |
| 6 | **Template 2** – Layout overview | Layout screenshot, design principles, pros/cons | 2 min |
| 7 | **Template 3** – Key findings | Key findings, improvement potentials, recommendations | 2 min |
| 8 | **Investment case** | What it costs: 1× ME1b station; what it buys: +88 pcs/mo = +29% on Line A | 1 min |
| 9 | **Next steps** | 3 actions with owners and dates | 1 min |
| — | **Q&A** | Open questions | rest |

**Total:** ~16 min content + Q&A buffer

### Slide 0 — Title / Context (1 min)
> "Gearbox Inc. is planning a new factory in Bavaria to produce high-performance gearboxes at scale. The challenge: two product lines must each hit 300 units/month in a brownfield site under tight space and time constraints. We used Siemens Tecnomatix Plant Simulation to validate the factory concept and identify the minimum changes needed to meet target."

### Slide 1 — As-Is GearboxSim Map (2 min)
Show the GearboxSim Visualizer screenshot with current results loaded:
- ME1 and ME2 highlighted red (Bottleneck)
- KPI pill: Gearbox A 242/300 (−19%), Gearbox B 287/300 (−4%)
- Assembly stations with orange/grey indicators
- Walk the audience through the color coding in 30 seconds, then point to ME1

**Key talking point:** "The model ran for 366 simulated days. This is not a theoretical estimate — it is a digital twin with 80% availability, realistic shift calendar, and measured process times."

### Slide 2 — Root Cause Analysis (2 min)
Three root causes, one visual each:
1. **ME1 is triple-loaded:** gear (10 min) + shaft (5 min) + final measuring (35 min) = only ~3,420 gearboxes/year capacity vs. 3,600 needed. Shows as 79% effective utilization.
2. **Assembly routing imbalance:** without Min. Contents, parts pile up at A4/A6 while A5, A10, A11 receive near-zero work — 0% working, 37–38% blocked.
3. **Cascade effect:** ME1 bottleneck → assembly blocks → housing sub-line backs up → lead time inflates to 11h despite ~2h actual processing time.

### Slide 3 — Template 1 (2 min)
Use the filled `TEMPLATES_filled.pptx` Slide 1. Walk through:
- KPI table (242→330+, 287→305+, lead time reduction, Max WIP reduction)
- Impact on premises bullet: "One ME1b station is the only hardware change needed"
- Activities and challenges bullets

### Slide 4 — Optimized GearboxSim Map (1 min)
Screenshot with ME1b added and routing fixed:
- Both lines green, KPI ≥ 300
- Empty recommendation cards
- "This is what the model looks like after two changes: one station and one configuration setting."

### Slide 5 — Sensitivity Analysis (2 min)
| Scenario | Line A | Line B |
|----------|--------|--------|
| As-is baseline | 242/mo | 287/mo |
| +ME1b only | ~330/mo | 287/mo |
| +ME1b + routing fix | ~340/mo | ~305/mo |
| +ME1b + ME2b + routing | ~340/mo | ~315/mo |
| ±20% process time (10 runs) | mean ±CI | mean ±CI |
| 1-shift scenario | ~170/mo | ~145/mo |
| Availability drops to 70% | ~290/mo | ~265/mo |

Point: the two-change fix (ME1b + routing) is robust. Even at 70% availability it stays close to target. 1-shift would require a completely different production concept.

### Slide 6 — Template 2 (2 min)
Use filled Slide 2. Walk through:
- Layout screenshot (Plant Simulation top-down view)
- Flow direction principle
- Measuring station isolation
- Gearbox C reserve zone

### Slide 7 — Template 3 (2 min)
Use filled Slide 3. Move quickly — this is the summary slide.

### Slide 8 — Investment Case (1 min)
| Item | Detail |
|------|--------|
| Change required | 1× ME1b measuring station (same spec as ME1) |
| Hardware cost | 1 additional SingleProc station footprint |
| Output gain | +88 pcs/mo on Line A (+29% vs. current 242) |
| Revenue impact | Depends on gearbox unit margin — quantify with client data |
| Routing fix | Zero hardware cost — configuration change only |
| ME2b (optional) | Insurance; brings B above target with headroom |

**Key message:** "The minimum viable fix is one station and one configuration change. Everything else is upside."

### Slide 9 — Next Steps (1 min)
| Action | Owner | By |
|--------|-------|-----|
| Approve ME1b station in budget | Steering committee | June 22 |
| Apply Min. Contents routing in final model | Simulation team | June 22 |
| Re-run final validation simulation (1 year) | Simulation team | June 22 |
| Finalize layout against building dimensions | Layout team | June 22 |

---

**Winning argument (verbatim for pitch):**
> "One additional ME1 measuring station adds 88 units/month to Line A — that's the entire output gap, gone. Assembly routing set to Min. Contents fixes the three dead stations at zero hardware cost. With these two changes, both lines clear 300 units/month. Our layout reserves a dedicated bay for Gearbox C so the investment is future-proof. We are recommending the minimum intervention with the maximum verified impact."

---

## 9a. Template 1 – Simulation Overview & Results (filled)

**Screenshot:** GearboxSim Visualizer with current validated model loaded — ME1/ME2 red, KPI pill 242/287, then side-by-side with optimized run (≥300 both lines, all green). A before/after screenshot is more compelling than one state alone.

### Evaluation of factory concept by simulation

| KPI | Gearbox A (as-is) | Gearbox A (optimized) | Gearbox B (as-is) | Gearbox B (optimized) |
|-----|-------------------|-----------------------|-------------------|-----------------------|
| Monthly output | **242 pcs/mo** | **~330–350 pcs/mo** | **287 pcs/mo** | **~305–315 pcs/mo** |
| Number of stations | 15 | 16 (+ ME1b) | 12 | 13 (+ ME2b, recommended) |
| Avg. lead time | 11h 15min | ~7h (−38%) | 12h 59min | ~9h (−30%) |
| Max WIP | 31 MUs | ~18 MUs (−42%) | 30 MUs | ~18 MUs (−40%) |

### Impact on factory premises
- Adding 1× ME1b measuring station (same footprint as ME1) resolves the entire Line A output gap
- Dead assembly stations A5/A10/A11 become productive after routing fix — no additional stations needed
- Gearbox B near target as-is; ME2b is low-cost insurance against process time variation

### Main activities during simulation
- Built 27-station model with process times, 80% availability and 2-shift calendar; validated with GearboxSim Visualizer
- Applied Min. Contents exit strategy on all parallel forks; confirmed load balance with Bottleneck Analyzer
- Added ME1b as parallel measuring station; compared as-is vs. optimised throughput scenarios

### Main challenges during simulation
- ME1 triple-role (gear 10 min / shaft 5 min / final measuring 35 min) caused cascading blocking across Line A
- Assembly stations at 0% working revealed exit-strategy misconfiguration, not a capacity issue
- Deadlock-free buffer design required return buffers + grinder-gating SimTalk control

---

## 9b. Template 2 – Layout Overview & Results (filled)

**Screenshot:** Top-down Plant Simulation layout view with machine placement, aisles, and storage zones labeled. Pair with a cropped GearboxSim map showing the two-line topology.

### Main design criteria / principles for layout
- **Flow direction:** Incoming goods → Machining → Assembly → Measuring → Shipping (left to right)
- **Product-based assembly zones:** Line A (A1–A6) and Line B (A7–A11) in parallel, no cross-traffic
- **Measuring stations isolated** from vibration-generating machining (grinding/turning) — quality constraint
- **1 m minimum clearance** on all machine sides (safety/maintenance requirement)
- **Rectangular storage areas** as connected blocks (incoming 162 m² + shipping 90 m²)
- **Gearbox C expansion zone** reserved in dedicated bay without disrupting A/B lines

### Qualitative evaluation of layout

| Pros | Cons |
|------|------|
| Short transport paths between sequential steps | Requires discipline to keep shared machining aisles clear |
| Measuring stations separated from machining vibration | Parallel A/B assembly zones need clear visual separation |
| Clean left→right material flow, easy for operators | Shared machining cluster needs scheduling logic for A/B mixing |
| Reserved Gearbox C zone future-proofs the investment | Slightly larger total footprint than pure product-based layout |

### Main activities during layout development
- Evaluated three layout concepts (Functional, Product-based, Variants/Synergies) against boundary conditions
- Placed 27 stations respecting 1 m clearance rule, crane path constraints and aisle widths
- Sized and positioned storage areas as connected rectangles; reserved Gearbox C expansion bay

### Main challenges during layout development
- Fitting 27 machines + 2 storage areas + aisles + Gearbox C expansion zone within building footprint
- Balancing measuring station isolation requirement against short transport distance from assembly
- Routing both rolling gates to logically connect incoming and shipping areas

---

## 9c. Template 3 – Key Findings / Improvement Potentials / Recommendations (filled)

### Key findings

**Simulation related**
- ME1 (final measuring, 35 min) is the sole bottleneck for Line A — 79% effective utilization, output 242/mo vs. target 300
- Assembly stations A5, A10, A11 at 0% working / 37–38% blocked — wrong exit strategy, not a capacity gap
- ME2 at 79% effective utilization — fragile; a process time increase or availability dip risks missing the Line B target
- Lead Time A = 11h 15min vs. ~2h actual processing: 5× overhead confirms heavy queuing in front of ME1

**Layout related**
- Product-based assembly zones (A-line / B-line) minimize cross-traffic and simplify routing
- Measuring stations placed away from grinding/turning cluster prevents vibration interference with quality results
- Incoming and shipping areas at opposite building ends — clean directional material flow

### Improvement potentials
- **+88 pcs/mo on Line A (+29%):** Add 1× ME1b measuring station → ME1 utilization 79% → ~40%
- **Eliminate dead assembly stations:** Min. Contents routing on A4–A6 and A7–A11 → A5/A10/A11 resume ~10–15% utilization — zero hardware cost
- **Lead time −38%:** ME1 queue dissolves after ME1b added; WIP drops from 31 to ~18 MUs on Line A
- **Protect Line B:** Add ME2b → ME2 drops from 79% to ~40%, robust against ±20% process time variation

### Recommendation to steering committee
- Approve 1× ME1b measuring station immediately — minimum change to meet the Gearbox A output target
- Implement Min. Contents routing on all assembly stations before start-of-production — zero hardware cost
- Reserve building area for Gearbox C expansion; do not fill with temporary storage

### Next steps
- Finalize model with ME1b + routing fix; re-run 1-year simulation to confirm ≥300 pcs/mo for both lines
- Conduct sensitivity analysis (±20% process times, 1-shift scenario) to stress-test before investment approval

---

## 10. Process Times Reference (from PDF p.24)

### Gearbox A
| Process | Station(s) | Time (min) | Parts affected |
|---------|-----------|-----------|---------------|
| Milling Part 1 | MI1 | 10 | Housing Part 1 |
| Milling Part 2 | MI2 | 10 | Housing Part 2 |
| Preassembly | A1/A2/A3 | 30 | Housing (combined) |
| Drilling | D1/D2 | 21 | Housing assembly |
| Turning Gears | T1 | 8 | Gears |
| Grinding Gears | G1 | 9 | Gears |
| Measuring Gears | ME1 | 10 | Gears |
| Turning Shafts | T2 | 5 | Shafts |
| Grinding Shafts | G2 | 3 | Shafts |
| Measuring Shafts | ME1 | 5 | Shafts |
| Assembly | A4/A5/A6 | 30 | Full gearbox |
| **Measuring (final)** | **ME1** | **35** | **Finished GbxA** |

### Gearbox B
| Process | Station(s) | Time (min) | Parts affected |
|---------|-----------|-----------|---------------|
| Milling | MI3 | 5 | Housing |
| Drilling B | D3 | 6 | Housing |
| Turning Gears | T3 | 6 | Gears |
| Grinding Gears | G3 | 7 | Gears |
| Measuring Gears | ME2 | 10 | Gears |
| Turning Shafts | T4 | 3 | Shafts |
| Grinding Shafts | G4 | 3 | Shafts |
| Measuring Shafts | ME2 | 5 | Shafts |
| Assembly | A7/A8/A9/A10/A11 | 25 | Full gearbox |
| **Measuring (final)** | **ME2** | **10** | **Finished GbxB** |

**Critical insight:** ME1 handles gears (10 min), shafts (5 min), AND final assembly measuring (35 min) for Gearbox A — it is triple-loaded. This is the single most important resource to expand.

---

*Document updated: 2026-06-15 | Baseline from 14-06-2026-17-44-PlantSim.spp (validated deadlock-free model)*
