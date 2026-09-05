# 12 — Observability

Goal is **to learn production telemetry**, not to ship a dashboard quickly. The app never talks to Tempo, Loki, Prometheus, or Grafana. It speaks **OpenTelemetry (OTLP)** only. The **OpenTelemetry Collector** routes signals. That is the pattern you will see at real companies; swapping a backend later is collector config, not a rewrite.

Install the **SDK hooks in Phase 1**. Stand up the full local stack when you can follow one `message.send` through traces, logs, and metrics. Do not skip the collector and export from Node straight to Tempo — that is the “finish faster” path and it teaches the wrong boundary.

## Three signals (learn these names)

| Signal | Question it answers | This product |
| --- | --- | --- |
| **Traces** | What happened on *this* send? Which hop was slow? | Grafana Tempo |
| **Metrics** | Is the *system* healthy? Error rate, pool, connections | Prometheus |
| **Logs** | What did *this* process print, with context? | Loki |

Grafana is the **UI**. It is not a backend. The OTel Collector is the **router**. Tempo / Loki / Prometheus are **stores**.

Correlation is the skill: one `trace_id` on the WS envelope, on every span, on every log line. In Grafana you jump log → Tempo timeline. That is more important than pretty panels.

## Why this stack (learning, not fashion)

| Piece | Role | Why not skip it |
| --- | --- | --- |
| **OpenTelemetry SDK** (Node, later Elixir) | Create spans / metrics / log correlation | Vendor-neutral. Phoenix `:opentelemetry` keeps the same span names (`chat.send`). |
| **OpenTelemetry Collector** | Receive OTLP; optional scrape; export to backends | Portable collector skill (CNCF). Batch, retry, drop PII, route. Same binary in front of Node and Phoenix. **Grafana Alloy** is the Grafana-native agent (Promtail successor) — add it later if you want Grafana-dialect scrape/tail; do not run Collector **and** Alloy at the start. |
| **Grafana Tempo** | Trace store | Same OTLP span model, Grafana Explore / TraceQL, exemplars with Prometheus later. **Zipkin** / **Jaeger** are extra UIs; skip them until Tempo in Grafana is boring. |
| **Loki** | Log store | Label-indexed (not full-text Elasticsearch). Forces you to design labels (`service`, `gateway`, `level`) and put `trace_id` in the line. That is the Loki lesson. |
| **Prometheus** | Metrics store | Grafana without Prometheus is a half stack. Learn exposition, scrape, histograms, recording rules. **Mimir** is Prometheus-at-scale later. |
| **Grafana** | Dashboards + Explore | One pane: Prometheus, Loki LogQL, Tempo traces, derived field `trace_id` → Tempo. |

**Do not use Grafana Cloud as the first lab.** Run Compose locally so each box has a port you can curl. Cloud hides the topology you are trying to learn.

## Process topology

```
[ Node gateway / later Phoenix ]
  OTel SDK
  W3C traceparent on HTTP
  envelope request_id === trace_id on message.send
           │  OTLP (HTTP or gRPC) only
           ▼
[ OpenTelemetry Collector ]
  otlp receiver
  optional prometheus receiver (postgres / redis exporters)
  optional filelog (if SDK logs are not OTLP yet)
           │
     ┌─────┼──────────────┐
     ▼     ▼              ▼
 [Tempo] [Loki]   [Prometheus]
     └─────┴──────────────┘
                 ▼
            [Grafana]
```

Compose services (add when you instrument, not on day one of Postgres): `otel-collector`, `tempo`, `loki`, `prometheus`, `grafana`. Gateways stay 2×; Collector is one service they both send to.

## Trace context

- HTTP: W3C `traceparent` (and `tracestate` if you use it).
- WS envelope `request_id` = trace id (or a span id you can join). Inject the same context in `ChatService.send`.
- Span tree: `chat.send` parent of `db.messages.insert` and `redis.publish`. Later: `tus.upload`, `scan`, `cdn`.
- Attributes: `chat_id`, `user_id`, `device_id`, `idempotency_key`, `duplicate`, `seq`. **No `body`.** No access tokens.
- Sampling: 100% in local/dev so you can learn. Production: head-sample (e.g. 1–10%) plus **always-on** for errors. Learn why tail sampling exists before you enable it.

Tempo lesson: in Grafana Explore → Tempo, find `chat.send`, confirm child spans, confirm a duplicate send is a short trace with `duplicate=true` and no second INSERT.

## Metrics

Instrument with OTel; Collector exports to Prometheus (or Prometheus scrapes the Collector’s Prometheus exporter). Prefer **histograms** for latency (you will need percentiles).

| Metric | Type |
| --- | --- |
| `chat_send_total{result=ok\|duplicate\|error}` | counter |
| `chat_send_duration_ms` | histogram |
| `ws_connected` | gauge |
| `outbox_backlog` (client: debug) | gauge |
| `pg_pool_in_use` | gauge |
| `rate_limited_total` | counter |
| `seq_gap_detected_total` | counter |

RED for the send path: Rate (`chat_send_total`), Errors (`result=error`), Duration (histogram). USE for gateways: Utilization (`pg_pool_in_use`, CPU), Saturation (`ws_connected` vs limit), Errors.

## Logs

Structured JSON: `event`, `trace_id` / `request_id`, `message_id`, `hostname` (sticky/adapter debug). Same fields as span attributes when it is the same request.

Loki labels: **low cardinality** only — `service=gateway`, `level`, maybe `gateway_id`. Never `chat_id` or `user_id` as labels (cardinality explosion). Those stay in the JSON line.

Collector: if the process still writes stdout JSON, filelog/docker logs → Loki; once OTel logs are stable, OTLP → Loki is enough. Learn one path fully before enabling both.

## Client

Log transport state transitions and outbox `attempt_count`. Sample 1% of successful sends. Client traces are optional later (RUM); do not block server OTel on them.

## Grafana (what “done” looks like)

1. Datasources: Prometheus, Loki, Tempo.
2. Loki derived field: regex `trace_id` → Tempo query.
3. One dashboard: `ws_connected` per gateway, send RED, PG pool, rate-limit count.
4. Explore: pick a failed send in Loki → open Tempo → see whether PG or Redis failed.

If you cannot do (4), the stack is not wired. Dashboards without correlation are decoration.

## What not to do (speed traps)

- App exporter → Tempo + app filebeat → Loki + `/metrics` scraped with no trace ids.
- Elasticsearch “because logs.” Loki is the Grafana-stack lesson; ES is a different career path.
- Tempo + Jaeger + Zipkin at once. One trace backend until you are bored.
- OTel Collector and Grafana Alloy together on day one.
- Logging `body`, tokens, or raw Authorization headers.

## Later (after this stack is boring)

- Prometheus → **Mimir** if cardinality/retention hurts.
- **Grafana Alloy** for Grafana-native log tail / extra scrapes, still OTLP from the app (or replace Collector only when you can explain both configs).
- Phoenix: same span names, OTLP to the same Collector.
- Exemplars: Prometheus histogram bucket → Tempo trace.
- TraceQL and Tempo metrics-generator.

## Learning exercises (do in order)

1. Hit Collector health/zpages after one HTTP health check with the SDK on; confirm OTLP is accepted.
2. Send a chat message; in Grafana Tempo find `chat.send` → `db.messages.insert`.
3. Retry the same `idempotency_key`; see `duplicate=true` and no second insert span.
4. Grep Loki for that `trace_id`; click through to Tempo from Grafana.
5. Kill Redis; send; confirm error span + log + `chat_send_total{result=error}`.
6. Scale two gateways; filter Loki by `hostname`; confirm sticky debug.
7. Only then: draw a Grafana dashboard. Then read the Collector YAML until you can explain each receiver / processor / exporter.
