# Module 06 - GitHub Actions / CI-CD platform: scenario-based production Q&A

---

### 1. A popular third-party action is compromised and republishes a malicious `v4` tag
**Answer.** Immediate: check which repos reference it (`gh search code`), identify runs in the window, rotate every secret those jobs could read, revoke OIDC trust temporarily for affected repos, disable the action via the allow-list (org-level). Why impact was limited (or not): we pin by SHA, so unchanged pins are unaffected; if anyone used tags, they ran it. Prevent: SHA pinning enforced (allow-list + required workflow with `zizmor`), Renovate to bump pins with review, least-privilege tokens, OIDC (no long-lived secrets to steal), egress controls on runners. Post-incident: inventory and notify owners, add detection (audit log for allow-list changes).

### 2. Build times for 300 repos doubled after a platform release
**Answer.** Roll back by moving the `v1` tag to the previous good release (the major tag is the lever), then root-cause on a canary. Use Lab 05 metrics: did duration, queue time or cache hit-rate change? Typical culprit: a new scan step without caching its DB, cache key change causing misses, or runner capacity. Fix forward behind `v1-beta`, canary with 5 repos, then promote. Lesson to state: release process for the pipeline itself, with canary + SLO on duration.

### 3. Developers complain the secure pipeline is slow and want to skip scans
**Answer.** Find the data: which step costs time. Make scans fast, not optional: cache Trivy DB, scan only changed layers, run SAST in parallel with tests, run deep scans nightly while PR path does the quick gate, use diff-aware Semgrep. Provide an exception path with expiry and owner. Report to the VP: 'scans add 90s p50; we removed 4 vulnerabilities from prod last quarter'. Never agree to a silent bypass.

### 4. How do you roll out a new mandatory check to 500 repos without breaking everyone?
**Answer.** Ruleset in `evaluate` mode first (insights show who would fail), publish a list to owners with fixes, ship auto-fix PRs (codemod/Renovate) for the common case, announce date, flip to `active` for new repos immediately and old repos in waves, keep a time-boxed bypass list. Metrics: % compliant weekly. Staff skill: sequencing and communication, not just enforcement.

### 5. Deployment to prod was triggered by a PR from a fork. What happened and how to stop it?
**Answer.** Probably `pull_request_target` or a `workflow_run` chain that checks out untrusted code with secrets, or a self-hosted runner open to forks. Contain: disable the workflow, rotate secrets, review runs. Prevent: never run PR-head code under `pull_request_target`; require approval for outside collaborators' workflows; deploy jobs only on `push` to protected branches and via environments with required reviewers; self-hosted runners only for private trusted repos; OIDC subject restricted to `environment:prd` so forks cannot mint tokens.

### 6. Explain OIDC federation and why it beats secrets
**Answer.** The job requests a short-lived JWT from GitHub (claims: repo, ref, environment, workflow ref); the cloud validates signature against GitHub's JWKS and the trust policy conditions on `sub`/`aud`, exchanging it for a short-lived access token. No stored credential to leak, rotate or forget; scope is per repo/environment; audit shows the identity. Pitfall: loose `sub` conditions. Mitigation: pin repo + environment, use `job_workflow_ref` to trust only the platform workflow.

### 7. Reusable workflow vs composite action vs starter workflow: when?
**Answer.** Composite: bundle steps used within many jobs (own no runner/permissions). Reusable workflow: whole pipelines/jobs with runner, permissions and secrets contract, centrally versioned, appears in required workflows. Starter workflow: a template copied into a new repo (then drifts). So: golden path = reusable workflow called by a ~10-line caller scaffolded by Backstage; starter workflows only for bootstrapping the caller. Avoid copy-paste because you cannot fix 500 copies.

### 8. Self-hosted runners: the queue is 20 minutes every morning at 9:00
**Answer.** Check `maxRunners` and node capacity; look at ARC listener logs and pending pods (`kubectl get pods -n arc-runners`, `describe` for unschedulable); cluster autoscaler lag? Fix: warm pool with scheduled `minRunners`, node pool headroom/overprovisioning pods, cache and image pre-pull, split heavy jobs across runner sizes, set per-team concurrency fairness via runner groups. Verify with p95 queue time alert before/after.

### 9. Supply chain: an auditor asks you to prove what is running in prod came from reviewed code
**Answer.** Chain of evidence: protected branch + required reviews -> hosted build by the platform workflow -> image digest signed keyless with the workflow identity -> SLSA provenance and SBOM attestations -> GitOps commit references the digest -> admission policy (Kyverno `verifyImages`) rejects unsigned or off-platform images -> ArgoCD sync history. Demonstrate with `cosign verify`, `gh attestation verify` and `kubectl get` of the running digest. Call out limits: SLSA level reached, what is not covered (base image trust, dependencies mitigated by SBOM + scanning).

### 10. A team wants to deploy straight from CI with `kubectl apply` using a kubeconfig secret
**Answer.** Decline the pattern, offer the better path: CI builds and writes to GitOps; ArgoCD deploys. Reasons: no cluster credentials in CI, audit trail in git, drift correction, rollback by revert, consistent multi-cluster. Meet their need: if the pain is speed, show dev auto-sync deploys within a minute; if pain is control, show promotion PRs. Only exceptions: bootstrap/break-glass.

### 11. Monorepo with 40 services: CI runs everything on every PR (45 min)
**Answer.** Path filters/`paths` per workflow, dynamic matrix from changed-files (`git diff` based job outputs), per-service reusable workflow calls, remote build cache (Nx/Bazel/Turborepo/Gradle), test sharding, required status check via an aggregator job (`ci-gate`) so the ruleset does not need to know the dynamic matrix. Measure: p95 PR duration per touched service.

### 12. Flaky tests are blocking merges and people just click "re-run"
**Answer.** Quantify (same-SHA fail/pass rate, Lab 05), rank offenders, auto-quarantine with owner and SLA, retry only at test level with a report (not whole-pipeline reruns), fix root causes (time, ordering, shared state, network), publish a weekly flake leaderboard per team. Culture: flake budget; blocking merges on trunk-health.

### 13. Secrets: a dev committed an AWS key
**Answer.** Revoke/rotate immediately (the key is compromised regardless of history rewrite), check CloudTrail for use, then remove from history if needed. Prevent: push protection + secret scanning org-wide (`gh api -X PATCH /orgs/$ORG -f secret_scanning_push_protection_enabled_for_new_repositories=true` and repo-level), pre-commit hooks, OIDC so no keys exist, short-lived creds. Report as a learning, not blame.

### 14. How do you version and release the pipeline platform itself?
**Answer.** Semver tags + moving major, changelog, canary tag for early adopters, rulesets to protect tags, deprecation windows, telemetry on which repos call which version, a test suite (sample repos run against the candidate in CI via `act` or real repos), and a rollback by moving the major tag back. Treat pipeline code like a product API.

### 15. Cost of Actions is up 60% this quarter
**Answer.** Break down by repo/workflow/runner type from billing API; find: matrix explosions, un-cancelled superseded runs (add `concurrency: cancel-in-progress` on PRs), long-running jobs without timeouts, scheduled jobs nobody owns, macOS minutes, cache misses. Move heavy Linux builds to ARC on spot nodes. Show back per team. Targets: -30 % without slower feedback.

### 16. Required status check names change with a matrix; PRs get stuck "waiting for status"
**Answer.** Required checks match job names; renaming or dynamic matrices break them. Use one stable aggregator job (`ci-gate` with `needs: [...]` and `if: always()` checking results) as the sole required check; keep the name stable and part of the platform contract.

### 17. Explain how you would prevent privilege escalation through reusable workflows
**Answer.** Permissions are a ceiling set by the caller; the called workflow cannot exceed it. Restrict who may modify the platform repo (CODEOWNERS, rulesets, 2 reviewers), require signed commits, immutable release tags, allow-list the repo as the only source for reusable workflows org-wide, forbid `secrets: inherit` in callers except for trusted internal flows, and use environments for anything that touches prod.

### 18. Your migration from Jenkins stalled at 60%
**Answer.** Diagnose using the inventory: the remaining 40 % are special cases and politically sensitive teams. Actions: cost of Jenkins made visible, executive sponsor sets date, dedicated migration squad for long-tail, 'white-glove' for blocked teams, feature gap list with workarounds, and honesty about the jobs that should be deleted rather than migrated. Report weekly burn-down; remove Jenkins capabilities (new job creation freeze) to create pull toward the new path.
