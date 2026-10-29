# Lab 01-06 - Operating Backstage: upgrades, scale, observability, product metrics

## Goal
Run Backstage as a production service: upgrade it safely every month, make it observable and fast, and run the portal like a product with metrics and a roadmap.

## Prereqs
Labs 01-05, `acme-backstage` repo and deployment.

## Step 1 - The upgrade routine (monthly)
Backstage ships monthly releases; skipping makes upgrades painful (the "upgrades are painful" problem is mostly deferred upgrades).
```bash
cd acme-backstage
git checkout -b chore/backstage-upgrade
cat backstage.json                              # current
yarn backstage-cli versions:bump                # bumps all @backstage/* to the latest release
git diff --stat
yarn install
yarn tsc                                        # type errors reveal API changes
yarn lint:all && yarn test:all
yarn build:backend
```
Then: read the release notes between your version and the target (`https://github.com/backstage/backstage/tree/master/docs/releases` and the upgrade helper at `backstage.github.io/upgrade-helper`), fix breaking changes, check **each community plugin** for a compatible release (`yarn why @backstage/plugin-catalog-react` to find duplicate versions; fix with `yarn dedupe`). Rules:
- Pin plugin versions exactly in `package.json`; let Renovate propose bumps weekly grouped as "backstage-core" and "backstage-community".
- One upgrade PR, small diffs, CI must pass: unit tests, type check, a Playwright smoke test (login, open catalog, open an entity, run template dry-run).
- Run `yarn backstage-cli repo fix` / `versions:check` if offered by your version to detect duplicates.

## Step 2 - Smoke tests as the safety net
```bash
cd packages/app && yarn add -D @playwright/test
cat > e2e-tests/smoke.spec.ts <<'EOF'
import { test, expect } from '@playwright/test';

test('catalog loads and an entity page renders', async ({ page }) => {
  await page.goto('/catalog');
  await expect(page.getByText('My Company Catalog').or(page.getByText('Catalog'))).toBeVisible();
  await page.getByRole('link', { name: 'payments-api' }).first().click();
  await expect(page.getByText('payments-api')).toBeVisible();
});

test('scaffolder lists the golden path', async ({ page }) => {
  await page.goto('/create');
  await expect(page.getByText('Java Spring service on AKS')).toBeVisible();
});
EOF
npx playwright test
```
Authentication in CI: use `auth.providers.guest` only in the CI environment (never in prod) or a test token.

## Step 3 - Canary and rollout
Treat Backstage like any service: a `canary` Deployment (or Argo Rollout, Module 03 Lab 06) with 10 % traffic via the ingress, analysis on 5xx rate and `catalog` processing errors; automatic rollback on failure. Run DB migrations carefully: Backstage plugins migrate on startup (Knex); with multiple replicas the first one migrates while others wait on a lock. For risky upgrades:
1. Snapshot the DB (CNPG `Backup` or Azure PITR).
2. Deploy to a **staging Backstage** pointing at a restored copy of prod DB.
3. Run smoke tests + compare entity counts (`curl /api/catalog/entities/by-query?limit=0 | jq .totalItems`).
4. Roll to prod during low traffic; rollback = previous image digest; DB forward-only migrations mean the rollback needs the snapshot restore if a migration ran: know this before starting.
```bash
kubectl -n backstage apply -f - <<'EOF'
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata: { name: pre-upgrade-2026-11 }
spec: { cluster: { name: backstage-pg } }
EOF
kubectl -n backstage get backup pre-upgrade-2026-11 -w
```
(Backups need a configured object store in the CNPG `Cluster` (`spec.backup.barmanObjectStore`) or a volume-snapshot class; add one, plus a `ScheduledBackup`, before relying on this. On Azure Postgres Flexible Server, PITR is built in.)

## Step 4 - Performance and scale
Typical bottlenecks and fixes:
| Symptom | Cause | Fix |
|---|---|---|
| Catalog list page slow with 5k+ entities | large unfiltered queries, entity page relations | server-side filtering/pagination (`/entities/by-query`), tune DB indexes, increase Postgres resources |
| High CPU during refresh | processing loop over all entities every 100-150 s default | increase `catalog.processingInterval`, split providers, remove noisy locations |
| GitHub rate limiting | many providers polling | GitHub App auth (higher limits), events (webhooks), longer schedules |
| Slow search | collators indexing everything in-memory (Lunr) | switch to Elasticsearch/OpenSearch or Postgres search engine (`@backstage/plugin-search-backend-module-pg`) |
| Cold-start spikes | single replica | 2+ replicas, readiness probes, HPA on CPU |
| Many concurrent scaffolder tasks | long-running actions in-process | scale replicas; scaffolder tasks run distributed with DB-based task queue; keep actions short; offload heavy work to workflows |
```yaml
catalog:
  processingInterval: { minutes: 5 }      # trade freshness for load
search:
  pg: { highlightOptions: { useHighlight: true } }
backend:
  cache: { store: redis, connection: redis://backstage-redis:6379 }
```
Backend horizontal scaling: stateless; DB is the bottleneck; use PgBouncer/connection limits (each plugin has its own DB and pool: `max_connections` budget = replicas x plugins x pool size; configure `backend.database.knexConfig.pool.max`).
Load test (hand-typed k6):
```bash
cat > load.js <<'EOF'
import http from 'k6/http'; import { check } from 'k6';
export const options = { vus: 50, duration: '2m', thresholds: { http_req_duration: ['p(95)<800'], http_req_failed: ['rate<0.01'] } };
export default function () {
  const r = http.get(`${__ENV.BASE}/api/catalog/entities/by-query?limit=50&filter=kind=component`, { headers: { Authorization: `Bearer ${__ENV.TOKEN}` } });
  check(r, { 'status 200': (x) => x.status === 200 });
}
EOF
k6 run -e BASE=http://backstage.127.0.0.1.nip.io:8080 -e TOKEN=$STATIC_TOKEN load.js
```
Set SLOs: availability 99.9 % for `/` and `/api/catalog`, p95 latency < 800 ms, catalog freshness (time from merge to appearing) < 5 min.

## Step 5 - Observability for Backstage itself
```yaml
backend:
  # structured JSON logs to stdout; scrape with Loki/Alloy
```
- Metrics: enable OTel (`@backstage/backend-defaults` supports OpenTelemetry via `instrumentation.js`) -> Prometheus/Tempo; watch HTTP latency, error rate, DB connections, catalog processing queue (`catalog_processing_*`), scaffolder task failures.
- Alerts: readiness failures, 5xx ratio, `catalog` processing errors, provider sync failures, scaffolder failed tasks > threshold, DB connection saturation.
- Audit: the **auditor service** (`coreServices.auditor`) records security-relevant events; ship to your SIEM.
- Synthetic: the nightly scaffolder canary (Lab 03) measures end-to-end health.

## Step 6 - The product layer
**Metrics (instrument them):**
| Metric | How |
|---|---|
| Weekly active users / engineers | Backstage analytics API (`analytics.*` config with a custom `AnalyticsApi` sending events to PostHog/Amplitude/Matomo, or OTel events) |
| Template usage, success rate, time-to-first-deploy | scaffolder task DB + deploy event timestamps |
| Catalog completeness | % with valid owner/docs/on-call (Tech Insights) |
| Search success | search events vs click-through |
| Support load | tickets/questions per week about "where is X" (should fall) |
| NPS/CSAT | quarterly 3-question survey linked in the portal |
Custom analytics:
```ts
// packages/app/src/apis.ts: add a custom AnalyticsApi implementing captureEvent(event) that POSTs to your collector.
```
**Roadmap process:** quarterly: (1) user interviews + survey, (2) data review, (3) top 5 pains, (4) themes with owners (e.g. "time-to-first-deploy under 30 min", "ownership accuracy 95 %"), (5) publish and review in office hours. Run a **platform advisory group** of 6-8 developers, release notes monthly, demo in all-hands, and deprecate plugins nobody uses.
**Adoption playbook:** start with one painful workflow (new service creation), make Backstage the *only* path for it, migrate catalog data with a bot PR adding `catalog-info.yaml` to all repos, announce office hours, embed champions per org, track adoption weekly, use scorecards (not mandates) to pull teams in.

## Step 7 - Security hardening checklist
- SSO only (no guest in prod); sign-in resolver tied to catalog users; token TTLs.
- Permission framework enabled with a tested policy; scaffolder templates restricted by owner groups.
- Integrations via GitHub App with minimal permissions; secrets via ESO; no tokens in `app-config.yaml`.
- Backend external access tokens scoped (`accessRestrictions`) and rotated.
- Restrict `catalog.rules`, `allowedHosts` in templates, and `reading.allow` (URL reader allowlist) to block SSRF via `$text`/`$url` includes.
- Content Security Policy headers; dependency scanning (the app has ~2k transitive deps): Trivy/Dependabot, image rebuilt weekly.
- Network policy: backend can reach only Postgres, GitHub, IdP, internal APIs it needs.

## Break it on purpose
1. Run `versions:bump` after skipping 6 months of releases in a branch: count the breaking changes: this is the argument for monthly cadence.
2. Exhaust DB connections: set `pool.max: 50` with 5 replicas and 15 plugin DBs; Postgres rejects connections (`too many clients`); compute the budget and fix.
3. Kill the Postgres primary (`kubectl -n backstage delete pod backstage-pg-1`) and observe CNPG failover and Backstage reconnect; measure the blip.

## Cleanup
Nothing to delete.

## Talking points
Monthly upgrade routine + smoke tests + staging with prod DB copy + rollback plan, scaling limits and the DB connection budget, SLOs for the portal, analytics-driven roadmap, adoption mechanics, hardening checklist.
