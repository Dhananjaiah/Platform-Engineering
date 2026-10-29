# Module 00 - Foundations: scenario-based Q&A

### 1. Why separate the app repo from the GitOps config repo?
**Answer.** Different change cadence and access (developers merge code; only CI/promotion workflows and approvers change environment state), no CI loops from config commits, a single audit trail of "what is deployed and who approved it", clean rollback by reverting config without touching source, and the ability to apply stricter protections (CODEOWNERS for prod paths) to config only. Trade-off: extra repo and an automation step; mitigated by templates and the bump workflow.

### 2. How do you design environments for a regulated company?
**Answer.** Separate prod from non-prod by cloud subscription/account, cluster and credentials; promotion only via reviewed PRs of immutable digests; least-privilege identities per environment (OIDC subjects bound to GitHub environments); policy as code with audit evidence; network segmentation; break-glass access with PIM and alerts; data classification governs what can exist in lower environments.

### 3. Your lab clusters keep running out of resources; how do you keep the learning environment sane?
**Answer.** Right-size (hub 3 nodes, spokes 2), start only needed clusters, teardown commands each lab, quotas and `kubectl top` checks. This also demonstrates platform thinking: ephemeral, reproducible environments created and deleted on demand, cost-aware.

### 4. What is your standard for naming and tags, and how do you enforce it?
**Answer.** `<org>-<env>-<region>-<component>` plus mandatory tags (`owner`, `service`, `env`, `cost-center`, `managed-by`); enforced in Terraform validation, Conftest on plans, Azure Policy/tag policies, Kyverno for labels, and Backstage scaffolding that sets them automatically. Enforced standards are only the ones tooling checks.

### 5. Why disable the default CNI in the lab and install Cilium?
**Answer.** kindnet does not enforce NetworkPolicy; to practise default-deny, FQDN policies and Hubble flows you need a policy-capable CNI. It mirrors the production choice (Azure CNI Overlay with Cilium dataplane) and shows CNI as a day-0 decision that is hard to swap later.
