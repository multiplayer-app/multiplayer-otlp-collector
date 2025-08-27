![Description](.github/header-otlpCollector.png)

<div align="center">
<a href="https://github.com/multiplayer-app/multiplayer-otlp-collector">
  <img src="https://img.shields.io/github/stars/multiplayer-app/multiplayer-otlp-collector.svg?style=social&label=Star&maxAge=2592000" alt="GitHub stars">
</a>
  <a href="./LICENSE">
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

# Multiplayer OpenTelemetry (OTLP) Collector


The OpenTelemetry Collector offers a vendor-agnostic implementation of how to receive, process and export telemetry data. It removes the need to run, operate, and maintain multiple agents/collectors. This works with improved scalability and supports open source observability data formats (e.g. Jaeger, Prometheus, Fluent Bit, etc.) sending to one or more open source or commercial backends.

This repository provides an example OpenTelemetry Collector configuration which sends OTLP traces/logs for Multiplayer Full Stack Session recordings to Multiplayer while also sending OTLP data to any other observability platform.

Learn more about the OpenTelemetry Collector and configuration here: https://opentelemetry.io/docs/collector/

## Prerequisites

- Docker and Docker Compose: https://www.docker.com/
- Multiplayer API credentials (OTLP key): https://www.multiplayer.app/docs/
- Network access to `https://otlp.multiplayer.app:443`

## Quick Start

### Run the Collector

Source code for the OTLP Collector can be found here: https://github.com/open-telemetry/opentelemetry-collector-contrib

Prebuilt docker images can be found here: https://hub.docker.com/r/otel/opentelemetry-collector-contrib

Below is docker-compose [configuration](./docker-compose.yml) which will start to run OTLP Collector which will run on default ports:

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

### Environment variables

Docker compose will set the following environment variables:
- `MULTIPLAYER_OTLP_COLLECTOR_ENDPOINT = https://otlp.multiplayer.app:443`
- `MULTIPLAYER_OTLP_KEY = {{YOUR_MULTIPLAYER_OTLP_KEY}}` 

OTLP configuration will replace environment variables in following way: `${ENVIRONMENT_VARIABLE_NAME}`

### OTLP Collector Configuration

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

  memory_limiter/multiplayer_session_recorder_manual:
    # limit memory usage for multiplayer traces

  memory_limiter/multiplayer_session_recorder_continuous:
    # limit memory usage for multiplayer traces

  resourcedetection/system:
    # automatically detects and adds resource attributes to telemetry data

  filter/multiplayer_session_recorder_manual:
    # leave only manual session recorder traces

  filter/multiplayer_session_recorder_continuous:
    # leave only continuous session recorder traces
```

### Extensions

Extensions are additional functional modules, such as health checks.

```yaml
extensions:
  healthcheckv2:
    # ...
```

`health_check`: Sets up a health check extension to monitor the health of the OpenTelemetry Collector.

### Exporters

Exporters are the exit points for data out of the OpenTelemetry Collector.

```yaml
exporters:
  otlphttp/multiplayer:
    endpoint: "${MULTIPLAYER_OTLP_COLLECTOR_ENDPOINT}"
    headers:
      Authorization: "${MULTIPLAYER_OTLP_KEY}"
    timeout: 10s
    encoding: json
```

## Service

Service defines the service configuration for the OpenTelemetry
Collector, including data pipelines.

```yaml
service:
  extensions: [healthcheckv2]

  pipelines:
    traces/multiplayer_session_recorder_manual:
      receivers: [otlp]
      processors:
        - transform/set_trace_type
        - filter/multiplayer_session_recorder_manual
        - memory_limiter/multiplayer_session_recorder_manual
        - resourcedetection/system
        - batch
      exporters: [otlphttp/multiplayer]

    logs/multiplayer_session_recorder_manual:
      receivers: [otlp]
      processors:
        - transform/set_trace_type
        - filter/multiplayer_session_recorder_manual
        - memory_limiter/multiplayer_session_recorder_manual
        - batch
      exporters: [otlphttp/multiplayer]

    traces/multiplayer_session_recorder_continuous:
      receivers: [otlp]
      processors:
        - transform/set_trace_type
        - filter/multiplayer_session_recorder_continuous
        - memory_limiter/multiplayer_session_recorder_continuous
        - resourcedetection/system
        - batch
      exporters: [otlphttp/multiplayer]

    logs/multiplayer_session_recorder_continuous:
      receivers: [otlp]
      processors:
        - transform/set_trace_type
        - filter/multiplayer_session_recorder_continuous
        - memory_limiter/multiplayer_session_recorder_continuous
        - resourcedetection/system
        - batch
      exporters: [otlphttp/multiplayer]
```

## License

MIT — see [LICENSE](./LICENSE).
