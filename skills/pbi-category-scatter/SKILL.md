---
name: pbi-category-scatter
description: >
  Builds a fully interactive 4-quadrant bubble scatter plot for category-level analysis as a DAX
  measure in Power BI. Bubbles represent category groups (e.g. industry sectors, product segments,
  geographic regions). X-axis shows total metric value, Y-axis shows YoY growth %. Bubble size
  scales as square root of value. Features: right detail panel with Top-N sub-entity records table
  plus secondary entity summary, 12-month sparklines, plan gauge, period selector (YTD/FY1/FY2),
  Top-N selector (10/20/all), frame-based entry animation (requestAnimationFrame), 3-sigma outlier
  exclusion, draggable quadrant threshold lines, category label rendering, custom X/Y thresholds,
  and persistent state. Works on any dataset in any industry.
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

# /pbi-category-scatter — Category Bubble Scatter Plot for Power BI

Produces an interactive **bubble scatter plot** where each bubble represents a category (e.g. industry
sector, product group, region), plotted by total metric value (X) and year-over-year growth (Y).
Clicking a bubble opens a right-side detail panel showing individual records and a secondary entity
summary within that category.

Works on **any dataset** with a category dimension, a year column, a month column, numeric metrics,
and sub-entity records. No domain-specific assumptions.

Arguments: `$ARGUMENTS`

---

## SECTION A — MANDATORY BEHAVIOUR RULES

### A1 — Connect, inspect, version, test
Same discipline: connect → schema → build → new version → validate.

### A2 — Column names from schema only
Zero guessing.

### A3 — Always increment version
Check `list_measures`. Auto-increment.

### A4 — Reserved DAX VAR names — NEVER use
`Table`, `Filter`, `Values`, `All`, `Calculate`, `Row`, `Date`, `Time`, `Today`, `Now`,
`DataTable`, `Calendar`, `Union`, `Intersect`, `Except`, `Generate`, `AddColumns`,
`SelectColumns`, `Summarize`, `CrossJoin`, `Distinct`

### A5 — Animation is implemented via requestAnimationFrame, not CSS transitions
The bubble entry animation uses a numeric counter `AP` (Animation Progress, 0→1) that is incremented
in each `requestAnimationFrame` callback. Bubble sizes are scaled by `AP` during animation.
Never use CSS `transition` on canvas elements.

### A6 — TOPN for sub-entity detail is split into two separate queries
Query 1: TOPN(200) primary sub-entity rows (sorted by CY metric DESC).
Query 2: TOPN(250) secondary entity rows.
Both are serialised as CONCATENATEX. Use TOPN, not FILTER with ranking, to avoid circular dependencies.

---

## SECTION B — DISCOVERY PHASE

### B1 — Data table
`list_tables` → ask user. Save as **TABLE**.

### B2 — Schema inspection
`get_table_schema(TABLE)`. Classify:
- **YEAR_COL**: integer 4-digit year
- **MONTH_COL**: integer 1–12
- **CATEGORY_COL**: text column whose distinct values become bubbles (recommended: 5–25 distinct values)
- **ENTITY_COL**: sub-entity identifier shown in the right detail table (e.g. Company, Customer, Deal ID)
- **SECONDARY_ENTITY_COL** (optional): secondary entity type shown in a second detail table (e.g. Sales Rep, Agent)
- **METRIC_A**: main additive numeric metric (size + X-axis)
- **METRIC_B** (optional): second metric to toggle to
- **MONTHLY_METRIC** (optional): metric column to use for monthly sparkline data

### B3 — Year, month, and period
Ask for YEAR_COL and MONTH_COL.

> "How many time periods should the visual support?
> 1. YTD only
> 2. YTD + 1 prior full year (FY1)
> 3. YTD + 2 prior full years (FY1 + FY2)"
Save as **PERIOD_COUNT** (1/2/3).

### B4 — Categories (bubbles)

> "Which column defines the **category groups** for the bubbles?
> (Each unique value becomes one bubble — recommended: 5–25 groups)"
Save as **CATEGORY_COL**.

> "How many categories should be shown by default?
> 1. Top 10 (by current-year metric, default)
> 2. Top 20
> 3. All (not recommended if > 30 categories)"
Save as **DEFAULT_TOPN** (10/20/9999).

> "Should the visual show **category labels** directly on the bubbles?
> 1. Yes — always show labels
> 2. Yes — show labels only on hover
> 3. No — show labels only in tooltip"
Save as **LABEL_MODE** ('always'/'hover'/'tooltip').

### B5 — Metrics

> "Which column is the **main metric**? This determines both bubble size and X-axis position.
> (e.g. Total Revenue, Deal Count, Premium Volume)"
Save as **METRIC_A**. Label → **LABEL_A**.

> "Is there a **second metric** to toggle to?
> 1. Yes — pick it
> 2. No"
Save as **METRIC_B** (or null). Label → **LABEL_B**.

> "Which column should be used for the **12-month sparklines** in the detail panel?
> (Same as main metric is fine — just confirm)
> 1. Same as main metric: <<METRIC_A>>
> 2. Different column"
Save as **SPARKLINE_METRIC** (usually = METRIC_A).

### B6 — Detail panel data

> "Which column identifies **individual records** shown in the right detail panel?
> (e.g. Deal Name, Customer Name, Contract ID)"
Save as **ENTITY_COL**.

Ask for up to 4 additional display columns for the entity table → **ENTITY_EXTRA_COLS**.

> "Is there a **secondary entity type** to show in a second table below the first?
> (e.g. if primary entities are Deals, secondary might be Sales Reps)
> 1. Yes — pick a column
> 2. No"
Save as **SEC_ENTITY_COL** (or null). Label → **SEC_ENTITY_LABEL**.

### B7 — Plan/target data

> "Is there a **plan/budget/target table** for the speedometer gauge?
> 1. Yes
> 2. No"
If yes: ask for PLAN_TABLE, PLAN_METRIC_COL, PLAN_YEAR_COL, PLAN_CAT_COL.
Save as **PLAN** object. If no: **PLAN = null**.

### B8 — Quadrant labels (same as pbi-entity-scatter)
Ask for 4 quadrant names or accept defaults: Star Performers, Growth Potential, Cash Cows, Action Required.
Save as **QUAD_LABELS**.

### B9 — Measure name
Ask, auto-increment, save as **MEASURE_NAME**.

---

## SECTION C — DAX ARCHITECTURE

### C1 — Category bubble data

```dax
VAR _ly  = CALCULATE(MAX(TABLE[YEAR_COL]), ALL(TABLE))
VAR _py  = _ly - 1
VAR _py2 = _ly - 2
VAR _mm  = CALCULATE(MAX(TABLE[MONTH_COL]), ALL(TABLE[YEAR_COL]), ALL(TABLE[MONTH_COL]))

-- Grand totals for % of portfolio
VAR _allLy = CALCULATE(SUM(TABLE[METRIC_A]),
    ALL(TABLE[YEAR_COL]), ALL(TABLE[MONTH_COL]),
    TABLE[YEAR_COL]=_ly, TABLE[MONTH_COL]<=_mm)
VAR _allPy = CALCULATE(SUM(TABLE[METRIC_A]),
    ALL(TABLE[YEAR_COL]), ALL(TABLE[MONTH_COL]),
    TABLE[YEAR_COL]=_py, TABLE[MONTH_COL]<=_mm)
-- _allFy1, _allFy2 (full year, no month filter)

-- Per-category summary
VAR _catBase = CALCULATETABLE(
    SUMMARIZE(TABLE, TABLE[CATEGORY_COL]),
    ALL(TABLE[YEAR_COL]), ALL(TABLE[MONTH_COL]))

VAR _catData = ADDCOLUMNS(_catBase,
    "@ly",  CALCULATE(SUM(TABLE[METRIC_A]), ALL(TABLE[YEAR_COL]), ALL(TABLE[MONTH_COL]),
                TABLE[YEAR_COL]=_ly, TABLE[MONTH_COL]<=_mm),
    "@py",  CALCULATE(SUM(TABLE[METRIC_A]), ALL(TABLE[YEAR_COL]), ALL(TABLE[MONTH_COL]),
                TABLE[YEAR_COL]=_py, TABLE[MONTH_COL]<=_mm),
    "@fy1", CALCULATE(SUM(TABLE[METRIC_A]), ALL(TABLE[YEAR_COL]), ALL(TABLE[MONTH_COL]),
                TABLE[YEAR_COL]=_py),                     -- full prior year
    "@fy2", CALCULATE(SUM(TABLE[METRIC_A]), ALL(TABLE[YEAR_COL]), ALL(TABLE[MONTH_COL]),
                TABLE[YEAR_COL]=_py2),                    -- full PY-2
    "@nl",  CALCULATE(SUM(TABLE[METRIC_B]), ...),          -- if METRIC_B != null
    "@nv",  CALCULATE(SUM(TABLE[METRIC_B]), ...),
    "@cntLy", CALCULATE(COUNTROWS(TABLE), ..., TABLE[YEAR_COL]=_ly, TABLE[MONTH_COL]<=_mm),
    "@cntPy", CALCULATE(COUNTROWS(TABLE), ..., TABLE[YEAR_COL]=_py, TABLE[MONTH_COL]<=_mm),
    -- 12 monthly values for this category:
    "@m1",  CALCULATE(SUM(TABLE[SPARKLINE_METRIC]), ALL(TABLE[YEAR_COL]), ALL(TABLE[MONTH_COL]),
                TABLE[YEAR_COL]=_ly, TABLE[MONTH_COL]=1),
    -- ... "@m2" through "@m12" ...
    "@m12", CALCULATE(SUM(TABLE[SPARKLINE_METRIC]), ..., TABLE[MONTH_COL]=12)
)

-- Serialise: up to TOPN(60) categories (safety cap — scatter gets unreadable above ~40 bubbles)
VAR _catRows = CONCATENATEX(
    TOPN(60, _catData, [@ly], DESC),
    "{""c"":""" & SUBSTITUTE(COALESCE(TABLE[CATEGORY_COL],"(None)"),"""","'") &
    """,""ly"":" & FORMAT(COALESCE([@ly],0),"0") &
    ",""py"":" & FORMAT(COALESCE([@py],0),"0") &
    ",""fy1"":" & FORMAT(COALESCE([@fy1],0),"0") &
    ",""fy2"":" & FORMAT(COALESCE([@fy2],0),"0") &
    ",""nl"":" & FORMAT(COALESCE([@nl],0),"0") &
    ",""nv"":" & FORMAT(COALESCE([@nv],0),"0") &
    ",""clY"":" & FORMAT(COALESCE([@cntLy],0),"0") &
    ",""clP"":" & FORMAT(COALESCE([@cntPy],0),"0") &
    ",""ms"":[" & FORMAT(COALESCE([@m1],0),"0") & "," &
                  FORMAT(COALESCE([@m2],0),"0") & ",...," &
                  FORMAT(COALESCE([@m12],0),"0") & "]}",
    ","
)
```

### C2 — Sub-entity detail rows (TOPN 200)

```dax
VAR _detBase = CALCULATETABLE(
    SUMMARIZE(TABLE, TABLE[ENTITY_COL], TABLE[CATEGORY_COL], <<ENTITY_EXTRA_COLS>>),
    ALL(TABLE[YEAR_COL]), ALL(TABLE[MONTH_COL]),
    TABLE[YEAR_COL]=_ly, TABLE[MONTH_COL]<=_mm)

VAR _detData = ADDCOLUMNS(_detBase,
    "@v",   CALCULATE(SUM(TABLE[METRIC_A]), ALL(TABLE[YEAR_COL]), ALL(TABLE[MONTH_COL]),
                TABLE[YEAR_COL]=_ly, TABLE[MONTH_COL]<=_mm),
    "@vp",  CALCULATE(SUM(TABLE[METRIC_A]), ALL(TABLE[YEAR_COL]), ALL(TABLE[MONTH_COL]),
                TABLE[YEAR_COL]=_py, TABLE[MONTH_COL]<=_mm),
    "@ms",  "[" & FORMAT(CALCULATE(SUM(TABLE[SPARKLINE_METRIC]),...,TABLE[MONTH_COL]=1),"0") &
            ",...," & FORMAT(CALCULATE(SUM(TABLE[SPARKLINE_METRIC]),...,TABLE[MONTH_COL]=12),"0") & "]"
)

VAR _detRows = CONCATENATEX(
    TOPN(200, _detData, [@v], DESC),
    "{""id"":""" & SUBSTITUTE(TABLE[ENTITY_COL],"""","'") &
    """,""cat"":""" & SUBSTITUTE(TABLE[CATEGORY_COL],"""","'") &
    -- extra display cols
    """,""v"":" & FORMAT(COALESCE([@v],0),"0") &
    ",""vp"":" & FORMAT(COALESCE([@vp],0),"0") &
    ",""ms"":" & COALESCE([@ms],"[]") & "}",
    ","
)
```

### C3 — Secondary entity rows (TOPN 250)

```dax
VAR _secBase = CALCULATETABLE(
    SUMMARIZE(TABLE, TABLE[SEC_ENTITY_COL], TABLE[CATEGORY_COL]),
    ALL(TABLE[YEAR_COL]), ALL(TABLE[MONTH_COL]),
    TABLE[YEAR_COL]=_ly, TABLE[MONTH_COL]<=_mm)

VAR _secData = ADDCOLUMNS(_secBase,
    "@sv",  CALCULATE(SUM(TABLE[METRIC_A]), ...),
    "@svp", CALCULATE(SUM(TABLE[METRIC_A]), ...),
    "@cnt", CALCULATE(COUNTROWS(TABLE), ...)
)

VAR _secRows = CONCATENATEX(
    TOPN(250, _secData, [@sv], DESC),
    "{""id"":""" & SUBSTITUTE(TABLE[SEC_ENTITY_COL],"""","'") &
    """,""cat"":""" & SUBSTITUTE(TABLE[CATEGORY_COL],"""","'") &
    """,""v"":" & FORMAT(COALESCE([@sv],0),"0") &
    ",""vp"":" & FORMAT(COALESCE([@svp],0),"0") &
    ",""cnt"":" & FORMAT(COALESCE([@cnt],0),"0") & "}",
    ","
)
```

---

## SECTION D — JAVASCRIPT ARCHITECTURE

### D1 — Layout

```
+------------------------------------------+-----------------------------+
| TOOLBAR                                  | (right panel, initially     |
| [Period: YTD|FY1|FY2] [Metric: A|B]     |  hidden, slides in on       |
| [Top-N: 10|20|All] [Labels: On|Off]      |  bubble click)              |
| [X-line|Y-line toggles] [?]              |                             |
+------------------------------------------+                             |
| KPI strip: Bubble count | Portfolio total |                             |
+------------------------------------------+                             |
| CANVAS (fills remaining height)          |   <<CATEGORY>> Details      |
|                                          |   [×]                       |
|   [bubble scatter, animated on load]     |   Plan gauge (if PLAN)      |
|                                          |   ---                       |
|                                          |   Top-N <<ENTITY_COL>>      |
|                                          |   [scrollable table]        |
|                                          |   ---                       |
|                                          |   Top-N <<SEC_ENTITY_COL>>  |
|                                          |   [scrollable table]        |
+------------------------------------------+-----------------------------+
```

### D2 — Bubble size formula

```js
var mnR = 6;    // minimum bubble radius (px)
var mxR = 48;   // maximum bubble radius (px)
var mxC = Math.max.apply(null, CP.map(function(p){ return p.v; }));

function br(v) {
  if (!v || v <= 0) return mnR;
  return mnR + Math.sqrt(v / mxC) * (mxR - mnR);
}
```

Using **square root** ensures visual area is proportional to value, not radius.
`mxC` is updated whenever the period or metric type changes.

### D3 — Animation (entry)

```js
var AP = 0;      // animation progress: 0 → 1
var AF = null;   // requestAnimationFrame handle
var ASPD = 0.035; // animation speed per frame

function animate() {
  AP = Math.min(AP + ASPD, 1);
  draw();
  if (AP < 1) AF = requestAnimationFrame(animate);
}

function startAnim() {
  AP = 0;
  if (AF) cancelAnimationFrame(AF);
  AF = requestAnimationFrame(animate);
}
```

During animation, bubble radius is scaled: `br(v) * AP`.

### D4 — Canvas drawing function

```js
function draw() {
  var cv = document.getElementById('cv');
  var ctx = cv.getContext('2d');
  var W = cv.width, H = cv.height;

  ctx.clearRect(0, 0, W, H);

  // Compute data for current period/metric
  var pts = buildPoints();  // returns [{c, v, g, r, q, m12}, ...]

  // 3-sigma outlier detection (identical to pbi-entity-scatter)
  // SC: computed scale object {mxP, yMn, yMx, pw, ph, M}

  // 1. Draw quadrant background rects
  // 2. Draw grid lines and axis labels
  // 3. Draw axis titles
  // 4. Draw quadrant name labels (semi-transparent, larger font)
  // 5. Draw avg/median reference lines (if toggled)
  // 6. Draw draggable divider lines (dashed)
  // 7. Draw bubbles: for each pt in pts (sorted by size desc, so small bubbles on top)
  //    ctx.beginPath(); ctx.arc(sx(pt.v), sy(pt.g), br(pt.v) * AP, 0, 2*Math.PI);
  //    ctx.fillStyle = QC[pt.q] with 0.72 alpha
  //    ctx.strokeStyle = QC[pt.q] at full opacity, lineWidth=1.5
  //    ctx.fill(); ctx.stroke();
  // 8. Draw labels on bubbles (if LABEL_MODE='always' or hover)
  //    ctx.fillStyle = '#fff' or dark depending on bubble colour
  //    clip label to bubble radius
  // 9. Update KPI strip
}
```

### D5 — Point building

```js
function buildPoints() {
  var period = S.ym;  // 'ytd'/'fy1'/'fy2'
  var metric = S.t;   // 'g'/'n'
  var topN   = S.n;   // 10/20/9999

  var pts = CAT.map(function(c) {
    var v  = metric==='g' ? (period==='ytd' ? c.ly : period==='fy1' ? c.fy1 : c.fy2)
                          : (period==='ytd' ? c.nl : c.nfy1);
    var vp = metric==='g' ? (period==='ytd' ? c.py : c.fy2) : c.nv;
    var g  = vp !== 0 ? (v - vp) / Math.abs(vp) * 100 : 0;
    return {c: c.c, v: v, vp: vp, g: g, m12: c.ms, cnt: c.clY};
  });

  // Outlier exclusion (3-sigma, same pattern as pbi-entity-scatter)
  pts = applyOutlierExclusion(pts);

  // Top-N filter
  pts.sort(function(a,b){ return b.v - a.v; });
  if (topN < 9999) pts = pts.slice(0, topN);

  // Assign quadrant
  pts.forEach(function(p) {
    p.q = qi(p.v, p.g, S.xD, S.yD);
  });

  CP = pts;
  return pts;
}
```

### D6 — Coordinate mapping

```js
// World → canvas pixel
function sx(v) {
  return SC.M.l + (v / SC.mxP) * SC.pw;
}
function sy(g) {
  return SC.M.t + (1 - (g - SC.yMn) / (SC.yMx - SC.yMn)) * SC.ph;
}
```

Margin object `SC.M = {l:64, r:20, t:20, b:40}` for axis labels.
`SC.mxP` = max value in data × 1.08 (5% headroom).
`SC.yMn` / `SC.yMx` = round to nice increments.

### D7 — Right detail panel

Slides in from the right (CSS `transform: translateX(100%)` → `translateX(0)`).
Width: `min(420px, 36vw)`.

**Header:** Category name + [×] close button.

**Plan gauge** (if PLAN != null):
- One speedometer for the selected category
- Shows plan target vs actual for CY
- Identical implementation to `drawSp()` in `pbi-tree-hierarchy`

**Primary entity table** (TOPN 200 rows from `DT` filtered by selected category):
- Columns: ENTITY_COL | ENTITY_EXTRA_COLS | CY | PY | Var% | Sparkline
- Each row has an inline sparkline canvas (56×22px) drawn from `pt.ms[0..11]`
- Sorted by CY value descending
- Sticky header row
- Scrollable (max-height: 40vh)

**Secondary entity table** (TOPN 250 from `ST` filtered by selected category):
- Columns: SEC_ENTITY_COL | CY | PY | Var% | Count
- Same visual style as primary table

```js
function openPanel(catName) {
  S.sel = catName;
  _sv('sel', catName);

  var dtRows = DT.filter(function(r){ return r.cat === catName; });
  var stRows = ST.filter(function(r){ return r.cat === catName; });
  var catObj = CAT.find(function(c){ return c.c === catName; });

  // render plan gauge (if PLAN)
  // render primary table from dtRows
  // render secondary table from stRows (if SEC_ENTITY_COL != null)
  // draw sparklines on all mini-canvas elements
  document.getElementById('rp').classList.add('open');
}
```

### D8 — Sparkline drawing in detail table

```js
function drawSpark(ctx, W, H, ms, isPos) {
  var mx = Math.max.apply(null, ms);
  var mn = Math.min.apply(null, ms);
  if (mx === mn) { ctx.clearRect(0,0,W,H); return; }
  var pad = 2;
  var xs = (W - pad*2) / (ms.length - 1);
  ctx.clearRect(0,0,W,H);
  ctx.strokeStyle = isPos ? '#16a34a' : '#dc2626';
  ctx.lineWidth = 1.5;
  ctx.beginPath();
  ms.forEach(function(v, i) {
    var x = pad + i * xs;
    var y = pad + (1 - (v - mn) / (mx - mn)) * (H - pad*2);
    if (i === 0) ctx.moveTo(x, y); else ctx.lineTo(x, y);
  });
  ctx.stroke();
}
```

### D9 — Toolbar controls

```
[Period pills: YTD | FY1 | FY2]      (show only if PERIOD_COUNT >= 2)
[Metric pills: <<LABEL_A>> | <<LABEL_B>>] (show only if METRIC_B != null)
[Top-N: 10 | 20 | All]
[Sort: By Value | By Growth]
[X-line: None | Avg | Median]
[Y-line: None | Avg | Median]
[Labels: Off | Hover | Always]
[Tooltips: On | Off]
[Min Value filter: input + Apply]    (hide categories below a min threshold)
[Custom X threshold: input + Apply]
[Custom Y threshold: input + Apply]
[?]                                  (help overlay)
```

### D10 — State persistence

```js
// State keys:
// 't'     metric: 'g'/'n'
// 'ym'    period: 'ytd'/'fy1'/'fy2'
// 'n'     top-N count: 10/20/9999
// 'sort'  sort: 'v'=by value, 'g'=by growth
// 'lxm'   x-line mode: 'none'/'avg'/'med'
// 'lym'   y-line mode: 'none'/'avg'/'med'
// 'cxd'   custom X threshold (number or null)
// 'cyd'   custom Y threshold (number or null)
// 'tip'   tooltips on/off (boolean)
// 'minp'  minimum value filter threshold (number, default 0)
// 'sel'   selected category name for detail panel (string or null)
// 'lbl'   label mode: 'off'/'hover'/'always'
// State prefix: <<STPREFIX>> = lowercase table fragment + 'ns_'
//   (Note: 'cat_sc_' pattern used in reference implementation;
//    always use a unique prefix to avoid collisions with other visuals)
```

---

## SECTION E — VALIDATION AND TESTING

### E1 — Pre-deploy JS audit
- [ ] No DAX reserved VAR names
- [ ] `draw()` called as: `startAnim();` (never `draw()` directly on first load)
- [ ] `window.addEventListener('resize', function(){ startAnim(); })` present
- [ ] `AP` initialized to 0 before `requestAnimationFrame(animate)` call
- [ ] `cancelAnimationFrame(AF)` called before starting a new animation
- [ ] Detail panel close button calls `closePanel()` which sets `S.sel=null` and `_sv('sel',null)`
- [ ] Sparkline canvases drawn after panel HTML is injected into DOM (use `setTimeout(drawSparks,10)`)
- [ ] `buildPoints()` returns empty array gracefully if CAT is empty
- [ ] Outlier exclusion does not modify original `CAT` array — creates a new filtered array
- [ ] No ternary `=` (must be `:`)
- [ ] Final JS call: `startAnim();`

### E2 — Live test
```dax
EVALUATE ROW(
  "len",   LEN([<<MEASURE_NAME>>]),
  "start", LEFT([<<MEASURE_NAME>>], 120),
  "cats",  LEN("<<_catRows>>"),
  "dets",  LEN("<<_detRows>>")
)
```
Pass: `len > 3000`, starts with `<!DOCTYPE`.
`cats > 10` confirms category data is populated.
`dets > 10` confirms detail rows are populated.

### E3 — Troubleshooting

| Symptom | Fix |
|---|---|
| No bubbles on canvas | `CAT` array empty — confirm `_catRows` not blank; test year/month filter |
| All bubbles same size | `mxC = 0` — check `buildPoints()` is not returning v=0 for all rows |
| Animation doesn't stop | `AP >= 1` condition in `animate()` missing — ensure `if(AP < 1)` guard |
| Sparklines not drawn | DOM not ready when `drawSparks()` called — add `setTimeout(drawSparks, 10)` |
| Panel doesn't close | `#rp.open` class not removed — check `closePanel()` implementation |
| Labels render outside bubble | Missing clip/crop — use `ctx.save(); ctx.beginPath(); ctx.arc(...); ctx.clip();` before drawing text |
| Outliers not excluded | `_withGrowth` includes rows where `g=0` (vp=0 handled as `g=0` not outlier) — add `ISBLANK(vp)` check |
| Secondary table missing | `SEC_ENTITY_COL = null` — ensure `openPanel()` checks for this before rendering second table |

---

## SECTION F — USER REPORT

On success:

> "Done! **<<MEASURE_NAME>>** is ready.
>
> **To use:** Add HTML Viewer visual → drag measure into Values.
>
> **Features:**
> - Each bubble represents a <<CATEGORY_COL>>: X-axis = <<LABEL_A>>, Y-axis = Year-over-Year Growth %
> - Bubble size = square root of value (so area is proportional to size)
> - 4 quadrants: <<QUAD_LABELS[3]>> | <<QUAD_LABELS[2]>> | <<QUAD_LABELS[1]>> | <<QUAD_LABELS[0]>>
> - Click any bubble to open the detail panel showing top-200 <<ENTITY_COL>> records + sparklines
> - <<if SEC_ENTITY_COL>>: Secondary table shows <<SEC_ENTITY_LABEL>> summary for that category
> - <<if PLAN>>: Plan achievement gauge appears in the detail panel
> - Period buttons: YTD / FY1 / FY2 to change the comparison basis
> - Top-N filter: show 10, 20, or all categories
> - Outliers (3-sigma) are excluded from the scatter to keep axes readable
> - Bubbles animate in on load and whenever the period/metric changes
> - All settings persist across filter changes"
