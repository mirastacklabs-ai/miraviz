# PLAN.md — miraviz: Claude Code plugin for observability data visualisation

> Working name: `miraviz` (rename freely). License: Apache-2.0.
> Reference for structure and conventions: https://github.com/mirastacklabs-ai/platformcosmo
> (copy the *patterns*, not the code).

## 1. Goal

User gives a data file **or a configured data source** (metrics, logs, traces, profiles) and a plain-English ask.
Claude decides **what to query, what to plot and how**, writes a small JSON spec, and a deterministic CLI renders it
as an interactive self-contained HTML file plus static SVG/PNG.

Two output modes:
- **Snapshot (default):** static, deterministic, offline, safe to share. Built from files or fetched snapshots.
- **Live:** `miraviz serve --live` runs a local server; the page gets a time-range picker, refresh and a
  data-source switcher, and re-queries backends through the server.

**Core principle:** Claude never reads raw data and never writes chart code or numbers.
- The CLI fetches and profiles the data, giving Claude a compact shape summary.
- Claude writes the spec (queries + representation decisions only).
- The CLI validates the spec and renders from the actual data.

## 2. Non-goals (v1)

- No root-cause analysis, anomaly detection, alerting or "insights".
- No browser-to-backend calls: the page never holds credentials or talks to anything but its own local server.
- No query editing in the page: queries come from the spec. miraviz is not a Grafana replacement.
- No multi-user server, auth or persistence: the live server is local, single-user, loopback-only.
- No commercial SaaS backends (Datadog, New Relic, CloudWatch…) in v1.
- No hand-written SVG/HTML by Claude, ever.

## 3. User flow

### 3.1 From a file
```
/miraviz:visualize "p95 latency per service over the last day" ./latency.json
  1. miraviz profile ./latency.json             → shape summary (≤ 4 KB JSON)
  2. Claude: ask + summary → writes latency.miraviz.json (spec)
  3. miraviz validate latency.miraviz.json       → errors with codes + JSON pointer + suggestion
  4. Claude fixes errors, re-validates until "valid"
  5. miraviz render latency.miraviz.json         → latency.html (+ .svg / .png if asked)
  6. Claude reports output paths + one-line rationale for the chosen representation
```

### 3.2 From a live source
```
/miraviz:visualize "5xx rate per service, last 6h" --source prom-prod
  1. miraviz sources list --json                 → configured sources (names, types; never secrets)
  2. Claude writes spec: source + query per data entry, default time range
  3. miraviz fetch errors.miraviz.json           → snapshots in .miraviz/snapshots/ (or F_* backend errors)
  4. miraviz profile <snapshot>                  → shape summary
  5. validate → fix query or representation → re-fetch if a query changed
  6. miraviz render errors.miraviz.json          → static snapshot HTML
     miraviz serve --live errors.miraviz.json    → live page on 127.0.0.1 (if the user wants live)
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
    visualize/references/    # representation-rules.md, spec-reference.md, signal-guides/*.md, query-guides/*.md
    export/SKILL.md          # re-render existing spec to other formats/themes
packages/
  core/          # spec types, parse, validate, semantic checks, did-you-mean, render pipeline
  adapters/      # response/file parsers → normalized model (one per wire format)
  connectors/    # backend query clients (HTTP, auth, pagination, limits), one per protocol (§5.3)
  snapshot/      # snapshot file format: normalized model + provenance
  profile/       # shape summary from normalized model
  transform/     # rate, sum/avg by label, percentile, histogram_quantile, top-k, LTTB downsample
  render-echarts/# spec + data → ECharts option (shared by HTML and SSR SVG)
  render-flame/  # flame graph + diff flame graph → SvgNode tree / HTML
  render-trace/  # trace waterfall + trace list → SvgNode tree / HTML
  html/          # self-contained HTML template + inliner (snapshot and live bootstraps)
  server/        # live mode: loopback HTTP server, session token, panel data API
  raster/        # SVG → PNG via @resvg/resvg-wasm
  themes/        # light, dark (JSON tokens)
  cli/           # commands, arg parsing, output
schema/v1.json          # spec JSON Schema (source of truth; TS types generated from it)
schema/sources-v1.json  # sources config JSON Schema
examples/        # sample data + specs + rendered outputs (gallery)
tests/           # unit, golden (byte-identical output), e2e CLI, integration (containerised backends)
```

Tooling: pnpm workspaces, TypeScript, tsup, vitest, eslint, prettier, changesets. Node >= 22.12.

## 5. Data sources

Adapters and connectors are written **per protocol, not per vendor**. One adapter parses both a file export and
the live API response of the same format; connectors only handle HTTP, auth, pagination and limits.

### 5.1 Normalized data model (packages/adapters)

| Kind | Shape |
|---|---|
| `metric` | `{ series: [{ labels: Record<string,string>, points: [tsMs, value][] }], meta: { type: counter\|gauge\|histogram\|summary\|unknown, unit?, step? } }` |
| `log` | `{ entries: [{ ts, level?, labels, message }] }` |
| `trace` | `{ spans: [{ traceId, spanId, parentId?, name, service, startNs, endNs, status, attrs }] }` (one trace or a search result) |
| `profile` | `{ samples: [{ stack: string[], value: number }], unit: cpu-ns\|bytes\|count }` |

Metric type inference: name suffix (`_total` → counter, `_bucket` with `le` → histogram),
monotonic check, explicit override in spec. Adapters must stream / cap memory; reject files > 500 MB with a clear error.

### 5.2 File formats (v1)

| Format | Kind | Detection |
|---|---|---|
| Prometheus HTTP API `query_range` JSON (`resultType: matrix`); also VictoriaMetrics, Thanos, Mimir | metric | `data.resultType` |
| VictoriaMetrics `/api/v1/export` JSON lines | metric | `metric` + `values` + `timestamps` keys |
| CSV (`timestamp,<labels...>,value` or wide format) | metric | header sniff |
| OTLP JSON metrics (`resourceMetrics`) | metric | top-level key |
| Loki `query_range` JSON (`resultType: streams`; `matrix` → metric) | log | `data.resultType` |
| Elasticsearch / OpenSearch `_search` response | log | `hits.hits` |
| NDJSON logs, incl. VictoriaLogs output (`_time`, `_msg`, `_stream`) | log | first line parse |
| OTLP JSON logs (`resourceLogs`) | log | top-level key |
| OTLP JSON traces (`resourceSpans`), incl. Tempo trace-by-id | trace | top-level key |
| Jaeger JSON (export or query API), incl. VictoriaTraces | trace | `data[].spans` |
| Zipkin v2 JSON | trace | array of spans with `traceId` + `localEndpoint` |
| pprof (`.pb.gz`), incl. Go, Pyroscope and Parca exports | profile | gzip magic + protobuf decode |
| Collapsed stacks (`a;b;c 123`): async-profiler, perf, py-spy | profile | line regex |
| speedscope JSON | profile | `$schema` speedscope |
| Chrome / V8 `.cpuprofile` | profile | `nodes` + `samples` + `timeDeltas` |
| miraviz snapshot (`.miraviz/snapshots/*.json`) | any | `miravizSnapshot` key |

### 5.3 Live connectors (packages/connectors)

| Connector `type` | Backends | Kind | Query object |
|---|---|---|---|
| `prometheus` | Prometheus, VictoriaMetrics (`flavor`), Thanos, Mimir, Cortex | metric | `{ expr }` (PromQL; MetricsQL on VM), step auto |
| `loki` | Loki | log (metric for LogQL metric queries) | `{ expr, limit? }` |
| `victorialogs` | VictoriaLogs | log (metric for `stats` queries) | `{ expr, limit? }` (LogsQL) |
| `elasticsearch` | Elasticsearch, OpenSearch (`flavor`) | log | `{ index, query, timeField, fields: { message, level, labels[] } }` (query DSL) |
| `jaeger` | Jaeger, VictoriaTraces | trace | `{ traceId }` or `{ service, operation?, tags?, minDuration?, limit }` |
| `tempo` | Grafana Tempo | trace | `{ traceId }` or `{ traceql, limit }` |
| `zipkin` | Zipkin | trace | `{ traceId }` or `{ serviceName, spanName?, limit }` |
| `pyroscope` | Grafana Pyroscope | profile | `{ query }` (profile type + label selector) |
| `pprof-http` | Go `net/http/pprof` endpoints | profile | `{ profile: cpu\|heap\|allocs\|goroutine\|mutex\|block, seconds? }` (captured at fetch time; no time range) |

Time range is injected by the connector (`start/end/step` params, ES `range` filter on `timeField`…); queries never
contain absolute times. Step defaults to `range / 1000` rounded to a nice value; the spec may pin it.

Default fetch limits (overridable per source): 500 series, 11 000 points/series, 100 000 log lines,
10 000 spans/trace, 100 traces per search, 30 s timeout. Hitting a limit truncates and flags it; never silent.

### 5.4 Sources config (`miraviz.sources.json`)

Committable; holds **no secrets**, only env-var names. Values come from the environment or a gitignored `.env.local`.

```json
{
  "sources": {
    "prom-prod":  { "type": "prometheus", "url": "https://prom.example.com",
                    "auth": { "type": "bearer", "tokenEnv": "PROM_PROD_TOKEN" } },
    "vm-staging": { "type": "prometheus", "flavor": "victoriametrics", "url": "http://vm:8428" },
    "mimir":      { "type": "prometheus", "url": "https://mimir.example.com/prometheus",
                    "headers": { "X-Scope-OrgID": "team-a" } },
    "logs":       { "type": "elasticsearch", "flavor": "opensearch", "url": "https://os:9200",
                    "auth": { "type": "basic", "userEnv": "OS_USER", "passwordEnv": "OS_PASSWORD" } }
  }
}
```

Auth types v1: `none`, `basic`, `bearer`, `apikey` (ES), static `headers`. See §14 for mTLS / cloud IAM.

## 6. `miraviz profile` output (what Claude sees)

Hard cap: **4 KB JSON**. Must be enough to choose a representation, nothing more. Works on files and snapshots.

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
  "truncated": false,
  "hints": ["histogram: use percentile or heatmap", "pod is high-cardinality: group or top-k"]
}
```

Equivalent summaries for log (count, levels, label cardinality, time range, top message templates ≤ 10),
trace (trace count, span count, services, depth, duration range), profile (sample count, unit, depth, top 10 frames by self).
For snapshots, `source` carries provenance (source name, connector type, query, time range).

## 7. Spec (schema/v1.json)

```json
{
  "$schema": "https://<site>/schema/v1.json",
  "version": "1",
  "title": "Service health",
  "ask": "p95 latency and 5xx rate per service over the last 6h",
  "time": { "from": "now-6h", "to": "now" },
  "data": {
    "latency": { "source": "prom-prod",
                 "query": { "expr": "sum by (service, le) (rate(http_request_duration_seconds_bucket[5m]))" } },
    "errors":  { "source": "prom-prod",
                 "query": { "expr": "sum by (service) (rate(http_requests_total{code=~\"5..\"}[5m]))" } },
    "archive": { "file": "latency.json" }
  },
  "layout": { "columns": 2 },
  "theme": "light",
  "panels": [
    {
      "id": "p95-by-service",
      "title": "p95 latency",
      "intent": "trend",
      "data": "latency",
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
- A data entry is either `{ file }` or `{ source, query }`. `source` names an entry in `miraviz.sources.json`.
- `time` is required when any data entry uses a source; relative (`now-6h`) or absolute ISO-8601.
- `intent` is required: `trend | distribution | ranking | comparison | composition | correlation | single-value | log-volume | log-table | trace-waterfall | trace-list | flame | flame-diff`.
- `rationale` is required (≤ 200 chars) and is shown in the HTML as the panel subtitle/tooltip.
- `transform` ops are a closed set (see §8). No expressions, no code. Queries are passed to the backend as-is.
- Credentials never appear in a spec. Ids are kebab-case and unique. Text is plain text with length limits. Unknown fields are an error.

## 8. Transforms (packages/transform), deterministic

`rate`, `increase`, `sum|avg|min|max by [labels]`, `histogram_quantile(q, by)`, `percentile(q)` (raw samples),
`topk(k, by: max|avg|last)`, `bottomk`, `filter(label op value)`, `window(start,end)`, `resample(step)`,
`count_over_time(step)` (logs), `group_by_level` (logs), `bucketize` (heatmap), `self_time` / `total_time` (profiles),
`diff(baseline)` (profiles, comparison). Downsampling (LTTB, default ≤ 2000 points per series) is always applied at render, not in the spec.

For live sources, push aggregation into the query (`rate`, `sum by`) to cut data volume; spec transforms still run
after fetch. Transforms always run in Node (CLI or live server), never in the browser.

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
| trace-list | scatter (duration over time), table |
| flame / flame-diff | flamegraph |

Validator errors (with code, JSON pointer and suggestion):
- `E_COUNTER_NO_RATE`: counter plotted raw → suggest `rate` (in the query or as a transform).
- `E_HIST_RAW_BUCKETS`: histogram buckets plotted directly → suggest `histogram_quantile` or heatmap.
- `E_TOO_MANY_SERIES`: > 20 series on line/area after transforms → suggest `topk` or `sum by`.
- `E_INTENT_CHART_MISMATCH`: chart not allowed for intent.
- `E_SIGNAL_CHART_MISMATCH`: e.g. flamegraph on metric data.
- `E_UNKNOWN_LABEL`: label not in profile → did-you-mean.
- `E_UNIT_MIX`: series with different units on one axis → suggest dual axis or split panel.
- `E_SOURCE_UNKNOWN`: `source` not in sources config → did-you-mean.
- `E_QUERY_SHAPE`: query object missing/extra fields for the connector type.
- `E_NO_SNAPSHOT`: source-backed data entry not fetched yet → suggest `miraviz fetch`.

Warnings: `W_LOG_SCALE` (value span > 3 orders), `W_GAPS` (> 5 % gaps), `W_STACKED_GAUGE` (stacking non-additive gauges),
`W_TRUNCATED` (fetch hit a limit), `W_STALE_SNAPSHOT` (snapshot query differs from spec query).

Fetch errors (from `miraviz fetch` / live server): `F_AUTH`, `F_TIMEOUT`, `F_QUERY` (backend parse/exec error,
backend message passed through, truncated), `F_UNREACHABLE`, `F_LIMIT`.

## 10. Rendering

### 10.1 Interactive HTML (snapshot, default output)
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
Same spec + same data/snapshots + same version = byte-identical output. No `Date.now()`, no random, stable key and attribute order,
fixed number precision. Enforced by golden tests. `render` never touches the network. Live mode is inherently
non-deterministic; a snapshot saved from it renders deterministically.

### 10.4 Live mode (`miraviz serve --live <spec>`)
- Local Node server on `127.0.0.1`, random port, per-session token in the opened URL.
- Serves the same HTML template with a live bootstrap; CSP adds `connect-src 'self'` and nothing else.
- Page → `POST /api/panels/:id/data { from, to, source? }`. The server looks up the query **from the spec**,
  fetches via the connector, runs adapters + transforms + downsampling, returns the panel payload.
  It never accepts query text, URLs or credentials from the page.
- Page controls: time-range picker (presets + absolute), "query this range" after a zoom brush, refresh,
  auto-refresh (off by default, ≥ 10 s), per-data-entry source dropdown limited to configured sources of the same type,
  per-panel loading/error/truncated states.
- "Save snapshot" writes snapshot data files + a static HTML (§10.1) under the working directory.
- `pprof-http` panels show a "capture again" button instead of a time picker.

## 11. CLI (`miraviz`)

```
miraviz profile <data-file|snapshot> [--json]
miraviz sources list | test <name> [--json]            # reads miraviz.sources.json; never prints secrets
miraviz fetch <spec> [--data <id>] [--from <t> --to <t>] [--source <dataId>=<name>]   # writes .miraviz/snapshots/
miraviz validate <spec> [--json]
miraviz render <spec> [--html] [--svg] [--png] [--theme light|dark] [--out <dir>]     # no network
miraviz init <spec> [--from <data-file|source>]       # starter spec from profile
miraviz examples list | show <name> | copy <name>
miraviz serve <file.html>                              # open a static page in the default browser
miraviz serve --live <spec> [--port <n>] [--no-open]   # live mode, 127.0.0.1 only
```

Exit codes: 0 ok, 1 validation error, 2 input error, 3 internal, 4 backend error (fetch/serve).
`--json` on every command for machine-readable output. All stdout capped (profile ≤ 4 KB, validate ≤ 50 diagnostics).
Network access happens only in `fetch`, `sources test` and `serve --live`.

## 12. Skills

### `visualize/SKILL.md`
- Frontmatter `allowed-tools`: only `Bash(miraviz profile *)`, `Bash(miraviz validate *)`, `Bash(miraviz render *)`,
  `Bash(miraviz init *)`, `Bash(miraviz examples *)`, `Bash(miraviz sources list *)`, `Bash(miraviz fetch *)`,
  `Bash(miraviz serve *)`, plus `node ${CLAUDE_PLUGIN_ROOT}/bin/miraviz.mjs ...` equivalents, `Read`, `Write(*.miraviz.json)`.
- Hard rules: never Read raw data or snapshot files; never write HTML/SVG/JS; never put credentials in a spec;
  always (fetch →) profile → spec → validate → fix → render; every panel needs intent + rationale; prefer 1–4 panels;
  push aggregation into live queries; ask the user only if the ask is ambiguous about the metric, the time range,
  or which of several matching sources to use.
- Workflow and spec-in-one-screen, as in platformcosmo's diagram skill.
- References: representation rules, spec reference, per-signal guides (metrics, logs, traces, profiles) with worked examples,
  query guides (PromQL/MetricsQL, LogQL, LogsQL, ES query DSL, TraceQL, Jaeger/Zipkin search, Pyroscope).

### `export/SKILL.md`
Re-render an existing spec to other formats or themes.

## 13. Milestones (each = PR(s) with green CI; acceptance criteria in bold)

| M | Scope | Acceptance |
|---|---|---|
| M0 | Monorepo scaffold, CI, schema v1 skeleton, type generation, plugin manifests, launcher | **`claude plugin install` works; `miraviz --version` runs from plugin bin** |
| M1 | Metrics from files: Prometheus matrix, VictoriaMetrics export, CSV adapters, profile, transforms (rate, sum by, histogram_quantile, topk), line/area/bar/stat, HTML output | **E2E: prompt → spec → valid → HTML opens offline with synced zoom; golden tests pass** |
| M2 | Validator semantics + did-you-mean + all §9 rules; SVG via ECharts SSR; PNG | **Each error code has a failing-fixture test; SVG byte-identical across runs** |
| M3 | Live foundation: sources config + schema, connector framework, `prometheus` connector (Prometheus, VictoriaMetrics, Thanos, Mimir), snapshot format, `sources`, `fetch`, `serve --live` with time picker, refresh, source dropdown, save snapshot | **E2E against containerised Prometheus and VictoriaMetrics: fetch → render; live page re-queries on range change; server rejects requests without the session token, with a foreign Host header, or carrying query text** |
| M4 | Profiles: pprof, collapsed, speedscope, `.cpuprofile` adapters; `pyroscope` + `pprof-http` connectors; flame graph + diff (HTML interactive + SVG) | **pprof from Go and Pyroscope renders correctly; live Pyroscope range change re-renders the flame graph** |
| M5 | Traces: OTLP, Jaeger, Zipkin adapters; `jaeger` (incl. VictoriaTraces), `tempo`, `zipkin` connectors; waterfall (critical path highlight is visual only) + trace list | **1k-span trace renders < 1 s, HTML < 2 MB; same trace via Jaeger and VictoriaTraces renders identically** |
| M6 | Logs: Loki, ES/OpenSearch, NDJSON (incl. VictoriaLogs), OTLP logs adapters; `loki`, `elasticsearch`, `victorialogs` connectors; volume-by-level chart, virtualized table | **100k log lines → HTML < 3 MB, table scrolls smoothly; live page caps lines and shows truncation** |
| M7 | Multi-panel layouts, heatmap/histogram/boxplot, dual axis, dark theme | **Gallery of ≥ 10 examples (file and snapshot-from-live) generated by CI** |
| M8 | Docs site, npm publish with provenance, marketplace release | **Fresh machine install → first chart from a file in < 2 min, first live chart in < 5 min incl. source setup** |
| Later | ClickHouse (SigNoz / Uptrace / OTel exporter schemas), mTLS / cloud IAM auth, commercial SaaS backends | — |

## 14. Open decisions (confirm before M1 unless noted)

1. ~~Files only, or live fetch?~~ **Resolved:** live via connectors behind `fetch` and `serve --live`; the browser never calls backends.
2. Own spec (assumed), or Perses dashboard spec compatibility / export?
3. Default output: HTML only, or HTML + SVG?
4. ECharts licence check of any bundled extensions; fonts (OFL) for PNG.
5. (Before M3) Auth beyond v1 set: mTLS, AWS SigV4 (Amazon Managed Prometheus / OpenSearch Service), GCP/Azure identity?
6. (Before M3) Live server loopback-only (assumed), or `--host` for team sharing? Sharing needs real auth, currently a non-goal.
7. (Before M5) Confirm VictoriaTraces' Jaeger query API covers both search and trace-by-id; otherwise add a dedicated connector.
8. (Before M6) Supported ES / OpenSearch versions (assumed ES 7.17+ / 8.x, OpenSearch 2.x).

## 15. Testing

- Unit: adapters (fixtures per format, including malformed), transforms (numeric against hand-computed values), validator (one fixture per code),
  connectors (recorded HTTP fixtures: auth, pagination, limits, error mapping to `F_*`).
- Golden: spec + data/snapshots → byte-identical HTML/SVG; update via explicit `pnpm golden:update`.
- E2E: CLI commands, exit codes, output caps.
- Integration: docker compose with Prometheus, VictoriaMetrics, Loki, VictoriaLogs, Elasticsearch, OpenSearch, Jaeger,
  VictoriaTraces, Tempo, Zipkin, Pyroscope, seeded with synthetic data. Tests never point at real or shared endpoints.
- Live server: no token → 401, foreign Host → 403, query/URL in request body → 400, unknown source → 400, secrets absent from every response.
- Browser: Playwright loads each gallery HTML, asserts no console errors, no network requests, zoom sync works;
  live page re-fetches on range change and makes no request outside `'self'`.
- Skill eval: 20 prompt + data pairs (files and live sources) with expected intent/chart; measure the share where the spec validates on the first or second try.

## 16. Security & safety

- Data and text from files and backends are untrusted: escape all labels/messages in SVG and HTML; JSON embedded via `<script type="application/json">` with `<` escaped.
- Snapshot HTML has no network access and no external references (CSP enforced). Live HTML may only call its own server (`connect-src 'self'`).
- Live server: binds 127.0.0.1 only, random port, per-session token required on every API call, Host header checked
  (DNS rebinding), only executes queries defined in the spec against configured sources (not an open proxy),
  timeouts, response size caps, per-source concurrency limit.
- Credentials: only env-var references in `miraviz.sources.json`; values from env / gitignored `.env.local`;
  never logged, never sent to the page, never written to specs or snapshots.
- Snapshots may contain sensitive data: written under `.miraviz/`, and `fetch` warns if that directory is not gitignored.
- Path handling: CLI reads/writes only under the working directory unless `--out` is given.
- Size limits on inputs, fetches, series counts and output files, with clear errors.

## 17. Conventions

- Decision log in `DECISIONS.md` (decision, options, reason).
- Changesets for versioning; plugin version synced with the CLI package.
- `THIRD_PARTY_NOTICES.md` for ECharts, resvg and fonts.
