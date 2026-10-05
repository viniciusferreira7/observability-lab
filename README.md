# Observability

# ⚠️ WIP

A study project focused on learning and implementing observability concepts in modern applications using the **Grafana LGTM stack**, fed through an **OpenTelemetry Collector** and backed by **Garage** for S3-compatible object storage.

This stack is used by [fast-feet-api](https://github.com/viniciusferreira7/fast-feet-api) as its observability backend.

## Stack

| Tool | Version | Purpose | Host Port |
|------|---------|---------|-----------|
| **OpenTelemetry Collector** | `contrib:nightly` | Receives OTLP and fans signals out to the backends | 4317 (gRPC), 4318 (HTTP), 8888/8889 (metrics), 13133 (health), 55679 (zpages), 1888 (pprof) |
| **Loki** | x.x.x | Log aggregation | 3100 |
| **Grafana Tempo** | x.x.x | Distributed tracing | — (internal: 3200 HTTP, 4317 gRPC, 4318 OTLP HTTP) |
| **Grafana Mimir** | x.x.x | Long-term metrics storage | 9008, 9009 |
| **Prometheus** | `main-distroless` | Metrics (remote-write receiver enabled) | 9008 |
| **Garage** | x.x.x| S3-compatible object storage for Loki/Tempo/Mimir | 3900 (S3 API), 3903 (Admin API) |
| **Grafana** | x.x.x | Visualization & dashboards | 3000 |

## Goals

- Understand logs, metrics, and tracing (the three pillars of observability)
- Gain hands-on experience monitoring distributed systems
- Explore how to correlate signals across the LGTM stack in practice

## Architecture

Applications send OTLP telemetry to the **OpenTelemetry Collector** (`localhost:4317` gRPC / `localhost:4318` HTTP), which routes each signal to its backend:

- **Logs** → Loki (`/otlp`)
- **Traces** → Tempo (OTLP gRPC)
- **Metrics** → Mimir (remote-write) and exposed locally for Prometheus to scrape

Loki, Tempo, and Mimir persist their data in **Garage** buckets (`loki-data`, `loki-ruler`, `tempo`, `mimir`), which are created at startup by the one-shot `garage-init` service. **Grafana** is provisioned with Loki, Tempo, Mimir, and Prometheus datasources out of the box.