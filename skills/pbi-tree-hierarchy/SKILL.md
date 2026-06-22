---
name: pbi-tree-hierarchy
description: >
  Builds a fully interactive HTML Collapsible Tree visual with multi-dimensional hierarchy,
  dual-metric toggle, 3-year comparison, sparkline mini-charts, plan-overlay speedometer gauges,
  variance drill-down slide-in panel, and FY toggle buttons as a DAX measure in Power BI.
  Works on any dataset in any industry — discovers tables, year/month columns, metrics, and
  dimension columns automatically. Supports up to 6 hierarchy levels, segment filter (e.g. Segment A/B),
  and persistent state across report interactions.
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

# /pbi-tree-hierarchy — Interactive Collapsible Tree Visual for Power BI

Produces a fully interactive **Collapsible Tree** visual rendered inside Power BI's HTML Viewer
custom visual. The visual displays a stacked-bar chart for each combination of
[Primary Segment × Secondary Segment] using data from any model table — no hardcoded column names.

**Supported datasets:** any time-series data with a year column, a month column, one or more numeric
metrics, a primary segment column (e.g. product group, channel, region), and optionally a secondary
segment column. Works in finance, retail, logistics, healthcare, SaaS, manufacturing, and any other domain.

Arguments: `$ARGUMENTS`

---

## SECTION A — MANDATORY BEHAVIOUR RULES

### A1 — Connect first
Call `connect_powerbi` before any other tool. Fail loudly if Power BI is not reachable.

### A2 — Never guess column names
Every column reference must come from `get_table_schema`. If a column name is ambiguous,
ask the user to confirm it by number.

### A3 — Always create a new version
Check `list_measures`. If `Tree Visual v4` exists, create `Tree Visual v5`. Never overwrite.

### A4 — Validate after creation
After `create_measure`, run `execute_dax` to verify. If broken, self-diagnose and fix with a new
version number. Never leave unverified.

### A5 — Silent DAX construction
Never show raw DAX to the user. Report success/failure only.

### A6 — Never inline DAX in create_measure
Build the expression string in a variable, verify it, then pass it.

### A7 — Plain language
No DAX/Power BI jargon in user-facing messages. Use numbered choices.

### A8 — Reserved DAX variable names — NEVER use as VAR names
`Table`, `Filter`, `Values`, `All`, `Calculate`, `Row`, `Date`, `Time`, `Today`, `Now`,
`DataTable`, `Calendar`, `Union`, `Intersect`, `Except`, `Generate`, `AddColumns`,
`SelectColumns`, `Summarize`, `CrossJoin`, `Distinct`

Use prefixed alternatives: `_tblRows`, `_dataSet`, etc.

---

## SECTION B — DISCOVERY PHASE

### B1 — Identify primary data table

Call `list_tables`. Present non-hidden tables:

> "Which table contains the data for this tree visual?"
> 1. SalesData
> 2. RevenueHistory
> _(other)_

Save as **TABLE**.

### B2 — Inspect schema

Call `get_table_schema(TABLE)`. Classify columns:

| Bucket | Criteria |
|---|---|
| **YEAR_CANDIDATES** | Integer, name contains `year`, `yr`, `jhjj`, `fy`, `gj` |
| **MONTH_CANDIDATES** | Integer 1–12, name contains `month`, `mm`, `monat`, `per` |
| **METRIC_CANDIDATES** | Numeric, additive (revenue, count, volume, amount) |
| **SEG1_CANDIDATES** | Text, low cardinality (<30 unique values preferred), represents a grouping dimension |
| **SEG2_CANDIDATES** | Text, represents a sub-grouping dimension |
| **DIM_CANDIDATES** | All remaining text columns usable as hierarchy levels |
| **ENTITY_CANDIDATES** | Text or numeric ID columns identifying individual records |

### B3 — Year and month

Ask for YEAR_COL (confirm or numbered list). Ask for MONTH_COL (Yes/No → numbered list).
Save as **YEAR_COL**, **MONTH_COL** (or null).

### B4 — Metrics

> "Which column is the **main metric** (e.g. Total Revenue, Gross Premium, Total Units)?"

Save as **METRIC_TOTAL**. Ask for short label → **LABEL_TOTAL** (e.g. "Total", "Gross").

> "Is there a **New/Incremental metric** column to compare alongside Total?
> (e.g. New Business, New Sales, Incremental Units)
> 1. Yes — pick it
> 2. No — single metric"

Save as **METRIC_NEW** (or = METRIC_TOTAL if No). Ask for label → **LABEL_NEW**.

> "Is there a **Renewal/Retained metric** column (e.g. Renewal Premium, Retained Revenue)?
> 1. Yes — pick it
> 2. No"

Save as **METRIC_RENEW** (or null). Ask for label → **LABEL_RENEW**.

### B5 — Segments (primary split)

> "Which column should be used as the **primary segment filter** shown as pills at the top of the visual?
> (e.g. Product Type, Channel, Region, Customer Segment)
> 1. Pick a column
> 2. No segment filter needed"

If yes: save as **SEG_COL**, ask for the two or three main segment values to show as pills.
If no: **SEG_COL = null**.

### B6 — Hierarchy levels

> "Which columns define the **hierarchy** in the tree? Choose up to 6, ordered broadest → most detailed.
> Example: Division → Department → Team → Individual"

Save as **DIMS** array `[{col, label, key:'l1'...'l6'}]`.

> "For the comparison bars, how many prior years should be shown alongside the current year?
> 1. 1 year (PY only)
> 2. 2 years (PY + PY-2 full year)
> 3. 3 years"

Save as **NUM_YEARS** (1/2/3).

### B7 — Plan data (optional)

> "Is there a **plan/budget/target table** in your model that I can overlay on the visual?
> 1. Yes — show me the tables
> 2. No — skip plan overlay"

If yes:
- Ask for PLAN_TABLE, PLAN_VALUE_COL, PLAN_YEAR_COL, PLAN_MONTH_COL, PLAN_SEG_COL (must match SEG_COL values).
- Save as **PLAN** object.

If no: **PLAN = null**. Plan panel and speedometers will be hidden.

### B8 — Entity detail panel

> "Which column identifies individual records for the drill-down panel?
> (e.g. Contract ID, Order Number, Customer Code)
> 1. Pick a column
> 2. No detail panel needed"

Save as **ENTITY_COL** (or null).

If yes, ask for up to 6 additional display columns → **DETAIL_COLS**.

### B9 — Measure name

Ask for name, auto-increment version, save as **MEASURE_NAME**.

---

## SECTION C — DAX ARCHITECTURE

### C1 — High-level structure

The measure returns a single HTML string. Use this exact layering:

```
VAR _mm  = ...month pointer...
VAR _CY  = ...current year...
VAR _PY  = _CY - 1  (prior year)
VAR _PY2 = _CY - 2
VAR _PY3 = _CY - 3

-- Grand totals: 3 years × 3 metrics (TOTAL, NEW, RENEW) = up to 9 scalars
VAR _tGC = CALCULATE(SUM(TABLE[METRIC_TOTAL]), ...)
VAR _tGP = ...
...

-- Per-segment totals if SEG_COL != null
-- These feed the speedometer gauges and KPI header

-- Summary rows: DIMS hierarchy × metrics × years
VAR _S = CALCULATETABLE(SUMMARIZE(TABLE, TABLE[DIM_L1], TABLE[DIM_L2], ...), ...)
VAR _T = ADDCOLUMNS(_S, "cg", ..., "pg", ..., ...)

-- Serialise: TOPN 3000 hierarchy rows
VAR _hRows = CONCATENATEX(TOPN(3000, _T, [cg], DESC), "{ ... }", ",")

-- Detail rows (entity-level) if ENTITY_COL != null
VAR _dRows = CONCATENATEX(TOPN(2000, _dData, [cg], DESC), "{ ... }", ",")

-- Plan scalars if PLAN != null
VAR _planData = ...

RETURN
"<!DOCTYPE html>..." & CSS &
"<script>" &
"const CY=...,PY=...,PY2=...,PY3=...,MM=...;" &
"const TG={...totals...};" &
"const HR=[...hierarchy rows...];" &
"const DR=[...detail rows...];" &
"const PL={...plan data...};" &
"...all JS...render();</script></body></html>"
```

### C2 — Layout: 3-row × 2-column grid

```
+--------------------------------------------------+
| HEADER ROW: title + KPI cards + segment pills    |
+---------------------------+----------------------+
| MAIN AREA                 | SIDE PANEL           |
| ROW 1 (Portfolio): 2 bars | speedometer gauges   |
| ROW 2 (Segment A): 2 bars | (visible only when   |
| ROW 3 (Segment B): 2 bars |  PLAN != null)       |
+---------------------------+----------------------+
```

Each grid cell in the main area shows a stacked bar chart for:
- Left bar: prior year (PY)
- Right bar: current year (CY)
Plus a connecting variance arrow between the bars (blue oval, shows % change).

### C3 — Data structure for hierarchy rows

```js
// Each row in HR:
{
  l1: 'string',   // DIM level 1 value
  l2: 'string',   // DIM level 2 value
  // ... up to l6
  cg: 1234567,    // METRIC_TOTAL current year
  pg: 1100000,    // METRIC_TOTAL prior year
  g2: 1050000,    // METRIC_TOTAL PY-2 full year (if NUM_YEARS >= 2)
  g3: 980000,     // METRIC_TOTAL PY-3 full year (if NUM_YEARS >= 3)
  cn: 234567,     // METRIC_NEW current year
  pn: 210000,     // METRIC_NEW prior year
  n2: 200000,
  n3: 190000,
  rg: 1000000,    // METRIC_RENEW current year (if RENEW != null)
  rp: 950000
}
```

### C4 — Stacked bar chart per cell

Each bar cell (`rc()` function) receives:
- `bars`: array of [PY_bars, CY_bars, FY2_bars, FY3_bars] where each entry is an array of [metric_A, metric_B, metric_C, ...]
- `labels`: x-axis labels ['PY YTD', 'CY YTD', 'FY PY-2', ...]
- `cnts`: optional contract/record count array for golden trend line overlay
- `title`: cell title string

Stacked bar segments use the **SEGMENT_COLORS** array (4 colours, one per metric).

The **variance connector** (blue oval with % value) connects bar 0 top to bar 1 top:
- Vertical stems from each bar top → horizontal bridge → oval at midpoint
- Arrow head pointing into bar 1

### C5 — Sparkline trend column

In the ranking/table view (right side), each row gets a small inline sparkline canvas (56×22px).
Colour: green if last value >= first value, red otherwise.
Data source: monthly values `ms:[m1,m2,...,m12]` in each HR row.

### C6 — Speedometer gauges (Plan panel)

When PLAN != null, the side panel shows one speedometer per segment:

```js
function drawSp(id, spd, title) {
  // spd = { planTotal, actualTotal, bySegment: [...] }
  // Draw semicircle arc from left to right
  // Needle points to actual/plan fraction (0=0%, 1=100%)
  // Target tick at the plan month fraction (currentMonth/12)
  // Colour: green pill if actual >= target fraction, red if below
  // Show plan vs actual in k-units below needle
}
```

### C7 — Variance drill-down panel

When user clicks a variance oval, open a slide-in panel (right edge, 860px wide):

```
+----------------------------------------------+
| Header: Δ {Title} — Variance Detail  [×]     |
+----------------------------------------------+
| ▼ Top {n} Negative Variance (most neg first) |
| [scrollable table: ID | Dim1 | Seg | CY | PY | Δ] |
+----------------------------------------------+
| ▲ Top {n} Positive Variance (most pos first) |
| [scrollable table]                            |
+----------------------------------------------+
| Footer: CY total | PY total | Net variance   |
+----------------------------------------------+
```

Data comes from **DR** (detail rows). Filter by segment if SEG_COL != null.

### C8 — FY toggle buttons

Show "FY {PY-2}" and "FY {PY-3}" buttons in the header when NUM_YEARS >= 2.
Clicking adds a third/fourth bar to every cell. State persisted via `_sv`.

### C9 — State variables and persistence

```js
var _ST = {};
function _sv(k,v) { _ST[k]=v; postMessage...; localStorage... }
function _lv(k,fb) { ...check __pbifch_lv, _ST, localStorage, return fb }

// State keys:
// 't'    — metric type: 'g'=TOTAL, 'n'=NEW (string)
// 'ym'   — time period: 'ytd'=YTD, 'cy1'=PY full-year, 'cy2'=PY-2 full-year
// 'fy2'  — FY PY-2 bars on/off (boolean)
// 'fy3'  — FY PY-3 bars on/off (boolean)
// 'seg'  — active segment filter value (string or 'all')
// 'pnl'  — plan panel open (boolean)
// 'sel'  — selected row key for detail panel (string or null)
// State prefix: <<STPREFIX>> = first 4-6 chars of table name + 'tr_'
```

---

## SECTION D — JAVASCRIPT FUNCTIONS

| Function | Purpose |
|---|---|
| `_sv(k,v)` / `_lv(k,fb)` | Persist and load state |
| `fm(v, short)` | Format number: M/K/integer, de-DE locale |
| `fg(v)` | Format growth: "+3.5%" or "-1.2%" |
| `fgc(v)` | `fg` wrapped in `<span class='pos'/'neg'>` |
| `nm(v)` | Round up to nearest "nice" number (1/2/5 × 10^n) for y-axis max |
| `ft(v)` | Format as integer k-units (round to 0 dp) |
| `fM(v)` | Format as millions (2 dp) |
| `fmt3(v)` | Round to nearest k, de-DE locale |
| `rc(id,bars,lbl,title,cnts,sf,un)` | Render one stacked bar cell into element `id` |
| `drawSp(id,spd,title)` | Render one speedometer into element `id` |
| `drSpark(ctx,cx,cy,r,ms)` | Draw sparkline on canvas context |
| `renderTable()` | Render the ranking table on the right side |
| `renderAll()` | Render all 6 grid cells + right table + FY buttons |
| `renderFyBtns()` | Inject FY-2 and FY-3 toggle buttons if not already present |
| `toggleFy2()` / `toggleFy3()` | Toggle FY bars on/off, re-render |
| `openDelta(sf,un,title)` | Open variance drill-down panel |
| `closeDelta()` | Close variance panel |
| `showTip(evt,bars,bi,lbl,cnt)` | Show tooltip on bar hover |
| `showSpTip(evt,spd,title,totA,totP)` | Show speedometer tooltip |
| `moveTip(evt)` | Position tooltip near cursor |
| `hideTip()` | Hide tooltip |
| `sT(type)` | Switch metric type, re-render |
| `sYM(ym)` | Switch time period, re-render |
| `setSeg(seg)` | Filter by segment, re-render |
| `togglePanel()` | Toggle plan side panel open/closed |
| `setMainH()` | Resize main area height on window resize |
| `startAnim()` | Kick off requestAnimationFrame bar-entry animation |
| `draw()` | Immediate (non-animated) redraw |
| `drawFrame()` | Single animation frame draw call |

---

## SECTION E — VISUAL DESIGN CONSTANTS

```js
// Segment colours (4 metrics): red, green, amber, blue
var QC = ['#ff3b30','#0071e3','#ff9500','#34c759'];
var QBG = ['rgba(255,59,48,.06)','rgba(0,113,227,.06)',
           'rgba(255,149,0,.06)','rgba(52,199,89,.06)'];
var QN = [LABEL_TOTAL, LABEL_NEW, LABEL_RENEW, 'Other'];

// Font: Inter via Google Fonts
// Background: #f5f5f7
// Control bar: rgba(255,255,255,.88) + backdrop-filter:blur(12px)
// Pill groups: background:#e8e8ea, active pill: #fff + #0071e3
```

---

## SECTION F — VALIDATION AND TESTING

### F1 — JS pre-deploy audit
- [ ] No DAX reserved word used as VAR name (Section A8)
- [ ] All function bodies closed with `}`
- [ ] No ternary `=` (must be `:`)
- [ ] No template literals (use string concatenation)
- [ ] Final JS call is `renderAll();`
- [ ] DAX RETURN string ends with `renderAll();</script></body></html>"`
- [ ] No `id]` stray bracket
- [ ] All `querySelector` strings are valid CSS selectors

### F2 — Live test
```dax
EVALUATE ROW("len", LEN([<<MEASURE_NAME>>]), "start", LEFT([<<MEASURE_NAME>>], 120))
```
Pass: `len > 2000` and `start` begins with `<!DOCTYPE html>`.

### F3 — Self-diagnosis table

| Error | Cause | Fix |
|---|---|---|
| "Column not found" | Column name mismatch | Re-read schema, copy exactly |
| "Exceeds string length" | TOPN too large | Reduce to 2000 then 1500 |
| "Circular dependency" | Reserved name as VAR | Rename VAR with `_` prefix |
| Blank visual | JS runtime error | Check browser console; scan for ternary `=` |
| Bars show 0 | Key mismatch in data object | Confirm `cg`/`cn` keys match JS accessor |
| FY buttons missing | `renderFyBtns()` not called | Ensure call inside `renderAll()` |

---

## SECTION G — USER REPORT

On success:

> "Done! **<<MEASURE_NAME>>** is ready.
>
> **To use it:** Add an HTML Viewer visual, drag the measure into Values.
>
> **Features:**
> - 3×2 grid shows <<LABEL_TOTAL>> and <<LABEL_NEW>> for each segment
> - Prior-year bar + current-year bar with % variance arrow per cell
> - Click any % oval to see the full variance detail (top positive + top negative records)
> - FY buttons add prior full-year bars for multi-year comparison
> - Speedometer panel shows plan achievement per segment (if plan data was provided)
> - Hover any bar for a breakdown tooltip
> - All settings persist when you change report filters"
