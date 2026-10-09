# Running Debezium Platform Locally (README-aligned)

> Supersedes `.claude/Final_local_run.md` and `.claude/FRESH_START_CHECKLIST.md`.
> Tracks the official [`README.md`](../README.md) + [`helm/README.md`](../helm/README.md) as of **2026-10-06**.

The goal of this runbook is to be **the README, in order, with verification at every hop**. The README tells you
what to run; it doesn't tell you how to know it worked. Every step here has:

- **What / Why** — one line.
- **Run** — the command, copied from the README wherever possible.
- **Check** — how to confirm it actually worked. Don't continue until the Check passes.

Platform-specific bits (minikube on macOS) and the handful of places where the README is incomplete are
marked **`[LOCAL]`** and **`[DEVIATION]`** respectively, each with a reason.

---

## 0. Mental model — how the pieces fit

```
  PostgreSQL (source DB)                      Kafka (Strimzi, destination)
        │  CDC                                        ▲
        ▼                                             │
  Pipeline pod (Debezium Server — created by the operator from a Pipeline you define in the UI/API)
        │ OTLP gRPC :4317 / HTTP :4318
        ▼
  OpenTelemetry Collector  ──(Prometheus exporter :8889)──►  Prometheus (scrapes via ServiceMonitor)
                                                                     ▲
                                                                     │ PromQL
  Stage UI (React) ──HTTP──► Conductor API (/api/monitoring/*) ──────┘
```

Four operators do the heavy lifting:

| Operator | Watches | Creates | Installed in |
|---|---|---|---|
| **OpenTelemetry** | `OpenTelemetryCollector` | the collector pod | `opentelemetry-operator-system` |
| **Prometheus** (kube-prometheus-stack) | `ServiceMonitor` | scrape config for Prometheus | `monitoring` |
| **debezium-operator** (chart dependency) | `DebeziumServer` | the pipeline pod | `debezium-platform` |
| **Strimzi** | `Kafka` | the Kafka cluster pods | `debezium-platform` |

**Namespace layout** (this is the README's layout — three namespaces, not one):

```
ingress-nginx                   ingress controller (minikube addon)
opentelemetry-operator-system   OTel operator
monitoring                      Prometheus, Grafana, Alertmanager, kube-state-metrics
debezium-platform               conductor, stage, platform DB, debezium-operator,
                                OTel Collector, ServiceMonitor, source Postgres,
                                Strimzi, Kafka, pipeline pods
```

Because they're split, two things must be **fully-qualified or labelled** rather than relying on same-namespace
shortcuts — that's what Steps 7 and 9 are about.

---

## Deviations from the README — and why

The README is followed verbatim except for these. Each is a real gap, not a preference.

### `[DEVIATION 1]` — ServiceMonitor needs a `release` label (Step 9)

**Without this, monitoring silently produces nothing.** Everything looks healthy — collector `Running`,
ServiceMonitor created — but Prometheus never scrapes it and `/api/monitoring/*` returns empty series.

`kube-prometheus-stack` defaults to `serviceMonitorSelectorNilUsesHelmValues: true`, which renders:

```yaml
# kube-prometheus-stack/templates/prometheus/prometheus.yaml:196-199
serviceMonitorSelector:
  matchLabels:
    release: "kube-prometheus-stack"
```

But the platform chart stamps its ServiceMonitor with a different label:

```yaml
# helm/values.yaml  (monitoring.prometheus.serviceMonitor.labels)
labels:
  prometheus: kube-prometheus      # <- does not match `release: kube-prometheus-stack`
```

**Fix:** add the matching label in `examples/example.yaml`. This keeps *both* README helm commands byte-identical
— the fix lives entirely in the values file you already edit.

> Cross-namespace discovery is **not** a problem: the chart always emits `serviceMonitorNamespaceSelector: {}`
> (prometheus.yaml:206), which Prometheus Operator reads as "all namespaces". Only the label is wrong.

> **Upstream:** the chart's default `serviceMonitor.labels` is arguably the bug — `{prometheus: kube-prometheus}`
> matches no standard Prometheus install. Worth a PR.

### `[DEVIATION 2]` — Collector image must be set explicitly (Step 7)

The root `README.md:108-114` monitoring snippet shows only `otel.enabled` and `prometheus.url`. That's
incomplete — `helm/README.md:53-60` documents the rest:

> The OTel Collector must include the **Prometheus exporter**. The base `otelcol` distribution includes it,
> but when the OTel Operator is installed via Helm it defaults to the `otelcol-k8s` distribution which does **not**.

`monitoring.otel.collector.image` defaults to `""` → the operator picks `otelcol-k8s` → the collector pod
crash-loops on an unknown `prometheus` exporter in its config. The commented block already shipping in
`examples/example.yaml` has the right image, so just uncomment it.

> **Upstream:** the root README's snippet should either include `collector.image` or link to `helm/README.md`.

### `[DEVIATION 3]` — `prometheus.url` in `example.yaml` is wrong for the README's layout (Step 7)

The repo contradicts itself:

| File | Value |
|---|---|
| `README.md:113` | `http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090` |
| `examples/example.yaml:16` | `http://kube-prometheus-stack-prometheus:9090` |

The short name only resolves if Prometheus is in the *same* namespace as the conductor. The README puts it in
`monitoring`, so the FQDN is required. Use the README's value.

> **Upstream:** `examples/example.yaml` should carry the FQDN to match `README.md`.

### `[DEVIATION 4]` — Example transform breaks the pipeline (Step 13)

`examples/payloads/transform.json` uses `io.debezium.transforms.ExtractNewRecordState`. Observed on
**2026-08-17** with the server image pulled by `debezium-operator 3.7.0-alpha2`
(`quay.io/debezium/server:3.7.0.Alpha2`): the pipeline pod CrashLoops with `ClassNotFoundException`, because
that class isn't bundled in the image. `README.md:169` runs `seed.sh`, which POSTs that transform.

**Re-verify before working around it** — this may have been fixed since. Step 13 shows how to check, and gives a
transform-less pipeline as the fallback. The transform is a demo nicety; monitoring does not need it.

### `[LOCAL]` — minikube on macOS

The README is cluster-agnostic. Steps 1–3 and 10 add minikube specifics: resource sizing (the monitoring stack
is heavy), `minikube tunnel`, and the `/etc/hosts` IP. These aren't corrections to the README, just the
concrete form of "have a cluster with an ingress controller".

### Not a deviation: cert-manager is gone

Older notes installed cert-manager as an OTel Operator prerequisite. The README now uses Helm-generated
self-signed webhook certs instead (`admissionWebhooks.certManager.enabled=false`), so **cert-manager is not
needed**. If you already installed it, it just sits idle in its own namespace — harmless, no need to remove it.

> Trade-off, per `README.md:97`: the Helm-generated cert expires after 365 days with no auto-renewal. Fine for
> a local cluster you recreate often; use cert-manager in production.

---

## 1. `[LOCAL]` Clean slate

**What / Why:** Delete any previous cluster so stale CRDs and pods don't collide. Skip if you're starting fresh.

**Run:**
```bash
minikube delete -p debezium
```

**Check:**
```bash
minikube status -p debezium
# Expect: "Profile 'debezium' not found"
```

---

## 2. `[LOCAL]` Start minikube with an ingress controller

**What / Why:** The README requires "an ingress controller installed in your cluster"
(`helm/README.md:18-21`). On minikube that's the `ingress` addon. The resource bump is because
Prometheus + Grafana + Alertmanager + two operators is a lot for defaults.

**Run:**
```bash
minikube start -p debezium --cpus=6 --memory=8192 --addons ingress
```

**Check:**
```bash
minikube status -p debezium
# Expect: host / kubelet / apiserver all "Running"

kubectl get pods -n ingress-nginx
# Expect: ingress-nginx-controller-xxxx  1/1  Running   (takes ~30-60s)
```

> Using `kind` instead? Follow `README.md:38-70` — you need `extraPortMappings` for 80/443 and
> `ingress-nginx` with `controller.hostPort.enabled=true`. Everything from Step 4 on is identical.

---

## 3. `[LOCAL]` Start the tunnel (separate terminal, leave running)

**What / Why:** `README.md:82` — on macOS the minikube node IP isn't routable from the host, so the ingress is
unreachable without a tunnel. It needs sudo and must stay open for the whole session.

**Run (in a NEW terminal):**
```bash
sudo minikube tunnel -p debezium
```

**Check:** leave it running; you'll verify it in Step 10.

---

## 4. Map the domain into `/etc/hosts`

**What / Why:** `README.md:72-79`. The ingress serves `platform.debezium.io`; your machine has to resolve it.

**Run:**
```bash
export DEBEZIUM_PLATFORM_DOMAIN=platform.debezium.io
sudo ./examples/update_hosts.sh
```

**Check:**
```bash
grep platform.debezium.io /etc/hosts
# Expect: an entry exists
```

> `[LOCAL]` The script writes the IP from `kubectl cluster-info` (e.g. `192.168.49.2`). On macOS with the
> Docker driver that address is **not** reachable from the host — if `curl` fails in Step 10, edit the entry to
> `127.0.0.1 platform.debezium.io` (this is what `README.md:85` tells Windows users to do, and it's what the
> tunnel exposes).
>
> `[LOCAL]` The script's "update an existing entry" branch runs `sudo sed -i "/$HOSTNAME/d"`, which **fails on
> macOS** — BSD `sed -i` requires a backup-suffix argument. If you hit `invalid command code`, delete the stale
> line by hand: `sudo sed -i '' '/platform.debezium.io/d' /etc/hosts` and re-run.

---

## 5. Install the OpenTelemetry Operator

**What / Why:** `README.md:87-95`. This operator turns the `OpenTelemetryCollector` CR (which the platform chart
creates in Step 9) into a running collector pod. The webhook flags make Helm generate the self-signed TLS cert
the Kubernetes API server needs to call the operator's admission webhooks — this is what replaces cert-manager.

**Run:** *(verbatim from the README)*
```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm install opentelemetry-operator open-telemetry/opentelemetry-operator \
  -n opentelemetry-operator-system --create-namespace \
  --set admissionWebhooks.certManager.enabled=false \
  --set admissionWebhooks.autoGenerateCert.enabled=true
```

**Check:**
```bash
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=opentelemetry-operator \
  -n opentelemetry-operator-system --timeout=300s
kubectl get pods -n opentelemetry-operator-system
# Expect: opentelemetry-operator-xxxx  1/1  Running  (2/2 on some chart versions)
```

> Don't skip this Check. If the webhook cert is wrong the operator sits at `0/1` and you won't find out until
> Step 9, where the collector just never appears.

---

## 6. Install the Prometheus stack

**What / Why:** `README.md:99-104`. Provides the Prometheus Operator (which consumes `ServiceMonitor` CRs) plus
the Prometheus server that actually stores the metrics the conductor queries.

**Run:** *(verbatim from the README)*
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace
```

**Check (takes 2–3 min):**
```bash
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=prometheus-operator \
  -n monitoring --timeout=300s
kubectl get pods -n monitoring

# Expect (all Running):
#   prometheus-kube-prometheus-stack-prometheus-0   2/2
#   kube-prometheus-stack-operator-xxxx             1/1
#   kube-prometheus-stack-kube-state-metrics-xxxx   1/1

kubectl get svc -n monitoring kube-prometheus-stack-prometheus
# Expect: a service on 9090/TCP
```

> ⚠️ Keep the release name **`kube-prometheus-stack`** exactly. The service name is derived from it, and both
> `monitoring.prometheus.url` (Step 7) and the `release:` label (Deviation 1) hard-code it. Rename the release
> and you must update both.

**Confirm the selector you're about to satisfy** — this is what Deviation 1 is working around:
```bash
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.serviceMonitorSelector}'; echo
# Expect: {"matchLabels":{"release":"kube-prometheus-stack"}}
```

---

## 7. Configure monitoring in `examples/example.yaml`

**What / Why:** `README.md:106-114`. The shipped `examples/example.yaml` has the monitoring block commented out.
Uncomment it and apply Deviations 1–3.

**Run:** make `examples/example.yaml` read exactly:

```yaml
domain:
  name: platform.debezium.io
database:
  enabled: true
ingress:
  className: nginx

monitoring:
  otel:
    enabled: true
    collector:
      # DEVIATION 2: operator defaults to the `otelcol-k8s` distro, which has no
      # Prometheus exporter. Must be base or contrib. See helm/README.md:53-60.
      image: "ghcr.io/open-telemetry/opentelemetry-collector-releases/opentelemetry-collector-contrib:0.152.0"
      replicas: 1
  prometheus:
    # DEVIATION 3: FQDN, not the short name — Prometheus is in the `monitoring`
    # namespace, the conductor is in `debezium-platform`. Matches README.md:113.
    url: "http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090"
    serviceMonitor:
      enabled: true
      scrapeInterval: 15s
      labels:
        # DEVIATION 1: must match kube-prometheus-stack's serviceMonitorSelector,
        # which is `release: <helm release name>`. The chart's default label
        # (`prometheus: kube-prometheus`) matches nothing.
        release: kube-prometheus-stack
```

Everything else (receiver ports 4317/4318, batch processor, Prometheus exporter on 8889,
`jmxIntervalMs: 1000`, `metricExportIntervalMs: 5000`) has working defaults in `helm/values.yaml:112-146`
— don't restate them.

**Check:**
```bash
grep -c "^#" examples/example.yaml   # the monitoring block should no longer be commented
kubectl get svc -n monitoring kube-prometheus-stack-prometheus -o name
# The URL above must resolve to THIS service.
```

---

## 8. Create the platform namespace

**What / Why:** `README.md:116-120`.

**Run:**
```bash
kubectl create ns debezium-platform
```

**Check:**
```bash
kubectl get ns debezium-platform
# Expect: STATUS Active
```

> This runbook passes `-n debezium-platform` explicitly everywhere (as the README does), so you do **not** need
> to change your kubectl context.

---

## 9. Install the Debezium Platform

**What / Why:** `README.md:122-129`. `helm dependency build` pulls the `debezium-operator` and `database`
subcharts per `Chart.lock`. The install brings up Conductor (API), Stage (UI), the Debezium operator, the
embedded Postgres, **and** — because `monitoring.otel.enabled: true` — the `OpenTelemetryCollector` CR and its
ServiceMonitor.

**Run:** *(verbatim from the README)*
```bash
cd helm && \
helm dependency build && \
helm install debezium-platform . -n debezium-platform -f ../examples/example.yaml && \
cd ..
```

**Check:**
```bash
helm list -n debezium-platform
# Expect: debezium-platform  STATUS deployed

kubectl get pods -n debezium-platform
# Expect (all eventually 1/1 Running):
#   conductor-xxxx
#   stage-xxxx
#   debezium-operator-xxxx
#   postgres-xxxx                                     (platform's own DB)
#   debezium-platform-otel-collector-collector-xxxx   (the monitoring pod)

kubectl get otelcol -n debezium-platform
# Expect: debezium-platform-otel-collector

kubectl get servicemonitor -n debezium-platform
# Expect: debezium-platform-otel-collector
```

**Verify Deviation 1 actually took** — one command, saves a long debug later:
```bash
kubectl get servicemonitor debezium-platform-otel-collector -n debezium-platform \
  -o jsonpath='{.metadata.labels.release}'; echo
# Expect: kube-prometheus-stack
# Empty output = the label didn't apply; Prometheus will never scrape it.
```

**Verify the conductor got the right Prometheus URL:**
```bash
kubectl get deploy conductor -n debezium-platform \
  -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="MONITORING_PROMETHEUS_URL")].value}'; echo
# Expect: http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090
```

> `monitoring.prometheus.url` is marked `required` in `helm/templates/_helpers.tpl:126` — if you forgot it, the
> install fails outright rather than silently misbehaving.

> **Operator version:** `helm/Chart.yaml:13` already pins `debezium-operator: 3.7.0-alpha2`, the first version
> whose CRD declares all three `openTelemetry.collector` fields the nightly conductor emits. Nothing to edit.
> Older notes telling you to bump this are stale. See Troubleshooting if you ever *change* it.

---

## 10. Confirm the UI loads

**What / Why:** `README.md:131` — proves ingress + tunnel + hosts entry all line up.

**Run / Check:**
```bash
kubectl get ingress -n debezium-platform
# Expect: an ingress with host platform.debezium.io

curl -sS -o /dev/null -w "%{http_code}\n" http://platform.debezium.io/
# Expect: 200
```

Then open **http://platform.debezium.io/** — the Stage UI should load.

> `curl` returns `000` or hangs? In order: (1) is `sudo minikube tunnel -p debezium` still running (Step 3)?
> (2) is the `/etc/hosts` entry `127.0.0.1` (Step 4 note)? (3) `kubectl get pods -n ingress-nginx`.

---

## 11. Deploy the source database

**What / Why:** `README.md:144-148`. The Postgres the pipeline captures changes *from* — distinct from the
platform's own Postgres. Ships pre-seeded with the `inventory` schema.

**Run:** *(verbatim from the README)*
```bash
kubectl create -n debezium-platform -f examples/k8s/database/001_postgresql.yml
```

**Check:**
```bash
kubectl get pods -n debezium-platform | grep postgresql
# Expect: postgresql-xxxx  1/1  Running

kubectl exec -n debezium-platform deployment/postgresql -- \
  psql -U debezium -d debezium -c "\dt inventory.*"
# Expect: inventory tables (products, customers, orders, ...)
```

---

## 12. Install Strimzi and create the Kafka cluster

**What / Why:** `README.md:150-162`. The pipeline's destination. Strimzi's operator provisions Kafka from the
`Kafka` CR.

**Run:** *(verbatim from the README — note version `0.45.1`)*
```bash
helm repo add strimzi https://strimzi.io/charts/ && \
helm repo update strimzi && \
helm install strimzi-operator strimzi/strimzi-kafka-operator \
  --version 0.45.1 --namespace debezium-platform

kubectl create -n debezium-platform -f examples/k8s/kafka/001_kafka.yml
```

**Check (Kafka takes 3–5 min):**
```bash
kubectl get pods -n debezium-platform | grep strimzi-cluster-operator
# Expect: strimzi-cluster-operator-xxxx  1/1  Running

kubectl wait kafka/dbz-kafka --for=condition=Ready --timeout=600s -n debezium-platform
kubectl get kafka -n debezium-platform
# Expect: dbz-kafka  READY True
```

---

## 13. Create a test pipeline

**What / Why:** `README.md:164-170`. Conductor turns these payloads into a `DebeziumServer` CR; the
debezium-operator turns that into the pipeline pod, which starts emitting OTLP metrics to the collector.

**Run:** *(the README's way — requires [HTTPie](https://httpie.io/))*
```bash
./examples/seed.sh platform.debezium.io 80 examples/payloads/
```

**Check:**
```bash
sleep 10
kubectl get debeziumserver -n debezium-platform
# Expect: one DebeziumServer

kubectl get pods -n debezium-platform | grep -i pipeline
# Expect: the pipeline pod eventually 1/1 Running
```

### `[DEVIATION 4]` If the pipeline pod CrashLoops

```bash
kubectl logs -n debezium-platform -l debezium.io/kind=DebeziumServer --tail=50 | grep -i "ClassNotFound"
```

If you see `ClassNotFoundException: io.debezium.transforms.ExtractNewRecordState`, the server image doesn't
bundle that SMT. Recreate without the transform — monitoring doesn't need it:

```bash
BASE=http://platform.debezium.io

# clean up the failed pipeline, keep the rest
curl -s $BASE/api/pipelines | jq -r '.[].id' | xargs -I{} curl -X DELETE $BASE/api/pipelines/{}

cat > /tmp/pipeline-notransform.json <<'JSON'
{
  "name": "test-pipeline",
  "description": "It goes from here to there!",
  "source": { "id": 1, "name": "test-source" },
  "destination": { "id": 1, "name": "test-destination" },
  "transforms": [],
  "logLevel": "INFO"
}
JSON
curl -X POST $BASE/api/pipelines -H "Content-Type: application/json" -d @/tmp/pipeline-notransform.json
```

> No HTTPie? The `curl` equivalent of `seed.sh`:
> ```bash
> BASE=http://platform.debezium.io
> for f in connection-db connection-kafka; do
>   curl -X POST $BASE/api/connections -H "Content-Type: application/json" -d @examples/payloads/$f.json
> done
> curl -X POST $BASE/api/sources      -H "Content-Type: application/json" -d @examples/payloads/source.json
> curl -X POST $BASE/api/destinations -H "Content-Type: application/json" -d @examples/payloads/destination.json
> curl -X POST $BASE/api/transforms   -H "Content-Type: application/json" -d @examples/payloads/transform.json
> curl -X POST $BASE/api/pipelines    -H "Content-Type: application/json" -d @examples/payloads/pipeline.json
> ```

**Confirm the pipeline was told to emit OTel metrics** — this is the key monitoring wiring:
```bash
kubectl get debeziumserver -n debezium-platform -o yaml | grep -A6 "metrics:"
# Expect:
#   metrics:
#     openTelemetry:
#       collector:
#         endpoint: http://debezium-platform-otel-collector-collector...:4318
```
If `openTelemetry` is absent, see Troubleshooting → "Pipeline has no openTelemetry".

---

## 14. Verify the monitoring data path

Four hops, in order. Each confirms the next arrow in the Step 0 diagram.

### 14a. Pipeline → Collector
```bash
kubectl port-forward -n debezium-platform \
  svc/debezium-platform-otel-collector-collector 8889:8889 &
sleep 3
curl -s http://localhost:8889/metrics | grep debezium | head
# Expect: several debezium_* metric lines
pkill -f "port-forward.*8889"
```
Nothing? The collector isn't receiving OTLP — check Deviation 2 and the pipeline pod's logs.

### 14b. Collector → Prometheus
This is the hop Deviation 1 exists for.
```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090 &
sleep 3
curl -s 'http://localhost:9090/api/v1/targets' \
  | jq -r '.data.activeTargets[] | select(.labels.job|test("otel")) | "\(.labels.job) \(.health)"'
# Expect: a line ending in "up". NO OUTPUT = the ServiceMonitor isn't selected (Deviation 1).
pkill -f "port-forward.*9090"
```

### 14c. Conductor → Prometheus
```bash
curl -s http://platform.debezium.io/api/monitoring/panels | jq -r '.[].panelId'
# Expect: ~15 panel IDs (streaming-event-count, connection-status, snapshot-*, ...)
```
This only proves the panel registry loads. 14d is the one that proves PromQL works.

### 14d. Full round-trip: UI-style query → Conductor → PromQL → data
```bash
PIPELINE_ID=$(curl -s http://platform.debezium.io/api/pipelines | jq -r '.[0].name')
START=$(date -u -v-5M +'%Y-%m-%dT%H:%M:%S.000Z')   # macOS date syntax
END=$(date -u +'%Y-%m-%dT%H:%M:%S.000Z')

curl -sG "http://platform.debezium.io/api/monitoring/panels/connection-status/query" \
  --data-urlencode "pipeline_id=${PIPELINE_ID}" \
  --data-urlencode "start=${START}" \
  --data-urlencode "end=${END}" \
  --data-urlencode "step=15s" | jq
# Expect: a series with datapoints (connection-status 1.0 = connected)
```
> `pipeline_id` is the pipeline **name**, not the numeric id.

### 14e. Live CDC events
```bash
kubectl exec -n debezium-platform deployment/postgresql -- \
  psql -U debezium -d debezium -c \
  "INSERT INTO inventory.products (name, description, weight) VALUES ('Test', 'monitoring test', 1.5);"
```
Wait ~30s, re-run 14d against panel `streaming-event-count` — you should see a non-zero create rate. Or watch
the pipeline's monitoring tab in the UI.

---

## 15. Final health snapshot

```bash
for ns in ingress-nginx opentelemetry-operator-system monitoring debezium-platform; do
  echo "--- $ns"; kubectl get pods -n $ns
done
```

Expect all `Running`; Prometheus is `2/2`, everything else `1/1`:

```
ingress-nginx                   ingress-nginx-controller-xxxx
opentelemetry-operator-system   opentelemetry-operator-xxxx
monitoring                      prometheus-kube-prometheus-stack-prometheus-0        2/2
                                kube-prometheus-stack-operator-xxxx
                                kube-prometheus-stack-kube-state-metrics-xxxx
                                kube-prometheus-stack-grafana-xxxx
                                alertmanager-kube-prometheus-stack-alertmanager-0     2/2
debezium-platform               conductor-xxxx
                                stage-xxxx
                                debezium-operator-xxxx
                                postgres-xxxx                        (platform DB)
                                postgresql-xxxx                      (source DB)
                                debezium-platform-otel-collector-collector-xxxx
                                strimzi-cluster-operator-xxxx
                                dbz-kafka-...  (x2)
                                <your-pipeline>-xxxx
```

---

## Troubleshooting

### Prometheus target missing / monitoring API returns empty
Walk the hops in Step 14 and stop at the first failure. Most often it's 14b. Diagnose:
```bash
# What does Prometheus require?
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.serviceMonitorSelector}'; echo
# What does the ServiceMonitor have?
kubectl get servicemonitor debezium-platform-otel-collector -n debezium-platform -o jsonpath='{.metadata.labels}'; echo
# These must intersect.
```
Fix by setting `monitoring.prometheus.serviceMonitor.labels` to match (Deviation 1) and `helm upgrade`.

**Blunt fallback** if the labels still won't line up (e.g. a differently-named Prometheus release) — make
Prometheus accept every ServiceMonitor regardless of label:
```bash
helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false
```

### OTel collector pod not starting
```bash
kubectl describe otelcol debezium-platform-otel-collector -n debezium-platform
kubectl logs -n debezium-platform -l app.kubernetes.io/component=opentelemetry-collector
```
Most common cause: the `otelcol-k8s` image, which has no Prometheus exporter (Deviation 2). Confirm:
```bash
kubectl get otelcol debezium-platform-otel-collector -n debezium-platform -o jsonpath='{.spec.image}'; echo
# Expect: ...opentelemetry-collector-contrib:0.152.0   (NOT otelcol-k8s)
```

### Collector CR exists but no pod
The OTel Operator's webhook isn't working:
```bash
kubectl get pods -n opentelemetry-operator-system
kubectl logs -n opentelemetry-operator-system -l app.kubernetes.io/name=opentelemetry-operator
```

### Conductor can't reach Prometheus
```bash
kubectl get deploy conductor -n debezium-platform \
  -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="MONITORING_PROMETHEUS_URL")].value}'; echo
kubectl run -it --rm dnstest --image=busybox --restart=Never -n debezium-platform -- \
  nslookup kube-prometheus-stack-prometheus.monitoring.svc.cluster.local
```

### Pipeline has no `openTelemetry` in its DebeziumServer
The operator's CRD doesn't declare the field:
```bash
kubectl get crd debeziumservers.debezium.io -o jsonpath='{.spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.runtime.properties.metrics.properties.openTelemetry.properties.collector.properties}' | jq 'keys'
# Expect: ["endpoint","jmxIntervalMs","metricExportIntervalMs"]
```
Shouldn't happen on a fresh install — `Chart.yaml` pins `3.7.0-alpha2` and Helm applies subchart `crds/` on
first install. It *does* happen if you **changed** the operator version on an existing release, because
**`helm upgrade` never touches `crds/`**. Apply the CRD by hand, then upgrade with `-f` (not
`--reuse-values`, which would freeze the subchart's image tag too):
```bash
cd helm
tar xzf charts/debezium-operator-3.7.0-alpha2.tgz -C /tmp \
  debezium-operator/crds/debeziumservers.debezium.io-v1.yml
kubectl apply -f /tmp/debezium-operator/crds/debeziumservers.debezium.io-v1.yml
helm upgrade debezium-platform . -n debezium-platform -f ../examples/example.yaml
cd ..
```

### Pipeline won't deploy at all
```bash
kubectl logs -n debezium-platform deployment/conductor --tail=100 | grep -i "error\|pipeline"
```

### UI unreachable
See the note under Step 10.

---

## Teardown

```bash
helm uninstall debezium-platform strimzi-operator -n debezium-platform
helm uninstall kube-prometheus-stack -n monitoring
helm uninstall opentelemetry-operator -n opentelemetry-operator-system
kubectl delete -n debezium-platform -f examples/k8s/kafka/001_kafka.yml
kubectl delete -n debezium-platform -f examples/k8s/database/001_postgresql.yml
kubectl delete ns debezium-platform monitoring opentelemetry-operator-system

# Full reset:
minikube delete -p debezium
```

---

## Quick sequence (once you trust the checks)

```bash
minikube start -p debezium --cpus=6 --memory=8192 --addons ingress
# (separate terminal) sudo minikube tunnel -p debezium

export DEBEZIUM_PLATFORM_DOMAIN=platform.debezium.io
sudo ./examples/update_hosts.sh

helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm install opentelemetry-operator open-telemetry/opentelemetry-operator \
  -n opentelemetry-operator-system --create-namespace \
  --set admissionWebhooks.certManager.enabled=false \
  --set admissionWebhooks.autoGenerateCert.enabled=true

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace

# >>> edit examples/example.yaml per Step 7 (Deviations 1-3) before continuing <<<

kubectl create ns debezium-platform
cd helm && helm dependency build && \
  helm install debezium-platform . -n debezium-platform -f ../examples/example.yaml && cd ..

kubectl create -n debezium-platform -f examples/k8s/database/001_postgresql.yml
helm repo add strimzi https://strimzi.io/charts/ && helm repo update strimzi
helm install strimzi-operator strimzi/strimzi-kafka-operator --version 0.45.1 -n debezium-platform
kubectl create -n debezium-platform -f examples/k8s/kafka/001_kafka.yml
kubectl wait kafka/dbz-kafka --for=condition=Ready --timeout=600s -n debezium-platform

./examples/seed.sh platform.debezium.io 80 examples/payloads/   # see Deviation 4
```

---

## Upstream fixes this runbook implies

If you want the README to be followable without deviations:

1. `examples/example.yaml:16` — change `prometheus.url` to the `monitoring` FQDN so it matches `README.md:113`.
2. `helm/values.yaml` — change the default `monitoring.prometheus.serviceMonitor.labels` from
   `{prometheus: kube-prometheus}` to `{release: kube-prometheus-stack}`, which is what
   `kube-prometheus-stack` actually selects on.
3. `README.md:108-114` — include `monitoring.otel.collector.image` in the snippet, or link to the
   `helm/README.md` note explaining why the default image doesn't work.
4. `helm/README.md:144` — `monitoring.prometheus.serviceMonitor.scrapeInterval` is listed twice in the
   config table (lines 142 and 144).
5. `examples/payloads/transform.json` — verify `ExtractNewRecordState` exists in the server image the pinned
   operator pulls; `seed.sh` is the README's happy path and currently may break it (Deviation 4).
