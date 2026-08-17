# Integrate the Edge Export into Plant Simulation — Step by Step

Goal: make your model export a **From/To edge table** so GearboxSim can draw the flow map dynamically. One-time setup (~10 min), then one click per run.

> Companion reference (full method code + fallback): `simtalk_edge_export.md`.
> What the visualizer does with the file: `dynamic_topology_execution_plan.md`.

---

## Part A — One-time setup

### Step 1 — Create the `EdgeTable` (a TableFile)
1. Open your model frame.
2. Toolbox → **Information Flow** → drag a **TableFile** into the frame.
3. Press **F2**, rename it `EdgeTable`.
4. Double-click it → in the dialog define exactly two **string** columns:

   | Column | Name | Data type |
   |--------|------|-----------|
   | 1 | `From` | string |
   | 2 | `To`   | string |

5. Click **OK**.

### Step 2 — Create the `dumpEdges` method
1. Toolbox → **Information Flow** → drag a **Method** into the frame.
2. Press **F2**, rename it `dumpEdges`.
3. Double-click to open the editor and **paste this code** (primary version):

```simtalk
-- dumpEdges
-- Records every successor pair (From, To) into EdgeTable, then exports to HTML.
-- Read-only: no flow logic, no exitLocked, no @.move.
-- Verify the flagged attribute names in Step 3 before trusting a run.

var exportPath: string := "edges_export.htm"   -- rename per run in Step 5

EdgeTable.clear                                 -- (verify: maybe .delete)

for each var obj in root.Model                  -- direct children of the model frame
    if obj.numSucc > 0                          -- (verify: numSucc / succ)
        for var i := 1 to obj.numSucc
            var row: integer := EdgeTable.numRows + 1   -- (verify: maybe .yDim)
            EdgeTable.setCell(row, 1, obj.name)          -- (verify: maybe EdgeTable[1, row] := ...)
            EdgeTable.setCell(row, 2, obj.succ(i).name)
        next
    end
next

EdgeTable.exportHTML(exportPath)                -- (verify: maybe .saveHTMLFile)

print "dumpEdges: " + to_str(EdgeTable.numRows) + " edges written to " + exportPath
```

4. Press **F7** (Check Syntax). Do **not** run it yet — first verify the attribute names in Step 3.

> **Fallback (use only if `numSucc`/`succ` don't exist in your build):** enumerate Connector objects instead. Same setup, different loop body:
> ```simtalk
> -- dumpEdges (Connector fallback)
> var exportPath: string := "edges_export.htm"
> EdgeTable.clear
> for each var obj in root.Model
>     if obj is Connector                        -- (verify class name via What's This?)
>         var row: integer := EdgeTable.numRows + 1
>         EdgeTable.setCell(row, 1, obj.from.name)
>         EdgeTable.setCell(row, 2, obj.to.name)
>     end
> next
> EdgeTable.exportHTML(exportPath)
> print "dumpEdges (connectors): " + to_str(EdgeTable.numRows) + " edges"
> ```

### Step 3 — Verify the attribute/method names (DO NOT SKIP)
PlantSim's TableFile and successor API names vary by version. Confirm each name the method uses, via **What's This?** (click the **?** on a dialog title bar, then click the field) or the SimTalk reference (F1 on the keyword):

| Used in method | What it should do | If F7 errors, likely correct name |
|---|---|---|
| `obj.numSucc` | count of successors | `numSucc` is standard; else check successor list attr |
| `obj.succ(i)` | i-th successor object | `succ(index)` |
| `EdgeTable.clear` | empty the table | try `EdgeTable.delete` |
| `EdgeTable.numRows` | current row count | try `EdgeTable.yDim` |
| `EdgeTable.setCell(r,c,v)` | write a cell | try `EdgeTable[c, r] := v` (note: **column, row** order) |
| `EdgeTable.exportHTML(path)` | save table as HTML | try `EdgeTable.saveHTMLFile(path)` / `writeFile` |

> Fix any that F7 flags, using the right column of this table, until **F7 reports no errors.** If `numSucc`/`succ` aren't available at all, switch to the **Connector `.from`/`.to` fallback** in `simtalk_edge_export.md`.

### Step 4 — Smoke-test on one run
1. Run a short simulation (or load a completed run).
2. **Tools → Execute Method** → pick `dumpEdges` → **Execute**.
3. Check the **Console**: you should see `dumpEdges: <N> edges written to edges_export.htm` with N in the dozens (not 0).
4. Open the produced `edges_export.htm` in a text editor — confirm it contains `<table class="Auswertungen">` with `<th>From</th><th>To</th>` and rows of object names.

If N = 0 or you see "attribute not found": go back to Step 3 / switch to the fallback.

---

## Part B — Per-run export (do this for every results file)

### Step 5 — Name the edge file to match the results file
The visualizer pairs files by **filename stem**. The edge file must be the results file name **+ `_edges`**:

| Results file you saved | Set `exportPath` in `dumpEdges` to |
|---|---|
| `results_2020_19-06-26_14-46.htm` | `results_2020_19-06-26_14-46_edges.htm` |

1. In `dumpEdges`, edit the first line:
   `var exportPath: string := "results_2020_19-06-26_14-46_edges.htm"`
2. (Tip) Put both files in the same folder so they're easy to grab together.

### Step 6 — Export after the run
1. Run the simulation to completion (statistics valid).
2. Save the **statistics report** as usual → `results_<date>.htm`.
3. **Execute `dumpEdges`** → writes `results_<date>_edges.htm`.
4. The two files now share a stem and will auto-bundle.

---

## Part C — Use in GearboxSim

### Step 7 — Drop the files
1. Open `gearboxsim.html` in Chrome/Edge, adding **`?dynamic=1`** to the URL to enable the edge-driven map (e.g. `…/gearboxsim.html?dynamic=1`).
2. Drag in **all files at once** — e.g. 6 files (3 results + 3 `_edges`). The tool groups them into **3 run tabs**, each results file bundled with its matching edge file.
3. Switch tabs — each run shows its **own** KPIs and its **own** topology.

### Step 8 — Sanity-check
- New buffers (e.g. `Buffer_GearboxA_HousingPart_MI1_A1`) now appear as nodes on the map, positioned left→right by process depth, Line A above Line B, return loops dashed.
- Without `?dynamic=1`, or for any results file with no paired edge file, the tool falls back to the existing hardcoded map (unchanged).

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Console: `dumpEdges: 0 edges` | `numSucc`/`succ` wrong or model nested in a sub-frame. Verify names (Step 3); change `root.Model` to the actual frame path; or use the Connector fallback. |
| F7 error "unknown attribute …" | Name differs in your build — use the right-hand column of the Step 3 table. |
| Edge file dropped but map unchanged | URL missing `?dynamic=1`, **or** the edge file stem ≠ results stem. Check the browser console for `[bundle] edge file with no matching results`. |
| Map shows nodes but no/zero edges | The edge table parsed but names don't match the stats rows. Confirm cell values are bare names (no `.Models.Model.` prefix) — Step 4. |
| Two models' edges mixed on one tab | Should not happen after the per-run fix; if it does, re-drop the files (each tab now stores its own edges). |
| Wrong arrow direction | The method writes `From = obj`, `To = successor`. If reversed, swap columns 1/2 in the `setCell` calls. |

---

## Re-export checklist (each time topology changes)
- [ ] Added/removed/renamed a station, buffer, or connector → re-run `dumpEdges`.
- [ ] `exportPath` set to this run's `<resultStem>_edges.htm`.
- [ ] Stats file and edge file are from the **same** run and share the stem.

*Setup guide authored 2026-06-19. Method/source of truth: `simtalk_edge_export.md`.*
