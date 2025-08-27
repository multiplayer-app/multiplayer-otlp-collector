![Description](.github/header-otlpCollector.png)

<div align="center">
<a href="https://github.com/multiplayer-app/multiplayer-otlp-collector">
  <img src="https://img.shields.io/github/stars/multiplayer-app/multiplayer-otlp-collector.svg?style=social&label=Star&maxAge=2592000" alt="GitHub stars">
</a>
  <a href="https://github.com/multiplayer-app/multiplayer-otlp-collector/blob/main/LICENSE">
    <img src="https://img.shields.io/github/license/multiplayer-app/multiplayer-otlp-collector" alt="License">
  </a>
  <a href="https://multiplayer.app">
    <img src="https://img.shields.io/badge/Visit-multiplayer.app-blue" alt="Visit Multiplayer">
  </a>
  
</div>
<div>
  <p align="center">
    <a href="https://x.com/trymultiplayer">
      <img src="https://img.shields.io/badge/Follow%20on%20X-000000?style=for-the-badge&logo=x&logoColor=white" alt="Follow on X" />
    </a>
    <a href="https://www.linkedin.com/company/multiplayer-app/">
      <img src="https://img.shields.io/badge/Follow%20on%20LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white" alt="Follow on LinkedIn" />
    </a>
    <a href="https://discord.com/invite/q9K3mDzfrx">
      <img src="https://img.shields.io/badge/Join%20our%20Discord-5865F2?style=for-the-badge&logo=discord&logoColor=white" alt="Join our Discord" />
    </a>
  </p>
</div>

# Multiplayer OTLP Collector

A custom OpenTelemetry Collector configuration for Multiplayer's telemetry data processing and routing.

## Overview

This OTLP Collector is designed to receive, process, and forward telemetry data (traces and logs) to Multiplayer's API. It includes specialized filtering and processing for different trace types used by Multiplayer's session recording and debugging features.

## Prerequisites

- Docker and Docker Compose installed
- Multiplayer API credentials (OTLP key)
- Network access to `https://otlp.multiplayer.app:443`

## Quick Start

### Launch the Collector

Docker-compose [configuration](./docker-compose.yml) 

```yaml
services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib-dev:e9276a78341765d080347ee23f274f01e562fe5a
    platform: linux/amd64
    container_name: otel-collector
    hostname: otel-collector
    restart: on-failure
    ports:
      - "4317:4317"   # gRPC OTLP
      - "4318:4318"   # HTTP OTLP
      - "13133:13133" # Health check
    command: ["--config=/etc/otel/config.yaml"]
    environment:
      MULTIPLAYER_OTLP_KEY: "{{MULTIPLAYER_OTLP_KEY}}"
      MULTIPLAYER_OTLP_COLLECTOR_ENDPOINT: https://otlp.multiplayer.app:443
    volumes:
      - ./config.yaml:/etc/otel/config.yaml
```

**Note**: The `platform: linux/amd64` specification ensures compatibility with Apple Silicon Macs by running the container under emulation.

To start run command: `docker-compose up -d`

The collector will start and be available on:

- **gRPC endpoint**: `localhost:4317`
- **HTTP endpoint**: `localhost:4318`
- **Health check**: `http://localhost:13133/health/status`


### Configuration

The [config.yaml](./config.yaml) file defines the collector's behavior:

#### Extensions

Health monitoring for the collector:

```yaml
extensions:
  healthcheckv2:
    use_v2: true
    component_health:
      include_permanent_errors: false
      include_recoverable_errors: true
      recovery_duration: 5m
    http:
      endpoint: "0.0.0.0:13133"
      status:
        enabled: true
        path: "/health/status"
      config:
        enabled: true
        path: "/health/config"

receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
        cors:
          allowed_origins:
            - "*"
          allowed_headers:
            - "*"

processors:
  transform/set_trace_type:
    error_mode: ignore
    trace_statements:
      - context: span
        statements:
          - set(resource.attributes["multiplayer.trace_type"], Substring(trace_id.string, 0, 6))
    log_statements:
      - context: log
        statements:
          - set(resource.attributes["multiplayer.trace_type"], Substring(trace_id.string, 0, 6))

  groupbytrace:
    wait_duration: 10s
    num_traces: 1000
    num_workers: 2

  batch:
    send_batch_size: 5
    send_batch_max_size: 5
    timeout: 3s

  memory_limiter/deb:
    check_interval: 1s
    limit_percentage: 80
    spike_limit_percentage: 20

  memory_limiter/cdb:
    check_interval: 1s
    limit_percentage: 80
    spike_limit_percentage: 20

  resourcedetection/system:
    detectors: ["system"]
    system:
      hostname_sources: ["os"]

  filter/deb:
    error_mode: ignore
    traces:
      span:
        - resource.attributes["multiplayer.trace_type"] != "debdeb"
        - attributes["http.target"] == "/jaeger/v1/traces"
        - attributes["http.target"] == "/v1/traces"
        - attributes["http.target"] == "/v1/logs"
        - attributes["http.route"] == "/health"
        - attributes["http.route"] == "/healthz"
    logs:
      log_record:
        - resource.attributes["multiplayer.trace_type"] != "debdeb"

  filter/cdb:
    error_mode: ignore
    traces:
      span:
        - resource.attributes["multiplayer.trace_type"] != "cdbcdb"
        - attributes["http.target"] == "/jaeger/v1/traces"
        - attributes["http.target"] == "/v1/traces"
        - attributes["http.target"] == "/v1/logs"
        - attributes["http.route"] == "/health"
        - attributes["http.route"] == "/healthz"
    logs:
      log_record:
        - resource.attributes["multiplayer.trace_type"] != "cdbcdb"

  filter/not-deb-cdb:
    error_mode: ignore
    traces:
      span:
        - attributes["multiplayer.trace_type"] != "debdeb"
        - attributes["multiplayer.trace_type"] != "cdbcdb"
        - attributes["http.target"] == "/jaeger/v1/traces"
        - attributes["http.target"] == "/v1/traces"
        - attributes["http.target"] == "/v1/logs"
        - attributes["http.route"] == "/health"
        - attributes["http.route"] == "/healthz"

exporters:
  otlphttp/multiplayer:
    endpoint: "${MULTIPLAYER_OTLP_COLLECTOR_ENDPOINT}"
    headers:
      Authorization: "${MULTIPLAYER_OTLP_KEY}"
    timeout: 10s
    encoding: json

service:
  extensions: [healthcheckv2]

  pipelines:
    traces/deb:
      receivers: [otlp]
      processors:
        - transform/set_trace_type
        - filter/deb
        - memory_limiter/deb
        - resourcedetection/system
        - batch
      exporters: [otlphttp/multiplayer]

    logs/deb:
      receivers: [otlp]
      processors:
        - transform/set_trace_type
        - filter/deb
        - memory_limiter/deb
        - batch
      exporters: [otlphttp/multiplayer]

    traces/cdb:
      receivers: [otlp]
      processors:
        - transform/set_trace_type
        - filter/cdb
        - memory_limiter/cdb
        - resourcedetection/system
        - batch
      exporters: [otlphttp/multiplayer]

    logs/cdb:
      receivers: [otlp]
      processors:
        - transform/set_trace_type
        - filter/cdb
        - memory_limiter/cdb
        - resourcedetection/system
        - batch
      exporters: [otlphttp/multiplayer]
```

This is a configuration file for the OpenTelemetry Collector, describing how to receive data from receivers, process it, and export it to specified destinations. Here’s a detailed explanation of each section:

### Receivers

`receivers` are the entry points for data into the OpenTelemetry Collector. Here, we define an OTLP HTTP and gRPC receiver (OpenTelemetry Protocol).

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
        cors:
          allowed_origins:
            - "*"
          allowed_headers:
            - "*"
```

### Processors

`processors` define the intermediate steps used to process data. In this configuration file, no processors are defined.

```yaml
processors:
  transform/set_trace_type:
    # set multiplayer trace type

  batch:
    # export collected traces in batches

  memory_limiter/deb:
    # limit memory usage for multiplayer traces

  memory_limiter/cdb:
    # limit memory usage for multiplayer traces

  resourcedetection/system:
    # automatically detects and adds resource attributes to telemetry dataa

  filter/deb:
    # leave only session recorder traces

  filter/cdb:
    # leave only session recorder traces
```

### Extensions

extensions are additional functional modules, such as health checks.

```yaml
extensions:
  healthcheckv2:
    # ...
```

health_check: Sets up a health check extension to monitor the health of the OpenTelemetry Collector.

### Exporters

exporters are the exit points for data out of the OpenTelemetry Collector.

```yaml
otlphttp/multiplayer:
  # ...
```

## Service

service defines the service configuration for the OpenTelemetry
Collector, including data pipelines.

```yaml
service:
  extensions: [healthcheckv2]

  pipelines:
    traces/deb:
      receivers: [otlp]
      processors:
        - transform/set_trace_type
        - filter/deb
        - memory_limiter/deb
        - resourcedetection/system
        - batch
      exporters: [otlphttp/multiplayer]

    logs/deb:
      receivers: [otlp]
      processors:
        - transform/set_trace_type
        - filter/deb
        - memory_limiter/deb
        - batch
      exporters: [otlphttp/multiplayer]

    traces/cdb:
      receivers: [otlp]
      processors:
        - transform/set_trace_type
        - filter/cdb
        - memory_limiter/cdb
        - resourcedetection/system
        - batch
      exporters: [otlphttp/multiplayer]

    logs/cdb:
      receivers: [otlp]
      processors:
        - transform/set_trace_type
        - filter/cdb
        - memory_limiter/cdb
        - resourcedetection/system
        - batch
      exporters: [otlphttp/multiplayer]
```

## License

MIT — see [LICENSE](./LICENSE).
