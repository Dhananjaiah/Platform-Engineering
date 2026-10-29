# Lab 06-04 - Self-hosted runners at scale with Actions Runner Controller (ARC)

## Goal
Run ephemeral, autoscaling runners on Kubernetes using ARC runner scale sets, assign them via runner groups, add caching, and observe queue time.

## Prereqs
kind `hub` cluster (Module 00). A GitHub App for ARC auth (preferred over PAT).

## Step 1 - GitHub App for ARC
GitHub org > Settings > Developer settings > GitHub Apps > New: permissions `Repository: Actions (read), Administration (read/write), Metadata (read)`, `Organization: Self-hosted runners (read/write)`. Generate a private key, install on the org, note App ID and Installation ID.
```bash
kubectl create namespace arc-systems
kubectl create namespace arc-runners
kubectl -n arc-runners create secret generic arc-github-app \
  --from-literal=github_app_id=<APP_ID> \
  --from-literal=github_app_installation_id=<INSTALLATION_ID> \
  --from-file=github_app_private_key=./arc-app.private-key.pem
```
Why an App: scoped permissions, higher API rate limit, no human-tied PAT that expires or leaves with an employee.

## Step 2 - Install the controller, then a scale set
```bash
helm install arc oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller \
  --version 0.12.1 -n arc-systems
kubectl -n arc-systems rollout status deploy/arc-gha-rs-controller
```
```bash
cat > values-runners.yaml <<'EOF'
githubConfigUrl: https://github.com/acme-lab-<you>
githubConfigSecret: arc-github-app
runnerGroup: platform-linux
minRunners: 0
maxRunners: 10
containerMode:
  type: dind
template:
  spec:
    containers:
      - name: runner
        image: ghcr.io/actions/actions-runner:latest   # pin a version tag/digest in production
        command: ["/home/runner/run.sh"]
        resources:
          requests: { cpu: "1", memory: 2Gi }
          limits:   { cpu: "2", memory: 4Gi }
EOF
helm install linux-small oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  --version 0.12.1 -n arc-runners -f values-runners.yaml
kubectl -n arc-runners get autoscalingrunnersets,pods
```
Pin the chart version you tested; controller and scale-set versions must match. Expected: listener pod in `arc-systems`, zero runner pods (minRunners 0).

## Step 3 - Runner group and targeting
Create the group in org settings (or API) and restrict to selected repositories:
```bash
gh api -X POST /orgs/acme-lab-<you>/actions/runner-groups \
  -f name=platform-linux -f visibility=selected \
  -F 'selected_repository_ids[]='$(gh api repos/acme-lab-<you>/payments-api --jq .id)
```
In a workflow: `runs-on: linux-small` (the scale-set release name is the label).
```bash
kubectl -n arc-runners get pods -w &
gh workflow run ci.yml -R acme-lab-<you>/payments-api
```
Expected: an ephemeral runner pod appears, runs one job, is deleted. Ephemeral = no state leakage between jobs, no persistent credentials.

## Step 4 - Docker builds without privileged DinD (production view)
`dind` is simple but privileged. Options: Kaniko/BuildKit rootless, or a dedicated build node pool with taints, or hosted large runners for builds. Place runners on a tainted pool:
```yaml
template:
  spec:
    nodeSelector: { pool: ci }
    tolerations: [{ key: ci, operator: Exists, effect: NoSchedule }]
```
Network-isolate runners (NetworkPolicy egress to registry, GitHub, package mirror only). Never run untrusted fork PRs on self-hosted runners in public repos.

## Step 5 - Speed and cost
| Lever | How |
|-------|-----|
| Dependency cache | `actions/cache` or setup-* `cache:`; keyed by lockfile hash |
| Docker layer cache | `cache-from/to: type=gha` or registry cache; BuildKit cache mounts |
| Warm pool | `minRunners: 2` during business hours (patch via CronJob/KEDA) to cut cold start |
| Right-size | check pod CPU/mem with `kubectl top pods -n arc-runners` during jobs; set requests accordingly |
| Spot/cheap nodes | CI pool on spot with `maxRunners` cap; jobs are retryable |
| Mirror | local pull-through registry & package proxy (Artifactory/Nexus) |

Measure cold start: time from `queued` to `in_progress` in the next lab.

## Step 6 - Operate it
```bash
kubectl -n arc-systems logs deploy/arc-gha-rs-controller --tail=50
helm upgrade linux-small oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set --version 0.12.1 -n arc-runners -f values-runners.yaml
```
Upgrade order: uninstall/upgrade scale sets then controller per release notes (they are tightly coupled); drain by setting `maxRunners: 0` and waiting for pods to finish.

## Break it on purpose
1. Set `maxRunners: 1` and enqueue 5 jobs; watch queue time grow (this is the queue-time alert in Lab 05).
2. Delete the secret and see the listener fail auth; restore.
3. Set memory limit 256Mi and run `npm ci`; OOMKilled runner pod mid-job: `kubectl describe pod` shows `OOMKilled`; job shows "runner lost communication".

## Cleanup
```bash
helm uninstall linux-small -n arc-runners && helm uninstall arc -n arc-systems
kubectl delete ns arc-runners arc-systems
```

## Talking points
Ephemeral runners, scale-to-zero with a warm floor, runner groups for isolation, tainted pools, network egress limits, no fork PRs on self-hosted, version coupling between controller and scale set.
