# PLAN.md — miraviz: Claude Code plugin for observability data visualisation

> Working name: `miraviz` (rename freely). License: Apache-2.0.
> Reference for structure and conventions: https://github.com/mirastacklabs-ai/platformcosmo
> (copy the *patterns*, not the code).

## 1. Goal

User gives a data file (metrics, logs, traces, profiles) and a plain-English ask.
Claude decides **what to plot and how**, writes a small JSON spec, and a deterministic CLI renders it
as an interactive self-contained HTML file plus static SVG/PNG.

**Core principle:** Claude never reads raw data and never writes chart code or numbers.
- The CLI profiles the data, giving Claude a compact shape summary.
- Claude writes the spec (representation decisions only).
- The CLI validates the spec and renders from the actual data.

## 2. Non-goals (v1)

- No root-cause analysis, anomaly detection or "insights".
- No live backend queries (Prometheus/Loki/Tempo/Pyroscope APIs). Files only.
- No dashboard server, auth or persistence.
- No hand-written SVG/HTML by Claude, ever.

## 3. User flow

```
/miraviz:visualize "p95 latency per service over the last day" ./latency.json
  1. miraviz profile ./latency.json             → shape summary (≤ 4 KB JSON)
  2. Claude: ask + summary → writes latency.miraviz.json (spec)
  3. miraviz validate latency.miraviz.json       → errors with codes + JSON pointer + suggestion
  4. Claude fixes errors, re-validates until "valid"
  5. miraviz render latency.miraviz.json         → latency.html (+ .svg / .png if asked)
  6. Claude reports output paths + one-line rationale for the chosen representation
```

## 4. Repository layout

```
.claude-plugin/marketplace.json
plugin/
  .claude-plugin/plugin.json
  bin/miraviz                 # POSIX sh launcher: checks Node >= 22.12, execs miraviz.mjs
  bin/miraviz.mjs             # single self-contained bundle (tsup/esbuild), committed on release
  skills/
    visualize/SKILL.md       # main workflow
    visualize/references/    # representation-rules.md, spec-reference.md, signal-guides/*.md
    export/SKILL.md          # re-render existing spec to other formats/themes
packages/
  core/          # spec types, parse, validate, semantic checks, did-you-mean, render pipeline
  adapters/      # input parsers → normalized model
  profile/       # shape summary from normalized model
  transform/     # rate, sum/avg by label, percentile, histogram_quantile, top-k, LTTB downsample
  render-echarts/# spec + data → ECharts option (shared by HTML and SSR SVG)
  render-flame/  # flame graph + diff flame graph → SvgNode tree / HTML
  render-trace/  # trace waterfall → SvgNode tree / HTML
  html/          # self-contained HTML template + inliner
  raster/        # SVG → PNG via @resvg/resvg-wasm
  themes/        # light, dark (JSON tokens)
  cli/           # commands, arg parsing, output
schema/v1.json   # JSON Schema (source of truth; TS types generated from it)
examples/        # sample data + specs + rendered outputs (gallery)
tests/           # unit, golden (byte-identical output), e2e CLI
```

Tooling: pnpm workspaces, TypeScript, tsup, vitest, eslint, prettier, changesets. Node >= 22.12.

## 5. Normalized data model (packages/adapters)

All inputs convert to one of four normalized shapes:

| Kind | Shape |
|---|---|
| `metric` | `{ series: [{ labels: Record<string,string>, points: [tsMs, value][] }], meta: { type: counter\|gauge\|histogram\|summary\|unknown, unit?, step? } }` |
| `log` | `{ entries: [{ ts, level?, labels, message }] }` |
| `trace` | `{ spans: [{ traceId, spanId, parentId?, name, service, startNs, endNs, status, attrs }] }` |
| `profile` | `{ samples: [{ stack: string[], value: number }], unit: cpu-ns\|bytes\|count }` |

### v1 input formats

| Format | Kind | Detection |
|---|---|---|
| Prometheus HTTP API `query_range` JSON (`resultType: matrix`) | metric | `data.resultType` |
| CSV (`timestamp,<labels...>,value` or wide format) | metric | header sniff |
| Loki `query_range` JSON (`resultType: streams`) | log | `data.resultType` |
| NDJSON logs (one JSON object per line) | log | first line parse |
| OTLP JSON traces (`resourceSpans`) | trace | top-level key |
| Jaeger JSON export | trace | `data[].spans` |
| Collapsed stacks (`a;b;c 123`) | profile | line regex |
| pprof (`.pb.gz`) | profile | gzip magic + protobuf decode |

Metric type inference: name suffix (`_total` → counter, `_bucket` with `le` → histogram),
monotonic check, explicit override in spec. Adapters must stream / cap memory; reject files > 500 MB with a clear error.

## 6. `miraviz profile` output (what Claude sees)

Hard cap: **4 KB JSON**. Must be enough to choose a representation, nothing more.

```json
{
  "kind": "metric",
  "source": { "file": "latency.json", "format": "prometheus-matrix" },
  "metric": {
    "name": "http_request_duration_seconds_bucket",
    "inferredType": "histogram",
    "unit": "seconds",
    "seriesCount": 412,
    "labels": [
      { "name": "service", "cardinality": 12, "sample": ["api", "auth", "cart"] },
      { "name": "le", "cardinality": 14, "role": "bucket" },
      { "name": "pod", "cardinality": 380, "highCardinality": true }
    ],
    "time": { "start": "...", "end": "...", "stepSec": 30, "points": 2880, "gapsPct": 0.4 },
    "value": { "min": 0, "max": 9.8, "spanOrders": 4 }
  },
  "hints": ["histogram: use percentile or heatmap", "pod is high-cardinality: group or top-k"]
}
```

Equivalent summaries for log (count, levels, label cardinality, time range, top message templates ≤ 10),
trace (trace count, span count, services, depth, duration range), profile (sample count, unit, depth, top 10 frames by self).

## 7. Spec (schema/v1.json)

```json
{
  "$schema": "https://<site>/schema/v1.json",
  "version": "1",
  "title": "p95 latency by service",
  "ask": "p95 latency per service over the last day",
  "data": { "main": { "file": "latency.json" } },
  "layout": { "columns": 2 },
  "theme": "light",
  "panels": [
    {
      "id": "p95-by-service",
      "title": "p95 latency",
      "intent": "trend",
      "data": "main",
      "transform": [
        { "op": "histogram_quantile", "q": 0.95, "by": ["service"] },
        { "op": "topk", "k": 10, "by": "max" }
      ],
      "chart": "line",
      "encoding": { "y": { "unit": "seconds", "scale": "linear" }, "color": "service" },
      "rationale": "Histogram metric; user asked per-service trend → p95 line, top 10 services."
    }
  ]
}
```

Rules:
- `intent` is required: `trend | distribution | ranking | comparison | composition | correlation | single-value | log-volume | log-table | trace-waterfall | flame | flame-diff`.
- `rationale` is required (≤ 200 chars) and is shown in the HTML as the panel subtitle/tooltip.
- `transform` ops are a closed set (see §8). No expressions, no code.
- Ids are kebab-case and unique. Text is plain text with length limits. Unknown fields are an error.

## 8. Transforms (packages/transform), deterministic

`rate`, `increase`, `sum|avg|min|max by [labels]`, `histogram_quantile(q, by)`, `percentile(q)` (raw samples),
`topk(k, by: max|avg|last)`, `bottomk`, `filter(label op value)`, `window(start,end)`, `resample(step)`,
`count_over_time(step)` (logs), `group_by_level` (logs), `bucketize` (heatmap), `self_time` / `total_time` (profiles),
`diff(baseline)` (profiles, comparison). Downsampling (LTTB, default ≤ 2000 points per series) is always applied at render, not in the spec.

## 9. Representation rules (validator + references/representation-rules.md)

Intent → allowed charts:

| Intent | Charts |
|---|---|
| trend | line, area |
| distribution | histogram, heatmap, boxplot |
| ranking | bar-horizontal |
| comparison | line (overlay), bar-grouped |
| composition | area-stacked, bar-stacked |
| correlation | line-dual-axis, scatter |
| single-value | stat, gauge |
| log-volume | bar (time) stacked by level |
| log-table | table (virtualized) |
| trace-waterfall | waterfall |
| flame / flame-diff | flamegraph |

Validator errors (with code, JSON pointer and suggestion):
- `E_COUNTER_NO_RATE`: counter plotted raw → suggest `rate`.
- `E_HIST_RAW_BUCKETS`: histogram buckets plotted directly → suggest `histogram_quantile` or heatmap.
- `E_TOO_MANY_SERIES`: > 20 series on line/area after transforms → suggest `topk` or `sum by`.
- `E_INTENT_CHART_MISMATCH`: chart not allowed for intent.
- `E_SIGNAL_CHART_MISMATCH`: e.g. flamegraph on metric data.
- `E_UNKNOWN_LABEL`: label not in profile → did-you-mean.
- `E_UNIT_MIX`: series with different units on one axis → suggest dual axis or split panel.

Warnings: `W_LOG_SCALE` (value span > 3 orders), `W_GAPS` (> 5 % gaps), `W_STACKED_GAUGE` (stacking non-additive gauges).

## 10. Rendering

### 10.1 Interactive HTML (default output)
One self-contained file, works offline, no network requests:

```
<name>.html
├── <style> theme tokens </style>
├── <script> ECharts custom build (only used chart types), inlined </script>
├── <script type="application/json" id="miraviz-data"> transformed + downsampled data </script>
├── <script type="application/json" id="miraviz-config"> per-panel ECharts options </script>
├── <script> bootstrap: read config + data → echarts.init per panel; flame/waterfall widgets </script>
```

Interactions: hover tooltips, time-axis zoom/brush, legend toggle, **synced cursor + zoom across time panels**,
flame graph click-to-zoom + search, waterfall span expand/collapse, log table filter.
Header shows title, ask, data source, time range. Each panel shows its rationale.
Target size: ≤ 3 MB typical. Strict CSP meta tag (`default-src 'none'; script-src 'unsafe-inline'; style-src 'unsafe-inline'`).

### 10.2 Static SVG / PNG
- Metric/log charts: ECharts SSR (`renderer: 'svg', ssr: true` → `renderToSVGString()`), no DOM.
- Flame/waterfall: own renderer emitting an `SvgNode` tree; one serializer handles escaping and number formatting.
- PNG: `@resvg/resvg-wasm` (not native, so the bundle stays portable), bundled fonts.
- Multi-panel: grid composition into one SVG.

### 10.3 Determinism
Same spec + same data + same version = byte-identical output. No `Date.now()`, no random, stable key and attribute order,
fixed number precision. Enforced by golden tests.

## 11. CLI (`miraviz`)

```
miraviz profile <data-file> [--json]
miraviz validate <spec> [--json]
miraviz render <spec> [--html] [--svg] [--png] [--theme light|dark] [--out <dir>]
miraviz init <spec> [--from <data-file>]        # starter spec from profile
miraviz examples list | show <name> | copy <name>
miraviz serve <file.html>                        # optional: open in default browser
```

Exit codes: 0 ok, 1 validation error, 2 input error, 3 internal. `--json` on every command for machine-readable output.
All stdout capped (profile ≤ 4 KB, validate ≤ 50 diagnostics).

## 12. Skills

### `visualize/SKILL.md`
- Frontmatter `allowed-tools`: only `Bash(miraviz profile *)`, `Bash(miraviz validate *)`, `Bash(miraviz render *)`,
  `Bash(miraviz init *)`, `Bash(miraviz examples *)`, plus `node ${CLAUDE_PLUGIN_ROOT}/bin/miraviz.mjs ...` equivalents, `Read`, `Write(*.miraviz.json)`.
- Hard rules: never Read raw data files; never write HTML/SVG/JS; always profile → spec → validate → fix → render;
  every panel needs intent + rationale; prefer 1–4 panels; ask the user only if the ask is ambiguous about the metric or time range.
- Workflow and spec-in-one-screen, as in platformcosmo's diagram skill.
- References: representation rules, spec reference, per-signal guides (metrics, logs, traces, profiles) with worked examples.

### `export/SKILL.md`
Re-render an existing spec to other formats or themes.

## 13. Milestones (each = PR(s) with green CI; acceptance criteria in bold)

| M | Scope | Acceptance |
|---|---|---|
| M0 | Monorepo scaffold, CI, schema v1 skeleton, type generation, plugin manifests, launcher | **`claude plugin install` works; `miraviz --version` runs from plugin bin** |
| M1 | Metrics: Prometheus matrix + CSV adapters, profile, transforms (rate, sum by, histogram_quantile, topk), line/area/bar/stat, HTML output | **E2E: prompt → spec → valid → HTML opens offline with synced zoom; golden tests pass** |
| M2 | Validator semantics + did-you-mean + all §9 rules; SVG via ECharts SSR; PNG | **Each error code has a failing-fixture test; SVG byte-identical across runs** |
| M3 | Profiles: collapsed + pprof adapters, flame graph + diff (HTML interactive + SVG) | **pprof from Go and Pyroscope export renders correctly** |
| M4 | Traces: OTLP + Jaeger adapters, waterfall (critical path highlight is visual only) | **1k-span trace renders < 1 s, HTML < 2 MB** |
| M5 | Logs: Loki + NDJSON adapters, volume-by-level chart, virtualized table | **100k log lines → HTML < 3 MB, table scrolls smoothly** |
| M6 | Multi-panel layouts, heatmap/histogram/boxplot, dual axis, dark theme | **Gallery of ≥ 10 examples generated by CI** |
| M7 | Docs site, npm publish with provenance, marketplace release | **Fresh machine install → first chart in < 2 min** |

## 14. Open decisions (confirm before M1)

1. Files only (assumed), or add `miraviz fetch` for live Prometheus/Loki/Tempo/Pyroscope later? It would write a snapshot file, keeping render deterministic.
2. Own spec (assumed), or Perses dashboard spec compatibility / export?
3. Default output: HTML only, or HTML + SVG?
4. ECharts licence check of any bundled extensions; fonts (OFL) for PNG.

## 15. Testing

- Unit: adapters (fixtures per format, including malformed), transforms (numeric against hand-computed values), validator (one fixture per code).
- Golden: spec + data → byte-identical HTML/SVG; update via explicit `pnpm golden:update`.
- E2E: CLI commands, exit codes, output caps.
- Browser: Playwright loads each gallery HTML, asserts no console errors, no network requests, zoom sync works.
- Skill eval: 20 prompt + data pairs with expected intent/chart; measure the share where the spec validates on the first or second try.

## 16. Security & safety

- Data and text from files are untrusted: escape all labels/messages in SVG and HTML; JSON embedded via `<script type="application/json">` with `<` escaped.
- No network access in CLI v1. HTML output has no external references (CSP enforced).
- Path handling: CLI reads/writes only under the working directory unless `--out` is given.
- Size limits on inputs, series counts and output files, with clear errors.

## 17. Conventions

- Decision log in `DECISIONS.md` (decision, options, reason).
- Changesets for versioning; plugin version synced with the CLI package.
- `THIRD_PARTY_NOTICES.md` for ECharts, resvg and fonts.
