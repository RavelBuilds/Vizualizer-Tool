# Sensitivity Analysis: +20% Process Times
## Step-by-step guide — Gearbox Inc. Plant Simulation

> **Goal:** Run a scenario where every machine takes 20% longer to process, to stress-test whether the optimized model (ME1b + ME2b) still meets the 300 pcs/mo target.
>
> **Time required:** ~15 minutes to set up, then one 1-year simulation run.

---

## Before you start

- [ ] Open your **optimized model** (with ME1b/ME2b added) in Plant Simulation
- [ ] **File → Save As** → name it `model_sensitivity_120pct.spp` — all edits go here, never on the baseline

---

## Method A — Manual entry (copy-paste the table below)

Open each station → **Times tab** → replace the processing time with the value from the **"Enter this"** column. No SimTalk required.

### Line A — Fixed stations

| Station | Base time | **Enter this (×1.2)** |
|---|---|---|
| MI1 Milling GearBoxA Housing1 | 10:00 | **12:00** |
| MI2 Milling GearBoxA Housing2 | 10:00 | **12:00** |
| A1 PreAssembly GearBoxA | 30:00 | **36:00** |
| A2 PreAssembly GearBoxA | 30:00 | **36:00** |
| A3 PreAssembly GearBoxA | 30:00 | **36:00** |
| D1 Drilling GearBoxA | 21:00 | **25:12** |
| D2 Drilling GearBoxA | 21:00 | **25:12** |
| T1 Turning GearBoxA Gears | 08:00 | **09:36** |
| G1 Grinding GearBoxA Gears | 09:00 | **10:48** |
| T2 Turning GearBoxA Shafts | 05:00 | **06:00** |
| G2 Grinding GearBoxA Shafts | 03:00 | **03:36** |
| A4 Assembly GearBoxA | 30:00 | **36:00** |
| A5 Assembly GearBoxA | 30:00 | **36:00** |
| A6 Assembly GearBoxA | 30:00 | **36:00** |

### Line B — Fixed stations

| Station | Base time | **Enter this (×1.2)** |
|---|---|---|
| MI3 Milling GearBoxB Housing | 05:00 | **06:00** |
| D3 Drilling GearBoxB | 06:00 | **07:12** |
| T3 Turning GearBoxB Gears | 06:00 | **07:12** |
| G3 Grinding GearBoxB Gears | 07:00 | **08:24** |
| T4 Turning GearBoxB Shafts | 03:00 | **03:36** |
| G4 Grinding GearBoxB Shafts | 03:00 | **03:36** |
| A7 Assembly GearBoxB | 25:00 | **30:00** |
| A8–A11 Assembly GearBoxB *(V1 only)* | 25:00 | **30:00** |

### ME1 / ME2 — DataTable rows

Open the DataTable linked in each ME station's Times tab and update the time column:

| Station | MU type | Base time | **Enter this (×1.2)** |
|---|---|---|---|
| ME1 | Raw Gears A | 10:00 | **12:00** |
| ME1 | Raw Shafts A | 05:00 | **06:00** |
| ME1 | Finished GearboxA | 35:00 | **42:00** |
| ME1b *(if added)* | Raw Gears A | 10:00 | **12:00** |
| ME1b | Raw Shafts A | 05:00 | **06:00** |
| ME1b | Finished GearboxA | 35:00 | **42:00** |
| ME2 | Raw Gears B | 10:00 | **12:00** |
| ME2 | Raw Shafts B | 05:00 | **06:00** |
| ME2 | Finished GearboxB | 10:00 | **12:00** |
| ME2b *(if added)* | Raw Gears B | 10:00 | **12:00** |
| ME2b | Raw Shafts B | 05:00 | **06:00** |
| ME2b | Finished GearboxB | 10:00 | **12:00** |

Once all values are entered → skip to **[Step 5 — Run the simulation](#step-5--run-the-simulation)**.

---

## Method B — SimTalk (automated, for repeated runs)

Skip this if you used Method A.

### Step 1 — Create the `scaleProcTimes` method

This method scales the `procTime` of every fixed-time SingleProc station by 1.2.

1. In the **Toolbox** panel, expand **Information Flow**
2. Drag a **Method** object into the Model frame
3. Press **F2** and rename it `scaleProcTimes`
4. Double-click to open the editor, paste the following code:

```simtalk
-- scaleProcTimes
-- Scales procTime of every SingleProc in the frame by 1.2
-- Call ONCE manually before the run (Tools -> Execute Method)

for each var obj in root.Model
    if isKindOf(obj, SingleProc)
        obj.procTime := obj.procTime * 1.2
        print obj.name + ": " + to_str(obj.procTime)
    end
next
print "--- All SingleProc times scaled x1.2 ---"
```

5. Press **F7** to compile — fix any errors before continuing
6. If `root.Model` causes an "identifier not found" error, check the frame path in any station's tooltip (format: `.Models.Model.*`) and adjust accordingly

> **Stations this covers:** MI1, MI2, MI3, A1, A2, A3, D1, D2, D3, T1, T2, T3, T4, G1, G2, G3, G4, A4, A5, A6, A7 (and ME1b/ME2b if they have a fixed `procTime`).

---

### Step 2 — Find the ME1 / ME2 DataTable name

ME1 and ME2 serve three MU types (raw gears, raw shafts, finished gearboxes) and pull their times from a **DataTable** — `procTime` does not drive them.

1. Double-click **ME1_Measuring_GearBoxA** in the model
2. Go to the **Times** tab
3. Note the name in the "Processing Time" field — it will reference a DataTable or a method that reads one
4. Open that DataTable (double-click it in the frame), confirm the layout:

| Row | MU class | Time (min) |
|-----|----------|------------|
| 1 | GearBoxA Gear | 10 |
| 2 | GearBoxA Shaft | 5 |
| 3 | Finished GearboxA | 35 |

5. Note the **exact object name** of the DataTable (e.g. `DataTable_ME1`) and which **column number** holds the time value
6. Repeat for ME2

---

### Step 3 — Create the `scaleMeTimes` method

1. Drag a second **Method** into the Model frame, rename it `scaleMeTimes`
2. Paste the code below, **replacing the DataTable names and column index** with what you found in Step 2:

```simtalk
-- scaleMeTimes
-- Scales ME1 and ME2 DataTable process times by 1.2
-- Call ONCE manually before the run (Tools -> Execute Method)

var dt: object
var row: integer

-- ME1
dt := root.Model.DataTable_ME1   -- REPLACE with your actual DataTable name
for row := 1 to dt.yDim
    dt[row, 2] := dt[row, 2] * 1.2   -- REPLACE 2 with your time column index
    print "ME1 row " + to_str(row) + ": " + to_str(dt[row, 2])
next

-- ME2
dt := root.Model.DataTable_ME2   -- REPLACE with your actual DataTable name
for row := 1 to dt.yDim
    dt[row, 2] := dt[row, 2] * 1.2
    print "ME2 row " + to_str(row) + ": " + to_str(dt[row, 2])
next

print "--- ME1/ME2 DataTables scaled x1.2 ---"
```

3. Press **F7** to compile

---

### Step 4 — Execute the scaling methods

> Do this in order. **Do not** assign these methods to any entrance control or `init` — calling them more than once compounds the scaling (1.44×, 1.73×, ...).

1. **Tools → Execute Method → scaleProcTimes**
   - Watch the Console — every station name and its new time should print
   - Expected output example: `MI1_Milling_GearBoxA_Housing1: 00:12:00`

2. **Tools → Execute Method → scaleMeTimes**
   - Console should show all three rows for ME1 and ME2
   - Expected: `ME1 row 1: 00:12:00`, `ME1 row 2: 00:06:00`, `ME1 row 3: 00:42:00`

**Verify a spot-check:** double-click MI1 → Times tab → should now show `00:12:00` (was `00:10:00`).

---

## Step 5 — Run the simulation

1. **Reset** the model (clears all MUs, resets statistics)
2. Confirm the **EventController** stop time is set to **1 year** (8,760 h or 365 days)
3. Press **Run**
4. Wait for the run to complete

---

## Step 6 — Export results

1. **Results → Statistics → Save as HTML**
2. Name the file: `results_sensitivity_120pct_YYYY-MM-DD.htm`
3. Save to the Visualizer Tool folder

---

## Step 7 — Analyze with GearboxSim Visualizer

1. Open `gearboxsim.html` in Chrome or Edge
2. **Drag and drop** the new `results_sensitivity_120pct_*.htm` file onto the dashboard
3. Check the KPI pill for:
   - Gearbox A pcs/mo vs. 300 target
   - Gearbox B pcs/mo vs. 300 target
   - ME1/ME2 utilization — do they tip back into Bottleneck (>75%)?
   - Lead time increase vs. baseline

---

## Expected results

| KPI | V2 Baseline (100%) | After +20% proc times |
|-----|-------------------|----------------------|
| Gearbox A pcs/mo | 362 | ~290–320 (estimate) |
| Gearbox B pcs/mo | 382 | ~310–350 (estimate) |
| ME1 effective util | ~40% | ~48–52% |
| ME2 effective util | ~40% | ~48–52% |
| Lead Time A | ~3h 48min | ~5–6h (estimate) |

> If either line drops below 300 pcs/mo, the optimized design is not robust to a 20% process time increase — this becomes a key risk slide in the pitch.

---

## Returning to baseline

**Do not Reset+Run again** after scaling — that will NOT undo the changes. The `procTime` and DataTable values are permanently modified in this model file.

To return to baseline:
- **Option A (recommended):** Close without saving → reopen the original `model_optimized_baseline.spp`
- **Option B:** Run `scaleProcTimes` / `scaleMeTimes` again with factor `/ 1.2` instead of `* 1.2` (exact inverse), but rounding errors may accumulate

---

## Pitch slide talking point

> "Even with 20% longer process times across all stations — simulating supplier delays, aging equipment, or operator variability — Line A produces ~X pcs/mo and Line B produces ~Y pcs/mo. The ME1b/ME2b investment maintains a safety margin above the 300/mo target even under adverse conditions."

---

*Guide generated: 2026-06-18 | Based on validated model: `14-06-2026-17-44-PlantSim.spp` with V2 optimizations*
