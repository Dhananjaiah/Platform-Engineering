# Lab 04-04 - Networking: Cilium policy, Gateway API, cert-manager, ExternalDNS, service mesh mTLS

## Goal
Expose a service through Gateway API with automatic TLS, restrict traffic with Cilium L3/L4/L7 and FQDN egress policies, understand ExternalDNS, and see mesh mTLS (Linkerd) at a basic level.

## Prereqs
`dev` or `hub` cluster with Cilium (Module 00) and the `payments-dev` workload.

## Step 1 - Gateway API with a controller (Cilium or NGINX Gateway Fabric)
```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/standard-install.yaml
helm upgrade cilium cilium/cilium --version 1.17.4 -n kube-system --reuse-values \
  --set gatewayAPI.enabled=true --set kubeProxyReplacement=true
kubectl -n kube-system rollout restart deploy/cilium-operator ds/cilium
kubectl get gatewayclass
```
Expected: `cilium` GatewayClass `ACCEPTED True`. (On kind, `kubeProxyReplacement` needs `k8sServiceHost/Port` set; use NGINX Gateway Fabric if you prefer: `helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric -n nginx-gateway --create-namespace`.)
Why Gateway API over Ingress: role separation (platform owns `Gateway`, teams own `HTTPRoute`), portable features (header match, weights, redirects) without controller-specific annotations, TLS and `ReferenceGrant` for cross-namespace safety.

## Step 2 - Platform-owned Gateway, team-owned HTTPRoute
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata: { name: shared, namespace: gateway-system }
spec:
  gatewayClassName: cilium
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      hostname: "*.apps.127.0.0.1.nip.io"
      tls: { mode: Terminate, certificateRefs: [{ name: wildcard-apps-tls }] }
      allowedRoutes:
        namespaces: { from: Selector, selector: { matchLabels: { gateway-access: "true" } } }
EOF
kubectl label ns payments-dev gateway-access=true
cat <<'EOF' | kubectl apply -n payments-dev -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: { name: payments }
spec:
  parentRefs: [{ name: shared, namespace: gateway-system }]
  hostnames: ["payments.apps.127.0.0.1.nip.io"]
  rules:
    - matches: [{ path: { type: PathPrefix, value: / } }]
      backendRefs: [{ name: payments, port: 80 }]
EOF
kubectl -n payments-dev get httproute payments -o jsonpath='{.status.parents[0].conditions}'
```
Expected: `Accepted: True`. Failure drill: remove the namespace label; the route becomes `NotAllowedByListeners`: that is the tenancy control.

## Step 3 - cert-manager with a ClusterIssuer
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata: { name: selfsigned }
spec: { selfSigned: {} }
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata: { name: wildcard-apps, namespace: gateway-system }
spec:
  secretName: wildcard-apps-tls
  dnsNames: ["*.apps.127.0.0.1.nip.io"]
  issuerRef: { name: selfsigned, kind: ClusterIssuer }
  duration: 2160h
  renewBefore: 360h
EOF
kubectl -n gateway-system get certificate -w
curl -sk https://payments.apps.127.0.0.1.nip.io:8443 || true
```
Production issuers: ACME DNS-01 against Azure DNS/Route53 with workload identity (wildcards, no inbound HTTP), or internal CA (Vault/Private CA) for mTLS. Monitor expiry: alert on `certmanager_certificate_expiration_timestamp_seconds - time() < 14*24*3600` and on `Ready=False`.
```bash
kubectl describe certificate wildcard-apps -n gateway-system | tail -20
kubectl get certificaterequest,order,challenge -A
```
Debug chain to recite: Certificate -> CertificateRequest -> Order -> Challenge.

## Step 4 - ExternalDNS (conceptual + dry run)
```bash
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
helm install external-dns external-dns/external-dns -n external-dns --create-namespace \
  --set provider.name=azure --set policy=upsert-only --set txtOwnerId=aks-acme-dev \
  --set "sources={gateway-httproute,service}" --set domainFilters={apps.acme.example} --set dryRun=true 2>&1 | tail -3
```
Needs a DNS zone and workload identity to write records (`DNS Zone Contributor` on the zone only). Use `policy=upsert-only` (never delete) at first, `txtOwnerId` per cluster to avoid clusters stomping on each other's records, `domainFilters` to restrict. Without a cloud zone, run with `--set provider.name=inmemory` and watch logs: `kubectl -n external-dns logs deploy/external-dns`.

## Step 5 - Cilium policy beyond K8s NetworkPolicy
```bash
cat <<'EOF' | kubectl apply -n payments-dev -f -
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata: { name: payments-egress }
spec:
  endpointSelector: { matchLabels: { app: payments } }
  egress:
    - toEndpoints: [{ matchLabels: { "k8s:io.kubernetes.pod.namespace": kube-system, "k8s-app": kube-dns } }]
      toPorts: [{ ports: [{ port: "53", protocol: ANY }], rules: { dns: [{ matchPattern: "*" }] } }]
    - toFQDNs: [{ matchName: "api.github.com" }]
      toPorts: [{ ports: [{ port: "443", protocol: TCP }] }]
EOF
```
FQDN egress policy: allow a specific external API, deny everything else. The DNS rule lets Cilium learn IPs. L7: `rules: http: [{ method: GET, path: "/health" }]` on ingress.
Observe drops with Hubble:
```bash
cilium hubble enable --ui    # or: helm upgrade cilium ... --set hubble.relay.enabled=true --set hubble.ui.enabled=true
hubble observe -n payments-dev --verdict DROPPED -f
```
Why Hubble matters: "why can't service A reach B" is answered in seconds with verdicts and policy names.

## Step 6 - Service mesh basics (Linkerd mTLS)
```bash
curl -sL https://run.linkerd.io/install-edge | sh   # CLI; pin versions in real use
linkerd check --pre
linkerd install --crds | kubectl apply -f - && linkerd install | kubectl apply -f -
linkerd check
kubectl -n payments-dev annotate ns payments-dev linkerd.io/inject=enabled
kubectl -n payments-dev rollout restart deploy
linkerd viz install | kubectl apply -f - && linkerd viz stat deploy -n payments-dev
linkerd viz edges deploy -n payments-dev
```
Expected: edges show `SECURED` (mTLS). When to adopt a mesh: need mTLS everywhere with identity, retries/timeouts, traffic shifting, rich L7 telemetry. Costs: sidecars (or ambient/Cilium mesh), upgrade complexity, debugging layers. Alternatives: Cilium mTLS/WireGuard encryption, Istio ambient (no sidecars). Say "I would not adopt a mesh for its own sake; adopt when a requirement (mTLS compliance, zero-trust, canary L7) needs it."

## Step 7 - CNI decision (be ready)
| | Azure CNI (Overlay) | Azure CNI (VNet) | Calico | Cilium |
|---|---|---|---|---|
| Pod IPs | overlay (scales to 250k) | VNet IPs (planning pain) | depends | overlay/VNet |
| Policy | Azure NPM / Cilium / Calico | same | rich | L3-L7, FQDN, identity-based |
| Observability | basic | basic | basic | Hubble (flows, DNS, drops) |
| Notes | default sane choice | direct VNet reachability for pods | mature | eBPF, kube-proxy replacement, mesh features |

## Break it on purpose
1. Create a Certificate with a wrong issuerRef; `kubectl describe` shows `issuer not found`; follow the chain.
2. Apply the FQDN policy and `curl https://example.com` from the pod: blocked; `hubble observe` shows `policy-verdict:none DROPPED`.
3. HTTPRoute with a `backendRef` to a service in another namespace without a `ReferenceGrant`: `RefNotPermitted`.

## Cleanup
```bash
kubectl delete httproute payments -n payments-dev; kubectl delete cnp payments-egress -n payments-dev
linkerd viz uninstall | kubectl delete -f -; linkerd uninstall | kubectl delete -f -
```

## Talking points
Gateway API role split, wildcard certs via DNS-01 with workload identity, ExternalDNS ownership/policies, identity- and FQDN-aware policies with Hubble debugging, mesh only when needed, CNI trade-offs.
