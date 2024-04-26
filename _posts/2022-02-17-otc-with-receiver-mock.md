---
layout: post
title: "OTC with receiver-mock"
published: false
---

# OTC with receiver-mock

Here's an example config file for [Sumo Logic OpenTelemetry Collector](https://github.com/SumoLogic/sumologic-otel-collector/) to send data to Sumo Logic's [receiver-mock](https://github.com/SumoLogic/sumologic-kubernetes-tools/tree/v2.9.0/src/rust/receiver-mock):

```sh
cat <<EOF > config.yaml
exporters:
#   logging:
#     loglevel: debug
  sumologic:
    endpoint: http://localhost:3000/
    metric_format: prometheus

receivers:
  telegraf:
    separate_field: false
    agent_config: |
      [agent]
        interval = "3s"
        flush_interval = "3s"
      [[inputs.mem]]

service:
  pipelines:
    metrics:
      receivers:
      - telegraf
      exporters:
#       - logging
      - sumologic
EOF
```

The important part is to set [sumologicexporter](https://github.com/SumoLogic/sumologic-otel-collector/blob/v0.0.50-beta.0/pkg/exporter/sumologicexporter/README.md)'s output format to anything other than OTLP,
as `receiver-mock` currently cannot speak OTLP.

Uncomment the [loggingexporter](https://github.com/SumoLogic/sumologic-otel-collector/blob/v0.0.50-beta.0/docs/Configuration.md#logging-exporter) bits if you want to see the data in OTC console logs.

Before running an OTC instance with this config, start a `receiver-mock` instance:

```sh
docker run --rm -it -p 3000:3000 sumologic/kubernetes-tools:2.9.0 receiver-mock --print-metrics
```

Now you can run your OpenTelemetry Collector in another console.
Here's a bit to download one and run it on a Linux amd64 machine:

```sh
curl -Lo otelcol-sumo "https://github.com/SumoLogic/sumologic-otel-collector/releases/download/v0.0.50-beta.0/otelcol-sumo-0.0.50-beta.0-linux_amd64"
chmod a+x ./otelcol-sumo
./otelcol-sumo --config config.yaml
```

You should see the memory metrics from Telegraf in receiver-mock's output.
