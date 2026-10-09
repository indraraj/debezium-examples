# Testing the Monitoring Panels (simple version)

> Companion to [`running_locally.md`](running_locally.md). That runbook gets the platform **up**.
> This one makes the monitoring panels **move**, so you can see them working.
>
> A longer, more technical version lives in [`monitoring_testing.md`](monitoring_testing.md).
> This file is the one to follow if you just want to copy-paste commands and watch the UI react.

Same format as `running_locally.md`:

- **What / Why** — what this step does, in plain words.
- **Run** — the command. Plain commands, no scripts. Some you fire more than once; that's fine and intentional.
- **Look** — where in the UI the effect shows up.
- **Check** — how to confirm from the terminal. Don't move on until the Check passes.

---

## 0. Mental model — where does a number on a panel come from?

You write a row into Postgres. Six hops later it's a dot on a chart:

```
 1. You INSERT a row into Postgres (the source DB)
         │
 2. The pipeline pod (Debezium Server) reads it and counts it
         │  pushes counters every 5s (OTLP)
         ▼
 3. OpenTelemetry Collector   ──  holds the numbers, exposes them on :8889
         │  Prometheus pulls every 15s
         ▼
 4. Prometheus               ──  stores the history
         │  Conductor asks it with PromQL
         ▼
 5. Conductor  /api/monitoring/panels/<id>/query
         │
 6. Stage UI  →  Pipeline → Monitoring tab  →  the chart you see
```

**Two things to take away from this diagram:**

1. **Nothing is instant.** The pipeline exports every 5s, Prometheus scrapes every 15s, and most panels
   average over the last 5 minutes. After you insert rows, **wait ~45 seconds** before expecting anything.
2. **A panel is just a saved PromQL query.** `panels.yml` in the conductor holds 14 of them. The UI draws
   whatever the API returns — it has no metrics logic of its own. That's why you can add your own (Step 12).

**One phrase you'll see everywhere:** `pipeline_id`. It is simply the **pipeline's name** — `test-pipeline`.
Not a number, not the pod name. The platform turns it into a `service_name="test-pipeline"` filter in PromQL.

---

## 1. Open the UI and park it on the Monitoring tab

**What / Why:** Keep this open on a second screen for the whole runbook. Every exercise below says what
should change here. Reading the chart is the point; the terminal commands only exist to prove it.

**Run:** open in a browser:

```
http://platform.debezium.io/pipeline/1/monitoring
```

**Look:** two sections — **Streaming** (9 panels) and **Snapshot** (5 panels).

**Check:** the **Connection Status** donut should read **connected**. If it doesn't, stop and do Step 3.

---

## 2. Set up three variables

**What / Why:** Every command below reuses these. Setting them once means you can copy-paste the rest
without editing anything. Do this in **every new terminal tab** you open.

**Run:**
```bash
export NS=debezium-platform
export PIPELINE=test-pipeline
export HOST=http://platform.debezium.io
```

**Check:**
```bash
echo "$NS / $PIPELINE / $HOST"
# Expect: debezium-platform / test-pipeline / http://platform.debezium.io
```

---

## 3. Health check — is data flowing at all?

**What / Why:** Four commands, one per hop in the diagram above. If a panel is empty, it is almost always
one of these four — not the panel. Run them in order and stop at the first failure.

### 3a. Is the pipeline pod healthy?

```bash
kubectl get pods -n $NS | grep $PIPELINE
```
```
# Expect: test-pipeline-xxxxx   1/1   Running
# 0/1 or climbing RESTARTS = the pipeline is broken; no metrics will exist.
```

### 3b. Does the ServiceMonitor have the `release` label?

**Why this one matters:** without this label Prometheus silently ignores the collector. Everything looks
healthy, no error appears anywhere, and **every panel is just empty**. This is the single most common cause
of "monitoring doesn't work". (Deviation 1 in `running_locally.md`.)

```bash
kubectl get servicemonitor debezium-platform-otel-collector -n $NS -o jsonpath='{.metadata.labels.release}'; echo
```
```
# Expect: kube-prometheus-stack
# Empty = Prometheus is not scraping. Fix Deviation 1 and helm upgrade.
```

### 3c. Is the collector producing Debezium numbers?

```bash
kubectl port-forward -n $NS svc/debezium-platform-otel-collector-collector 18889:8889
```

Leave that running, and **in a second terminal**:

```bash
curl -s http://localhost:18889/metrics | grep -c '^debezium_'
```
```
# Expect: a few hundred (e.g. 300+)
# 0 = the pipeline isn't exporting. Go back to 3a.
```

Then `Ctrl+C` the port-forward.

### 3d. Is Prometheus actually scraping the collector?

**Why:** This is the hop everyone forgets. 3c proves the numbers *exist*; this proves someone is *collecting*
them.

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 19090:9090
```

Second terminal:

```bash
curl -s --get http://localhost:19090/api/v1/query \
  --data-urlencode 'query=up{job="debezium-platform-otel-collector-collector"}' | jq '.data.result[0].value[1]'
```
```
# Expect: "1"
# "0" or null/empty = Prometheus can see the target but can't scrape it, or 3b failed.
```

Keep this port-forward around — Step 11 uses it again. Otherwise `Ctrl+C`.

---

## 4. See the list of panels

**What / Why:** This is the API the UI calls to decide which charts to draw. Every panel id you use later
comes from here.

**Run:**
```bash
curl -s "$HOST/api/monitoring/panels" | jq -r '.panels[] | "\(.id)\t\(.category)\t\(.visualization.type)\t\(.title)"' | column -t -s$'\t'
```

**Check:**
```
# Expect 14 rows (or 17 if you've done Step 12), including:
#   streaming-event-count    streaming   area               Streaming Event Count Rate
#   connection-status        streaming   donut-utilization  Connection Status
#   snapshot-status          snapshot    donut-utilization  Snapshot Status
```

```bash
curl -s "$HOST/api/monitoring/panels" | jq '.panels | length'
# Expect: 14
```

`category` decides **which UI section** the chart lands in: `streaming` or `snapshot`. That's all it does.

---

## 5. Read one panel from the terminal

**What / Why:** The query API needs a **time window** — a start and an end. You'll build that window over
and over, so get comfortable with these two lines. `-v-15M` means "15 minutes ago".

**Run:**
```bash
START=$(date -u -v-15M '+%Y-%m-%dT%H:%M:%SZ')
END=$(date -u '+%Y-%m-%dT%H:%M:%SZ')
echo "$START -> $END"
```

Now ask for one panel:

```bash
curl -s "$HOST/api/monitoring/panels/streaming-event-count/query?pipeline_id=$PIPELINE&start=$START&end=$END&step=15s" \
  | jq -r '.series[] | "\(.labels.debezium_event_type)  last=\(.datapoints[-1][1])"'
```

**Check:**
```
# Expect five lines: create / update / delete / read / total
# create  last=0
# delete  last=0
# read    last=0
# total   last=0
# update  last=0
```

All zeros is **correct** right now — nothing has changed in the database yet. Step 6 fixes that.

> **Re-run the two `date` lines every time** before a query. If you don't, `END` stays in the past and you'll
> keep seeing the same stale numbers and think nothing is working.

**To see the raw shape once** (useful when you're wiring the UI):

```bash
curl -s "$HOST/api/monitoring/panels/connection-status/query?pipeline_id=$PIPELINE&start=$START&end=$END&step=15s" | jq '.'
```
```
# Note the field names: series[] -> { labels, datapoints }
# datapoints is a list of [epoch_seconds, value]  -- lowercase "datapoints", not "values"
```

---

## 6. Make the event chart move — insert, update, delete

**What / Why:** The most basic proof that monitoring works. Each INSERT becomes a `create` event, each UPDATE
an `update`, each DELETE a `delete`. The chart splits them by colour.

**Run:**
```bash
kubectl exec -n $NS deploy/postgresql -- psql -U debezium -d debezium -c \
  "INSERT INTO inventory.products (name, description, weight) SELECT 'demo-'||g, 'row '||g, g FROM generate_series(1,300) g;"
```
```bash
kubectl exec -n $NS deploy/postgresql -- psql -U debezium -d debezium -c \
  "UPDATE inventory.products SET description = description || ' [edited]' WHERE name LIKE 'demo-%';"
```
```bash
kubectl exec -n $NS deploy/postgresql -- psql -U debezium -d debezium -c \
  "DELETE FROM inventory.products WHERE name LIKE 'demo-%';"
```

Now **wait 45 seconds.**

**Look:** Monitoring tab → **Streaming Event Count Rate** lifts off zero, with separate bands for create /
update / delete. **Committed Transactions Rate** also lifts (3 commits).

**Check:**
```bash
START=$(date -u -v-5M '+%Y-%m-%dT%H:%M:%SZ'); END=$(date -u '+%Y-%m-%dT%H:%M:%SZ')
curl -s "$HOST/api/monitoring/panels/streaming-event-count/query?pipeline_id=$PIPELINE&start=$START&end=$END&step=15s" \
  | jq -r '.series[] | "\(.labels.debezium_event_type)  max=\(.datapoints | map(.[1]) | max)"'
```
```
# Expect something like:
#   create  max=1.01
#   update  max=1.01
#   delete  max=0.2
#   total   max=2.22
```

> **Why is it only ~1 event/s when I inserted 300 rows?**
> The panel shows a **rate averaged over the last 5 minutes**, not a total. 300 events spread over 300
> seconds ≈ 1/s. This is normal, and it's why a single burst looks small. Step 7 is the better demo.

---

## 7. Get a chart that moves continuously

**What / Why:** A one-shot burst gives you one bump that slowly decays — bad for a live demo. A steady drip
gives you a chart that visibly climbs and holds while you talk over it.

**Run** — this inserts 20 rows every 5 seconds for 5 minutes. Leave it running and watch the UI:

```bash
for i in $(seq 1 60); do
  kubectl exec -n $NS deploy/postgresql -- psql -U debezium -d debezium -q -c \
    "INSERT INTO inventory.products (name,description,weight) SELECT 'tick-$i-'||g,'tick',g FROM generate_series(1,20) g;"
  echo "batch $i sent"
  sleep 5
done
```

**Look:** **Streaming Event Count Rate** climbs for the first ~2 minutes, then flattens at roughly 4 events/s
and stays there. **Time Since Last Event** pins near 0. That plateau is your demo money-shot.

**Check:** while the loop is running, in another terminal:
```bash
START=$(date -u -v-5M '+%Y-%m-%dT%H:%M:%SZ'); END=$(date -u '+%Y-%m-%dT%H:%M:%SZ')
curl -s "$HOST/api/monitoring/panels/streaming-event-count/query?pipeline_id=$PIPELINE&start=$START&end=$END&step=15s" \
  | jq -r '.series[] | select(.labels.debezium_event_type=="total") | .datapoints | map(.[1]) | max'
```
```
# Expect: a number clearly above 0 and rising between runs (e.g. 1.4 -> 2.8 -> 4.0)
```

---

## 8. Show a stalled pipeline — "Time Since Last Event"

**What / Why:** The opposite demo, and the one operators actually care about: *is my pipeline stuck?*
This panel counts seconds since the last change event. Stop writing and it climbs in a straight line.

**Run:** stop the Step 7 loop (`Ctrl+C`) and do nothing for 3 minutes.

**Look:** **Time Since Last Event** turns into a steadily rising diagonal line. Then run any INSERT from
Step 6 and watch it **drop straight back to ~0** — that snap-back is the thing to demo.

**Check** (run a few times, ~30s apart — the number must grow):
```bash
START=$(date -u -v-10M '+%Y-%m-%dT%H:%M:%SZ'); END=$(date -u '+%Y-%m-%dT%H:%M:%SZ')
curl -s "$HOST/api/monitoring/panels/time-since-last-event/query?pipeline_id=$PIPELINE&start=$START&end=$END&step=15s" \
  | jq -r '[.series[].datapoints[-1][1]] | max'
```
```
# Expect: a growing number of seconds, e.g. 31 -> 64 -> 95
```

> **Why `max` over all series and not `.series[0]`?** If the pipeline has restarted in the last few minutes
> you'll get **two** series — the dead pod's and the live one — and `.series[0]` often picks the dead one,
> which reports a frozen value like `-0.001`. Taking the max always gives you the real one. Same trap as the
> note in Step 10.

---

## 9. Stress the pipeline — queue and throughput panels

**What / Why:** The queue panels show how full Debezium's internal buffer is. On a quiet local setup they sit
at 0 because the connector drains faster than you can write. You need a big burst of **wide** rows to make
them twitch. `repeat('x',2000)` makes each row ~2KB, which is what moves the *bytes* panel.

**Run:**
```bash
kubectl exec -n $NS deploy/postgresql -- psql -U debezium -d debezium -c \
  "INSERT INTO inventory.products (name,description,weight) SELECT 'burst-'||g, repeat('x',2000), g FROM generate_series(1,5000) g;"
```

Wait 40 seconds.

**Look:** **Queue Utilization** and **Queue Size Utilization** donuts blip above 0%. **Source Lag** bumps.

**Check:**
```bash
START=$(date -u -v-5M '+%Y-%m-%dT%H:%M:%SZ'); END=$(date -u '+%Y-%m-%dT%H:%M:%SZ')
for p in queue-utilization queue-size-utilization source-lag; do
  printf '%-24s ' $p
  curl -s "$HOST/api/monitoring/panels/$p/query?pipeline_id=$PIPELINE&start=$START&end=$END&step=15s" \
    | jq -r '[.series[].datapoints[][1]] | max'
done
```
```
# Expect: three numbers. Small values (even 0) are normal on a laptop —
# the point is that the panels respond at all, not that they go red.
```

**Clean up the burst rows** so later steps aren't slowed down:
```bash
kubectl exec -n $NS deploy/postgresql -- psql -U debezium -d debezium -c \
  "DELETE FROM inventory.products WHERE name LIKE 'burst-%' OR name LIKE 'tick-%';"
```

---

## 10. Break the connection — "Connection Status"

**What / Why:** Shows the donut flipping from **connected** to **disconnected** and back. This is the best
"something is wrong" visual in the whole dashboard.

> ### ⚠️ Do NOT use `kubectl scale deploy/postgresql --replicas=0`
> The source Postgres in this setup has **no persistent volume**. Scaling it to zero **deletes the whole
> database**. I did this by accident while writing this file: `inventory.products` went from 749 rows back to
> 9, the replication slot vanished, and the pipeline crash-looped because its saved position in the WAL no
> longer existed. Recovery needed the offset reset in Step 11.

**Safe way:** kill the database's replication connection over and over. Debezium reconnects by itself and the
pod never restarts. You must keep killing it for **at least 70 seconds** — Prometheus only samples every 15s,
and a single kill reconnects too fast to ever be recorded.

**Run:**
```bash
for i in $(seq 1 14); do
  kubectl exec -n $NS deploy/postgresql -- psql -U debezium -d debezium -tAc \
    "select pg_terminate_backend(pid) from pg_stat_activity where backend_type='walsender';" >/dev/null 2>&1
  echo "kill $i"
  sleep 5
done
```

**Look:** **Connection Status** flips to **disconnected** partway through, then returns to **connected**
about 30s after the loop ends.

**Check** — read the whole window so you can see the dip, not just the current value:
```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 19090:9090
```

Second terminal:
```bash
curl -s --get http://localhost:19090/api/v1/query_range \
  --data-urlencode 'query=max(debezium_connection_status{service_name="test-pipeline",debezium_connection_state="connected"})' \
  --data-urlencode "start=$(date -u -v-4M '+%Y-%m-%dT%H:%M:%SZ')" \
  --data-urlencode "end=$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
  --data-urlencode 'step=15s' | jq -c '[.data.result[0].values[][1]]'
```
```
# Expect a clear dip, e.g.:
#   ["1","1","1","1","1","1","1","0","0","0","0","0","1"]
```

**Also check the pod never restarted** — that's what makes this method safe:
```bash
kubectl get pods -n $NS | grep $PIPELINE
# Expect: 1/1 Running, RESTARTS unchanged
```

> **Note the `max(...)` in the query.** After any pipeline restart, the old pod's metrics hang around for a
> few minutes, so you get *two* `connected` series — a stale `0` and the live `1`. `max()` picks the live one.
> The built-in panel doesn't do this, so the UI can briefly look disconnected right after a restart. Not a bug
> you caused.

---

## 11. Make the Snapshot panels light up

**What / Why:** The five Snapshot panels look permanently empty, and people assume they're broken. They're
not — a snapshot is the one-time initial copy of the tables, and yours finished the moment the pipeline was
created. To see them move you have to make the pipeline **snapshot again**.

Debezium remembers how far it got in a database table called `test_pipeline_offset` (named after the
pipeline). It lives in the **`postgres`** deployment — the platform's own database — *not* in `postgresql`
(the source) and not in the pipeline pod. Delete that row and restart the pipeline: with no memory of where
it was, it starts over with a fresh snapshot.

**Run:**
```bash
kubectl exec -n $NS deploy/postgres -- psql -U user -d postgres -c "DELETE FROM test_pipeline_offset;"
```
```bash
kubectl rollout restart deploy/$PIPELINE -n $NS
kubectl rollout status  deploy/$PIPELINE -n $NS --timeout=240s
```

**Look:** switch to the **Snapshot** section immediately. **Snapshot Status** goes `running` → `completed`,
**Snapshot Table Progress** briefly shows tables counting down, **Snapshot Duration** and **Snapshot Event
Count** fill in. It's quick on this dataset — watch for it.

**Check** (the status one is the clearest):
```bash
START=$(date -u -v-10M '+%Y-%m-%dT%H:%M:%SZ'); END=$(date -u '+%Y-%m-%dT%H:%M:%SZ')
curl -s "$HOST/api/monitoring/panels/snapshot-status/query?pipeline_id=$PIPELINE&start=$START&end=$END&step=15s" \
  | jq -r '.series[] | "\(.labels.debezium_snapshot_status)=\(.datapoints[-1][1])"'
```
```
# Expect:
#   aborted=0
#   completed=1
#   running=0
#   skipped=0
```

```bash
curl -s "$HOST/api/monitoring/panels/snapshot-table-count/query?pipeline_id=$PIPELINE&start=$START&end=$END&step=15s" \
  | jq -r '.series[0].datapoints[-1][1]'
# Expect: 5   (number of tables snapshotted)
```

> **Keep this step in your back pocket.** It is also the fix if the pipeline ever crash-loops because its
> saved WAL position is stale — exactly what happens after the mistake warned about in Step 10.

---

## 12. Add your own panel

**What / Why:** The most interesting thing to demo. The pipeline already exports far more than the 14 shipped
panels show — there's JVM data (`jvm_*`, from the Java agent) and ~60 Kafka producer metrics
(`kafka_producer_*`, from JMX). **No built-in panel uses any of them.** You can add panels for them with a
YAML file — no code, no rebuild, no restart.

**See the unused metrics first** — this makes the demo land:

```bash
kubectl port-forward -n $NS svc/debezium-platform-otel-collector-collector 18889:8889
```

Second terminal:
```bash
curl -s http://localhost:18889/metrics | grep '^# HELP' | awk '{print $3}' | sort -u | sed 's/_.*//' | uniq -c
```
```
# Expect (measured):
#   20  debezium    <- the 14 built-in panels use only these
#   12  jvm         <- no panel ships for these
#   61  kafka       <- no panel ships for these either
#    1  http
#    2  otlp
#   ... plus a few collector-internal ones
```

### How the override works

The conductor loads its 14 built-in panels, then loads **your** file on top and merges them **by `id`**:

- An `id` that **already exists** → your version **replaces** the built-in one (it keeps its position in the UI).
- An `id` that is **new** → it gets appended, and the UI renders it automatically, two cards per row.

A ready-made file with 1 override + 3 new panels is at **`.claude/demo/custom-panels.yml`**. The shape of one
panel:

```yaml
panels:
  - id: jvm-heap-utilization    # new id -> new panel. Reuse a built-in id -> override it.
    title: "JVM Heap Utilization"
    description: "Heap used as a percentage of the limit"
    category: streaming         # streaming | snapshot -> which UI section it appears in
    query: 'sum(jvm_memory_used_bytes{service_name="{{pipeline_id}}",jvm_memory_type="heap"}) / sum(jvm_memory_limit_bytes{service_name="{{pipeline_id}}",jvm_memory_type="heap"}) * 100'
    unit: "%"
    visualization:
      type: donut-utilization   # area | line | donut-utilization
      suggestedStep: 15s
```

`{{pipeline_id}}` is swapped for the pipeline name before the query runs. That's the whole templating system.

### Load it

**Run** — put the file in a ConfigMap:
```bash
cd /Users/ishukla/Desktop/Work/debezium-platform
kubectl create configmap custom-panels -n $NS \
  --from-file=panels.yml=.claude/demo/custom-panels.yml \
  --dry-run=client -o yaml | kubectl apply -f -
```

Mount it into the conductor and tell the conductor where to look:
```bash
kubectl patch deploy conductor -n $NS --type=strategic -p '{
  "spec":{"template":{"spec":{
    "volumes":[{"name":"custom-panels","configMap":{"name":"custom-panels"}}],
    "containers":[{"name":"conductor",
      "volumeMounts":[{"name":"custom-panels","mountPath":"/deployments/panels","readOnly":true}],
      "env":[{"name":"MONITORING_PANELS_PATH","value":"/deployments/panels/panels.yml"}]}]}}}}'
```
```bash
kubectl rollout status deploy/conductor -n $NS --timeout=180s
```

**Look:** reload the Monitoring tab. The Streaming section now has three extra cards —
**JVM Heap Utilization**, **Kafka Producer Throughput**, **JVM GC Pause Time** — and
**Streaming Event Count Rate** is retitled, because the demo file overrides it with a faster 1-minute window.

**Check:**
```bash
curl -s "$HOST/api/monitoring/panels" | jq '.panels | length'
# Expect: 17   (was 14)
```
```bash
curl -s "$HOST/api/monitoring/panels" | jq -r '.panels[] | select(.id|test("jvm|kafka")) | "\(.id)  \(.title)"'
# Expect:
#   jvm-heap-utilization       JVM Heap Utilization
#   kafka-producer-throughput  Kafka Producer Throughput
#   jvm-gc-pause               JVM GC Pause Time
```

Confirm they return real data:
```bash
START=$(date -u -v-5M '+%Y-%m-%dT%H:%M:%SZ'); END=$(date -u '+%Y-%m-%dT%H:%M:%SZ')
for p in jvm-heap-utilization kafka-producer-throughput; do
  printf '%-28s ' $p
  curl -s "$HOST/api/monitoring/panels/$p/query?pipeline_id=$PIPELINE&start=$START&end=$END&step=15s" \
    | jq -r '[.series[].datapoints[][1]] | max'
done
```
```
# Expect e.g.
#   jvm-heap-utilization         5.32        (percent)
#   kafka-producer-throughput    41611.1     (bytes/sec, during traffic)
```

> **Why the 1-minute override is worth showing:** the same 300-row burst from Step 6 reads **2.22 events/s**
> on the built-in 5-minute panel and **11.1 events/s** on the override, and reacts in ~30s instead of
> several minutes. Much better on stage.

### The best bit: edit a panel with no restart

**What / Why:** The conductor re-reads the file from disk every 30 seconds. So you can change a panel **live,
in front of an audience**, and the UI picks it up without anything being restarted.

**Run** — change a title in the ConfigMap:
```bash
kubectl edit configmap custom-panels -n $NS
```
Find `title: "JVM Heap Utilization"` and change it to `title: "JVM Heap — LIVE EDIT"`. Save and quit.

**Look:** refresh the Monitoring tab every 15s. The card title changes within about 10–90 seconds.

**Check:**
```bash
curl -s "$HOST/api/monitoring/panels" | jq -r '.panels[] | select(.id=="jvm-heap-utilization") | .title'
# Expect, after a short wait: JVM Heap — LIVE EDIT
```
```bash
kubectl get pods -n $NS | grep conductor
# Expect: RESTARTS still 0   <- this is the point of the demo
```
```
# Measured: title was live in ~10 seconds, conductor restart count 0.
# Worst case ~90s (kubelet syncs the ConfigMap file up to 60s + the conductor's own 30s cache).
```

**Put it back:**
```bash
kubectl create configmap custom-panels -n $NS \
  --from-file=panels.yml=.claude/demo/custom-panels.yml --dry-run=client -o yaml | kubectl apply -f -
```

### Make it permanent

The `kubectl patch` above is **thrown away by the next `helm upgrade`**. To keep it, add this to
`examples/example.yaml` instead and re-run the upgrade:

```yaml
conductor:
  extraVolumes:
    - name: custom-panels
      configMap: { name: custom-panels }
  extraVolumeMounts:
    - name: custom-panels
      mountPath: /deployments/panels
      readOnly: true
monitoring:
  panels:
    additionalPanelsPath: /deployments/panels/panels.yml
    refreshInterval: 30s
```

> Mount the **folder** (`mountPath: /deployments/panels`), never a single file with `subPath`.
> Files mounted with `subPath` never receive updates, which breaks the live-edit demo above.

---

## Troubleshooting

**Every panel is empty.**
Run Step 3 in order. 9 times out of 10 it's 3b — the missing `release` label on the ServiceMonitor.

**One panel is empty, the others work.**
Normal for these three:
- the 5 **Snapshot** panels — nothing is snapshotting. Use Step 11.
- **Erroneous Events** — you have no errors. That's good.
- **Events Filtered** — nothing is being filtered; no SMT is configured.

**Numbers look frozen.**
You didn't re-run the two `date` lines, so you're querying an old window. Re-run them.

**I inserted rows and nothing happened.**
Wait longer. Export every 5s + scrape every 15s + a 5-minute averaging window means ~45s minimum before a
burst is visible, and up to 2 minutes before it reaches full height.

**Connection Status shows disconnected but the pipeline is fine.**
Stale metrics from a previous pipeline pod. They age out in a few minutes. See the note in Step 10.

**The pipeline is crash-looping.**
Two usual causes:
1. A transform was added to the pipeline — see `running_locally.md` Step 13. Remove it.
2. The saved WAL position is stale (typically after the source DB was wiped). Use Step 11 to reset the offset.

**Panels went back to 14 after a `helm upgrade`.**
Expected — the `kubectl patch` in Step 12 isn't part of Helm. Use the "Make it permanent" block.

---

## Quick sequence (once you trust the checks)

A ~12-minute demo run, in order:

```bash
export NS=debezium-platform PIPELINE=test-pipeline HOST=http://platform.debezium.io

# 1. prove it's wired up
kubectl get servicemonitor debezium-platform-otel-collector -n $NS -o jsonpath='{.metadata.labels.release}'; echo
curl -s "$HOST/api/monitoring/panels" | jq '.panels | length'

# 2. steady traffic — leave running, talk over it  (Step 7)
for i in $(seq 1 60); do
  kubectl exec -n $NS deploy/postgresql -- psql -U debezium -d debezium -q -c \
    "INSERT INTO inventory.products (name,description,weight) SELECT 'tick-$i-'||g,'tick',g FROM generate_series(1,20) g;"
  sleep 5
done

# 3. Ctrl+C the loop -> "Time Since Last Event" climbs   (Step 8)
# 4. insert once more -> it snaps back to 0              (Step 6)

# 5. connection drops and recovers                       (Step 10)
for i in $(seq 1 14); do
  kubectl exec -n $NS deploy/postgresql -- psql -U debezium -d debezium -tAc \
    "select pg_terminate_backend(pid) from pg_stat_activity where backend_type='walsender';" >/dev/null 2>&1
  sleep 5
done

# 6. snapshot panels come alive                          (Step 11)
kubectl exec -n $NS deploy/postgres -- psql -U user -d postgres -c "DELETE FROM test_pipeline_offset;"
kubectl rollout restart deploy/$PIPELINE -n $NS

# 7. add three panels with no code                       (Step 12)
# 8. edit one live with no restart                       (Step 12)
```

---

## Reset to a clean state

```bash
# remove demo rows
kubectl exec -n $NS deploy/postgresql -- psql -U debezium -d debezium -c \
  "DELETE FROM inventory.products WHERE name LIKE 'demo-%' OR name LIKE 'tick-%' OR name LIKE 'burst-%';"

# drop the custom panels, back to the built-in 14
kubectl patch deploy conductor -n $NS --type=json -p '[
  {"op":"remove","path":"/spec/template/spec/volumes"},
  {"op":"remove","path":"/spec/template/spec/containers/0/volumeMounts"}]'
kubectl delete configmap custom-panels -n $NS --ignore-not-found
kubectl rollout status deploy/conductor -n $NS --timeout=180s
curl -s "$HOST/api/monitoring/panels" | jq '.panels | length'   # Expect: 14
```
