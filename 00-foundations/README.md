# 00 - Foundations: lab environment, repo topology, standards

Everything in later modules runs on this base. Do it by hand, once, and understand each piece.

## Reference architecture (planes, per CNCF Platforms white paper)
| Plane | Acme Platform component |
|-------|-------------------------|
| Developer control plane | Backstage (catalog, scaffolder, TechDocs) |
| Integration and delivery | GitHub, GitHub Actions (reusable workflows), ArgoCD, Argo Rollouts |
| Resource plane | Terraform modules, Crossplane, Kubernetes clusters (AKS/EKS/GKE) |
| Monitoring and logging | Prometheus, Grafana, Loki, OpenTelemetry |
| Security and compliance | Kyverno/Gatekeeper, Cosign, Trivy, External Secrets, OIDC |

## Repository topology (design this before writing code)
| Repo | Purpose | Owner |
|------|---------|-------|
| `acme-platform-workflows` | Reusable workflows + composite actions, semver tags | Platform |
| `acme-gitops` | Desired state of clusters and apps (config repo) | Platform (apps via PR) |
| `acme-infra` | Terraform live config (per env/component) | Platform |
| `acme-tf-modules` | Versioned Terraform modules | Platform |
| `acme-backstage` | Backstage app + custom plugins | Platform |
| `acme-templates` | Scaffolder templates + skeletons | Platform |
| `<service>` | Application code (app repo), separate from config | Product team |

Why app repo and config repo are separate: different change cadence, different access (devs push code, only CI/promoters push config), clean audit trail, no CI loops from config commits.

## Standards (enforced later by policy, not by wiki pages)
- Naming: `<org>-<env>-<region>-<component>` e.g. `acme-prd-weu-aks01`.
- Mandatory labels/tags: `owner` (Backstage Group), `service`, `env`, `cost-center`, `managed-by`.
- Environments: `dev`, `stg`, `prd`. Prod is a separate cluster and separate cloud account/subscription.

## Lab 0.1 - Tooling

Install the tools (Linux; adapt for macOS with Homebrew). Pin what you use and record it.

```bash
docker version --format '{{.Server.Version}}'
```
Why: kind runs Kubernetes nodes as containers; Docker must be running. Expected: a version string.

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.29.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind
kind version
```

```bash
curl -LO "https://dl.k8s.io/release/v1.33.1/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
kubectl version --client
```

```bash
curl -fsSL https://get.helm.sh/helm-v3.18.2-linux-amd64.tar.gz | tar xz
sudo mv linux-amd64/helm /usr/local/bin/helm && helm version --short
```

Also install: `terraform` (or `tofu`), `gh`, `az`, `jq`, `yq`, `argocd` CLI, `kustomize`, `kubeconform`, `cosign`, `trivy`, `node` 20 + `yarn`. Check each with `<tool> version`.
Verify: every command prints a version. Interview point: you pin tool versions (e.g. `.tool-versions`) so labs and CI are reproducible.

## Lab 0.2 - A multi-cluster lab (hub + spoke)

Three clusters let you rehearse hub-and-spoke ArgoCD and environment separation. Each needs ~2 GB RAM; if tight, start only `hub` and `dev` and add `prd` later.

Create a kind config for the hub (save as `kind-hub.yaml`):

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: hub
networking:
  ipFamily: ipv4
  disableDefaultCNI: true      # we install Cilium ourselves (Module 04)
  podSubnet: 10.10.0.0/16
  serviceSubnet: 10.11.0.0/16
nodes:
  - role: control-plane
    image: kindest/node:v1.33.1
    extraPortMappings:
      - containerPort: 30080   # NodePort for ingress/Backstage access
        hostPort: 8080
  - role: worker
    image: kindest/node:v1.33.1
  - role: worker
    image: kindest/node:v1.33.1
```

```bash
kind create cluster --config kind-hub.yaml
```
Why: `disableDefaultCNI` so NetworkPolicy is actually enforceable (kindnet does not enforce policy). Expected: node containers start. Note nodes will be `NotReady` until a CNI is installed.

```bash
kubectl --context kind-hub get nodes
```
Expected: 3 nodes `NotReady`. That is correct for now.

Install Cilium (CNI):

```bash
helm repo add cilium https://helm.cilium.io && helm repo update
helm install cilium cilium/cilium --version 1.17.4 -n kube-system \
  --set operator.replicas=1 --set ipam.mode=kubernetes
kubectl --context kind-hub -n kube-system rollout status ds/cilium
kubectl --context kind-hub get nodes
```
Verify: all nodes `Ready`. Debug if not: `kubectl -n kube-system logs ds/cilium`. Interview point: why CNI choice matters (NetworkPolicy, observability, eBPF, Azure CNI Overlay with Cilium dataplane on AKS).

Repeat for the spokes with different subnets (copy the file as `kind-dev.yaml` with `name: dev`, `podSubnet: 10.20.0.0/16`, `serviceSubnet: 10.21.0.0/16`, one worker, no `extraPortMappings`; same for `prd` with `10.30/10.31`), then install Cilium with `--kube-context kind-dev` / `kind-prd`.

```bash
kind get clusters
kubectl config get-contexts
```
Verify: `hub`, `dev`, `prd` listed.

## Lab 0.3 - Baseline add-ons on the hub

Metrics and ingress used by almost every module:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace \
  --set controller.service.type=NodePort --set controller.service.nodePorts.http=30080
kubectl -n ingress-nginx rollout status deploy/ingress-nginx-controller
```

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.17.2/cert-manager.yaml
kubectl -n cert-manager rollout status deploy/cert-manager-webhook
```
Verify: `kubectl get pods -A` all Running/Completed. Hostnames: use `*.127.0.0.1.nip.io` (resolves to localhost) so `backstage.127.0.0.1.nip.io:8080` reaches the cluster.

## Lab 0.4 - Accounts and credentials (done once, no long-lived secrets)
1. GitHub: create an org `acme-lab-<yourname>` (free). Create repos from the topology table (empty).
2. `gh auth login`, then `gh auth status`.
3. Azure (optional, for Module 04/05 applies): `az login`, `az account show`, create a dedicated lab subscription/resource group with a budget alert:
```bash
az group create -n rg-acme-lab -l westeurope
az consumption budget create --budget-name acme-lab --amount 50 --category cost --time-grain monthly --start-date 2026-11-01 --end-date 2027-11-01 --resource-group rg-acme-lab
```
Why: Terraform labs that create AKS/Postgres cost money; budgets and teardown commands are part of production discipline.

## Teardown (always do this)
```bash
kind delete cluster --name prd
kind delete cluster --name dev
kind delete cluster --name hub
```

## Say it in 2 minutes
"I separate planes: a developer control plane (Backstage), delivery (GitHub Actions + ArgoCD), resource plane (Terraform/Crossplane + clusters), observability and security. App repo and config repo are separate for access, cadence and audit. Prod is isolated by cluster and cloud account. Standards like labels and naming are enforced by policy, not documentation."
