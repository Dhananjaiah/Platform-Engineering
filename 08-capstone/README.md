# 08 - Capstone: the end-to-end golden path (hands-on, command by command)

You will run the full flow from the brief, by hand, on the lab platform built in Modules 00-07, break it three ways, then rehearse the whiteboard and the 15-minute demo.

```
1 Developer opens Backstage, picks the golden-path template
2 Scaffolder creates GitHub repo (code, catalog-info.yaml, reusable workflows)
3 Template opens infra PR (Terraform) and GitOps PR (ArgoCD app) 
4 CI builds, tests, scans, signs, pushes image, updates the GitOps repo
5 ArgoCD (ApplicationSet) deploys to dev, promotion PRs to stg/prd, Argo Rollouts canary
6 Backstage shows owner, docs, deployments, health, cost, scorecard
7 DORA + developer-experience metrics drive the roadmap
```

## Preconditions checklist (verify each; each line is a command)
```bash
kubectl --context kind-hub get nodes                                  # hub up (Module 00)
kubectl --context kind-hub -n argocd get applications                 # ArgoCD up (Module 03)
kubectl --context kind-hub -n backstage get pods                      # Backstage + Postgres up (Module 01)
kubectl --context kind-hub -n arc-runners get pods 2>/dev/null        # runners (optional, Module 06 Lab 04)
gh repo view acme-lab-<you>/acme-platform-workflows --json name       # workflows (Module 06)
gh repo view acme-lab-<you>/acme-templates --json name                # template (Module 01 Lab 03)
kubectl --context kind-hub get clusterpolicy                          # Kyverno (Module 04 Lab 03)
```
If any fails, go back to that module's lab. A capstone that depends on a missing piece is the point: you learn the dependency graph.

## Step 1 - Create the service from the portal (minutes 0-5)
1. Open `http://backstage.127.0.0.1.nip.io:8080`, sign in with GitHub.
2. Create -> "Java Spring service on AKS" -> name `capstone-api`, owner `team-payments`, system `checkout`, database yes, cost center `cc-1234`, repo `acme-lab-<you>/capstone-api`.
3. Watch the task log. Expected steps: fetch -> publish repo -> platform registration -> GitOps PR -> infra PR -> catalog register.
Verify (commands):
```bash
gh repo view acme-lab-<you>/capstone-api --json name,visibility
gh pr list -R acme-lab-<you>/acme-gitops --search "onboard capstone-api"
gh pr list -R acme-lab-<you>/acme-infra --search "capstone-api"
curl -s "http://backstage.127.0.0.1.nip.io:8080/api/catalog/entities/by-name/component/default/capstone-api" -H "Authorization: Bearer $BACKSTAGE_TOKEN" | jq '.spec'
```
Explain while waiting: why PRs for infra and gitops (review, policy, audit), why a thin skeleton (drift), what the catalog annotations enable.

## Step 2 - Review and merge the PRs (minutes 5-8)
```bash
gh pr checks <infra-pr-number> -R acme-lab-<you>/acme-infra --watch
gh pr view <infra-pr-number> -R acme-lab-<you>/acme-infra --comments       # plan comment, policy results
gh pr merge <infra-pr-number> -R acme-lab-<you>/acme-infra --squash
gh pr merge <gitops-pr-number> -R acme-lab-<you>/acme-gitops --squash
```
Infra: `tf-plan` ran fmt/validate/Conftest, posted the plan, merge triggers `tf-apply` in the `dev` environment (OIDC; reviewers). If you can't apply to a cloud account, skip the apply and show the plan output and policy results; or compose Postgres via the Crossplane claim (Module 02 Lab 03) by putting that claim in the GitOps PR instead.
GitOps: the ApplicationSet (git directory generator) discovers `apps/capstone-api/overlays/dev` and creates the Application.
```bash
kubectl --context kind-hub -n argocd get applications | grep capstone
argocd app get capstone-api-dev
```

## Step 3 - CI: build, scan, sign, push, hand off (minutes 8-12)
Make a code change in the new repo and push to `main`:
```bash
gh repo clone acme-lab-<you>/capstone-api && cd capstone-api
echo "// capstone $(date +%s)" >> src/main/java/Application.java 2>/dev/null || echo "// capstone" >> README.md
git add -A && git commit -m "feat: first change" && git push
gh run watch -R acme-lab-<you>/capstone-api
```
Expected pipeline jobs: `ci` (build/test/lint) -> `image` (Buildx push, Trivy gate, SBOM, Cosign sign, provenance) -> `gitops-bump` opens/merges a PR to `acme-gitops` updating the dev digest.
Verify evidence:
```bash
DIGEST=$(gh api /orgs/acme-lab-<you>/packages/container/capstone-api/versions --jq '.[0].name')
cosign verify ghcr.io/acme-lab-<you>/capstone-api@$DIGEST \
  --certificate-identity-regexp '^https://github.com/acme-lab-<you>/acme-platform-workflows/.github/workflows/build-image.yml@refs/tags/v1.*' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com | head -c 400
gh attestation verify oci://ghcr.io/acme-lab-<you>/capstone-api@$DIGEST -R acme-lab-<you>/capstone-api
git -C ../acme-gitops pull && yq '.images[0].digest' apps/capstone-api/overlays/dev/kustomization.yaml
```

## Step 4 - Deploy with ArgoCD and progressive delivery (minutes 12-15)
```bash
argocd app sync capstone-api-dev
argocd app wait capstone-api-dev --health --timeout 180
kubectl --context kind-dev -n capstone-api-dev get pods,rollout
kubectl argo rollouts get rollout capstone-api -n capstone-api-dev
```
Promote to stg then prd using the promotion workflow (Module 03 Lab 05): copies the digest, opens a PR; show the checks (render, policy, signature) and CODEOWNERS gate for prd:
```bash
gh workflow run promote.yml -R acme-lab-<you>/acme-gitops -f service=capstone-api -f from=dev -f to=stg
gh pr list -R acme-lab-<you>/acme-gitops
```
During the prd canary: `kubectl argo rollouts get rollout capstone-api -n capstone-api-prd -w`, show analysis runs passing.

## Step 5 - Observe it all in Backstage (minutes 15-18)
Entity page for `capstone-api`: Overview (owner, system, links), CI (GitHub Actions), Kubernetes (pods, rollout), ArgoCD (sync status), Docs, Dependencies graph (Postgres Resource), PagerDuty (on-call), Cost, Scorecard.
```bash
curl -s "$BACKSTAGE/api/tech-insights/checks/..." 2>/dev/null | head      # or open the Scorecard tab
```
Open Grafana golden dashboard for `capstone-api`; show SLO burn-rate panel; show the DORA dashboard row for `capstone-api` (Module 02 Lab 04 query).

## Step 6 - Failure injection (manual commands; practise narrating)
**F1: Policy blocks a bad deploy.** Try an unsigned/`latest` image:
```bash
kubectl --context kind-dev -n capstone-api-dev set image deploy/capstone-api app=nginx:latest 2>&1 | head -5
kubectl --context kind-dev -n capstone-api-dev run bad --image=docker.io/library/nginx:latest 2>&1 | head
```
Expected: Kyverno denies (registry + tag + signature), and ArgoCD self-heal reverts a manual edit. Narrate: guardrails at admission, GitOps correction, and the signed-image chain.
**F2: Canary aborts automatically.** Push a change that makes the service return 5xx on 30 % of requests (or commit a bad digest of a faulty build):
```bash
kubectl argo rollouts get rollout capstone-api -n capstone-api-prd -w
kubectl -n capstone-api-prd get analysisrun
```
Expected: analysis fails, Rollout `Degraded`, traffic returns to stable. Then `git revert` the bump commit in `acme-gitops` and show ArgoCD converge. Time the sequence: detection < 2 minutes, mitigation automatic.
**F3: CI scan gate fails the build.** Add a dependency with a known HIGH CVE (or `FROM node:14`), push, show Trivy failing the job and the SARIF in the Security tab; fix by upgrading, or show an exception with an expiry in `.trivyignore`.
**F4 (bonus): Drift and rebuild.** `kubectl -n capstone-api-dev scale deploy/capstone-api --replicas=0` (if not a Rollout) shows self-heal; delete the whole app namespace and watch ArgoCD recreate.

## Step 7 - Measure and close the loop (minutes 18-20)
```bash
PGPASSWORD=... psql -h localhost -p 5433 -U app -d app -c "SELECT service, count(*) deploys, min(deployed_at) first_deploy FROM dora.deployments WHERE service='capstone-api' GROUP BY 1;"
gh api repos/acme-lab-<you>/capstone-api --jq .created_at     # time zero
```
Compute **time-to-first-deploy** = first successful dev deploy - template run time. Report: "from form to running in dev in N minutes; prod after approvals in M". Then state what the data suggests for next quarter's roadmap.

## Step 8 - Teardown (also a skill)
```bash
gh repo delete acme-lab-<you>/capstone-api --yes
kubectl --context kind-hub -n argocd delete application capstone-api-dev capstone-api-stg capstone-api-prd --ignore-not-found
# remove folders from acme-gitops and acme-infra via PR; terraform destroy of the dev stack via the pipeline or:
cd ~/acme-infra/live/dev/capstone-api && terragrunt destroy
```

## Whiteboard and demo materials
- [whiteboard.md](whiteboard.md): the diagram, per-step trade-offs and "what if X fails" answers.
- [demo-script.md](demo-script.md): a 15-minute spoken demo with timing and fallback plan.
- [Scenario interview Q&A](interview.md): end-to-end, cross-cutting questions.
