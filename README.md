# Observability Stack — SigNoz

Self-hosted SigNoz (logs, traces, metrics) backed by ClickHouse.

## Quick Start

```bash
docker compose up -d
```

- **SigNoz UI**: http://localhost:9100
- **OTLP gRPC**: localhost:4317
- **OTLP HTTP**: localhost:4318

On first launch you'll be asked to create an admin account.

## Retention

Set to **7 days** for logs and traces via SigNoz UI:
**Settings > General > Retention** after first login.

## Sending Telemetry from Apps

Your FastAPI apps use the `opentelemetry` Python SDK. Set this env var:

```
OTEL_EXPORTER_OTLP_ENDPOINT=http://host.docker.internal:4317
```

(`host.docker.internal` if the app runs in a separate Docker Compose stack)

## Stopping

```bash
docker compose down        # stop, keep data
docker compose down -v     # stop and delete all data
```
