# Observability — Logging & Monitoring — Interview Questions

> Grounded in the `feature/fluent-bit-logging` branch of zen-gitops: Fluent Bit → Elasticsearch (Elastic Cloud) for logs, kube-prometheus-stack (Prometheus + Grafana + Alertmanager) for metrics, everything deployed via ArgoCD.
> 22 questions from fundamentals to real incident scenarios.

---

## Table of Contents

1. [Observability Fundamentals](#1-observability-fundamentals) — Q1–Q3
2. [Fluent Bit](#2-fluent-bit) — Q4–Q10
3. [Elasticsearch & Kibana](#3-elasticsearch--kibana) — Q11–Q12
4. [Prometheus](#4-prometheus) — Q13–Q17
5. [Grafana & PromQL](#5-grafana--promql) — Q18–Q19
6. [Alertmanager](#6-alertmanager) — Q20
7. [GitOps for Observability](#7-gitops-for-observability) — Q21
8. [Real-Time Troubleshooting Scenarios](#8-real-time-troubleshooting-scenarios) — Q22–Q24

---

## 1. Observability Fundamentals

**Q1.** What are the two pillars of observability in this project, and what does each one answer?

<details>
<summary>Expected answer</summary>

| Pillar | Tool | Answers |
|--------|------|---------|
| **Logs** | Fluent Bit → Elasticsearch (Elastic Cloud) | *What happened?* — errors, events, individual request traces |
| **Metrics** | Prometheus → Grafana → Alertmanager | *How is it performing?* — CPU, memory, request rate, error rate |

The project doesn't run distributed tracing (no Jaeger/Tempo) — the two implemented pillars are logs and metrics. In an interview, be upfront about that gap rather than implying tracing exists: "We have logs and metrics; tracing would be the natural next addition once we have more than one service calling another."

</details>

---

**Q2.** Walk me through the full observability data flow in this cluster, end to end.

<details>
<summary>Expected answer</summary>

```
EKS Cluster
│
├── App pods write stdout/stderr → node's /var/log/containers/
│    └── and expose a /metrics endpoint
│
├── Fluent Bit (DaemonSet, one pod per node, namespace: dev)
│    └── tails /var/log/containers/ → enriches with k8s metadata
│        → tags per-service → ships over TLS → Elasticsearch (Elastic Cloud)
│
├── Prometheus (kube-prometheus-stack, namespace: monitoring)
│    ├── scrapes /metrics from ServiceMonitors/PodMonitors every 15–30s
│    ├── scrapes Node Exporter + kube-state-metrics
│    └── evaluates PrometheusRules → fires to Alertmanager
│
├── Grafana
│    └── queries Prometheus via PromQL, dashboards auto-loaded by a sidecar
│
└── Alertmanager
     └── groups/dedupes alerts → routes to a webhook receiver
```

Both stacks are deployed declaratively through ArgoCD `Application` resources — nothing is `kubectl apply`'d by hand in the target state.

</details>

---

**Q3.** Why do logs and metrics live in two completely separate backends (Elasticsearch vs. Prometheus) instead of one system?

<details>
<summary>Expected answer</summary>

They have fundamentally different storage and query shapes:

- **Metrics** are numeric time series with low cardinality per series — Prometheus's model (label sets + timestamped floats) is optimized for fast aggregation (`rate()`, `sum()`) over that shape, and its local TSDB is cheap.
- **Logs** are unstructured/semi-structured text with high cardinality and need full-text search — Elasticsearch's inverted index is built for that, and Prometheus has no full-text search at all.

Using one tool for both would mean either bloating Prometheus with high-cardinality log strings (it isn't built for that and will fall over) or using Elasticsearch for metrics aggregation (slower and more expensive than a purpose-built TSDB). Splitting them and cross-referencing by timestamp/labels/pod name is the standard pattern.

</details>

---

## 2. Fluent Bit

**Q4.** Why does Fluent Bit run as a DaemonSet in this project, and not a Deployment or a sidecar in every pod?

<details>
<summary>Expected answer</summary>

Container logs are written by the container runtime to the **node's local filesystem** at `/var/log/containers/`. A Deployment's pods land on arbitrary nodes and wouldn't guarantee coverage of every node's log directory. A DaemonSet guarantees **exactly one Fluent Bit pod per node**, so every node's `/var/log/containers/` is read by something.

The DaemonSet mounts the host paths read-only:
```yaml
volumes:
  - name: varlog
    hostPath: { path: /var/log }
  - name: varlibdockercontainers
    hostPath: { path: /var/lib/docker/containers }
```

A sidecar-per-pod pattern is the alternative (used when apps write to files instead of stdout/stderr), but it costs one extra container per pod cluster-wide — far more resource overhead than one Fluent Bit pod per node.

</details>

---

**Q5.** Describe the actual Fluent Bit pipeline configured in `k8s/fluent-bit/configmap.yaml` — every stage, in order.

<details>
<summary>Expected answer</summary>

```
[INPUT: tail]
  Path /var/log/containers/*.log, multiline.parser docker,cri
  DB /var/lib/fluent-bit/flb_kube.db   (tracks read offset, survives restarts)
        │
[FILTER: kubernetes]
  Calls the API server to enrich each record with pod name, namespace, labels
        │
[FILTER: grep — keep]
  Regex $kubernetes['namespace_name'] dev   → only logs from the dev namespace
        │
[FILTER: grep — exclude]
  Exclude $kubernetes['container_name'] fluent-bit   → drops its own logs
        │
[FILTER: lua]
  service_index.lua → sets _service_name from the pod's "app" label
  (falls back to stripping the ReplicaSet/pod hash suffix off the pod name)
        │
[OUTPUT: es]
  Host my-elasticsearch-project-*.es.us-central1.gcp.elastic.cloud:443, TLS On
  http_api_key ${ELASTIC_API_KEY}
  Logstash_Format On, Logstash_Prefix_Key _service_name, Logstash_Prefix dev-service
  → creates one daily index per service: <service>-YYYY.MM.DD
```

Two separate `grep` filters matter here for a reason: the first is an allowlist (only `dev` namespace), the second is a denylist (exclude Fluent Bit's own container) to avoid a feedback loop where Fluent Bit ships logs about shipping logs.

</details>

---

**Q6.** What does the Lua filter (`service_index.lua`) actually do, and what's the fallback logic when the `app` label is missing?

<details>
<summary>Expected answer</summary>

```lua
function set_service_index(tag, timestamp, record)
    local svc = "dev-service"
    if record["kubernetes"] ~= nil then
        local labels = record["kubernetes"]["labels"]
        if labels ~= nil and labels["app"] ~= nil then
            svc = labels["app"]
        elseif record["kubernetes"]["pod_name"] ~= nil then
            -- strip trailing "-<replicaset-hash>-<pod-hash>"
            local stripped = record["kubernetes"]["pod_name"]:match("^(.-)%-[%w]+%-[%w]+$")
            if stripped ~= nil then svc = stripped end
        end
    end
    record["_service_name"] = svc
    return 1, timestamp, record
end
```

Priority order: use the `app` label if present → else strip the two trailing hash segments off the pod name (e.g. `catalog-service-7d8f9c6b5-x2p4q` → `catalog-service`) → else default to `dev-service` so nothing is ever dropped for lacking metadata. This `_service_name` becomes the Elasticsearch index prefix via `Logstash_Prefix_Key`, so every service gets its own daily index instead of one giant shared index — faster Kibana searches and per-service index lifecycle policies become possible later.

</details>

---

**Q7.** How does Fluent Bit authenticate to Elastic Cloud, and why is the credential handled the way it is in a GitOps repo?

<details>
<summary>Expected answer</summary>

The API key lives in a Kubernetes Secret (`fluent-bit-elastic-credentials`, created imperatively with `kubectl create secret`, **not committed to Git**) and is injected as the `ELASTIC_API_KEY` env var on the DaemonSet:

```yaml
env:
  - name: ELASTIC_API_KEY
    valueFrom:
      secretKeyRef:
        name: fluent-bit-elastic-credentials
        key: api_key
```

The ConfigMap's OUTPUT block references it as `${ELASTIC_API_KEY}` — an env-var interpolation, not a literal value. This is the standard GitOps secret-hygiene pattern: manifests in Git reference **secret names**, never secret values. The actual key is provisioned once out-of-band (or via something like External Secrets Operator in a more mature setup) and ArgoCD never needs to see it.

</details>

---

**Q8.** What RBAC does the Fluent Bit ServiceAccount need, and why a ClusterRole instead of a namespaced Role?

<details>
<summary>Expected answer</summary>

```yaml
rules:
  - apiGroups: [""]
    resources: [namespaces, pods, "pods/logs", nodes, "nodes/proxy"]
    verbs: [get, list, watch]
```

The `kubernetes` filter calls the API server to enrich each log line with pod metadata (labels, namespace, node). A **ClusterRole** is required — even though the `grep` filter later restricts what's *kept* to the `dev` namespace — because Fluent Bit's Kubernetes filter needs to look up pod/namespace metadata **cluster-wide** as records stream in from every node the DaemonSet runs on; namespace filtering happens downstream in the pipeline, not at the RBAC layer.

</details>

---

**Q9.** What happens if Fluent Bit can't reach Elasticsearch, or a node's log file gets rotated mid-read?

<details>
<summary>Expected answer</summary>

**Elasticsearch unreachable:** the `es` output has `Retry_Limit 5` — Fluent Bit retries with backoff before dropping a chunk. Records are buffered in the `Mem_Buf_Limit 50MB` in-memory buffer while retrying; for durability across pod restarts in production you'd add `storage.type filesystem` backed by a PVC instead of relying on memory alone.

**Log rotation:** the kubelet rotates `/var/log/containers/*.log` past a size threshold, which changes the file's inode. The `tail` input tracks position by inode and detects the old inode disappearing / new file appearing, so it resumes correctly. The `DB /var/lib/fluent-bit/flb_kube.db` (a SQLite file on a `hostPath` volume, not the pod's ephemeral filesystem) persists the read offset across **Fluent Bit pod restarts** too — so a Fluent Bit restart doesn't re-ship or lose the whole log file.

</details>

---

**Q10.** How would you add a brand-new microservice to this observability stack for logging — what do you actually need to change?

<details>
<summary>Expected answer</summary>

**Nothing**, as long as the new service:
1. Runs in the `dev` namespace (the `grep` filter already allows it through)
2. Writes to stdout/stderr (Fluent Bit only tails `/var/log/containers/`)
3. Has a Deployment with an `app` label set

Fluent Bit already collects from every pod in that namespace and the Lua filter derives the index name from the `app` label automatically — a new `<service>-YYYY.MM.DD` index appears in Elasticsearch on its own. The only manual step is adding a Kibana index pattern (`<service>-*`) so it shows up in Discover. This is the payoff of node-level DaemonSet collection over a sidecar-per-service approach: onboarding a new service to logging is zero-config.

</details>

---

## 3. Elasticsearch & Kibana

**Q11.** How are logs organized in Elasticsearch in this setup, and why one index per service per day instead of one big index?

<details>
<summary>Expected answer</summary>

`Logstash_Format On` + `Logstash_Prefix_Key _service_name` + `Logstash_DateFormat %Y.%m.%d` produces indices like `catalog-service-2026.07.08`, one new index per service per day.

Reasons this beats a single shared index:
- **Faster queries** — Kibana/Elasticsearch only has to search the relevant service's shard, not everything
- **Independent lifecycle management** — you could set a shorter retention (ILM policy) on a noisy/low-value service's logs without touching others
- **Isolation** — one service's log volume spike doesn't bloat a shared index that every other service also queries against

The tradeoff is more indices to manage overall, but for a handful of services this is the standard convention (it's literally what "Logstash format" naming was designed for).

</details>

---

**Q12.** How would you find all 5xx errors from a specific service in the last hour using Kibana?

<details>
<summary>Expected answer</summary>

Set the index pattern to `<service-name>-*` (so it spans day-boundary rollovers), set the time filter to "Last 1 hour", then in Discover with KQL:

```kql
kubernetes.labels.app : "catalog-service" and log : *500*
```

or, if the app logs structured status codes as a field:
```kql
kubernetes.labels.app : "catalog-service" and http.status_code >= 500
```

Since this pipeline doesn't currently parse app-specific fields out of the log body (no app-specific `PARSER` beyond `docker`/`cri`/generic `json`), in practice you're often searching the raw `log` text field unless the application itself logs structured JSON — worth calling out as a real limitation and the next improvement (adding a per-service parser).

</details>

---

## 4. Prometheus

**Q13.** Why does Prometheus use a pull model, and how does that show up in this project's config?

<details>
<summary>Expected answer</summary>

Prometheus scrapes `/metrics` endpoints on an interval rather than apps pushing metrics to it. In Kubernetes this means:
- Prometheus knows exactly which targets it expects and can flag a target as `DOWN` if scraping fails — a push model can't distinguish "app is silent because it's dead" from "app has nothing to report"
- No inbound firewall/ingress rules needed on application pods
- Target discovery is declarative via CRDs — this project uses **PodMonitor** for Fluent Bit:

```yaml
kind: PodMonitor
metadata: { name: fluent-bit, namespace: monitoring }
spec:
  namespaceSelector: { matchNames: [dev] }
  selector: { matchLabels: { app: fluent-bit } }
  podMetricsEndpoints:
    - port: http
      path: /api/v1/metrics/prometheus
      interval: 30s
```

A PodMonitor (not a ServiceMonitor) is used here specifically because Fluent Bit's metrics port (`2020`, exposed via `HTTP_Server On` in its `[SERVICE]` block) isn't fronted by a Kubernetes `Service`.

</details>

---

**Q14.** What do `serviceMonitorSelectorNilUsesHelmValues: false` and `podMonitorSelectorNilUsesHelmValues: false` do, and why are they set in this project's `prometheus-values.yaml`?

<details>
<summary>Expected answer</summary>

By default, kube-prometheus-stack's Prometheus only picks up ServiceMonitors/PodMonitors carrying labels matching the Helm release's own selector — a safety default to avoid accidentally scraping unrelated CRDs. Setting both to `false` tells Prometheus to pick up **every** ServiceMonitor/PodMonitor in the cluster, regardless of labels.

This project needs it because `fluent-bit-podmonitor.yaml` is a raw manifest applied from a different ArgoCD source (`k8s/monitoring/`, sync-waved separately from the Helm release) and was never labeled to match the monitoring Helm release's selector. Without the override, that PodMonitor would be silently ignored and Fluent Bit metrics would never appear in Prometheus — a classic "the resource exists but nothing is scraping it" gotcha.

</details>

---

**Q15.** Explain the two retention settings on Prometheus here and what happens when they disagree.

<details>
<summary>Expected answer</summary>

```yaml
prometheusSpec:
  retention: 15d
  retentionSize: "8GB"
  storageSpec:
    volumeClaimTemplate:
      spec:
        resources: { requests: { storage: 10Gi } }
```

Prometheus deletes old data blocks whenever **either** limit is hit — 15 days old, or total data exceeds 8 GB — whichever comes first. `retentionSize` exists as a safety net independent of time: if scrape volume is higher than expected, 15 days of data could exceed the PVC's 10 Gi before 15 days elapse, and without a size cap Prometheus would eventually fail to write and crash-loop. The 8 GB cap leaves ~2 Gi headroom below the 10 Gi PVC so Prometheus proactively prunes before actually running out of disk.

</details>

---

**Q16.** What's the difference between kube-state-metrics and Node Exporter, and does this project need both?

<details>
<summary>Expected answer</summary>

| | kube-state-metrics | Node Exporter |
|-|---------------------|----------------|
| Exposes | Kubernetes **object** state (pod restarts, deployment replica counts, PVC phase) | Host-level OS/hardware metrics (CPU %, disk I/O, memory) |
| Source | Kubernetes API server | `/proc`, `/sys` on each node |
| Deployment shape | Single Deployment | DaemonSet (one per node) |

Yes — they answer different questions and both are enabled (`kubeStateMetrics.enabled: true`, `nodeExporter.enabled: true`). The `crash-demo-alert.yaml` PrometheusRule in this project depends specifically on kube-state-metrics:

```yaml
expr: increase(kube_pod_container_status_restarts_total{pod="crash-demo", namespace="dev"}[2m]) > 0
```

`kube_pod_container_status_restarts_total` is a kube-state-metrics metric, not something the pod itself exposes — Node Exporter has no visibility into pod restart counts at all.

</details>

---

**Q17.** Write (or explain) the `AlertmanagerNotificationsFailing` PrometheusRule from this project. What is it actually detecting, and why alert on Alertmanager itself?

<details>
<summary>Expected answer</summary>

```yaml
expr: |
  (
    rate(alertmanager_notifications_failed_total{...}[15m])
    / ignoring (reason) group_left ()
    rate(alertmanager_notifications_total{...}[15m])
  ) > 0.01
for: 5m
```

This computes the **failure ratio** of Alertmanager's own outbound notifications (e.g. webhook/email delivery) over a 15-minute rate window, and fires if more than 1% are failing, sustained for 5 minutes. `ignoring(reason) group_left()` is needed because `_failed_total` is labeled per failure `reason` while `_total` isn't — the division needs to ignore that extra label to match series correctly.

The reason to alert on Alertmanager itself: if Alertmanager can't deliver notifications, every other alert in the cluster becomes silent even though Prometheus is correctly firing them — this is a "who watches the watchman" rule, and it's one of the most important alerts to have precisely because its own failure is invisible everywhere else.

</details>

---

## 5. Grafana & PromQL

**Q18.** Grafana crashes and restarts. Do you lose your dashboards' historical data? Why or why not?

<details>
<summary>Expected answer</summary>

No metric data is lost. Grafana is a **pure visualization layer** — it stores dashboard JSON definitions and datasource config (backed here by a `grafana` PVC on `gp2-csi`, 10Gi, `reclaimPolicy: Retain`), but the actual time-series numbers live entirely in Prometheus. On restart, Grafana just re-runs each dashboard's PromQL queries against Prometheus and re-renders — nothing about historical metric values depends on Grafana staying up. The `reclaimPolicy: Retain` on the StorageClass means even if the PVC itself were deleted, the underlying EBS volume survives so dashboard state isn't lost either.

</details>

---

**Q19.** How does this project get Kubernetes dashboards into Grafana without anyone manually importing JSON, and what does the fluent-bit dashboard config in `prometheus-values.yaml` do?

<details>
<summary>Expected answer</summary>

Two mechanisms:

**1. Sidecar auto-discovery** — `grafana.sidecar.dashboards.enabled: true` with `searchNamespace: ALL`: a sidecar container in the Grafana pod watches ConfigMaps cluster-wide labeled `grafana_dashboard: "1"`, copies their JSON into Grafana's dashboard path, and Grafana picks them up live — no restart, no manual import.

**2. Pre-provisioned dashboard from Grafana.com:**
```yaml
dashboards:
  default:
    fluent-bit:
      gnetId: 7752
      revision: 1
      datasource: Prometheus
```
This tells the Helm chart to fetch community dashboard #7752 (the standard Fluent Bit dashboard) from grafana.com at install time and wire it to the `Prometheus` datasource automatically — so Fluent Bit metrics (buffer usage, output errors, retries) have a working dashboard from day one without hand-building panels.

</details>

---

## 6. Alertmanager

**Q20.** Explain this project's Alertmanager routing config, and how you'd reduce alert fatigue without losing critical pages.

<details>
<summary>Expected answer</summary>

```yaml
route:
  group_by: [alertname, cluster, service]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  receiver: webhook-demo
  routes:
    - match: { severity: critical }
      receiver: webhook-demo
```

- `group_by` batches related alerts (same alertname/cluster/service) into one notification instead of one-per-alert
- `group_wait: 30s` — wait 30s after the first alert in a new group to see if related alerts join it, before sending
- `group_interval: 5m` — minimum gap before sending an update about additional alerts joining an existing group
- `repeat_interval: 12h` — how often to re-send a still-firing alert if it's never resolved

To reduce noise further: raise `group_wait`/`repeat_interval`, add `inhibit_rules` (e.g. suppress `severity: warning` when a `severity: critical` alert with the same `alertname`/`namespace` is already firing), and require a longer `for:` duration on flappy rules so transient blips don't page anyone. None of these should touch alerts already scoped `severity: critical` with a real receiver — the goal is fewer *duplicate/low-value* notifications, not fewer *real* ones.

</details>

---

## 7. GitOps for Observability

**Q21.** The `monitoring` ArgoCD Application uses three separate `sources` instead of one. What is each source for, and why split them?

<details>
<summary>Expected answer</summary>

```yaml
sources:
  - repoURL: https://prometheus-community.github.io/helm-charts   # the Helm chart itself
    chart: kube-prometheus-stack
    targetRevision: 84.3.0
    helm:
      valueFiles: [$values/envs/monitoring/prometheus-values.yaml]

  - repoURL: .../zen-gitops.git                                   # values-file source, aliased "values"
    targetRevision: feature/fluent-bit-logging
    ref: values

  - repoURL: .../zen-gitops.git                                   # raw manifests (PodMonitor, etc.)
    targetRevision: feature/fluent-bit-logging
    path: k8s/monitoring
```

ArgoCD's **multi-source Application** feature lets one Application combine an upstream Helm chart (pinned to `84.3.0`, not vendored into this repo) with values files that live in this Git repo (referenced via the `ref: values` alias so the first source's `valueFiles` can point at `$values/...`), plus a third source for hand-written manifests that aren't part of the chart at all (the `PodMonitor`, `PrometheusRule`s, PVC, StorageClass). This avoids forking/vendoring the upstream chart just to add a couple of extra resources, while still keeping everything — chart version, values, and extras — declared in one Application and one Git history.

Also worth knowing: this Application's `ignoreDifferences` block excludes `caBundle` on the webhook configs from drift detection — the kube-prometheus-stack admission webhook injects its own CA at runtime, and without ignoring that field ArgoCD would show permanent, unfixable drift ("OutOfSync") on every sync even though nothing is actually wrong.

</details>

---

## 8. Real-Time Troubleshooting Scenarios

**Q22.** Logs stopped showing up in Kibana for one service, but other services' logs are fine. Walk through your debugging steps.

<details>
<summary>Expected answer</summary>

Since it's isolated to one service, connectivity/auth to Elastic Cloud is probably fine — focus on that service's pods and metadata:

```bash
# 1. Confirm the pod is actually running and has the 'app' label set correctly
kubectl get pod <pod> -n dev -o jsonpath='{.metadata.labels}'

# 2. Confirm Fluent Bit is running on that pod's node
kubectl get pods -n dev -l app=fluent-bit -o wide | grep <node-name>

# 3. Check Fluent Bit logs on that node's pod for grep/lua filter errors
kubectl logs -n dev <fluent-bit-pod-on-that-node> --tail=100

# 4. Check if the index actually exists but under an unexpected name
#    (missing 'app' label → Lua fallback strips pod-hash → could produce
#     an unexpected index prefix, or default to "dev-service")
# Kibana Dev Tools:
GET _cat/indices?v&s=index | grep -i <expected-service-name>

# 5. Check the fluent-bit-podmonitor / Prometheus fluentbit_output_errors_total
#    for that specific node's pod to see if it's silently failing writes
```

The most common root cause when it's isolated to one service (not all of them) is a missing/changed `app` label on the Deployment, which changes the derived `_service_name` and therefore which index the logs land in — they aren't lost, they're just landing somewhere unexpected.

</details>

---

**Q23.** Prometheus shows the `fluent-bit` PodMonitor target as `DOWN`. What do you check, in order?

<details>
<summary>Expected answer</summary>

```bash
# 1. Read the actual error on the Targets page (Prometheus UI → Status → Targets)
#    "connection refused" vs "context deadline exceeded" point to different causes

# 2. Confirm the pod is exposing the port the PodMonitor expects
kubectl get pod <fluent-bit-pod> -n dev -o jsonpath='{.spec.containers[0].ports}'
# PodMonitor expects port name "http" → containerPort 2020

# 3. Curl the metrics endpoint directly from inside the cluster
kubectl exec -n monitoring <prometheus-pod> -- \
  curl -s http://<fluent-bit-pod-ip>:2020/api/v1/metrics/prometheus | head

# 4. Confirm label selectors actually match
kubectl get pods -n dev -l app=fluent-bit --show-labels
kubectl get podmonitor fluent-bit -n monitoring -o yaml   # check selector.matchLabels

# 5. Confirm the "picked up regardless of labels" override is actually applied
#    (a common false lead: assuming the PodMonitor isn't found, when actually
#     it *is* found but scrape is failing for a network/port reason)
helm get values monitoring -n monitoring | grep -i SelectorNilUsesHelmValues
```

The one gotcha specific to this project: the PodMonitor lives in `namespace: monitoring` but targets pods in `namespace: dev` via `namespaceSelector.matchNames: [dev]` — if that cross-namespace selector were ever mistyped, Prometheus would report zero targets discovered rather than a target being down, which is a different symptom worth distinguishing early.

</details>

---

**Q24.** The on-call channel is getting flooded with duplicate alerts for the same crash-looping pod every few minutes. What's actually happening, and what do you change?

<details>
<summary>Expected answer</summary>

Look at `crash-demo-alert.yaml`:
```yaml
expr: increase(kube_pod_container_status_restarts_total{pod="crash-demo", namespace="dev"}[2m]) > 0
for: 1m
interval: 30s
```

This fires every time restarts *increase* in a 2-minute window — a genuinely crash-looping pod (restarting every 30-60s) will keep re-triggering the condition, and with `repeat_interval: 12h` at the Alertmanager level but `group_interval: 5m`, updates to an *existing* firing group still go out every 5 minutes while new distinct alert instances can also be grouped in.

What to change, in priority order:
1. **Fix the underlying pod** — this is a symptom, not a config problem, if the pod genuinely won't stabilize
2. Increase the rule's `for:` duration so a single restart doesn't fire — require sustained crash-looping over a longer window
3. Widen `group_interval` so repeated updates to the same firing alert are batched further apart
4. Add an `inhibit_rule` if a higher-severity alert (e.g. "namespace unhealthy") should suppress this lower-level one while it's active

The fix is *not* to raise the threshold in `expr` from `> 0` to something higher — that would mask genuinely low-frequency crash loops instead of just de-duplicating notifications about a known one.

</details>

---
