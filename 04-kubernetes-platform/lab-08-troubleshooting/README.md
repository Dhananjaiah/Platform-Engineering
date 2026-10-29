# Lab 04-08 - Troubleshooting dojo (manual fault injection)

## Goal
Inject six real production faults by hand, diagnose each with a fixed method, and be able to narrate it. Each drill: **Symptom -> Commands -> Root cause -> Fix -> Prevention**.

## Method (say this first in any troubleshooting interview)
1. Scope: what, since when, who is affected, what changed (deploys, config, node events).
2. Top-down: `kubectl get pods -o wide` -> `describe` (Events!) -> `logs` (`--previous`) -> node -> network -> dependencies.
3. Hypothesis, one change at a time, verify with data.
4. Mitigate before root-causing when users are hurting.

```bash
kubectl create ns dojo
```

## Drill 1 - OOMKilled
```bash
kubectl -n dojo run mem --image=polinux/stress --restart=Never --overrides='{"spec":{"containers":[{"name":"mem","image":"polinux/stress","command":["stress","--vm","1","--vm-bytes","300M","--vm-hang","0"],"resources":{"limits":{"memory":"100Mi"},"requests":{"memory":"50Mi"}}}]}}'
kubectl -n dojo get pod mem -w
kubectl -n dojo describe pod mem | grep -A6 "Last State"
```
Symptom: `OOMKilled`, exit code 137, restarts climbing. Root cause: container exceeded its **memory limit** (cgroup kill, no graceful stop). Check usage vs limit: `kubectl top pod`, Prometheus `container_memory_working_set_bytes`. Fix: right-size from VPA recommendation/p99 usage, fix leaks (heap dumps), for JVM set `-XX:MaxRAMPercentage=75`, `GOMEMLIMIT`. Prevention: memory limit = request, alert on `working_set / limit > 0.9`, load tests. Distinguish node-level OOM (kernel kills, events `System OOM encountered`) from container OOM.

## Drill 2 - CrashLoopBackOff
```bash
kubectl -n dojo create deployment crash --image=busybox:1.36 -- sh -c "echo starting; cat /config/app.yaml"
kubectl -n dojo get pods -w
kubectl -n dojo logs deploy/crash --previous
kubectl -n dojo describe pod -l app=crash | tail -15
```
Symptom: restarts with exponential backoff (10 s -> 5 min). Root cause here: missing file/config (`No such file`). Other common: bad command/args, failed probe (liveness kills), missing secret/env, permission denied due to non-root + volume ownership, dependency unreachable at startup. Fix: supply the ConfigMap/mount, correct command. Tools: `kubectl debug` for an ephemeral container:
```bash
kubectl -n dojo debug -it deploy/crash --image=busybox:1.36 --target=busybox --copy-to=crash-debug -- sh
```
Prevention: startup probes, config validation in CI (`kubeconform`, schema), init containers for dependencies only if truly required, ArgoCD health gating.

## Drill 3 - ImagePullBackOff / ErrImagePull
```bash
kubectl -n dojo create deployment pull --image=ghcr.io/acme-lab-<you>/does-not-exist:1.0
kubectl -n dojo describe pod -l app=pull | grep -A8 Events
```
Root cause candidates: wrong tag/digest, private registry without `imagePullSecret`, registry rate limit (Docker Hub 429), network/DNS egress block, node cannot auth to ACR (AKS needs `AcrPull` for the kubelet identity: `az aks update --attach-acr`). Fix: correct image ref, add pull secret/identity. Prevention: Kyverno verify images, pull-through cache/mirror to avoid rate limits, digest pinning, admission checks that tag exists.

## Drill 4 - Pending pods (scheduling)
```bash
kubectl -n dojo create deployment big --image=nginxinc/nginx-unprivileged:1.27
kubectl -n dojo set resources deploy big --requests=cpu=100,memory=1000Gi
kubectl -n dojo get pods; kubectl -n dojo describe pod -l app=big | grep -A5 Events
```
Events: `0/3 nodes are available: 3 Insufficient memory`. Others: taints without tolerations, node selectors/affinity mismatch, PVC unbound/zone mismatch, quota exceeded (`kubectl describe quota`), topology spread unsatisfiable. Fix by reading the exact reason, not guessing. Tools: `kubectl get events --sort-by=.lastTimestamp -A`.

## Drill 5 - Node pressure and eviction
```bash
kubectl describe node <worker> | grep -A8 "Conditions"
kubectl get events -A --field-selector reason=Evicted
kubectl top nodes
```
Simulate disk pressure with a pod writing to `emptyDir` until the node reports `DiskPressure` (use a small kind node; stop quickly). Symptoms: `Evicted` pods, `node.kubernetes.io/disk-pressure` taint, pods rescheduled. Diagnose which pod caused it (`du`, ephemeral-storage requests). Prevention: ephemeral-storage requests/limits, log rotation, image GC thresholds, node monitoring (`node_filesystem_avail_bytes`), pods with QoS awareness (BestEffort evicted first, Guaranteed last), `system-reserved`/`kube-reserved`.

## Drill 6 - DNS and NetworkPolicy
```bash
kubectl -n dojo create deployment api --image=nginxinc/nginx-unprivileged:1.27 && kubectl -n dojo expose deploy api --port=80 --target-port=8080
kubectl -n dojo run t --rm -it --restart=Never --image=busybox:1.36 -- wget -qO- -T3 http://api
# now apply default-deny (Lab 02) in dojo and repeat:
kubectl -n dojo apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny }
spec: { podSelector: {}, policyTypes: [Ingress, Egress] }
EOF
kubectl -n dojo run t --rm -it --restart=Never --image=busybox:1.36 -- wget -qO- -T3 http://api
```
Two failure modes: `bad address 'api'` (DNS blocked or CoreDNS down) vs timeout (connection blocked). Distinguish them: `nslookup api` (DNS) vs `wget http://<podIP>:8080` (connectivity). Checks: `kubectl -n kube-system get pods -l k8s-app=kube-dns`, CoreDNS logs, `ndots:5` latency issue (use FQDNs with trailing dot, tune `dnsConfig`), conntrack table full, `hubble observe --verdict DROPPED`. Fix: add DNS egress allow + app allows.

## Bonus drills
- **Service has no endpoints**: selector label mismatch: `kubectl get endpointslices`, compare `svc.spec.selector` with pod labels, readiness failing.
- **Ingress 502/504**: backend not ready, port mismatch, upstream timeout; `kubectl logs ingress-nginx-controller`.
- **Cert expired**: `kubectl get certificate`, cert-manager chain.
- **kubectl hangs / API slow**: apiserver audit/metrics, webhook timeouts (a dead admission webhook with `failurePolicy: Fail`), etcd latency, `kubectl get --raw /readyz?verbose`.

## Cheat sheet to memorise
```bash
kubectl get events -A --sort-by=.lastTimestamp | tail -30
kubectl describe pod <p> | sed -n '/Events/,$p'
kubectl logs <p> -c <c> --previous --tail=200
kubectl get pod <p> -o jsonpath='{.status.containerStatuses[*].lastState}'
kubectl top pod --containers -n <ns> ; kubectl top node
kubectl auth can-i --list --as system:serviceaccount:<ns>:<sa>
kubectl debug node/<node> -it --image=busybox:1.36
kubectl get --raw /readyz?verbose
```
Exit codes: 0 ok, 1 app error, 137 SIGKILL (OOM or kill), 139 segfault, 143 SIGTERM, 126/127 command not executable/not found.

## Cleanup
```bash
kubectl delete ns dojo
```

## Talking points
A method before commands, Events first, previous logs, distinguishing DNS vs connectivity, resource semantic differences (requests vs limits), prevention via limits/probes/policies and alerts.
