---
name: echarts-to-vchart-migrator
description: "Use when migrating any ECharts-based chart library to VChart. Works on any project that wraps ECharts. Two modes: (1) PROJECT SCAN — auto-discover chart types, adaptor architecture, coupling points, produce a full migration plan with per-chart todos and a runnable HTML side-by-side demo; (2) SINGLE CHART — convert one EChartsOption/MChartOption to VChart spec with risk grading, field mapping table, unsupported items, and validation cases."
---

# ECharts → VChart Migration Skill (Universal)

Two modes depending on what the user provides:

| User Input | Mode |
|---|---|
| File path / "scan my project" / "help me migrate" | **PROJECT SCAN** — full plan |
| A single option JSON snippet | **SINGLE CHART** — spec conversion + report |

> **Project-specific version**: If a `.github/skills/echarts-to-vchart-migrator/SKILL.md` exists in the current workspace, it may supplement this skill with project-specific file paths, known plugin patterns, and current migration status. Read it first and merge the context.

---

## MODE A — PROJECT SCAN

### When to use
- User says "help me migrate", "scan the project", "what charts do we have", "what do I need to do"
- No specific chart config given
- User wants scope and a step-by-step plan

### Step 1 — Auto-Discover Project Structure

Do NOT assume file paths. Run these discovery steps instead:

**1a. Find the adaptor/renderer layer**
```bash
# Look for a pluggable adaptor pattern
grep -r "RenderAdaptor\|registerRenderAdaptor\|chartAdaptor\|renderAdaptor" \
  --include="*.ts" --include="*.tsx" -l
```
Read the files returned. Identify:
- The adaptor interface (what methods: `mounted`, `update`, `resize`, `unmounted`, `effectHooks`?)
- The adaptor registry (how are adaptors registered and looked up?)
- Whether a VChart adaptor already exists

**1b. Find all chart components**
```bash
# Chart components typically live in a chart/ directory
find . -path "*/chart/*/index.tsx" -o -path "*/chart/*/index.ts" | grep -v node_modules
# Also look for plugin files that patch option per chart type
find . -name "plugin.ts" | grep -v node_modules
```

**1c. Find all ECharts series types in use**
```bash
grep -r "type:\s*['\"]bar\|line\|pie\|scatter\|radar\|funnel\|map\|wordCloud\|custom" \
  --include="*.ts" --include="*.tsx" | grep -v node_modules | grep "series"
```

**1d. Find ECharts-specific imports in hooks/utilities**
```bash
grep -r "EChartsType\|EChartsInstance\|import.*echarts" \
  --include="*.ts" --include="*.tsx" | grep -v node_modules | grep -v "adaptor"
```

**1e. Find formatter usages**
```bash
grep -rn "formatter\s*:" --include="*.ts" --include="*.tsx" | grep -v node_modules | grep -v "\.d\.ts"
```

**1f. Find where ECharts and VChart UMD builds are**
```bash
find . -path "*/echarts/dist/echarts.min.js" | grep -v node_modules/.cache | head -3
find . -path "*/@visactor/vchart/build/index.min.js" | grep -v node_modules/.cache | head -3
```
Record these paths — they are needed to generate the HTML demo in Step 6.

### Step 2 — Build Chart Inventory

For each discovered chart component/type, produce a table row:

| Chart Type | Component Path | Plugin Special Logic | Risk | Status | Notes |
|---|---|---|---|---|---|
| (discovered) | (from Step 1b) | (from plugin.ts) | A/B/C/D | not started | |

**Risk level rubric:**
- **A** — Basic declarative (bar, line, area, pie, scatter). No custom ECharts extensions. VChart has a direct declarative equivalent. `normalizeOption → toVChartSpec` can auto-convert.
- **B** — Moderate. VChart has support but spec structure diverges significantly (e.g., radar `indicator[]`, funnel), OR upstream plugin mutates series data before the adaptor sees it.
- **C** — High. ECharts-exclusive feature, custom series extension (`type: 'custom'`), visualMap+geo, or fundamentally different rendering model (map, mixed-type combination charts).
- **D** — No VChart equivalent. ECharts-only plugin (e.g., heavily customized word cloud layout, proprietary custom series). Must keep ECharts adaptor permanently.

### Step 3 — Identify Structural Coupling Points

Always check all four coupling categories, even if none seem blocking at first glance.

**Category 1: Plugin-layer data mutation**

Some chart libraries have a "plugin" or "option processor" layer that runs before the adaptor and can rewrite `series[].data`. Common patterns to search for:
```bash
grep -rn "item\.data\s*=" --include="*.ts" -A 2 | grep -v node_modules
grep -rn "series\.forEach\|seriesArr\.map" --include="plugin.ts" | grep -v node_modules
```

If found: the normalizer (`normalizeOption`) receives already-transformed data. Features that VChart handles declaratively (like `stack: 'percent'`, small-value merging in pie) will be broken unless:
- The normalizer runs upstream of the plugin layer, OR
- The plugin either skips mutation when `adaptor = 'vchart'` is detected, OR
- The normalizer detects and reverses the mutation (fragile, not recommended)

**Category 2: ECharts-specific types in shared hooks/utilities**

Found via Step 1d. Files that import `EChartsType` directly will cause TypeScript errors or incorrect behavior at runtime when switching to the VChart adaptor. 

Resolution: introduce a runtime-neutral instance interface (e.g., `ChartInstance = EChartsType | VChartInstance`) or make hooks adaptor-aware via the `$adaptor` ref that the render layer already injects.

**Category 3: Function formatters**

Found via Step 1e. ECharts `formatter: (params) => string` callbacks receive an ECharts-specific `params` object. These cannot be transparently carried to VChart — VChart uses `formatMethod` with a different signature.

For each formatter found: record it in `unsupportedMappings` with `impact: "visual-parity-loss"`. Provide the VChart `formatMethod` equivalent signature so the developer knows exactly what to rewrite.

**Category 4: Theme token coupling**

Find the project's theme type definition:
```bash
grep -rn "MChartTheme\|ChartTheme\|ThemeConfig" --include="*.ts" -l | grep -v node_modules
```
If theme maps ECharts-specific token names (`backgroundColor`, `textStyle`, `axisPointer`) directly, these need re-mapping to VChart's theme schema. This is usually a separate tracking item rather than a blocker.

### Step 4 — Check VChart Adaptor Readiness

Search for an existing VChart adaptor file:
```bash
find . -name "vchartAdaptor*" | grep -v node_modules
```

If it exists, verify these checklist items by reading the file:
- [ ] `mounted`: converts option via `normalizeOption + toVChartSpec` before calling `new VChart()`
- [ ] `mounted`: passes `dom`, `width`, `height`, `devicePixelRatio` to VChart constructor
- [ ] `update`: calls `instance.updateSpec(spec, false, { reuse: true })` not `setOption`
- [ ] `resize`: calls `instance.resize({ width, height })`
- [ ] `unmounted`: calls `instance.release()`
- [ ] `effectHooks`: re-inits on `devicePixelRatio / renderType / theme` changes
- [ ] VChart is loaded as `optionalDependencies` (not hard dep) — check `package.json`
- [ ] Adaptor is registered with a key (e.g., `registerRenderAdaptor(vchartAdaptor, 'vchart')`)

If the adaptor does not exist yet, generate it: implement all lifecycle methods using the interface discovered in Step 1a.

### Step 5 — Generate Per-Chart Todo List

For each chart type discovered, produce a checklist. Mark items as `[x]` only if verified by reading the actual code.

```
[ ] bar
    [ ] normalizeOption: series[].type = 'bar' → 'bar' ✓ / needs implementation?
    [ ] toVChartSpec: single-series + multi-series flat table handled?
    [ ] toVChartSpec: stack handled?
    [ ] Plugin mutation: does any plugin.ts rewrite bar series data? If yes → blocking issue
    [ ] Formatters: any tooltip/label formatters? Count: N
    [ ] Test: render parity verified in HTML demo

[ ] pie
    [ ] normalizeOption: type = 'pie' → pieSeries path handled?
    [ ] toVChartSpec: valueField/categoryField mapped?
    [ ] Plugin mutation: any pre-processing of pie data (small value merging, etc.)?
    [ ] doughnut support: innerRadius/outerRadius handled?
    [ ] Test: render parity verified in HTML demo

[ ] radar   ← usually B class
    [ ] indicator[] structure: needs categoryField mapping in VChart
    [ ] Multi-dim data: each dimension becomes a flat row
    [ ] areaStyle on radar series: VChart supports but different field name
    [ ] Test: render parity verified in HTML demo

... (one block per discovered chart type, never skip any)
```

### Step 6 — Generate Runnable HTML Demo

Produce a self-contained HTML file — no build system, no npm install, open directly in browser.

**Path selection**: Place the file at the repo root if possible. Use relative `./node_modules/` paths to reference UMD bundles found in Step 1f.

**Structure:**
1. `<script src="...echarts.min.js">` and `<script src="...vchart/build/index.min.js">`
2. Side-by-side layout: ECharts (left) vs VChart (right), same `MChartOption` input
3. Tab bar to switch between all discovered chart types that have normalizer coverage
4. Collapsible panel showing: input `MChartOption` | intermediate `NormalizedChartSpec` | final `VChart ISpec`
5. Unsupported/incomplete chart types show a warning banner (`⚠️ not yet migrated — keeping ECharts`) instead of crashing

**Critical rule**: The inline JS normalizer and transformer in the HTML must be a faithful copy of the actual `normalizeOption` and `toVChartSpec` implementation (or the compiled output). Do not simplify. If `vchartSpec.ts` is updated, the demo must be updated to match.

**VChart UMD export detection** (VChart's UMD export shape varies by version):
```js
const VChartCtor = window.VChart?.VChart || window.VChart || null;
```

### Step 7 — Produce Migration Plan

Output a structured markdown plan:

```markdown
## Migration Status

Adaptor layer: [✅ registered / ⚠️ exists but incomplete / ⬜ not started]
Normalizer coverage: X/N chart types covered (A class: N, B class: N, C class: N)

## Chart-by-Chart Breakdown

### A class — auto-migrate
...

### B class — semi-auto (spec design needed)
...

### C class — manual (keep ECharts for now)
...

### D class — permanent ECharts (no VChart equivalent)
...

## Blocking Issues (must resolve before switching)
1. [chartType] description — suggested fix
...

## Recommended Rollout Order
Phase 1: A-class charts with no formatters and no plugin data mutation
Phase 2: A-class charts that have plugin data mutation issues (fix timing first)
Phase 3: B-class charts (design VChart spec for each)
Phase 4: Hook layer decoupling (remove EChartsType dependencies from shared hooks)
Phase 5: C-class charts (evaluate VChart capability, may need to keep ECharts)
```

---

## MODE B — SINGLE CHART CONVERSION

### When to use
- User pastes a specific `EChartsOption` or `MChartOption`
- User asks "convert this chart" or "what's the VChart spec for this"

### Step 1 — Detect Source Format

- `echarts-option` — raw ECharts option object
- `mchart-option` — wrapper library option (may have extra preset fields, adaptor key, etc.)
- `business-schema` — higher-level schema that generates one of the above at runtime

### Step 2 — Feature Inventory

Extract from the provided option:
- `series[].type` — main chart type(s)
- Coordinate model: `xAxis/yAxis` | `radar` | `geo` | `polar`
- Data binding: `dataset/encode` vs inline `series.data`
- Formatters: `tooltip.formatter`, `label.formatter`, `axisLabel.formatter` — note each one's complexity (string template vs function)
- Advanced features: `markLine`, `markArea`, `markPoint`, `visualMap`, `dataZoom`, `brush`, `graphic`
- Custom/extension series: `type: 'custom'` or third-party type strings
- Event handlers referenced in the option object

### Step 3 — Risk Grading

| Level | Conditions |
|---|---|
| **A** | All series declarative, no formatters, no markLine/graphic/custom series, no non-cartesian/non-pie coords |
| **B** | Has formatters OR uses moderate ECharts features (markLine, dataZoom, polar) that VChart supports but needs manual mapping |
| **C** | Has `graphic`, `custom` series, mixed series types, or `combinedChart` needed in VChart |
| **D** | ECharts-only extension type, proprietary plugin, or feature with no VChart equivalent |

### Step 4 — Walk Through Normalizer Logic

Apply the following rules to the provided option mentally (or using the actual `normalizeOption` implementation if available in the project):

1. `series[0].type` → which `SupportedChartType` does it resolve to?
2. `type = 'line'` + `areaStyle != null` → promote to `'area'`
3. `type = 'pie'` or `'doughnut'` → `pieSeries` path
4. `xAxis[0].data` → categories array
5. Multiple series → flat table with `seriesField` column
6. `series[i].stack` → `vcSpec.stack = true`
7. Any series type not in the supported list → `null` returned → C/D risk

### Step 5 — Generate Outputs

Produce all three objects:

1. **`normalizedSpec`** — the intermediate semantic layer (runtime-neutral)
2. **`targetVChartSpec`** — the final VChart ISpec ready to pass to `new VChart(spec, { dom })`
3. **`mappingNotes`** — for every field in the source option, show:
   - `sourcePath` → `targetPath`
   - `status`: `"mapped"` | `"partially-mapped"` | `"not-mapped"`
   - `note`: why, and what manual action is needed if not mapped

### Step 6 — Unsupported Mappings

Every source field with no VChart equivalent must appear here with:
```json
{
  "sourcePath": "tooltip.formatter",
  "targetPath": "tooltip.formatMethod",
  "status": "not-mapped",
  "reason": "ECharts formatter params shape differs from VChart formatMethod params",
  "impact": "visual-parity-loss",
  "manualAction": "Rewrite using VChart formatMethod: (text, datum, model, datum2, extra) => string"
}
```

Impact levels: `"visual-parity-loss"` | `"interaction-loss"` | `"data-loss"` | `"cosmetic-only"`

### Step 7 — Validation Cases

Produce at minimum 3 test cases:
1. **Static** — does VChart accept this spec without throwing? (check required fields)
2. **Render parity** — does the chart visually match ECharts for the same data?
3. **Edge case** — null values in data, empty series, single data point

### Step 8 — Rollout Decision

```
"migrate-now"         → riskLevel A, no non-cosmetic unsupported mappings
"migrate-with-fallback" → riskLevel B, manual items exist but adaptor='echarts' fallback is available
"keep-echarts"        → riskLevel C/D, blocking gaps with no VChart equivalent
```

---

## Hard Rules (both modes)

1. **Never claim full parity** if any `unsupportedMappings` entry has `impact` other than `"cosmetic-only"`.
2. **Formatters are always `"not-mapped"`** until explicitly rewritten with VChart signature — never silently drop them.
3. **Plugin data mutations must be called out** as blocking issues for any chart type where they were found — even if the mutation seems minor.
4. **Risk level D requires an explicit note**: "Keep ECharts adaptor for this chart type unless VChart adds native support."
5. **HTML demo inline JS must faithfully mirror the actual normalizer/transformer** — do not simplify, abbreviate, or paraphrase the logic.
6. **Never skip chart types in the todo list** — every type discovered in the codebase gets an entry, even if it's just `⬜ not assessed yet`.
7. **Discovery before assumption** — do not assume file paths based on mchart conventions. Always run the grep/find commands in Step 1 first.

---

## Output Contract

**Mode A** → markdown documents:
- Chart inventory table
- Coupling point analysis
- Per-chart todo checklist
- Migration plan with rollout phases
- HTML demo file (written to disk at repo root)

**Mode B** → JSON matching `templates/migration-output.schema.json`

Do not mix formats. Mode A primary output is always markdown + HTML file, not JSON.
