# Module 03 - GitOps / ArgoCD: scenario-based production Q&A

---

### 1. An app is `OutOfSync` forever, and `argocd app sync` fixes it for a minute, then it flips back
**Answer.** Something else mutates the live object, so Argo sees a diff each time: HPA changing `replicas`, a mutating webhook adding fields/sidecars, an operator defaulting values, or a Helm template generating non-deterministic output (`randAlphaNum`, `now`). Diagnose: `argocd app diff <app>`, look at which fields; `kubectl get -o yaml` `managedFields` to see who writes. Fix by removing the competing writer from git (drop `replicas` when HPA owns it), `ignoreDifferences` with JSON pointers/`managedFieldsManagers` for fields owned by other controllers, ServerSideApply, or make templates deterministic. Do not blanket-ignore the spec.

### 2. A bad release reached production at 16:50 Friday. Walk me through the next 15 minutes
**Answer.** (1) Detect: alert from Argo notification (Degraded) / SLO burn alert. (2) Triage: is it the latest deploy? `argocd app history`, correlate with the config-repo commit. (3) Mitigate first: if a Rollout, check whether analysis aborted it; otherwise `git revert` the bump commit through the pre-approved emergency path, sync; avoid UI rollback with auto-sync on. (4) Verify: health, error-rate, dashboards. (5) Communicate in the incident channel. (6) Later: blameless review, why did canary/analysis not catch it, add a metric or test. Target: mitigation < 10 min; the Friday rule is a signal for gaps in automated rollback, not a reason to ban deploys.

### 3. Design GitOps for 300 microservices across 4 clusters and 3 environments
**Answer.** One `acme-gitops` monorepo: `apps/<svc>/{base,overlays/<env>}`, `addons/`, `clusters/`. Hub ArgoCD per tier (nonprod, prod) registering spokes via workload identity. ApplicationSets: git directory generator for apps, cluster generator with labels for add-ons, matrix for app x cluster with selectors; prod ApplicationSet without auto-sync for stateful tiers, or with sync windows. AppProjects per team limit repos/namespaces/kinds. CODEOWNERS enforce prod approval. Promotion = PR copying the digest; Rollouts for canary. Scale: controller sharding, webhooks, manifest-generate-paths, resource exclusions. Observability: ArgoCD metrics + notifications into Backstage/DORA.

### 4. Hub-and-spoke or ArgoCD per cluster? Defend it
**Answer.** Hub: one thing to operate and one pane of glass, but it holds credentials to every cluster (high-value target), is a deployment SPOF (running apps keep running), and needs network reach. Per-cluster: isolated blast radius, no inbound credentials, works for private/air-gapped, but N upgrades and fragmented visibility. My answer: hub per tier (prod separated from nonprod), workload identity not static tokens, tested DR rebuild; per-cluster or agent mode for regulated/edge. Revisit if cluster count > ~100 or network constraints appear.

### 5. A team wants to deploy to `kube-system` / create ClusterRoles through ArgoCD
**Answer.** No. AppProject `destinations` and `clusterResourceWhitelist` prevent it, and platform-owned namespaces/netpols/quotas are blacklisted for tenants. Offer the right path: if they need CRDs/cluster-scoped resources, the platform team ships it as an add-on (reviewed, versioned) or a Crossplane/Backstage self-service; if they need elevated rights temporarily, a time-boxed exception with approval. Explain that ArgoCD controller is very privileged so projects are the security boundary.

### 6. Secrets: a developer committed a base64 Secret to the config repo
**Answer.** Base64 is not encryption; treat as leaked. Rotate the credential first, then purge history if policy requires (the secret is compromised regardless). Prevent: gitleaks in CI, push protection, Kyverno or CI rule rejecting `kind: Secret` with `data` in the repo, ESO as the standard path (Lab 07) with templates in the golden chart that already create `ExternalSecret`s, and a CODEOWNERS rule on anything secret-like.

### 7. ArgoCD itself goes down. What is the impact and how do you recover?
**Answer.** Impact: no new deployments or drift correction; workloads keep running (Argo is not in the data path). Recovery: if only a component, restart (repo-server cache, redis); if the hub cluster is lost, rebuild from git using the pinned Helm chart + bootstrap root Application, re-import cluster/repo credentials from the secret manager or `argocd admin import`, and resources resolve to `Synced` since the clusters still match git. Have a break-glass: manual `kubectl apply -k` of an overlay. Measured RTO from quarterly drills. Mention that never running two Argo instances with prune on the same targets avoids dueling.

### 8. Sync is slow: 2,000 apps take 20 minutes to refresh
**Answer.** Measure: controller reconcile p95, repo-server CPU/queue, git fetch latency. Apply in order: webhooks instead of short polling, `manifest-generate-paths` so unrelated commits don't re-render everything, exclude noisy resource kinds, increase processors and shard the controller (per-cluster shards), scale repo-servers and tune parallelism, split monorepo paths, and consider two hubs. Verify improvements with the same metrics.

### 9. Promotion: product wants prod deploys only after QA sign-off and a change ticket, but engineers want daily deploys
**Answer.** Automate the compliance, not the gate-keeping: promotion PR to prod auto-generated with evidence (tests, scan, signed digest, canary results, linked ticket via branch naming); CODEOWNERS approval from SRE/QA; GitHub environment/Kargo gate; change record created by automation (ServiceNow API). Keep a standard-change category so low-risk deploys skip CAB. Report lead time and change failure rate to show safer-and-faster with data.

### 10. Developers love `argocd app rollback` in the UI but it breaks things. Why?
**Answer.** With auto-sync enabled, rollback is blocked or immediately re-synced to git; if you disable auto-sync, the cluster drifts from git, the next sync reapplies HEAD, and the audit trail is wrong. The sound rollback is a git revert (fast path automated) or Rollouts abort/undo. Teach: Git is the control surface; UI rollback is break-glass with a follow-up PR.

### 11. Sync waves: the app deploys but the DB migration runs after the new pods start
**Answer.** Use a `PreSync` hook Job (runs before manifests, blocks sync on failure) or wave annotations with health-gated ordering; hooks must be idempotent with a delete policy. Remember hooks aren't rolled back, so migrations must be backward compatible (expand/contract) because old pods may run against the new schema during rollback or canary. For heavy migrations run as a separate controlled release.

### 12. Multi-tenant ApplicationSet incident: deleting a folder deleted a team's production namespace
**Answer.** Git directory generator removes the generated Application when the folder disappears, and the finalizer cascades deletion. Fixes: `preserveResourcesOnDeletion: true` on prod ApplicationSets, no `CreateNamespace` of prod namespaces via apps (platform namespace provisioning owns them), AppProject restrictions on Namespace kind, CODEOWNERS + required reviews on deletions, `prune` semantics reviewed, and ApplicationSet `applicationsSync: create-only` where appropriate. Recovery: restore from git history; PVC data from Velero backups.

### 13. How do you handle a Helm chart upgrade (cert-manager, ingress) across all clusters?
**Answer.** Version pinned in `addons/` once; ApplicationSet cluster generator targets; roll through clusters in waves using labels (`wave: 1` dev, `2` stg, `3` prd) with a progressive ApplicationSet strategy (RollingSync) or sequential PR merges; read release notes/CRD changes, `helm template | kubeconform`, diff the rendered output between versions in the PR; canary cluster soak 24 h; automated rollback by revert. Dependency on CRD upgrades handled with a separate CRD app or `--server-side`.

### 14. Compare ArgoCD to Flux. Why did you choose Argo?
**Answer.** Both are CNCF graduated GitOps controllers. Argo: UI, ApplicationSets, rich RBAC/AppProjects, Rollouts/Image Updater/Kargo ecosystem, strong for platform teams serving many tenants. Flux: lighter, Kubernetes-native composition (Kustomization/HelmRelease CRDs), multi-tenancy via namespaces/RBAC, native image automation and OCI sources, no central UI. Choose Argo when you want a visual control plane for developers and central multi-cluster; Flux when you prefer pure controllers per cluster and minimal footprint. Say it is a team/skills decision as much as a technical one.

### 15. Preview environments are leaking: 200 namespaces remain after PRs close
**Answer.** Check the PR generator `requeueAfterSeconds` and token rate limits, that finalizers exist so resources are cleaned, that labels filter correctly, add namespace TTL (a controller or scheduled cleanup using a `ttl` label), ResourceQuota per preview, cost alerting. Use a GitHub App token (higher rate limits). Verify with a count metric alert.

### 16. Your CISO asks how you prevent an engineer from deploying an unsigned/unscanned image
**Answer.** Layers: only CI identity can write image digests; promotion PR checks `cosign verify` against the platform workflow identity; admission policy (Kyverno `verifyImages`) rejects unsigned images and non-approved registries in every cluster; ArgoCD AppProject limits who can create apps; audit through git + Argo events. Demonstrate by trying to deploy an unsigned image and show the admission denial.

### 17. Who should be allowed to sync to production, and how do you encode it?
**Answer.** Merge to `main` is the deployment authorization; the sync itself is automatic. Humans don't need `sync` in prod; give them `get`/`logs` only. Break-glass role requires IdP group membership with just-in-time elevation (PIM), audited. Encode in `argocd-rbac-cm` with `policy.default: role:none`, project roles, and prod path protected by CODEOWNERS and branch rulesets.
