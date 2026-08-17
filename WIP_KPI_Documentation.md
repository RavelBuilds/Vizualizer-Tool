# WIP KPI Documentation — Gearbox Inc. Plant Simulation
**Project:** OPP Praktikum SoSe 2026 · Siemens Advanta / HS München  
**Scope:** Work-in-Progress (WIP) methodology, three data sources, V1 vs. V2 comparison

---

## 1. The Problem: Three Different WIP Numbers

The Plant Simulation model produces three distinct WIP figures that appear contradictory:

| Source | V1 Line A | V1 Line B | V2 Line A | V2 Line B |
|---|---|---|---|---|
| HTM statistics report (`WIPs per month`) | 1,277 MU | 1,763 MU | 2,174 MU | 2,675 MU |
| GearboxSim Visualizer (`System WIP`) | 21 MU | 13 MU | 10 MU | 12 MU |
| Little's Law (calculated) | **~3.7** | **~5.1** | **~1.9** | **~1.3** |

Each number is technically correct — but they measure fundamentally different things. Using the wrong one in a steering committee presentation will undermine the credibility of the entire KPI analysis.

---

## 2. What Each Number Actually Measures

### 2.1 HTM Report — `WIPs per month`: Cumulative throughput artifact

The HTM statistics field labelled "WIPs per month" does **not** represent the number of parts simultaneously in the system. It is the **sum of all Material Unit (MU) movements** recorded by the simulation over the full 366-day run, divided by a time window.

The root cause is **source overproduction**: the simulation generates one MU per sub-part, not per finished gearbox. Each Gearbox A requires:
- 2× housing sub-parts (HousingPart1, HousingPart2)
- 2× gear sub-parts (Gears)
- 2× shaft sub-parts (Shafts)

This means approximately **6 MUs are created per finished gearbox**. The simulation then tracks every individual MU through every station, buffer, and return loop. The cumulative count over 366 days for 5,255 finished Gearbox A units is therefore on the order of **5,255 × 6 ≈ 31,500 MU movements** — which, when collapsed into a monthly figure, produces the 1,277–2,675 MU range seen in the HTM report.

**Conclusion:** This number must not be used as a WIP KPI. It is a simulation bookkeeping artefact, not a measure of inventory in the system.

### 2.2 GearboxSim Visualizer — `System WIP`: All concurrent MUs across buffers

The visualizer reports the **instantaneous count of all MUs present in the system** at any given moment, averaged over the simulation run. This includes:
- Finished gearbox assemblies in transit between stations
- Sub-parts waiting in intermediate buffers (Buffer_Gears, Buffer_Shafts, Buffer_Return_A/B)
- Parts currently being processed at a station

Since each finished gearbox has ~6 sub-parts in flight at various stages simultaneously, the visualizer correctly reports values in the **10–21 MU range**. This is a valid and meaningful number — it tells you how much physical inventory exists in the factory at any given moment, including raw sub-parts.

For V1: System WIP = 34 MU total (A: 21, B: 13)  
For V2: System WIP = 22 MU total (A: 10, B: 12)

The reduction from V1 to V2 reflects the elimination of the bottleneck queue ahead of ME1/ME2.

### 2.3 Little's Law — `WIP per finished gearbox`: Theoretical minimum

Little's Law states:

> **WIP = Throughput × Lead Time**

where both quantities must be expressed in the same time unit. Using calendar time (24-hour days):

**Version 1:**

| | Line A | Line B |
|---|---|---|
| Monthly output | 212 pcs/mo | 252 pcs/mo |
| Throughput (pcs/day) | 212 ÷ 30.42 = **6.97 pcs/day** | 252 ÷ 30.42 = **8.28 pcs/day** |
| Mean lead time | 12h 46min = **0.532 days** | 14h 50min = **0.618 days** |
| **WIP (Little's Law)** | 6.97 × 0.532 = **~3.7 finished gearboxes** | 8.28 × 0.618 = **~5.1 finished gearboxes** |

**Version 2:**

| | Line A | Line B |
|---|---|---|
| Monthly output | 362 pcs/mo | 382 pcs/mo |
| Throughput (pcs/day) | 362 ÷ 30.42 = **11.90 pcs/day** | 382 ÷ 30.42 = **12.56 pcs/day** |
| Mean lead time | 3h 48min = **0.158 days** | 2h 29min = **0.104 days** |
| **WIP (Little's Law)** | 11.90 × 0.158 = **~1.9 finished gearboxes** | 12.56 × 0.104 = **~1.3 finished gearboxes** |

Little's Law measures only **finished gearboxes simultaneously in the system** — from the moment a housing enters pre-assembly until the finished gearbox exits the measuring station. It deliberately excludes sub-parts, buffers, and return loops. This is the theoretical lower bound: the minimum possible WIP given the throughput and lead time.

---

## 3. Why the Numbers Diverge: Parallel Stations and Return Loops

The factory model contains two structural elements that cause the Visualizer WIP to exceed the Little's Law estimate:

**Parallel measuring stations (ME1+ME3, ME2+ME4 in V2):**  
Each finished gearbox passes through either ME1 or ME3 (Line A), never both. However, at any given moment, both stations may hold a gearbox simultaneously. Little's Law already accounts for this correctly — higher throughput through parallel stations is reflected in the throughput term. The divergence from the Visualizer arises because the Visualizer counts sub-parts, not finished gearboxes.

**Return buffers (Buf_Ret_A, Buf_Ret_B):**  
Parts that re-enter the measuring loop via the return buffer extend their individual lead time beyond the mean. This increases the observed mean lead time slightly above the theoretical minimum, and causes a small number of MUs to be "double-counted" in the system at any moment. This is why the Visualizer's 10–12 MU per line slightly exceeds the Little's Law estimate of 1.3–1.9 finished gearboxes × 6 sub-parts = ~8–11 MU.

The two estimates are therefore consistent once the sub-part multiplier (~6 MU per gearbox) and the return loop overhead are accounted for.

---

## 4. Which Number to Use and When

| Context | Recommended figure | Rationale |
|---|---|---|
| Steering committee KPI table | **Visualizer System WIP (10/12 MU, V2)** | Directly readable, tool-validated, includes all physical inventory |
| Bottleneck analysis / lead time discussion | **Little's Law (~1.9 / ~1.3, V2)** | Shows impact of bottleneck resolution on in-process inventory |
| Do not use | HTM `WIPs per month` | Simulation bookkeeping artefact, not a physical inventory measure |

**Key message for the presentation:**  
V2 reduces System WIP from 34 MU (V1) to 22 MU (V2) — a 35% reduction — driven entirely by the elimination of the queue ahead of the measuring stations. Little's Law confirms this: mean lead time drops 70–83%, which mechanically forces WIP down even as throughput increases.

---

## 5. Summary Table

| KPI | V1 Line A | V1 Line B | V2 Line A | V2 Line B |
|---|---|---|---|---|
| Monthly output | 212 pcs | 252 pcs | 362 pcs | 382 pcs |
| Mean lead time | 12h 46min | 14h 50min | 3h 48min | 2h 29min |
| Little's Law WIP (finished gearboxes) | ~3.7 | ~5.1 | ~1.9 | ~1.3 |
| System WIP — Visualizer (all MUs) | 21 MU | 13 MU | 10 MU | 12 MU |
| HTM `WIPs per month` (artefact — do not use) | 1,277 MU | 1,763 MU | 2,174 MU | 2,675 MU |

---

*Methodology: Little's Law applied with calendar-time normalization (30.42 days/month, 24h/day). Throughput from GearboxSim Visualizer (250 work days/year normalization). Lead time from Plant Simulation Drain statistics (Mean Life Time). System WIP from GearboxSim Visualizer instantaneous average.*
