# Lab 03-08 - ArgoCD lifecycle: drift, sharding, notifications, disaster recovery

## Goal
Operate ArgoCD like a production service: alerts on drift/failed sync via notifications, shard the controller, back up and rebuild ArgoCD from scratch, and upgrade safely.

## Prereqs
Labs 01-03 complete (ApplicationSets and a few apps running).

## Step 1 - Notifications (Slack webhook or any webhook receiver)
For a local receiver, run a webhook catcher: `docker run -d -p 9999:8080 --name hook tarampampam/webhook-tester` (or use a Slack incoming webhook URL).
```bash
kubectl -n argocd patch cm argocd-notifications-cm --type merge -p '{"data":{
"service.webhook.local":"url: http://host.docker.internal:9999/abc\nheaders:\n- name: Content-Type\n  value: application/json",
"template.app-degraded":"webhook:\n  local:\n    method: POST\n    body: |\n      {\"app\":\"{{.app.metadata.name}}\",\"health\":\"{{.app.status.health.status}}\",\"sync\":\"{{.app.status.sync.status}}\"}",
"trigger.on-degraded":"- when: app.status.health.status == '\''Degraded'\''\n  send: [app-degraded]",
"subscriptions":"- recipients: [webhook:local]\n  triggers: [on-degraded]\n"}}'
kubectl -n argocd rollout restart deploy/argocd-notifications-controller
```
(Subscribe per-app instead with annotation `notifications.argoproj.io/subscribe.on-degraded.slack: team-payments-alerts` so each team receives only theirs.) Test: apply the bad digest from Lab 05; the webhook receives `Degraded`. Triggers worth having: `on-sync-failed`, `on-health-degraded`, `on-sync-status-unknown`, `on-deployed` (feeds DORA deployment events, Module 02).

## Step 2 - Metrics and alerts for ArgoCD itself
```bash
kubectl -n argocd get servicemonitor 2>/dev/null; kubectl -n argocd get svc | grep metrics
```
Key PromQL:
```
sum by (name) (argocd_app_info{sync_status="OutOfSync"}) > 0        # drift
argocd_app_info{health_status!~"Healthy|Progressing"}                # unhealthy
histogram_quantile(0.95, sum by (le) (rate(argocd_app_reconcile_bucket[5m])))   # controller lag
increase(argocd_app_sync_total{phase="Error"}[10m]) > 0
```
Alert when reconcile p95 > 60 s or git fetch fails (`argocd_git_request_total{request_type="fetch"}` errors).

## Step 3 - Scale: tuning and sharding the application controller
Symptoms at scale (1000+ apps): slow refresh, high memory on controller, repo-server CPU spikes. Levers:
```bash
helm upgrade argocd argo/argo-cd --version 8.0.0 -n argocd -f argocd-values.yaml \
  --set controller.replicas=3 \
  --set controller.env[0].name=ARGOCD_CONTROLLER_REPLICAS --set-string controller.env[0].value=3 \
  --set configs.params."controller\.status\.processors"=50 \
  --set configs.params."controller\.operation\.processors"=25 \
  --set configs.params."timeout\.reconciliation"=180s \
  --set repoServer.replicas=3
kubectl -n argocd get pods -l app.kubernetes.io/name=argocd-application-controller
```
Controller shards clusters across replicas (`ARGOCD_CONTROLLER_REPLICAS` must equal the StatefulSet replica count; algorithm `legacy`, `round-robin`, `consistent-hashing`). Sharding is *per cluster*: one huge cluster cannot be split. Other levers: webhooks instead of polling (`timeout.reconciliation` up), `manifest-generate-paths`, repo-server cache and parallelism limits (`reposerver.parallelism.limit`), exclude noisy resources (`resource.exclusions` for Events, EndpointSlices, cilium identities), separate ArgoCD per tier.
Verify shard assignment: `argocd admin cluster stats`.

## Step 4 - Back up
What is state? Applications/AppProjects/ApplicationSets (CRs, in etcd, but reproducible from git if you bootstrap), cluster secrets (credentials), repo creds, `argocd-cm`/`rbac-cm`/`secret`, TLS. Everything else is cache.
```bash
argocd admin export -n argocd > argocd-backup-$(date +%F).yaml
ls -lh argocd-backup-*.yaml
```
The export contains secrets: encrypt (`sops`/age) and store off-cluster; schedule as a CronJob writing to object storage (or Velero with the `argocd` namespace). Better: keep *all* ArgoCD config in git (Helm values + app-of-apps bootstrap) and only back up what git cannot hold (cluster credentials, repo keys, SSO secrets, which also live in your secret manager via ESO).

## Step 5 - Disaster recovery drill (timed)
Scenario: hub cluster lost. Spokes keep running (workloads are unaffected; deploys stop).
1. Create a new hub (`kind create cluster --name hub2 ...`).
2. Install ArgoCD with the same pinned chart/values.
3. Restore state:
```bash
argocd admin import -n argocd - < argocd-backup-2026-xx-xx.yaml
```
   or re-apply the bootstrap root Application and re-add clusters (`argocd cluster add`) + repo creds from your secret store.
4. Apps reappear and show `Synced` (no changes), because the spokes already match git. Resource tracking labels/annotations must be identical (same `application.resourceTrackingMethod`) otherwise Argo thinks resources are untracked.
Measure RTO. Write it as a runbook. Practise quarterly.
Risk to mention: never run two hubs with `prune: true` pointed at the same cluster and path.

## Step 6 - Upgrade runbook
```bash
helm diff upgrade argocd argo/argo-cd --version <new> -n argocd -f argocd-values.yaml   # needs helm-diff plugin
kubectl apply --server-side --force-conflicts -f <crds of new version>   # chart CRD handling per release notes
helm upgrade argocd argo/argo-cd --version <new> -n argocd -f argocd-values.yaml
```
Read the upgrade notes for breaking changes (RBAC defaults, deprecated fields, resource tracking). Upgrade nonprod hub first; keep the previous chart version one `helm rollback` away; export a backup before.

## Step 7 - Drift policy
Define per env: dev `selfHeal: true`; stg `selfHeal: true`; prd `selfHeal: true` but human-approved *changes via git*; for emergencies a documented break-glass: pause with `argocd app set --sync-policy none`, apply the fix, and open a reconciling PR within 24 h (alert on any app with auto-sync disabled > 4 h).

## Break it on purpose
1. Stop the repo-server (`kubectl -n argocd scale deploy argocd-repo-server --replicas=0`): apps show `Unknown`; running workloads unaffected; restore.
2. Exceed limits: create 300 ApplicationSet-generated apps pointing at one path and watch reconcile p95 and repo-server CPU; apply `manifest-generate-paths` and compare.
3. Change `ARGOCD_CONTROLLER_REPLICAS` without matching replicas and observe unmanaged clusters; fix.

## Cleanup
`docker rm -f hook`

## Talking points
Alert on degraded/out-of-sync and on ArgoCD's own health; sharding by cluster; webhook > polling; backups of credentials not of generated state; rebuild-from-git DR with timed drill; upgrade nonprod first.
