# Lab 06-02 - Secure build pipeline: scan, SBOM, sign, provenance, GitOps handoff

## Goal
Extend the platform with `build-image.yml`: build and push a container, scan it, generate an SBOM, sign keyless with Cosign, attest SLSA provenance, and commit the image digest to the GitOps repo. Verify the signature from your laptop.

## Prereqs
Lab 01. A registry: GHCR (no extra setup; used here). `cosign`, `trivy`, `docker`.

## Step 1 - A Dockerfile for the sample app
```bash
cd payments-api
cat > index.js <<'EOF'
require('http').createServer((q,r)=>{r.end('payments ok\n')}).listen(8080)
EOF
cat > Dockerfile <<'EOF'
FROM node:20-slim AS run
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY index.js .
USER 10001
EXPOSE 8080
CMD ["node","index.js"]
EOF
docker build -t payments-api:local . && docker run --rm -p 8080:8080 -d --name pay payments-api:local && curl localhost:8080 && docker rm -f pay
```
Expected: `payments ok`. Production Dockerfile habits to say aloud: pinned digest base, non-root, multi-stage, no secrets in layers, `.dockerignore`.

## Step 2 - Reusable build workflow
In `acme-platform-workflows`:
```bash
cat > .github/workflows/build-image.yml <<'EOF'
name: build-image
on:
  workflow_call:
    inputs:
      image-name: { type: string, required: true }
    outputs:
      digest: { value: "${{ jobs.build.outputs.digest }}" }
permissions:
  contents: read
jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    permissions:
      contents: read
      packages: write          # push to GHCR
      id-token: write          # keyless signing via OIDC
      attestations: write
      security-events: write   # upload SARIF
    outputs:
      digest: ${{ steps.push.outputs.digest }}
    steps:
      - uses: actions/checkout@SHA
        with: { persist-credentials: false }
      - uses: docker/setup-buildx-action@SHA
      - uses: docker/login-action@SHA
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - id: push
        uses: docker/build-push-action@SHA
        with:
          context: .
          push: true
          tags: ghcr.io/${{ inputs.image-name }}:${{ github.sha }}
          provenance: false     # we attest separately below
      - name: Trivy vulnerability scan (fail on HIGH/CRITICAL with fix)
        uses: aquasecurity/trivy-action@SHA
        with:
          image-ref: ghcr.io/${{ inputs.image-name }}@${{ steps.push.outputs.digest }}
          severity: HIGH,CRITICAL
          ignore-unfixed: true
          exit-code: "1"
          format: sarif
          output: trivy.sarif
      - uses: github/codeql-action/upload-sarif@SHA
        if: always()
        with: { sarif_file: trivy.sarif }
      - name: SBOM
        uses: anchore/sbom-action@SHA
        with:
          image: ghcr.io/${{ inputs.image-name }}@${{ steps.push.outputs.digest }}
          format: spdx-json
          output-file: sbom.spdx.json
      - uses: sigstore/cosign-installer@SHA
      - name: Sign image (keyless) and attach SBOM attestation
        run: |
          IMG=ghcr.io/${{ inputs.image-name }}@${{ steps.push.outputs.digest }}
          cosign sign --yes "$IMG"
          cosign attest --yes --type spdxjson --predicate sbom.spdx.json "$IMG"
      - name: SLSA build provenance
        uses: actions/attest-build-provenance@SHA
        with:
          subject-name: ghcr.io/${{ inputs.image-name }}
          subject-digest: ${{ steps.push.outputs.digest }}
          push-to-registry: true
EOF
```
Replace every `SHA` using the Lab 01 `gh api` procedure. Why `@digest` everywhere: tags are mutable; signing and scanning must refer to the exact bytes.
Add SAST and dependency checks as a separate reusable `security.yml`: `github/codeql-action/init+analyze` (CodeQL), `returntocorp/semgrep` container job (`semgrep ci`), `actions/dependency-review-action` on PRs (fails on new high vulns or disallowed licenses).

## Step 3 - Call from the service
```bash
cat > .github/workflows/ci.yml <<'EOF'
name: ci
on:
  pull_request:
  push: { branches: [main] }
permissions:
  contents: read
jobs:
  ci:
    uses: acme-lab-<you>/acme-platform-workflows/.github/workflows/ci-node.yml@v1
  image:
    needs: ci
    if: github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write
      id-token: write
      attestations: write
      security-events: write
    uses: acme-lab-<you>/acme-platform-workflows/.github/workflows/build-image.yml@v1
    with: { image-name: acme-lab-<you>/payments-api }
EOF
git add -A && git commit -m "ci: build, scan, sign" && git push && gh run watch
```
Tag the platform repo `v1.1.0` and move `v1` first (Lab 01 Step 4).

## Step 4 - Verify like a consumer or an admission controller
```bash
DIGEST=$(gh api /orgs/acme-lab-<you>/packages/container/payments-api/versions --jq '.[0].name')
IMG=ghcr.io/acme-lab-<you>/payments-api@$DIGEST
cosign verify "$IMG" \
  --certificate-identity-regexp '^https://github.com/acme-lab-<you>/acme-platform-workflows/.github/workflows/build-image.yml@refs/tags/v1.*' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
cosign verify-attestation "$IMG" --type spdxjson \
  --certificate-identity-regexp '...same...' --certificate-oidc-issuer https://token.actions.githubusercontent.com | head -c 300
gh attestation verify oci://$IMG -R acme-lab-<you>/payments-api
```
Expected: cosign prints the verified claims. The *identity is the reusable workflow ref*: only images built by the platform workflow verify. Module 04 enforces this at admission with Kyverno `verifyImages`.

## Step 5 - GitOps handoff
A second reusable workflow `gitops-bump.yml` updates the digest in `acme-gitops` via a short-lived GitHub App token (not a PAT):
```bash
cat > .github/workflows/gitops-bump.yml <<'EOF'
name: gitops-bump
on:
  workflow_call:
    inputs:
      service: { type: string, required: true }
      env:     { type: string, default: dev }
      image:   { type: string, required: true }   # ghcr.io/org/app@sha256:...
    secrets:
      APP_PRIVATE_KEY: { required: true }
    # app id passed as vars.GITOPS_APP_ID
permissions: { contents: read }
jobs:
  bump:
    runs-on: ubuntu-latest
    steps:
      - id: tok
        uses: actions/create-github-app-token@SHA
        with:
          app-id: ${{ vars.GITOPS_APP_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}
          owner: ${{ github.repository_owner }}
          repositories: acme-gitops
      - uses: actions/checkout@SHA
        with:
          repository: ${{ github.repository_owner }}/acme-gitops
          token: ${{ steps.tok.outputs.token }}
      - run: |
          yq -i '.images[0].digest = "'"${DIGEST}"'"' apps/${{ inputs.service }}/overlays/${{ inputs.env }}/kustomization.yaml
        env: { DIGEST: "${{ inputs.image }}" }
      - run: |
          git config user.name "acme-ci[bot]" && git config user.email "ci@acme.invalid"
          git checkout -b bump/${{ inputs.service }}-${{ github.run_id }}
          git commit -am "chore(${{ inputs.service }}): bump ${{ inputs.env }} image" && git push -u origin HEAD
          gh pr create -t "bump ${{ inputs.service }} ${{ inputs.env }}" -b "Automated. Source: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
        env: { GH_TOKEN: "${{ steps.tok.outputs.token }}" }
EOF
```
Dev may auto-merge; stg/prd via PR approval (Module 03 promotion). Why a PR not direct push: audit, branch protection, policy checks on the config repo.

## Break it on purpose
1. Introduce a vulnerable base (`FROM node:14`), watch Trivy fail the build; then add a time-boxed `.trivyignore` entry with a CVE ID and expiry comment.
2. Push an *unsigned* image manually: `docker tag ... && docker push`, then run `cosign verify`: fails `no matching signatures`. This is what admission policy will block.
3. Remove `id-token: write` in the caller: cosign fails with an OIDC error. Shows permission ceilings from Lab 01.

## Cleanup
Delete old GHCR versions: `gh api -X DELETE /orgs/acme-lab-<you>/packages/container/payments-api/versions/<id>`.

## Talking points
Digest-based everything, keyless signing with workflow identity, SBOM + provenance, scan gates with expiring exceptions, CI never touches the cluster. SLSA level language: hosted build, signed provenance (L2), hardened reusable-workflow builder with isolated signing (L3 target).
