# Lab 03-06 - Progressive delivery with Argo Rollouts (canary, blue-green, Prometheus analysis)

## Goal
Convert `payments` to a Rollout with a weighted canary gated by a Prometheus `AnalysisTemplate`, watch an automatic abort on a bad version, then run a blue-green.

## Prereqs
Hub (or dev) cluster with ingress-nginx and Prometheus (kube-prometheus-stack from Module 04, or quick install below).

## Step 1 - Install
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kps prometheus-community/kube-prometheus-stack -n monitoring --create-namespace \
  --set grafana.enabled=false --set alertmanager.enabled=false
helm install argo-rollouts argo/argo-rollouts -n argo-rollouts --create-namespace --set dashboard.enabled=true
kubectl -n argo-rollouts rollout status deploy/argo-rollouts
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts-linux-amd64 && sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
kubectl argo rollouts version
```
(Pin the plugin version in real use instead of `latest`.)

## Step 2 - A demo app that can be made "bad"
Use an image that exposes request metrics and can return 5xx by flag. A tiny option: `ghcr.io/stefanprodan/podinfo:6.7.1` (exposes `/metrics`, `/status/500`). Install as dev app in namespace `rollouts-demo` and create a ServiceMonitor.
```bash
kubectl create ns rollouts-demo
cat <<'EOF' | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata: { name: podinfo, namespace: rollouts-demo, labels: { release: kps } }
spec:
  selector: { matchLabels: { app: podinfo } }
  endpoints: [{ port: http, interval: 15s }]
EOF
```

## Step 3 - AnalysisTemplate (success rate from Prometheus)
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata: { name: success-rate, namespace: rollouts-demo }
spec:
  args: [{ name: service }]
  metrics:
    - name: success-rate
      interval: 30s
      count: 5
      failureLimit: 1
      successCondition: result[0] >= 0.95
      provider:
        prometheus:
          address: http://kps-kube-prometheus-stack-prometheus.monitoring:9090
          query: |
            sum(rate(http_requests_total{service="{{args.service}}",status!~"5.."}[1m]))
            /
            sum(rate(http_requests_total{service="{{args.service}}"}[1m]))
EOF
```
(Adjust metric/label names to what your app exports; podinfo exports `http_requests_total` with `status`.) Why analysis: the decision to proceed is made by SLI data, not a human watching a graph. `failureLimit: 1` tolerates one blip.
Guard against "no data = success": add `inconclusiveLimit` or a second metric that asserts traffic volume > 0.

## Step 4 - The Rollout with canary steps
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata: { name: podinfo, namespace: rollouts-demo }
spec:
  replicas: 5
  selector: { matchLabels: { app: podinfo } }
  strategy:
    canary:
      canaryService: podinfo-canary
      stableService: podinfo
      trafficRouting:
        nginx: { stableIngress: podinfo }
      steps:
        - setWeight: 10
        - pause: { duration: 1m }
        - analysis:
            templates: [{ templateName: success-rate }]
            args: [{ name: service, value: podinfo-canary }]
        - setWeight: 50
        - pause: { duration: 1m }
        - setWeight: 100
  template:
    metadata: { labels: { app: podinfo } }
    spec:
      containers:
        - name: podinfo
          image: ghcr.io/stefanprodan/podinfo:6.7.0
          ports: [{ name: http, containerPort: 9898 }]
          resources: { requests: { cpu: 25m, memory: 32Mi } }
EOF
# plus Services podinfo and podinfo-canary (selector app=podinfo) and Ingress podinfo -> service podinfo
kubectl argo rollouts get rollout podinfo -n rollouts-demo -w
```
Drive traffic: `while true; do curl -s -o /dev/null http://podinfo.127.0.0.1.nip.io:8080/; sleep 0.2; done`.

## Step 5 - Good release, then bad release
```bash
kubectl argo rollouts set image podinfo podinfo=ghcr.io/stefanprodan/podinfo:6.7.1 -n rollouts-demo
kubectl argo rollouts get rollout podinfo -n rollouts-demo -w
```
Expected: 10 % -> analysis `Successful` -> 50 % -> 100 %. Now a bad release: run a version/config that returns 500s, for example set env `PODINFO_FAULT=...` or point at an image tag that fails readiness. With errors in the canary, analysis fails:
```bash
kubectl argo rollouts get rollout podinfo -n rollouts-demo
kubectl -n rollouts-demo get analysisrun
kubectl -n rollouts-demo describe analysisrun <name>
```
Expected: `Degraded - RolloutAborted`, weight returns to 0 for canary, stable keeps 100 % traffic. Users saw at most 10 % of traffic degraded for < 2 minutes. Recover: `kubectl argo rollouts undo podinfo` or revert git.

## Step 6 - Blue-green
```yaml
strategy:
  blueGreen:
    activeService: podinfo
    previewService: podinfo-preview
    autoPromotionEnabled: false
    prePromotionAnalysis:
      templates: [{ templateName: success-rate }]
      args: [{ name: service, value: podinfo-preview }]
    scaleDownDelaySeconds: 600     # keep old pods for instant rollback
```
Promote with `kubectl argo rollouts promote podinfo`. Trade-offs vs canary: instant rollback and full pre-prod validation of the new stack, but 2x capacity and no gradual exposure; DB migrations still need expand/contract.

## Step 7 - With ArgoCD
Rollout is a CRD that ArgoCD health-checks natively. With `selfHeal`, a manual `kubectl argo rollouts promote` is allowed (it patches status/pause, not spec). Do not `ignoreDifferences` the whole spec. In dev, `analysis` can be shorter; stg/prd use the real SLO windows. Put the AnalysisTemplate in the platform chart so every service gets a standard canary.

## Break it on purpose
1. Query typo in the AnalysisTemplate: the run is `Error`; the rollout pauses, not promotes (fail-safe). Fix and `retry`.
2. Remove traffic generator: `result` is NaN; understand "no data" semantics and add a traffic-volume metric.
3. Set `maxUnavailable` too aggressive with few replicas and see capacity dip during surge.

## Cleanup
`kubectl delete ns rollouts-demo`

## Talking points
SLI-gated promotion, abort limits blast radius, canary vs blue-green trade-offs, traffic routers (NGINX, Gateway API, Istio), the "no data" trap, DB compatibility, native ArgoCD integration.
