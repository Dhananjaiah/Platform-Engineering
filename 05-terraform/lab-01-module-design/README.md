# Lab 05-01 - Reusable module design and composition

## Goal
Build three small modules (`identity`, `keyvault`, `postgres`) and compose them into a product module `standard-service` ("a service with DB + Key Vault + identity"). Validate it with no cloud credentials.

## Prereqs
`terraform` >= 1.9, `git`. Work in `~/acme-tf-modules` (this is your `acme-tf-modules` repo).

## Step 1 - Repo and layout
```bash
mkdir -p ~/acme-tf-modules && cd ~/acme-tf-modules && git init -b main
mkdir -p modules/identity modules/keyvault modules/postgres modules/standard-service examples/basic
```
Why: `modules/` holds primitives and product modules; `examples/` is both documentation and the test fixture.

## Step 2 - Identity primitive
```bash
cat > modules/identity/main.tf <<'EOF'
resource "azurerm_user_assigned_identity" "this" {
  name                = var.name
  resource_group_name = var.resource_group_name
  location            = var.location
  tags                = var.tags
}
EOF
cat > modules/identity/variables.tf <<'EOF'
variable "name" {
  type        = string
  description = "Identity name, e.g. id-acme-prd-weu-payments."
}
variable "resource_group_name" { type = string }
variable "location"            { type = string }
variable "tags" {
  type    = map(string)
  default = {}
}
EOF
cat > modules/identity/outputs.tf <<'EOF'
output "id"           { value = azurerm_user_assigned_identity.this.id }
output "principal_id" { value = azurerm_user_assigned_identity.this.principal_id }
output "client_id"    { value = azurerm_user_assigned_identity.this.client_id }
EOF
cat > modules/identity/versions.tf <<'EOF'
terraform {
  required_version = ">= 1.9.0"
  required_providers {
    azurerm = { source = "hashicorp/azurerm", version = "~> 4.0" }
  }
}
EOF
```
Why: `required_providers` in modules uses a *range* (`~>`), the root config pins exactly via the lock file. Modules must never configure providers themselves (no `provider {}` block), or consumers cannot use `for_each`/`depends_on` on them.

## Step 3 - Key Vault primitive with secure defaults and validation
```bash
cat > modules/keyvault/variables.tf <<'EOF'
variable "name" {
  type = string
  validation {
    condition     = can(regex("^kv-[a-z0-9-]{3,21}$", var.name))
    error_message = "Key Vault name must match ^kv-[a-z0-9-]{3,21}$ (24 char Azure limit)."
  }
}
variable "resource_group_name" { type = string }
variable "location"            { type = string }
variable "tenant_id"           { type = string }
variable "reader_principal_ids" {
  type        = list(string)
  default     = []
  description = "Principals granted Key Vault Secrets User (RBAC)."
}
variable "public_network_access_enabled" {
  type        = bool
  default     = false
  description = "Secure default is false. Opting in must be a visible, reviewed change."
}
variable "tags" {
  type    = map(string)
  default = {}
}
EOF
cat > modules/keyvault/main.tf <<'EOF'
resource "azurerm_key_vault" "this" {
  name                          = var.name
  resource_group_name           = var.resource_group_name
  location                      = var.location
  tenant_id                     = var.tenant_id
  sku_name                      = "standard"
  enable_rbac_authorization     = true
  purge_protection_enabled      = true
  soft_delete_retention_days    = 30
  public_network_access_enabled = var.public_network_access_enabled
  tags                          = var.tags
}

resource "azurerm_role_assignment" "reader" {
  for_each             = toset(var.reader_principal_ids)
  scope                = azurerm_key_vault.this.id
  role_definition_name = "Key Vault Secrets User"
  principal_id         = each.value
}
EOF
cat > modules/keyvault/outputs.tf <<'EOF'
output "id"        { value = azurerm_key_vault.this.id }
output "vault_uri" { value = azurerm_key_vault.this.vault_uri }
EOF
cp modules/identity/versions.tf modules/keyvault/versions.tf
```
Why: RBAC authorization (not access policies), purge protection and no public access are the secure defaults. `for_each` over a set keyed by principal gives stable addresses when the list changes (a `count` index would shift and recreate assignments).

## Step 4 - Postgres primitive
```bash
cat > modules/postgres/variables.tf <<'EOF'
variable "name"                { type = string }
variable "resource_group_name" { type = string }
variable "location"            { type = string }
variable "sku_name" {
  type    = string
  default = "B_Standard_B1ms"
}
variable "storage_mb" {
  type    = number
  default = 32768
}
variable "postgres_version" {
  type    = string
  default = "16"
}
variable "administrator_login" {
  type    = string
  default = "pgadmin"
}
variable "high_availability" {
  type    = bool
  default = false
}
variable "tags" {
  type    = map(string)
  default = {}
}
EOF
cat > modules/postgres/main.tf <<'EOF'
resource "random_password" "admin" {
  length  = 32
  special = false
}

resource "azurerm_postgresql_flexible_server" "this" {
  name                          = var.name
  resource_group_name           = var.resource_group_name
  location                      = var.location
  version                       = var.postgres_version
  sku_name                      = var.sku_name
  storage_mb                    = var.storage_mb
  administrator_login           = var.administrator_login
  administrator_password        = random_password.admin.result
  public_network_access_enabled = false
  backup_retention_days         = 14
  geo_redundant_backup_enabled  = var.high_availability
  zone                          = "1"
  tags                          = var.tags

  dynamic "high_availability" {
    for_each = var.high_availability ? [1] : []
    content {
      mode                      = "ZoneRedundant"
      standby_availability_zone = "2"
    }
  }

  lifecycle {
    ignore_changes = [zone]
  }
}
EOF
cat > modules/postgres/outputs.tf <<'EOF'
output "fqdn" { value = azurerm_postgresql_flexible_server.this.fqdn }
output "admin_password" {
  value     = random_password.admin.result
  sensitive = true
}
EOF
cat > modules/postgres/versions.tf <<'EOF'
terraform {
  required_version = ">= 1.9.0"
  required_providers {
    azurerm = { source = "hashicorp/azurerm", version = "~> 4.0" }
    random  = { source = "hashicorp/random",  version = "~> 3.6" }
  }
}
EOF
```
Interview point: the password lands in state. State therefore needs encryption + restricted access, and for production you move to Entra auth / push the secret straight to Key Vault (next step) and mark the output sensitive.

## Step 5 - Product module `standard-service`
```bash
cat > modules/standard-service/variables.tf <<'EOF'
variable "service"             { type = string }
variable "env" {
  type = string
  validation {
    condition     = contains(["dev", "stg", "prd"], var.env)
    error_message = "env must be dev, stg or prd."
  }
}
variable "region_short"        { type = string }
variable "resource_group_name" { type = string }
variable "location"            { type = string }
variable "tenant_id"           { type = string }
variable "owner" {
  type        = string
  description = "Backstage Group that owns this service."
}
variable "cost_center"         { type = string }
variable "with_database" {
  type    = bool
  default = true
}
EOF
cat > modules/standard-service/main.tf <<'EOF'
locals {
  base = "${var.env}-${var.region_short}-${var.service}"
  tags = {
    owner      = var.owner
    service    = var.service
    env        = var.env
    cost-center = var.cost_center
    managed-by = "terraform"
  }
}

module "identity" {
  source              = "../identity"
  name                = "id-acme-${local.base}"
  resource_group_name = var.resource_group_name
  location            = var.location
  tags                = local.tags
}

module "keyvault" {
  source               = "../keyvault"
  name                 = "kv-${substr(replace(local.base, "-", ""), 0, 18)}"
  resource_group_name  = var.resource_group_name
  location             = var.location
  tenant_id            = var.tenant_id
  reader_principal_ids = [module.identity.principal_id]
  tags                 = local.tags
}

module "postgres" {
  count               = var.with_database ? 1 : 0
  source              = "../postgres"
  name                = "pg-acme-${local.base}"
  resource_group_name = var.resource_group_name
  location            = var.location
  high_availability   = var.env == "prd"
  tags                = local.tags
}

resource "azurerm_key_vault_secret" "db_password" {
  count        = var.with_database ? 1 : 0
  name         = "db-admin-password"
  value        = module.postgres[0].admin_password
  key_vault_id = module.keyvault.id
}
EOF
cat > modules/standard-service/outputs.tf <<'EOF'
output "identity_client_id" { value = module.identity.client_id }
output "key_vault_uri"      { value = module.keyvault.vault_uri }
output "db_fqdn"            { value = try(module.postgres[0].fqdn, null) }
EOF
cp modules/postgres/versions.tf modules/standard-service/versions.tf
```
Why: a consumer supplies 8 inputs, gets identity + vault + DB + wiring + tags + prod-only HA. This is the "thin, opinionated product" idea.

## Step 6 - Example and validation (no cloud login)
```bash
cat > examples/basic/main.tf <<'EOF'
provider "azurerm" {
  features {}
  skip_provider_registration = true
  subscription_id            = "00000000-0000-0000-0000-000000000000"
}

module "payments" {
  source              = "../../modules/standard-service"
  service             = "payments"
  env                 = "dev"
  region_short        = "weu"
  resource_group_name = "rg-acme-lab"
  location            = "westeurope"
  tenant_id           = "00000000-0000-0000-0000-000000000000"
  owner               = "team-payments"
  cost_center         = "cc-1234"
}
EOF
cd examples/basic && terraform init -backend=false && terraform fmt -recursive ../.. && terraform validate
```
Expected: `Success! The configuration is valid.` If `validate` fails, read the error: it names file and line. Verify naming: `terraform plan` needs credentials; with a real subscription, `terraform plan -out tf.plan` then `terraform show tf.plan | grep -E "name|tags"`.

## Break it on purpose
1. Set `env = "production"`: `terraform validate` passes but `terraform plan` fails the validation block. Lesson: validation runs at plan time for input values.
2. Change `reader_principal_ids` from a set to `count` in the keyvault module, add a second principal at the front, and plan. Observe recreation of role assignments. Revert to `for_each`.
3. Add `provider "azurerm" {}` inside a module, then try to `count` the module in the root: Terraform errors that the module is not compatible with count/for_each.

## Cleanup
```bash
cd ~/acme-tf-modules && git add -A && git commit -m "feat: identity, keyvault, postgres, standard-service modules"
```
(No cloud resources were created.)

## Talking points
- Composition over giant modules; product modules encode opinions, primitives stay generic.
- Secure defaults, explicit opt-out; validation blocks give early, readable errors.
- `for_each` with stable keys vs `count`; no provider config in modules.
- Secrets in state: mitigate with encrypted backend + RBAC, prefer identity-based auth, never print outputs (`sensitive = true`).
