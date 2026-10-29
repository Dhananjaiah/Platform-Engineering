# Lab 06-01 - Central reusable workflows and composite actions

## Goal
Create `acme-platform-workflows`, publish a reusable CI workflow and a composite action, version them with tags, call them from a service repo, and pin third-party actions by commit SHA.

## Prereqs
`gh` authenticated; org from Module 00.

## Step 1 - Create the platform repo
```bash
gh repo create acme-lab-<you>/acme-platform-workflows --public --clone && cd acme-platform-workflows
mkdir -p .github/workflows actions/setup-node-cached
```
Why public/internal: reusable workflows in a private repo must have "Access" enabled (Settings > Actions > General > Access) to be callable from other private repos in the org. Check it via:
```bash
gh api repos/acme-lab-<you>/acme-platform-workflows/actions/permissions/access
```

## Step 2 - A composite action (shared steps)
```bash
cat > actions/setup-node-cached/action.yml <<'EOF'
name: setup-node-cached
description: Node setup with the org's standard cache settings
inputs:
  node-version:
    description: Node version
    default: "20"
runs:
  using: composite
  steps:
    - uses: actions/setup-node@REPLACE_WITH_SHA   # v4.x
      with:
        node-version: ${{ inputs.node-version }}
        cache: npm
    - run: npm ci --prefer-offline --no-audit
      shell: bash
EOF
```
Composite action = reusable *steps* inside a job; reusable workflow = reusable *job(s)* with its own runner, permissions and secrets. Use composite for step bundles, reusable workflows for whole pipelines.

## Step 3 - Pin third-party actions by SHA
```bash
gh api repos/actions/setup-node/git/ref/tags/v4.4.0 --jq '.object | {type,sha}'
```
If `type` is `tag` (annotated), dereference once more:
```bash
gh api repos/actions/setup-node/git/tags/<sha-from-above> --jq .object.sha
```
Put the final commit SHA in `uses: actions/setup-node@<sha> # v4.4.0`. Why: tags are mutable; a compromised upstream can move `v4` (this is how the tj-actions/changed-files supply chain attack worked). Renovate/Dependabot update SHA pins and keep the comment. Do the same for `actions/checkout`, `docker/*`, `sigstore/cosign-installer`, etc.

## Step 4 - The reusable CI workflow
```bash
cat > .github/workflows/ci-node.yml <<'EOF'
name: ci-node
on:
  workflow_call:
    inputs:
      node-version: { type: string, default: "20" }
      working-directory: { type: string, default: "." }
    outputs:
      version:
        description: Computed version
        value: ${{ jobs.build.outputs.version }}
permissions:
  contents: read            # least privilege; callers can only narrow, not widen
jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    outputs:
      version: ${{ steps.v.outputs.version }}
    defaults:
      run: { working-directory: "${{ inputs.working-directory }}" }
    steps:
      - uses: actions/checkout@REPLACE_WITH_SHA   # v4.x
        with: { persist-credentials: false }
      - uses: acme-lab-<you>/acme-platform-workflows/actions/setup-node-cached@v1
        with: { node-version: "${{ inputs.node-version }}" }
      - run: npm run lint --if-present
      - run: npm test --if-present -- --ci
      - id: v
        run: echo "version=$(jq -r .version package.json)-${GITHUB_SHA::7}" >> "$GITHUB_OUTPUT"
EOF
git add -A && git commit -m "feat: ci-node reusable workflow and composite action" && git push
git tag -a v1.0.0 -m v1.0.0 && git tag -f v1 && git push origin v1.0.0 && git push -f origin v1
```
Versioning scheme: immutable `v1.0.0` plus a moving major tag `v1` (documented exception to "never move tags"; protect `v1.*` immutable via rulesets, allow only the platform team to move `v1`). Teams choose: `@v1` (auto-patches) or `@<sha>` (frozen).
Note: composite action inside the *same* repo referenced with `@v1` uses the tag, not local path, so the reusable workflow and its actions version together.

## Step 5 - A service repo calls it
```bash
cd .. && gh repo create acme-lab-<you>/payments-api --private --clone && cd payments-api
npm init -y >/dev/null && npm pkg set scripts.test="node -e \"console.log('ok')\"" 
mkdir -p .github/workflows && cat > .github/workflows/ci.yml <<'EOF'
name: ci
on:
  pull_request:
  push: { branches: [main] }
permissions:
  contents: read
jobs:
  ci:
    uses: acme-lab-<you>/acme-platform-workflows/.github/workflows/ci-node.yml@v1
    with: { node-version: "20" }
EOF
git add -A && git commit -m "ci: use platform workflow" && git push -u origin main
gh run watch
```
Verify: run succeeds; in the run UI the job is labelled `ci / build`. Debug path: caller's `permissions` must be >= what the called workflow needs, or the run fails at startup with "The nested job ... is requesting ... permissions".

## Step 6 - Make breaking changes safely
Add a new required input in `v2`, keep `v1` for the migration window; publish release notes; Renovate PR bumps `@v1` to `@v2`. Show who is on what:
```bash
gh search code "acme-platform-workflows/.github/workflows/ci-node.yml@" --owner acme-lab-<you> --json repository,path
```

## Break it on purpose
1. Remove `permissions: contents: read` from the reusable workflow and set org default token permissions to read-write; show the token is broader than needed. Restore.
2. Call a workflow from a private repo not granted Access; read the "workflow was not found" error.
3. Move the `v1` tag to a commit that fails; observe every consumer breaks at once. Lesson: promote via a canary tag (`v1-beta`) first.

## Cleanup
Keep repos; later labs build on them.

## Talking points
Reusable workflow vs composite; permission ceiling; SHA pinning; major tag strategy and blast radius; discoverability of who uses what.
