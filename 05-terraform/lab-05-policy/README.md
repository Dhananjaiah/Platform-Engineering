# Lab 05-05 - Policy as code for Terraform

## Goal
Gate Terraform with tflint, Checkov, Trivy config scan and Conftest/OPA on the **plan JSON**, with fixtures that must pass and fixtures that must fail. Then wire them as the pipeline's required checks.

## Prereqs
Lab 01 repo. Install: `tflint`, `checkov` (`pip install checkov`), `trivy`, `conftest`, `opa`.

## Step 1 - Static linting
```bash
cd ~/acme-tf-modules
cat > .tflint.hcl <<'EOF'
plugin "terraform" {
  enabled = true
  preset  = "recommended"
}
plugin "azurerm" {
  enabled = true
  version = "0.27.0"
  source  = "github.com/terraform-linters/tflint-ruleset-azurerm"
}
EOF
tflint --init && tflint --recursive
```
Why: catches invalid SKUs/regions, unused variables, deprecated syntax before plan. Expected: no issues, or specific findings with file/line.

## Step 2 - Security scanning of code
```bash
checkov -d modules --framework terraform --compact --quiet
trivy config modules --severity HIGH,CRITICAL
```
Read results. Intentionally weaken a module (`public_network_access_enabled = true` default in keyvault) and rerun; both tools flag it. Suppress only with a justified inline skip and an owner:
```hcl
#checkov:skip=CKV_AZURE_189: public access required for vendor webhook, ticket PLAT-123, owner team-payments
```
Why: scanners report on code; they cannot know *values* computed at plan. That is what plan-level policy is for.

## Step 3 - Policy on the plan with OPA/Conftest
Generate plan JSON (needs creds or use the fixture below):
```bash
cd examples/basic
terraform plan -out tf.plan && terraform show -json tf.plan > plan.json
```
Write policy:
```bash
mkdir -p ~/acme-tf-modules/policy && cd ~/acme-tf-modules/policy
cat > tags.rego <<'EOF'
package main

import rego.v1

required := {"owner", "service", "env", "cost-center"}

taggable := {
  "azurerm_key_vault", "azurerm_postgresql_flexible_server",
  "azurerm_user_assigned_identity", "azurerm_kubernetes_cluster",
}

deny contains msg if {
  some rc in input.resource_changes
  rc.type in taggable
  rc.change.actions[_] != "delete"
  tags := object.get(rc.change.after, "tags", {})
  missing := required - {k | tags[k]}
  count(missing) > 0
  msg := sprintf("%s is missing tags: %v", [rc.address, missing])
}

deny contains msg if {
  some rc in input.resource_changes
  rc.type == "azurerm_postgresql_flexible_server"
  rc.change.after.public_network_access_enabled == true
  msg := sprintf("%s must not enable public network access", [rc.address])
}

deny contains msg if {
  some rc in input.resource_changes
  rc.change.actions[_] == "delete"
  rc.type in {"azurerm_postgresql_flexible_server", "azurerm_key_vault"}
  msg := sprintf("%s deletion requires the 'allow-destroy' change label (checked in CI)", [rc.address])
}
EOF
```

## Step 4 - Must-pass and must-fail fixtures (tests for your policy)
```bash
mkdir -p fixtures && cat > fixtures/good.json <<'EOF'
{"resource_changes":[{"address":"azurerm_key_vault.a","type":"azurerm_key_vault",
 "change":{"actions":["create"],"after":{"tags":{"owner":"t","service":"s","env":"dev","cost-center":"c"}}}}]}
EOF
cat > fixtures/bad_tags.json <<'EOF'
{"resource_changes":[{"address":"azurerm_key_vault.a","type":"azurerm_key_vault",
 "change":{"actions":["create"],"after":{"tags":{"owner":"t"}}}}]}
EOF
cat > fixtures/bad_public.json <<'EOF'
{"resource_changes":[{"address":"azurerm_postgresql_flexible_server.a","type":"azurerm_postgresql_flexible_server",
 "change":{"actions":["create"],"after":{"public_network_access_enabled":true,
 "tags":{"owner":"t","service":"s","env":"dev","cost-center":"c"}}}}]}
EOF
conftest test fixtures/good.json --policy .          # expect PASS
conftest test fixtures/bad_tags.json --policy .      # expect FAIL: missing tags
conftest test fixtures/bad_public.json --policy .    # expect FAIL: public access
```
Verify each exit code: `echo $?` is 0 for good, 1 for bad. Policies without failing tests are untested policies. Put these three commands in CI as a "policy tests" job.

Run against the real plan: `conftest test ~/acme-tf-modules/examples/basic/plan.json --policy ~/acme-tf-modules/policy`.

## Step 5 - Policy exceptions without rotting
Exceptions via a reviewed file, not a flag:
```bash
cat > exceptions.yaml <<'EOF'
- address: azurerm_postgresql_flexible_server.legacy
  rule: public-access
  owner: team-billing
  expires: 2026-12-31
  ticket: PLAT-456
EOF
```
Policy reads it as data (`conftest --data exceptions.yaml`) and fails after `expires`. Interview point: every exception has an owner, a ticket and an expiry.

## Where each tool sits
| Stage | Tool | Catches |
|-------|------|---------|
| pre-commit/PR | fmt, validate, tflint | syntax, provider rules |
| PR | Checkov/Trivy | insecure config in code |
| PR (after plan) | Conftest/OPA, Sentinel in TFC | actual planned values, deletions, cost guardrails |
| runtime | Azure Policy, Kyverno/Gatekeeper | anything created outside Terraform |

## Break it on purpose
Add a resource without tags, run `terraform plan`, then `conftest`. Fix by adding tags. Then deliberately add a `#checkov:skip` without a reason and add a grep check: `grep -rn "checkov:skip" modules | grep -vE "skip=[A-Z_0-9]+: .{10,}"` must be empty.

## Talking points
Defence in depth: code scan, plan policy, runtime policy. Policies are code with tests. Exceptions expire. Sentinel is the TFC/TFE-native equivalent of the OPA plan checks.
