Using Debezium Platform with built-in Monitoring
===

This example walks you through running the Debezium Management Platform with **monitoring enabled**:
Debezium Server pipelines export metrics via OpenTelemetry to an OTel Collector, which exposes them to
Prometheus, and the Debezium-platform-stage(UI) queries Prometheus (through the Conductor) to draw the
**Monitoring** tab of a pipeline.

This example builds directly on [`../postgresql-kafka-example`](../postgresql-kafka-example). If you haven't
already, read that one first — it walks through the Debezium-platform-stage(UI) in detail (Connections,
Sources, Destinations, Transforms, Pipeline) and explains every field. **This example does not repeat that
walkthrough.** Here we:

- stand the platform up with monitoring wired in from the start,
- create the same kind of pipeline using a one-shot seed script instead of clicking through the UI (no
  Transform this time — it isn't needed to demonstrate monitoring),
- then shift the UI focus to the part that *is* new: the **Monitoring** tab, its panels, and how to extend it.

Every step below is backed by a script in this directory, so you can either read along and run the scripts, or
open them to see exactly what they do.

Mental model — where does a number on a panel come from?
---
```
 PostgreSQL (source DB)                      Kafka (Strimzi, destination)
       │  CDC                                        ▲
       ▼                                             │
 Pipeline pod (Debezium Server, created by debezium-operator from the Pipeline you seed below)
       │  OTLP metrics every 5s
       ▼
 OpenTelemetry Collector  ──(Prometheus exporter :8889)──►  Prometheus (scrapes every 15s)
                                                                    ▲
                                                                    │ PromQL
 Stage UI  ──HTTP──►  Conductor API (/api/monitoring/*)  ──────────┘
```

Nothing is instant: the pipeline exports every 5s, Prometheus scrapes every 15s, and several panels average
over a multi-minute window. After you make a change in the database, give it ~30-45s before expecting it on a
chart.

Prerequisites
---
- [minikube](https://minikube.sigs.k8s.io/docs/)
- `kubectl`, `helm`
- `curl` and [`jq`](https://jqlang.org/) (used by the seed/verify scripts to talk to the Conductor API)

> **_NOTE:_** This example reuses the same minikube profile name (`debezium`) and `/etc/hosts` entry
> (`platform.debezium.io`) as [`../postgresql-kafka-example`](../postgresql-kafka-example). Run `./clean-up.sh`
> in that example first if you still have it running, or edit `CLUSTER` in [`env.sh`](./env.sh) to use a
> different profile.

All scripts source [`env.sh`](./env.sh) for shared configuration (namespaces, versions, hostnames). Edit it
once if you want different values, and every script picks them up.

Preparing the Environment
---
Just like the base example, we provision a local minikube cluster with an ingress controller and point
`platform.debezium.io` at it via `/etc/hosts`. The monitoring stack (Prometheus, Grafana, Alertmanager, two
operators) is heavier than the base example, so this cluster is sized up a bit (`--cpus=6 --memory=8192`).

```sh
./create-environment.sh
```

> **_NOTE:_** On macOS, keep the printed `sudo minikube tunnel -p debezium` command running in a separate
> terminal for the rest of this example — the ingress isn't reachable from the host without it.

Installing the Monitoring Stack
---
Before installing the platform itself, we install the two operators monitoring depends on:

- the **OpenTelemetry Operator**, which turns the `OpenTelemetryCollector` resource the platform chart creates
  into an actual running collector pod,
- **kube-prometheus-stack**, which provides the Prometheus Operator (consumes `ServiceMonitor` resources) and
  the Prometheus server the Conductor queries.

```sh
./setup-monitoring.sh
```

> **_NOTE:_** The script installs `kube-prometheus-stack` under that exact release name. Don't rename it —
> both `monitoring.prometheus.url` and the `ServiceMonitor` label used below are derived from it.

```sh
$ kubectl get pods -n monitoring

NAME                                                     READY   STATUS    RESTARTS   AGE
prometheus-kube-prometheus-stack-prometheus-0            2/2     Running   0          2m10s
kube-prometheus-stack-operator-6c6d4f8d9-abcde           1/1     Running   0          2m40s
kube-prometheus-stack-kube-state-metrics-7f9d8c6-fghij   1/1     Running   0          2m40s
alertmanager-kube-prometheus-stack-alertmanager-0        2/2     Running   0          2m10s
```

Preparing the Source Database and Destination Kafka cluster
---
Same role as in the base example: a PostgreSQL instance pre-seeded with the `inventory` schema as our source,
and a Strimzi-managed Kafka cluster as our destination.

```sh
./setup-infra.sh
```

```sh
$ kubectl get pods -n debezium-platform

NAME                                        READY   STATUS    RESTARTS   AGE
postgresql-85cc668d48-pjn58                 1/1     Running   0          90s
dbz-kafka-dual-role-0                       1/1     Running   0          80s
dbz-kafka-entity-operator-9f4d8fbc4-twq7j   2/2     Running   0          40s
strimzi-cluster-operator-7dc6fbcbf5-h28dl   1/1     Running   0          2m
```

Deploying Debezium Management Platform with monitoring enabled
---
This is the key difference from the base example. We install the same `debezium-platform` Helm chart, but with
an extra `monitoring` block in [`values.yaml`](./values.yaml) on top of the usual `domain`/`database`/`ingress`
settings:

```yaml
# values.yaml
monitoring:
  otel:
    enabled: true
    collector:
      # The OpenTelemetry Operator's default image (otelcol-k8s) does not include
      # the Prometheus exporter this chart configures, so the collector crash-loops.
      # Use the contrib distribution, which does include it.
      image: "ghcr.io/open-telemetry/opentelemetry-collector-releases/opentelemetry-collector-contrib:0.152.0"
      replicas: 1
  prometheus:
    # Prometheus runs in the `monitoring` namespace. The short service name only
    # resolves inside that namespace, and the conductor runs in `debezium-platform`.
    url: "http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090"
    serviceMonitor:
      enabled: true
      scrapeInterval: 15s
      labels:
        # kube-prometheus-stack scrapes ServiceMonitors labeled
        # `release: <helm release name>`. The chart default
        # (`prometheus: kube-prometheus`) matches nothing, so Prometheus never
        # scrapes the collector and every monitoring panel stays empty.
        release: kube-prometheus-stack
```

A few things worth knowing before you hit `helm install`:

- `monitoring.otel.enabled: true` tells the chart to create an `OpenTelemetryCollector` resource and wire the
  pipelines it later creates to send metrics to it.
- `monitoring.otel.collector.image` must point at a distribution that bundles the **Prometheus exporter**
  (`contrib`, not the operator's default `otelcol-k8s`), otherwise the collector pod crash-loops.
- `monitoring.prometheus.url` must be the **fully-qualified** service name, since Prometheus lives in a
  different namespace than the Conductor.
- `monitoring.prometheus.serviceMonitor.labels.release` must match the Prometheus Helm release name
  (`kube-prometheus-stack`) — this is what makes kube-prometheus-stack's Prometheus actually select and scrape
  the `ServiceMonitor` the chart creates. This is the single most common reason monitoring "looks installed"
  but every panel stays empty.

Install it:

```sh
./setup-platform.sh
```

The script waits for `conductor`, `stage` and the OTel collector deployment to roll out, then prints the
`ServiceMonitor` release label and the Conductor's configured Prometheus URL so you can confirm both line up
before moving on.

```sh
$ kubectl get pods -n debezium-platform

NAME                                                READY   STATUS    RESTARTS   AGE
conductor-7c48c54c5c-rmjw9                          1/1     Running   0          2m
stage-6c64f68df6-cfhjs                               1/1     Running   0          2m
debezium-operator-666f7b44d9-6tf4n                  1/1     Running   0          2m
postgres-69c4c64ff5-2tfmw                           1/1     Running   0          2m
debezium-platform-otel-collector-collector-xxxxx    1/1     Running   0          90s
```

After all pods are running, the Stage UI is reachable at `http://platform.debezium.io/`.

Creating the data pipeline
---
With the platform up, we need a Connection → Source → Destination → Pipeline, exactly like in the base example
— **minus the Transform**, since it isn't needed here and only adds another moving part. You have two options:

1. **Follow the UI, manually.** Use [`../postgresql-kafka-example`'s "Using the debezium-platform-stage(UI) for
   setting up our data pipeline"](../postgresql-kafka-example/README.md#using-the-debezium-platform-stageui-for-setting-up-our-data-pipeline)
   section step by step, with the same field values (this example's source/destination connection settings are
   identical: PostgreSQL host `postgresql`, Kafka bootstrap `dbz-kafka-kafka-bootstrap.debezium-platform:9092`,
   topic prefix `inventory`). Just stop before the **Transform** section and add the source and destination
   directly to the pipeline.
2. **Use the seed script (recommended for this example).** [`seed-pipeline.sh`](./seed-pipeline.sh) posts the
   same resources to the Conductor API directly — `postgres-connection`, `kafka-connection`, `test-source`,
   `test-destination` and `test-pipeline` (no transform) — and is safe to re-run (it reuses anything that
   already exists by name).

```sh
./seed-pipeline.sh
```

The script waits for the pipeline deployment to roll out and prints the direct link to its Monitoring tab,
e.g. `http://platform.debezium.io/pipeline/1/monitoring`.

```sh
$ kubectl get pods -n debezium-platform | grep test-pipeline
test-pipeline-7d9c8f6b5-k2n4p   1/1     Running   0   45s
```

Using the debezium-platform-stage(UI) — the Monitoring tab
---
This is where the UI comes back into focus. Open the link printed by `seed-pipeline.sh` (or **Pipelines** →
`test-pipeline` → **Monitoring** in the UI) and leave it open while you work through the rest of this example.

![Pipeline monitoring tab](./resources/monitoring-overview.png)

The tab is split into two sections, matching the `category` of each panel:

- **Streaming** — everything about ongoing change-data-capture, once the initial snapshot is done.
- **Snapshot** — the one-time initial copy of the captured tables. These stay flat at startup once the
  snapshot (which runs once, right after the pipeline is created) has finished — see
  [`testing_monitoring.md`](./testing_monitoring.md#11-make-the-snapshot-panels-light-up) for how to make the
  pipeline snapshot again so you can watch them move.

### Streaming panels

| Panel | What it shows |
| --- | --- |
| **Streaming Event Count Rate** | Rate of change events/sec, split by `create` / `update` / `delete`, plus a `total` line. The core "is my pipeline doing work" chart. |
| **Connection Status** | Whether the connector is currently connected to and listening on the source database (donut: connected/disconnected). |
| **Committed Transactions Rate** | Rate of committed source-database transactions being processed. |
| **Time Since Last Event** | Seconds since the last change event was processed. Climbs steadily when nothing is happening; snaps back to ~0 on the next change. The go-to "is my pipeline stuck?" panel. |
| **Source Lag** | Time between a change happening in the source database and Debezium processing it. |
| **Queue Utilization** | % of Debezium's internal event queue (by event count) currently in use. |
| **Queue Size Utilization** | % of the internal event queue's byte capacity currently in use. |
| **Erroneous Events Rate** | Rate of events that errored out during processing. Empty is good news here. |
| **Events Filtered Rate** | Rate of events excluded by the connector/SMT filter configuration. Empty if you aren't filtering anything (true in this example, since we skipped the transform). |

### Snapshot panels

| Panel | What it shows |
| --- | --- |
| **Snapshot Status** | Current snapshot lifecycle state: `running`, `completed`, `aborted`, or `skipped`. |
| **Snapshot Table Count** | Total number of tables included in the snapshot. |
| **Snapshot Table Progress** | Number of tables still remaining to be captured — counts down to 0 while a snapshot runs. |
| **Snapshot Duration** | Elapsed time of the current or most recent snapshot. |
| **Snapshot Event Count Rate** | Rate of snapshot (initial-load) change events/sec. |

You can always pull the authoritative, live list (14 panels by default) straight from the Conductor API instead
of relying on this table:

```sh
curl -s http://platform.debezium.io/api/monitoring/panels | jq -r '.panels[] | "\(.id)\t\(.category)\t\(.title)"'
```

Configuring the monitoring panels
---
The 14 panels above are not hard-coded into the UI — the Stage UI simply renders whatever the Conductor's
`/api/monitoring/panels` API returns, and the Conductor builds that list from a `panels.yml` file of
PromQL-backed panel definitions, loaded from the
[`debezium-platform`](https://github.com/debezium/debezium-platform) repository (see
`debezium-platform-conductor/src/main/resources/panels.yml`). You can **add your own panels, or override a
built-in one**, by pointing `monitoring.panels.additionalPanelsPath` at an extra YAML file with the same shape:

```yaml
panels:
  - id: my-custom-panel       # reuse a built-in id to override it, or pick a new one to add a panel
    title: "Custom Metric"
    description: "My custom monitoring panel"
    category: streaming       # streaming | snapshot -> which section it renders in
    query: 'rate(my_custom_metric_total{service_name="{{pipeline_id}}"}[5m])'
    unit: ops/s
    visualization:
      type: line              # area | line | donut-utilization
      suggestedStep: 15s
```

This directory ships a ready-made example — [`custom-panels.yml`](./custom-panels.yml) — which overrides
`streaming-event-count` with a faster 1-minute window and adds three brand-new panels built from metrics the
platform already exports but doesn't chart by default: JVM heap utilization, Kafka producer throughput, and JVM
GC pause time. [`values-custom-panels.yaml`](./values-custom-panels.yaml) mounts it via `conductor.extraVolumes`
/ `extraVolumeMounts` and sets `monitoring.panels.additionalPanelsPath`. Try it with:

```sh
./add-custom-panels.sh
```

```sh
curl -s http://platform.debezium.io/api/monitoring/panels | jq '.panels | length'
# Expect: 17 (14 built-in + 3 new)
```

> **_NOTE:_** Mount the panels **directory**, not a single file via `subPath` — `subPath` mounts never receive
> ConfigMap updates. Mounting the directory means the Conductor (which reloads the file roughly every
> `monitoring.panels.refreshInterval`, 30s by default) picks up edits to the ConfigMap live, no pod restart
> needed. See [`testing_monitoring.md`](./testing_monitoring.md#12-add-your-own-panel) for a step-by-step demo
> of editing a panel's title live and watching it change in the UI with zero restarts.
>
> For the full set of `monitoring.*` Helm values, see the
> [`debezium-platform` Helm chart README](https://github.com/debezium/debezium-platform/blob/main/helm/README.md#additional-monitoring-panels).
> <!-- TODO: expand this section with more detail / screenshots once reviewed against the latest chart docs -->

Testing the monitoring panels and pipeline liveliness
---
With the pipeline running and the Monitoring tab open, confirm the whole metrics path is actually wired end to
end, then generate some traffic so the charts move.

**Automated smoke test** — walks pipeline pod → ServiceMonitor label → collector → Prometheus → Conductor API
in one go and fails fast on the first broken hop:

```sh
./verify-monitoring.sh
```

**Generate live traffic** — inserts a steady drip of rows into the source database so **Streaming Event Count
Rate** climbs and holds instead of a single short-lived bump (defaults to 60 batches of 20 rows, one batch
every 5s — about 5 minutes):

```sh
./generate-traffic.sh            # defaults: 60 batches x 20 rows
./generate-traffic.sh 20 50      # or customize: batches, rows-per-batch
```

Watch the **Streaming** section of the Monitoring tab while it runs — **Streaming Event Count Rate** should
climb for the first couple of minutes and then plateau, and **Time Since Last Event** should stay pinned near
0 for as long as the loop keeps running.

For a much more thorough, copy-paste walkthrough — proving data flow hop by hop, making the Snapshot panels
light up on demand, safely flipping **Connection Status** to `disconnected` and back, stress-testing the queue
panels, and live-editing a custom panel — see [`testing_monitoring.md`](./testing_monitoring.md).

Cleanup
---
To remove the Kubernetes environment used in this example, execute the cleanup script:

```sh
./clean-up.sh
```

This deletes the `debezium` minikube profile (everything deployed in it goes with it). The script also prints
the command to drop the `/etc/hosts` entry it added, if you want to remove that too.
