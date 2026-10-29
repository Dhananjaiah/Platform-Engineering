# Lab 02-03 - Self-service resources and abstraction design

## Goal
(A) Build a real self-service **Postgres-as-a-Service** API on Kubernetes with Crossplane (runs locally: it composes a CloudNativePG cluster; on cloud you swap the composed resource for Azure/AWS DB). (B) Compare three abstraction styles for describing a service: custom CRD, Score, and Helm values, and decide when to expose raw YAML.

## Prereqs
`hub` cluster, CloudNativePG operator (Module 01 Lab 01 Step 6), Crossplane installed (Module 05 Lab 08). Check: `kubectl get crd | grep -E "crossplane|postgresql.cnpg"`.

## Part A - Postgres-as-a-Service

### Step 1 - Crossplane functions and provider-kubernetes
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Function
metadata: { name: function-go-templating }
spec: { package: xpkg.crossplane.io/crossplane-contrib/function-go-templating:v0.9.2 }
---
apiVersion: pkg.crossplane.io/v1
kind: Function
metadata: { name: function-auto-ready }
spec: { package: xpkg.crossplane.io/crossplane-contrib/function-auto-ready:v0.4.2 }
---
apiVersion: pkg.crossplane.io/v1beta1
kind: DeploymentRuntimeConfig
metadata: { name: provider-kubernetes }
spec:
  serviceAccountTemplate: { metadata: { name: provider-kubernetes } }
---
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata: { name: provider-kubernetes }
spec:
  package: xpkg.crossplane.io/crossplane-contrib/provider-kubernetes:v0.18.0
  runtimeConfigRef: { name: provider-kubernetes }
EOF
kubectl get providers,functions
```
Wait for `HEALTHY True` on all. (Pin versions you tested; check the Upbound/crossplane-contrib marketplace for current tags.) Give the provider permission to create what the Composition creates; **least privilege in production** (only CNPG kinds); for the lab:
```bash
kubectl create clusterrolebinding provider-kubernetes-admin --clusterrole=cluster-admin --serviceaccount=crossplane-system:provider-kubernetes
cat <<'EOF' | kubectl apply -f -
apiVersion: kubernetes.crossplane.io/v1alpha1
kind: ProviderConfig
metadata: { name: default }
spec: { credentials: { source: InjectedIdentity } }
EOF
```

### Step 2 - The API (XRD): what developers can ask for
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata: { name: xpostgresinstances.platform.acme.io }
spec:
  group: platform.acme.io
  names: { kind: XPostgresInstance, plural: xpostgresinstances }
  claimNames: { kind: PostgresInstance, plural: postgresinstances }
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
              required: [owner, size]
              properties:
                owner: { type: string, description: "Backstage Group, e.g. team-payments" }
                size:  { type: string, enum: [small, medium, large], default: small }
                storageGB: { type: integer, minimum: 5, maximum: 500, default: 10 }
                highAvailability: { type: boolean, default: false }
            status:
              type: object
              properties:
                host: { type: string }
                secretName: { type: string }
EOF
```
Design choices to explain: three T-shirt sizes instead of raw CPU/RAM (the platform can change what "small" means); bounded numeric fields; defaults; `owner` required for showback; **no** exposure of engine version, parameters, or network settings (platform decides, secure by default).

### Step 3 - The implementation (Composition, pipeline mode)
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata: { name: postgres-cnpg, labels: { provider: cnpg } }
spec:
  compositeTypeRef: { apiVersion: platform.acme.io/v1alpha1, kind: XPostgresInstance }
  mode: Pipeline
  pipeline:
    - step: render
      functionRef: { name: function-go-templating }
      input:
        apiVersion: gotemplating.fn.crossplane.io/v1beta1
        kind: GoTemplate
        source: Inline
        inline:
          template: |
            {{- $xr := .observed.composite.resource -}}
            {{- $name := $xr.metadata.name -}}
            {{- $ns := index $xr.metadata.labels "crossplane.io/claim-namespace" | default "default" -}}
            {{- $sizes := dict "small" (dict "cpu" "250m" "mem" "512Mi") "medium" (dict "cpu" "1" "mem" "2Gi") "large" (dict "cpu" "2" "mem" "8Gi") -}}
            {{- $s := index $sizes $xr.spec.size -}}
            ---
            apiVersion: kubernetes.crossplane.io/v1alpha2
            kind: Object
            metadata:
              annotations:
                gotemplating.fn.crossplane.io/composition-resource-name: cluster
            spec:
              providerConfigRef: { name: default }
              forProvider:
                manifest:
                  apiVersion: postgresql.cnpg.io/v1
                  kind: Cluster
                  metadata:
                    name: {{ $name }}
                    namespace: {{ $ns }}
                    labels:
                      owner: {{ $xr.spec.owner }}
                      managed-by: platform
                  spec:
                    instances: {{ if $xr.spec.highAvailability }}3{{ else }}1{{ end }}
                    storage: { size: {{ $xr.spec.storageGB }}Gi }
                    resources:
                      requests: { cpu: {{ $s.cpu }}, memory: {{ $s.mem }} }
                      limits:   { memory: {{ $s.mem }} }
                    bootstrap: { initdb: { database: app, owner: app } }
            ---
            apiVersion: platform.acme.io/v1alpha1
            kind: XPostgresInstance
            status:
              host: {{ $name }}-rw.{{ $ns }}.svc
              secretName: {{ $name }}-app
    - step: ready
      functionRef: { name: function-auto-ready }
EOF
```
(Check the installed `provider-kubernetes` API version for `Object`: `kubectl get crd objects.kubernetes.crossplane.io -o jsonpath='{.spec.versions[*].name}'`. Adjust `v1alpha2`/`v1alpha1`.) Note: if your Crossplane version reports `ns` label differently, inspect `kubectl get xpostgresinstances -o yaml`.

### Step 4 - A developer's request (the claim)
```bash
kubectl create ns payments-dev 2>/dev/null
cat <<'EOF' | kubectl apply -f -
apiVersion: platform.acme.io/v1alpha1
kind: PostgresInstance
metadata: { name: payments-db, namespace: payments-dev }
spec: { owner: team-payments, size: small, storageGB: 10 }
EOF
kubectl -n payments-dev get postgresinstance payments-db -w
kubectl get xpostgresinstances,objects
kubectl -n payments-dev get cluster.postgresql.cnpg.io payments-db
kubectl -n payments-dev get secret payments-db-app -o jsonpath='{.data.uri}' | base64 -d; echo
```
Expected: claim `SYNCED True`, `READY True` after the DB pod is up; the connection secret created by CNPG (`<name>-app`) is what the app consumes (via ESO or direct). Debug chain: claim -> composite (`kubectl describe xpostgresinstance`) -> `Object` -> provider logs -> CNPG `Cluster` events.
This claim lives in the service's gitops folder (created by the Backstage template), applied by ArgoCD. **Developers get a DB with 12 lines and no tickets.**

### Step 5 - Guardrails and day-2
- **Validation**: XRD schema (enum/bounds) + Kyverno on claims (only owners' namespaces may use their owner group).
- **Defaults that encode policy**: backups (CNPG `backup`), TLS, resource limits, labels, anti-affinity for HA.
- **Evolution**: version the API (`v1alpha1` -> `v1beta1`), use Composition revisions so changes roll out deliberately (`compositionUpdatePolicy: Manual` for prod claims, `Automatic` for dev).
- **Deletion safety**: `deletionPolicy: Orphan` / finalizer protections and a Kyverno rule blocking claim deletion in prod without an annotation `platform.acme.io/allow-delete: "true"`.
- **Cost**: size mapping is the cost lever; add `cost-center` label from the namespace.
- **Cloud variant**: replace the composed Object with provider-azure `FlexibleServer`/provider-aws `Instance`, same XRD: developers don't change a line when you move from on-cluster to managed (that is the benefit of the API).

### Step 6 - Terraform-backed alternative (when Crossplane is not your tool)
Same developer experience via PR: Backstage opens a PR adding `live/dev/payments-db/terragrunt.hcl` using the `postgres` module; CI plans, policy checks, apply on merge (Module 05). Latency minutes, strong review, no new control plane. Say when you'd pick each: **Crossplane** for continuous reconcile and K8s-native self-service (in-cluster resources, many small requests); **Terraform via PR** for heavyweight/long-lived cloud resources where plan review and existing modules/skills matter.

## Part B - Abstraction design: how much to hide?
Describe the same service three ways and compare.

### B1 - Raw Kubernetes YAML (no abstraction)
Deployment + Service + HPA + PDB + ServiceMonitor + NetworkPolicy + HTTPRoute (~200 lines). Full power, every team re-learns Kubernetes, inconsistent security, hard to change centrally.

### B2 - Custom CRD / platform API (`AcmeService`)
```yaml
apiVersion: platform.acme.io/v1alpha1
kind: AcmeService
metadata: { name: payments, namespace: payments-dev }
spec:
  image: ghcr.io/acme-lab-<you>/payments-api@sha256:...
  port: 8080
  replicas: { min: 2, max: 10 }
  cpu: 250m
  memory: 256Mi
  expose: { host: payments.apps.example.com }
  dependencies: { postgres: payments-db }
  slo: { availability: "99.9" }
```
A Crossplane Composition (or an operator) renders Deployment/HPA/PDB/netpol/route/monitoring from 15 lines. Pros: central evolution, strong validation, policy by default, uniform. Cons: you own an API product (versioning, docs, escape hatches), harder debugging (more layers), risk of becoming a worse Kubernetes.

### B3 - Helm values on a golden chart
```yaml
# values/prd.yaml
image: { repository: ghcr.io/acme-lab-<you>/payments-api, digest: sha256:... }
replicas: { min: 4, max: 20 }
resources: { cpu: 500m, memory: 512Mi }
route: { host: payments.apps.example.com }
```
Pros: familiar, easy to adopt, upgrades via chart version, simple to inspect (`helm template`). Cons: values schema creeps toward YAML-in-YAML, templating complexity, weaker validation unless `values.schema.json` is enforced.

### B4 - Score (workload spec, platform-agnostic)
```yaml
apiVersion: score.dev/v1b1
metadata: { name: payments }
containers:
  app:
    image: ghcr.io/acme-lab-<you>/payments-api:latest
    variables:
      DB_URL: postgresql://${resources.db.username}:${resources.db.password}@${resources.db.host}:${resources.db.port}/${resources.db.name}
service: { ports: { web: { port: 8080 } } }
resources:
  db: { type: postgres }
```
`score-k8s init && score-k8s generate score.yaml` renders manifests locally, `score-compose` renders Docker Compose for local dev. Pros: developer-owned, portable across local/dev/prod, resources are abstract (`type: postgres` is resolved by the platform), pairs well with orchestrators like Humanitec. Cons: limited expressivity for advanced needs, another spec to learn, tooling maturity.
```bash
score-k8s --version
score-k8s init && score-k8s generate score.yaml && cat manifests.yaml | head -40
```

### Decision framework
| Question | Leans toward |
|---|---|
| Team K8s expertise high, few special needs? | raw YAML/Kustomize + policy |
| 100+ services, need uniform security and central evolution | golden chart or platform CRD |
| Need dev-local parity and cross-environment portability | Score (or compose) + orchestrator |
| Platform team capacity small | Helm golden chart (cheapest) |
| Need rich lifecycle (provisioning + deployment) as one API | CRD/Crossplane or Humanitec |
**Rules of thumb:** hide what teams shouldn't decide (security, networking, observability wiring); expose what is their domain (resources, scaling bounds, env config); provide an **escape hatch** (`extraManifests`, annotation overrides, "eject" to raw manifests with a supportability note) so edge cases don't force off-path; version the abstraction; measure how often escape hatches are used.

## Break it on purpose
1. Claim with `size: huge`: rejected by the API server (schema).
2. Delete the Composition; the claim stays `Composite resource claim is waiting for composite resource to become Ready`, then fix.
3. Change `small` mapping in the Composition and observe existing claims: with `Automatic` update they roll; discuss why prod uses `Manual` + revisions.

## Cleanup
```bash
kubectl -n payments-dev delete postgresinstance payments-db
kubectl delete composition postgres-cnpg; kubectl delete xrd xpostgresinstances.platform.acme.io
```

## Talking points
Self-service means API + guardrails + defaults; T-shirt sizes; claim/composite/managed debugging chain; Crossplane vs Terraform-via-PR; abstraction spectrum (raw, chart, CRD, Score) with escape hatches and versioning.
