# Lab 03-02 - Applications, sync policy, waves, hooks, health, diff customization

## Goal
Deploy a real app with an `Application`, then exercise automated sync/self-heal/prune, sync waves with a DB migration hook, a custom health check, and `ignoreDifferences` for an HPA-managed field.

## Prereqs
Lab 01. Create `acme-gitops` structure:
```bash
git clone git@github.com:acme-lab-<you>/acme-gitops.git && cd acme-gitops
mkdir -p apps/payments/base apps/payments/overlays/dev
```

## Step 1 - Base manifests (Kustomize)
```bash
cat > apps/payments/base/deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata: { name: payments }
spec:
  replicas: 2
  selector: { matchLabels: { app: payments } }
  template:
    metadata: { labels: { app: payments } }
    spec:
      containers:
        - name: payments
          image: ghcr.io/acme-lab-<you>/payments-api
          ports: [{ containerPort: 8080 }]
          readinessProbe: { httpGet: { path: /, port: 8080 } }
          resources: { requests: { cpu: 50m, memory: 64Mi }, limits: { memory: 128Mi } }
EOF
cat > apps/payments/base/service.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata: { name: payments }
spec: { selector: { app: payments }, ports: [{ port: 80, targetPort: 8080 }] }
EOF
cat > apps/payments/base/kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources: [deployment.yaml, service.yaml]
EOF
cat > apps/payments/overlays/dev/kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: payments-dev
resources: [../../base]
images:
  - name: ghcr.io/acme-lab-<you>/payments-api
    digest: sha256:REPLACE_WITH_YOUR_DIGEST
EOF
kubectl kustomize apps/payments/overlays/dev | head -30
git add -A && git commit -m "payments base and dev overlay" && git push
```
`kubectl kustomize` renders locally: always render before committing. Pull a real digest from your Module 06 build, or temporarily use `nginx` + digest.

## Step 2 - The Application
```bash
kubectl create namespace payments-dev
cat <<'EOF' | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payments-dev
  namespace: argocd
  finalizers: [resources-finalizer.argocd.argoproj.io]   # delete app => delete its resources
spec:
  project: payments
  source:
    repoURL: git@github.com:acme-lab-<you>/acme-gitops.git
    targetRevision: main
    path: apps/payments/overlays/dev
  destination: { server: https://kubernetes.default.svc, namespace: payments-dev }
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=false, PruneLast=true, ServerSideApply=true]
    retry: { limit: 5, backoff: { duration: 10s, factor: 2, maxDuration: 3m } }
EOF
argocd app get payments-dev
argocd app wait payments-dev --health --timeout 120
```
Explain each field aloud: `prune` deletes resources removed from git, `selfHeal` reverts manual cluster edits, `ServerSideApply` avoids annotation size limits and field ownership conflicts, `finalizer` cascades deletion.

## Step 3 - Prove self-heal and drift
```bash
kubectl -n payments-dev scale deploy/payments --replicas=5
kubectl -n payments-dev get deploy payments -w     # returns to 2 within seconds
argocd app history payments-dev
```
Now `argocd app set payments-dev --sync-policy none` and scale again: app shows `OutOfSync`; `argocd app diff payments-dev` prints the drift. Re-enable automation after.

## Step 4 - Sync waves and hooks (ordering a migration)
```bash
cat > apps/payments/base/migrate-job.yaml <<'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: payments-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation,HookSucceeded
spec:
  backoffLimit: 1
  template:
    spec:
      restartPolicy: Never
      containers:
        - { name: migrate, image: busybox:1.36, command: ["sh","-c","echo migrating; sleep 5; echo done"] }
EOF
cat > apps/payments/base/cm-config.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: payments-config
  annotations: { argocd.argoproj.io/sync-wave: "-1" }   # applied before the Deployment (wave 0)
data: { LOG_LEVEL: info }
EOF
sed -i 's/resources: \[deployment.yaml, service.yaml\]/resources: [deployment.yaml, service.yaml, migrate-job.yaml, cm-config.yaml]/' apps/payments/base/kustomization.yaml
git add -A && git commit -m "add presync migration and wave" && git push
argocd app sync payments-dev --async && argocd app get payments-dev --refresh
```
Order: PreSync hooks -> waves (-1, 0, 1...) each waiting for health -> PostSync hooks. Interview point: migrations as PreSync hooks must be idempotent and backward compatible (expand/contract), because rollbacks do not roll back the schema.
Break: make the Job `exit 1`; the sync stops at PreSync and the Deployment is untouched, which is the desired safety behavior.

## Step 5 - Custom health check (for a CRD that Argo cannot interpret)
```bash
kubectl -n argocd patch configmap argocd-cm --type merge -p '{"data":{"resource.customizations.health.certmanager.io_Certificate":"hs = {}\nif obj.status ~= nil and obj.status.conditions ~= nil then\n  for _, c in ipairs(obj.status.conditions) do\n    if c.type == \"Ready\" and c.status == \"False\" then hs.status = \"Degraded\" hs.message = c.message return hs end\n    if c.type == \"Ready\" and c.status == \"True\" then hs.status = \"Healthy\" hs.message = c.message return hs end\n  end\nend\nhs.status = \"Progressing\"\nhs.message = \"Waiting for certificate\"\nreturn hs\n"}}'
```
(cert-manager health is built in on recent versions; use this pattern for your own CRDs, e.g. Crossplane claims.) Why: a waves/health chain only works if every resource reports truthful health.

## Step 6 - Diff customization (stop fighting other controllers)
HPA changes `spec.replicas`; Argo with selfHeal would fight it. Tell Argo to ignore that field:
```bash
kubectl -n argocd patch app payments-dev --type merge -p '{"spec":{"ignoreDifferences":[{"group":"apps","kind":"Deployment","jsonPointers":["/spec/replicas"]}]}}'
```
Better: remove `replicas` from the manifest when an HPA owns it. Cluster-wide defaults via `resource.customizations.ignoreDifferences.<group_kind>` in `argocd-cm`. Mutating webhooks adding fields (e.g. Istio sidecar) need `managedFieldsManagers`/`jqPathExpressions`.

## Break it on purpose
1. Commit an invalid manifest; sync fails with a rendering error: `argocd app get` shows `ComparisonError`; the cluster is untouched.
2. Delete the Deployment manually: self-heal recreates it.
3. Remove `Deployment` from git with `prune: false` first: resource stays as orphan; then set prune true.

## Cleanup
Keep `payments-dev` for later labs.

## Talking points
Auto-sync/prune/selfHeal semantics, SSA, hooks for migrations, waves for ordering, accurate health, ignoreDifferences only for fields owned by other controllers.
