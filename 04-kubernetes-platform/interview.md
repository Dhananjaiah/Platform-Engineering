# Module 04 - Kubernetes / cloud-native platform: scenario-based production Q&A

---

### 1. A node pool upgrade has been stuck for two hours
**Answer.** Drain is blocked. `kubectl get pods -A -o wide | grep <node>`, `kubectl get pdb -A` (ALLOWED DISRUPTIONS = 0?), look for pods with `emptyDir`/local storage flags, unmanaged pods, or a PDB with `minAvailable` equal to replicas (or a single-replica app). Fix: scale up the offender or relax the PDB temporarily with owner approval; evict stubborn pods; as last resort force with documented risk. Prevent: policy requiring PDB `maxUnavailable: 1` and replicas >= 2 for prod, upgrade pre-checks listing 0-disruption PDBs, surge capacity.

### 2. Production latency spikes every day at 02:00
**Answer.** Correlate by time: cron jobs, backups (Velero), node auto-upgrade maintenance window, CronJob stampede, log rotation, GC, autoscaler scale-down evicting pods. Check `kubectl get events`, Prometheus node CPU/IO at that minute, CoreDNS latency, noisy neighbour from batch pods sharing nodes. Fix: move maintenance windows, stagger cron, isolate batch on a tainted pool, PDBs and topology spread, set priority classes. Show the data (graphs) rather than guess.

### 3. Design multi-tenancy for 40 teams on shared AKS
**Answer.** Namespace per team per env with a platform-delivered bundle: PSS restricted, quota + LimitRange, default-deny netpol + DNS allow, RBAC from Entra groups (read-only humans in prod), Kyverno policies, cost labels, ArgoCD AppProjects. Node isolation via pools/taints for special tenants (GPU, PCI), shared system pool tainted. Be honest about limits: shared kernel and control plane, so hostile or regulated tenants get dedicated clusters (or vCluster/Kata). Add noisy-neighbour protections: requests/limits, priority classes, quotas, API priority and fairness, per-tenant observability.

### 4. A developer says "my pod is Pending forever"
**Answer.** `kubectl describe pod` Events give the reason: Insufficient resources (autoscaler lag/limit reached or requests too big), taints/affinity, PVC not bound or zone mismatch, quota exceeded, topology spread unsatisfiable, scheduler gates. Each has a fix; I show the developer how to read the scheduler message and add the common causes to a Backstage "troubleshooting" doc and a Kyverno policy that rejects impossible requests early.

### 5. CoreDNS is overloaded, intermittent 5s DNS delays
**Answer.** Symptoms: random 5s latency (conntrack race/UDP drops) or SERVFAIL. Checks: CoreDNS metrics (`coredns_dns_requests_total`, latency), CPU throttling, `ndots:5` causing 5+ queries per lookup, UDP conntrack insertion race, node-local DNS cache missing. Fixes: NodeLocal DNSCache, scale CoreDNS (proportional autoscaler), reduce `ndots` for chatty apps or use FQDNs, cache TTLs, avoid external lookups in hot paths, Cilium DNS proxy tuning.

### 6. Kyverno/Gatekeeper outage froze deployments cluster-wide
**Answer.** An admission webhook with `failurePolicy: Fail` and unavailable pods blocks matching API calls (including its own recovery). Immediate: delete or patch the `ValidatingWebhookConfiguration` (`failurePolicy: Ignore`) or scale up; recover pods. Prevent: HA (3 replicas, anti-affinity, PDB), exclude the policy engine's namespace and `kube-system`, webhook `timeoutSeconds` short, `Ignore` for non-security policies, load-test, alert on webhook latency/errors, game-day. Introduced via Audit -> Enforce rollout.

### 7. Explain how a pod gets access to Azure Key Vault without secrets
**Answer.** Workload Identity: AKS OIDC issuer; Entra app/managed identity with a federated credential whose subject is `system:serviceaccount:<ns>:<sa>`; the SA is annotated with the client ID and pods labelled; the webhook projects a service-account token; the Azure SDK exchanges it for an Entra token; RBAC role on the vault is least-privilege. Nothing stored; revocation = remove the federation. EKS analogue: IRSA / Pod Identity; GKE: Workload Identity Federation. Mention token audience and that the identity per workload is created by Terraform/Backstage template.

### 8. Cluster costs doubled in a quarter; leadership wants a plan
**Answer.** Data first: OpenCost by team/service, idle %, request vs usage, node count trend, spot share, storage/logs growth. Plan: quick wins (delete zombies, dev nightly scale-down, right-size top 20 workloads, log retention), structural (autoscaling/Karpenter consolidation, spot for stateless, savings plan for baseline), governance (quotas, budgets, `cost-center` enforcement, monthly showback reviews). Commit to a target (-25 %), report weekly, and protect SLOs (cost cut can't harm reliability).

### 9. Rolling out a breaking Kubernetes upgrade (API removal) to 12 clusters
**Answer.** Detect usage early: Pluto/kubent against manifests in git and live clusters, apiserver `apiserver_requested_deprecated_apis`. Fix manifests/charts in git first, upgrade add-ons to compatible versions, then dev -> stg -> prd clusters with a soak between, one minor at a time, control plane before nodes, with maintenance windows and rollback plan (blue/green node pools, or new cluster for risky jumps). Communicate dates to teams through Backstage scorecards; make upgrade a routine monthly/quarterly cadence rather than an event.

### 10. Pods keep getting OOMKilled after a release, but memory usage in dashboards looks fine
**Answer.** Dashboards may show average or RSS; the cgroup limit counts working set including page cache and kernel memory; spikes between scrapes; JVM off-heap/direct buffers/metaspace beyond limit; sidecar containers sharing nothing but limits per container. Check `container_memory_working_set_bytes` max, `container_oom_events_total`, `describe pod` last state, heap settings relative to limit. Fix: align runtime memory flags with limits, headroom for non-heap, raise limit with data, fix leak, add max-over-time alerts.

### 11. How do you decide between one big shared cluster and many small clusters?
**Answer.** Shared (per environment tier) maximises bin-packing, lowers ops cost, simplifies add-ons; many clusters reduce blast radius, enable strict isolation, region/compliance fit, independent upgrade cadence. I choose cluster per env tier, regional, with exceptions for regulated tenants, noisy workloads (GPU/batch), and different upgrade needs; automate cluster creation so count isn't painful, and manage add-ons by ApplicationSets so N clusters ~ 1 cluster operationally.

### 12. Zero-downtime deploys are failing: users get 502 during rollouts
**Answer.** Causes: no readiness probe or too optimistic, `maxUnavailable` > 0 with low replicas, SIGTERM handling (app exits immediately), endpoint removal lag vs `preStop` sleep, keep-alive connections to terminated pods at the ingress, HPA/PDB interplay. Fixes: proper readiness, `preStop: sleep 10-15`, `terminationGracePeriodSeconds` > drain time, graceful shutdown in the app, `maxUnavailable: 0`, connection draining settings on the load balancer, tested with a load loop during rollout.

### 13. Which ingress/Gateway approach and why?
**Answer.** Gateway API: platform owns `Gateway` (listeners, TLS, allowed namespaces), teams own `HTTPRoute` — clear RBAC split and portable features; ingress-nginx is widely used but is being retired upstream (check its current support status) and is annotation-driven. Choose a Gateway API implementation (Cilium, NGINX Gateway Fabric, Envoy Gateway, Istio) by features: WAF/auth, mTLS, traffic splitting, cloud LB integration (AKS App Gateway for Containers). Migration: run both, move routes per team, keep hostnames via ExternalDNS.

### 14. A team requests privileged pods / hostPath for a monitoring agent
**Answer.** Evaluate need: use the platform-provided agent (DaemonSet owned by platform) rather than allowing tenant privileged workloads. If an exception is valid: dedicated namespace with PSS `privileged` managed by platform, Kyverno `PolicyException` with owner/expiry, minimal capabilities instead of privileged, node pool isolation, runtime security (Falco/Tetragon) auditing. Document the decision as an ADR.

### 15. After enabling default-deny NetworkPolicy, everything broke in staging
**Answer.** Typical misses: DNS egress, ingress from the gateway/controller namespace, Prometheus scrape ingress, egress to external APIs/DB private endpoints, webhooks from the API server, intra-namespace calls. Triage with Hubble drop verdicts, add allows from the golden-path template (DNS, metrics, ingress), roll out policy in `audit`/log mode first (Cilium policy audit mode), namespace by namespace, and ship the baseline allows automatically with the tenant bundle.

### 16. How do you keep 15 add-ons consistent and upgraded across 20 clusters?
**Answer.** Version pins in git per add-on and per cluster class; ApplicationSets with cluster labels and waves; render-diff in PRs; Renovate for chart bumps; soak in dev; automated smoke tests post-sync (Argo PostSync hooks); ownership and support window per add-on; scorecard for "cluster on supported add-on versions"; emergency path for CVEs.

### 17. etcd/API server latency is high on a self-managed cluster
**Answer.** (Managed AKS abstracts this, but know the answer.) Check `apiserver_request_duration_seconds`, `etcd_disk_wal_fsync_duration_seconds`, object counts (`etcd_object_counts`), large lists from controllers (no pagination, aggressive resync), webhooks adding latency, too many events/CRs, defragmentation and DB size, API Priority and Fairness queueing. Fix the noisy clients (rate limits, informers), prune objects (Jobs TTL, events), SSD disks, scale control plane.
