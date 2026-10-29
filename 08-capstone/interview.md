# Module 08 - End-to-end and cross-cutting scenario Q&A

---

### 1. "Walk me through what happens from the moment a developer clicks 'Create' until the service serves production traffic."
**Answer.** Use the whiteboard (`whiteboard.md`): Backstage scaffolder generates repo + two PRs + catalog registration; infra PR plans/applies Terraform product module via OIDC after policy checks; gitops PR adds the app folder discovered by an ApplicationSet; first push triggers reusable CI (tests, SAST, deps, Trivy, SBOM, Cosign, provenance) and a bump PR with the image digest; ArgoCD syncs to dev; promotion PRs to stg/prd with checks and approvals; Argo Rollouts canary with Prometheus analysis; Kyverno verifies signatures at admission; Backstage aggregates ownership, health, cost, scorecard; DORA data feeds the roadmap. Add timings and ownership for each hop and failure behaviour.

### 2. "Where are the single points of failure in this design and how do you mitigate them?"
**Answer.** GitHub (outage blocks merges/CI): running prod unaffected, break-glass sev1 hotfix procedure; ArgoCD hub (stops deploys/drift correction): HA install, backups of credentials, rebuild-from-git drill, per-tier hubs; registry (new pods can't pull): geo-replicated registry/pull-through cache; admission controllers (cluster-wide API impact): HA, excludes, failurePolicy, alerts; Backstage (UX only); Terraform state backend (locks/corruption): versioned storage, restricted access; secrets store: replicas, caching in ESO, runbooks; people: bus factor, on-call, runbooks. Show you thought about *degraded modes*, not only prevention.

### 3. "How would you secure this supply chain end to end?"
**Answer.** Developer identity (SSO/MFA, signed commits, branch protection/CODEOWNERS), pinned/allow-listed actions, least-privilege tokens and OIDC (no secrets), isolated ephemeral runners, scans (SAST, deps, container, IaC) with expiring exceptions, SBOM + keyless signing + provenance, deploy by digest, admission verification of signature/identity, config repo protections and policy tests, secrets from a vault via workload identity, audit logs across GitHub/ArgoCD/Kubernetes/cloud, and incident procedures (revocation, rebuild). Name the residual risks (compromised base image, malicious maintainer, zero-day) and compensating controls (SBOM inventory, rapid rebuild).

### 4. "Production is down after a deploy. Show me how your platform helps and how it might hurt."
**Answer.** Helps: Backstage points to owner and on-call in seconds; ArgoCD history shows the exact change; Rollouts likely auto-aborted a bad canary; revert PR rolls back in minutes; dashboards/SLO alerts localise; runbooks linked. Hurts: if the platform component itself (ArgoCD, CI) is the failure, you need the break-glass path; alert noise; stale catalog ownership; slow CI delaying the revert (so the emergency revert path skips non-essential checks yet keeps signature/policy gates). Postmortem: add detection/guardrails; measure MTTR.

### 5. "How do you onboard 100 existing services onto this platform?"
**Answer.** Baseline and segment; make onboarding near-zero effort with automation: bot PR adds `catalog-info.yaml`, CI caller, gitops overlay generated from existing manifests (`kustomize` conversion), parity checks; waves (volunteers, then bulk, then hold-outs); migration squad, office hours; burn-down dashboard; pull via scorecards and features only on the path; push via dated sunset after proof; decommission legacy with credential revocation. (Module 07 Lab 02.)

### 6. "Estimate the team, timeline and milestones to build this from scratch for 400 engineers."
**Answer.** Team: 6-8 (product-minded lead, 2 portal/DX, 2 delivery (CI/GitOps), 2 runtime/IaC, 1 SRE/security partner part-time). Timeline: Q1 foundations and pilot (Backstage MVP, golden path v1, GitOps for 3 teams, reusable CI); Q2 self-service (DB/namespace), promotion/rollouts, catalog accuracy; Q3 scale adoption (migration waves, scorecards, cost showback); Q4 resilience/efficiency (upgrade automation, DR drills, right-sizing). Milestone metrics per quarter (time-to-first-deploy, adoption %, lead time). Risks: scope creep, adoption, capacity; mitigations: TVP discipline, design partners, advisory group, explicit "not doing".

### 7. "What would you do differently if the company were 30 engineers instead of 400?"
**Answer.** Much thinner: no Backstage initially (a README catalog or a SaaS portal), a single reusable workflow, one cluster per env, ArgoCD with plain Kustomize, Terraform modules minimal, no policy engine beyond essentials, buy rather than build, avoid the platform team becoming a headcount sink; the golden path is a repo template. Revisit at 100-150 engineers when coordination cost rises.

### 8. "How do you ensure the platform doesn't become a bottleneck for security patches (e.g. a critical base-image CVE)?"
**Answer.** Inventory by SBOM (query where the vulnerable package runs), rebuild centrally: base image updated -> Renovate PRs to all repos or a central rebuild workflow; reusable CI picks the new base; auto-merge for dev; promote through the pipeline with prioritised canaries; admission policy can warn/block images older than N days; track exposure time as a metric; run a game day for "log4shell-style" events. Platform provides the mechanism; teams own their services' patch SLAs with dashboards.

### 9. "What's your approach to documentation and developer onboarding?"
**Answer.** Docs-as-code (TechDocs) next to code, templates generate docs skeleton, runbook and ADR templates, search integrated, "golden path" quickstart that gets a new hire to first deploy in a day, office hours and recorded demos, docs freshness checks in scorecards, feedback buttons and analytics on docs gaps, owners per doc set.

### 10. "Pick the three riskiest assumptions in your platform plan and how you'd test them early."
**Answer.** (1) Developers will prefer the golden path over their scripts: pilot with 3 teams, measure time-to-first-deploy and satisfaction, interview. (2) The catalog can stay accurate: pilot ownership sync and automated validation for 20 services, measure drift after 4 weeks. (3) GitOps config repo and ArgoCD scale to our service count: load-test with 500 generated apps and measure reconcile latency. Each test is cheap and time-boxed with a go/no-go threshold.
