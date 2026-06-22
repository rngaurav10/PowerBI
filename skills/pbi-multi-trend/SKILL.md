---
name: pbi-multi-trend
description: >
  Builds a fully interactive multi-series monthly trend line chart as a DAX measure in Power BI.
  Supports up to 8 category series, 3 metric types (Total/New/Renewal), 2 segment filters (e.g. Type A/B),
  3 prior-year comparison overlays, optional plan/budget line overlay, interactive color customization,
  KPI summary cards, popover tooltip, and SVG-based rendering. Works on any dataset with a year column,
  a month column, one or more additive metrics, and one or more category columns. Plan overlay uses
  a separate optional plan table. All settings persist across report filter changes.
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

# /pbi-multi-trend — Interactive Multi-Series Trend Visual for Power BI

Produces a **monthly trend line chart** with category breakdown, plan overlay, prior-year
comparison, and interactive controls as a single DAX measure. Rendered via the HTML Viewer custom visual.

Works on **any dataset** — time-series data with a year column, a month column, a numeric metric, and
one or more dimension/category columns. No hardcoded assumptions about the domain.

Arguments: `$ARGUMENTS`

---

## SECTION A — MANDATORY BEHAVIOUR RULES

### A1 — Connect, inspect, version, test, report
Follow the same four-phase discipline as every other skill:
connect → inspect schema → build → create new version → validate → report.

### A2 — Never guess column names
Every column reference comes from `get_table_schema`. Zero guessing.

### A3 — Always increment version
Check `list_measures`. If `Trend Visual v3` exists, create `Trend Visual v4`.

### A4 — Reserved DAX VAR names — NEVER use
`Table`, `Filter`, `Values`, `All`, `Calculate`, `Row`, `Date`, `Time`, `Today`, `Now`,
`DataTable`, `Calendar`, `Union`, `Intersect`, `Except`, `Generate`, `AddColumns`,
`SelectColumns`, `Summarize`, `CrossJoin`, `Distinct`

### A5 — String length management
The trend visual generates many per-month per-category series. Total DAX string can exceed 150K characters.
If `create_measure` returns "exceeds string length":
1. Reduce series from 8 to 6 categories.
2. Reduce year range from 5 to 4 years.
3. Reduce plan detail from per-segment to total-only.

---

## SECTION B — DISCOVERY PHASE

### B1 — Primary data table
`list_tables` → ask user to pick.  Save as **TABLE**.

### B2 — Schema inspection
`get_table_schema(TABLE)`. Classify:
- **YEAR_COL**: integer, 4-digit year
- **MONTH_COL**: integer 1–12
- **METRIC_CANDIDATES**: numeric additive columns
- **CATEGORY_COL**: text column whose distinct values become the coloured series (recommended: 4–8 values)
- **SEG_COL** (optional): text column for segment filter pills (recommended: 2–4 values)
- **ENTITY_COL** (optional): individual record identifier

### B3 — Year range
> "What is the **earliest year** to include in the chart? (Default: 5 years back)"
Save as **START_YEAR**. **END_YEAR** = MAX of the year column (auto-detected).

> "Does your data also have a month column for monthly detail?"
Save as **MONTH_COL** (or null for annual-only).

### B4 — Metrics (1–3)
Ask for:
1. **METRIC_TOTAL** — main aggregation (e.g. Total Revenue). Label → **LABEL_TOTAL**.
2. **METRIC_NEW** (optional) — e.g. New Business. Label → **LABEL_NEW**.
3. **METRIC_RENEW** (optional) — e.g. Renewal. Label → **LABEL_RENEW**.

If METRIC_NEW + METRIC_RENEW are provided, the visual adds stacked-total context:
Stacked = METRIC_NEW + METRIC_RENEW per month, METRIC_TOTAL is the separate total line.

### B5 — Category series
Show distinct values preview (if possible via execute_dax):
> "Which column should define the **coloured series** in the chart?
> (Each unique value becomes a separate line — recommended: 4–8 distinct values)"

Save as **CAT_COL**. Ask for display label for each category if auto-detection finds ≤ 8 values.
If > 8 values: ask user to limit or group.

Save as **SERIES**: `[{value, label, defaultColor}, ...]`.

### B6 — Segment filter (optional)
> "Should the visual have **segment filter pills** that let users switch between views?
> (e.g. Type A / Type B / All)
> 1. Yes — pick a column
> 2. No"

If yes: save **SEG_COL**, detect distinct values (max 3), ask for labels → **SEGMENTS**.
If no: **SEG_COL = null**.

### B7 — Plan table (optional)
> "Is there a **plan/budget/target** table in your model?
> 1. Yes
> 2. No"

If yes: ask for PLAN_TABLE, PLAN_VALUE_COL, PLAN_YEAR_COL, PLAN_MONTH_COL, PLAN_CAT_COL, PLAN_SEG_COL.
Save as **PLAN** object. If no: **PLAN = null**.

### B8 — Prior-year comparison
> "How many **prior-year lines** should be overlaid on the chart?
> 1. None (show only current period)
> 2. 1 prior year (PY)
> 3. 2 prior years (PY + PY-2)"

Save as **PY_LINES** (0/1/2).

### B9 — Measure name
Ask, auto-increment, save as **MEASURE_NAME**.

---

## SECTION C — DAX ARCHITECTURE

### C1 — Month axis construction

The trend visual requires generating a continuous month axis even if some months have no data.
Use GENERATESERIES to create the full axis:

```dax
VAR _sYr = <<START_YEAR>>
VAR _eYr = MAX(TABLE[YEAR_COL])
-- Generate all months from _sYr Jan through _eYr Dec
VAR _allPeriods = FILTER(
    ADDCOLUMNS(
        GENERATESERIES(1, (_eYr - _sYr + 1) * 12, 1),
        "Yr", _sYr + INT(([Value] - 1) / 12),
        "Mn", MOD([Value] - 1, 12) + 1
    ),
    -- Only keep months that have actual data:
    VAR _y = [Yr]  VAR _m = [Mn]
    RETURN CALCULATE(
        COUNTROWS(TABLE),
        ALL(TABLE[YEAR_COL]), ALL(TABLE[MONTH_COL]),
        TABLE[YEAR_COL] = _y, TABLE[MONTH_COL] = _m
    ) > 0
)
VAR _lastActYr = MAXX(ALL(TABLE), TABLE[YEAR_COL])
VAR _lastActMn = MAXX(FILTER(ALL(TABLE), TABLE[YEAR_COL]=_lastActYr), TABLE[MONTH_COL])

-- Future plan months (months after last actual, within current year)
VAR _planPeriods = FILTER(
    ADDCOLUMNS(GENERATESERIES(1,12,1), "Yr",_lastActYr, "Mn",[Value],
               "Dt",DATE(_lastActYr,[Value],1)),
    [Value] > _lastActMn
)

VAR _sortedPeriods = TOPN(72,
    UNION(
        SELECTCOLUMNS(_allPeriods,"Yr",[Yr],"Mn",[Mn],
                      "Dt",DATE([Yr],[Mn],1),"IP",0),
        SELECTCOLUMNS(_planPeriods,"Yr",[Yr],"Mn",[Mn],"Dt",[Dt],"IP",1)
    ), [Dt], ASC)
```

### C2 — Per-series per-month data VARs

For each series in SERIES and each month in _sortedPeriods, generate a CONCATENATEX string.
Total VARs = |SERIES| × |METRICS| × |SEGMENTS|.

Pattern for one series, one metric, no segment:
```dax
VAR _s1_total = CONCATENATEX(
    _sortedPeriods,
    VAR _y = [Yr]  VAR _m = [Mn]
    VAR _v = COALESCE(
        CALCULATE(SUM(TABLE[METRIC_TOTAL]),
            ALL(TABLE[YEAR_COL]), ALL(TABLE[MONTH_COL]),
            TABLE[YEAR_COL]=_y, TABLE[MONTH_COL]=_m,
            TABLE[CAT_COL]="<<SERIES[0].value>>"
        ), 0)
    RETURN FORMAT(_v,"0"),
    ",",
    [Dt], ASC
)
-- Repeat for _s1_new, _s1_renew
-- Repeat for series 2–8: _s2_total, _s2_new, _s2_renew, ...
-- Repeat for each segment if SEG_COL != null
```

**String length warning:** For 8 series × 3 metrics × 2 segments × 72 months = 3456 scalar calculations.
This is near the DAX engine limit. If the measure fails with "exceeds string length", reduce series count.

### C3 — Months JSON array

```dax
VAR _mJ = "[" & CONCATENATEX(
    _sortedPeriods,
    VAR _y=[Yr] VAR _m=[Mn] VAR _ip=[IP]
    RETURN "{""yr"":" & _y & ",""mn"":" & _m &
           ",""lbl"":""" & FORMAT(DATE(_y,_m,1),"MMM YY") &
           """,""ip"":" & _ip & "}",
    ",", [Dt], ASC
) & "]"
```

### C4 — Data JSON object

Bundle all series arrays into one JS object:

```dax
VAR _dJ = "{" &
  """s1_total"":[" & _s1_total & "]," &
  """s1_new"":[" & _s1_new & "]," &
  """s1_renew"":[" & _s1_renew & "]," &
  -- ... all series and segments ...
  """s1_plan"":[" & _s1_plan & "]" &
  -- plan arrays: zero unless PLAN != null
"}"
```

---

## SECTION D — JAVASCRIPT ARCHITECTURE

### D1 — Layout

```
+------------------------------------------+
| Toolbar: Title | Color pickers | Controls |
+------------------------------------------+
| KPI summary strip: series totals          |
+------------------------------------------+
| Legend: colored dots + series labels      |
+------------------------------------------+
| Main SVG chart area (flex:1)              |
+------------------------------------------+
| Mini overview scroll bar (optional)       |
+------------------------------------------+
```

### D2 — Control bar elements

Left side (in order):
1. **Chart title** (static, from measure name)
2. **Color pickers**: one `<select>` per series, showing color name options
3. **Info badge**: shows data range and last actual month

Right side:
1. **Year filter pills**: one pill per year (toggle on/off)
2. **PY comparison toggles**: "1Y", "2Y", "3Y" (toggle prior-year overlay lines)
3. **Metric type buttons**: "Total OFF/ON", "New OFF/ON", "Renewal OFF/ON"
4. **Segment pills** (if SEG_COL != null): e.g. "Seg A", "Seg B"
5. **Plan toggle** (if PLAN != null): "Plan OFF/ON"
6. **Var Bars toggle** (if PLAN != null): shows variance bars between actual and plan
7. **Summary toggle**: "Gesamt OFF/ON" — adds a total-all-series line
8. **Tooltip toggle**: "Tooltip ON/OFF"
9. **Line trend toggle**: "Line Trend OFF/ON" — adds regression trend line
10. **Help button** `?`

### D3 — SVG chart rendering

```js
function renderChart() {
  // Compute data range, axis ticks, series lines
  // For each series:
  //   draw line path: d3-style SVG path using M/L commands
  //   draw data points: small circles on each month
  //   if PY overlay: dashed line in 60% opacity of series colour
  //   if plan: dashed grey line
  //   if var bars: small vertical bars between actual and plan

  // X axis: month labels, tick marks
  // Y axis: nice round values, horizontal grid lines

  // Tooltip zone: invisible rects per month x-band, onmouseenter shows tooltip
}
```

### D4 — KPI summary cards

One card per series, showing:
- Series colour dot + name
- YTD total (current year up to last actual month)
- vs PY: absolute variance + % (green/red pill)
- vs Plan (if PLAN != null): % achievement

### D5 — Tooltip content

On hover over a month band, show:
```
Month: <<month label>>
--- Line for each series: ---
  [colour dot] <<series name>>: <<value>>
  vs PY: +/-<<abs>> (<<pct>>%)
  Plan: <<plan_value>> | Var: +/-<<var>>
--- Total: <<sum of all series>> ---
```

### D6 — Color customization

Each series has a `<select>` in the toolbar with 9–12 colour options:
Royal Blue, Cobalt, Cornflower, Deep Blue, Teal, Mint, Ocean, Midnight, Orange, Purple, Green, Amber.

When user changes a colour: update the line, legend dot, and KPI card. Persist via `_sv('lc', colorsArray)`.

### D7 — Plan popover

If PLAN != null, clicking the plan line opens a small popover above the cursor:
- Input field: override the plan value for that series/month
- Buttons: OK | Clear | Reset All
- Entering a value creates a local override stored in `_ST.planOverrides`
- Reset restores the DAX-sourced values

### D8 — State variables

```js
// Key           Default    Description
// 'yr'          [all]      Active years array
// 'py1','py2'   false      PY comparison toggles
// 'mt'          'total'    Metric type: 'total'/'new'/'renew'
// 'sg'          'all'      Active segment filter
// 'showSum'     false      Summary total line on/off
// 'showTT'      true       Tooltips on/off
// 'showLT'      false      Line trend on/off
// 'showPl'      false      Plan line on/off
// 'showVB'      false      Var bars on/off
// 'lc'          null       Color overrides array
// 'planOv'      {}         Plan value overrides map
// State prefix: <<STPREFIX>> = lowercase table name fragment + 'tr_'
```

### D9 — Pan and slide overview bar

Optional: a mini SVG overview bar at the bottom that mirrors the full data range.
User drags a selection window on the overview to zoom into a date range.
Persist zoom range in `_ST`.

---

## SECTION E — PLAN OVERLAY — DETAILED SPEC

When PLAN != null:

1. DAX fetches per-series per-month plan values from PLAN_TABLE.
2. JS renders a dashed line in a lighter shade of each series colour.
3. Variance bars: thin vertical bars between actual and plan, green if actual > plan, red if below.
4. Plan popover (per series per month): input to override plan value locally.
5. Reset button: clears all local overrides, restores DAX values.
6. Plan panel (slide-in from right): speedometer gauges for current-year plan achievement per series.

---

## SECTION F — VALIDATION AND TESTING

### F1 — Pre-deploy JS audit
- [ ] No DAX reserved name as VAR
- [ ] All function bodies closed
- [ ] No ternary `=` (must be `:`)
- [ ] Final JS: `renderChart();` (or equivalent entry point)
- [ ] DAX RETURN ends with `renderChart();</script></body></html>"`
- [ ] All `querySelector` strings valid
- [ ] No `id]` stray bracket
- [ ] String length: test with small TOPN (5 series, 3 years) first if uncertain

### F2 — Live test
```dax
EVALUATE ROW("len", LEN([<<MEASURE_NAME>>]), "start", LEFT([<<MEASURE_NAME>>], 120))
```
Pass: `len > 3000`, `start` = `<!DOCTYPE html>`.

### F3 — Fallback if string too long
1. Reduce series: keep only top-N by YTD total (ask user: "Keep top 4 or top 6 series?").
2. Reduce year range: START_YEAR + 1.
3. Remove per-segment variants: compute all segments combined only.
4. Remove plan overlay if PLAN != null but measure still too long.

---

## SECTION G — USER REPORT

On success:

> "Done! **<<MEASURE_NAME>>** is ready.
>
> **To use:** Add an HTML Viewer visual → drag the measure into Values.
>
> **Features:**
> - Monthly trend lines for: <<series names>>
> - Toggle between <<LABEL_TOTAL>> / <<LABEL_NEW>> / <<LABEL_RENEW>> using the toolbar buttons
> - PY comparison: overlay prior-year lines with "1Y" / "2Y" buttons
> - Color pickers to customise each series colour
> - <<if PLAN>>: Plan line overlay + variance bars + speedometer gauges
> - Segment filter pills: <<SEGMENTS>>
> - Hover any month for a detailed breakdown tooltip
> - All settings persist across filter changes"
