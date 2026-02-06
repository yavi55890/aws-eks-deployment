# OpenSLO POC: Complete Onboarding Guide

> **Your mission:** By the end of 2026, build production SLO dashboards in Grafana. This document explains everything in this POC codebase — the concepts, the code, and the decisions — so you can hit the ground running.

---

## Table of Contents

1. [The Big Picture: Why Does This Project Exist?](#1-the-big-picture-why-does-this-project-exist)
2. [Core Concepts You Need to Know](#2-core-concepts-you-need-to-know)
   - [SLA, SLO, SLI — The Reliability Trinity](#sla-slo-sli--the-reliability-trinity)
   - [Error Budgets](#error-budgets)
   - [Burn Rate](#burn-rate)
   - [Multi-Window Burn Rate Alerting](#multi-window-burn-rate-alerting)
   - [What is OpenSLO?](#what-is-openslo)
3. [Architecture Overview](#3-architecture-overview)
4. [Project Structure: Every File Explained](#4-project-structure-every-file-explained)
5. [The OpenSLO YAML Files (The Source of Truth)](#5-the-openslo-yaml-files-the-source-of-truth)
6. [The Go Source Code (The Engine)](#6-the-go-source-code-the-engine)
7. [The Prometheus Configuration (The Metrics Pipeline)](#7-the-prometheus-configuration-the-metrics-pipeline)
8. [The Grafana Configuration (The Visualization Layer)](#8-the-grafana-configuration-the-visualization-layer)
9. [The Docker Setup (The Runtime)](#9-the-docker-setup-the-runtime)
10. [Data Flow: End-to-End](#10-data-flow-end-to-end)
11. [How to Run This POC](#11-how-to-run-this-poc)
12. [Key Design Decisions](#12-key-design-decisions)
13. [Known Gaps and Missing Pieces](#13-known-gaps-and-missing-pieces)
14. [Glossary](#14-glossary)

---

## 1. The Big Picture: Why Does This Project Exist?

Most teams build Grafana dashboards by hand. Someone writes PromQL queries, drags panels around in the Grafana UI, and eventually exports a JSON file. This works fine until you have 50 services, each needing SLO dashboards. At that point, manually maintaining dashboards becomes a nightmare.

**This POC proves a different approach:**

```
OpenSLO YAML (human-readable definitions)
    → Go program (dashboard generator)
        → Grafana Dashboard JSON (auto-generated)
```

Instead of hand-crafting dashboards, you **declare** your SLIs and SLOs in standard YAML files (following the [OpenSLO specification](https://openslo.com/)), and a Go program reads those YAML files and automatically generates the Grafana dashboard JSON. Change the YAML, re-run the generator, and your dashboard updates.

The payoff: **consistency, scalability, and a single source of truth** for SLO definitions across the entire organization.

---

## 2. Core Concepts You Need to Know

### SLA, SLO, SLI — The Reliability Trinity

Think of these as a layered cake, from bottom to top:

| Concept | What It Is | Example | Who Cares |
|---------|-----------|---------|-----------|
| **SLI** (Service Level Indicator) | A *measurement* — the metric you watch | "The ratio of successful HTTP requests to total requests" | Engineers |
| **SLO** (Service Level Objective) | A *target* — the goal for that measurement | "99.5% of requests should succeed over 30 days" | Engineering + Product |
| **SLA** (Service Level Agreement) | A *contract* — what happens if you miss the target | "If availability drops below 99.5%, customers get service credits" | Business + Legal |

**The relationship:** SLIs feed into SLOs, and SLOs underpin SLAs. You *measure* SLIs, you *set* SLOs, and you *promise* SLAs.

In this POC, there are **two SLIs** and **two SLOs**:

1. **Availability SLI/SLO**: "99.5% of HTTP requests return a 2xx status code" over a 30-day rolling window.
2. **Latency SLI/SLO**: "95% of HTTP requests complete within 500ms" over a 30-day rolling window.

### Error Budgets

Here's where SLOs get interesting. If your SLO is 99.5% availability, that means you're *allowed* 0.5% errors. That 0.5% is your **error budget**.

Over a 30-day window:
- **99.5% availability** = you can afford 0.5% failures
- In minutes: 30 days × 24 hours × 60 minutes × 0.005 = **216 minutes of downtime per month**

The error budget is not a punishment — it's a **tool**. It lets teams make rational trade-offs:
- Still have budget left? Ship that risky feature.
- Budget burned? Focus on reliability.
- Budget gone? Freeze deployments and fix things.

**In PromQL terms** (from this POC's Prometheus rules):

```promql
# Error budget ratio = 1 - availability
# For a 99.5% SLO, the total error budget is 0.005 (0.5%)
slo:http_requests:error_budget:ratio = 1 - slo:http_requests:availability:ratio
```

### Burn Rate

Error budgets tell you *how much* budget you have. Burn rate tells you *how fast* you're spending it.

**Burn rate = 1** means you're consuming your error budget at exactly the sustainable rate — you'll use 100% of it by the end of the window (30 days). That's fine.

**Burn rate = 2** means you're consuming it twice as fast — you'll exhaust it in 15 days.

**Burn rate = 14.4** means you'll blow through your entire 30-day error budget in **2 days**. That's a five-alarm fire.

The formula used in this POC:

```promql
# Burn rate for 1-hour window (availability, 99.5% target → 0.005 error budget)
burn_rate_1h = (1 - (good_requests_1h / total_requests_1h)) / 0.005
```

Dividing the *actual* error rate by the *allowed* error rate gives you the multiplier — how many times faster than sustainable you're burning.

### Multi-Window Burn Rate Alerting

This is the Google SRE approach (from the *SRE Workbook*, Chapter 5). A single burn rate window has problems:
- **Short windows (1h):** Catch fast burns but generate false alarms from brief spikes.
- **Long windows (3d):** Catch slow burns but react too slowly to fast ones.

The solution: **require BOTH a short AND a long window to exceed the threshold before alerting.**

This POC implements two alert tiers:

| Alert | Short Window | Long Window | Burn Rate | Detection Time | What It Catches |
|-------|-------------|-------------|-----------|----------------|-----------------|
| **Critical** | 1h | 6h | > 14.4x | ~2 minutes | Complete outages — 100% budget gone in ~2 days |
| **Warning** | 6h | 3d | > 6x | ~15 minutes | Slow degradation — budget gone in ~5 days |

From `config/prometheus/rules/slo-rules.yml`:

```yaml
# Critical: both 1h AND 6h burn rates must exceed 14.4
- alert: AvailabilityCriticalBurnRate
  expr: |
    slo:http_requests:burn_rate:1h > 14.4
    and
    slo:http_requests:burn_rate:6h > 14.4
  for: 2m

# Warning: both 6h AND 3d burn rates must exceed 6
- alert: AvailabilityHighBurnRate
  expr: |
    slo:http_requests:burn_rate:6h > 6
    and
    slo:http_requests:burn_rate:3d > 6
  for: 15m
```

The `and` is crucial. A brief 1h spike that doesn't sustain over 6h won't trigger the critical alert. A slow 3d trend that isn't visible in 6h won't trigger the warning. This dramatically reduces false positives.

### What is OpenSLO?

[OpenSLO](https://openslo.com/) is an **open specification** for defining SLOs in a vendor-neutral, YAML-based format. Think of it like OpenAPI (Swagger) but for reliability objectives instead of APIs.

Key properties of OpenSLO:
- **Vendor-neutral:** Your SLO definitions aren't tied to Datadog, Grafana, Nobl9, or any specific tool.
- **Declarative:** You describe *what* you want, not *how* to compute it.
- **Version-controlled:** YAML files live in Git, so SLOs are treated like code.
- **Standardized:** Teams across an organization use the same structure.

An OpenSLO document has an `apiVersion`, a `kind` (Service, SLI, or SLO), `metadata`, and a `spec`. This POC uses `openslo/v1`.

**Why use OpenSLO instead of just writing Grafana dashboards directly?**

Because OpenSLO is the *source of truth*. From a single set of YAML definitions, you could generate:
- Grafana dashboards (what this POC does)
- Prometheus recording rules
- PagerDuty alert configurations
- SLO reports for stakeholders
- Compliance documentation

The YAML is the contract. The tooling is interchangeable.

---

## 3. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        DESIGN TIME (You run this once)              │
│                                                                     │
│  ┌──────────────────┐    ┌─────────────────────┐    ┌────────────┐ │
│  │  OpenSLO YAML    │───▶│  Dashboard Generator │───▶│  Dashboard │ │
│  │  (config/openslo) │    │  (Go program)        │    │  JSON      │ │
│  └──────────────────┘    └─────────────────────┘    └────────────┘ │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                        RUNTIME (Docker Compose)                     │
│                                                                     │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐        │
│  │  Simulator   │────▶│  Prometheus  │────▶│   Grafana    │        │
│  │  (Go app)    │     │  (scrapes    │     │  (visualizes │        │
│  │  :8080       │     │   metrics)   │     │   data)      │        │
│  │              │     │  :9090       │     │  :3000       │        │
│  │  Generates   │     │  Evaluates   │     │  Loads the   │        │
│  │  fake HTTP   │     │  recording   │     │  generated   │        │
│  │  metrics     │     │  rules &     │     │  dashboard   │        │
│  │              │     │  alerts      │     │  JSON        │        │
│  └──────────────┘     └──────────────┘     └──────────────┘        │
│         │                      │                     │              │
│         └──────────monitoring network───────────────┘              │
└─────────────────────────────────────────────────────────────────────┘
```

There are two distinct phases:

1. **Design time:** The dashboard generator reads OpenSLO YAML and produces `slo-dashboard.json`. This happens on your machine (not in Docker).
2. **Runtime:** Docker Compose spins up a simulator (generates fake metrics), Prometheus (scrapes and stores them), and Grafana (displays the pre-generated dashboard).

---

## 4. Project Structure: Every File Explained

```
oslo-poc/
│
├── cmd/                                    # Executable entry points
│   └── dashboard-generator/
│       └── main.go                         # ★ The main event: reads OpenSLO YAML, outputs Grafana JSON
│
├── internal/                               # Internal Go packages (not importable externally)
│   ├── openslo/
│   │   └── types.go                        # Go structs that mirror the OpenSLO YAML structure
│   ├── datasource/
│   │   └── datasource.go                   # MetricSource interface + Prometheus/Chronosphere/OpenTSDB implementations
│   ├── generator/
│   │   └── generator.go                    # Simulated HTTP traffic generator (for the demo simulator)
│   └── metrics/
│       └── metrics.go                      # Prometheus metric definitions (counters, histograms)
│
├── config/                                 # All configuration files
│   ├── openslo/                            # ★ OpenSLO definitions (THE source of truth)
│   │   ├── service.yaml                    # Service definition
│   │   ├── sli-availability.yaml           # Availability SLI (what to measure)
│   │   ├── sli-latency.yaml                # Latency SLI (what to measure)
│   │   ├── slo-availability.yaml           # Availability SLO (the target)
│   │   └── slo-latency.yaml                # Latency SLO (the target)
│   ├── prometheus/
│   │   ├── prometheus.yml                  # Prometheus server config (scrape targets, etc.)
│   │   └── rules/
│   │       └── slo-rules.yml               # ★ Recording rules + alerting rules for SLO math
│   └── grafana/
│       ├── datasources.yml                 # Tells Grafana where Prometheus lives
│       ├── dashboards.yml                  # Tells Grafana where to find dashboard JSON files
│       └── dashboards/
│           ├── dashboards.yml              # Empty placeholder
│           └── slo-dashboard.json          # ★ The generated dashboard (output of the generator)
│
├── scripts/
│   └── generate-dashboard.sh               # Convenience script to build + run the generator
│
├── docker-compose.yml                      # Spins up simulator + Prometheus + Grafana
├── Dockerfile                              # Builds the Go simulator binary
├── Makefile                                # Build/run/test commands
├── go.mod                                  # Go module definition
├── go.sum                                  # Go dependency checksums
│
├── README.md                               # Project overview
├── ARCHITECTURE.md                         # Architecture details
├── IMPLEMENTATION.md                       # Implementation notes
├── QUICKSTART.md                           # Getting started
├── EXAMPLES.md                             # Usage examples
└── SUMMARY.md                              # Summary
```

Files marked with ★ are the ones you'll interact with most.

---

## 5. The OpenSLO YAML Files (The Source of Truth)

These files in `config/openslo/` are the heart of the project. Everything downstream is derived from them.

### 5.1 Service Definition (`service.yaml`)

```yaml
apiVersion: openslo/v1
kind: Service
metadata:
  name: demo-service
  displayName: Demo Metrics Service
spec:
  description: Simulated HTTP service for OpenSLO testing
```

This is the simplest file. It just declares that a service named `demo-service` exists. SLOs reference this service by name. In production, you'd have one of these per microservice.

### 5.2 Availability SLI (`sli-availability.yaml`)

```yaml
apiVersion: openslo/v1
kind: SLI
metadata:
  name: availability-sli
  displayName: Availability SLI
  labels:
    team: platform
    component: api
    criticality: high
  annotations:
    owner: platform-team@company.com
    runbook: https://wiki.company.com/runbooks/availability
    cost-center: "1234"
spec:
  description: Availability based on successful HTTP responses
  ratioMetric:
    counter: true
    good:
      metricSource:
        type: Prometheus
        spec:
          query: sum(rate(http_requests_total{status_code=~"2.."}[5m]))
    total:
      metricSource:
        type: Prometheus
        spec:
          query: sum(rate(http_requests_total[5m]))
```

Let's break this down piece by piece:

**Metadata section:**
- `name: availability-sli` — this is the ID that SLOs use to reference this SLI.
- `labels` — key-value pairs for categorization. The dashboard generator uses these to create dashboard tags (e.g., `team:platform`). Labels are searchable/filterable.
- `annotations` — key-value pairs for non-filterable context. Runbook URLs, owners, cost centers. These show up in panel descriptions.

**Spec section:**
- `ratioMetric` — This SLI uses a ratio: good events / total events. This is the most common SLI type.
  - `good.metricSource.spec.query` — PromQL that counts successful requests (HTTP 2xx):
    - `http_requests_total{status_code=~"2.."}` — Counter of requests where the status code matches regex `2..` (i.e., 200, 201, 204, etc.)
    - `rate(...[5m])` — Converts a counter into a per-second rate over a 5-minute window.
    - `sum(...)` — Aggregates across all dimensions (methods, endpoints).
  - `total.metricSource.spec.query` — Same but for *all* requests, regardless of status code.

- There's also a `thresholdMetric` defined (the single-query version: `good/total`), but the generator primarily uses the `ratioMetric` because it's more flexible.

**Why `rate()` and not just the raw counter?**

Prometheus counters only go up. If you look at `http_requests_total`, it might be 1,000,000. That number alone is useless. `rate()` computes the per-second increase, which gives you the *throughput*: "We're handling 150 requests per second." The `[5m]` window smooths out brief fluctuations.

### 5.3 Latency SLI (`sli-latency.yaml`)

```yaml
spec:
  description: Percentage of requests completed within 500ms
  ratioMetric:
    counter: true
    good:
      metricSource:
        type: Prometheus
        spec:
          query: |
            sum(rate(http_request_duration_seconds_bucket{le="0.5"}[5m]))
    total:
      metricSource:
        type: Prometheus
        spec:
          query: |
            sum(rate(http_request_duration_seconds_count[5m]))
```

This one measures latency using **Prometheus histograms**. Here's how histograms work:

When you record a request that took 0.3 seconds, Prometheus increments *every bucket* whose upper bound (`le` = "less than or equal") is ≥ 0.3s. The buckets in this POC are: `{le="0.005"}, {le="0.01"}, {le="0.025"}, ..., {le="0.5"}, {le="1"}, ...`

So `http_request_duration_seconds_bucket{le="0.5"}` contains the count of all requests that completed in 500ms or less — which is exactly what we need for this SLI.

The total is `http_request_duration_seconds_count`, which is the total number of observed requests.

**Result:** `good / total` = the fraction of requests completing within 500ms.

### 5.4 Availability SLO (`slo-availability.yaml`)

```yaml
apiVersion: openslo/v1
kind: SLO
metadata:
  name: availability-slo
  displayName: Availability SLO - 99.5%
  labels:
    team: platform
    tier: critical
    service: demo-api
    environment: production
  annotations:
    alert-channel: "#alerts-critical"
    escalation-policy: "oncall-platform"
    business-impact: "Customer-facing API unavailable"
    sla-commitment: "99.5% monthly uptime"
spec:
  description: 99.5% of requests should be successful
  service: demo-service                    # Links to service.yaml
  indicator:
    metadata:
      name: availability-sli               # Links to sli-availability.yaml
  budgetingMethod: Occurrences             # Count events, not time slices
  objectives:
    - displayName: 99.5% availability over 30 days
      target: 0.995                        # ← THE TARGET (99.5%)
      op: gte                              # greater than or equal
  timeWindows:
    - duration: 30d                        # 30-day rolling window
      isRolling: true
```

**Key concepts here:**

- `indicator.metadata.name: availability-sli` — This links the SLO to its SLI. The dashboard generator uses this link to find the PromQL queries.
- `target: 0.995` — This is the magic number. The generator uses it to:
  - Set gauge thresholds (green ≥ 99.5%, orange ≥ 94.5%, red below that).
  - Calculate error budget: `1 - 0.995 = 0.005` (0.5%).
  - Compute burn rate denominators.
- `budgetingMethod: Occurrences` — Count individual events (requests), not time slices. Each request either succeeds or fails.
- `timeWindows: 30d rolling` — The window continuously slides forward. Yesterday's data eventually drops off.

**The annotations are organizational gold.** Notice `alert-channel`, `escalation-policy`, and `business-impact`. These don't affect the dashboard generator today, but they're embedded in the SLO definition so that future tooling (alert routers, incident management) can read them.

### 5.5 Latency SLO (`slo-latency.yaml`)

Same structure as the availability SLO, but with:
- `target: 0.95` (95% of requests under 500ms)
- `tier: high` (vs. `critical` for availability)
- Error budget = `1 - 0.95 = 0.05` (5% — ten times more generous than availability)

This makes sense: 95% latency is easier to meet than 99.5% availability. You're allowed more "slow" requests than "failed" ones.

---

## 6. The Go Source Code (The Engine)

### 6.1 OpenSLO Types (`internal/openslo/types.go`)

This file defines Go structs that map directly to the OpenSLO YAML structure. When the YAML is parsed, it fills these structs.

```go
type Metadata struct {
    Name        string            `yaml:"name"`
    DisplayName string            `yaml:"displayName"`
    Labels      map[string]string `yaml:"labels,omitempty"`
    Annotations map[string]string `yaml:"annotations,omitempty"`
}

type SLI struct {
    APIVersion string   `yaml:"apiVersion"`
    Kind       string   `yaml:"kind"`        // "SLI"
    Metadata   Metadata `yaml:"metadata"`
    Spec       struct {
        RatioMetric     *RatioMetric     `yaml:"ratioMetric,omitempty"`
        ThresholdMetric *ThresholdMetric `yaml:"thresholdMetric,omitempty"`
    } `yaml:"spec"`
}

type SLO struct {
    // ... includes Objectives (target floats), TimeWindows, etc.
}
```

The `yaml:"..."` struct tags tell Go's YAML parser how to map YAML fields to struct fields. The `omitempty` means "if this field is missing in the YAML, that's fine — leave it as zero/nil."

Notice `RatioMetric` is a pointer (`*RatioMetric`). This lets the code distinguish between "not specified" (nil) and "specified but empty" (non-nil zero value).

### 6.2 Datasource Abstraction (`internal/datasource/datasource.go`)

This is the **extensibility layer**. The senior engineer designed an interface that makes the dashboard generator agnostic to the metrics backend:

```go
type MetricSource interface {
    GetType() string              // "Prometheus", "Chronosphere", "OpenTSDB"
    ValidateQuery(query string) error
    TransformQuery(query string) string   // Transform PromQL to target query language
    GetDatasourceName() string            // Grafana datasource name
    SupportsHistograms() bool             // Can we do latency percentile queries?
    GetQueryLanguage() string             // "PromQL", "OpenTSDB", etc.
}
```

**Three implementations exist:**

| Source | Status | `TransformQuery` | Histograms |
|--------|--------|-------------------|------------|
| `PrometheusSource` | Fully working | Pass-through (queries are already PromQL) | Yes |
| `ChronosphereSource` | Stub, Prometheus-compatible | Pass-through for now | Yes |
| `OpenTSDBSource` | Stub | Pass-through (would need real transforms) | No |

**Why this matters for your 2026 work:** If the company uses Chronosphere (or any Prometheus-compatible backend), you'll implement `TransformQuery()` to add Chronosphere-specific optimizations. The rest of the dashboard generator doesn't need to change.

The factory function `GetMetricSource()` creates the right implementation:

```go
func GetMetricSource(sourceType, datasourceName string) (MetricSource, error) {
    switch sourceType {
    case "Prometheus":
        return NewPrometheusSource(datasourceName), nil
    case "Chronosphere":
        return NewChronosphereSource(datasourceName), nil
    // ...
    }
}
```

### 6.3 Dashboard Generator (`cmd/dashboard-generator/main.go`)

This is the star of the show — 603 lines that turn OpenSLO YAML into a Grafana dashboard JSON. Let's walk through the flow:

**Step 1: CLI argument parsing (lines 66-82)**

```go
func main() {
    opensloDir := os.Args[1]    // e.g., "config/openslo"
    outputFile := os.Args[2]    // e.g., "config/grafana/dashboards/slo-dashboard.json"
    datasourceType := "Prometheus"  // default, can be overridden
}
```

**Step 2: Read all YAML files (lines 96-140)**

The generator walks the `config/openslo/` directory recursively. For each `.yaml` file, it tries to parse it as an SLI and as an SLO. If `sli.Kind == "SLI"`, it's stored in a map keyed by name. If `slo.Kind == "SLO"`, it's added to a slice.

```go
// SLIs are stored by name so SLOs can reference them
slis := make(map[string]*openslo.SLI)    // "availability-sli" → *SLI struct
slos := make([]*openslo.SLO, 0)          // list of all SLOs
```

**Step 3: Generate the dashboard (lines 170-271)**

`GenerateDashboard()` creates panels in a specific order, positioning them on a 24-column Grafana grid:

1. **SLO Gauge panels** — Two per row (12 columns each), showing current SLI value with red/orange/green thresholds.
2. **Error Budget panel** — Full width (24 columns), time-series chart showing budget consumption over time.
3. **Burn Rate panels** — One per SLO, showing 1h, 6h, and 3d burn rates as overlapping lines.
4. **Request Rate panel** — RED metrics (Rate, Errors, Duration) showing total, successful, and error request rates.
5. **Latency panel** — p50, p95, p99 percentiles (only if the datasource supports histograms).
6. **Status Code panel** — Distribution of 2xx, 4xx, 5xx responses.

**Step 4: The Gauge Panel in detail (lines 273-354)**

This is worth understanding because it shows how OpenSLO data becomes a Grafana panel:

```go
func (g *DashboardGenerator) createGaugePanel(id int, slo *openslo.SLO, x, y int) GrafanaPanel {
    // 1. Find the SLI linked to this SLO
    sli := g.slis[slo.Spec.Indicator.Metadata.Name]

    // 2. Build the PromQL query: (good / total) * 100
    goodQuery := sli.Spec.RatioMetric.Good.MetricSource.Spec.Query
    totalQuery := sli.Spec.RatioMetric.Total.MetricSource.Spec.Query
    query := fmt.Sprintf("(%s) / (%s) * 100", goodQuery, totalQuery)

    // 3. Extract the target from the SLO objectives
    target := slo.Spec.Objectives[0].Target * 100  // 0.995 → 99.5

    // 4. Create thresholds: red → orange at (target-5) → green at target
    // So for 99.5%: red < 94.5% < orange < 99.5% < green
}
```

**Step 5: Burn Rate Panel (lines 430-512)**

The most mathematically interesting panel. It takes the SLI's PromQL queries and generates three time-window variants:

```go
// Original query uses [5m] window. Replace with different windows:
burnRate1h := strings.ReplaceAll(goodQuery, "[5m]", "[1h]")   // 1-hour window
burnRate6h := strings.ReplaceAll(goodQuery, "[5m]", "[6h]")   // 6-hour window
burnRate3d := strings.ReplaceAll(goodQuery, "[5m]", "[3d]")   // 3-day window

// Then wrap in the burn rate formula:
// (1 - (good / total)) / errorBudget
```

The threshold lines at 6x and 14.4x match the alerting thresholds in `slo-rules.yml`. When the burn rate line crosses into yellow (6x) or red (14.4x), you know an alert is (or should be) firing.

### 6.4 Metrics Simulator (`internal/generator/generator.go` + `internal/metrics/metrics.go`)

These files power the demo data. The simulator generates fake HTTP traffic with controllable characteristics:

**Metric definitions (`metrics.go`):**

```go
// Counter: increments on every request, labeled by method, endpoint, status_code
HTTPRequestsTotal = promauto.NewCounterVec(...)

// Histogram: records request duration in predefined buckets
HTTPRequestDuration = promauto.NewHistogramVec(
    prometheus.HistogramOpts{
        Buckets: []float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10},
    }, ...
)
```

**Traffic generation (`generator.go`):**

The generator ticks at a configurable rate (default 15 req/sec) and for each tick:

1. Picks a random endpoint (`/api/users`, `/api/orders`, etc.) and HTTP method.
2. Rolls dice against the error rate (default 2%) to decide the status code:
   - Below `errorRate/2`: → 500 (Internal Server Error)
   - Below `errorRate`: → 503 (Service Unavailable)
   - Small additional chance: 404 or 400
   - Otherwise: 200 (Success)
3. Assigns realistic latency based on status code:
   - 200s: 10-200ms (fast)
   - 4xx: 5-50ms (fastest — server rejects quickly)
   - 5xx: 50-1000ms (slow — server struggling)
4. 1% chance of a 5x latency spike (simulates outliers)

**Spike mode:** Every 10 minutes (configurable), the error rate jumps 5x for 30 seconds. This creates visible error budget burns in the dashboard, making the demo more interesting.

> **Note:** The `cmd/simulator/main.go` file that would wire this together and serve the `/metrics` HTTP endpoint is **missing** from this repo. This is a known gap (see Section 13).

---

## 7. The Prometheus Configuration (The Metrics Pipeline)

### 7.1 Prometheus Server Config (`config/prometheus/prometheus.yml`)

```yaml
global:
  scrape_interval: 15s          # How often Prometheus pulls metrics from targets
  evaluation_interval: 15s      # How often recording rules are evaluated

scrape_configs:
  - job_name: 'simulator'
    static_configs:
      - targets: ['simulator:8080']   # The simulator container
    scrape_interval: 5s               # Overrides global — scrape simulator more often
```

Prometheus operates on a **pull model** — it reaches out to your service's `/metrics` endpoint and scrapes the current values. This is the opposite of tools like StatsD where your app *pushes* metrics.

### 7.2 Recording Rules (`config/prometheus/rules/slo-rules.yml`)

Recording rules pre-compute expensive queries and store the results as new time series. This is critical for SLO dashboards because:

1. **Performance:** `sum(rate(http_requests_total{status_code=~"2.."}[5m]))` is evaluated once per rule interval (30s), not every time someone opens the dashboard.
2. **Consistency:** Everyone queries the same pre-computed metric.
3. **Historical data:** The pre-computed series is stored, so you can query it over long time ranges without recomputing.

**The rules are organized in three groups:**

**Group 1: `slo_availability`** — Pre-computes availability metrics:

```yaml
# Step 1: Count good requests per second
- record: slo:http_requests:good
  expr: sum(rate(http_requests_total{status_code=~"2.."}[5m]))

# Step 2: Count total requests per second
- record: slo:http_requests:total
  expr: sum(rate(http_requests_total[5m]))

# Step 3: Compute availability ratio (0.0 to 1.0)
- record: slo:http_requests:availability:ratio
  expr: slo:http_requests:good / slo:http_requests:total

# Step 4: Compute error rate (error budget consumption)
- record: slo:http_requests:error_budget:ratio
  expr: 1 - slo:http_requests:availability:ratio

# Steps 5-7: Burn rates at 1h, 6h, 3d windows
- record: slo:http_requests:burn_rate:1h
  expr: |
    (1 - (
      sum(rate(http_requests_total{status_code=~"2.."}[1h]))
      /
      sum(rate(http_requests_total[1h]))
    )) / 0.005    # 0.005 = error budget for 99.5% target
```

**Naming convention:** `slo:metric_name:aggregation` — The `slo:` prefix is a Prometheus convention for recording rules. The colons make it instantly clear this is a derived metric, not a raw one.

**Group 2: `slo_latency`** — Same pattern for latency:

```yaml
- record: slo:http_latency:good
  expr: sum(rate(http_request_duration_seconds_bucket{le="0.5"}[5m]))

# Error budget divisor is 0.05 (5% budget for 95% target)
```

**Group 3: `slo_alerts`** — Multi-window burn rate alerts (covered in Section 2).

---

## 8. The Grafana Configuration (The Visualization Layer)

### 8.1 Datasource Provisioning (`config/grafana/datasources.yml`)

```yaml
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy        # Grafana proxies queries (browser → Grafana → Prometheus)
    url: http://prometheus:9090   # Docker network hostname
    isDefault: true
```

This tells Grafana: "There's a Prometheus instance at `http://prometheus:9090`, and it's the default datasource." The `access: proxy` means queries go through Grafana's backend, not directly from the user's browser.

### 8.2 Dashboard Provisioning (`config/grafana/dashboards.yml`)

```yaml
providers:
  - name: 'default'
    type: file
    updateIntervalSeconds: 10     # Check for changes every 10 seconds
    options:
      path: /etc/grafana/provisioning/dashboards
```

This tells Grafana: "Look for JSON files in this directory and auto-import them as dashboards. Check every 10 seconds for changes." This is called **provisioning** — dashboards are loaded from files, not created through the UI.

### 8.3 The Generated Dashboard (`config/grafana/dashboards/slo-dashboard.json`)

This 534-line JSON file is the **output** of the dashboard generator. You should never edit this file manually — it's regenerated every time you run the generator.

The dashboard contains 8 panels:

| # | Panel | Type | Width | What It Shows |
|---|-------|------|-------|---------------|
| 1 | Availability SLO | `gauge` | 12 col | Current availability %, color-coded against 99.5% target |
| 2 | Latency SLO | `gauge` | 12 col | Current latency %, color-coded against 95% target |
| 3 | Error Budget Consumption | `timeseries` | 24 col | Error rate over time for both SLOs |
| 4 | Availability Burn Rate | `timeseries` | 12 col | 1h, 6h, 3d burn rates with 6x/14.4x threshold lines |
| 5 | Latency Burn Rate | `timeseries` | 12 col | Same but for latency |
| 6 | Request Rate (RED) | `timeseries` | 12 col | Total, successful, and error request rates |
| 7 | Request Duration | `timeseries` | 12 col | p50, p95, p99 latency percentiles |
| 8 | HTTP Status Codes | `timeseries` | 12 col | Stacked bar chart of 2xx/4xx/5xx distribution |

**Dashboard metadata:**

```json
{
  "uid": "openslo-generated",
  "title": "OpenSLO Generated Dashboard (Prometheus)",
  "tags": ["openslo", "slo", "generated", "team:platform", "tier:critical", "tier:high"],
  "refresh": "10s",
  "time": { "from": "now-1h", "to": "now" }
}
```

The `uid` is a stable identifier. Even if you regenerate the dashboard, the same UID means Grafana updates the existing dashboard rather than creating a duplicate.

The `tags` are extracted from the SLO labels during generation — `team:platform` comes from `metadata.labels.team: platform` in the SLO YAML.

---

## 9. The Docker Setup (The Runtime)

### 9.1 Docker Compose Services

The `docker-compose.yml` defines three services on a shared `monitoring` bridge network:

**1. `simulator` (port 8080)**
- Built from the project's `Dockerfile`
- Generates fake HTTP metrics at 15 requests/second
- 2% base error rate, with spikes every 10 minutes
- Exposes a `/metrics` endpoint that Prometheus scrapes

**2. `prometheus` (port 9090)**
- Official `prom/prometheus:latest` image
- Scrapes the simulator every 5 seconds
- Evaluates recording rules and alerting rules from `slo-rules.yml`
- Retains data for 30 days (`--storage.tsdb.retention.time=30d`)
- `--web.enable-lifecycle` allows hot-reloading config via HTTP POST

**3. `grafana` (port 3000)**
- Official `grafana/grafana:latest` image
- Default credentials: `admin`/`admin`
- Auto-provisions the Prometheus datasource and the SLO dashboard
- Depends on Prometheus (starts after it)

### 9.2 Dockerfile

```dockerfile
FROM golang:1.21-alpine AS simulator-builder
# ... builds simulator and dashboard-generator binaries ...

FROM alpine:latest
COPY --from=simulator-builder /app/simulator .
CMD ["./simulator"]
```

Multi-stage build: the first stage compiles Go code, the second stage runs the binary in a minimal Alpine image. This keeps the final image small.

---

## 10. Data Flow: End-to-End

Here's the complete journey of a single data point, from fake HTTP request to dashboard pixel:

```
1. Generator.generateRequest() runs
   ├── Picks: endpoint=/api/orders, method=GET, status=200, duration=0.12s
   ├── Calls: metrics.HTTPRequestsTotal.WithLabelValues("GET", "/api/orders", "200").Inc()
   └── Calls: metrics.HTTPRequestDuration.WithLabelValues("GET", "/api/orders", "200").Observe(0.12)

2. Prometheus scrapes http://simulator:8080/metrics every 5 seconds
   ├── Sees: http_requests_total{method="GET",endpoint="/api/orders",status_code="200"} = 12345
   └── Sees: http_request_duration_seconds_bucket{le="0.25",...} = 11000

3. Prometheus evaluates recording rules every 30 seconds
   ├── Computes: slo:http_requests:good = 14.8 req/s
   ├── Computes: slo:http_requests:total = 15.1 req/s
   ├── Computes: slo:http_requests:availability:ratio = 0.9801
   ├── Computes: slo:http_requests:burn_rate:1h = 3.98
   └── Checks: AvailabilityCriticalBurnRate → 3.98 < 14.4 → NOT FIRING

4. Grafana dashboard refreshes every 10 seconds
   ├── Gauge panel queries: (sum(rate(good[5m])) / sum(rate(total[5m]))) * 100
   │   └── Prometheus returns: 98.01 → Gauge shows 98.01% in ORANGE (below 99.5%)
   ├── Error budget panel queries: 1 - (good / total)
   │   └── Prometheus returns: 0.0199 → Line chart shows 1.99% error rate
   └── Burn rate panel queries: burn rates at 1h, 6h, 3d
       └── Prometheus returns: [3.98, 2.1, 1.5] → Three lines on the chart
```

---

## 11. How to Run This POC

### Prerequisites
- Go 1.21+
- Docker and Docker Compose
- `make` (standard on macOS/Linux)

### Quick Start

```bash
# Step 1: Generate the Grafana dashboard from OpenSLO YAML
make generate-dashboard

# Step 2: Build and start all services
make build
make run

# Step 3: Open in browser
# Grafana:    http://localhost:3000 (admin/admin)
# Prometheus: http://localhost:9090
# Simulator:  http://localhost:8080/metrics
```

### Useful Commands

```bash
make logs              # Follow all container logs
make logs-simulator    # Follow simulator logs only
make test-metrics      # Curl the simulator's /metrics endpoint
make status            # Check container status
make stop              # Stop all services
make clean             # Stop and remove all data (fresh start)
make verify            # Validate Prometheus config and rules
```

### Regenerating the Dashboard

After editing any file in `config/openslo/`:

```bash
make generate-dashboard
# Grafana auto-reloads within 10 seconds (no restart needed)
```

---

## 12. Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Language** | Go | Strong YAML/JSON support, compiles to a single binary, matches Prometheus/Grafana ecosystem |
| **OpenSLO version** | v1 | Current stable version of the spec |
| **SLI type** | Ratio metric | Most flexible — numerator/denominator pattern works for availability, latency, throughput |
| **Burn rate windows** | 1h / 6h / 3d | Google SRE recommended set for multi-window alerting |
| **Dashboard generation** | Build-time, not runtime | Simpler, more predictable; dashboard JSON is version-controlled |
| **Datasource abstraction** | Interface pattern | Allows swapping Prometheus for Chronosphere/OpenTSDB without changing dashboard logic |
| **Grafana provisioning** | File-based | Dashboards live in Git, not in Grafana's database; reproducible across environments |

---

## 13. Known Gaps and Missing Pieces

These are things to be aware of as you take this POC toward production:

| Gap | Description | Impact |
|-----|-------------|--------|
| **Missing simulator entry point** | `cmd/simulator/main.go` is referenced in `Dockerfile` and `Makefile` but doesn't exist in the repo | Docker build will fail. The `internal/generator/` and `internal/metrics/` packages exist but need a `main.go` to wire them together with an HTTP server |
| **No tests** | Zero test files in the entire project | No safety net for refactoring |
| **No template variables** | Dashboard has no Grafana template variables (dropdowns) for filtering by service, environment, etc. | In production, you'll want `$service`, `$environment`, `$namespace` variables |
| **Hardcoded metric names** | `http_requests_total` and `http_request_duration_seconds` are baked into both the OpenSLO YAML and the generator | Works for this demo, but production services may use different metric names |
| **No error handling in query transforms** | `TransformQuery()` for Chronosphere and OpenTSDB just pass-through | Real implementations need actual query transformation |
| **Single dashboard** | One flat dashboard with all panels | Production needs: per-service dashboards, overview dashboards, drill-down links |
| **No recording rules generation** | Prometheus recording rules are hand-written, not generated from OpenSLO | Ideally, the generator would also produce `slo-rules.yml` |
| **Static SLO targets** | SLO targets are in YAML, not parameterized | Consider making targets configurable per environment (99.9% prod, 99% staging) |

---

## 14. Glossary

| Term | Definition |
|------|-----------|
| **SLA** | Service Level Agreement — a contractual commitment to a level of reliability |
| **SLO** | Service Level Objective — an internal target for a reliability metric |
| **SLI** | Service Level Indicator — the actual metric being measured |
| **Error Budget** | The amount of unreliability you're allowed (1 - SLO target) |
| **Burn Rate** | How fast you're consuming your error budget relative to the sustainable rate |
| **Recording Rule** | A Prometheus rule that pre-computes a query and stores the result as a new metric |
| **PromQL** | Prometheus Query Language — the language used to query Prometheus metrics |
| **Histogram** | A Prometheus metric type that counts observations in configurable buckets |
| **Counter** | A Prometheus metric type that only increases (or resets to 0 on restart) |
| **Gauge** | A Prometheus metric type that can go up and down (like temperature) |
| **RED Method** | Rate, Errors, Duration — three signals for monitoring request-driven services |
| **Scrape** | The act of Prometheus pulling metrics from a target's `/metrics` endpoint |
| **Provisioning** | Loading Grafana dashboards/datasources from files instead of the UI |
| **OpenSLO** | An open specification for declaring SLOs in vendor-neutral YAML |
| **Multi-window** | Using multiple time windows (1h AND 6h) together to reduce false alert positives |
| **Ratio Metric** | An SLI type defined as good_events / total_events |
| **Threshold Metric** | An SLI type defined as a single metric compared against a threshold |
| **Rolling Window** | A time window that continuously moves forward (vs. calendar-aligned) |
| **Bridge Network** | A Docker network type that allows containers to communicate by name |

---

*Document generated for onboarding. Last updated: February 2026.*
