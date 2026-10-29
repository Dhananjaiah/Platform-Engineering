# Lab 05-03 - State: remote backends, layout, import, moved, lock incident

## Goal
Run a real remote backend with locking (MinIO as local S3; Azure Storage variant shown), lay out state per env/component, then rehearse `import`, `moved` refactors and a stuck-lock incident.

## Prereqs
Docker, Terraform >= 1.10 (native S3 lockfile), `mc` (MinIO client) optional.

## Step 1 - Start a local S3 (MinIO)
```bash
docker run -d --name minio -p 9000:9000 -p 9001:9001 \
  -e MINIO_ROOT_USER=tfadmin -e MINIO_ROOT_PASSWORD=tfadmin-secret-123 \
  quay.io/minio/minio server /data --console-address ":9001"
docker run --rm --network host --entrypoint sh quay.io/minio/mc -c \
  "mc alias set local http://localhost:9000 tfadmin tfadmin-secret-123 && mc mb local/tfstate && mc version enable local/tfstate"
```
Why: bucket **versioning** is your state backup; you can restore a previous state version after corruption. Expected: `Bucket created successfully`.

## Step 2 - State layout (decide, then create)
```
live/
  dev/network/      -> key: dev/network/terraform.tfstate
  dev/payments/     -> key: dev/payments/terraform.tfstate
  prd/network/
  prd/payments/
```
Rule: split by **environment** (blast radius, access) and **component lifecycle** (network changes rarely, apps often). Cross-state reads use `terraform_remote_state` sparingly or, better, data sources / explicit outputs published to a parameter store, to avoid tight coupling.

## Step 3 - Configure the backend
```bash
mkdir -p ~/acme-infra/live/dev/network && cd ~/acme-infra/live/dev/network
cat > backend.tf <<'EOF'
terraform {
  backend "s3" {
    bucket                      = "tfstate"
    key                         = "dev/network/terraform.tfstate"
    region                      = "us-east-1"
    endpoints                   = { s3 = "http://localhost:9000" }
    use_path_style              = true
    use_lockfile                = true          # native S3 locking (no DynamoDB)
    skip_credentials_validation = true
    skip_requesting_account_id  = true
    skip_metadata_api_check     = true
    skip_region_validation      = true
  }
}
EOF
cat > main.tf <<'EOF'
resource "terraform_data" "vnet_stub" {
  input = { name = "vnet-acme-dev-weu", cidr = "10.100.0.0/16" }
}
output "vnet_cidr" { value = terraform_data.vnet_stub.input.cidr }
EOF
export AWS_ACCESS_KEY_ID=tfadmin AWS_SECRET_ACCESS_KEY=tfadmin-secret-123
terraform init && terraform apply -auto-approve
```
Verify: `docker run --rm --network host --entrypoint sh quay.io/minio/mc -c "mc alias set l http://localhost:9000 tfadmin tfadmin-secret-123 >/dev/null && mc ls --recursive l/tfstate"` shows `dev/network/terraform.tfstate`.
Production equivalents: AWS = S3 with versioning + SSE-KMS + (lockfile or DynamoDB `dynamodb_table`); Azure = storage account `azurerm` backend (blob lease = lock), `use_azuread_auth = true`, private endpoint, soft delete, no shared keys.

Azure bootstrap commands (chicken-and-egg: create once with `az`, not Terraform):
```bash
az group create -n rg-tfstate -l westeurope
az storage account create -n stacmetfstate001 -g rg-tfstate --sku Standard_GRS --allow-shared-key-access false --min-tls-version TLS1_2 --allow-blob-public-access false
az storage container create -n tfstate --account-name stacmetfstate001 --auth-mode login
az storage account blob-service-properties update --account-name stacmetfstate001 -g rg-tfstate --enable-versioning true --enable-delete-retention true --delete-retention-days 30
```

## Step 4 - `import` (adopt a pre-existing resource)
Simulate an unmanaged resource using a file:
```bash
echo "hello" > /tmp/legacy.txt
cat >> main.tf <<'EOF'

import {
  to = local_file.legacy
  id = "/tmp/legacy.txt"
}
EOF
terraform plan -generate-config-out=generated.tf
```
Expected: Terraform writes `generated.tf` describing the resource, plan says `1 to import`. Review generated code, then `terraform apply`. Why the `import` block (1.5+): reviewable in PR, repeatable, no manual `terraform import` against prod state. After apply, delete the `import` block.
(If `local_file` has no import support in your provider version, use any resource that does, e.g. `azurerm_resource_group` with `id = /subscriptions/.../resourceGroups/rg-acme-lab`.)

## Step 5 - Safe refactor with `moved`
Rename a resource:
```bash
sed -i 's/vnet_stub/vnet/g' main.tf
cat >> main.tf <<'EOF'

moved {
  from = terraform_data.vnet_stub
  to   = terraform_data.vnet
}
EOF
terraform plan
```
Expected: `terraform_data.vnet_stub has moved to terraform_data.vnet` and `0 to add, 0 to destroy`. Alternative for one-off, un-reviewed moves: `terraform state mv` (imperative, no review trail; avoid on shared state).

## Step 6 - Split a giant state (the real-world refactor)
Scenario: `dev/payments` owns network + app. Move network resources to a separate state.
```bash
terraform state list
terraform state pull > backup-$(date +%F).tfstate        # ALWAYS back up first
terraform state mv -state-out=../network-new.tfstate terraform_data.vnet terraform_data.vnet
```
Then `terraform init -migrate-state` in the target with its backend. Rules: freeze applies (lock the pipeline), back up, move, run `plan` in both roots expecting no changes, unfreeze.

## Step 7 - Incident drill: stuck lock
Terminal A: `terraform apply` with a long-running `local-exec` (`sleep 120`) in a `terraform_data` provisioner; kill it with `kill -9` mid-run.
Terminal B:
```bash
terraform plan
```
Expected: `Error acquiring the state lock` with a lock ID and holder info. Diagnose: is any pipeline/person still running? Check CI. Only then:
```bash
terraform force-unlock <LOCK_ID>
```
Lesson: `force-unlock` is safe only when you are sure nothing is applying; with S3 native locking the lock is the `.tflock` object, inspect it with `mc cat`.

## Break it on purpose
1. Corrupt state: overwrite the object with garbage, then restore the previous version: `mc ls --versions l/tfstate/dev/network/` then `mc cp --version-id <id> ...`.
2. Run apply from two shells at once; watch the second block.

## Cleanup
```bash
terraform destroy -auto-approve
docker rm -f minio
```

## Talking points
State = blast radius and secret store, so encrypt, version, restrict, and split. Prefer `import`/`moved` blocks (reviewable) over CLI state surgery. Locks protect against concurrent applies; stuck lock runbook starts with "verify nobody is applying".
