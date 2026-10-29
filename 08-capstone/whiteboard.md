# Whiteboard: the end-to-end platform flow

## Draw this (practise until it takes 90 seconds)
```
 Developer
    |  (1) picks golden-path template
    v
 Backstage (catalog, scaffolder, TechDocs, scorecards, plugins)
    |  (2) scaffold                          (3) PRs
    v                                         |-> acme-infra  (Terraform product module, policy, plan/apply via OIDC)
 GitHub repo: code + catalog-info + thin     |-> acme-gitops (ApplicationSet picks up app dir)
 caller workflows --> reusable platform workflows
    |  (4) CI: build, test, SAST, deps, Trivy, SBOM, Cosign sign, provenance, push image@digest
    v
 Registry --digest--> commit/PR to acme-gitops (dev overlay)
    |
    v  (5) ArgoCD hub (ApplicationSets, AppProjects) --> AKS dev -> stg -> prd (promotion PRs)
                          Kyverno verifies signature/policy at admission; Argo Rollouts canary + Prometheus analysis
    |
    v  (6) Observability: Prometheus/Loki/OTel -> dashboards + SLO alerts -> Backstage (owner, deploys, health, cost, scorecard)
    |
    v  (7) DORA + DX metrics -> roadmap
```

## Per step: what to say, trade-off, failure behaviour
| Step | Key design choice | Trade-off | If it fails |
|---|---|---|---|
| 1 Backstage | portal as product, thin first release | build vs buy; run cost | deploys continue via git; portal is a convenience layer |
| 2 Scaffold | thin skeleton, logic in reusable workflows/modules | central updates vs per-repo flexibility; drift handled by stamping and bots | task fails mid-way: idempotent actions, retry, cleanup |
| 3 Infra/GitOps PRs | PRs over direct API calls | latency (minutes) vs audit/policy/rollback | PR checks fail visibly, nothing applied |
| 4 CI | OIDC, SHA-pinned actions, digest everywhere, keyless signing | scan time vs security; mitigated by caching/parallelism | build fails, nothing deployed |
| 5 ArgoCD | hub per tier with ApplicationSets | hub = credential concentration and SPOF for deploys; per-cluster is more ops | workloads keep running; rebuild hub from git |
| 5 Rollouts | SLI-gated canary | needs good metrics and traffic; no-data trap | auto-abort, stable keeps serving |
| 6 Observability | golden dashboards, SLO burn alerts by owner | cost/cardinality | alert on pipeline lag; fallback to kubectl |
| 7 Metrics | DORA/SPACE/adoption with SHA linkage | gaming risk (Goodhart) | roadmap uses qualitative data until fixed |

## "What if" questions to rehearse
- **Backstage is down?** Delivery path is git + CI + ArgoCD, independent of the portal. Only discovery/self-service is degraded; documented direct paths exist.
- **ArgoCD hub is down?** No new rollouts/drift correction; running workloads unaffected; restore from git + credentials backup; RTO from drills.
- **GitHub is down?** No merges/CI; running prod unaffected; emergency path: break-glass sync from a mirror/runbook; plan for provider outage is mostly "wait", with a tested manual deploy for sev1 hotfix only.
- **Registry down?** Pods already running are fine; new pods fail to pull: pull-through cache/mirror, multi-region registry, image pre-pulled on nodes.
- **A compromised dependency?** SBOM tells us where it runs; Trivy/Dependabot alert; rebuild with patched dependency; admission policy blocks images older than N days if needed; sign/provenance chain prevents rogue images.
- **A compromised developer account?** SSO + MFA, branch protection and required reviews, signed commits, OIDC scoped by environment, CODEOWNERS for prod, short-lived tokens, audit trails.
- **10x growth?** Hub sharding and per-tier hubs, ARC runners autoscale, Postgres vertical + read replicas for Backstage, monorepo path filters, templates thin, cost showback.
- **Multi-region/cloud?** Same module/product API, region-parameterised ApplicationSets and Terraform; policy as code reused; Crossplane/Terraform abstractions hide provider differences where valuable.

## Trade-off prompts interviewers use
1. Monorepo vs polyrepo for GitOps config? (Module 03 Lab 04)
2. Terraform via PR vs Crossplane for self-service? (Module 02 Lab 03, Module 05 Lab 08)
3. Hub vs per-cluster ArgoCD? (Module 03 Lab 03)
4. Helm vs Kustomize vs platform CRD? (Module 03 Lab 04, Module 02 Lab 03)
5. Gatekeeper vs Kyverno? (Module 04 Lab 03)
6. Self-hosted ARC vs hosted runners? (Module 06 Lab 04)
7. Build vs buy portal? (Module 02 Lab 05)
8. Mandate vs pull for adoption? (Module 07 Lab 02)
