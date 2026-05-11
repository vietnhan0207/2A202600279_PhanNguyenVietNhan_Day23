# Day 23 Lab Reflection

**Student:** Phan Nguyen Viet Nhan
**Submission date:** 2026-05-12
**Lab repo URL:** https://github.com/MikeyNhan/2A202600279_PhanNguyenVietNhan_Day23

---

## 1. Hardware + setup output

```
Docker:        OK  (29.4.0)
Compose v2:    OK  (5.1.1)
RAM available: 7.6 GB (OK)
Ports free:    OK
Report written: /mnt/c/GET_A_JOB/VIN_AI/Lab/Day23/2A202600279_PhanNguyenVietNhan_Day23/00-setup/setup-report.json
```

All 7 services started and passed `make smoke`:
- app (FastAPI): OK
- prometheus: OK
- alertmanager: OK
- grafana: OK
- loki: OK
- jaeger: OK
- otel-collector: OK

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

See `submission/screenshots/dashboard-overview.png` — AI Service Overview dashboard showing:
Request Rate (RPS) by status, Latency P50/P95/P99, Error Rate (last 5m), GPU Utilization, Token Throughput (in/out per sec), In-Flight Requests.

### Burn-rate panel

See `submission/screenshots/slo-burn-rate.png` — Error Budget Remaining shows 100% (no significant errors during load test). SLO target: 99.5% success rate over 30 days.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| T0 | killed `day23-app` via `make alert` | `docker stop day23-app` |
| T0+75s | `ServiceDown` fired in Alertmanager | `submission/screenshots/alertmanager-firing.png` |
| T0+90s | Slack #all-day23ai20k received 🚨 CRITICAL: ServiceDown | `submission/screenshots/slack-firing.png` |
| T1 | app restarted via `docker start day23-app` | script step 3 |
| T1+60s | alert resolved | `submission/screenshots/slack-resolved.png` |

### One thing surprised me about Prometheus / Grafana

The most surprising aspect was that Grafana provisioned datasources do not automatically use a predictable UID — Grafana auto-assigns a random UID like `PBFA97CFB590B2093` instead of a stable string. Dashboard JSON files that hardcode `"uid": "prometheus"` silently show "No data" unless the datasource provisioning YAML explicitly sets `uid: prometheus`. This is a subtle gotcha that costs significant debugging time in production setups where dashboards are managed as code.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

See `submission/screenshots/jaeger-trace.png` — trace `7535941` showing:
- `inference-api: predict` (root, 155.22ms total)
  - `inference-api: embed-text` (5.15ms)
  - `inference-api: vector-search` (10.14ms)
  - `inference-api: generate-tokens` (138.53ms)

Total Spans: 4, Depth: 2, Services: 1

See `submission/screenshots/jaeger-span-attrs.png` — `generate-tokens` span attributes showing GenAI semantic conventions: `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.response.finish_reason`.

### Log line correlated to trace

```json
{"model": "llama3-mock", "input_tokens": 4, "output_tokens": 54, "quality": 0.82, "duration_seconds": 0.1555, "trace_id": "57eabfebf02f7cb6c6d9d1946609ed2f", "event": "prediction served", "level": "info", "timestamp": "2026-05-11T17:11:32.129474Z"}
```

The `trace_id` field (`57eabfebf02f7cb6c6d9d1946609ed2f`) links this structured log line directly to the corresponding trace in Jaeger, enabling log-to-trace correlation.

### Tail-sampling math

The OTel Collector tail-sampling policy has three rules (evaluated in order, first match wins):
1. **keep-errors**: retain all traces with ERROR status code (100% of errors)
2. **keep-slow**: retain all traces with latency > 2000ms (100% of slow traces)
3. **probabilistic-1pct**: retain 1% of all remaining healthy traces

During the load test at ~10 req/s:
- Error rate ≈ 0% (healthy traffic only) → keep-errors keeps ~0 traces/sec
- Slow traces (>2s) ≈ 0% during normal operation → keep-slow keeps ~0 traces/sec
- Healthy traces: 10/s × 1% = **0.1 traces/sec retained**

**Fraction kept ≈ 1% of healthy + 100% of errors**

This means if we generate 1000 healthy traces, ~10 reach Jaeger. But if we generate 1 error trace, it is guaranteed to reach Jaeger. This is the correct trade-off for production: full fidelity on failures, statistical sampling on the happy path.

---

## 4. Track 04 — Drift Detection

### PSI scores

```json
{
  "prompt_length": {
    "psi": 3.461,
    "kl": 1.798,
    "ks_stat": 0.702,
    "ks_pvalue": 0.0,
    "drift": "yes"
  },
  "embedding_norm": {
    "psi": 0.019,
    "kl": 0.032,
    "ks_stat": 0.052,
    "ks_pvalue": 0.136,
    "drift": "no"
  },
  "response_length": {
    "psi": 0.016,
    "kl": 0.018,
    "ks_stat": 0.056,
    "ks_pvalue": 0.087,
    "drift": "no"
  },
  "response_quality": {
    "psi": 8.849,
    "kl": 13.501,
    "ks_stat": 0.941,
    "ks_pvalue": 0.0,
    "drift": "yes"
  }
}
```

### Which test fits which feature?

| Feature | Best Test | Reason |
|---|---|---|
| `prompt_length` | **PSI** | Continuous, roughly normal. PSI captures histogram-level shifts well and gives a single stable number for monitoring dashboards. Ideal for operational alerting thresholds. |
| `embedding_norm` | **KS (Kolmogorov-Smirnov)** | Tightly distributed normal (σ=0.1). KS is sensitive to small shape changes in symmetric distributions and doesn't require binning, making it robust here. |
| `response_length` | **KS or PSI** | Also roughly normal. Either works; PSI is preferred in production for its interpretability, KS for its bin-free nature. |
| `response_quality` | **KL Divergence** | Bounded [0,1], heavily skewed (Beta distribution). KL divergence captures shape changes in asymmetric distributions better than PSI's histogram approach, and is especially sensitive to tail probability mass shifts — critical for quality regression detection. |

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

The hardest prior-day metric to expose would be **Day 17 — Airflow DAG duration**. Unlike Qdrant (Day 19) which natively exposes a Prometheus `/metrics` endpoint, or llama.cpp (Day 20) which has a built-in metrics path, Airflow requires a `statsd_exporter` sidecar to translate StatsD metrics into Prometheus format. This adds an extra infrastructure component, a separate scrape target, and requires knowing the exact metric naming scheme Airflow uses internally (`airflow_dag_run_duration_seconds`). Without prior day labs running, the cross-day dashboard shows descriptive "No Data (Day 17 not running)" messages rather than breaking — a deliberate fail-soft design that makes the dashboard safe to commit without all services running.

---

## 6. The single change that mattered most

The single most impactful change during this lab was fixing the OpenTelemetry span context propagation in `main.py`. The original code used `tracer.start_span("predict")` which creates a span object but does **not** set it as the current context span. This caused every child span (`embed-text`, `vector-search`, `generate-tokens`) to each become an independent root trace with a different trace ID — resulting in 4 separate single-span traces in Jaeger instead of one unified 4-span trace showing the full request waterfall.

The fix was a single conceptual change: replacing `tracer.start_span()` with `tracer.start_as_current_span()` used as a Python context manager. This sets the span as the active context, so all child `with tracer.start_as_current_span(...)` calls correctly nest underneath it. The difference in observability value is enormous: a fragmented view of isolated spans tells you nothing about causality or latency breakdown, while a proper nested trace immediately shows that `generate-tokens` consumes 90% of the 155ms request latency — actionable information that maps directly to the deck's discussion of distributed tracing as a tool for understanding *where time goes* across a request's lifecycle, not just *that* a request was slow.

This connects to a deeper principle from the Three Pillars of Observability (§2 of the deck): metrics tell you *that* something is wrong, logs tell you *what* happened, but traces tell you *where* and *why*. A broken trace pipeline silently degrades the third pillar while metrics and logs continue to look healthy — exactly the kind of subtle observability gap that causes 2am incidents where you can see the P99 latency spike in Grafana but cannot explain which service or function is responsible.
