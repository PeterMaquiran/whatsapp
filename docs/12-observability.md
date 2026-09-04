# 12 — Observability

Phase 3 of your roadmap asks for OpenTelemetry across services. Install the **hooks** in Phase 1 so you are not retrofitting later.

## Trace context

- HTTP: W3C `traceparent`.
- WS envelope `request_id` + inject same trace on `message.send`.
- Span: `chat.send` parent of `db.messages.insert` and `redis.publish`.

Attributes: `chat_id`, `user_id`, `idempotency_key`, `duplicate`, `seq`. No `body`.

## Metrics

| Metric | Type |
| --- | --- |
| `chat_send_total{result=ok\|duplicate\|error}` | counter |
| `chat_send_duration_ms` | histogram |
| `ws_connected` | gauge |
| `outbox_backlog` (client: debug) | gauge |
| `pg_pool_in_use` | gauge |
| `rate_limited_total` | counter |
| `seq_gap_detected_total` | counter |

## Logs

Structured JSON: `event`, `request_id`, `message_id`. Gateway id (`hostname`) to debug sticky/adapter issues.

## Client

Log transport state transitions and outbox `attempt_count`. Sample 1% of successful sends.

## Later

- OTel Collector → Tempo/Jaeger + Prometheus + Loki.
- Phoenix: `:opentelemetry` already maps well; keep the same span names (`chat.send`).
