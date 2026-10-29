# Lab 05-04 - Environment governance: directories vs workspaces vs Terragrunt

## Goal
Build the same stack three ways, compare, then standardise on directory-per-env with Terragrunt for DRY config, enforced naming and tags.

## Prereqs
`terraform`, `terragrunt` (`terragrunt --version`), Lab 03 MinIO still running (or any backend).

## Step 1 - Workspaces (and why most platform teams avoid them for envs)
```bash
mkdir -p ~/ws-demo && cd ~/ws-demo
cat > main.tf <<'EOF'
variable "instance_count" { default = 1 }
resource "terraform_data" "app" {
  count = var.instance_count
  input = "app-${terraform.workspace}-${count.index}"
}
EOF
terraform init
terraform workspace new dev && terraform workspace new prd && terraform workspace list
terraform workspace select prd && terraform apply -auto-approve
```
Why it is risky: one code path, same backend credentials, easy to `apply` in the wrong workspace (`terraform workspace show` is the only guard), no per-env access control, no per-env version pinning. Good for: ephemeral/PR environments of the *same* stack.

## Step 2 - Directory per environment
```
live/
  _envcommon/            shared module calls
  dev/payments/main.tf   -> source = module vX
  prd/payments/main.tf
```
Pro: separate state, access control (CODEOWNERS per dir), promote by changing the `ref`. Con: duplication of inputs.

## Step 3 - Terragrunt removes the duplication
```bash
mkdir -p ~/tg/live/{dev,prd}/payments && cd ~/tg
cat > live/root.hcl <<'EOF'
locals {
  env_vars = read_terragrunt_config(find_in_parent_folders("env.hcl"))
  env      = local.env_vars.locals.env
}

remote_state {
  backend = "s3"
  generate = { path = "backend.tf", if_exists = "overwrite_terragrunt" }
  config = {
    bucket                      = "tfstate"
    key                         = "${path_relative_to_include()}/terraform.tfstate"
    region                      = "us-east-1"
    endpoints                   = { s3 = "http://localhost:9000" }
    use_path_style              = true
    use_lockfile                = true
    skip_credentials_validation = true
    skip_requesting_account_id  = true
    skip_metadata_api_check     = true
    skip_region_validation      = true
  }
}

inputs = {
  tags = {
    env        = local.env
    managed-by = "terragrunt"
  }
}
EOF
cat > live/dev/env.hcl <<'EOF'
locals { env = "dev" }
EOF
cat > live/prd/env.hcl <<'EOF'
locals { env = "prd" }
EOF
mkdir -p modules/app && cat > modules/app/main.tf <<'EOF'
variable "replicas" { type = number }
variable "tags"     { type = map(string) }
resource "terraform_data" "app" {
  input = { replicas = var.replicas, tags = var.tags }
}
EOF
for e in dev prd; do
  r=1; [ $e = prd ] && r=3
  cat > live/$e/payments/terragrunt.hcl <<EOF
include "root" { path = find_in_parent_folders("root.hcl") }
terraform { source = "../../../modules/app" }
inputs = { replicas = $r }
EOF
done
export AWS_ACCESS_KEY_ID=tfadmin AWS_SECRET_ACCESS_KEY=tfadmin-secret-123
cd live/dev/payments && terragrunt apply --non-interactive
```
(The `for` loop here is plain shell typed by hand to write two small files; you can also type each file separately.)
Verify: state keys `dev/payments/terraform.tfstate` and `prd/payments/...` appear in the bucket. `terragrunt run-all plan` from `live/` plans every unit (use `--terragrunt-parallelism`).

## Step 4 - Enforce naming and tags
Naming/tag rules must fail the PR, not rely on reviews. Add a tag precondition in the product module:
```hcl
variable "tags" {
  type = map(string)
  validation {
    condition     = alltrue([for k in ["owner","service","env","cost-center"] : contains(keys(var.tags), k)])
    error_message = "tags must include owner, service, env, cost-center."
  }
}
```
and an org-wide Azure Policy / AWS tag policy as the second line of defence (Module 04). Provider-level `default_tags` (AWS) or a `tags` local (azurerm) for propagation.

## Decision table
| Approach | Use when |
|----------|----------|
| Workspaces | ephemeral clones of one stack |
| Dir-per-env, plain Terraform | small estate, clarity > DRY |
| Dir-per-env + Terragrunt | many envs/regions/components, need DRY backend + inputs + dependency ordering |

## Break it on purpose
1. In prd, `terraform workspace select dev` mistake equivalent: run `terragrunt apply` in the wrong directory and notice the state key printed in the plan header. Add a CI check that fails when the path and `env` tag disagree.
2. Add a `dependency` block from payments to a network unit and run `terragrunt run-all apply` to see ordering.

## Cleanup
```bash
cd ~/tg/live && terragrunt run-all destroy --non-interactive
```

## Talking points
Env isolation by directory + state + credentials; Terragrunt for DRY; naming/tags enforced by validation and cloud policy; promote an env by bumping the module `ref`.
