---
name: pbi-ytd-grouped-bar
description: >
  Builds a YTD vs Prior-Year grouped bar comparison visual as a DAX measure in Power BI.
  Displays a 3×2 SVG grid of stacked bar charts (one cell per Segment × Category combination),
  each showing PY YTD bar + CY YTD bar + optional FY prior-year bars with connecting variance
  arrows. Features: variance connectors (%, abs, arrow), plan speedometer gauges per segment,
  FY toggle buttons for multi-year comparison, delta slide-in panel with top-N positive/negative
  variance records from a 2000-row chunked detail dataset, segment filter pills, metric type switch,
  and full persistent state. Works on any dataset in any industry. Handles up to 2000 entity-level
  detail rows using a 10-chunk CONCATENATEX pattern to stay within DAX string length limits.
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

# /pbi-ytd-grouped-bar — YTD Grouped Bar Comparison Visual for Power BI

Produces a **YTD grouped bar chart grid** comparing current-year vs prior-year performance
across multiple segments and categories. Includes a delta drill-down panel showing which
individual entities drove the variance. Rendered as a single DAX measure via the HTML Viewer
custom visual.

Works on **any dataset** — no hardcoded segment names, category names, or column names.

Arguments: `$ARGUMENTS`

---

## SECTION A — MANDATORY BEHAVIOUR RULES

### A1 — Connect, inspect, version, validate
Same discipline as all other skills.

### A2 — Column names from schema only
Zero guessing.

### A3 — Always increment version
Check `list_measures`. Auto-increment.

### A4 — Reserved DAX VAR names — NEVER use
`Table`, `Filter`, `Values`, `All`, `Calculate`, `Row`, `Date`, `Time`, `Today`, `Now`,
`DataTable`, `Calendar`, `Union`, `Intersect`, `Except`, `Generate`, `AddColumns`,
`SelectColumns`, `Summarize`, `CrossJoin`, `Distinct`

### A5 — The 10-chunk CONCATENATEX pattern is mandatory for entity detail rows
The detail dataset targets 2000 rows. A single CONCATENATEX over 2000 rows often exceeds DAX's
internal string buffer. Split into 10 chunks of 200 rows each by RANKX, then join conditionally
in the RETURN string. Never attempt a single CONCATENATEX over more than 500 rows.

### A6 — Detail row JSON must be single-line arrays
Each row in the detail dataset must be serialized as a flat JSON array:
`["id","cat","seg",123456,110000,100000]`
NOT as an object with named keys. This reduces string size significantly.

---

## SECTION B — DISCOVERY PHASE

### B1 — Data table
`list_tables` → ask user. Save as **TABLE**.

### B2 — Schema inspection
`get_table_schema(TABLE)`. Classify:
- **YEAR_COL**: integer 4-digit year
- **MONTH_COL**: integer 1–12
- **ENTITY_COL**: individual record identifier (shown in delta panel)
- **CATEGORY_COL**: text column defining the ~3 categories (columns in the 3×2 grid)
- **SEGMENT_COL**: text column defining the ~2 segments (rows in the 3×2 grid)
- **METRIC_A**: main additive numeric metric (e.g. Total Revenue, Gross Premium)
- **METRIC_B**: optional second metric (e.g. New Business, Net Revenue)

> **Grid size note:** The visual is designed for exactly a 3-column × 2-row grid.
> If CATEGORY_COL has more than 3 distinct values, ask the user to select the top 3 to display.
> If SEGMENT_COL has more than 2 distinct values, ask the user to select the top 2.

### B3 — Year and month
Ask for YEAR_COL and MONTH_COL.

### B4 — Grid configuration

Ask for CATEGORY_COL and its 3 display values → **CATS** array `[{value, label}]` (3 items).
Ask for SEGMENT_COL and its 2 display values → **SEGS** array `[{value, label}]` (2 items).

If CATEGORY_COL has only 1–2 values: use those, leave remaining cells as totals or empty.
If SEGMENT_COL has only 1 value: use a 1-row grid; hide the segment row header.

> **All/Total row:** Ask:
> "Should the visual also include an 'All Categories' total row above the grid?
> 1. Yes — first row shows Portfolio/Total totals
> 2. No — show segment rows only"
Save as **SHOW_TOTAL_ROW** (boolean).

### B5 — Metrics

> "Which column is the **main metric** for the bars? (e.g. Total Revenue, Gross Premium)"
Save as **METRIC_A**. Label → **LABEL_A**.

> "Is there a **second metric** to toggle between? (e.g. New Business, Net Premium)
> 1. Yes — pick it
> 2. No"
Save as **METRIC_B** (or null). Label → **LABEL_B**.

### B6 — Prior year comparison bars (FY toggles)

> "How many prior full-year bars should be available as optional overlays?
> 1. None (YTD only)
> 2. 1 FY bar (full prior year FY1)
> 3. 2 FY bars (FY1 + FY2)"
Save as **FY_COUNT** (0/1/2).

### B7 — Plan data (speedometers)

> "Is there a **plan/budget table** in your model?
> 1. Yes
> 2. No — skip plan gauges"
If yes: ask for PLAN_TABLE, PLAN_METRIC_COL, PLAN_YEAR_COL, PLAN_MONTH_COL, PLAN_SEG_COL.
Save as **PLAN** object. If no: **PLAN = null**.

### B8 — Detail data for delta panel

> "The delta panel shows which individual <<ENTITY_COL>> values drove the variance.
> How many detail columns should appear in the delta table?
> (The entity ID, category, segment, CY value, and PY value are always included.)"
Ask for up to 3 additional display columns → **DELTA_EXTRA_COLS**.

### B9 — Measure name
Ask, auto-increment, save as **MEASURE_NAME**.

---

## SECTION C — DAX ARCHITECTURE

### C1 — High-level structure

```dax
-- Header scalars
VAR _cy = MAX(TABLE[YEAR_COL])
VAR _py = _cy - 1
VAR _py2 = _cy - 2
VAR _mm = IF(ISBLANK(MAX(TABLE[MONTH_COL])), 12, MAX(TABLE[MONTH_COL]))

-- ========================================================
-- COALESCE CALCULATE BLOCKS: ~100 scalars
-- Pattern: {cat} × {seg} × {metric} × {time_period}
-- Names: _g_{cat_key}_{seg_key}_{CY/PY/FY1/FY2}
-- ========================================================

-- Per-cell scalar: current year, metric A, category 1, segment 1, YTD:
VAR _g_c1_s1_cy = COALESCE(
    CALCULATE(SUM(TABLE[METRIC_A]),
        ALL(TABLE[YEAR_COL]), REMOVEFILTERS(<month_slicer>),
        TABLE[YEAR_COL] = _cy,
        TABLE[CATEGORY_COL] = "<<CATS[0].value>>",
        TABLE[SEGMENT_COL]  = "<<SEGS[0].value>>",
        FILTER(ALL(TABLE[MONTH_COL]), TABLE[MONTH_COL] <= _mm)
    ), 0)

-- Repeat: 3 cats × 2 segs × 2 metrics × 4 time periods = 48 scalars
-- + "All" totals: 2 segs × 2 metrics × 4 periods = 16 scalars
-- + Portfolio totals: 1 × 2 metrics × 4 periods = 8 scalars
-- Total: ~72 scalar VARs

-- Plan scalars (if PLAN != null): ~18 additional
-- ========================================================

-- ========================================================
-- ENTITY DETAIL ROWS: 2000 rows via 10-chunk pattern
-- ========================================================
VAR _rdRaw = ADDCOLUMNS(
    CALCULATETABLE(
        SUMMARIZE(TABLE, TABLE[ENTITY_COL], TABLE[CATEGORY_COL],
                  TABLE[SEGMENT_COL], <<DELTA_EXTRA_COLS>>),
        ALL(TABLE[YEAR_COL]), REMOVEFILTERS(<month_slicer>),
        FILTER(ALL(TABLE[YEAR_COL]), TABLE[YEAR_COL]=_cy || TABLE[YEAR_COL]=_py),
        FILTER(ALL(TABLE[MONTH_COL]), TABLE[MONTH_COL]<=_mm)
    ),
    "_cyg", CALCULATE(SUM(TABLE[METRIC_A]), ALL(TABLE[YEAR_COL]),
                      REMOVEFILTERS(<month_slicer>), TABLE[YEAR_COL]=_cy,
                      FILTER(ALL(TABLE[MONTH_COL]),TABLE[MONTH_COL]<=_mm)),
    "_pyg", CALCULATE(SUM(TABLE[METRIC_A]), ALL(TABLE[YEAR_COL]),
                      REMOVEFILTERS(<month_slicer>), TABLE[YEAR_COL]=_py,
                      FILTER(ALL(TABLE[MONTH_COL]),TABLE[MONTH_COL]<=_mm)),
    "_cyn", CALCULATE(SUM(TABLE[METRIC_B]), ALL(TABLE[YEAR_COL]),
                      REMOVEFILTERS(<month_slicer>), TABLE[YEAR_COL]=_cy,
                      FILTER(ALL(TABLE[MONTH_COL]),TABLE[MONTH_COL]<=_mm)),
    "_pyn", CALCULATE(SUM(TABLE[METRIC_B]), ALL(TABLE[YEAR_COL]),
                      REMOVEFILTERS(<month_slicer>), TABLE[YEAR_COL]=_py,
                      FILTER(ALL(TABLE[MONTH_COL]),TABLE[MONTH_COL]<=_mm))
)

-- Filter out rows with zero movement
VAR _rdFilt = FILTER(_rdRaw,
    ABS([_cyg] - [_pyg]) > 0 || ABS([_cyn] - [_pyn]) > 0)

-- Keep top 2000 by absolute variance (both metrics combined)
VAR _top2k = TOPN(2000, _rdFilt,
    ABS([_cyg] - [_pyg]) + ABS([_cyn] - [_pyn]), DESC)

-- Add rank for chunk splitting
VAR _rdRnk = ADDCOLUMNS(_top2k, "_rnk",
    RANKX(_top2k,
        ABS([_cyg]-[_pyg]) + ABS([_cyn]-[_pyn]),
        , DESC, Skip))

-- Precompute JSON string per row (single-line array format to minimize string size)
VAR _rdJs = ADDCOLUMNS(_rdRnk, "_js",
    "[""" & SUBSTITUTE(TABLE[ENTITY_COL],"""","'") &
    """,""" & SUBSTITUTE(TABLE[CATEGORY_COL],"""","'") &
    """,""" & SUBSTITUTE(TABLE[SEGMENT_COL],"""","'") &
    """," & FORMAT(COALESCE([_cyg],0),"0") &
    "," & FORMAT(COALESCE([_pyg],0),"0") &
    "," & FORMAT(COALESCE([_cyn],0),"0") &
    "," & FORMAT(COALESCE([_pyn],0),"0") & "]")

-- 10 chunks of 200 rows each
VAR _ch1  = FILTER(_rdJs, [_rnk]>=1    && [_rnk]<=200)
VAR _ch2  = FILTER(_rdJs, [_rnk]>=201  && [_rnk]<=400)
VAR _ch3  = FILTER(_rdJs, [_rnk]>=401  && [_rnk]<=600)
VAR _ch4  = FILTER(_rdJs, [_rnk]>=601  && [_rnk]<=800)
VAR _ch5  = FILTER(_rdJs, [_rnk]>=801  && [_rnk]<=1000)
VAR _ch6  = FILTER(_rdJs, [_rnk]>=1001 && [_rnk]<=1200)
VAR _ch7  = FILTER(_rdJs, [_rnk]>=1201 && [_rnk]<=1400)
VAR _ch8  = FILTER(_rdJs, [_rnk]>=1401 && [_rnk]<=1600)
VAR _ch9  = FILTER(_rdJs, [_rnk]>=1601 && [_rnk]<=1800)
VAR _ch10 = FILTER(_rdJs, [_rnk]>=1801 && [_rnk]<=2000)

-- CONCATENATEX per chunk
VAR _j1  = CONCATENATEX(_ch1,  [_js], ",")
VAR _j2  = CONCATENATEX(_ch2,  [_js], ",")
VAR _j3  = CONCATENATEX(_ch3,  [_js], ",")
VAR _j4  = CONCATENATEX(_ch4,  [_js], ",")
VAR _j5  = CONCATENATEX(_ch5,  [_js], ",")
VAR _j6  = CONCATENATEX(_ch6,  [_js], ",")
VAR _j7  = CONCATENATEX(_ch7,  [_js], ",")
VAR _j8  = CONCATENATEX(_ch8,  [_js], ",")
VAR _j9  = CONCATENATEX(_ch9,  [_js], ",")
VAR _j10 = CONCATENATEX(_ch10, [_js], ",")

-- ========================================================
-- RETURN — inject all data into HTML string
-- ========================================================
RETURN
"<!DOCTYPE html>..." & CSS &
"<script>" &
"const CY=<<_cy>>,PY=<<_py>>,PY2=<<_py2>>,MM=<<_mm>>;" &
"const SC={" &
  "...all 72+ scalar VARs serialised as named properties..." &
"};" &
"var RD=[" & _j1 &
    IF(LEN(COALESCE(_j2,""))>0, "," & _j2, "") &
    IF(LEN(COALESCE(_j3,""))>0, "," & _j3, "") &
    IF(LEN(COALESCE(_j4,""))>0, "," & _j4, "") &
    IF(LEN(COALESCE(_j5,""))>0, "," & _j5, "") &
    IF(LEN(COALESCE(_j6,""))>0, "," & _j6, "") &
    IF(LEN(COALESCE(_j7,""))>0, "," & _j7, "") &
    IF(LEN(COALESCE(_j8,""))>0, "," & _j8, "") &
    IF(LEN(COALESCE(_j9,""))>0, "," & _j9, "") &
    IF(LEN(COALESCE(_j10,""))>0,"," & _j10,"") &
"];" &
"...all JS...renderAll();</script></body></html>"
```

### C2 — SC (Scalar Constants) object

The `SC` object bundles all 72+ COALESCE CALCULATE scalars. Name convention:
```js
SC = {
  // g = METRIC_A, n = METRIC_B
  // c1/c2/c3 = CAT values 1–3, s1/s2 = SEG values 1–2, t = total
  // cy/py/fy1/fy2 = time periods
  g_c1_s1_cy: 12345, g_c1_s1_py: 11000, g_c1_s1_fy1: 10500, g_c1_s1_fy2: 9800,
  n_c1_s1_cy: 3456,  n_c1_s1_py: 3100,  ...
  g_c2_s1_cy: ..., ...
  // Totals per segment (across all cats):
  g_t_s1_cy: ..., g_t_s2_cy: ..., g_t_t_cy: ...
  // Plan (if PLAN != null):
  pl_s1_ytd: ..., pl_s1_fy: ..., pl_s2_ytd: ..., ...
}
```

---

## SECTION D — JAVASCRIPT ARCHITECTURE

### D1 — Layout: 3×2 SVG grid

```
+------+--------------------+--------------------+--------------------+
|      | <<CATS[0].label>>  | <<CATS[1].label>>  | <<CATS[2].label>>  |
+------+--------------------+--------------------+--------------------+
| [S1] | Cell (s1, c1)      | Cell (s1, c2)      | Cell (s1, c3)      |
+------+--------------------+--------------------+--------------------+
| [S2] | Cell (s2, c1)      | Cell (s2, c2)      | Cell (s2, c3)      |
+------+--------------------+--------------------+--------------------+
```

Each cell renders as an inline SVG showing the grouped bars for that Segment × Category.

### D2 — Single cell render function

```js
function renderCell(svgId, bars, labels, title, subTitle) {
  // bars = array of {height, color, label} objects for each bar
  // Connect bar 0 top to bar 1 top with a variance arrow:
  //   - Vertical stem from bar 0 top → horizontal bridge → vertical stem into bar 1 top
  //   - Blue oval at midpoint of bridge showing % change and absolute variance
  //   - Arrow head on the bar 1 stem side pointing downward
  // FY bars (if toggled) rendered as additional bars after bar 1 in 70% opacity

  var svg = document.getElementById(svgId);
  var W = svg.getBoundingClientRect().width;
  var H = svg.getBoundingClientRect().height;
  var mx = SC['g_'+catKey+'_'+segKey+'_cy'];
  var my = findMaxBar(bars);
  // ...draw bars using SVG rect elements
  // ...draw variance connector using SVG path elements
  // ...draw axis labels using SVG text elements
}
```

### D3 — Variance connector (SVG path)

```js
function vConnector(x1, y1, x2, y2, pct, abs) {
  // x1,y1 = top of bar 0; x2,y2 = top of bar 1
  var mx = (x1 + x2) / 2;
  var ht = Math.min(y1, y2) - 22;    // bridge height
  // path: M x1,y1 L x1,ht L x2,ht L x2,y2
  // oval: ellipse centred at (mx, ht-10) with rx=28,ry=13
  // arrow: triangular path at (x2, y2-4)
  // text: % and abs inside oval
  var col = pct >= 0 ? '#2563eb' : '#dc2626';
  return '<g class="vc">' +
    '<path d="M' + x1 + ',' + y1 + ' L' + x1 + ',' + ht +
                 ' L' + x2 + ',' + ht + ' L' + x2 + ',' + y2 + '"' +
           ' stroke="' + col + '" stroke-width="1.5" fill="none" stroke-dasharray="4 3"/>' +
    '<ellipse cx="' + mx + '" cy="' + ht + '" rx="28" ry="13"' +
           ' fill="' + col + '" opacity=".92"/>' +
    '<text x="' + mx + '" y="' + (ht-3) + '" text-anchor="middle"' +
          ' fill="#fff" font-size="9.5" font-weight="600">' + fmtPct(pct) + '</text>' +
    '<text x="' + mx + '" y="' + (ht+7) + '" text-anchor="middle"' +
          ' fill="#fff" font-size="8">' + fmtAbs(abs) + '</text>' +
    '</g>';
}
```

### D4 — Delta slide-in panel

The "Δ" button in the header opens a full-height panel from the right edge (860px wide):

```
+----------------------------------------------------------+
| Δ VARIANCE DETAIL — <<current segment or All>>    [×]    |
+----------------------------------------------------------+
| ▼ TOP <<N>> NEGATIVE VARIANCE (most negative first)      |
| Columns: ENTITY_COL | CAT | SEG | CY | PY | Δ (€) | Δ% |
| (scrollable, max height 40vh)                            |
+----------------------------------------------------------+
| ▲ TOP <<N>> POSITIVE VARIANCE (most positive first)      |
| Columns: same structure                                  |
| (scrollable, max height 40vh)                            |
+----------------------------------------------------------+
| Footer: Total CY | Total PY | Net Δ (€) | Net Δ%        |
+----------------------------------------------------------+
```

Data: filter `RD` (2000 entity rows) by segment (`RD[i][2]`) and metric type (`t='g'` → compare [3] vs [4]).
Sort negatives by `[3]-[4]` ascending (most negative first).
Sort positives by `[3]-[4]` descending (most positive first).

```js
function openDelta(seg, metric) {
  var negRows = RD.filter(function(r) {
    return (seg==='all' || r[2]===seg) && (r[3]-r[4]) < 0;
  }).sort(function(a,b) { return (a[3]-a[4]) - (b[3]-b[4]); });
  var posRows = RD.filter(function(r) {
    return (seg==='all' || r[2]===seg) && (r[3]-r[4]) > 0;
  }).sort(function(a,b) { return (b[3]-b[4]) - (a[3]-a[4]); });
  // render both sections into #delta-panel
  // show top-N of each (state: 'dn' = delta N, default 20)
  document.getElementById('delta').classList.add('open');
}
```

### D5 — Plan speedometer gauges

When PLAN != null, a collapsible side panel (or footer strip) shows one gauge per segment.
Implementation matches `drawSp()` from `pbi-tree-hierarchy`: semicircle canvas arc,
needle, colour pill, plan vs actual values.

### D6 — FY toggle buttons

```html
<button onclick="toggleFy(1)" class="fy-btn" id="fy1-btn">FY <<PY>></button>
<button onclick="toggleFy(2)" class="fy-btn" id="fy2-btn">FY <<PY2>></button>
```

State: `'fy1'` (boolean), `'fy2'` (boolean).
On toggle: update the `bars` array passed to `renderCell()` — add/remove FY bars.
FY bars are shown in 70% opacity of their segment colour.

### D7 — Segment filter pills

```html
<div class="pill-group">
  <button class="pill active" onclick="setSeg('all')">All</button>
  <button class="pill" onclick="setSeg('<<SEGS[0].value>>')"><<SEGS[0].label>></button>
  <button class="pill" onclick="setSeg('<<SEGS[1].value>>')"><<SEGS[1].label>></button>
</div>
```

When a specific segment is active: the grid shows only that segment's row (full-width cells).
When "All" is active: show the 2-row grid.
State: `'seg'` (string: 'all' / segment value).

### D8 — Metric toggle

```html
<div class="pill-group">
  <button class="pill active" onclick="setMetric('g')"><<LABEL_A>></button>
  <button class="pill" onclick="setMetric('n')"><<LABEL_B>></button>
</div>
```

State: `'t'` ('g' or 'n'). When switching, re-render all cells. Delta panel also
switches between `r[3]/r[4]` (metric A) and `r[5]/r[6]` (metric B) comparison.

### D9 — Hover tooltip on bars

On `mousemove` over an SVG bar:
1. Use `document.elementFromPoint` or SVG event target to identify bar element.
2. Show tooltip: Cell title | Bar label (PY/CY/FY) | Value | % of segment total.

### D10 — State persistence

```js
// State keys:
// 't'     metric type: 'g'=METRIC_A, 'n'=METRIC_B
// 'ym'    time period (for delta reference): 'ytd'/'fy1'
// 'seg'   active segment filter: 'all' / segment value
// 'fy1'   FY1 bars on/off (boolean)
// 'fy2'   FY2 bars on/off (boolean)
// 'pnl'   plan panel open (boolean)
// 'dn'    delta panel N count (int, default 20)
// State prefix: <<STPREFIX>> = lowercase table fragment + 'yb_'
```

---

## SECTION E — COALESCE CALCULATE BLOCK GENERATION GUIDE

This section helps the engineer generate all ~72 COALESCE CALCULATE scalars systematically.

**Naming convention:** `_g_{catKey}_{segKey}_{period}` where:
- g/n = METRIC_A / METRIC_B
- catKey = c1/c2/c3 (matching CATS[0..2]) or `t` for total across all cats
- segKey = s1/s2 (matching SEGS[0..1]) or `t` for total across all segs
- period = `cy` (CY YTD) / `py` (PY YTD) / `fy1` (full PY) / `fy2` (full PY-2)

**YTD formula pattern (with month slicer):**
```dax
VAR _g_c1_s1_cy = COALESCE(CALCULATE(SUM(TABLE[METRIC_A]),
    ALL(TABLE[YEAR_COL]), REMOVEFILTERS(<ms>),
    TABLE[YEAR_COL]=_cy, TABLE[CATEGORY_COL]="<<v>>", TABLE[SEGMENT_COL]="<<w>>",
    FILTER(ALL(TABLE[MONTH_COL]),TABLE[MONTH_COL]<=_mm)), 0)
```

**Full-year formula pattern (no month filter):**
```dax
VAR _g_c1_s1_fy1 = COALESCE(CALCULATE(SUM(TABLE[METRIC_A]),
    ALL(TABLE[YEAR_COL]), REMOVEFILTERS(<ms>),
    TABLE[YEAR_COL]=_py, TABLE[CATEGORY_COL]="<<v>>", TABLE[SEGMENT_COL]="<<w>>"), 0)
```

**Cross-category total (no CAT filter):**
```dax
VAR _g_t_s1_cy = COALESCE(CALCULATE(SUM(TABLE[METRIC_A]),
    ALL(TABLE[YEAR_COL]), REMOVEFILTERS(<ms>),
    TABLE[YEAR_COL]=_cy, TABLE[SEGMENT_COL]="<<w>>",
    FILTER(ALL(TABLE[MONTH_COL]),TABLE[MONTH_COL]<=_mm)), 0)
```

Generate all combinations: 3 cats + 1 total × 2 segs + 1 total × 2 metrics × 4 periods = 96 VARs.
If FY_COUNT = 0: omit fy1/fy2 → 48 VARs.

---

## SECTION F — VALIDATION AND TESTING

### F1 — Pre-deploy JS audit
- [ ] No DAX reserved name as VAR
- [ ] All function bodies closed
- [ ] No ternary `=`
- [ ] All 10 chunk joins use COALESCE and LEN guard: `IF(LEN(COALESCE(_j2,""))>0, "," & _j2, "")`
- [ ] `renderAll()` is the last JS call
- [ ] DAX RETURN ends with `renderAll();</script></body></html>"`
- [ ] No `id]` stray bracket
- [ ] Delta panel `openDelta()` and `closeDelta()` functions present
- [ ] FY buttons only rendered if `FY_COUNT > 0`
- [ ] Metric toggle only rendered if `METRIC_B != null`

### F2 — Live test
```dax
EVALUATE ROW("len", LEN([<<MEASURE_NAME>>]), "start", LEFT([<<MEASURE_NAME>>], 120))
```
Pass: `len > 5000` (this is a large measure), starts with `<!DOCTYPE`.

Also test entity row count:
```dax
EVALUATE ROW("rlen", LEN(CONCATENATEX(TOPN(2000, <entity_table>, ...), ..., ",")))
```
If `rlen > 700000`, reduce to TOPN(1500) or TOPN(1000).

### F3 — Troubleshooting

| Symptom | Fix |
|---|---|
| `create_measure` fails: string too long | Reduce _top2k from 2000 to 1500; verify each chunk is ≤200 rows |
| Cells all show 0 | SC property name mismatch (catKey/segKey typo) | Cross-check naming with CATS/SEGS arrays |
| Delta panel empty | RD array is empty — check chunk join LEN guards all evaluate to valid strings |
| FY bars missing | `_sv('fy1',true)` not persisted; verify toggle function calls `_sv` before `renderAll` |
| Variance connector appears upside down | y1 > y2 in SVG (SVG Y is top-down); use `Math.min(y1,y2)-22` for bridge height |
| Plan gauges missing | `PL` is null string not JS null; check: `const PL=<<planJSON>>;` where planJSON is either `null` or `{...}` |
| Segment filter breaks grid | `setSeg` re-renders full grid; ensure `renderAll()` re-reads current `SEG` state |

---

## SECTION G — USER REPORT

On success:

> "Done! **<<MEASURE_NAME>>** is ready.
>
> **To use:** Add HTML Viewer visual → drag measure into Values.
>
> **Features:**
> - 3×2 bar grid: rows = <<SEGS[0].label>> / <<SEGS[1].label>>, columns = <<CATS[0].label>> / <<CATS[1].label>> / <<CATS[2].label>>
> - Each cell: prior-year bar + current-year bar + % variance oval
> - Segment pills to focus on one segment at full width
> - Toggle between <<LABEL_A>> and <<LABEL_B>> metrics
> - <<if FY_COUNT>>: FY prior-year buttons add historical full-year bars
> - Δ button opens the variance panel: top-N records that drove each cell's change
> - <<if PLAN>>: Speedometer gauges show plan achievement per segment
> - All settings persist across filter changes"
