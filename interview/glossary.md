# Glossary

| Term | Meaning |
|---|---|
| ADR | Architecture Decision Record: short immutable record of a decision, context and consequences |
| AppProject | ArgoCD tenancy boundary restricting repos, destinations and resource kinds |
| ApplicationSet | ArgoCD controller generating Applications from generators (git, cluster, matrix, list, PR) |
| ARC | Actions Runner Controller: autoscaling GitHub runners on Kubernetes |
| Claim / XR / Composition | Crossplane: developer request / composite resource / implementation mapping to managed resources |
| Cosign / Sigstore | Container signing and verification, keyless via OIDC identity and transparency log |
| DORA | Four delivery metrics: deployment frequency, lead time, change failure rate, time to restore |
| ESO | External Secrets Operator syncing secrets from a secret manager into Kubernetes |
| Golden path | Opinionated, supported way to build and run a type of service |
| HPA / VPA / KEDA | Horizontal / Vertical Pod Autoscaler / Event-driven autoscaling |
| IDP | Internal Developer Platform |
| Kargo | Promotion orchestration for GitOps environments |
| Kyverno / Gatekeeper | Kubernetes admission policy engines (YAML/CEL vs Rego) |
| OIDC federation | Exchanging a short-lived IdP-signed token for cloud credentials, no stored secrets |
| PDB | PodDisruptionBudget limiting voluntary disruptions |
| PSS | Pod Security Standards: privileged, baseline, restricted |
| RFC | Request for Comments: design proposal under review |
| SBOM | Software Bill of Materials listing components in an artifact |
| Scaffolder | Backstage software template engine |
| SLI / SLO | Service level indicator / objective; error budget = 1 - SLO |
| SLSA | Supply-chain Levels for Software Artifacts |
| SPACE | Developer productivity framework: Satisfaction, Performance, Activity, Communication, Efficiency |
| Sync wave / hook | ArgoCD ordering of resources / actions at sync phases |
| Tech Insights | Backstage plugin for facts, checks and scorecards |
| Team Topologies | Stream-aligned, platform, enabling and complicated-subsystem teams; interaction modes |
| TVP | Thinnest Viable Platform |
| Workload Identity | Pod-to-cloud identity without static secrets (AKS/GKE; IRSA/Pod Identity on EKS) |
