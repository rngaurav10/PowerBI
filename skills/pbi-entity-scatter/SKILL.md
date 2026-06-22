---
name: pbi-entity-scatter
description: >
  Builds a fully interactive 4-quadrant scatter plot for entity-level analysis (e.g. agents, sales reps,
  customers, products, stores) as a DAX measure in Power BI. X-axis shows total metric value,
  Y-axis shows year-over-year growth %. Quadrants classify entities into 4 performance groups.
  Features: draggable X/Y divider lines, outlier detection and exclusion (3-sigma), modal drill-down
  table per quadrant, KPI strip showing quadrant totals, avg/median line toggles, negative value filter,
  custom X/Y threshold inputs, and hover tooltips. Works on any dataset in any industry.
user-invocable: true
allowed-tools:
  - mcp__powerbi__connect_powerbi
  - mcp__powerbi__list_tables
  - mcp__powerbi__get_table_schema
  - mcp__powerbi__list_measures
  - mcp__powerbi__create_measure
  - mcp__powerbi__execute_dax
  - mcp__powerbi__validate_dax
---

# /pbi-entity-scatter — 4-Quadrant Entity Scatter Plot for Power BI

Produces an interactive **4-quadrant scatter plot** that classifies entities (agents, customers,
products, stores, regions, etc.) by value on the X-axis and year-over-year growth on the Y-axis.
Rendered as a DAX measure via the HTML Viewer custom visual.

Works on **any dataset** with entity identifiers, a year column, a month column, and one or more
numeric metric columns. No domain assumptions.

Arguments: `$ARGUMENTS`

---

## SECTION A — MANDATORY BEHAVIOUR RULES

### A1 — Connect, inspect, version, test
Same discipline as all skills: connect → schema → build → create new version → validate.

### A2 — Column names from schema only
Zero guessing. All column references verified against `get_table_schema`.

### A3 — Always increment version
Check `list_measures`. Auto-increment.

### A4 — Reserved DAX VAR names — NEVER use
`Table`, `Filter`, `Values`, `All`, `Calculate`, `Row`, `Date`, `Time`, `Today`, `Now`,
`DataTable`, `Calendar`, `Union`, `Intersect`, `Except`, `Generate`, `AddColumns`,
`SelectColumns`, `Summarize`, `CrossJoin`, `Distinct`

### A5 — Outlier handling is mandatory
The visual MUST compute outliers using 3-sigma. Outliers must be excluded from the scatter main area
but still counted in a separate "Outliers" quadrant KPI. This prevents the scatter from becoming
unreadable when a few entities are orders of magnitude larger than the rest.

---

## SECTION B — DISCOVERY PHASE

### B1 — Data table
`list_tables` → ask user. Save as **TABLE**.

### B2 — Schema
`get_table_schema(TABLE)`. Classify:
- **YEAR_COL**: integer 4-digit year
- **MONTH_COL**: integer 1–12
- **ENTITY_NAME_COL**: text column that names each entity (shown as dot labels and in modal tables)
- **METRIC_CANDIDATES**: numeric additive columns

> **Note on entity mapping table:** Some models store entity names in a separate mapping/dimension table
> joined to the fact table. If the schema shows no entity name column in TABLE, ask:
> "Is there a separate table that maps entity IDs to names? (e.g. an Agent mapping table)"
> If yes: show `list_tables`, let user pick **MAPPING_TABLE** and **MAPPING_NAME_COL**.

### B3 — Year and month
Ask for YEAR_COL and MONTH_COL (same pattern as other skills).

### B4 — Primary metric (X-axis)
> "Which column is the **main metric value** shown on the X-axis?
> (e.g. Total Revenue, Gross Premium, Units Sold — the overall size of the entity)"

Save as **METRIC_MAIN**. Label → **LABEL_MAIN** (e.g. "Total Revenue").

> "Is there a **second metric** to toggle to (e.g. New Business, Net Revenue)?
> 1. Yes
> 2. No"

Save as **METRIC_ALT** (or null). Label → **LABEL_ALT**.

### B5 — Prior year for growth
Growth Y-axis = (CurrentYear - PriorYear) / |PriorYear| × 100.

The prior year is auto-detected as MAX(YEAR_COL) - 1.
A second prior year (MAX - 2) is used for the "PY-2" column in the modal table.

### B6 — Quadrant labels
> "What should the 4 performance quadrants be called?
> The default names are:
> Top-right (high value, high growth): **Star Performers**
> Top-left (low value, high growth): **Growth Potential**
> Bottom-right (high value, low growth): **Cash Cows**
> Bottom-left (low value, low growth): **Action Required**
>
> Use these defaults, or type custom names?"

Save as **QUAD_LABELS** array `[bottom-left, bottom-right, top-left, top-right]`.

### B7 — "No mapping" entity
Entities not appearing in the mapping table (if MAPPING_TABLE used) are grouped into a special
"No Mapping" dot. Ask:
> "How should unmatched entities be handled?
> 1. Show as a grey 'No Mapping' dot (default)
> 2. Exclude them from the scatter"

### B8 — Measure name
Ask, auto-increment, save as **MEASURE_NAME**.

---

## SECTION C — DAX ARCHITECTURE

### C1 — Entity data aggregation

```dax
VAR _ly  = CALCULATE(MAX(TABLE[YEAR_COL]), ALL(TABLE))   -- current year
VAR _py  = _ly - 1
VAR _py2 = _ly - 2
VAR _mm  = CALCULATE(MAX(TABLE[MONTH_COL]),
               ALL(TABLE[YEAR_COL]), ALL(TABLE[MONTH_COL]))  -- last available month

-- Grand totals (for % of portfolio calculation)
VAR _allMainLy = CALCULATE(SUM(TABLE[METRIC_MAIN]),
    ALL(TABLE[YEAR_COL]), ALL(TABLE[MONTH_COL]),
    TABLE[YEAR_COL]=_ly, TABLE[MONTH_COL]<=_mm)
VAR _allMainPy = CALCULATE(SUM(TABLE[METRIC_MAIN]),
    ALL(TABLE[YEAR_COL]), ALL(TABLE[MONTH_COL]),
    TABLE[YEAR_COL]=_py, TABLE[MONTH_COL]<=_mm)
VAR _allAltLy  = CALCULATE(SUM(TABLE[METRIC_ALT]), ...)   -- same pattern, if METRIC_ALT != null
-- ... _allAltPy, _allMain2, _allAlt2

-- Per-entity aggregation (using MAPPING_TABLE if provided)
VAR _rawFull = ADDCOLUMNS(
    SUMMARIZE(<<ENTITY_SOURCE>>, <<ENTITY_NAME_COL>>),
    "@gl",  CALCULATE(SUM(TABLE[METRIC_MAIN]), ..., TABLE[YEAR_COL]=_ly, TABLE[MONTH_COL]<=_mm),
    "@gv",  CALCULATE(SUM(TABLE[METRIC_MAIN]), ..., TABLE[YEAR_COL]=_py, TABLE[MONTH_COL]<=_mm),
    "@g2",  CALCULATE(SUM(TABLE[METRIC_MAIN]), ..., TABLE[YEAR_COL]=_py2, TABLE[MONTH_COL]<=_mm),
    "@nl",  CALCULATE(SUM(TABLE[METRIC_ALT]),  ..., TABLE[YEAR_COL]=_ly, TABLE[MONTH_COL]<=_mm),
    "@nv",  CALCULATE(SUM(TABLE[METRIC_ALT]),  ..., TABLE[YEAR_COL]=_py, TABLE[MONTH_COL]<=_mm),
    "@n2",  CALCULATE(SUM(TABLE[METRIC_ALT]),  ..., TABLE[YEAR_COL]=_py2, TABLE[MONTH_COL]<=_mm)
)
VAR _raw = SELECTCOLUMNS(_rawFull,
    "@n", <<ENTITY_NAME_COL>>,
    "@gl", [@gl], "@gv", [@gv], "@g2", [@g2],
    "@nl", [@nl], "@nv", [@nv], "@n2", [@n2])

-- "No Mapping" residual (entities not in mapping table):
VAR _nmGl = _allMainLy - SUMX(_raw, IF(ISBLANK([@gl]),0,[@gl]))
-- ... _nmGv, _nmG2, _nmNl, _nmNv, _nmN2
VAR _nmRow = ... (add single row for "No Mapping" entity)
VAR _raw2 = UNION(_raw, FILTER(_nmRow, NOT ISBLANK([@gl]) && [@gl] > 0))

-- Filter for scatter main area (must have both CY and PY values)
VAR _hasLY     = FILTER(_raw2, NOT ISBLANK([@gl]) && [@gl] <> 0)
VAR _forScatter = FILTER(_hasLY, NOT ISBLANK([@gv]) && [@gv] <> 0)
VAR _withGrowth = ADDCOLUMNS(_forScatter, "@grp",
    DIVIDE([@gl] - [@gv], ABS([@gv])) * 100)

-- Outlier detection: 3-sigma on both axes
VAR _cnt   = COUNTROWS(_withGrowth)
VAR _meanP = AVERAGEX(_withGrowth, [@gl])
VAR _meanG = AVERAGEX(_withGrowth, [@grp])
VAR _stdP  = IF(_cnt > 1, SQRT(SUMX(_withGrowth, ([@gl] - _meanP)^2) / (_cnt-1)), 1)
VAR _stdG  = IF(_cnt > 1, SQRT(SUMX(_withGrowth, ([@grp] - _meanG)^2) / (_cnt-1)), 1)

-- Main scatter: within 3-sigma on both axes
VAR _clean = FILTER(_withGrowth,
    [@gl] <= _meanP + 3*_stdP &&
    [@grp] <= _meanG + 3*_stdG &&
    [@grp] >= _meanG - 3*_stdG)

-- Outliers: outside 3-sigma or missing PY
VAR _outliers = FILTER(_withGrowth,
    [@gl] > _meanP + 3*_stdP ||
    [@grp] > _meanG + 3*_stdG ||
    [@grp] < _meanG - 3*_stdG)
VAR _noVJ = FILTER(_hasLY, ISBLANK([@gv]) || [@gv] = 0)

-- Output: scatter rows JSON
VAR _rows = CONCATENATEX(_clean,
    "{""n"":""" & SUBSTITUTE([@n],"""","'") &
    """,""gl"":" & FORMAT(ROUND([@gl],0),"0") &
    ",""gv"":" & FORMAT(ROUND([@gv],0),"0") &
    ",""g2"":" & FORMAT(ROUND(IF(ISBLANK([@g2]),0,[@g2]),0),"0") &
    ",""nl"":" & FORMAT(ROUND(IF(ISBLANK([@nl]),0,[@nl]),0),"0") &
    ",""nv"":" & FORMAT(ROUND(IF(ISBLANK([@nv]),0,[@nv]),0),"0") &
    ",""n2"":" & FORMAT(ROUND(IF(ISBLANK([@n2]),0,[@n2]),0),"0") & "}",
    ",")

-- Outlier rows JSON
VAR _outRows = CONCATENATEX(
    UNION(
        SELECTCOLUMNS(_outliers, "@n",[@n],"@gl",[@gl],"@gv",[@gv],"@g2",[@g2],
                      "@nl",[@nl],"@nv",[@nv],"@n2",[@n2],"@grp",[@grp]),
        SELECTCOLUMNS(_noVJ,    "@n",[@n],"@gl",[@gl],"@gv",[@gv],"@g2",[@g2],
                      "@nl",[@nl],"@nv",[@nv],"@n2",[@n2],"@grp",0.0)
    ),
    "{""n"":""" & SUBSTITUTE([@n],"""","'") &
    """,""gl"":" & FORMAT(ROUND([@gl],0),"0") & ",...}", ",")
```

### C2 — Data injected into JS

```js
var D    = [<<_rows>>];      // scatter main data
var OUT  = [<<_outRows>>];   // outlier data
var GTMAIN = <<_allMainLy>>; // grand total (current year, main metric)
var GTALT  = <<_allAltLy>>;
var OUTMAIN = <<sum outliers main>>;
var OUTALT  = <<sum outliers alt>>;
var OUTCNT  = <<count outliers>>;
var LY = '<<_ly>>', PY = '<<_py>>', PY2 = '<<_py2>>';
```

---

## SECTION D — JAVASCRIPT ARCHITECTURE

### D1 — Global variables

```js
var D, OUT;                  // data arrays (from DAX)
var T = 'g';                 // metric toggle: 'g'=main, 'n'=alt
var XS = 0.5, YS = 0.5;     // quadrant divider positions (0–1 fraction)
var LXM = 'none';            // x-line mode: 'none'/'avgp'/'medp'
var LYM = 'avg';             // y-line mode: 'none'/'avg'/'med'
var SHOWNEG = true;          // show negative main-metric dots
var DR = null;               // drag state: 'x'/'y'/null
var CP = [];                 // current plot points (filtered)
var SC = {};                 // scale cache: {mxP, yMn, yMx, pw, ph, W, H, sx, sy}
var avgX, avgY, medX, medY;  // computed statistics
```

### D2 — Canvas drawing

```js
function draw() {
  var cv = document.getElementById('cv');
  // 1. Resize canvas to fill window minus toolbar/KPI bar heights
  // 2. Compute SC: mxP, yMn, yMx from data
  // 3. Draw quadrant background rectangles (4 colours, low opacity)
  // 4. Draw grid lines and axis labels
  // 5. Draw axis titles (X: entity value label, Y: Growth % vs PY)
  // 6. Draw quadrant name labels (semi-transparent)
  // 7. Draw dots for each entity in D[] (colour by quadrant)
  // 8. Draw "No Mapping" dot (grey, larger, 'NM' label)
  // 9. Draw draggable divider lines (dashed, with triangle handles)
  // 10. Draw avg/median reference lines if LXM/LYM active
  // 11. Update KPI strip with quadrant totals
}
```

### D3 — Quadrant classification

```js
function qi(p, g, xD, yD) {
  // Returns: 0=bottom-left, 1=bottom-right, 2=top-left, 3=top-right
  return p <= xD && g <= yD ? 0 :
         p  > xD && g <= yD ? 1 :
         p <= xD && g  > yD ? 2 : 3;
}
```

Quadrant colours: `QC = ['#d9534f','#0070c0','#e5a000','#22a34a']` (bottom-left red, bottom-right blue, top-left amber, top-right green).

### D4 — Draggable divider lines

X-line: dashed vertical line; draggable horizontally (mousedown when within 14px of line).
Y-line: dashed horizontal line; draggable vertically.

On drag:
```js
document.addEventListener('mousemove', function(e) {
  if (!DR || !SC.mxP) return;
  var r = cv.getBoundingClientRect();
  if (DR === 'x') XS = clamp((e.clientX - r.left - SC.M.l) / SC.pw, 0.05, 0.95);
  else            YS = clamp(1 - (e.clientY - r.top - SC.M.t) / SC.ph, 0.05, 0.95);
  _sv('xs', XS); _sv('ys', YS);
  draw();
});
```

### D5 — KPI strip

Below toolbar, above canvas. One card per quadrant + one "Outliers" card:

```html
<div class='ki' onclick='openModal(q)'>
  <div class='klb'><span class='kd' style='background:<<QC[q]>>'></span><<QN[q]>></div>
  <div class='kv'><<total>></div>
  <div class='ks'><<count>> entities <<pct>>%</div>
  <div class='ki-tip'>📋 Details</div>
</div>
```

### D6 — Modal drill-down table

Per quadrant, clicking the KPI card opens a modal with all entities in that quadrant:
Columns: Entity Name | PY-2 | PY | CY | Growth CY/PY | Growth CY/PY-2

Outliers modal shows entities flagged by 3-sigma, with their actual growth % displayed.

```js
function openModal(q) {
  // q = -1 for outliers, 0–3 for quadrants
  // Filter CP (main) or OUT (outliers)
  // Sort by CY value descending
  // Render table in #modal
  // Show #overlay
}
```

### D7 — Toolbar controls

```
[Metric toggle: Main | Alt]
[Sep]
[Y-line: Avg Growth | Med Growth]
[X-line: Avg Value | Med Value]
[Sep]
[Neg Value toggle]
[Sep]
[X threshold input: custom EUR/unit value]
[Y threshold input: custom % value]
[Apply button]
```

Custom threshold inputs:
- On apply, compute XS = threshold / SC.mxP; YS = 1 - (threshold% - SC.yMn) / (SC.yMx - SC.yMn)
- Save to state

### D8 — Hover tooltip

On `mousemove` over canvas (when not dragging):
1. Find the nearest dot within 10px radius.
2. If found: show tooltip with entity name, metric value, growth %, quadrant name.
3. Change cursor to `crosshair` normally; `ew-resize` near X-line; `ns-resize` near Y-line.

### D9 — Avg/Median stats display

Sidebar panel or inline below KPI strip:
- X axis: Avg value, Median value
- Y axis: Avg growth, Median growth
- Update on every draw() call.

### D10 — State persistence

```js
// State keys:
// 't'    toggle: 'g'=main metric, 'n'=alt metric
// 'xs'   X-divider position (0–1)
// 'ys'   Y-divider position (0–1)
// 'lxm'  X-line mode: 'none'/'avgp'/'medp'
// 'lym'  Y-line mode: 'none'/'avg'/'med'
// 'neg'  show negatives: true/false
// State prefix: <<STPREFIX>> = lowercase table fragment + 'sc_'
```

---

## SECTION E — VALIDATION AND TESTING

### E1 — Pre-deploy JS audit
- [ ] `draw()` called at end: `draw(); setTimeout(draw,50); setTimeout(draw,250);`
  (triple call ensures canvas resize is complete on initial render)
- [ ] `window.addEventListener('resize', draw)` present
- [ ] `DR=null` initialized before event listeners
- [ ] Outlier computation: `_clean` and `_outliers` are both populated
- [ ] Modal close on `Escape` key and overlay click
- [ ] No ternary `=` (must be `:`)
- [ ] No DAX reserved VAR names

### E2 — Live test
```dax
EVALUATE ROW("len", LEN([<<MEASURE_NAME>>]), "start", LEFT([<<MEASURE_NAME>>], 120))
```
Pass: `len > 2000`, starts with `<!DOCTYPE`.

### E3 — Troubleshooting

| Symptom | Fix |
|---|---|
| No dots rendered | Check `D.length > 0` in JS; confirm YEAR_COL/MONTH_COL filter |
| All dots in one quadrant | Divider at extreme: reset XS=0.5, YS=0.5 |
| "No Mapping" dot missing | Verify residual calculation `_nmGl > 0` |
| Modal is empty | `SC.xD` not set; ensure `draw()` runs before `openModal()` |
| Drag not working | `mousedown` listener on `cv` not `document` for initial detection |
| Outliers count wrong | Check `_noVJ` filter — entities with PY=0 should be in outliers |

---

## SECTION F — USER REPORT

On success:

> "Done! **<<MEASURE_NAME>>** is ready.
>
> **To use:** Add HTML Viewer visual → drag measure into Values.
>
> **Features:**
> - Every <<entity type>> is plotted as a dot: X = <<LABEL_MAIN>>, Y = Year-over-Year Growth %
> - 4 quadrants: <<QUAD_LABELS[3]>> | <<QUAD_LABELS[2]>> | <<QUAD_LABELS[1]>> | <<QUAD_LABELS[0]>>
> - Drag the dashed lines to move the quadrant boundaries
> - Click any KPI card to see all entities in that quadrant
> - Toggle between <<LABEL_MAIN>> and <<LABEL_ALT>> with the metric buttons
> - Outliers (3-sigma) are excluded from the main chart but shown in the Outliers card
> - Avg/Median reference lines toggle on/off
> - Custom threshold: type a value in the X/Y inputs and click Apply"
