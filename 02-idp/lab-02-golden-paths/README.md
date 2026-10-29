# Lab 02-02 - Golden paths: define, verify, support, deviate

## Goal
Define the catalogue of golden paths with an explicit **contract**, write automated conformance checks (so "on the path" is measurable, not tribal), establish support tiers, and a deviation (exception) process with an expiry.

## Prereqs
Previous modules' repos. `yq`, `jq`, `gh`.

## Step 1 - The catalogue (what paths exist)
| Path | For | Stack | Includes | Support tier |
|---|---|---|---|---|
| `java-spring-aks` | HTTP/gRPC services | Spring Boot 3, Postgres | repo, CI, image signing, ArgoCD app, Terraform module, dashboards, alerts, docs | Gold (platform on-call for the path's components) |
| `node-api-aks` | HTTP APIs | Node 20, Postgres/Redis | same | Gold |
| `python-worker-aks` | queue/batch workers | Python, KEDA | same + queue scaler | Silver |
| `static-web` | SPAs | any | CDN, CI, preview | Silver |
| `bring-your-own` | anything else | any | minimum contract only | Bronze (self-supported) |
Rule: a new path requires demand from 3+ teams, an owner, a maintenance budget and an exit plan. Fewer paths, better supported.

## Step 2 - The path contract (what "on the path" means)
Write `contract.md` for each path with **MUST / SHOULD / MAY**:
MUST (enforced): repo has `catalog-info.yaml` with Group owner; CI uses `ci-*` and `build-image` platform workflows at a supported major; images signed and deployed by digest via GitOps; Kubernetes manifests via golden chart or approved overlay; resource requests/limits; liveness/readiness; standard labels; secrets via ESO.
SHOULD: SLO defined; dashboards; runbook; canary rollout for tier-1.
MAY: language/framework choices inside the language stack, extra Helm values, extra sidecars with approval.
This is what separates a golden path from a cage: strict on interfaces and security, flexible inside the app.

## Step 3 - Conformance checks as code
Check a service repo against the contract, by hand first.
```bash
cd payments-api
echo "== catalog owner group"; yq '.spec.owner' catalog-info.yaml | grep -q '^group:' && echo PASS || echo FAIL
echo "== uses platform CI"; grep -rqE 'acme-platform-workflows/.github/workflows/ci-' .github/workflows && echo PASS || echo FAIL
echo "== workflow pins supported major"; grep -rhoE 'acme-platform-workflows/.github/workflows/[a-z-]+\.yml@v[0-9]+' .github/workflows | sed 's/.*@//' | sort -u
echo "== has docs"; test -f mkdocs.yml && test -d docs && echo PASS || echo FAIL
echo "== CODEOWNERS"; test -f CODEOWNERS -o -f .github/CODEOWNERS && echo PASS || echo FAIL
```
Turn each into a Tech Insights fact/check (Module 01 Lab 05) or a nightly workflow in the platform repo that scans the org and writes results to a JSON artifact / the catalog. Cluster-side checks with Kyverno policies (Module 04) and GitOps repo checks (`kubeconform`, policy tests). Output = **path conformance %** per team and per path, trending up.
Deployment-side evidence (Prometheus/Argo): images deployed by digest, signed, resources set:
```bash
kubectl get deploy -A -o json | jq -r '.items[] | select(.spec.template.spec.containers[].resources.limits.memory==null) | .metadata.namespace+"/"+.metadata.name'
```

## Step 4 - Make the path the *easy* path (not just the mandated one)
Measure and optimise "time to first deploy" on the path versus off the path:
```bash
# Example: lead time from repo creation to first successful dev deploy, using GitHub + ArgoCD events
gh api repos/$ORG/orders-api --jq .created_at
gh run list -R $ORG/orders-api --workflow ci --limit 5 --json conclusion,createdAt,updatedAt
```
Targets: first deploy < 60 min on path; off-path takes days (a natural incentive). Add benefits that only the path has: pre-wired observability, free on-call support, automatic dependency updates, default security exceptions already handled, cost visibility.

## Step 5 - Support tiers and responsibilities
| Tier | Platform commits to | Team commits to |
|---|---|---|
| Gold | on-call for platform components, upgrades done for you via PRs, SLOs | keep up with PRs within 2 weeks, own app reliability |
| Silver | best-effort support in business hours, upgrades via PRs | same |
| Bronze (BYO) | docs only, security baseline enforced | everything else, incl. upgrades and compliance evidence |
Write this in a one-page "platform contract" and publish in TechDocs.

## Step 6 - Deviation process (paved road, with exits)
1. Team opens an RFC issue from a template: need, why the path doesn't fit, which contract items are affected, owner, review date.
2. Platform + security review within 5 business days. Outcomes: (a) the path gets a feature (best), (b) approved exception with conditions, (c) decline.
3. Record as a catalog annotation and a policy exception with expiry:
```yaml
metadata:
  annotations:
    acme.io/path: java-spring-aks
    acme.io/deviation: "rfc-142: custom ingress for websockets; expires 2027-03-31"
```
4. Kyverno `PolicyException` or Terraform policy exception file (Module 05 Lab 05) with the same expiry; a weekly bot lists expiring exceptions to the owner and platform.
5. Track the **deviation rate** by reason: recurring reasons become roadmap items. If 30 % of deviations ask for websockets, the path is missing a feature.

## Step 7 - Evolving paths without breaking teams
Versioned paths (`java-spring-aks@v2`), changelog, migration guide, deprecation window, automated update PRs (Module 01 Lab 03), and consumer-visible roadmap. Keep a **path health review** quarterly: adoption %, conformance %, time-to-first-deploy, satisfaction, open deviations.

## Break it on purpose
1. Write a contract MUST that no existing service meets: observe 0 % conformance and demoralisation; fix by making it SHOULD with a date, or ship an automated fix PR first.
2. Approve an exception without expiry; six months later nobody remembers: add the expiry check workflow.

## Talking points
Few, supported paths; contract = interfaces + security strict, internals flexible; conformance measured by code; path wins on convenience; tiers define responsibilities; deviations expire and feed the roadmap.
