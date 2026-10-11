# Observability

# ⚠️ WIP

A study project focused on learning and implementing observability concepts in modern applications using the **Grafana LGTM stack**, fed through an **OpenTelemetry Collector** and backed by **Garage** for S3-compatible object storage.

This stack is used by [fast-feet-api](https://github.com/viniciusferreira7/fast-feet-api) as its observability backend.

## Stack

| Tool | Version | Purpose | Host Port |
|------|---------|---------|-----------|
| **OpenTelemetry Collector** | `contrib:0.161.0` | Receives OTLP and fans signals out to the backends | 4317 (gRPC), 4318 (HTTP), 8888 (own metrics), 13133 (health), 55679 (zpages), 1888 (pprof) |
| **Loki** | `main-7bfb71d` | Log aggregation | 3100 |
| **Grafana Tempo** | `r268-049347c-2099-1` | Distributed tracing | — (internal: 3200 HTTP, 4317 gRPC, 4318 OTLP HTTP) |
| **Grafana Mimir** | `r410-1bbffa50` | Long-term metrics storage (not in the metrics pipeline yet) | 9008, 9009 |
| **Prometheus** | `v3.15.0-busybox` | Metrics store and alert rules (remote-write receiver enabled) | 9090 |
| **Garage** | x.x.x| S3-compatible object storage for Loki/Tempo/Mimir | 3900 (S3 API), 3903 (Admin API) |
| **Grafana** | `13.2.3` | Visualization & dashboards | 3000 |

## Goals

- Understand logs, metrics, and tracing (the three pillars of observability)
- Gain hands-on experience monitoring distributed systems
- Explore how to correlate signals across the LGTM stack in practice

## Architecture

Applications send OTLP telemetry to the **OpenTelemetry Collector** (`localhost:4317` gRPC / `localhost:4318` HTTP), which routes each signal to its backend:

- **Logs** → Loki (`/otlp`)
- **Traces** → Tempo (OTLP gRPC)
- **Metrics** → Prometheus (remote-write to `/api/v1/write`); `service.name` becomes the `job` label

Loki, Tempo, and Mimir persist their data in **Garage** buckets (`loki-data`, `loki-ruler`, `tempo`, `mimir`), which are created at startup by the one-shot `garage-init` service. **Grafana** is provisioned with Loki, Tempo and Prometheus datasources and the dashboards in `config/grafana/dashboards/` out of the box.

## Alerts

Prometheus loads `config/prometheus/alert.rules.yml`. The `marketplace` group only
looks at the marketplace-ms services (`api-gateway`, `users-service`,
`products-service`, `checkout-service`, `payments-service`), so other projects on
this stack never trigger it.

| Alert | Fires when | For | Severity |
|-------|------------|-----|----------|
| `ServiceDown` | A service that reported metrics in the last hour sent nothing in the last minute | 30s | critical |
| `HighErrorRate` | More than 10% of a service's requests return 5xx (1m window) | 1m | warning |
| `HighLatencyP95` | A service's P95 request latency is above 2000ms (5m window) | 1m | warning |
| `HighMemoryUsage` | A service's V8 heap is above 512MiB | 2m | warning |
| `NoPaymentsProcessed` | payments-service made no charge in 5 minutes | — | info |
| `HighPaymentRejectionRate` | More than 50% of payments-service charges are rejected (5m window) | 2m | warning |

There is no Alertmanager: alerts show up at <http://localhost:9090/alerts> and on
the Grafana **Overview** dashboard, but nobody is notified.

The rules are unit-tested with `promtool`, from the Prometheus image, with no stack running:

```bash
docker compose run --rm --no-deps --entrypoint promtool prometheus check rules /etc/prometheus/alert.rules.yml
docker compose run --rm --no-deps --entrypoint promtool prometheus test rules /etc/prometheus/alert.rules.test.yml
```

## Dashboards

Dashboards in `config/grafana/dashboards/` are provisioned read-only.

- **Overview** (`/d/marketplace-overview`) — firing alert counts per severity and a
  table of the marketplace alerts currently pending or firing.
