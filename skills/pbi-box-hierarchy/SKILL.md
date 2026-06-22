---
name: pbi-box-hierarchy
description: >
  Builds a fully interactive HTML Box Hierarchy visual with drag-and-drop dimension management,
  zoom/pan canvas, prior-year variance indicators, and a drill-down detail table as a DAX measure
  in Power BI. Works on any dataset in any industry — discovers all tables, columns, and metrics
  automatically. Supports up to 5 configurable hierarchy levels, dual-metric toggle (e.g. Total vs New),
  up to 3 year comparisons, Excel export from detail table, focus mode, and persistent state.
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

# /pbi-box-hierarchy — Interactive Box Hierarchy Visual for Power BI

Produces a fully interactive **Box Hierarchy** visual with a **drill-down detail table** as a single
DAX measure, rendered via Power BI's HTML Viewer custom visual.

Works on **any dataset in any industry** — retail, finance, insurance, healthcare, logistics, SaaS,
manufacturing, and more. Claude auto-discovers all required columns by inspecting the live model.

Arguments: `$ARGUMENTS`

---

## SECTION A — MANDATORY BEHAVIOUR RULES

Read every rule before taking any action. They override all defaults.

### A1 — Connect first, always
Never assume Power BI is open. Call `connect_powerbi` at the start of every invocation.
On failure: tell the user "Power BI Desktop isn't reachable — please open it with a report loaded, then retry." Stop.

### A2 — Inspect before building, always
Call `list_tables` and `get_table_schema`. **Never guess column names.** Every column reference in
the DAX must come from the schema response.

### A3 — Always create a new version
Call `list_measures` before creating. If `Box Hierarchy v3` exists, create `Box Hierarchy v4`.
Never overwrite an existing measure. Never ask the user to delete anything.

### A4 — Validate and test after creation
After `create_measure` succeeds, execute a live test with `execute_dax`. If the test fails, self-diagnose
and fix — incrementing the version number each retry. Never leave a broken measure without reporting the issue.

### A5 — Never display raw DAX to the user
Work silently. Only report success, failure, or ask a clarifying question. Do not dump the DAX string
into chat.

### A6 — Never type DAX inline in create_measure
Build the full expression in a local variable. Verify it. Then pass the exact string to `create_measure`.
A single mismatched quote or ternary `=` instead of `:` silently breaks all JS and produces a blank visual.

### A7 — Plain language only
No Power BI jargon, no DAX syntax, no column-name references in questions. Show numbered choices for
every decision the user must make.

### A8 — Reserved DAX variable names — NEVER use as VAR names
The following words are built-in DAX function names and will cause a circular dependency error if used
as VAR identifiers. Replace with prefixed alternatives (e.g. `_tblData`, `_dataSet`):

> `Table`, `Filter`, `Values`, `All`, `Calculate`, `Row`, `Date`, `Time`, `Today`, `Now`,
> `DataTable`, `Calendar`, `Union`, `Intersect`, `Except`, `Generate`, `AddColumns`,
> `SelectColumns`, `Summarize`, `CrossJoin`, `Distinct`

---

## SECTION B — DISCOVERY PHASE

### B1 — List tables

Call `list_tables`. Show non-hidden tables to the user:

> "I found these tables in your model. Which one contains the data you want to visualise?"
> 1. TableA
> 2. TableB
> _(other option)_

Save the selected table as **TABLE**.

### B2 — Inspect schema

Call `get_table_schema(TABLE)`. Classify all columns into five buckets:

| Bucket | Criteria |
|---|---|
| **YEAR_CANDIDATES** | Integer or DateTime, name contains: `year`, `yr`, `jhjj`, `jahr`, `fy`, `gj`, `period` (case-insensitive) |
| **MONTH_CANDIDATES** | Integer, name contains: `month`, `mm`, `monat`, `per`, `period`, `woche` |
| **METRIC_CANDIDATES** | Numeric (Decimal, Double, Int64), not hidden, represents an additive measure (revenue, count, units) |
| **DIM_CANDIDATES** | Text/String, not hidden, not a pure surrogate key |
| **ENTITY_CANDIDATES** | Text columns likely to identify individual records: names containing `id`, `nr`, `num`, `vsnr`, `ref`, `key`, `code`, or short string columns |

### B3 — Year column

If exactly one YEAR_CANDIDATE exists: confirm it.
If multiple: show numbered list and ask user to pick.
If none: ask for the column that holds the calendar year as a 4-digit integer.

Save as **YEAR_COL**.

> "Does your data also have a **month column** that lets you filter to a specific point in the year?
> (This enables year-to-date comparisons.)
> 1. Yes — pick it
> 2. No — compare full-year totals only"

If yes: show MONTH_CANDIDATES, save as **MONTH_COL**.
If no: **MONTH_COL = null**.

### B4 — Primary metric

Show METRIC_CANDIDATES:

> "Which column holds the **main numeric value** you want to show in the boxes?
> (e.g. Revenue, Sales, Premium, Count, Units)"

Save as **METRIC_A**. Ask for a short display label (e.g. "Total", "Revenue", "Gesamt") → **LABEL_A**.

> "Is there a **second numeric column** to compare alongside the first?
> (e.g. New Business vs Total, Net vs Gross, Actuals vs Budget)
> 1. Yes — pick it
> 2. No — single metric only"

If yes: save as **METRIC_B**, ask for label → **LABEL_B**.
If no: **METRIC_B = METRIC_A**, **LABEL_B = LABEL_A**. The toggle button will be hidden.

### B5 — Hierarchy dimensions

Show DIM_CANDIDATES:

> "Pick the columns to use as **hierarchy levels** in the visual.
> Choose up to 5, ordered from broadest (top) to most detailed (bottom).
> Example: Region → Product Line → Sales Rep → Customer"

Accept 2–5 columns. For each, ask for a short display label if the column name is cryptic.
Save as **DIMS** array: `[{col, label, key:'d1'}, {col, label, key:'d2'}, ...]`.

Assign keys `d1`–`d5` in order.

> "How many items should be shown per level at most?
> 1. 10 per level (default, keeps the visual clean)
> 2. 15 per level
> 3. 20 per level
> 4. No limit (show all — may be slow for large datasets)"

Save as **LEVEL_LIMIT** (10 / 15 / 20 / 9999).

### B6 — Entity column (detail table)

Show ENTITY_CANDIDATES:

> "Which column identifies **individual records** in the drill-down table?
> (e.g. Contract Number, Order ID, Customer ID, Policy Number)"

Save as **ENTITY_COL**.

> "Which extra columns should appear in the detail table?
> (Pick up to 6 text columns — the two metric columns are always included)"

Show DIM_CANDIDATES + ENTITY_CANDIDATES (excluding ENTITY_COL). Save as **DETAIL_COLS** (max 6).

For each selected DETAIL_COL, ask for a short display header if the column name is cryptic.

### B7 — Measure name

> "What should this measure be called?
> Example: 'Revenue Hierarchy', 'Sales Box Visual', 'Portfolio Overview'"

Auto-check `list_measures`. If the name already exists, append `v2`, `v3`, etc.
Save final name as **MEASURE_NAME** and target table as **TARGET_TABLE** (default: TABLE).

---

## SECTION C — DAX ARCHITECTURE

### C1 — High-level structure

The measure returns a single HTML string. Build it in exactly this sequence:

```
Lines 1–N:  VAR declarations
RETURN
"<!DOCTYPE html>..." & CSS & JS init (to opening <script>)
& "const CY=<_CY>,PY=<_PY>,PY2=<_PY2>,PY3=<_PY3>,MM=<_mm>;"
& "const TG={...total values...};"
& "const TR=[...hierarchy rows JSON...];"
& "const DR=[...detail rows JSON...];"
& "...all JS functions...render();</script></body></html>"
```

### C2 — VAR declarations

```dax
-- Month pointer (defaults to 12 if no month column or no filter active)
VAR _mm = IF(ISBLANK(MAX(TABLE[MONTH_COL])), 12, MAX(TABLE[MONTH_COL]))
-- If MONTH_COL is null: VAR _mm = 12

-- Year pointers
VAR _CY  = MAX(TABLE[YEAR_COL])
VAR _PY  = _CY - 1
VAR _PY2 = _CY - 2
VAR _PY3 = _CY - 3

-- Grand-total scalars: 4 years × 2 metrics = 8 VARs
-- Pattern for _tGC (metric A, current year, YTD up to month _mm):
VAR _tGC = CALCULATE(
    SUM(TABLE[METRIC_A]),
    ALL(TABLE[YEAR_COL]),
    TABLE[YEAR_COL] = _CY,
    REMOVEFILTERS(<month_slicer_table>),   -- omit if no slicer table
    FILTER(ALL(TABLE[MONTH_COL]), TABLE[MONTH_COL] <= _mm)
)
-- Repeat for _tGP (_PY), _tG2 (_PY2), _tG3 (_PY3)
-- Repeat for _tNC / _tNP / _tN2 / _tN3 using METRIC_B

-- Hierarchy summary: cross-join all DIMS, add 4y × 2m = 8 metric columns
VAR _S = CALCULATETABLE(
    SUMMARIZE(TABLE, TABLE[DIM0_COL], TABLE[DIM1_COL], ...),
    ALL(TABLE[YEAR_COL]),
    REMOVEFILTERS(<month_slicer_table>),
    FILTER(ALL(TABLE[MONTH_COL]), TABLE[MONTH_COL] <= _mm)
)
VAR _T = ADDCOLUMNS(_S,
    "cg", CALCULATE(SUM(TABLE[METRIC_A]), ALL(TABLE[YEAR_COL]),
              TABLE[YEAR_COL]=_CY, REMOVEFILTERS(<ms>),
              FILTER(ALL(TABLE[MONTH_COL]), TABLE[MONTH_COL]<=_mm)),
    "pg", CALCULATE(SUM(TABLE[METRIC_A]), ALL(TABLE[YEAR_COL]),
              TABLE[YEAR_COL]=_PY, ...),
    "g2", ..., "g3", ...,
    "cn", ...(METRIC_B)..., "pn", ..., "n2", ..., "n3", ...
)

-- Serialise hierarchy rows: TOPN 3000 by cg desc
VAR _tR = CONCATENATEX(
    TOPN(3000, _T, [cg], DESC),
    "{d1:'" & SUBSTITUTE(IF(ISBLANK([DIM0_COL]),"(None)",[DIM0_COL]),"'","") &
    "',d2:'" & SUBSTITUTE(IF(ISBLANK([DIM1_COL]),"(None)",[DIM1_COL]),"'","") &
    -- ... repeat for each dim ...
    "',cg:" & FORMAT(ROUND(IF(ISBLANK([cg]),0,[cg]),0),"0") &
    ",pg:" & FORMAT(ROUND(IF(ISBLANK([pg]),0,[pg]),0),"0") &
    ",g2:" & FORMAT(ROUND(IF(ISBLANK([g2]),0,[g2]),0),"0") &
    ",g3:" & FORMAT(ROUND(IF(ISBLANK([g3]),0,[g3]),0),"0") &
    ",cn:" & FORMAT(ROUND(IF(ISBLANK([cn]),0,[cn]),0),"0") &
    ",pn:" & FORMAT(ROUND(IF(ISBLANK([pn]),0,[pn]),0),"0") &
    ",n2:" & FORMAT(ROUND(IF(ISBLANK([n2]),0,[n2]),0),"0") &
    ",n3:" & FORMAT(ROUND(IF(ISBLANK([n3]),0,[n3]),0),"0") & "}",
    ","
)

-- Detail rows: entity-level, CY + PY only
VAR _DS = CALCULATETABLE(
    SUMMARIZE(TABLE, TABLE[ENTITY_COL], TABLE[DIM0_COL], ...),
    ALL(TABLE[YEAR_COL]),
    FILTER(ALL(TABLE[YEAR_COL]), TABLE[YEAR_COL]=_CY || TABLE[YEAR_COL]=_PY),
    REMOVEFILTERS(<ms>),
    FILTER(ALL(TABLE[MONTH_COL]), TABLE[MONTH_COL] <= _mm)
)
VAR _D = ADDCOLUMNS(_DS,
    "lbl1", CALCULATE(MAX(TABLE[DETAIL_COL1]), ALL(TABLE[YEAR_COL]), ...),
    ...,
    "cg",  CALCULATE(SUM(TABLE[METRIC_A]),  ALL(TABLE[YEAR_COL]), TABLE[YEAR_COL]=_CY, ...),
    "pg",  CALCULATE(SUM(TABLE[METRIC_A]),  ALL(TABLE[YEAR_COL]), TABLE[YEAR_COL]=_PY, ...),
    "cn",  CALCULATE(SUM(TABLE[METRIC_B]),  ALL(TABLE[YEAR_COL]), TABLE[YEAR_COL]=_CY, ...),
    "pn",  CALCULATE(SUM(TABLE[METRIC_B]),  ALL(TABLE[YEAR_COL]), TABLE[YEAR_COL]=_PY, ...)
)
VAR _dR = CONCATENATEX(
    TOPN(3000, _D, [cg], DESC),
    "{id:'" & SUBSTITUTE(IF(ISBLANK([ENTITY_COL]),"",[ENTITY_COL]),"'","") &
    "',l1:'" & SUBSTITUTE(IF(ISBLANK([lbl1]),"",[lbl1]),"'","") &
    -- ... all DETAIL_COLS ...
    "',cg:" & FORMAT(ROUND(IF(ISBLANK([cg]),0,[cg]),0),"0") &
    ",pg:" & FORMAT(ROUND(IF(ISBLANK([pg]),0,[pg]),0),"0") &
    ",cn:" & FORMAT(ROUND(IF(ISBLANK([cn]),0,[cn]),0),"0") &
    ",pn:" & FORMAT(ROUND(IF(ISBLANK([pn]),0,[pn]),0),"0") & "}",
    ","
)
```

### C3 — HTML layout skeleton

```html
<!DOCTYPE html><html><head><meta charset='UTF-8'>
<style>
  /* Inter font via Google Fonts */
  @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600&display=swap');
  * { box-sizing:border-box; margin:0; padding:0 }
  html, body { font-family:'Inter','Segoe UI',sans-serif; background:#eef2f7;
               height:100%; overflow:hidden }
  #app { display:flex; flex-direction:column; height:100vh }
  #cb  { /* control bar */ }
  #main { display:flex; flex:1; overflow:hidden }
  #cvw { /* canvas wrapper — pan/zoom container */ flex:1; position:relative;
          overflow:hidden; cursor:grab; user-select:none }
  #cv  { /* boxes are absolutely positioned inside here */ position:absolute }
  #rp  { /* right panel — detail table */ width:44%; overflow:hidden;
          display:flex; flex-direction:column }
</style>
</head><body>
<div id='app'>
  <div id='cb'></div>
  <div id='main'>
    <div id='cvw'><div id='cv'></div></div>
    <div id='rp'></div>
  </div>
</div>
<div id='_ntt'></div>   <!-- tooltip -->
<script>
/* --- DAX-injected data --- */
const CY=<<_CY>>, PY=<<_PY>>, PY2=<<_PY2>>, PY3=<<_PY3>>, MM=<<_mm>>;
const TG={cg:<<_tGC>>,pg:<<_tGP>>,g2:<<_tG2>>,g3:<<_tG3>>,
          cn:<<_tNC>>,pn:<<_tNP>>,n2:<<_tN2>>,n3:<<_tN3>>};
const TR=[<<_tR>>];   // hierarchy rows
const DR=[<<_dR>>];   // entity-level detail rows
```

### C4 — DIMS definition

```js
const DIMS = {
  d1:{k:'d1', lb:'<<DIMS[0].label>>'},
  d2:{k:'d2', lb:'<<DIMS[1].label>>'},
  // up to d5
};
const AD = ['d1','d2', /* all keys */ ];   // all available dim keys
```

### C5 — State variables

```js
var _ST = {};

/* --- persistent state helpers --- */
function _sv(k,v) {
  _ST[k]=v;
  try { window.parent.postMessage({type:'pbifch_state',state:_ST},'*'); } catch(e){}
  try { localStorage.setItem('<<STPREFIX>>'+k, JSON.stringify(v)); } catch(e){}
}
function _lv(k,fb) {
  try { if(typeof window.__pbifch_lv==='function'){
    var r=window.__pbifch_lv(k,null); if(r!==null&&r!==undefined)return r; } } catch(e){}
  if(_ST.hasOwnProperty(k)) return _ST[k];
  try { var s=localStorage.getItem('<<STPREFIX>>'+k);
        if(s!==null) return JSON.parse(s); } catch(e){}
  return fb;
}
/*
  State prefix (STPREFIX): first 4–6 chars of table name, lowercase, no spaces + 'bx_'
  Examples: 'salesbx_', 'revbx_', 'portbx_'
*/

/* --- state initialisation --- */
var _rm  = _lv('m','a');       // metric mode: 'a'=METRIC_A, 'b'=METRIC_B
var _ro  = _lv('ori','h');     // orientation: 'h'=horizontal, 'v'=vertical
var _rd  = _lv('d',['d1','d2']); // active dim keys
var _rs  = _lv('s',null);      // selected box key
var _re  = _lv('e',null);      // expand mode: null / 'box' / 'tbl'
var _rf  = _lv('f',null);      // focus key
var _rz  = _lv('z',1);         // zoom level
var _rcy = _lv('cy',[1]);      // active year comparisons: always [1], optional +2/+3
var _rtt = _lv('tt',true);     // tooltips on/off
// Clamp and validate loaded values
if(!Array.isArray(_rd)||!_rd.length) _rd=['d1','d2'];
_rd=_rd.filter(function(x){return AD.indexOf(x)>=0;});
if(!Array.isArray(_rcy)) _rcy=[1];
_rcy=[1].concat(_rcy.filter(function(y){return y===2||y===3;}));
_rz=Math.max(0.25,Math.min(2,typeof _rz==='number'?_rz:1));
// Write back validated state
_ST.m=_rm; _ST.ori=_ro; _ST.d=_rd; _ST.s=_rs;
_ST.e=_re; _ST.f=_rf; _ST.z=_rz; _ST.cy=_rcy; _ST.tt=_rtt;
try{window.parent.postMessage({type:'pbifch_state',state:_ST},'*');}catch(e){}
```

### C6 — Required JavaScript functions

Implement every function below. Never omit any.

| Function | Signature | Purpose |
|---|---|---|
| `fmt(v)` | `fmt(number)` | Format: ≥1M → "1.23M", ≥1K → "1.23K", else integer. Uses de-DE locale separators. |
| `fmtN(v)` | `fmtN(number)` | Integer with de-DE thousand separator. |
| `vrc(cy,py)` | `vrc(number,number)` | Returns `{a:'+1.2M', p:'+5.3%', c:'pos'/'neg'}`. When py=0 returns `{a:'n/a',p:'n/a',c:'neg'}`. |
| `esc(v)` | `esc(any)` | HTML-escape `&→&amp;`, `<→&lt;`, `>→&gt;`. |
| `gBy(arr,k)` | `gBy(array,string)` | Group array of objects by property `k`. Returns `{key:[items]}`. |
| `mk()` | `mk()` | Returns `{c,p,y2,y3}` key strings based on `S.m` ('a'→cg/pg/g2/g3; 'b'→cn/pn/n2/n3). |
| `bTree()` | `bTree()` | Clears `BN[]`. Builds root node from `TG`. Recursively expands via `S.dims`. Stores all nodes in `BN`. |
| `layout(root)` | `layout(node)` | Assigns `x,y` to every node for horizontal (left→right) or vertical (top→down) layout. Returns root. |
| `allN(r)` | `allN(node)` | Returns flat array of all nodes in subtree. |
| `allE(r)` | `allE(node)` | Returns array of `[parent,child]` edge pairs. |
| `ndH(n,i)` | `ndH(node,int)` | Returns HTML string for a single box (node). Includes label, value, progress bar, variance rows. |
| `ctrlH()` | `ctrlH()` | Returns HTML string for the control bar. Includes metric toggle, orientation toggle, dim pills (draggable), red drop zone, + Ebene button, year-comparison toggles, zoom controls, tooltips toggle, expand button, help (?) button. |
| `treeH()` | `treeH()` | Returns HTML string for the full canvas area (SVG edges + box divs). Calls `bTree()` then `layout()`. |
| `tblH()` | `tblH()` | Returns HTML string for the right-panel detail table. Calls `drRows()` for filtered data. |
| `drRows()` | `drRows()` | Returns filtered `DR` rows matching the selected box path. |
| `cpTbl()` | `cpTbl()` | Copies the current detail table to clipboard as HTML (Clipboard API + fallback execCommand). Pastes into Excel with formatting. |
| `fitTree()` | `fitTree()` | Auto-fits zoom and pan so all boxes are visible in the canvas viewport. |
| `initPan()` | `initPan()` | Attaches mouse/wheel/touch event listeners for pan (drag) and zoom (wheel/pinch). Idempotent — safe to call on every render. |
| `resetPan()` | `resetPan()` | Resets pan to initial offset (14px, 12px). |
| `render()` | `render()` | Master render function. Updates `#cb`, `#cvw`, `#rp` innerHTML. Calls `initPan()`. |
| `sm(i)` | `sm(0/1)` | Switch metric: 0=METRIC_A, 1=METRIC_B. Resets selection and focus. Calls `_sv` + `render()`. |
| `soOri(i)` | `soOri(0/1)` | Switch orientation: 0=horizontal, 1=vertical. Calls `_sv` + `render()`. |
| `sn(i)` | `sn(int)` | Select/deselect box at index `i` in `BN`. Ignores click if drag moved >3px. |
| `setFoc(e,i)` | `setFoc(Event,int)` | Toggle focus on node `i`. If focused, tree shows only that subtree. |
| `clrFoc()` | `clrFoc()` | Clear focus, re-render full tree. |
| `rd(i)` | `rd(int)` | Remove dim at index `i` from `S.dims`. Reset selection. Re-render. |
| `addD(i)` | `addD(int)` | Add the i-th available (not yet active) dim. Re-render. |
| `ta()` | `ta()` | Toggle the "add dimension" dropdown. |
| `tCy(y)` | `tCy(2/3)` | Toggle year 2 or year 3 comparison on/off. Always keep year 1 active. |
| `zm(dir)` | `zm(1/-1)` | Step zoom up or down using ZSTEPS array, keeping canvas centre fixed. |
| `tgTT()` | `tgTT()` | Toggle tooltips on/off. |
| `showTT(e,i)` | `showTT(Event,int)` | Show hover tooltip for node `i`. |
| `posTT(e)` | `posTT(Event)` | Position tooltip near cursor, flipping at viewport edges. |
| `hideTT()` | `hideTT()` | Hide tooltip. |
| `xpT()` | `xpT()` | Expand tree full-width (hide right panel). |
| `xpR()` | `xpR()` | Expand right panel full-width (hide tree). |
| `xpC()` | `xpC()` | Return to split view. |
| `dstart(e,i)` | `dstart(Event,int)` | Dragstart on a dim pill. Sets `_di=i`. |
| `dend(e)` | `dend(Event)` | Dragend. Removes `.dov` class. Hides red drop zone highlight. |
| `dover(e)` | `dover(Event)` | Dragover a dim pill (for reorder). |
| `ddrop(e,i)` | `ddrop(Event,int)` | Drop on dim pill → reorder `_di` to position `i`. |
| `rmOver(e)` | `rmOver(Event)` | Dragover red zone → add `.rov`. |
| `rmLeave(e)` | `rmLeave(Event)` | Dragleave red zone → remove `.rov`. |
| `rmDrop(e)` | `rmDrop(Event)` | Drop on red zone → call `rd(_di)`. |

### C7 — Box node HTML structure

Each box (`ndH`) must render:

```html
<div class='nd lv{0-3}' style='left:{x}px;top:{y}px'
     onclick='sn({i})' onmouseenter='showTT(event,{i})' onmouseleave='hideTT()'
     onmousemove='posTT(event)'>
  <div class='nhead'>
    <div class='nlb'>{node.lb}</div>
    <!-- focus button only on non-root nodes: -->
    <button class='fbtn' onclick='setFoc(event,{i})'>f</button>
  </div>
  <div class='nv'>{fmt(node.cv)}</div>
  <div class='nb'><div class='nf' style='width:{node.pct}%'></div></div>
  <!-- PY variance row (always): -->
  <div class='cr'><span class='cyl'>PY</span><span class='vc {pos/neg}'>{arrow} {abs} {pct}</span></div>
  <!-- 2Y row (only if S.cy includes 2): -->
  <div class='cr'><span class='cyl'>2Y</span>...</div>
  <!-- 3Y row (only if S.cy includes 3): -->
  <div class='cr'><span class='cyl'>3Y</span>...</div>
</div>
```

Level colours: `lv0=#0d2460`, `lv1=#1a449e`, `lv2=#2563eb`, `lv3=#0369a1` (left-border accent).

Box size: **W=166px**, **H=96 + (extra_year_rows × 20)px**. Gaps: H-gap=58px, V-gap=8px.

### C8 — Control bar must include the red drop-to-remove zone

```js
// Inside ctrlH(), after rendering dim pills:
if (S.dims.length > 0) {
  h += "<span id='_rmz' class='rmzone' " +
       "ondragover='rmOver(event)' ondragleave='rmLeave(event)' " +
       "ondrop='rmDrop(event)'>&#128465; Drop to Remove</span>";
}
```

CSS for `.rmzone`:
```css
.rmzone { display:inline-flex; align-items:center; gap:4px; padding:3px 12px;
  background:#fff0f0; border:2px dashed #fca5a5; border-radius:14px;
  font-size:11px; font-weight:700; color:#dc2626; cursor:default;
  transition:all .18s; margin-left:4px }
.rmzone.rov { background:#fee2e2; border-color:#ef4444; color:#b91c1c;
  box-shadow:0 0 0 2px #fecaca }
```

### C9 — Zoom levels

```js
const ZSTEPS = [0.25,0.30,0.35,0.40,0.45,0.50,0.55,0.60,0.65,0.70,
                0.75,0.80,0.85,0.90,0.95,1.00,1.05,1.10,1.15,1.20,
                1.25,1.30,1.35,1.40,1.45,1.50,1.55,1.60,1.65,1.70,
                1.75,1.80,1.85,1.90,1.95,2.00];
```

Zoom on wheel: use cursor position as zoom origin (world-space fixpoint formula).
Zoom on +/- buttons: use canvas centre as origin.

### C10 — Pan and touch

Pan: mousedown + mousemove on `#cvw`. Apply `transform: translate({px}px,{py}px) scale({zoom})` to `#cv`.
Touch pinch: track two-finger distance change → scale zoom.
Save zoom to state after wheel debounce (400ms); save after touch-end.

### C11 — Detail table

Columns: ENTITY_COL | DETAIL_COLS... | CY YTD ▼ | PY YTD | Var (€/units) | Var %

- Header row: dark blue `#1e3a8a` with white text, sticky `position:sticky; top:0`.
- Footer row: same style, shows totals for selected node.
- Variance cells: green pill `.vpill.pos` or red pill `.vpill.neg`.
- Copy button: green Excel-style button in table header; uses Clipboard API with `text/html` MIME + fallback.
- Count badge: shows `{n} items` in header.

### C12 — Help overlay (?)

Include a 17-step built-in guide accessible via the `?` button. Steps cover:
overview, metric toggle, orientation toggle, dimension pills, drag-to-reorder, drop-to-remove,
add dimension, year comparisons, clicking a box, focus button, clearing focus, detail table,
panning, zoom controls, tooltips toggle, expand tree, expand table, Excel copy.

Each step has: title, a visual HTML mock, and a plain-language description.

### C13 — Edge rendering (SVG)

Render bezier edges between parent and child boxes:
- Horizontal layout: cubic bezier from right-centre of parent to left-centre of child.
- Vertical layout: cubic bezier from bottom-centre of parent to top-centre of child.
- Focused path: animated dashed green stroke (`stroke-dasharray:10 6; animation: flowPath 1.1s linear infinite`).
- Normal path: `stroke:#bfdbfe; stroke-width:2`.

SVG is `position:absolute; top:0; left:0; pointer-events:none`.

---

## SECTION D — VALIDATION AND TESTING

### D1 — Pre-deploy JS audit

Before calling `create_measure`, mentally check every point in this list:

- [ ] No DAX reserved word used as a VAR name (see Section A8)
- [ ] All `function` declarations are closed with `}`
- [ ] All ternary operators use `:` not `=`
- [ ] All single quotes inside JS string literals are escaped as `\'`
- [ ] All backtick/template strings avoided (use string concatenation only)
- [ ] The final line of JS is `render();`
- [ ] The DAX RETURN string ends with `render();</script></body></html>"`
- [ ] No stray `id]` or other bracket errors (grep the JS for `id]`)
- [ ] All `querySelectorAll` and `querySelector` strings are valid CSS selectors
- [ ] `BN` array is populated by `bTree()` before any layout/render call

### D2 — Post-deploy live test

Run immediately after `create_measure` succeeds:

```dax
EVALUATE ROW(
  "len",   LEN([<<MEASURE_NAME>>]),
  "start", LEFT([<<MEASURE_NAME>>], 120)
)
```

**Pass criteria:**
- `len` > 2000
- `start` begins with `<!DOCTYPE html>`

**If len < 100 or blank:** the DAX is returning blank. Check:
1. `EVALUATE ROW("cy", MAX(TABLE[YEAR_COL]))` — confirm year column is populated.
2. `EVALUATE ROW("tot", SUM(TABLE[METRIC_A]))` — confirm metric is not empty.
3. Remove all filters and test bare CALCULATE.

### D3 — Self-diagnosis lookup table

| Symptom | Most likely cause | Fix |
|---|---|---|
| `create_measure` fails: "Column not found" | Column name has extra space or wrong case | Re-read schema, copy exactly |
| `create_measure` fails: "Exceeds string length" | TOPN too large | Reduce to 2000, then 1500 |
| `create_measure` fails: "Circular dependency" | Metric total VAR references a column that itself depends on a measure in the same table | Move total VARs above _S/_T; use `ALL()` explicitly |
| `create_measure` fails: "Expected )" | Mismatched parentheses in DAX | Count open/close parens in each VAR |
| `execute_dax` returns len=0 | YEAR_COL filter wrong, or table has no data for the current context | Test year filter independently |
| `execute_dax` returns "SyntaxError" in start | JS syntax error — ternary `=` instead of `:` | Scan all `? ... :` patterns |
| Visual is blank (no error) | JS runtime error | Check browser console (F12 in HTML Viewer) |
| Boxes render but show "0" | `cg` key mismatch between TR rows and `mk()` | Confirm key names match exactly |
| Drag-and-drop doesn't work | Event handler name typo | Check `dstart`, `dend`, `dover`, `ddrop`, `rmOver`, `rmLeave`, `rmDrop` |
| State not persisting after filter change | `_sv` not called on every state change | Check every `S.xxx=` assignment is followed by `_sv('xxx', S.xxx)` |

---

## SECTION E — REPORTING TO USER

On success:

> "Done! **<<MEASURE_NAME>>** is ready in your Power BI model.
>
> **How to use it:**
> 1. Add an **HTML Viewer** visual to your report canvas.
> 2. Drag **<<MEASURE_NAME>>** into the visual's Values field.
>
> **Features at a glance:**
> - Boxes are arranged in a hierarchy: <<DIMS[0].label>> → <<DIMS[1].label>> → ...
> - Drag any blue dimension pill to **reorder** hierarchy levels
> - Drag a pill onto **🗑 Drop to Remove** to remove a level
> - Click **+ Level** to add a level back
> - Click any box to see matching records in the detail table on the right
> - Press **f** on any box to focus on just that branch
> - Toggle between **<<LABEL_A>>** and **<<LABEL_B>>** using the top-left buttons
> - Use **⌂** to reset the view, **FIT** to auto-fit, **+/-** to zoom
> - Click **?** for the built-in step-by-step guide"

On failure after 3 retries:

> "I ran into a persistent issue building this visual. Here's what I tried and where it got stuck:
> [brief description]. Would you like me to try a simpler version without [feature that caused the error]?"

---

## SECTION F — PORTING CHECKLIST (for every new dataset)

Complete this checklist silently before writing any DAX:

- [ ] TABLE confirmed via `list_tables`
- [ ] YEAR_COL: integer 4-digit year column confirmed via `get_table_schema`
- [ ] MONTH_COL: integer 1–12 column, or null if not applicable
- [ ] METRIC_A: main numeric column (SUM aggregation)
- [ ] METRIC_B: second metric, or same as METRIC_A
- [ ] DIMS: 2–5 text columns in hierarchy order, keys d1–d5 assigned
- [ ] ENTITY_COL: individual record identifier
- [ ] DETAIL_COLS: up to 6 additional display columns
- [ ] MEASURE_NAME: unique name confirmed via `list_measures`
- [ ] TARGET_TABLE: table to host the measure
- [ ] STPREFIX: lowercase 4–6 char prefix + 'bx_' (e.g. 'salesbx_')
- [ ] No DAX reserved names used as VARs
- [ ] No company-specific strings hardcoded in the JS (labels come from user input or column names)
- [ ] All column names verified against schema (zero guessing)
- [ ] JS audit complete (Section D1)
- [ ] Post-deploy test passes (Section D2)
