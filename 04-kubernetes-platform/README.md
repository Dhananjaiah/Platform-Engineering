# 04 - Kubernetes and cloud-native platform (AKS first, EKS/GKE parity)

Running multi-tenant clusters, not just deploying to them.

| Lab | Topic | Where |
|-----|-------|-------|
| [01](lab-01-aks-terraform/README.md) | AKS cluster as Terraform (pools, autoscaling, workload identity, CNI, upgrades); EKS/GKE equivalents | Azure (plan-only OK) |
| [02](lab-02-multitenancy/README.md) | Tenant baseline: namespace, quota, LimitRange, default-deny, RBAC, Pod Security | kind |
| [03](lab-03-policy/README.md) | Kyverno suite (+ Gatekeeper equivalent), signed-image verification, policy tests | kind |
| [04](lab-04-networking/README.md) | Cilium policies, Gateway API, cert-manager, ExternalDNS, mesh mTLS | kind |
| [05](lab-05-autoscaling/README.md) | HPA, VPA, KEDA, Cluster Autoscaler/Karpenter concepts with load test | kind |
| [06](lab-06-observability/README.md) | Prometheus/Grafana, OpenTelemetry, Loki, SLOs, developer-facing dashboards | kind |
| [07](lab-07-reliability/README.md) | PDB, probes, topology spread, Velero backup/restore, zero-downtime upgrade, add-on management | kind |
| [08](lab-08-troubleshooting/README.md) | Fault-injection dojo: OOMKill, CrashLoop, node pressure, DNS/netpol, ImagePull | kind |
| [09](lab-09-cost/README.md) | OpenCost, showback, right-sizing | kind |

[Scenario interview Q&A](interview.md)

## Design stance to articulate
- **Cluster per environment tier, shared within tier**, namespaces per team; prod isolated by subscription and cluster. Dedicated clusters for regulated or noisy tenants.
- System node pool (tainted `CriticalAddonsOnly`) separate from user pools; spot pools for tolerant workloads; GPU pools tainted.
- Identity: workload identity everywhere, zero static cloud secrets.
- Policy at admission (Kyverno/Gatekeeper) + Azure Policy for cloud resources.
- Add-ons are products: versioned in git, installed by ApplicationSets (Module 03), upgraded in waves.
- Upgrades are routine: surge, PDBs, N-1 version skew rules, nonprod first.

## Say it in 2 minutes
"Clusters are cattle built by Terraform with a tainted system pool, autoscaled user pools and workload identity. Tenants get a namespace baseline (quota, limit ranges, default-deny, RBAC, PSS restricted) delivered by the platform, enforced by Kyverno, and visible in Backstage. Networking uses Cilium with Gateway API, cert-manager and ExternalDNS. Observability is Prometheus/Otel/Loki with SLO alerts. Reliability comes from PDBs, probes, spread constraints and tested Velero restores. Cost is showback per team via labels and OpenCost. Add-ons are rolled out across clusters by ApplicationSets in waves."
