# Lab 04-05 - Autoscaling: HPA, VPA, KEDA, node autoscaling

## Goal
Scale pods on CPU and on external signals, right-size with VPA recommendations, and reason about Cluster Autoscaler vs Karpenter.

## Prereqs
`dev` cluster; metrics-server:
```bash
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm install metrics-server metrics-server/metrics-server -n kube-system --set args={--kubelet-insecure-tls}
kubectl top nodes
```
(`--kubelet-insecure-tls` is for kind only.)

## Step 1 - HPA on CPU
```bash
kubectl create ns scale-demo
kubectl -n scale-demo create deployment php --image=registry.k8s.io/hpa-example
kubectl -n scale-demo set resources deploy php --requests=cpu=100m,memory=64Mi --limits=memory=128Mi
kubectl -n scale-demo expose deployment php --port=80
kubectl -n scale-demo autoscale deployment php --cpu-percent=50 --min=2 --max=10
kubectl -n scale-demo get hpa -w &
kubectl -n scale-demo run load --image=busybox:1.36 --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://php; done"
```
Expected: CPU % rises, replicas climb to 10, then (after the 5 min stabilization window) fall back. Edit for production using `behavior`:
```bash
kubectl -n scale-demo patch hpa php --type merge -p '{"spec":{"behavior":{"scaleDown":{"stabilizationWindowSeconds":300,"policies":[{"type":"Percent","value":25,"periodSeconds":60}]},"scaleUp":{"policies":[{"type":"Pods","value":4,"periodSeconds":30}]}}}}'
```
Rules: HPA utilisation is relative to **requests**, so wrong requests = wrong scaling. Don't combine CPU HPA and VPA `Auto` on the same resource. Set `minReplicas >= 2` for HA with PDB.

## Step 2 - VPA in recommendation mode (right-sizing source)
```bash
helm repo add fairwinds-stable https://charts.fairwinds.com/stable
helm install vpa fairwinds-stable/vpa -n vpa --create-namespace --set updater.enabled=false --set admissionController.enabled=false
cat <<'EOF' | kubectl apply -f -
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata: { name: php, namespace: scale-demo }
spec:
  targetRef: { apiVersion: apps/v1, kind: Deployment, name: php }
  updatePolicy: { updateMode: "Off" }
EOF
kubectl -n scale-demo describe vpa php | sed -n '/Recommendation/,$p'
```
`Off` mode produces target/lower/upper recommendations without evicting pods. Feed this into the cost report (Lab 09). Use `InPlaceOrRecreate` only where in-place resize is supported and apps tolerate restarts.

## Step 3 - KEDA (event-driven; scale to zero)
```bash
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda -n keda --create-namespace
kubectl -n keda rollout status deploy/keda-operator
```
Scale by a Prometheus query (or queue length, Kafka lag, Service Bus):
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata: { name: php-keda, namespace: scale-demo }
spec:
  scaleTargetRef: { name: php }
  minReplicaCount: 1
  maxReplicaCount: 20
  cooldownPeriod: 120
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://kps-kube-prometheus-stack-prometheus.monitoring:9090
        query: sum(rate(http_requests_total{namespace="scale-demo"}[1m]))
        threshold: "20"
EOF
kubectl -n scale-demo get scaledobject,hpa
```
KEDA creates and manages the HPA; delete the manual HPA first (one scaler per Deployment). Use cron triggers for known peaks (pre-scale), queue-based for workers, `minReplicaCount: 0` for non-critical event consumers.

## Step 4 - Node autoscaling (what to say and show)
- **Cluster Autoscaler** (AKS/EKS/GKE): scales node groups when pods are `Pending` due to insufficient resources; scales down underused nodes. Tunables: `expander`, `scale-down-unneeded`, `skip-nodes-with-local-storage`, `balance-similar-node-groups`. Requires pods to declare requests.
- **Karpenter** (EKS; AKS via node auto-provisioning): provisions right-sized nodes directly from pending pod specs across instance types, consolidates and replaces nodes, handles spot interruption, drift-based upgrades.
Simulate on kind (CA not applicable) by showing pending pods:
```bash
kubectl -n scale-demo scale deploy php --replicas=100
kubectl -n scale-demo get pods | grep -c Pending
kubectl -n scale-demo describe pod $(kubectl -n scale-demo get pods --field-selector=status.phase=Pending -o name | head -1 | cut -d/ -f2) | tail -8
```
Read `Insufficient cpu` events (what CA reacts to). On a cloud cluster, watch `kubectl get nodes -w` and `kubectl -n kube-system logs deploy/cluster-autoscaler`. Karpenter `NodePool` example (EKS):
```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata: { name: general }
spec:
  template:
    spec:
      nodeClassRef: { group: karpenter.k8s.aws, kind: EC2NodeClass, name: default }
      requirements:
        - { key: karpenter.sh/capacity-type, operator: In, values: ["spot", "on-demand"] }
        - { key: kubernetes.io/arch, operator: In, values: ["amd64"] }
        - { key: karpenter.k8s.aws/instance-category, operator: In, values: ["c", "m", "r"] }
      expireAfter: 720h
  disruption: { consolidationPolicy: WhenEmptyOrUnderutilized, consolidateAfter: 1m }
  limits: { cpu: 1000 }
```

## Step 5 - Putting it together (the interview answer)
Layered scaling: KEDA/HPA scale pods on demand signals -> pending pods trigger node autoscaler -> PDBs, topology spread and `terminationGracePeriodSeconds` make scale-down safe -> VPA/OpenCost feed right-sizing -> quotas cap runaway scale.

## Break it on purpose
1. Remove CPU requests from the Deployment: HPA shows `<unknown>/50%` and does nothing: the classic "HPA not working" cause.
2. Combine HPA and KEDA on the same target; see `conflicting scalers` events.
3. Set `scaleDown` stabilization to 0 and watch flapping under spiky load.

## Cleanup
```bash
kubectl delete ns scale-demo; helm uninstall keda -n keda; helm uninstall vpa -n vpa
```

## Talking points
Requests drive HPA and bin-packing, behavior tuning avoids flapping, VPA as advisor, KEDA for events/scale-to-zero, node autoscaler reacts to pending pods, Karpenter consolidation, always cap with quotas/limits.
