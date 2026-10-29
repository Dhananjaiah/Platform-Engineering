# Lab 02-04 - Measuring the platform: DORA, SPACE, adoption

## Goal
Build a small metrics pipeline you can actually run: collect deployment and change events, compute the four DORA metrics per service/team in SQL, show them in Grafana, add developer-experience measures (time-to-first-deploy, adoption, survey), and learn the pitfalls of metrics.

## Prereqs
Postgres (CNPG from Module 01 or `docker run postgres:16`), Grafana (Module 04 Lab 06), GitHub + ArgoCD from earlier modules. `psql`, `jq`.

## Definitions you must be able to state
| Metric | Definition (this platform) | Data source |
|---|---|---|
| Deployment frequency | production deploys per service per day/week | ArgoCD sync success events for `prd` apps (or GitHub Deployments API) |
| Lead time for changes | first commit of a change -> running in prod | commit timestamp (GitHub) -> prod sync timestamp (ArgoCD), joined by commit SHA/image digest |
| Change failure rate | % deploys causing degradation/rollback/hotfix | deploys followed by rollback/revert/incident within N hours |
| Time to restore (MTTR) | incident open -> resolved | PagerDuty/incident tool |
Benchmarks (DORA research tiers): elite = on-demand deploys, lead time < 1 day, CFR < 15 %, restore < 1 hour. Use them as directions, not targets to game.

## Step 1 - Schema
```bash
kubectl -n backstage port-forward svc/backstage-pg-rw 5433:5432 &
PGPASSWORD=$(kubectl -n backstage get secret backstage-pg-app -o jsonpath='{.data.password}' | base64 -d) \
psql -h localhost -p 5433 -U app -d app <<'EOF'
CREATE SCHEMA IF NOT EXISTS dora;
CREATE TABLE IF NOT EXISTS dora.commits (
  sha text PRIMARY KEY, repo text NOT NULL, service text NOT NULL, authored_at timestamptz NOT NULL, merged_at timestamptz
);
CREATE TABLE IF NOT EXISTS dora.deployments (
  id bigserial PRIMARY KEY, service text NOT NULL, env text NOT NULL, sha text NOT NULL,
  deployed_at timestamptz NOT NULL, status text NOT NULL,   -- succeeded|failed|rolled_back
  UNIQUE (service, env, sha, deployed_at)
);
CREATE TABLE IF NOT EXISTS dora.incidents (
  id text PRIMARY KEY, service text NOT NULL, opened_at timestamptz NOT NULL, resolved_at timestamptz, severity text
);
CREATE TABLE IF NOT EXISTS dora.services (service text PRIMARY KEY, team text NOT NULL, tier text);
EOF
```
Connection: use a dedicated DB/user in real setups with least privilege.

## Step 2 - Collect deployments from ArgoCD notifications
In Module 03 Lab 08 you created a webhook notification. Add a trigger/template that fires on successful sync with the Git revision and app name; send to a small collector (any HTTP receiver that inserts into Postgres: 30 lines of Python/Node, or a Grafana Alloy/Vector pipeline). Template body:
```json
{ "service":"{{.app.metadata.labels.service}}", "env":"{{.app.metadata.labels.env}}",
  "sha":"{{.app.status.operationState.syncResult.revision}}", "status":"succeeded",
  "deployed_at":"{{.app.status.operationState.finishedAt}}" }
```
Trigger: `when: app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'`. Note: ArgoCD sees the **config repo** commit; to link to the **app** commit, CI writes the app SHA into the config commit message or an annotation on the Deployment (`acme.io/source-sha`), and the collector reads it from the app's `status.summary.images` / annotation. This SHA propagation is the make-or-break design decision for lead time: do it in the `gitops-bump` workflow (Module 06 Lab 02).
Manual seeding for the lab (so you can query without a full pipeline):
```bash
PGPASSWORD=... psql -h localhost -p 5433 -U app -d app <<'EOF'
INSERT INTO dora.services VALUES ('payments','team-payments','tier-1'),('orders','team-orders','tier-2') ON CONFLICT DO NOTHING;
INSERT INTO dora.commits SELECT md5(g::text), 'payments-api','payments', now() - (g || ' hours')::interval - interval '5 hours', now() - (g || ' hours')::interval - interval '4 hours' FROM generate_series(1,40) g;
INSERT INTO dora.deployments (service, env, sha, deployed_at, status)
 SELECT 'payments','prd', md5(g::text), now() - (g || ' hours')::interval + interval '3 hours', CASE WHEN g % 9 = 0 THEN 'rolled_back' ELSE 'succeeded' END FROM generate_series(1,40) g;
INSERT INTO dora.incidents VALUES ('INC-1','payments', now() - interval '30 hours', now() - interval '29 hours','sev2'),('INC-2','payments', now() - interval '10 hours', now() - interval '9 hours 20 minutes','sev3');
EOF
```
Collect commits from GitHub (first commit of the PR -> merged_at) with `gh api` (hand-typed, then automated by a nightly workflow):
```bash
gh api repos/$ORG/payments-api/pulls?state=closed\&per_page=50 --jq '.[] | select(.merged_at!=null) | [.merge_commit_sha, .created_at, .merged_at] | @tsv'
```
Choose: lead time start = first commit in the PR (or PR open as a simpler proxy) and end = prod deploy. State the definition you used; teams must agree on it.

## Step 3 - The four metrics in SQL
```sql
-- Deployment frequency (prod, per service, per week)
SELECT service, date_trunc('week', deployed_at) wk, count(*) deploys
FROM dora.deployments WHERE env='prd' AND status<>'failed' GROUP BY 1,2 ORDER BY 2 DESC, 1;

-- Lead time for changes (median & p90, hours)
SELECT d.service,
  percentile_cont(0.5) WITHIN GROUP (ORDER BY extract(epoch FROM d.deployed_at - c.authored_at)/3600) AS p50_h,
  percentile_cont(0.9) WITHIN GROUP (ORDER BY extract(epoch FROM d.deployed_at - c.authored_at)/3600) AS p90_h
FROM dora.deployments d JOIN dora.commits c ON c.sha = d.sha
WHERE d.env='prd' AND d.status='succeeded' AND d.deployed_at > now() - interval '30 days' GROUP BY 1;

-- Change failure rate (30d)
SELECT service, round(100.0 * count(*) FILTER (WHERE status='rolled_back') / count(*), 1) AS cfr_pct
FROM dora.deployments WHERE env='prd' AND deployed_at > now() - interval '30 days' GROUP BY 1;

-- MTTR (hours)
SELECT service, round(avg(extract(epoch FROM resolved_at - opened_at)/3600)::numeric, 2) AS mttr_h
FROM dora.incidents WHERE resolved_at IS NOT NULL AND opened_at > now() - interval '90 days' GROUP BY 1;
```
Use medians/percentiles, not averages (skew). Compute per team via `dora.services`. Detect rollbacks from git reverts/`rollback` events, or Argo history showing a return to an older digest.

## Step 4 - Grafana dashboards
Add the Postgres datasource (`Connections -> Data sources -> PostgreSQL`, host `backstage-pg-rw.backstage:5432`, TLS mode per your cluster, read-only user). Panels: deployment frequency (bar per week), lead time p50/p90 (time series), CFR % (stat with thresholds), MTTR, each with a `team` variable. Provision as code (ConfigMap with `grafana_dashboard=1` label, Module 04 Lab 06) so it lives in git. Link from the Backstage team page.

## Step 5 - Developer experience measures (beyond DORA)
SPACE dimensions: **S**atisfaction/well-being, **P**erformance, **A**ctivity, **C**ommunication/collaboration, **E**fficiency/flow. Never use one metric; combine perception and telemetry.
| Measure | How | Target (example) |
|---|---|---|
| Time-to-first-deploy (new service) | scaffolder task start -> first prod/dev deploy | p50 < 60 min |
| Onboarding time (new hire to first merged PR) | HR/GitHub | < 3 days |
| Golden-path adoption | % of new services via templates; % of all services conformant | > 70 % in 2 quarters |
| CI feedback time | p50 PR pipeline duration (Module 06 Lab 05) | < 10 min |
| PR cycle time / wait time | GitHub API | trend down |
| Platform NPS/CSAT | quarterly survey | NPS > 30 |
| Support load | questions in platform channel/week | trend down |
Survey template (5 questions, 3 minutes): (1) How easy is it to ship a change to production? (1-5) (2) How often do you wait on another team to unblock yourself? (3) What is the most frustrating part of our platform? (free text) (4) Would you recommend the platform to a new team? (0-10) (5) How much time/week do you lose to tooling friction? Run quarterly with a >40 % response target; publish results and what you will do about them.

## Step 6 - Pitfalls (be ready to be challenged)
- **Goodhart's law**: when a metric becomes a target it stops being a good metric. Use for improvement conversations, never for individual performance reviews or team rankings.
- **Definition drift**: agree on definitions once; version them.
- **Mixing prod and non-prod**; counting config-only deploys vs code deploys; monorepo attribution.
- **Survivorship**: low deployment frequency may be right for regulated/embedded products; compare a team to its own trend.
- **Data quality**: missing SHA linkage makes lead time useless, instrument first.
- **Cost**: build only the metrics you will act on; start with deployment frequency + lead time (cheap), add CFR/MTTR when incident data is clean.

## Step 7 - Use data to drive the roadmap (the Staff move)
Example finding: "p90 lead time is 4 days; 70 % is waiting in PR review and manual prod approval." Actions: reviewer SLAs/auto-assign, automate prod approvals for low-risk changes with canary + SLO gates, CODEOWNERS tuning. Re-measure after one quarter and report the delta to leadership: "lead time p50 fell from 2.1 days to 0.6; deployment frequency doubled; CFR unchanged at 8 %".

## Break it on purpose
1. Drop the SHA linkage (insert deployments with unknown SHAs): the lead-time query returns nothing: demonstrates why the instrumentation matters.
2. Average vs median: insert one 30-day outlier and compare.
3. Create a team leaderboard and discuss the behaviour it invites (gaming, splitting deploys): replace with team-vs-own-trend.

## Cleanup
`kill %1` (port-forward). Keep the schema for the capstone.

## Talking points
Precise definitions and SHA linkage, SQL for the four metrics, percentiles, SPACE for balance, DX measures (time-to-first-deploy, adoption, surveys), Goodhart's law, using findings to set and prove roadmap outcomes.
