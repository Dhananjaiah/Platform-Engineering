# Lab 03-03 - ApplicationSets, app-of-apps, multi-cluster hub-and-spoke

## Goal
Register `dev` and `prd` kind clusters into the hub ArgoCD, generate Applications with Git directory, cluster, matrix, list and pull-request generators, and compare with app-of-apps.

## Prereqs
Labs 01-02; kind clusters `hub`, `dev`, `prd` (Module 00). Switch to the hub: `kubectl config use-context kind-hub`.

## Step 1 - Register spoke clusters
kind API servers are reachable from the hub through the Docker network. Get the internal address:
```bash
docker inspect dev-control-plane --format '{{ .NetworkSettings.Networks.kind.IPAddress }}'
kubectl config view --minify --context kind-dev --raw -o jsonpath='{.clusters[0].cluster.server}'
```
Use a kubeconfig whose server is `https://<that-IP>:6443`:
```bash
kind get kubeconfig --internal --name dev > /tmp/dev-internal.kubeconfig
argocd cluster add kind-dev --kubeconfig /tmp/dev-internal.kubeconfig --name dev --label env=dev --label tier=nonprod --yes
argocd cluster list
```
(`--internal` makes the server `https://dev-control-plane:6443`, resolvable by the hub's nodes because all kind clusters share the `kind` Docker network.)
(Repeat for `prd` with `--label env=prd --label tier=prod`.) `argocd cluster add` creates a ServiceAccount + ClusterRole in the spoke and stores its token as a Secret in the hub `argocd` namespace. Verify: `kubectl -n argocd get secrets -l argocd.argoproj.io/secret-type=cluster` and `argocd cluster list` shows `Successful`.
Production: use workload identity/exec auth (AKS: `argocd-k8s-auth` with Azure workload identity, EKS: IRSA) instead of long-lived tokens; scope the ClusterRole down from cluster-admin.

## Step 2 - Git directory generator (one Application per app folder)
```bash
mkdir -p apps/orders/base apps/orders/overlays/{dev,prd} && cp apps/payments/base/* apps/orders/base/ && \
 sed -i 's/payments/orders/g' apps/orders/base/*.yaml
```
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata: { name: apps-dev, namespace: argocd }
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
    - git:
        repoURL: git@github.com:acme-lab-<you>/acme-gitops.git
        revision: main
        directories:
          - path: apps/*/overlays/dev
  template:
    metadata:
      name: '{{ index .path.segments 1 }}-dev'
    spec:
      project: default
      source:
        repoURL: git@github.com:acme-lab-<you>/acme-gitops.git
        targetRevision: main
        path: '{{ .path.path }}'
      destination: { name: dev, namespace: '{{ index .path.segments 1 }}-dev' }
      syncPolicy:
        automated: { prune: true, selfHeal: true }
        syncOptions: [CreateNamespace=true]
EOF
kubectl -n argocd get applicationset,applications
```
Expected: an Application per folder: add a folder `apps/billing/overlays/dev` in git and a new Application appears without touching ArgoCD. That is the point: onboarding a service = a PR adding a directory.

## Step 3 - Cluster generator with labels (add-ons to every cluster)
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata: { name: addon-kyverno, namespace: argocd }
spec:
  goTemplate: true
  generators:
    - clusters:
        selector: { matchExpressions: [{ key: env, operator: In, values: [dev, prd] }] }
  template:
    metadata: { name: 'kyverno-{{ .name }}' }
    spec:
      project: default
      source:
        repoURL: https://kyverno.github.io/kyverno
        chart: kyverno
        targetRevision: 3.4.1
        helm: { releaseName: kyverno }
      destination: { server: '{{ .server }}', namespace: kyverno }
      syncPolicy:
        automated: { prune: true, selfHeal: true }
        syncOptions: [CreateNamespace=true, ServerSideApply=true]
EOF
```
This is how you install platform add-ons consistently on every cluster (Module 04). A new cluster with the right labels gets the add-on automatically.

## Step 4 - Matrix generator (apps x clusters) with environment values
```yaml
generators:
  - matrix:
      generators:
        - git:
            repoURL: git@github.com:acme-lab-<you>/acme-gitops.git
            revision: main
            directories: [{ path: "apps/*" }]
        - clusters:
            selector: { matchLabels: { tier: nonprod } }
template:
  metadata: { name: '{{ index .path.segments 1 }}-{{ .name }}' }
  spec:
    source:
      path: 'apps/{{ index .path.segments 1 }}/overlays/{{ .metadata.labels.env }}'
```
Apply as `apps-matrix` and note the explosion risk: N apps x M clusters. Use selectors and keep prod in a separate ApplicationSet with manual sync for safety.

## Step 5 - Pull-request generator (ephemeral preview environments)
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata: { name: payments-previews, namespace: argocd }
spec:
  goTemplate: true
  generators:
    - pullRequest:
        github:
          owner: acme-lab-<you>
          repo: payments-api
          labels: [preview]
          tokenRef: { secretName: github-token, key: token }
        requeueAfterSeconds: 120
  template:
    metadata: { name: 'payments-pr-{{ .number }}' }
    spec:
      project: payments
      source:
        repoURL: git@github.com:acme-lab-<you>/acme-gitops.git
        targetRevision: main
        path: apps/payments/overlays/dev
        kustomize: { nameSuffix: '-pr{{ .number }}', namespace: 'payments-pr-{{ .number }}' }
      destination: { name: dev, namespace: 'payments-pr-{{ .number }}' }
      syncPolicy: { automated: { prune: true }, syncOptions: [CreateNamespace=true] }
EOF
```
Label a PR `preview`; an environment appears; close the PR; it disappears (cleanup is the reason to use this generator). Use a GitHub App token, set TTL/quota on the preview namespaces.

## Step 6 - App-of-apps vs ApplicationSet
App-of-apps: a root Application whose path contains Application manifests.
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: { name: root, namespace: argocd }
spec:
  project: default
  source: { repoURL: git@github.com:acme-lab-<you>/acme-gitops.git, targetRevision: main, path: bootstrap/apps }
  destination: { server: https://kubernetes.default.svc, namespace: argocd }
  syncPolicy: { automated: { prune: true, selfHeal: true } }
EOF
```
| | App-of-apps | ApplicationSet |
|---|---|---|
| Generates from | hand-written Application YAML in git | generators (dirs, clusters, PRs, matrix) |
| Per-app customisation | easy | via template + generator params |
| Scale | each app a file | thousands from one definition |
| Good for | bootstrap order, heterogeneous apps | uniform fleets, multi-cluster |
Pattern: one app-of-apps root bootstraps the ApplicationSets (the root is the only thing applied by hand).

## Step 7 - Hub-and-spoke vs per-cluster ArgoCD
| | Hub (one ArgoCD) | ArgoCD per cluster |
|---|---|---|
| Ops burden | one install to upgrade/back up | N installs |
| Blast radius | hub outage stops deploys everywhere (running apps unaffected) | contained |
| Credentials | hub holds credentials to all clusters (high value target) | none leave the cluster |
| Network | needs reachability hub -> spokes (private clusters hard) | pull stays local |
| Visibility | single pane | need aggregation |
Decision: hub per environment tier (nonprod hub, prod hub) with workload identity; agent-based pull (Argo CD agent / Akuity) or per-cluster for air-gapped/regulated sites.

## Break it on purpose
1. Delete a directory in git: with `preserveResourcesOnDeletion` unset the generated Application and its resources are deleted. Set `spec.syncPolicy.preserveResourcesOnDeletion: true` on the ApplicationSet for safety and re-test.
2. Make the template reference a missing param with `missingkey=error` and read the controller error `kubectl -n argocd logs deploy/argocd-applicationset-controller`.
3. Add a typo in a cluster label so the selector matches zero clusters: the add-on silently doesn't install. Monitor "expected clusters == generated apps" in a Prometheus rule.

## Cleanup
```bash
kubectl -n argocd delete applicationset --all
```

## Talking points
Folder-per-app onboarding via PR, label-driven add-ons, matrix risks, PR previews with cleanup, `preserveResourcesOnDeletion`, hub vs per-cluster trade-off with credential risk.
