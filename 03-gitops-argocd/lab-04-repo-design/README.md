# Lab 03-04 - Config repo design: monorepo vs polyrepo, Helm vs Kustomize, values hierarchy

## Goal
Compare three layouts with working renders, pick one, and prove the choice with `diff` between environments.

## Prereqs
`helm`, `kustomize` (`kubectl kustomize` works too), `acme-gitops` clone.

## Step 1 - App repo vs config repo (the rule)
App repo: code, Dockerfile, tests, CI. Config repo: manifests/values/overlays. Reasons: (1) CI commits to config would retrigger app CI, (2) different permissions (devs open PRs to config; only CI/promoters merge to prod paths), (3) audit of "what is deployed" is one history, (4) rollback = revert in config without touching source.

## Step 2 - Layout A: Kustomize base + overlays (shown in Lab 02)
```
apps/payments/base/
apps/payments/overlays/{dev,stg,prd}/
```
Render and compare environments:
```bash
diff <(kubectl kustomize apps/payments/overlays/dev) <(kubectl kustomize apps/payments/overlays/prd)
```
Add `prd` overlay with replicas 4, PDB and HPA patches:
```bash
mkdir -p apps/payments/overlays/prd && cat > apps/payments/overlays/prd/kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: payments-prd
resources: [../../base, pdb.yaml]
patches:
  - target: { kind: Deployment, name: payments }
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 4
images:
  - name: ghcr.io/acme-lab-<you>/payments-api
    digest: sha256:REPLACE
EOF
cat > apps/payments/overlays/prd/pdb.yaml <<'EOF'
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: payments }
spec: { minAvailable: 2, selector: { matchLabels: { app: payments } } }
EOF
```
Pros: plain YAML, easy review, patches explicit. Cons: no templating logic, copy/paste across services.

## Step 3 - Layout B: one platform Helm chart, per-env values
```bash
mkdir -p charts/service/templates && cd charts/service
cat > Chart.yaml <<'EOF'
apiVersion: v2
name: service
version: 1.0.0
EOF
cat > values.yaml <<'EOF'
image: { repository: "", digest: "" }
replicas: 2
port: 8080
resources: { requests: { cpu: 50m, memory: 64Mi }, limits: { memory: 128Mi } }
pdb: { enabled: false }
EOF
cat > templates/deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
  labels: { app: {{ .Release.Name }}, owner: {{ required "owner is required" .Values.owner }} }
spec:
  replicas: {{ .Values.replicas }}
  selector: { matchLabels: { app: {{ .Release.Name }} } }
  template:
    metadata: { labels: { app: {{ .Release.Name }} } }
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}@{{ .Values.image.digest }}"
          ports: [{ containerPort: {{ .Values.port }} }]
          resources: {{ toYaml .Values.resources | nindent 12 }}
EOF
cd ../..
helm lint charts/service --set owner=team-payments --set image.repository=x --set image.digest=sha256:abc
helm template payments charts/service -f apps/payments/values/base.yaml -f apps/payments/values/prd.yaml
```
Values hierarchy (later wins): chart defaults < `values/base.yaml` < `values/<env>.yaml` < (never) CLI overrides. In ArgoCD:
```yaml
source:
  repoURL: git@github.com:acme-lab-<you>/acme-gitops.git
  path: charts/service
  helm:
    releaseName: payments
    valueFiles: [../../apps/payments/values/base.yaml, ../../apps/payments/values/prd.yaml]
```
Pros: one chart enforces org standards for 200 services (the golden path chart); `required` fails fast. Cons: chart becomes a mini-platform to maintain; values sprawl; template debugging harder; `helm template` output is what Argo applies (no Helm hooks/lookup caveats).

## Step 4 - Layout C: Helm for third-party, Kustomize for ours, plus Helm-inflated Kustomize
Third-party charts (ingress, cert-manager) are consumed as Helm with pinned versions in Applications; own services use Kustomize or the golden chart. When you must patch a third-party chart output: Kustomize `helmCharts:` inflate, then patches (requires `--enable-helm` in Argo's kustomize build options).

## Step 5 - Monorepo vs polyrepo for the config
| | Monorepo (`acme-gitops`) | Repo per team/app |
|---|---|---|
| Discoverability, global refactors | great | poor |
| Access control | CODEOWNERS + path rules | native repo permissions |
| PR noise, CI scale | grows; needs path filters | isolated |
| ArgoCD refresh load | webhook + path annotations required | spread |
| Typical choice | platform-owned repo with `apps/`, `clusters/`, `addons/` | large orgs where teams own prod config |
Recommendation: monorepo for platform config + per-team repos only when team autonomy beats uniformity. Use ArgoCD webhook and `argocd.argoproj.io/manifest-generate-paths` annotation to avoid re-rendering everything on every commit.
```bash
kubectl -n argocd patch app payments-dev --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/manifest-generate-paths":"/apps/payments"}}}'
```

## Step 6 - Directory structure to defend
```
acme-gitops/
  clusters/{dev,prd}/        # per-cluster: addons list, cluster-scoped config
  addons/                    # kyverno, cert-manager, ingress (versions pinned here)
  apps/<service>/{base,overlays/<env>}
  charts/service/            # golden chart (optional)
  bootstrap/                 # root app-of-apps + ApplicationSets
  CODEOWNERS
```
CODEOWNERS: `/apps/payments/ @acme/team-payments`, `/apps/*/overlays/prd/ @acme/platform @acme/sre` (prod promotion requires platform/SRE).

## Break it on purpose
1. Remove `owner` from values: `helm template` fails on `required`.
2. Edit the Helm chart's default `replicas` and see 200 services change at once: argue for chart versioning (`targetRevision` of chart), so behaviour changes roll out intentionally.

## Talking points
Kustomize = explicit, simple; Helm = reuse and enforcement; use both deliberately; repo separation reasons; CODEOWNERS for prod paths; manifest-generate-paths for performance.
