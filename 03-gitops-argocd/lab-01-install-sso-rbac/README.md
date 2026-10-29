# Lab 03-01 - ArgoCD install, SSO, RBAC, AppProjects, repo credentials

## Goal
Install ArgoCD with Helm (HA-capable values), add GitHub SSO through Dex, map GitHub teams to ArgoCD roles, create per-team AppProjects, and connect the config repo with a read-only deploy key.

## Prereqs
kind `hub` (Module 00) with ingress-nginx; GitHub org with teams `platform-admins`, `team-payments`; `argocd` CLI.

## Step 1 - Install with pinned chart
```bash
kubectl config use-context kind-hub
helm repo add argo https://argoproj.github.io/argo-helm && helm repo update
helm search repo argo/argo-cd --versions | head -3
```
Pick the chart version you will pin (example `8.0.0`; record it).
```bash
cat > argocd-values.yaml <<'EOF'
global:
  domain: argocd.127.0.0.1.nip.io
configs:
  params:
    server.insecure: true            # TLS terminates at ingress
  cm:
    url: http://argocd.127.0.0.1.nip.io:8080
    exec.enabled: "false"
    application.resourceTrackingMethod: annotation+label
  rbac:
    policy.default: role:none        # deny by default
    scopes: "[groups]"
controller:
  replicas: 1                        # >1 requires sharding config (Lab 08)
  metrics: { enabled: true }
server:
  replicas: 2
  metrics: { enabled: true }
  ingress:
    enabled: true
    ingressClassName: nginx
repoServer:
  replicas: 2
  metrics: { enabled: true }
applicationSet:
  replicas: 1
redis-ha:
  enabled: false                     # true for real HA (3 redis + haproxy); skipped on a laptop
EOF
helm install argocd argo/argo-cd --version 8.0.0 -n argocd --create-namespace -f argocd-values.yaml
kubectl -n argocd rollout status deploy/argocd-server deploy/argocd-repo-server deploy/argocd-applicationset-controller
kubectl -n argocd rollout status statefulset/argocd-application-controller
```
Why: `policy.default: role:none` means authenticated users see nothing until mapped; `resourceTrackingMethod: annotation+label` avoids label-collision problems with Helm charts that set `app.kubernetes.io/instance`.
Verify:
```bash
kubectl -n argocd get pods
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
argocd login argocd.127.0.0.1.nip.io:8080 --plaintext --grpc-web --username admin
argocd version
```
After SSO works, disable admin: `admin.enabled: "false"` in `argocd-cm` and delete the initial secret.

## Step 2 - SSO with GitHub (Dex)
Create a GitHub OAuth App: callback `http://argocd.127.0.0.1.nip.io:8080/api/dex/callback`; note client ID/secret.
```bash
kubectl -n argocd patch secret argocd-secret --type merge -p \
 '{"stringData":{"dex.github.clientSecret":"<CLIENT_SECRET>"}}'
cat > dex-patch.yaml <<'EOF'
data:
  dex.config: |
    connectors:
      - type: github
        id: github
        name: GitHub
        config:
          clientID: <CLIENT_ID>
          clientSecret: $dex.github.clientSecret
          orgs:
            - name: acme-lab-<you>
EOF
kubectl -n argocd patch configmap argocd-cm --patch-file dex-patch.yaml
kubectl -n argocd rollout restart deploy/argocd-dex-server deploy/argocd-server
```
Open the URL, "Log in via GitHub". Expected: login works but you see nothing (policy default). That proves deny-by-default.
Production: use OIDC directly against Entra ID/Okta (`oidc.config` in `argocd-cm`, groups claim = group object IDs), not Dex, when your IdP is OIDC-compliant.

## Step 3 - RBAC mapped to groups
```bash
kubectl -n argocd patch configmap argocd-rbac-cm --type merge -p '{"data":{
"policy.csv":"p, role:team-payments, applications, get, payments/*, allow\np, role:team-payments, applications, sync, payments/*, allow\np, role:team-payments, logs, get, payments/*, allow\ng, acme-lab-<you>:platform-admins, role:admin\ng, acme-lab-<you>:team-payments, role:team-payments\n"}}'
```
GitHub groups appear as `<org>:<team-slug>`. Teams may *sync* but not delete or override in prod (add a prod-specific deny and require PR for prod). Verify: `argocd account can-i sync applications 'payments/*'` as each user; `argocd admin settings rbac validate --policy-file` before applying.

## Step 4 - AppProject: the tenancy boundary
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata: { name: payments, namespace: argocd }
spec:
  description: Payments team
  sourceRepos:
    - https://github.com/acme-lab-<you>/acme-gitops.git
  destinations:
    - { server: https://kubernetes.default.svc, namespace: "payments-*" }
  clusterResourceWhitelist: []          # no cluster-scoped objects (no ClusterRole, no Namespace)
  namespaceResourceBlacklist:
    - { group: "", kind: ResourceQuota }
    - { group: "", kind: LimitRange }
    - { group: networking.k8s.io, kind: NetworkPolicy }   # platform owns these
  orphanedResources: { warn: true }
  roles:
    - name: ci
      description: CI may sync only
      policies: ["p, proj:payments:ci, applications, sync, payments/*, allow"]
EOF
kubectl -n argocd get appproject payments -o yaml | head -40
```
Why: a team cannot deploy to another team's namespace, install CRDs/ClusterRoles, or change quotas/netpols, even though ArgoCD's controller is powerful. This is the multi-tenancy control.

## Step 5 - Repo credentials (read-only deploy key)
```bash
ssh-keygen -t ed25519 -f argocd-deploy -N "" -C argocd-readonly
gh repo deploy-key add argocd-deploy.pub -R acme-lab-<you>/acme-gitops -t argocd-readonly   # read-only by default
argocd repo add git@github.com:acme-lab-<you>/acme-gitops.git --ssh-private-key-path argocd-deploy --project payments
argocd repo list
```
Expected: `STATUS Successful`. Production: GitHub App credentials per org (`argocd repocreds add`), no personal tokens. Delete the private key file after loading it into the cluster: `shred -u argocd-deploy`.

## Break it on purpose
1. Remove `payments` from `sourceRepos`, create an Application using it: `application repo ... is not permitted in project`.
2. Point an Application at namespace `kube-system`: `destination ... not permitted`.
3. Put a syntax error in `policy.csv`; observe RBAC not loading, check `kubectl -n argocd logs deploy/argocd-server | grep -i rbac`.

## Cleanup
Keep ArgoCD installed for later labs.

## Talking points
Deny-by-default RBAC from IdP groups, AppProjects as tenancy, no admin account, read-only repo creds, pinned chart, resource tracking choice, `exec` disabled.
