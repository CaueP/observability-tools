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

1. Clone and enter the repo
3. Create a shared docker network
   ```bash
   docker network create shared-observability-network
   ```
2. Start the stack

   ```bash
   docker compose up
   ```

3. Open Grafana
   - URL: [http://localhost:4000](http://localhost:4000)

4. Send telemetry  
   Point your instrumented apps at the OTLP endpoints:

   - **HTTP (from host):** `http://localhost:4318`
   - **gRPC (from host):** `http://localhost:4317`

   These are OpenTelemetry's default endpoints, so no extra config is usually needed when running apps on the host.

   For containers connected to the shared Docker network, see:
   - **Networking and endpoints:** [Use the shared observability Docker network](#use-the-shared-observability-docker-network)  
   - **Signal-specific details (Prometheus, etc.):** [Send telemetry signals](#send-telemetry-signals)

## Ports

| Port | Service      | Purpose                    |
|------|--------------|----------------------------|
| 4000 | Grafana      | Web UI                     |
| 4040 | Pyroscope    | Profiles                   |
| 4317 | OTLP gRPC    | Ingest telemetry           |
| 4318 | OTLP HTTP    | Ingest telemetry           |
| 9090 | Prometheus   | Metrics                    |

## Send telemetry signals

### Use the shared observability Docker network

This stack exposes the `observability-collector` service on the external Docker network `shared-observability-network`.  
Any container attached to this network can reach the collector and Grafana by name:

- **OTLP HTTP (from containers):** `http://observability-collector:4318`
- **OTLP gRPC (from containers):** `http://observability-collector:4317`
- **Grafana UI (from containers):** `http://observability-collector:3000`

From the **host machine**, you still use:

- `http://localhost:4318` / `http://localhost:4317` for OTLP
- `http://localhost:4000` for Grafana

#### How to attach another compose stack to the shared network

1. **Create the shared network once** (if you haven’t already):

   ```bash
   docker network create shared-observability-network
   ```

2. **In your service’s `docker-compose.yaml`**, attach services that should talk to the collector:

   ```yaml
   services:
     my-app:
       image: myorg/myapp
       networks:
         - shared-observability-network

   networks:
     shared-observability-network:
       external: true
   ```

3. **Configure your OTEL exporter inside that service** to send telemetry to the collector:

   - For HTTP OTLP (typical case):
     - `OTEL_EXPORTER_OTLP_ENDPOINT=http://observability-collector:4318`
   - For gRPC OTLP:
     - `OTEL_EXPORTER_OTLP_ENDPOINT=http://observability-collector:4317`

All features that depend on the collector (traces, metrics, logs, profiles, and Prometheus scraping) can use this shared network, so multiple compose stacks can share the same observability backend.

### Prometheus metrics

Prometheus metrics can be scraped from your applications and sent to the LGTM stack via the built-in OpenTelemetry Collector.  
When your services are on the `shared-observability-network`, you can refer to them by their container name in `otelcol-config.yaml`.

#### How to scrape service metrics

The Collector's [`otelcol-config.yaml`](./otelcol-config.yaml) comes pre-configured to scrape its own metrics for visibility. To scrape metrics from your own service(s), edit the `scrape_configs` section under `prometheus/collector` in `otelcol-config.yaml`:

```yaml
receivers:
  prometheus/collector:
    config:
      scrape_configs:
        - job_name: "opentelemetry-collector"
          scrape_interval: 5s
          static_configs:
            - targets: ["127.0.0.1:8888"]
        # Add a job to scrape metrics from your service
        - job_name: "service-metrics"
          scrape_interval: 5s
          scheme: "http"
          metrics_path: "/metrics"
          static_configs:
            # Replace with your service's container name (on the shared-observability-network) and port
            - targets: ["my-app:8000"]
```

**Steps:**

1. **Uncomment or add a new `job_name` block for your service** under `scrape_configs`.
2. **Set the `targets` field** to point to the service exposing Prometheus metrics (hostname and port, e.g., `"my-app:8000"`).
   - If your instrumented service is another Docker container on the `shared-observability-network`, use its container name as the hostname.
3. **Restart the stack** to pick up config changes:

   ```bash
   docker compose restart
   ```

**Tips:**
- The `shared-observability-network` Docker network allows all participating containers to reach `observability-collector` and to be scraped by container name.
- Make sure your application's metrics endpoint is reachable from the collector container.

For more details, see [Prometheus Receiver docs](https://opentelemetry.io/docs/collector/configuration/#prometheus-receiver).



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
