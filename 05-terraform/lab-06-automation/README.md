# Lab 05-06 - Terraform automation: plan on PR, apply on merge, OIDC, drift, Atlantis

## Goal
Run Terraform from GitHub Actions with Azure OIDC (no stored secrets), a plan comment on every PR, apply on merge to `main` behind an environment approval, and a scheduled drift check. Then compare to Atlantis.

## Prereqs
`acme-infra` repo with `live/dev/network` from Lab 03 (switch backend to azurerm for the real run), `az`, `gh`.

## Step 1 - Create the identity GitHub will assume (OIDC federation)
```bash
APP_ID=$(az ad app create --display-name gh-acme-infra --query appId -o tsv)
az ad sp create --id $APP_ID
SUB=$(az account show --query id -o tsv)
az role assignment create --assignee $APP_ID --role Contributor --scope /subscriptions/$SUB/resourceGroups/rg-acme-lab
az role assignment create --assignee $APP_ID --role "Storage Blob Data Contributor" --scope $(az storage account show -n stacmetfstate001 -g rg-tfstate --query id -o tsv)
```
Federated credentials, one per trust subject (least privilege: plan from PRs, apply only from the environment):
```bash
az ad app federated-credential create --id $APP_ID --parameters '{
  "name":"pr","issuer":"https://token.actions.githubusercontent.com",
  "subject":"repo:acme-lab-<you>/acme-infra:pull_request","audiences":["api://AzureADTokenExchange"]}'
az ad app federated-credential create --id $APP_ID --parameters '{
  "name":"env-dev","issuer":"https://token.actions.githubusercontent.com",
  "subject":"repo:acme-lab-<you>/acme-infra:environment:dev","audiences":["api://AzureADTokenExchange"]}'
```
Why: the subject claim pins *which repo and which context* may get a token. Plan on PRs uses a read-only role in production setups (Reader + state read); apply runs only in the protected `dev`/`prd` environment.

Store non-secret IDs as repo variables:
```bash
gh variable set AZURE_CLIENT_ID --body $APP_ID -R acme-lab-<you>/acme-infra
gh variable set AZURE_TENANT_ID --body $(az account show --query tenantId -o tsv) -R acme-lab-<you>/acme-infra
gh variable set AZURE_SUBSCRIPTION_ID --body $SUB -R acme-lab-<you>/acme-infra
```

## Step 2 - Environments with required reviewers
```bash
gh api -X PUT repos/acme-lab-<you>/acme-infra/environments/dev
gh api -X PUT repos/acme-lab-<you>/acme-infra/environments/prd \
  -f 'reviewers[][type]=User' -F 'reviewers[][id]='$(gh api user --jq .id)
```

## Step 3 - Plan on PR
```bash
mkdir -p .github/workflows && cat > .github/workflows/tf-plan.yml <<'EOF'
name: tf-plan
on:
  pull_request:
    paths: ["live/**", "modules/**"]
permissions:
  contents: read
  id-token: write
  pull-requests: write
concurrency:
  group: tf-${{ github.ref }}
  cancel-in-progress: true
jobs:
  plan:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        dir: [live/dev/network]
    env:
      ARM_USE_OIDC: "true"
      ARM_USE_AZUREAD: "true"
      ARM_CLIENT_ID: ${{ vars.AZURE_CLIENT_ID }}
      ARM_TENANT_ID: ${{ vars.AZURE_TENANT_ID }}
      ARM_SUBSCRIPTION_ID: ${{ vars.AZURE_SUBSCRIPTION_ID }}
    steps:
      - uses: actions/checkout@v4   # pin to a full SHA in production (Module 06)
      - uses: hashicorp/setup-terraform@v3
        with: { terraform_version: 1.10.5 }
      - run: terraform fmt -check -recursive
      - run: terraform init -input=false
        working-directory: ${{ matrix.dir }}
      - run: terraform validate
        working-directory: ${{ matrix.dir }}
      - run: terraform plan -input=false -lock-timeout=60s -out=tfplan
        working-directory: ${{ matrix.dir }}
      - run: terraform show -json tfplan > plan.json
        working-directory: ${{ matrix.dir }}
      - run: |
          docker run --rm -v "$PWD:/w" openpolicyagent/conftest test /w/${{ matrix.dir }}/plan.json --policy /w/policy
EOF
```
Add a step that posts `terraform show -no-color tfplan` as a PR comment (e.g. `actions/github-script`) so reviewers read the plan, not just the diff.

## Step 4 - Apply on merge, with the *reviewed* plan
Apply the plan saved at PR time (upload as artifact) or re-plan and require no drift between them; never apply a different plan than the one approved.
```bash
cat > .github/workflows/tf-apply.yml <<'EOF'
name: tf-apply
on:
  push:
    branches: [main]
    paths: ["live/**", "modules/**"]
permissions:
  contents: read
  id-token: write
concurrency:
  group: tf-apply-dev
  cancel-in-progress: false      # never cancel an apply mid-flight
jobs:
  apply:
    runs-on: ubuntu-latest
    environment: dev
    env:
      ARM_USE_OIDC: "true"
      ARM_USE_AZUREAD: "true"
      ARM_CLIENT_ID: ${{ vars.AZURE_CLIENT_ID }}
      ARM_TENANT_ID: ${{ vars.AZURE_TENANT_ID }}
      ARM_SUBSCRIPTION_ID: ${{ vars.AZURE_SUBSCRIPTION_ID }}
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with: { terraform_version: 1.10.5 }
      - run: terraform init -input=false
        working-directory: live/dev/network
      - run: terraform apply -input=false -auto-approve
        working-directory: live/dev/network
EOF
git add -A && git commit -m "ci: terraform plan/apply with OIDC" && git push
```
Verify: open a PR changing a tag; see the plan job; merge; see the `dev` environment gate and apply.

## Step 5 - Drift detection
```bash
cat > .github/workflows/tf-drift.yml <<'EOF'
name: tf-drift
on:
  schedule: [{ cron: "17 5 * * *" }]
  workflow_dispatch:
permissions: { contents: read, id-token: write, issues: write }
jobs:
  drift:
    runs-on: ubuntu-latest
    env:
      ARM_USE_OIDC: "true"
      ARM_USE_AZUREAD: "true"
      ARM_CLIENT_ID: ${{ vars.AZURE_CLIENT_ID }}
      ARM_TENANT_ID: ${{ vars.AZURE_TENANT_ID }}
      ARM_SUBSCRIPTION_ID: ${{ vars.AZURE_SUBSCRIPTION_ID }}
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init -input=false
        working-directory: live/dev/network
      - id: plan
        run: |
          set +e
          terraform plan -input=false -detailed-exitcode
          echo "code=$?" >> "$GITHUB_OUTPUT"
        working-directory: live/dev/network
      - if: steps.plan.outputs.code == '2'
        run: gh issue create -t "Terraform drift in dev/network" -b "Scheduled plan found changes. Run: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}" -l drift
        env: { GH_TOKEN: "${{ github.token }}" }
EOF
```
`-detailed-exitcode`: 0 no changes, 1 error, 2 changes (drift). Test: change a resource tag in the Azure portal or with `az tag update`, run `gh workflow run tf-drift`, watch the issue open.

## Step 6 - Atlantis (alternative, PR-comment driven)
```bash
cat > atlantis.yaml <<'EOF'
version: 3
automerge: false
projects:
  - name: dev-network
    dir: live/dev/network
    autoplan: { when_modified: ["*.tf", "../../../modules/**/*.tf"] }
    apply_requirements: [approved, mergeable]
EOF
```
Run Atlantis as a pod, webhooks from GitHub, comment `atlantis plan` / `atlantis apply`. Comparison: Actions = no extra service, native OIDC, matrix scales; Atlantis = locking per dir/workspace across PRs, plan/apply UX in the PR; TFC/Spacelift = hosted runs, policy, RBAC, cost.

## Break it on purpose
1. Remove `id-token: write` and read the OIDC error (`Unable to get ACTIONS_ID_TOKEN_REQUEST_URL`).
2. Change federated credential subject typo, read `AADSTS70021: No matching federated identity record`.
3. Open two PRs against the same state; the second plan waits or fails on the lock (`-lock-timeout`).

## Talking points
OIDC subjects per context, least privilege per stage, concurrency groups, apply the approved plan, drift via scheduled `-detailed-exitcode` plans, reviewers on prod only.
