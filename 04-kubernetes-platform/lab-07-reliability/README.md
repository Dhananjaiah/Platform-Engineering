# Lab 04-07 - Reliability: PDB, probes, topology spread, Velero, node drain/upgrade, add-on management

## Goal
Make a workload survive node drains and zone loss, prove backups restore, and manage add-ons across clusters declaratively.

## Prereqs
`dev` cluster with 2+ workers (add a third: `kind` config with 3 workers and zone labels).

## Step 1 - Label nodes as zones (kind simulation)
```bash
i=0; for n in $(kubectl get nodes -l '!node-role.kubernetes.io/control-plane' -o name); do
  kubectl label $n topology.kubernetes.io/zone=zone-$((i%3)) --overwrite; i=$((i+1)); done
kubectl get nodes -L topology.kubernetes.io/zone
```

## Step 2 - A resilient Deployment (probes, spread, PDB)
```bash
kubectl create ns rel-demo
cat <<'EOF' | kubectl apply -n rel-demo -f -
apiVersion: apps/v1
kind: Deployment
metadata: { name: web }
spec:
  replicas: 4
  strategy: { rollingUpdate: { maxSurge: 25%, maxUnavailable: 0 } }
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }
    spec:
      terminationGracePeriodSeconds: 30
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector: { matchLabels: { app: web } }
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector: { matchLabels: { app: web } }
      containers:
        - name: web
          image: ghcr.io/stefanprodan/podinfo:6.7.1
          ports: [{ containerPort: 9898 }]
          resources: { requests: { cpu: 25m, memory: 32Mi }, limits: { memory: 64Mi } }
          startupProbe:  { httpGet: { path: /healthz, port: 9898 }, failureThreshold: 30, periodSeconds: 2 }
          readinessProbe: { httpGet: { path: /readyz, port: 9898 }, periodSeconds: 5 }
          livenessProbe:  { httpGet: { path: /healthz, port: 9898 }, periodSeconds: 10, failureThreshold: 3 }
          lifecycle: { preStop: { exec: { command: ["sleep", "10"] } } }
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: web }
spec: { minAvailable: 3, selector: { matchLabels: { app: web } } }
EOF
kubectl -n rel-demo get pods -o wide
```
Explain: startup probe protects slow starters from liveness kills; readiness gates traffic; liveness only for deadlocks (never depend on downstream in liveness); `preStop` sleep lets endpoint removal propagate before SIGTERM; `maxUnavailable: 0` + PDB protects capacity.

## Step 3 - Prove the PDB during a drain
```bash
NODE=$(kubectl -n rel-demo get pods -o jsonpath='{.items[0].spec.nodeName}')
kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data --timeout=120s
kubectl -n rel-demo get pods -w
kubectl uncordon $NODE
```
Expected: pods evicted one by one; at least 3 always Ready. Break: set `minAvailable: 4` with 4 replicas, and the drain hangs (`Cannot evict pod as it would violate the pod's disruption budget`). That is the "upgrade stuck for hours" incident: PDB too strict or `maxUnavailable: 0` with a single replica. Rule: PDB `maxUnavailable: 1` (or 25 %), replicas >= 2.

## Step 4 - Backup and restore with Velero (MinIO as S3)
```bash
docker run -d --name velero-minio --network kind -p 9100:9000 -e MINIO_ROOT_USER=minio -e MINIO_ROOT_PASSWORD=minio12345 quay.io/minio/minio server /data
docker run --rm --network kind --entrypoint sh quay.io/minio/mc -c "mc alias set l http://velero-minio:9000 minio minio12345 && mc mb l/velero"
cat > credentials-velero <<'EOF'
[default]
aws_access_key_id=minio
aws_secret_access_key=minio12345
EOF
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm install velero vmware-tanzu/velero -n velero --create-namespace \
  --set-file credentials.secretContents.cloud=credentials-velero \
  --set configuration.backupStorageLocation[0].name=default \
  --set configuration.backupStorageLocation[0].provider=aws \
  --set configuration.backupStorageLocation[0].bucket=velero \
  --set configuration.backupStorageLocation[0].config.region=minio \
  --set configuration.backupStorageLocation[0].config.s3ForcePathStyle=true \
  --set configuration.backupStorageLocation[0].config.s3Url=http://velero-minio:9000 \
  --set initContainers[0].name=velero-plugin-for-aws --set initContainers[0].image=velero/velero-plugin-for-aws:v1.11.0 \
  --set initContainers[0].volumeMounts[0].mountPath=/target --set initContainers[0].volumeMounts[0].name=plugins \
  --set snapshotsEnabled=false
kubectl -n velero get backupstoragelocation
```
Expected: `PHASE Available`. Take a backup, destroy, restore:
```bash
kubectl -n rel-demo create configmap important --from-literal=data=precious
velero backup create rel-demo-1 --include-namespaces rel-demo --wait
velero backup describe rel-demo-1 --details | head -30
kubectl delete ns rel-demo
velero restore create --from-backup rel-demo-1 --wait
kubectl -n rel-demo get cm important -o jsonpath='{.data.data}'
```
Expected: `precious`. Production: volume backups via CSI snapshots/`fs-backup`, cross-region bucket, schedules (`velero schedule create daily --schedule "0 2 * * *" --ttl 720h`), restore drills in a clean cluster, alert on `velero_backup_failure_total`. With GitOps, you rarely need Velero for manifests; use it for PVs, CRDs state and namespaces not in git. Databases: use native PITR (managed Postgres) as primary.

## Step 5 - Zone-failure drill
```bash
kubectl cordon <all nodes in zone-0>; kubectl drain <those nodes> --ignore-daemonsets --delete-emptydir-data
kubectl -n rel-demo get pods -o wide
```
With `DoNotSchedule` spread and `maxSkew: 1`, pods that cannot be placed remain Pending: decide per service whether `ScheduleAnyway` (availability) or strict (balance). Interview point: capacity must exist to absorb a zone loss (N+1 zones).

## Step 6 - Add-on management across many clusters
Add-ons live in `acme-gitops/addons/<name>/` (Helm values pinned), installed by one ApplicationSet per add-on class (Module 03 Lab 03), with waves per cluster label:
```
addons/
  ingress-nginx/  cert-manager/  kyverno/  external-secrets/  kube-prometheus-stack/  velero/
clusters/dev/addons.yaml        # list of add-ons + versions for this cluster
```
Upgrade process: PR bumps version in `clusters/dev`; render diff (`helm template` before/after in CI); dev soak; promote to stg then prd. Policy: each add-on has an owner, an SLO, a version support window, and a Backstage entry (`Resource`). Add a CI job comparing add-on versions across clusters to catch drift (`yq` over `clusters/*/addons.yaml`).

## Step 7 - Zero-downtime node upgrade runbook (write this down)
1. Check deprecated APIs and PDBs (`kubectl get pdb -A`, any with 0 disruptions allowed).
2. Surge new node, cordon + drain old one, honour grace period, `--timeout`.
3. Watch error-rate SLI during the upgrade; abort criteria defined.
4. Order: system pool, then user pools; one zone at a time for stateful sets.
5. Post: version table, remove old pool.

## Break it on purpose
1. Liveness probe pointing at a dependency (DB): DB blip causes all pods to restart (cascade). Fix to local-only liveness.
2. No readiness probe: rolling update sends traffic to cold pods: show 5xx during rollout with a load loop.
3. Single replica + PDB `minAvailable: 1`: drain blocks forever.

## Cleanup
```bash
helm uninstall velero -n velero; docker rm -f velero-minio; kubectl delete ns rel-demo --ignore-not-found
```

## Talking points
Probes' distinct purposes, spread + PDB + surge for zero downtime, Velero vs GitOps vs DB-native backups, tested restores, zone-failure capacity, add-ons as versioned products rolled in waves.
