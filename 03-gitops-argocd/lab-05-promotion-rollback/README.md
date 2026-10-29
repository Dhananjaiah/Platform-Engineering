# Lab 03-05 - Promotion across environments and rollback

## Goal
Implement three promotion mechanisms (PR-based, Image Updater, Kargo), then rehearse rollback with `git revert` versus Argo history rollback.

## Prereqs
Labs 02-04; `payments` has `dev`, `stg`, `prd` overlays pinned by digest.

## Step 1 - PR-based promotion (the default I recommend)
Pipeline step (Module 06 `gitops-bump`) updates `overlays/dev` automatically (auto-merge). Promotion to stg/prd copies the *exact digest* from the lower env:
```bash
cd acme-gitops
DIGEST=$(yq '.images[0].digest' apps/payments/overlays/dev/kustomization.yaml)
yq -i '.images[0].digest = "'$DIGEST'"' apps/payments/overlays/stg/kustomization.yaml
git checkout -b promote/payments-stg-$(date +%s) && git commit -am "promote payments dev->stg" && git push -u origin HEAD
gh pr create -t "Promote payments to stg" -b "Digest $DIGEST, verified in dev for 24h"
```
Gates: required checks (kubeconform render, Kyverno CLI policy test, signature verify), CODEOWNERS approval for `overlays/prd`, and a status check "digest exists and is signed" using `cosign verify`. This is a *workflow*, so wrap it as a reusable `promote.yml` dispatched by a button in Backstage.
Pros: audit trail, approvals, policy. Cons: more steps; mitigate with auto-merge for stg when dev is healthy.

## Step 2 - ArgoCD Image Updater (automatic, per-app rules)
```bash
helm install argocd-image-updater argo/argocd-image-updater -n argocd
kubectl -n argocd annotate app payments-dev \
  argocd-image-updater.argoproj.io/image-list=payments=ghcr.io/acme-lab-<you>/payments-api \
  argocd-image-updater.argoproj.io/payments.update-strategy=digest \
  argocd-image-updater.argoproj.io/payments.allow-tags=regexp:^main$ \
  argocd-image-updater.argoproj.io/write-back-method=git
```
(Newer chart versions move to a CRD `ImageUpdater`; follow the chart notes.) Good for dev/stg that follow a tag; for prod prefer explicit PR promotion. Cons: write-back commits to git with its own credentials, tag-based flow weakens digest pinning, less approval control.

## Step 3 - Kargo (promotion as a first-class pipeline)
```bash
helm install kargo oci://ghcr.io/akuity/kargo-charts/kargo -n kargo --create-namespace \
  --set api.adminAccount.passwordHash='<bcrypt>' --set api.adminAccount.tokenSigningKey='<random>'
```
Define `Warehouse` (watches the registry), `Stage`s dev -> stg -> prd with `promotionTemplate` steps (`git-clone`, `kustomize-set-image`, `git-commit`, `git-push`, `argocd-update`) and verification (`AnalysisTemplate`). Kargo gives freight (artifact bundle) tracking, a UI of what is where, and gating. Compare: PR promotion = simplest and auditable; Kargo = best UX/visibility at scale, extra control plane to run.

## Step 4 - Rollback strategy
Scenario: prd deploy of digest B is bad; A was good.
Option 1 (Argo history rollback):
```bash
argocd app history payments-prd
argocd app rollback payments-prd <ID>      # fails if automated sync is on
```
It requires disabling auto-sync, creates drift from git, and someone must reconcile git later. Use only in a true emergency with a note.
Option 2 (git revert, preferred):
```bash
git revert <bump-commit-sha> --no-edit && git push origin main    # or via emergency PR with fast-track approval
argocd app sync payments-prd
```
Git stays the truth, audit trail intact, works for every cluster using that path. Make it fast: a pre-approved `rollback.yml` workflow that opens and auto-merges the revert PR for on-call, with a protected "emergency" environment.
Option 3 (progressive delivery abort): Argo Rollouts aborts automatically on failed analysis (Lab 06): best, because the bad version never reaches 100 %.
State all three; recommend 3 then 2.

## Step 5 - Drill: bad release
```bash
yq -i '.images[0].digest = "sha256:0000000000000000000000000000000000000000000000000000000000000000"' apps/payments/overlays/dev/kustomization.yaml
git commit -am "bad digest" && git push
kubectl -n payments-dev get pods -w     # ImagePullBackOff
argocd app get payments-dev              # Health: Degraded
git revert HEAD --no-edit && git push
argocd app wait payments-dev --health
```
Time it. Target: detection < 2 min (alert on Degraded/Progressing > 5 min via notifications), rollback < 5 min.

## Break it on purpose
Promote while dev is red: add a check in the promotion workflow that reads the dev app health via `argocd app get -o json | jq -r .status.health.status` and refuses unless `Healthy` for N minutes.

## Cleanup
`helm uninstall argocd-image-updater -n argocd`; `helm uninstall kargo -n kargo`.

## Talking points
Promote the same digest, not rebuilds; PR promotion for audit/approval; revert is the rollback of record; automation lowers MTTR; the bad version should be stopped by progressive delivery before humans notice.
