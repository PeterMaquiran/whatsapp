# 16 — Local consoles (OSS, not custom)

Do **not** build chat admin panels, Redis UIs, or a second Grafana. Use upstream open source. Bind every console to **127.0.0.1**. They are a local lab, not production.

Two different jobs:

| Job | Question | Tool |
| --- | --- | --- |
| **Inspect** | What keys/rows exist right now? | Redis Insight, Adminer |
| **Operate** | Is the system in SLO? Why was this send slow? | **Grafana** ← Prometheus / Loki / Tempo / Pyroscope / Alertmanager |

Redis Insight does not replace Prometheus. Grafana does not replace looking at `presence:{userId}` in Redis.

## Compose (Sprint 1)

`docker compose up -d` in the repo root.

| Service | Image | Local URL | Login / notes |
| --- | --- | --- | --- |
| Postgres | `postgres:16-alpine` | `localhost:5432` | user/password/db `chat` / `chat` / `chat` |
| Redis | `redis:7-alpine` | `localhost:6379` | no password in dev |
| **Redis Insight** | `redis/redisinsight` | http://127.0.0.1:5540 | Add DB: host **`redis`** (Docker DNS), port `6379`. From the host machine use host `127.0.0.1` only if Insight ran on the host, not in Compose |
| **Adminer** | `adminer` | http://127.0.0.1:8080 | System **PostgreSQL**, server **`postgres`**, user `chat`, password `chat`, database `chat` |

Look at: `identities`, `devices`, `messages.idempotency_key`, Redis `rl:send:*`, `typing:*`, Socket.IO adapter channels (ephemeral). Never paste `body` into screenshots you share.

## Per component (what to open)

| Part of the system | Inspect (OSS GUI) | Metrics / health (Grafana, later) | Do not use |
| --- | --- | --- | --- |
| PostgreSQL | **Adminer** (or `psql`) | **postgres_exporter** → Prometheus → Grafana | Custom schema browser |
| Redis | **Redis Insight** | **redis_exporter** → Grafana (memory, hits, connected clients) | Redis Commander *and* Insight together |
| Nest gateways | — | App OTLP → Collector → **Grafana** (`ws_connected`, send RED, PG pool) | A homemade Socket.IO admin |
| Socket.IO / adapter | Insight: pub/sub noise; Loki: sticky/`hostname` | `ws_connected` per gateway | |
| Nginx | `stub_status` (optional) | **nginx-prometheus-exporter** later | Nginx Amplify |
| Next.js | Playwright / browser | Optional later RUM; not Sprint 1 | |
| Tempo / Loki / Prometheus | — | **Grafana** is the UI | Jaeger + Zipkin + extra Grafana |
| Object storage (media later) | **MinIO Console** when MinIO exists | MinIO metrics → Grafana | |
| Keycloak (backlog) | Keycloak admin console | Keycloak metrics if you enable them | |

Grafana **Explore** and **SLOs** first (doc 12). Provisioned folders: SLO, Chat, Infra, Collector. Import exporter dashboards; do not rebuild Redis Insight inside Grafana. Redis Insight / Adminer stay for keys and rows.

## Sprint 9 exporters (still OSS, still not custom)

Add to Compose with the Collector stack, not on day one of Postgres:

- `prometheus/postgres-exporter`
- `oliver006/redis_exporter`
- optional `nginx/nginx-prometheus-exporter`

Prometheus scrapes exporters (or the Collector does). Grafana remains the only metrics UI.

## Production

Adminer, Redis Insight, and unauthenticated exporters stay **off** the public network. Staging/prod: Grafana (SSO), cloud Postgres/Redis consoles if the provider has them, no Compose GUIs on the edge.

See [12 — Observability](./12-observability.md), [ADR-013](./14-decisions.md).
