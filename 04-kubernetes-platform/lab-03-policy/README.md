# Lab 04-03 - Policy and governance: Kyverno (and Gatekeeper equivalents)

## Goal
Enforce org standards at admission: no `latest` tags, required labels, resource limits, approved registries, signed images. Test policies offline with the Kyverno CLI, roll out Audit -> Enforce, and compare to Gatekeeper.

## Prereqs
`dev` cluster; `kyverno` CLI (`kyverno version`). Install Kyverno:
```bash
helm repo add kyverno https://kyverno.github.io/kyverno && helm repo update
helm install kyverno kyverno/kyverno -n kyverno --create-namespace --version 3.4.1 \
  --set admissionController.replicas=3
kubectl -n kyverno rollout status deploy/kyverno-admission-controller
```
Why 3 replicas: the admission controller is in the API request path; `failurePolicy: Fail` plus one replica = cluster outage on a restart. HA and PDB first.

## Step 1 - Policy: disallow `latest` and require a digest/tag, approved registries
```bash
mkdir -p policies && cat > policies/images.yaml <<'EOF'
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: image-standards
  annotations:
    policies.kyverno.io/title: Image standards
    policies.kyverno.io/severity: high
spec:
  validationFailureAction: Audit        # start in Audit, flip to Enforce after cleanup
  background: true
  rules:
    - name: no-latest-tag
      match: { any: [{ resources: { kinds: [Pod] } }] }
      validate:
        message: "Images must not use ':latest' or be untagged."
        pattern:
          spec:
            containers:
              - image: "!*:latest & *:* | *@sha256:*"
    - name: approved-registries
      match: { any: [{ resources: { kinds: [Pod] } }] }
      exclude: { any: [{ resources: { namespaces: [kube-system, kyverno, ingress-nginx, cert-manager, monitoring] } }] }
      validate:
        message: "Images must come from ghcr.io/acme-lab-<you>/ or registry.k8s.io/."
        pattern:
          spec:
            containers:
              - image: "ghcr.io/acme-lab-<you>/* | registry.k8s.io/*"
EOF
kubectl apply -f policies/images.yaml
kubectl get clusterpolicy
```
Kyverno auto-generates rules for Deployments/StatefulSets etc. from the Pod rule. Test in Audit mode: deploy `nginx:latest`; it is admitted, and a PolicyReport records the violation:
```bash
kubectl -n default create deployment t --image=nginx:latest
kubectl get policyreport -A -o wide
kubectl get clusterpolicyreport,policyreport -A | head
```

## Step 2 - Required labels and resource limits
```bash
cat > policies/standards.yaml <<'EOF'
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: { name: workload-standards }
spec:
  validationFailureAction: Audit
  background: true
  rules:
    - name: require-labels
      match: { any: [{ resources: { kinds: [Deployment, StatefulSet, CronJob] } }] }
      exclude: { any: [{ resources: { namespaces: [kube-system, kyverno] } }] }
      validate:
        message: "Labels owner, service and env are required."
        pattern:
          metadata:
            labels:
              owner: "?*"
              service: "?*"
              env: "?*"
    - name: require-requests-limits
      match: { any: [{ resources: { kinds: [Pod] } }] }
      exclude: { any: [{ resources: { namespaces: [kube-system, kyverno] } }] }
      validate:
        message: "CPU/memory requests and memory limits are required."
        pattern:
          spec:
            containers:
              - resources:
                  requests: { cpu: "?*", memory: "?*" }
                  limits:   { memory: "?*" }
    - name: forbid-privileged
      match: { any: [{ resources: { kinds: [Pod] } }] }
      validate:
        message: "Privileged containers are not allowed."
        pattern:
          spec:
            =(initContainers):
              - =(securityContext): { =(privileged): "false" }
            containers:
              - =(securityContext): { =(privileged): "false" }
EOF
kubectl apply -f policies/standards.yaml
```
Mutation for convenience (defaults rather than rejecting) is a good developer-experience trade: e.g. a `mutate` rule adding `seccompProfile: RuntimeDefault`.

## Step 3 - Offline tests with the Kyverno CLI (policies need tests)
```bash
mkdir -p tests && cat > tests/good-deploy.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ok
  labels: { owner: team-payments, service: payments, env: dev }
spec:
  selector: { matchLabels: { app: ok } }
  template:
    metadata: { labels: { app: ok } }
    spec:
      containers:
        - name: c
          image: ghcr.io/acme-lab-<you>/payments-api@sha256:0000000000000000000000000000000000000000000000000000000000000000
          resources: { requests: { cpu: 10m, memory: 32Mi }, limits: { memory: 64Mi } }
EOF
sed 's#ghcr.io/acme-lab-<you>/payments-api@sha256:0*#docker.io/library/nginx:latest#; s/owner: team-payments, //' tests/good-deploy.yaml > tests/bad-deploy.yaml
kyverno apply policies/ --resource tests/good-deploy.yaml
kyverno apply policies/ --resource tests/bad-deploy.yaml
```
Expected: good = pass; bad = fail on latest tag, registry, labels. Wire this into the gitops repo CI so PRs fail before reaching the cluster; and use `kyverno test` with a `kyverno-test.yaml` for table-driven results.

## Step 4 - Verify signed images (supply-chain enforcement)
```bash
cat > policies/verify-signature.yaml <<'EOF'
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: { name: verify-image-signatures }
spec:
  validationFailureAction: Audit
  webhookTimeoutSeconds: 30
  rules:
    - name: platform-signed
      match: { any: [{ resources: { kinds: [Pod], namespaces: ["payments-*"] } }] }
      verifyImages:
        - imageReferences: ["ghcr.io/acme-lab-<you>/*"]
          mutateDigest: true
          required: true
          attestors:
            - entries:
                - keyless:
                    issuer: https://token.actions.githubusercontent.com
                    subjectRegExp: "^https://github.com/acme-lab-<you>/acme-platform-workflows/.github/workflows/build-image.yml@refs/tags/v1.*"
                    rekor: { url: https://rekor.sigstore.dev }
EOF
kubectl apply -f policies/verify-signature.yaml
```
Test: deploy your signed image (passes) and an unsigned push of the same app (reported/blocked once Enforce). `mutateDigest: true` rewrites tags to digests at admission, closing the "tag moved after scan" hole. Air-gapped: use a key or private Rekor.

## Step 5 - Roll out safely: Audit -> Enforce
1. Install in `Audit`, collect reports a week: `kubectl get policyreport -A -o json | jq '[.items[].results[]|select(.result=="fail")]|group_by(.policy)|map({policy:.[0].policy,count:length})'`.
2. Notify owners with the list (Backstage scorecard "policy compliant").
3. Flip to `Enforce` for **new** namespaces/clusters first, then existing in waves; keep `exclude` for system namespaces; document exceptions with `PolicyException` resources (owner, expiry).
4. Set `failurePolicy` thoughtfully: `Ignore` for non-critical, `Fail` for security policies with HA admission controller and monitoring on webhook latency.
```bash
kubectl patch clusterpolicy image-standards --type merge -p '{"spec":{"validationFailureAction":"Enforce"}}'
kubectl -n default create deployment t2 --image=nginx:latest   # now denied
```

## Step 6 - Gatekeeper equivalent (be able to read and write it)
```bash
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/v3.19.1/deploy/gatekeeper.yaml
cat <<'EOF' | kubectl apply -f -
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata: { name: k8srequiredlabels }
spec:
  crd: { spec: { names: { kind: K8sRequiredLabels }, validation: { openAPIV3Schema: { type: object, properties: { labels: { type: array, items: { type: string } } } } } } }
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels
        violation[{"msg": msg}] {
          provided := {l | input.review.object.metadata.labels[l]}
          missing := {l | l := input.parameters.labels[_]} - provided
          count(missing) > 0
          msg := sprintf("missing labels: %v", [missing])
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata: { name: require-owner }
spec:
  enforcementAction: warn
  match: { kinds: [{ apiGroups: ["apps"], kinds: ["Deployment"] }] }
  parameters: { labels: ["owner"] }
EOF
```
| | Kyverno | Gatekeeper |
|---|---|---|
| Language | YAML patterns/CEL | Rego |
| Mutate/generate | yes, first-class | mutation yes; no generate |
| Verify images | built in | not built in |
| Learning curve | lower | higher, more expressive |
| Reuse with Terraform policy | no | Rego/OPA shared with Conftest |
On AKS: Azure Policy for Kubernetes is managed Gatekeeper; use it for baseline built-ins and Kyverno/Gatekeeper constraint templates for custom needs. Pick one engine per cluster for custom policy.

## Break it on purpose
1. Scale Kyverno admission controller to 0 with `failurePolicy: Fail` policies: observe API calls hang/fail; this is the "policy engine took the cluster down" incident. Restore, then add PDB + exclude kyverno namespace itself.
2. Write a policy with a typo in a field path; Audit shows no violations (false confidence); catch with `kyverno test` negative cases.
3. Apply policy to `kube-system` without excludes: system pods fail to restart.

## Cleanup
```bash
kubectl delete clusterpolicy --all; kubectl delete deploy t t2 -n default --ignore-not-found
```

## Talking points
Audit before Enforce, policy tests in CI, HA admission, exceptions with expiry, mutate for DX, signature verification tied to workflow identity, engine choice criteria.
