# SimTalk Edge Export — `dumpEdges`

Export the connector graph from PlantSim to an HTML file that GearboxSim can read as an edge table.

---

## Method: `dumpEdges`

Create a **Method** object in your model frame (Toolbox ▸ Information Flow ▸ Method), name it `dumpEdges`, paste the code below, and press **F7** to compile.

### Prerequisites

Before running, create one **TableFile** object in the same frame, named `EdgeTable`.  
Set its columns via its dialog to exactly two columns: `From` (string) and `To` (string).  
The method clears and refills this table on every call, so you can run it repeatedly.

---

### Primary implementation — `obj.numSucc` / `obj.succ(i)`

```simtalk
-- dumpEdges
-- Walks every object in root.Model and records each successor pair as a
-- (From, To) row in the EdgeTable TableFile, then exports it to HTML.
--
-- ATTRIBUTE NAMES TO VERIFY:
--   `numSucc` — number of successors on a material-flow object.
--   `succ(i)` — returns the i-th successor (1-based index).
--   Both names must be confirmed via What's This? (right-click the attribute
--   in the object dialog while the simulation is loaded). If the What's This?
--   tooltip shows a different name, update the two references below.
--   Fallback implementation using Connector objects is provided beneath this block.
--
-- Read-only method: no flow logic, no exitLocked, no @.move.

var exportPath: string := "edges_export.htm"  -- output file next to the model

-- Clear the table so repeated runs do not accumulate stale rows.
EdgeTable.clear

-- Walk every direct child of root.Model.
-- "for each var obj in <frame>" iterates direct children only.
-- Objects deeper in sub-frames are not reached; adjust the path if your
-- model is nested (e.g. root.Model.LineA).
for each var obj in root.Model

    -- Guard: only objects that expose numSucc carry connector information.
    -- Objects that do not have this attribute (Variables, TableFiles, Methods,
    -- FlowControl objects without direct connectors, etc.) will cause a runtime
    -- error if accessed. The `is` operator tests the object class; the safer
    -- guard here is a try/catch pattern — but SimTalk 2.0 has no try/catch,
    -- so we rely on numSucc being 0 or > 0 for station-like objects, and
    -- assume non-station objects simply return 0. Verify this assumption by
    -- running once on a scratch model and checking the Console for errors.
    --
    -- If you get "attribute not found" for numSucc, switch to the Connector
    -- enumeration fallback below.

    if obj.numSucc > 0
        for var i := 1 to obj.numSucc
            -- succ(i) returns the successor object; .name is its PlantSim name.
            -- The name matches the resource-stats rows in the report HTM
            -- (minus the ".Models.Model." prefix).
            var row: integer := EdgeTable.numRows + 1
            EdgeTable.setCell(row, 1, obj.name)
            EdgeTable.setCell(row, 2, obj.succ(i).name)
        next
    end

next

-- Export the table to an HTML file.
-- exportHTML is the verified TableFile export method in Plant Simulation 2024.
-- The file is written to the same directory as the model unless you supply
-- an absolute path.
EdgeTable.exportHTML(exportPath)

print "dumpEdges: " + to_str(EdgeTable.numRows) + " edges written to " + exportPath
```

---

### Fallback implementation — Connector `.from` / `.to` enumeration

Use this if `numSucc` / `succ(i)` are not available or cause errors. PlantSim exposes connector objects directly; each connector carries the source and destination objects.

```simtalk
-- dumpEdges_ViaConnectors
-- Alternative: enumerate Connector objects in root.Model instead of reading
-- successor attributes on each station object.
--
-- ATTRIBUTE NAMES TO VERIFY:
--   Connector objects: in the Plant Simulation object hierarchy the class may
--   be named "Connector", "FlowConnector", or "MaterialFlowConnector".
--   Check via What's This? on a connector line in the frame.
--   `.from` and `.to` return the connected station objects.
--   `.from.name` and `.to.name` return their PlantSim names.
--
-- If the class name differs, replace "Connector" below with the confirmed name.

var exportPath: string := "edges_export.htm"

EdgeTable.clear

for each var obj in root.Model
    -- Test whether this object is a Connector (material-flow link).
    -- "is" operator: "obj is Connector" returns true if obj's class is Connector.
    -- Replace "Connector" with the class name confirmed via What's This?.
    if obj is Connector
        var row: integer := EdgeTable.numRows + 1
        EdgeTable.setCell(row, 1, obj.from.name)
        EdgeTable.setCell(row, 2, obj.to.name)
    end
next

EdgeTable.exportHTML(exportPath)

print "dumpEdges_ViaConnectors: " + to_str(EdgeTable.numRows) + " edges written to " + exportPath
```

---

### TableFile column setup (do once)

Open the `EdgeTable` object dialog → Columns tab → add two columns:

| # | Name | Data type |
|---|------|-----------|
| 1 | From | String    |
| 2 | To   | String    |

Leave all other settings at defaults.

---

### Expected HTML output shape

The exported file must match this structure exactly (PlantSim's `exportHTML` produces it natively):

```html
<table class="Auswertungen">
<thead><tr><th>From</th><th>To</th></tr></thead>
<tbody>
<tr><td>MI1_Milling_GearBoxA_Housing1</td><td>Buffer_GearboxA_HousingPart_MI1_A1</td></tr>
<tr><td>Buffer_GearboxA_HousingPart_MI1_A1</td><td>A1_PreAssembly_GearBoxA</td></tr>
</tbody>
</table>
```

- Class `Auswertungen` — PlantSim's default export class; GearboxSim detects it.
- Column headers `From` / `To` must match exactly (case-insensitive in the parser).
- Cell values are the bare PlantSim object names, without any `.Models.Model.` prefix.

> **Note:** `exportHTML` may wrap the table in `<html><body>…</body></html>` tags.
> GearboxSim's parser scans the whole document for `table.Auswertungen`, so the
> wrapper is harmless.

---

## Operator how-to

### Running the export

1. Open your model in Plant Simulation.
2. Run a full simulation (or load a completed run's state).
3. Open **Tools → Execute Method…**, navigate to `dumpEdges`, click **Execute**.
4. The Console prints the edge count and the output file path.

### File naming for auto-pairing (IMPORTANT)

The visualizer bundles a results file with its edge file **by filename stem**. Name the
edge export as the results file **plus an `_edges` suffix**:

| Results file | Edge file (name it exactly like this) |
|---|---|
| `results_2020_19-06-26_14-46.htm` | `results_2020_19-06-26_14-46_edges.htm` |
| `results_2020_18-06-26_23-58.htm` | `results_2020_18-06-26_23-58_edges.htm` |

Set `exportPath` in `dumpEdges` to match the run you just saved, e.g.
`var exportPath := "results_2020_19-06-26_14-46_edges.htm"`. The pairing is
case-insensitive and also accepts an `edges_` **prefix** (`edges_<stem>.htm`) or a
`-edges`/`.edges` suffix — but `<stem>_edges.htm` is the recommended form.

> The stem (everything before `_edges`) must be **identical** to the results file's
> stem. A mismatch means the edge file won't pair and will be ignored (the tool logs
> a `[bundle] edge file with no matching results` warning to the console).

### Using the files in GearboxSim

1. Open `gearboxsim.html` in your browser (append `?dynamic=1` to enable dynamic layout).
2. Drag-and-drop **all files at once** — e.g. 6 files (3 results + 3 edge tables). The tool
   groups them into 3 run tabs, each results file bundled with its matching edge file.
   Order does not matter; you can also drop a single combined file that contains both tables.
3. Each run tab keeps its **own** topology — switching tabs swaps both the KPIs and the map.
4. If a results file has no paired edge file, the tool falls back to its hardcoded topology
   map for that run (existing behaviour, unchanged).

> **When to re-export:** Re-run `dumpEdges` whenever you add, remove, or rename
> objects or connectors. The statistics export and the edge export should always
> come from the same simulation run and share the same filename stem.

---

## Notes on attribute verification

PlantSim's object model is version-dependent. Before deploying to a shared model:

1. Right-click any station (e.g. `MI1_Milling_GearBoxA_Housing1`) → **What's This?**
2. In the attribute browser, search for "succ" and "numSucc".
3. If the names differ (e.g. `successor`, `successors`, `numSuccessors`), update the method accordingly.
4. For the Connector fallback, right-click a connector line → **What's This?** to confirm the class name and `.from` / `.to` attribute names.

The primary implementation has been written against the attribute names most commonly cited in Plant Simulation 2024 documentation. The fallback covers installations where those names differ.
