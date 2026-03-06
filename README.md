# Observability Stack

OTEL Collector + Loki + Tempo + Prometheus + Grafana running on Mac Mini.

## Quick Start

```bash
docker compose up -d
```

- **Grafana**: http://localhost:3000 (admin/admin)
- **OTLP gRPC**: localhost:4317
- **OTLP HTTP**: localhost:4318

## Instrumenting FastAPI Apps

### 1. Install dependencies

```bash
uv add opentelemetry-sdk \
  opentelemetry-api \
  opentelemetry-exporter-otlp \
  opentelemetry-instrumentation-fastapi \
  opentelemetry-instrumentation-sqlalchemy \
  opentelemetry-instrumentation-httpx \
  opentelemetry-instrumentation-logging
```

### 2. Add telemetry setup

```python
# app/telemetry.py
import logging

from opentelemetry import trace, metrics
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter
from opentelemetry.exporter.otlp.proto.grpc._log_exporter import OTLPLogExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk._logs import LoggerProvider, LoggingHandler
from opentelemetry.sdk._logs.export import BatchLogRecordProcessor
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
from opentelemetry.instrumentation.logging import LoggingInstrumentor


def setup_telemetry(app, service_name: str, otlp_endpoint: str = "http://localhost:4317"):
    """Initialize OpenTelemetry for a FastAPI app."""
    resource = Resource.create({"service.name": service_name})

    # Traces
    tracer_provider = TracerProvider(resource=resource)
    tracer_provider.add_span_processor(
        BatchSpanProcessor(OTLPSpanExporter(endpoint=otlp_endpoint, insecure=True))
    )
    trace.set_tracer_provider(tracer_provider)

    # Metrics
    metric_reader = PeriodicExportingMetricReader(
        OTLPMetricExporter(endpoint=otlp_endpoint, insecure=True),
        export_interval_millis=15000,
    )
    metrics.set_meter_provider(MeterProvider(resource=resource, metric_readers=[metric_reader]))

    # Logs
    logger_provider = LoggerProvider(resource=resource)
    logger_provider.add_log_record_processor(
        BatchLogRecordProcessor(OTLPLogExporter(endpoint=otlp_endpoint, insecure=True))
    )
    handler = LoggingHandler(logger_provider=logger_provider)
    logging.getLogger().addHandler(handler)

    # Auto-instrument
    FastAPIInstrumentor.instrument_app(app)
    LoggingInstrumentor().instrument(set_logging_format=True)

    return tracer_provider
```

### 3. Wire it into your app

```python
# main.py
from fastapi import FastAPI
from app.telemetry import setup_telemetry

app = FastAPI()
setup_telemetry(app, service_name="jobrex-api", otlp_endpoint="http://<mac-mini-ip>:4317")
```

### 4. Use in code

```python
import logging
from opentelemetry import trace

logger = logging.getLogger(__name__)
tracer = trace.get_tracer(__name__)

@app.post("/jobs")
async def create_job(job: JobCreate):
    with tracer.start_as_current_span("create_job") as span:
        span.set_attribute("job.title", job.title)
        logger.info("Creating job", extra={"job_title": job.title})
        # ... your logic
```

## Accessing from other machines

Point your apps to `http://<mac-mini-ip>:4317` (gRPC) or `http://<mac-mini-ip>:4318` (HTTP).

## Data Retention

All backends are configured for **7-day retention** to conserve disk space.

## Stopping

```bash
docker compose down        # stop, keep data
docker compose down -v     # stop and delete all data
```
