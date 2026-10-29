# Lab 06-05 - Pipeline metrics: duration, failure rate, queue time, flaky tests

## Goal
Pull workflow-run data from the GitHub API, compute the four numbers every CI platform owner must know, and show them in Grafana. All hand-typed commands.

## Prereqs
`gh`, `jq`, hub cluster with Grafana (Module 04 observability; or `docker run -d -p 3000:3000 grafana/grafana`).

## Step 1 - Raw data
```bash
ORG=acme-lab-<you>
gh api "/repos/$ORG/payments-api/actions/runs?per_page=100&status=completed" \
  --jq '.workflow_runs[] | {id, name, conclusion, created_at, run_started_at, updated_at, run_attempt}' | head -30
```
Fields: `created_at` (queued), `run_started_at` (started), `updated_at` (finished). Queue time = started - created. Duration = updated - started.

## Step 2 - Compute with jq
```bash
gh api --paginate "/repos/$ORG/payments-api/actions/runs?per_page=100&status=completed" --jq '.workflow_runs[]' \
 | jq -s '
  def secs(a;b): ((b|fromdate) - (a|fromdate));
  { runs: length,
    failure_rate: ((map(select(.conclusion=="failure"))|length) / length),
    p50_duration_s: (map(secs(.run_started_at;.updated_at))|sort|.[length/2|floor]),
    p95_duration_s: (map(secs(.run_started_at;.updated_at))|sort|.[(length*0.95)|floor]),
    p95_queue_s:    (map(secs(.created_at;.run_started_at))|sort|.[(length*0.95)|floor]),
    retried: (map(select(.run_attempt>1))|length) }'
```
Interpret: high queue p95 = runner capacity (Lab 04); high duration = caching/test parallelism; retries and "failure then success on same SHA" = flakiness.

## Step 3 - Flaky test detection (same SHA, different outcome)
```bash
gh api --paginate "/repos/$ORG/payments-api/actions/runs?per_page=100&status=completed" --jq '.workflow_runs[] | [.head_sha,.conclusion,.run_attempt]|@tsv' \
 | sort | awk '{c[$1]=c[$1]" "$2} END{for(s in c) if (c[s] ~ /failure/ && c[s] ~ /success/) print s, c[s]}'
```
Any SHA with both failure and success without a code change = flake. Per-test flakiness: emit JUnit XML in CI, upload as artifact, ingest into a test-analytics store (e.g. report with `dorny/test-reporter`, or push test results to Postgres/Prometheus) and rank tests by flip rate. Quarantine policy: auto-open an issue for any test flipping >2 times in 7 days, mark quarantined (still runs, non-blocking) with an owner and a 14-day SLA.

## Step 4 - Continuous collection into Prometheus
Run a small exporter (e.g. `github-actions-exporter`) in the cluster, or use the `workflow_run` webhook:
```bash
cat > .github/workflows/metrics.yml <<'EOF'
name: metrics
on:
  workflow_run:
    workflows: ["ci"]
    types: [completed]
permissions: { actions: read, contents: read }
jobs:
  push:
    runs-on: ubuntu-latest
    steps:
      - run: |
          cat <<M | curl -sS --data-binary @- "$PUSHGATEWAY/metrics/job/ci/repo/${{ github.event.repository.name }}"
          ci_run_duration_seconds $(( $(date -d "${{ github.event.workflow_run.updated_at }}" +%s) - $(date -d "${{ github.event.workflow_run.run_started_at }}" +%s) ))
          ci_run_queue_seconds $(( $(date -d "${{ github.event.workflow_run.run_started_at }}" +%s) - $(date -d "${{ github.event.workflow_run.created_at }}" +%s) ))
          ci_run_failed ${{ github.event.workflow_run.conclusion == 'failure' && 1 || 0 }}
          M
        env: { PUSHGATEWAY: "${{ vars.PUSHGATEWAY_URL }}" }
EOF
```
(Pushgateway is acceptable here for event-driven batch-like metrics; mention the better production design: an exporter/OTel collector consuming webhooks into histograms.)

## Step 5 - Dashboard and alerts
PromQL examples:
```
histogram_quantile(0.95, sum by (le) (rate(ci_queue_seconds_bucket[1h])))
sum(rate(ci_runs_failed_total[1d])) / sum(rate(ci_runs_total[1d]))
```
Alerts: p95 queue > 60 s for 15 min (capacity), failure rate on `main` > 10 % (broken trunk), p95 duration +30 % week-over-week (regression).
Dashboard rows: platform-level, per-repo top offenders, flaky tests, cost (runner-minutes x rate; GitHub billing API `/orgs/$ORG/settings/billing/actions`).

## Step 6 - Link to DORA
Deployment frequency and lead time come from deploy events (ArgoCD notifications/Deployments API, Module 02). CI metrics diagnose *why* lead time is bad (queue vs build vs wait-for-approval).

## Break it on purpose
Add `sleep 120` in a test and confirm p95 duration moves in your calculation; add a test with `Math.random()<0.3` failure and find it with Step 3.

## Talking points
Four numbers: duration, queue time, failure rate, flake rate. Metrics decide whether to buy runners, add caches or fix tests. Present per-team showback, not shaming.
