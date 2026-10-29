# Lab 04-09 - Cost: OpenCost, showback per team, right-sizing

## Goal
Measure cost by namespace/team label, find waste (idle, over-requested), and produce a showback report. Understand allocation models so you can defend numbers.

## Prereqs
Prometheus installed (Lab 06). Workloads labelled `owner`, `service`, `env`.

## Step 1 - Install OpenCost
```bash
helm repo add opencost https://opencost.github.io/opencost-helm-chart && helm repo update
helm install opencost opencost/opencost -n opencost --create-namespace \
  --set opencost.prometheus.internal.serviceName=kps-kube-prometheus-stack-prometheus \
  --set opencost.prometheus.internal.namespaceName=monitoring \
  --set opencost.ui.enabled=true
kubectl -n opencost rollout status deploy/opencost
kubectl -n opencost port-forward svc/opencost 9090:9090 &
```
UI `http://localhost:9090`. On AKS/EKS/GKE set cloud pricing (`opencost.exporter.cloudProviderApiKey` or the provider's billing integration) for real prices; on kind it uses default pricing: relative numbers are still useful for practice.

## Step 2 - Query the allocation API
```bash
curl -s "http://localhost:9090/allocation/compute?window=7d&aggregate=namespace&accumulate=true" | jq -r '.data[0] | to_entries[] | [.key, .value.totalCost, .value.cpuEfficiency, .value.ramEfficiency] | @tsv' | sort -k2 -nr | head
curl -s "http://localhost:9090/allocation/compute?window=7d&aggregate=label:owner&accumulate=true" | jq '.data[0] | map_values({cost: .totalCost, cpuEff: .cpuEfficiency})'
```
Aggregating by the `owner` label is **showback** (reporting cost to teams); **chargeback** moves money. Start with showback.
Allocation model to explain: container cost = max(request, usage) share of node cost (CPU and RAM weighted), plus shared costs (system namespaces, idle) distributed by a policy (`shareNamespaces=kube-system,monitoring`, `shareIdle=weighted`). Idle cost = capacity not requested: platform's efficiency KPI.

## Step 3 - Efficiency and waste
```
cpuEfficiency = used / requested     # low = over-requested
ramEfficiency = used / requested
```
Find the top over-requested workloads:
```bash
curl -s "http://localhost:9090/allocation/compute?window=7d&aggregate=controller&accumulate=true" | jq -r '.data[0] | to_entries[] | select(.value.cpuEfficiency < 0.2 and .value.cpuCoreRequestAverage > 0.2) | [.key, .value.cpuCoreRequestAverage, .value.cpuEfficiency] | @tsv'
```
Cross-check with VPA recommendations (Lab 05). Recommend requests = p90-p95 usage + headroom; memory request ~ p99 + 10-20 %.

## Step 4 - PromQL for a Grafana showback board
```
sum by (label_owner) (
  container_memory_working_set_bytes{container!=""} * on(namespace,pod) group_left(label_owner) kube_pod_labels
) / 1024^3
sum by (namespace) (avg_over_time(node_total_hourly_cost[1d]) * 24)
```
Panels: cost by team (7d, 30d), top 10 workloads by waste, idle %, spot vs on-demand, trend vs budget. Publish as a Backstage tab per owner (Module 01 plugin: OpenCost or a link with the team label).

## Step 5 - Levers (ranked by ROI)
1. Delete idle/zombie workloads and unattached disks/IPs (weekly report).
2. Right-size requests (VPA/OpenCost) and set sensible defaults via LimitRange.
3. Autoscaling: HPA/KEDA for apps, scale-to-zero for dev at night (`kube-downscaler` or KEDA cron), cluster autoscaler consolidation.
4. Spot/preemptible for stateless and batch with PDBs and fallback pools (~60-80 % cheaper).
5. Reserved instances/savings plans for the baseline (commit ~60-70 % of steady usage).
6. Storage: right tier, snapshot retention, log retention/sampling (observability can be 20-30 % of the bill).
7. Dev environments: smaller SKUs, TTL on previews.
Guardrails: ResourceQuota per team, budget alerts by label, Kyverno requiring `cost-center`.

## Step 6 - Make it actionable
Weekly digest per owner: top 3 savings opportunities with links. Track "savings realized" (before/after) to prove platform value: this ties to the leadership narrative (Module 07).

## Break it on purpose
1. Deploy a workload requesting 4 CPU but using 10 m; watch efficiency tank and its cost allocate to the owner.
2. Remove the `owner` label from a namespace: costs appear under `__unallocated__`. Add a policy making unlabeled workloads fail admission, and track % unallocated as a KPI (target < 5 %).

## Cleanup
```bash
helm uninstall opencost -n opencost; kill %1 2>/dev/null
```

## Talking points
Showback before chargeback, allocation = max(request, usage), idle cost visibility, label hygiene is the foundation, savings levers ranked, FinOps as continuous process with owners and data.
