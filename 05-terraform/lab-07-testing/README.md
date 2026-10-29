# Lab 05-07 - Testing Terraform modules

## Goal
Test a module at three levels: static (`validate`), native `terraform test` with mock providers (fast, no cloud), and Terratest (real infra, integration).

## Prereqs
Terraform >= 1.9 (mock providers), Go 1.22+ for Terratest. Uses the `keyvault` module from Lab 01.

## Step 1 - Static
```bash
cd ~/acme-tf-modules/modules/keyvault
terraform init -backend=false && terraform validate && terraform fmt -check
```

## Step 2 - Native tests with a mock provider (no credentials, runs in seconds)
```bash
mkdir -p tests && cat > tests/defaults.tftest.hcl <<'EOF'
mock_provider "azurerm" {}

variables {
  name                = "kv-test-app"
  resource_group_name = "rg-test"
  location            = "westeurope"
  tenant_id           = "00000000-0000-0000-0000-000000000000"
}

run "secure_defaults" {
  command = plan

  assert {
    condition     = azurerm_key_vault.this.public_network_access_enabled == false
    error_message = "public access must default to off"
  }
  assert {
    condition     = azurerm_key_vault.this.purge_protection_enabled == true
    error_message = "purge protection must be on"
  }
  assert {
    condition     = azurerm_key_vault.this.enable_rbac_authorization == true
    error_message = "RBAC authorization must be on"
  }
}

run "rejects_bad_name" {
  command = plan
  variables { name = "Bad_Name" }
  expect_failures = [var.name]
}

run "creates_role_per_reader" {
  command = plan
  variables { reader_principal_ids = ["11111111-1111-1111-1111-111111111111", "22222222-2222-2222-2222-222222222222"] }
  assert {
    condition     = length(azurerm_role_assignment.reader) == 2
    error_message = "one role assignment per principal"
  }
}
EOF
terraform test
```
Expected: `Success! 3 passed, 0 failed.` Each `run` is a test case; `expect_failures` proves your validation fires. With `mock_provider` computed attributes get generated values; use `override_resource` when a test needs specific ones.

## Step 3 - Prove the tests can fail
Edit `main.tf`: set `purge_protection_enabled = false`. Run `terraform test`; expect `1 failed` with your error message. Revert. A test that cannot fail is decoration.

## Step 4 - Terratest (integration against a real subscription)
```bash
mkdir -p ~/acme-tf-modules/test && cd ~/acme-tf-modules/test
go mod init acme/tf-modules-test
cat > keyvault_test.go <<'EOF'
package test

import (
	"testing"

	"github.com/gruntwork-io/terratest/modules/random"
	"github.com/gruntwork-io/terratest/modules/terraform"
	"github.com/stretchr/testify/assert"
)

func TestStandardService(t *testing.T) {
	t.Parallel()
	suffix := random.UniqueId()
	opts := &terraform.Options{
		TerraformDir: "../examples/basic",
		Vars: map[string]interface{}{"service": "t" + suffix},
		NoColor: true,
	}
	defer terraform.Destroy(t, opts)
	terraform.InitAndApply(t, opts)
	uri := terraform.Output(t, opts, "key_vault_uri")
	assert.Contains(t, uri, ".vault.azure.net")
}
EOF
go mod tidy && go test -v -timeout 45m ./...
```
Requires `az login` (or OIDC in CI), costs money, takes minutes. Use for product modules on a schedule or release tag, not every PR. Always `defer Destroy`, always use a dedicated sandbox subscription with a nightly sweeper (`az group list --tag ttl --query ...`) for leaked resources.

(Add `variable "service"` to the example so Vars apply: `variable "service" { default = "payments" }` and `service = var.service`.)

## Test pyramid for IaC
| Level | Tool | When | Cost |
|-------|------|------|------|
| Static | fmt, validate, tflint, checkov | every commit | seconds |
| Unit (plan, mocked) | `terraform test` + mock_provider | every PR | seconds |
| Integration (real apply) | Terratest / `terraform test` with `command = apply` | nightly / release | minutes, $ |
| Policy | conftest fixtures | every PR | seconds |

## Break it on purpose
Remove the `validation` block in `variables.tf`; `rejects_bad_name` now fails because no failure occurred. This shows tests protecting contracts.

## Talking points
Fast feedback with mocks, real apply only where mocks lie (RBAC propagation, provider quirks), tests assert the *contract* (secure defaults, validation, outputs).
