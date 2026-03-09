# Observability Tools

A local OpenTelemetry observability stack for development, demos, and testing. Uses [Grafana's docker-otel-lgtm](https://github.com/grafana/docker-otel-lgtm) to run Loki, Grafana, Tempo, Prometheus/Mimir, and Pyroscope in a single container, with a custom OpenTelemetry Collector configuration.

## Goal

Provide a ready-to-use observability backend that:

- Accepts OTLP telemetry (traces, metrics, logs, profiles) from your applications
- Stores and visualizes data in Grafana
- Avoids CORS issues when sending telemetry from browser-based apps
- Persists data across restarts
- Supports optional export to external backends (e.g., Grafana Cloud)

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/)
- [Docker Compose](https://docs.docker.com/compose/install/)

## Quick Start

1. **Clone and enter the repo**
2. **Start the stack**

   ```bash
   docker compose up
   ```

3. **Open Grafana**
   - URL: [http://localhost:4000](http://localhost:4000)

4. **Send telemetry**

   Point your instrumented apps at the OTLP endpoints:

   - **HTTP:** `http://localhost:4318`
   - **gRPC:** `http://localhost:4317`

   These are OpenTelemetry's default endpoints, so no extra config is usually needed.

## Ports

| Port | Service      | Purpose                    |
|------|--------------|----------------------------|
| 4000 | Grafana      | Web UI                     |
| 4040 | Pyroscope    | Profiles                   |
| 4317 | OTLP gRPC    | Ingest telemetry           |
| 4318 | OTLP HTTP    | Ingest telemetry           |
| 9090 | Prometheus   | Metrics                    |

## Configuration

### `.env`

Environment variables for the LGTM stack. Copy and edit as needed:

- **Logging:** `ENABLE_LOGS_ALL=true` or per-component (`ENABLE_LOGS_GRAFANA`, `ENABLE_LOGS_LOKI`, etc.)
- **External export:** `OTEL_EXPORTER_OTLP_ENDPOINT` and `OTEL_EXPORTER_OTLP_HEADERS` for Grafana Cloud or other OTLP backends
- **OBI (eBPF):** `ENABLE_OBI=true` for zero-code instrumentation (Linux only)
- **Plugins:** `GF_PLUGINS_PREINSTALL` for pre-installed Grafana plugins

### `otelcol-config.yaml`

Custom OpenTelemetry Collector config mounted into the container. It:

- Receives OTLP on gRPC (4317) and HTTP (4318)
- Allows all origins for CORS (`allowed_origins: ["*"]`)
- Scrapes Prometheus metrics from the collector
- Routes traces, metrics, logs, and profiles to Tempo, Mimir, Loki, and Pyroscope

Edit this file to change pipelines, add processors, or adjust exporters.

## Data Persistence

Data is stored under `./container/`:

- `container/grafana/` — Grafana dashboards and settings
- `container/prometheus/` — Prometheus metrics
- `container/loki/` — Loki logs

`container/` is in `.gitignore` and is not committed.

## Stopping

```bash
docker compose down
```

Data in `./container/` is kept unless you remove those directories.

## References

- [Grafana docker-otel-lgtm](https://github.com/grafana/docker-otel-lgtm)
- [Grafana docs: Docker OpenTelemetry LGTM](https://grafana.com/docs/opentelemetry/docker-lgtm/)
- [OpenTelemetry](https://opentelemetry.io/docs)
