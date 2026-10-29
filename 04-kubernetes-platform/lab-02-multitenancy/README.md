# Lab 04-02 - Multi-tenancy baseline: namespace, quota, LimitRange, NetworkPolicy, RBAC, Pod Security

## Goal
Create a "tenant" the way the platform would: one reviewed bundle per team namespace. Prove isolation with tests, then break it.

## Prereqs
kind `dev` cluster with Cilium (Module 00). `kubectl config use-context kind-dev`.

## Step 1 - Namespace with Pod Security Standards and ownership labels
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: payments-dev
  labels:
    owner: team-payments
    cost-center: cc-1234
    env: dev
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
EOF
kubectl get ns payments-dev --show-labels
```
Why: PSS `restricted` = non-root, no privilege escalation, drop ALL caps, seccomp RuntimeDefault. `enforce` rejects; `warn`/`audit` help migration. Test:
```bash
kubectl -n payments-dev run bad --image=nginx
```
Expected: `violates PodSecurity "restricted:latest": allowPrivilegeEscalation != false, unrestricted capabilities, runAsNonRoot != true, seccompProfile`. (A default `nginx` pod has no securityContext, so it is rejected.)

## Step 2 - ResourceQuota and LimitRange
```bash
cat <<'EOF' | kubectl apply -n payments-dev -f -
apiVersion: v1
kind: ResourceQuota
metadata: { name: compute }
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.memory: 16Gi
    pods: "50"
    services.loadbalancers: "0"
    persistentvolumeclaims: "10"
---
apiVersion: v1
kind: LimitRange
metadata: { name: defaults }
spec:
  limits:
    - type: Container
      default:        { memory: 256Mi }
      defaultRequest: { cpu: 50m, memory: 128Mi }
      max:            { cpu: "2", memory: 4Gi }
EOF
kubectl -n payments-dev describe quota compute
```
Why both: a quota requires pods to declare requests/limits, and the LimitRange injects defaults so teams do not get blocked by `must specify requests`. Test by creating a pod requesting 10 CPUs: rejected with `exceeded quota`.

## Step 3 - Default-deny NetworkPolicy + explicit allows
```bash
cat <<'EOF' | kubectl apply -n payments-dev -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny }
spec: { podSelector: {}, policyTypes: [Ingress, Egress] }
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-dns }
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
    - to:
        - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: kube-system } }
          podSelector: { matchLabels: { k8s-app: kube-dns } }
      ports: [{ port: 53, protocol: UDP }, { port: 53, protocol: TCP }]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-from-ingress }
spec:
  podSelector: { matchLabels: { app: payments } }
  policyTypes: [Ingress]
  ingress:
    - from: [{ namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: ingress-nginx } } }]
      ports: [{ port: 8080 }]
EOF
```
Test isolation (a compliant test pod needs a securityContext):
```bash
kubectl create ns other-team
kubectl -n payments-dev create deployment web --image=nginxinc/nginx-unprivileged:1.27 --port=8080
kubectl -n payments-dev expose deployment web --port=80 --target-port=8080
kubectl -n other-team run curl --rm -it --restart=Never --image=curlimages/curl -- curl -m 3 http://web.payments-dev
```
Expected: timeout (denied). Delete `default-deny` and retry: success. Interview point: default-deny is applied by the platform on every tenant namespace, and teams add allows in their own repos. Cilium extras (`CiliumNetworkPolicy` L7/FQDN egress) are in Lab 04.

## Step 4 - RBAC: team roles from IdP groups
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: developer, namespace: payments-dev }
rules:
  - apiGroups: ["", apps, batch]
    resources: [pods, pods/log, deployments, replicasets, jobs, cronjobs, services, configmaps, events]
    verbs: [get, list, watch]
  - apiGroups: [""]
    resources: [pods/exec, pods/portforward]
    verbs: []                         # not granted in prod
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { name: team-payments-developers, namespace: payments-dev }
subjects: [{ kind: Group, name: "team-payments", apiGroup: rbac.authorization.k8s.io }]
roleRef: { kind: Role, name: developer, apiGroup: rbac.authorization.k8s.io }
EOF
kubectl auth can-i list pods -n payments-dev --as dev1 --as-group team-payments
kubectl auth can-i delete pods -n payments-dev --as dev1 --as-group team-payments
kubectl auth can-i list pods -n other-team --as dev1 --as-group team-payments
```
Expected: yes / no / no. On AKS bind the Entra group object ID with Azure RBAC role assignments scoped to namespace (`Azure Kubernetes Service RBAC Reader` etc.). Writes in prod flow through GitOps only: humans read, ArgoCD writes.

## Step 5 - Delivering the bundle (this is the platform product)
Put these manifests under `tenants/<team>/<env>/` in `acme-gitops` (or a Helm chart `tenant`), generated by the Backstage template (Module 01) and applied by an ApplicationSet owned by the platform project, whose AppProject allows `Namespace`, `ResourceQuota`, `LimitRange`, `NetworkPolicy`, `RoleBinding` kinds that tenant projects are blacklisted from (Module 03 Lab 01).

## Step 6 - Noisy-neighbour test
```bash
kubectl -n payments-dev create deployment hog --image=busybox:1.36 -- sh -c "while true; do :; done"
kubectl -n payments-dev set resources deploy hog --requests=cpu=50m --limits=memory=64Mi
kubectl top pods -n payments-dev
```
Without CPU limits a pod may burst; CPU limits cause throttling. Platform stance (state it): set memory limits = requests, CPU requests yes, CPU limits usually no (avoid throttling), with quotas on requests and node-level protections (system-reserved, eviction thresholds).

## Break it on purpose
1. Label the namespace `pod-security.kubernetes.io/enforce: privileged` and deploy a privileged pod: show it succeeds; then add the Kyverno rule from Lab 03 that forbids teams relabeling PSS (tenants cannot edit namespace labels anyway: RBAC on `namespaces` is cluster-level).
2. Remove DNS allow: pods fail name resolution only: classic "netpol broke DNS" incident.
3. Give a team ClusterRole `edit` via ClusterRoleBinding by mistake and show cross-namespace access with `auth can-i`.

## Cleanup
```bash
kubectl delete ns payments-dev other-team
```

## Talking points
Namespace-as-tenant bundle, PSS restricted, quota + LimitRange pairing, default-deny + DNS, RBAC from IdP groups, read-only humans in prod, hard multi-tenancy limits (shared kernel/control plane, use separate clusters or vCluster/Kata for hostile tenants).
