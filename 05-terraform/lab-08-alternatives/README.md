# Lab 05-08 - OpenTofu, Crossplane, Pulumi, and the Backstage link

## Goal
Know the alternatives from experience: run the same "bucket" with OpenTofu, Crossplane (Kubernetes-native self-service) and Pulumi, then show how a Backstage template triggers infra via a PR to `acme-infra`.

## Prereqs
kind hub cluster (Module 00), Docker. MinIO from Lab 03 serves as the S3 target so everything is local.

## Part A - OpenTofu (drop-in)
```bash
curl --proto '=https' --tlsv1.2 -fsSL https://get.opentofu.org/install-opentofu.sh -o install-opentofu.sh
sh install-opentofu.sh --install-method standalone && tofu version
cd ~/acme-infra/live/dev/network && rm -rf .terraform && tofu init && tofu plan
```
Expected: same plan as Terraform (state format compatible). Why it matters: licence (BUSL vs MPL), client-side state encryption in OpenTofu, `tofu test`. Migration rule: pin the same major feature set, test plans match, switch CI image/binary, then lock file regenerate.

## Part B - Crossplane (infra as Kubernetes APIs)
```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable && helm repo update
helm install crossplane crossplane-stable/crossplane -n crossplane-system --create-namespace
kubectl -n crossplane-system rollout status deploy/crossplane
```
Install the S3 provider family (works against MinIO via endpoint config):
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata: { name: provider-aws-s3 }
spec: { package: xpkg.crossplane.io/crossplane-contrib/provider-aws-s3:v1.21.1 }
EOF
kubectl get providers -w
```
Expected: `INSTALLED=True HEALTHY=True`. Define the **platform API** (XRD) and its implementation (Composition). Developers ask for `Bucket` with 3 fields; the platform decides versioning, encryption, tags:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata: { name: xstoragebuckets.platform.acme.io }
spec:
  group: platform.acme.io
  names: { kind: XStorageBucket, plural: xstoragebuckets }
  claimNames: { kind: StorageBucket, plural: storagebuckets }
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: [owner, tier]
              properties:
                owner: { type: string }
                tier:  { type: string, enum: [standard, archive] }
EOF
```
Claims are namespaced (`StorageBucket` in a team namespace) and compositions render managed resources with enforced defaults. Walk through the Composition (pipeline mode with `function-patch-and-transform`), apply a claim, and verify:
```bash
kubectl get storagebuckets -A
kubectl get managed
kubectl describe xstoragebucket <name>
```
Debug path to say aloud: claim -> composite -> managed resource -> provider logs (`kubectl -n crossplane-system logs deploy/<provider>`), conditions `Synced` and `Ready`.

## Part C - Pulumi (general-purpose languages)
```bash
curl -fsSL https://get.pulumi.com | sh
mkdir ~/pulumi-demo && cd ~/pulumi-demo && pulumi login --local
pulumi new typescript -y --name demo --stack dev
pulumi preview
```
When Pulumi: complex logic/loops, unit tests in the team's language, existing TS/Python skills. Cost: smaller ecosystem of reviewable plan outputs/policy tooling, state backend choice (Pulumi Cloud/self-managed).

## Decision matrix (be able to defend it)
| Need | Terraform/OpenTofu | Crossplane | Pulumi |
|------|-------------------|------------|--------|
| Biggest module/provider ecosystem, hiring pool | strong | medium | medium |
| Developer self-service via K8s API/GitOps, continuous reconcile | weak (apply-driven) | strong | weak |
| Drift auto-correction | scheduled plans | built in | scheduled |
| Complex logic, real language tests | limited | via functions | strong |
| Plan review in PR | strong | diff of YAML, no plan | preview |

Typical staff answer: Terraform for foundational/landing-zone infra and long-lived platform resources; Crossplane (or Terraform behind a pipeline) for developer self-service resources; do not run two tools for the same resource.

## Part D - The Backstage link (two patterns)
1. **PR to infra repo** (recommended): scaffolder `publish:github:pull-request` adds `live/<env>/<service>/terragrunt.hcl` from a skeleton. Review + policy + plan + apply happen in the normal pipeline. Auditable, reversible, slower (minutes).
2. **Call a pipeline/API**: scaffolder `http:backstage:request` or a custom action triggers a workflow dispatch / Crossplane claim commit. Faster feedback, harder to audit unless it also commits.
Skeleton to add in Module 01 lab 03: `skeleton/infra/terragrunt.hcl` with `${{ values.name }}`, `${{ values.owner }}` placeholders.

## Break it on purpose
1. Crossplane: apply a claim with `tier: gold`; the API server rejects it with the enum message (schema = guardrail).
2. Delete the Provider's credentials secret and watch the managed resource go `Synced=False`; restore it.
3. `kubectl delete` a managed resource: Crossplane recreates it (continuous reconciliation) unless `deletionPolicy: Orphan`.

## Cleanup
```bash
kubectl delete storagebuckets -A --all
helm uninstall crossplane -n crossplane-system
cd ~/pulumi-demo && pulumi destroy -y && pulumi stack rm dev -y
```

## Talking points
OpenTofu is a low-risk fork you can adopt in a day; Crossplane gives a Kubernetes-native self-service API with continuous reconcile but loses plan-review; Pulumi trades ecosystem for language power. The deciding question is *who the consumer is* and *how they want to ask*.
