# 12 — Observability and monitoring

Phase 1 is a **production-shaped** telemetry system, not “a few Grafana panels.” The bar: every hop of `message.send` is in a trace, RED/USE and SLOs are in Prometheus, alerts have runbooks, infra exporters are scraped, dashboards are **provisioned as code**. You still **learn** this stack locally — you do not paste Datadog.

**Invariant:** the app never talks to Tempo, Loki, Prometheus, Grafana, or Alertmanager. It speaks **OpenTelemetry (OTLP)** only. The **OpenTelemetry Collector** is the router (batch, PII filter, spanmetrics, tail sampling). Phoenix later keeps the same span names and the same Collector.

Inspect UIs (Adminer, Redis Insight) are [doc 16](./16-local-consoles.md). They are not monitoring.

## Signals (Grafana LGTM + profiles)

| Signal | Question | Store | UI |
| --- | --- | --- | --- |
| **Traces** | Why was *this* send slow or duplicated? | Grafana Tempo | Grafana Explore / TraceQL |
| **Metrics** | Is the *system* in SLO? Error budget? | Prometheus | Grafana + Alertmanager |
| **Logs** | What did this process emit, with `requestId`? | Loki | Grafana LogQL |
| **Profiles** (Sprint 9) | Where is CPU on the gateway? | Grafana Pyroscope | Grafana |

Correlation is mandatory: `requestId` on the WS envelope = W3C trace id = span = log field = exemplar on the histogram. If you cannot click log → Tempo → span → exemplar, the stack is unfinished.

## Topology

```
[ Next.js ]                    [ Nest gateway × N ]
  Faro later (RUM)               OTel SDK (traces, metrics, logs)
                                 W3C traceparent on HTTP
                                 envelope requestId === trace id
           │                              │
           │         OTLP only            │
           └──────────────┬───────────────┘
                          ▼
              [ OpenTelemetry Collector ]
                otlp receiver
                processors: memory_limiter, batch, redaction
                connectors: spanmetrics
                receivers: prometheus (exporters), hostmetrics
                          │
        ┌─────────┬───────┼────────┬────────────┐
        ▼         ▼       ▼        ▼            ▼
     [Tempo]   [Loki] [Prometheus] [Pyroscope]  (spanmetrics → Prom)
        └─────────┴───────┴────────┴────────────┘
                          ▼
                    [ Grafana ]
                    [ Alertmanager ]
```

Exporters (Prometheus scrape via Collector or Prometheus itself): `postgres_exporter`, `redis_exporter`, `nginx_exporter`, `node_exporter` or Collector `hostmetrics`. Optional: `cadvisor` if you want container CPU/mem in Compose.

Compose (Sprint 9 profile): `otel-collector`, `tempo`, `loki`, `prometheus`, `alertmanager`, `grafana`, `pyroscope`, exporters. Gateways stay 2×; **one** Collector. App OTLP → Collector only.

## Collector (this is the sophistication)

Pipelines as code under `ops/otel/` (when you implement):

| Processor / connector | Why |
| --- | --- |
| `memory_limiter` + `batch` | Protect the Collector; export efficiently |
| `attributes` / `redaction` | Drop `body`, `Authorization`, cookies, refresh tokens. Hash `idempotencyKey` in logs if needed; keep it on spans for dedup debug |
| `resource` | `service.name=gateway`, `service.instance.id` = pod/container (`gateway_id`) |
| **`spanmetrics` connector** | RED from traces (span name `chat.send`) so Phoenix does not invent a second metric API |
| `tail_sampling` (prod) | Keep 100% of errors + latency > SLO + `duplicate=true`; else 1–5%. Dev: 100% |
| Prometheus receiver | Scrape exporters; relabel `instance` → `gateway_id` where it is a gateway |

Do **not** run Grafana Alloy beside the Collector at the start. Do **not** export from Nest straight to Tempo.

Tempo: enable metrics-generator **or** rely on Collector spanmetrics — pick **one** for span RED so you do not double-count. Prefer Collector spanmetrics (portable). Use Tempo for TraceQL + service graph from traces if you enable the generator later and then turn spanmetrics off.

## Trace design (chat)

Required spans (stable names across Nest and Phoenix):

| Span | Parent | Notes |
| --- | --- | --- |
| `chat.send` | root (HTTP or WS) | attributes: `chatId`, `userId`, `deviceId`, `idempotencyKey`, `duplicate`, `seq` |
| `db.messages.insert` | `chat.send` | db system postgres; **no** `body` |
| `db.chats.seq` | `chat.send` | row lock / `current_seq` |
| `redis.publish` | `chat.send` | adapter/pub; not the source of truth |
| `auth.jwt` | request | HTTP and handshake |

HTTP: W3C `traceparent`. Socket.IO: put the same trace id in `requestId`. Auto-instrument **Nest HTTP** (`@nestjs/core` / Express adapter) + `pg` + Redis; **manual** `chat.send` around `ChatService.send`. Do not use `@nestjs/platform-fastify`.

Sampling: 100% local. Prod: tail sampling as above. Always record SLO-violating sends.

## Metrics, RED/USE, recording rules

App (OTel meters) + spanmetrics:

| Metric | Type | Labels (low cardinality) |
| --- | --- | --- |
| `chat_send_total` | counter | `result=ok\|duplicate\|error`, `gateway_id` |
| `chat_send_duration_seconds` | histogram | `gateway_id` — **exemplars** → Tempo |
| `ws_connected` | gauge | `gateway_id` |
| `pg_pool_in_use` / `pg_pool_idle` | gauge | `gateway_id` |
| `rate_limited_total` | counter | `limit=send\|login\|typing`, `gateway_id` |
| `seq_gap_detected_total` | counter | `gateway_id` |
| `auth_fail_total` | counter | `reason=jwt\|revoked_device` |

No `chatId` or `userId` on metric **labels**. Those belong on spans/logs.

Prometheus **recording rules** (commit YAML):

- `job:chat_send:rate5m`, `job:chat_send:error_ratio5m`, `job:chat_send:p99_5m`
- `job:ws_connected:sum`

RED on send. USE on gateways: utilization (CPU, pool), saturation (`ws_connected` vs cap), errors (scrape fail, `result=error`).

## SLOs (Phase 1, 1:1 text)

Track in Grafana (Prometheus). Tune numbers after you have histograms; the **mechanism** is required even if targets move.

| SLO | SLI | Target (starting) |
| --- | --- | --- |
| Send availability | `sum(rate(chat_send_total{result=~"ok\|duplicate"}[5m])) / sum(rate(chat_send_total[5m]))` | 99.9% (30d in prod; 24h window locally) |
| Send latency | histogram p99 `chat_send_duration_seconds` | under 200ms local/staging (measure before you swear by it) |
| Gateway liveness | `/readyz` success (synthetic + scrape) | 99.9% |

**Error budget:** if burn is too fast, page. Multi-window multi-burn-rate alerts (Google SRE): short window (5m/1h) and long (1h/6h) on the availability SLI.

Duplicate acks (`result=duplicate`) are **success** for availability. `result=error` and timeouts are failures.

## Alerting

**Alertmanager** in Compose. Routes: local = Grafana UI + stdout; staging = email/Slack later. Every alert has a **runbook** link (`ops/runbooks/`).

Minimum alert set:

| Alert | Fires when | You do |
| --- | --- | --- |
| `ChatSendErrorBudgetBurn` | SLO burn rate | TraceQL `chat.send` status error; PG/Redis |
| `ChatSendP99High` | p99 above SLO for 10m | Tempo slow traces; pool/Redis |
| `GatewayReadyDown` | `/readyz` fail or scrape missing | Compose/k8s; PG+Redis |
| `PostgresDown` / `RedisDown` | exporter `up==0` | Insight/Adminer only after metrics |
| `PgPoolSaturation` | in_use / max &gt; 80% 5m | load shed, breaker (doc 10) |
| `WsConnectedDrop` | gauge drop over 50% in 2m on one instance | sticky/Nginx, that gateway |
| `RateLimitedSpike` | send limit rate &gt;&gt; baseline | abuse vs misconfig |
| `SeqGapSpike` | `seq_gap_detected_total` | missed adapter; HTTP repair |
| `CollectorOTLPIdle` | no spans while traffic exists | SDK config, Collector |

No alert without a graph on a provisioned dashboard. No dashboard-only “monitoring.”

## Logs (Loki)

JSON: `event`, `level`, `requestId`, `messageId`, `hostname` / `gateway_id`, `error.code`. Same fields as span attributes when it is the same request.

Labels **only**: `service`, `level`, `gateway_id`. Never `chatId`, `userId`, `idempotencyKey` as labels.

OTLP logs from the SDK once stable. Until then, stdout JSON → Collector filelog → Loki. Do not run filelog **and** OTLP logs for the same line.

## Profiles (Pyroscope)

OSS **Grafana Pyroscope**. Gateway CPU/heap while you load-test (doc 13 / k6). Tag profiles with `gateway_id`. No message bodies. This answers “is Socket.IO or `pg` burning CPU?” without guessing.

## Synthetics

Not Playwright. Continuous:

- **blackbox_exporter** or k6: `GET /healthz`, `GET /readyz` every 15s through Nginx.
- k6 (nightly / staging): login + `POST /messages` with a unique `idempotencyKey`; assert one row. Push k6 metrics to Prometheus (remote write or k6 output) so synthetics sit next to SLOs.

## Grafana as code

Provision under `ops/grafana/` (datasources, dashboards, folders, alert contact points). No click-ops as source of truth.

| Folder | Dashboards (import OSS + thin custom) |
| --- | --- |
| **SLO** | Error budget, burn, p99 vs target |
| **Chat** | Send RED, `ws_connected` by `gateway_id`, rate-limit, seq gaps, exemplars |
| **Infra** | Official postgres_exporter, redis_exporter, node/nginx dashboards from grafana.com |
| **Collector** | Collector’s own metrics (refused spans, queue) |

Loki derived field: `requestId` → Tempo. Tempo → logs. Exemplars on `chat_send_duration_seconds`.

Redis Insight / Adminer stay for **keys and rows**. Grafana stays for **SLIs**. Do not rebuild Insight in Grafana.

## Client (Next.js)

Outbox `attemptCount` and transport state in logs (sampled). **Grafana Faro** → Collector later (same OTLP path). Do not block gateway OTel on RUM. No `body` in the browser analytics either.

## Cardinality and PII

- Metric labels: `gateway_id`, `result`, `limit`, `service` — that is the budget.
- Span attributes may include `userId` / `chatId`; logs may too **in the JSON line**, never as Loki labels.
- Redact tokens. Never `body`. Collector processor is the last line of defense.

## What not to do

- Nest → Tempo + Filebeat → Loki with no `requestId`.
- Elasticsearch, Datadog, New Relic as the first lab (hides Collector).
- Jaeger + Zipkin + Tempo.
- Collector + Alloy together on day one.
- 30 uncorrelated dashboards, zero SLOs, zero alerts.
- `chatId` on Prometheus labels.
- Alert on CPU 5% with no SLO.

## Later (after this is boring)

- Prometheus → **Mimir**; Loki → larger retention; Tempo backend scaling.
- Alloy only if you can explain Collector vs Alloy and then pick one agent.
- Phoenix: same spans, same Collector, same SLO queries.
- Tail sampling in Collector for prod traffic.
- Faro RUM + Pyroscope on more services.
- Incident: Grafana OnCall / PagerDuty on Alertmanager receiver.

## Implementation order (Sprint 9)

1. SDK + Collector + Tempo + Loki + Prometheus + Grafana: **one send** correlated (exercises 1–4).
2. spanmetrics + histograms + exemplars.
3. Exporters + provisioned infra dashboards.
4. Recording rules + SLOs + Alertmanager + runbooks.
5. Pyroscope + a k6/blackbox synthetic.
6. Grafana SLO + Chat folders. Collector YAML you can explain line by line.

## Learning exercises (do in order)

1. Collector health/zpages; OTLP accepted.
2. Send; Tempo: `chat.send` → `db.messages.insert`.
3. Retry same `idempotencyKey`; `duplicate=true`; no second INSERT span.
4. Loki `requestId` → Tempo.
5. Kill Redis; error span + log + `chat_send_total{result=error}` + alert (once alerts exist).
6. Two gateways; filter by `gateway_id`; sticky debug.
7. p99 graph with exemplar → that trace.
8. Burn-rate alert in a forced error (fault proxy). Then read Collector YAML.
