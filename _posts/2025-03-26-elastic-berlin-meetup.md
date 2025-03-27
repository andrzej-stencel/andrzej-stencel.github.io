---
layout: post
title: "Slides from Elastic Berlin meetup in March 2025"
---

Slides from the [Elastic Berlin meetup](https://www.meetup.com/elasticsearch-berlin/events/306518862/) on March 26th, 2025: [2025-03-26-troubleshooting-opentelemetry-collector.pdf](/assets/2025-03-26-troubleshooting-opentelemetry-collector.pdf)

Below are the presentation notes.

## Title slide

Welcome to the talk titled "Troubleshooting the OpenTelemetry Collector with Elastic".

If you're like me and you like to have access to the slides during the presentation,
you can get them by scanning this QR code.

## Slide: Hi, my name is

Hi, my name is Andrzej Stencel, I work as a Senior Software Engineer at Elastic. I am also a maintainer on the OpenTelemetry Collector Contrib project.

And you still get the chance to download the slides here.

If you attended this meetup last October, you may remember that I presented then, also about the OpenTelemetry Collector.

## Slide: October presentation

The reason I'm so invested in this topic is that I work on the Elastic Agent team.

As you may know, the Elastic Agent is the primary way (or one of the primary ways) for users to ingest data into Elasticsearch.

Now, a big theme of the Elastic Agent's  agenda is to integrate the OpenTelemetry Collector with the Elastic Agent.

In fact, you can already use the OpenTelemetry Collector components embedded into the Elastic Agent.

## Slide: link to slides

Last chance to download the slides.

The link points to my GitHub.io page andrzej-stencel.github.io.

You can find the slides there.

## Slide: EDOT Collector

OK, back to the OpenTelemetry Collector.

As I said, you can already today use the OpenTelemetry Collector components embedded into the Elastic Agent.

We call this EDOT Collector, which stands for Elastic Distribution of the OpenTelemetry Collector. Here's how you can use it.

- Download the latest release of the Elastic Agent v8.17.4 (hot off the press, released just yesterday!)

- Unpack the tarball

- Go into the unpacked directory

- And run the Agent.

What is different here is the `otel` command passed to the `elastic-agent` binary, which tells the Agent to run in OTel mode, where "otel" is short for "OpenTelemetry".

Why don't we do this now?

## Terminal: `cd elastic/worklog/2025*`

I have already downloaded and unpacked the Elastic Agent, to save us time and potential trouble with the Wi-fi.

Let's now run the Agent in OTel mode.

```console
cd ./elastic-agent*/
sudo ./elastic-agent otel
```

As you can see, the first line says "Starting in otel mode", and it's followed by the logs from the OpenTelemetry Collector runtime.

If you've ever seen the logs from the Elastic Agent, this looks different.

This actually looks exactly like the logs from an upstream distribution of the OpenTelemetry Collector like otelcol-contrib.

What is different from an upstream distribution is that we did not need to provide a configuration file in the command line.

The EDOT Collector is preconfigured to read a file named `otel.yml` by default.

You can of course change this by explicitly specifying the configuration file:

```console
sudo ./elastic-agent otel --config=my-config.yaml
```

This fails, because the `my-config.yaml` file does not exist.

Ok, so much for the introduction of the EDOT Collector. Let's move on to the main topic of this presentation - troubleshooting the collector!

Let me just do one thing - start an Elastic Serverless environment, where we will send data to.
This will take a couple minutes in the background as we go through the initial steps.

## Browser: Go to cloud.elastic.co, create Serverless Observability project

OK, let's get back to troubleshooting the Collector!

## Slide: Troubleshooting EDOT Collector

First let me emphasize that what I'm going to talk about in this talk works for any distribution of the OpenTelemetry Collector -
whether it is the EDOT Collector embedded in the Elastic Agent,
or an upstream distribution you downloaded from the OpenTelemetry project's releases,
or a distribution that you maybe have built yourself.

What do I mean by troubleshooting?
Let me define it coarsly as finding out if something is doing what I think it should be doing, and when it's not doing it, finding out why - so that I can fix it and make it do what I want.

Does it make sense?

Let's create a simple configuration, run the collector and try to figure out how we can troubleshoot things.

## Slide: Example configuration file

Here's a configuration of the OpenTelemetry Collector to collect logs from a file and send them to Elasticsearch.

- First, we define a pipeline in the `service::pipelines` node. A pipeline is a collection of receivers, processors and exporters (there are also connectors, but let's leave them out of the picture for this presentation).

- In this specific logs pipeline, we define a File Log receiver named `pings` that is expected to read logs from a file named `pings.log`, starting at the end of the file.

- What's in the file, you would ask? This file is populated with the output of the `ping` command, as pictured to the side

- After we collect those logs, we use the Elasticsearch exporter to send them to Elasticsearch.

As we saw a minute before, when we ran the collector, the first thing we have immediate access to is the logs from the collector.

## Slide: Troubleshooting with logs

As many other command line programs do, the OpenTelemetry Collector by default spits out its logs to the console.

Let's run the collector with our prepared configuration and take a brief look at the logs from the collector to see what (if any) valuable information we can get from them.

Before that though, I need to get connection data from that Elastic serverless environment I started in the beginning.

## Browser: Go to cloud.elastic.co

```console
export ELASTIC_ENDPOINT=...
export ELASTIC_API_KEY=...
```

## Terminal: Run `sudo --preserve-env ./elastic-agent-*/elastic-agent otel --config 01-*`

- Setting up own telemetry - doesn't really tell us much
- dedot has been deprecated - this is a useful warning, telling us a change in the default configuration of the Elasticsearch exporter is coming
- Starting ./elastic-agent - this tells us the name of the binary, its version and the number of CPUs on the machine. Hm, OK.
- Starting extensions... - But we don't have any extensions configured
- Starting stanza receiver - This tells us that the `filelog/pings` receiver was started in a logs pipeline.
- finding files - Looks like the File Log receiver was not able to find any files to watch. This looks interesting
- Everything is ready - this line tells us that the configuration has been successfully validated and applied and the collector is now running with that configuration. What exactly is that configuration? We don't really know.

How useful was that? Some of it quite useful, some of it not so much. We found out that the collector is not watching any files. Any ideas why?

Yes, of course, we haven't actually run the `ping` command, so the `pings.log` file does not exist yet.

Let's fix that. Let's run the `ping` command and see if the collector picks it up.

## Terminal: Run `./ping.sh`

All right, the line "Started watching file" showed up in the collector's logs.

It tells us that the collector has picked up the `pings.log` file. Still, we're not seeing any signs of data being actually collected, right?

Let's break the configuration a little to see what happens if the Elasticsearch exporter has trouble exporting.

Let's change the exporter's endpoint to an incorrect one.

```console
$ vim 01*.yaml
# add "/oops" suffix to the endpoint
```

When we re-run the collector, we can see connection errors in the logs.

Now let's change the API key to an incorrect one.

```console
$ vim 01*.yaml
# restore correct endpoint
# suffix the API key
```

When we re-run the collector, we can see authentication (401) errors in the logs.

OK let's restore the correct configuration.

As you can see, when everything is fine, there are no logs from the collector. No news is good news I guess?

Before we summarize what information we can get from the logs, let's see what options we have to customize the logs from the collector.

## Slide: Customizing collector's logs

You can customize the logs output by modifying the configuration file under the `service::telemetry::logs` setting.

First, you can customize the logging level.
This is similar to what's probably available in any logging solution under the Sun.
The available levels are `error`, `warn`, `info` or `debug`.

Set the level to `debug` if you want to get all possible logs, to help you investigate an issue.

Set it to `warn` or `error` to only get really important information that may need to be acted on.

Another important aspect to be aware of is that logs sampling is enabled by default.
This may be a good setting for production, but it's often beneficial to disable sampling when working in a testing environment.

The `disable_caller` option removes some of the information from the logs that may or may not be noise to you depending on your needs.

`encoding: json` may be very helpful when sending the data to a storage like Elasticsearch for indexing, as opposed to reading the logs with your own eyes.

Finally, `output_paths` and `error_output_paths` allows you to easily write the logs to files on disk, from where they can be collected as any other data and shipped to Elasticsearch for indexing and analysis.

Let's summarize what useful information we can get from the collector's logs.

## Slide: Collector's logs useful for

- Noticing and fixing errors like misconfiguration, connection errors, data loss errors
- Noticing warnings like deprecations of configuraions
- Seeing basic information about what the collector is doing, like the File Log receiver starting to watch a file

Looking at the information that we got from the logs we looked at, we found out that the collector was watching the file `pings.log`, but was it actually collecting any data? Was the data exported?

Logs don't give us answers to these more detailed questions like how much data is being collected and exported.

That's where the collector's metrics are invaluable.

## Slide: Troubleshooting with metrics

The good news is, the collector exposes some metrics about itself out of the box.

The metrics are by default exposed in Prometheus exposition format at the endpoint <http://localhost:8888/metrics>.

Let's take a look.

## Browser: Open the metrics endpoint `http://localhost:8888/metrics`

This is a big wall of text. Let's break it down a bit.

The Prometheus exposition format lists each time series (i.e. a metric with a unique set of attribute keys and values) as a separate line.

Before each time series line there are two comment lines: one showing the description of the metric (HELP) and another showing the type of the metric (TYPE).

To get some quantitative information on the data flowing through the collector, let's look closer at the metrics that start with `otelcol_receiver_` and `otelcol_exporter_`.

## Terminal: Run `curl -s http://localhost:8888/metrics | grep '^otelcol_[er]'`

- `otelcol_receiver_accepted_log_records` tells us the number of logs that the receiver accepted
- `otelcol_receiver_refused_log_records` tells us the number of logs that the receiver rejected
- `otelcol_exporter_sent_log_records` tells us the number of logs that the exporter successfully sent
- `otelcol_exporter_send_failed_log_records` tells us the number of logs that the exporter failed to send

Here we can see that the number of logs accepted by the `filelog/pings` receiver is equal to the number of logs exported successfully by the `elasticsearch` exporter. That's good!

If our configuration has more components, each component should have some metrics describing its state.

The metrics may vary across different components, but the ones we just looked at are pretty common, and they are among the most useful too.

Let's take a look at how we can customize the metrics output.

## Slide: Customizing collector's metrics

Analogous to logs, you configure the collector metrics by setting values under `service::telemetry::metrics` in the configuration file.

Perhaps surprisingly, you can configure the level of detail the metrics are reported with, similar to the log level for logs.
The levels are named differently however.
Where logs have `error`, `warn`, `info` and `debug`, metrics have `none`, `basic`, `normal` and `detailed`.
Note that when you set `level` to `none`, the metrics endpoint is not exposed at all. This is a way to disable metrics altogether, which is not really possible with logs.

The second customization available is that you can change the port that the metrics are exposed at by setting `address` option to another port like `address: :8889`.

An important aspect of the metrics endpoint is that by default it is only exposed on the local network interface.
This means that clients from other hosts are not able to reach the endpoint.
This may be important when your collector is running in a containerized environment, for example in Kubernetes.
To change that, set `address: 0.0.0.0:8888`.

When you run the modified metrics configuration, you may notice a warning log from the collector saying `service::telemetry::metrics::address is being deprecated in favor of service::telemetry::metrics::readers`.

This is because what I just showed you is an old and deprecated way of customizing metrics output.

The new, much more capable way to do that is via Readers.

## Slide: New: Send collector's metrics via OTLP

While previously the Prometheus exposition format was the only way to expose the colletor's metrics, with readers you can configure the metrics to be sent via OTLP, for example to another collector instance. This is great!

And what's even greater is that you can do the same for logs and even traces!

## Slide: New: Send collector's logs via OTLP

This is how you configure the collector's logs to be sent out via OTLP.

Note that where metrics have readers, logs have processors.

Inconsistent? Yeah. Confusing? Probably.

Keep in mind that this is all work in progress and may change in future versions, hopefully in a good direction.

We're going to see how this works in a second, but for completeness, let's see how to configure collector's traces.

## Slide: New: Send collector's traces via OTLP

As you can see, the configuration for traces looks dot for dot like the one for logs, and again is different from the metrics one.

Unlike logs and metrics, the collector's traces are not enabled by default, and until recently, it was not possible at all to retrieve them.

This is now possible with this configuration.

## Slide: Monitoring the Collector via OTLP

Let's put all this together and compose a configuration to observe the collector via OTLP.

As described in the preceding slides, we're going to send the collector's logs, metrics and traces via OTLP to an OTLP receiver and then via Elasticsearch exporter to Elasticsearch.

## Terminal: Run EDOT Collector with 04-self-monitoring.yaml

The ultimate goal is to be able to observe the EDOT Collector by collecting its signals in its native OTLP format.

This is not quite ready yet, but things are changing fast in the OpenTelemetry space.

## Slide: Elastic + OpenTelemetry = ❤️

Elastic is thoroughly invested in the OpenTelemetry approach.

A lot of people in the organization are working on it, so keep an eye on the announcements from Elastic related to OpenTelemetry, because things are changing fast.

Every new release of the stack will most likely have some new features making it easier to work with the OpenTelemetry components.

Elastic is among the top contributors to the OpenTelemetry project.

We are currently working on documentation making it easy for users to onboard with OpenTelemetry and get value quickly. This is work in progress.

## Slide: Thank you

Again you get the link to the slides to the left, and to the right you can leave any feedback you like for me and my presentation.

Thank you!
