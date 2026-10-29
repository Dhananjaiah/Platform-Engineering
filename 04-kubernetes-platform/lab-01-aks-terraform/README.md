# Lab 04-01 - AKS cluster as code (pools, autoscaling, workload identity, CNI, upgrades)

## Goal
Write a production-shaped AKS module, validate and plan it, apply it in a sandbox if budget allows, and map the same design to EKS and GKE.

## Prereqs
`terraform`, `az` logged in, Module 05 modules repo. Plan-only is fine without a subscription (use `terraform validate`).

## Step 1 - Design decisions to write down first (ADR)
| Decision | Choice | Why |
|----------|--------|-----|
| Network plugin | Azure CNI Overlay + Cilium dataplane | pod IP space not consumed from VNet, eBPF policy and observability |
| Identity | Entra ID integration, Azure RBAC for Kubernetes, local accounts disabled | no shared admin kubeconfig |
| Workload auth | OIDC issuer + Workload Identity | no secrets in pods |
| System pool | 3 nodes across zones, tainted `CriticalAddonsOnly=true:NoSchedule` | add-ons isolated from tenants |
| User pools | autoscaled, per-workload-class (general, memory, spot, GPU) | right-size + cost |
| Upgrades | auto-upgrade channel `patch` (prod) / `rapid`-ish (dev), node OS channel `NodeImage`, maintenance windows, surge 33 % | predictable, low blast radius |
| API server | private cluster or authorized IP ranges | reduce exposure |
| Cluster per env | `dev`, `stg`, `prd` separate subscriptions | blast radius, RBAC, quotas |

## Step 2 - The module
```bash
mkdir -p ~/acme-tf-modules/modules/aks && cd ~/acme-tf-modules/modules/aks
cat > variables.tf <<'EOF'
variable "name"                { type = string }
variable "resource_group_name" { type = string }
variable "location"            { type = string }
variable "kubernetes_version"  { type = string }
variable "admin_group_object_ids" { type = list(string) }
variable "subnet_id"           { type = string }
variable "log_analytics_workspace_id" { type = string }
variable "authorized_ip_ranges" {
  type    = list(string)
  default = []
}
variable "upgrade_channel" {
  type    = string
  default = "patch"
  validation {
    condition     = contains(["none", "patch", "stable", "rapid", "node-image"], var.upgrade_channel)
    error_message = "Invalid channel."
  }
}
variable "user_pools" {
  type = map(object({
    vm_size   = string
    min_count = number
    max_count = number
    spot      = optional(bool, false)
    labels    = optional(map(string), {})
    taints    = optional(list(string), [])
  }))
  default = {}
}
variable "tags" {
  type    = map(string)
  default = {}
}
EOF
cat > main.tf <<'EOF'
resource "azurerm_kubernetes_cluster" "this" {
  name                = var.name
  resource_group_name = var.resource_group_name
  location            = var.location
  dns_prefix          = var.name
  kubernetes_version  = var.kubernetes_version
  sku_tier            = "Standard"                 # financially-backed API SLA, uptime SLA
  tags                = var.tags

  automatic_upgrade_channel = var.upgrade_channel
  node_os_upgrade_channel   = "NodeImage"

  oidc_issuer_enabled       = true
  workload_identity_enabled = true
  local_account_disabled    = true
  azure_policy_enabled      = true

  identity { type = "SystemAssigned" }

  azure_active_directory_role_based_access_control {
    azure_rbac_enabled     = true
    admin_group_object_ids = var.admin_group_object_ids
  }

  api_server_access_profile {
    authorized_ip_ranges = var.authorized_ip_ranges
  }

  default_node_pool {
    name                         = "system"
    vm_size                      = "Standard_D4ds_v5"
    vnet_subnet_id               = var.subnet_id
    zones                        = ["1", "2", "3"]
    auto_scaling_enabled         = true
    min_count                    = 3
    max_count                    = 5
    only_critical_addons_enabled = true
    temporary_name_for_rotation  = "systemtmp"
    upgrade_settings { max_surge = "33%" }
  }

  network_profile {
    network_plugin      = "azure"
    network_plugin_mode = "overlay"
    network_data_plane  = "cilium"
    network_policy      = "cilium"
    pod_cidr            = "10.244.0.0/16"
    service_cidr        = "10.0.0.0/16"
    dns_service_ip      = "10.0.0.10"
    outbound_type       = "loadBalancer"
  }

  auto_scaler_profile {
    balance_similar_node_groups      = true
    expander                         = "least-waste"
    scale_down_unneeded              = "10m"
    skip_nodes_with_local_storage    = false
  }

  maintenance_window_auto_upgrade {
    frequency   = "Weekly"
    interval    = 1
    duration    = 4
    day_of_week = "Sunday"
    start_time  = "02:00"
    utc_offset  = "+00:00"
  }

  oms_agent { log_analytics_workspace_id = var.log_analytics_workspace_id }

  lifecycle {
    ignore_changes = [kubernetes_version]   # auto-upgrade channel owns patch versions
  }
}

resource "azurerm_kubernetes_cluster_node_pool" "user" {
  for_each              = var.user_pools
  name                  = each.key
  kubernetes_cluster_id = azurerm_kubernetes_cluster.this.id
  vm_size               = each.value.vm_size
  vnet_subnet_id        = var.subnet_id
  zones                 = ["1", "2", "3"]
  auto_scaling_enabled  = true
  min_count             = each.value.min_count
  max_count             = each.value.max_count
  priority              = each.value.spot ? "Spot" : "Regular"
  eviction_policy       = each.value.spot ? "Delete" : null
  spot_max_price        = each.value.spot ? -1 : null
  node_labels           = merge({ pool = each.key }, each.value.labels)
  node_taints           = concat(each.value.taints, each.value.spot ? ["kubernetes.azure.com/scalesetpriority=spot:NoSchedule"] : [])
  upgrade_settings { max_surge = "33%" }
  tags                  = var.tags
  lifecycle { ignore_changes = [node_count] }
}
EOF
cat > outputs.tf <<'EOF'
output "oidc_issuer_url" { value = azurerm_kubernetes_cluster.this.oidc_issuer_url }
output "kubelet_identity_object_id" { value = azurerm_kubernetes_cluster.this.kubelet_identity[0].object_id }
output "id" { value = azurerm_kubernetes_cluster.this.id }
EOF
cat > versions.tf <<'EOF'
terraform {
  required_version = ">= 1.9.0"
  required_providers { azurerm = { source = "hashicorp/azurerm", version = "~> 4.0" } }
}
EOF
terraform init -backend=false && terraform fmt && terraform validate
```
Explain: `lifecycle.ignore_changes` on version and `node_count` because the platform (auto-upgrade, autoscaler) owns them, so Terraform doesn't fight them. System pool is tainted so only add-ons schedule there.

## Step 3 - Consumer (compose with network and workload identity)
Example root (shortened): VNet + subnet module, Log Analytics, the `aks` module with `user_pools`:
```hcl
user_pools = {
  general = { vm_size = "Standard_D4ds_v5", min_count = 2, max_count = 20 }
  spot    = { vm_size = "Standard_D8ds_v5", min_count = 0, max_count = 30, spot = true, labels = { workload = "batch" } }
  gpu     = { vm_size = "Standard_NC6s_v3", min_count = 0, max_count = 4, taints = ["sku=gpu:NoSchedule"] }
}
```
```bash
az aks get-versions -l westeurope --query "values[?isPreview==null].version" -o tsv | head
terraform plan -out aks.plan
terraform show aks.plan | grep -E "network_data_plane|oidc|local_account|only_critical"
```
Apply only in a sandbox (`terraform apply aks.plan`, ~10 minutes, cost ~USD 5-10/day for the minimum). Then:
```bash
az aks get-credentials -g rg-acme-lab -n aks-acme-dev --overwrite-existing
kubelogin convert-kubeconfig -l azurecli
kubectl get nodes -L pool,topology.kubernetes.io/zone
```

## Step 4 - Workload Identity end to end
```bash
az identity create -g rg-acme-lab -n id-payments
CLIENT_ID=$(az identity show -g rg-acme-lab -n id-payments --query clientId -o tsv)
ISSUER=$(az aks show -g rg-acme-lab -n aks-acme-dev --query oidcIssuerProfile.issuerUrl -o tsv)
az identity federated-credential create --name payments --identity-name id-payments -g rg-acme-lab \
  --issuer "$ISSUER" --subject system:serviceaccount:payments-dev:payments --audience api://AzureADTokenExchange
kubectl create ns payments-dev
kubectl -n payments-dev create sa payments
kubectl -n payments-dev annotate sa payments azure.workload.identity/client-id=$CLIENT_ID
```
Pod label `azure.workload.identity/use: "true"` + the SA. Test with a pod running `az login --federated-token "$(cat $AZURE_FEDERATED_TOKEN_FILE)" --service-principal -u $AZURE_CLIENT_ID -t $AZURE_TENANT_ID` then `az keyvault secret list`. Why: the pod exchanges its projected SA token for an Entra token; nothing static. This is the identity that ESO (Module 03 Lab 07) also uses.

## Step 5 - Upgrade strategy (be able to run it)
```bash
az aks get-upgrades -g rg-acme-lab -n aks-acme-dev -o table
az aks upgrade -g rg-acme-lab -n aks-acme-dev --control-plane-only --kubernetes-version <N+1>
az aks nodepool upgrade -g rg-acme-lab --cluster-name aks-acme-dev -n general --max-surge 33%
kubectl get nodes -w
```
Order: dev -> stg -> prd; control plane first, then pools; check deprecated APIs beforehand (`kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis`, `pluto detect-all-in-cluster`); PDBs and `terminationGracePeriodSeconds` decide disruption. Version skew: nodes may be up to N-3 behind the control plane on current AKS (verify policy), but don't live there. Blue/green alternative: new node pool or new cluster, shift traffic, delete old.

## Step 6 - EKS and GKE parity
| Concern | AKS | EKS | GKE |
|---|---|---|---|
| Pool autoscaling | Cluster Autoscaler / node auto-provisioning (Karpenter-based) | Karpenter (recommended) or CA + managed node groups | GKE Autopilot / NAP |
| Workload identity | Workload Identity (federated) | IRSA or EKS Pod Identity | Workload Identity Federation for GKE |
| CNI | Azure CNI Overlay + Cilium | VPC CNI (+Cilium chaining) | Dataplane V2 (Cilium) |
| Policy | Azure Policy (Gatekeeper) + Kyverno | Kyverno/Gatekeeper | Policy Controller / Kyverno |
| Upgrade control | channels + maintenance windows | managed node group update, Karpenter drift | release channels + surge |
Be ready to write the EKS Karpenter `NodePool`/`EC2NodeClass` and say Karpenter's value: bin-packing across instance types, consolidation, fast scale-up; AKS NAP gives the same.

## Break it on purpose
1. `terraform plan` after manually scaling a pool in the portal: no diff because of `ignore_changes`; remove it to see the fight.
2. Remove the system taint: user pods land on system nodes and a runaway tenant starves CoreDNS (simulate with a CPU hog deployment).
3. Bump `kubernetes_version` two minors at once: AKS rejects (cannot skip minor versions).

## Cleanup
```bash
terraform destroy
az group delete -n rg-acme-lab --yes --no-wait   # if the whole sandbox is disposable
```

## Talking points
Isolation of system vs user pools, zones, workload identity, Entra RBAC with local accounts off, CNI choice rationale, auto-upgrade channels and maintenance windows, ignore_changes ownership, upgrade runbook, same architecture on EKS/GKE.
