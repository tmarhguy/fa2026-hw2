## Part 1

> What does a Deployment give you that a bare Pod doesn't? Why do we need a separate Service object instead of talking to a Pod's IP directly?

A bare Pod is the smallest deployable unit in Kubernetes — one or more containers
sharing network and storage — but it is ephemeral and unmanaged. If its container
crashes, the Pod dies and stays dead; if its Node dies, the Pod is gone; there is
no record of "how many copies should exist," no rolling update strategy, and no
way to roll back. A Deployment adds that management layer on top of Pods via a
ReplicaSet control loop: you declare `replicas`, a pod template, and an update
strategy, and the controller continuously reconciles reality toward that desired
state. It restarts/replaces failed Pods, reschedules them onto healthy Nodes,
scales the ReplicaSet up/down, performs rolling updates (with `maxUnavailable` /
`maxSurge`) and rollbacks (`kubectl rollout undo`), and uses `selector.matchLabels`
so it always adopts the right Pods even as individual Pod names/IPs churn.

A Pod IP is also ephemeral: every time a Pod is recreated (crash, reschedule,
upgrade) it gets a new IP, and with multiple replicas there are multiple changing
IPs with no built-in load balancing. Nobody — not another Pod, not `kubectl
port-forward`, not the Open WebUI frontend with its `OLLAMA_BASE_URL` — can safely
hardcode a Pod IP. A Service fixes this by providing a stable virtual IP (ClusterIP)
plus a stable DNS name (e.g. `ollama` / `ollama:11434`) backed by label-selector
endpoint tracking and kube-proxy load balancing. When the Deployment swaps old Pods
for new ones, the Service's Endpoints list is updated automatically, so clients see
zero config change. In short: Deployments give you self-healing, scaling, and safe
updates; Services give you stable discovery and load balancing over an ever-changing
set of Pods.

## Part 2

> What is a Helm chart, and how do `values.yaml` plus `-f`/`--set` let the same chart produce different Kubernetes resources for different environments? Why is this better than keeping separate copies of your Part 1 manifests for staging and production?

A Helm chart is a versioned package of templated Kubernetes manifests
(`templates/*.yaml` with Go-template placeholders like `{{ .Release.Name }}` and
`{{ .Values.service.port }}`) plus a `values.yaml` file of default configuration,
a `Chart.yaml` of metadata, and optional hooks. Installing a chart creates a
**release**: a named, revision-tracked instance of that chart in a namespace
(e.g. `ollama-staging` in `staging` vs. `ollama-production` in `production`).
Helm renders the templates with a merged values dict and applies the result as one
atomic unit, recording the revision so `helm upgrade` / `helm rollback` /
`helm uninstall` work per release.

`values.yaml` holds the defaults (image, model `qwen2.5:0.5b`, ports, persistence
size, resource requests/limits). At install/upgrade time, `-f values-staging.yaml`
(or `-f values-production.yaml`, or `--set resources.limits.memory=1Gi`) deep-merges
over those defaults, so the *same* template files render to *different* live objects:
`{{ .Release.Name }}-ollama` / `{{ .Release.Name }}-ollama-data` keep object names
from colliding, `{{ .Values.model.name }}` picks a small model for staging and a
larger `qwen2.5:1.5b` for production, `{{ .Values.resources }}` / `{{ .Values.persistence.size }}`
give staging cheap 2Gi/1Gi limits and production larger 5Gi/4Gi limits, and the
`OLLAMA_BASE_URL` / `OLLAMA_HOST` strings automatically point at the per-release
Service DNS name. `helm template -f <file>` lets you preview that rendering without
touching the cluster.

This beats maintaining two copies of the Part 1 YAML because there is a single
source of truth: fix a probe, bump an image, or add a hook (like the
`post-install,post-upgrade` model-pull Job that replaces the manual
`kubectl exec ... ollama pull`) once in the chart and both environments inherit it
on next upgrade, instead of hand-editing duplicated files and drifting. Per-release
values are tracked (`helm get values`), so `helm upgrade ollama-staging -f
values-staging.yaml` changes only staging while production is untouched — independent
deploy/upgrade/rollback per environment from one chart, applied in dependency order
in one command rather than `kubectl apply -f` in a memorized order.
