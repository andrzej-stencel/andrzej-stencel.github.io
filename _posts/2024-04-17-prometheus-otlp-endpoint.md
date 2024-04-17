---
layout: post
title: "Prometheus OTLP endpoint"
---

# Prometheus OTLP endpoint

Here are the instructions to send metrics to the [Prometheus OTLP endpoint](https://prometheus.io/docs/prometheus/latest/querying/api/#otlp-receiver)
from the OpenTelemetry Collector.

## Run Prometheus

Run Prometheus in Docker:

```shell
docker run --rm -p 9090:9090 quay.io/prometheus/prometheus --config.file=/etc/prometheus/prometheus.yml --enable-feature=otlp-write-receiver
```

Verify that Prometheus is running and that the OTLP feature flag is enabled:

```shell
$ curl -sX POST http://localhost:9090/api/v1/otlp/v1/metrics
unsupported content type: , supported: [application/json, application/x-protobuf]
```

If you don't enable the feature flag and just run Prometheus with `docker run --rm -p 9090:9090 quay.io/prometheus/prometheus`,
you'll rather get this:

```shell
$ curl -sX POST http://localhost:9090/api/v1/otlp/v1/metrics
otlp write receiver needs to be enabled with --enable-feature=otlp-write-receiver
```

## Run OpenTelemetry Collector

I ran Otelcol v0.96.0 with the following configuration:

```yaml
exporters:
  debug:
    verbosity: detailed
  otlphttp:
    endpoint: http://localhost:9090/api/v1/otlp/
    tls:
      insecure: true
receivers:
  otlp:
    protocols:
      grpc:
service:
  pipelines:
    metrics:
      exporters:
        - debug
        - otlphttp
      receivers:
        - otlp
```

## Generate telemetry with `telemetrygen`

I installed the `telemetrygen` from https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/cmd/telemetrygen:

```shell
go install github.com/open-telemetry/opentelemetry-collector-contrib/cmd/telemetrygen@latest
```

The tool is by default installed in `~/go/bin`, make sure you have it in your `$PATH`.
After that, you can generate some metrics:

```shell
telemetrygen metrics --otlp-insecure --metrics 5
```

You should see the metrics in the Otelcol's logs (from Debug exporter's output).

You should also see the metrics in Prometheus UI at http://localhost:9090/graph by looking for `gen` metric name.
Or you can query the API with http://localhost:9090/api/v1/query?query=gen.
