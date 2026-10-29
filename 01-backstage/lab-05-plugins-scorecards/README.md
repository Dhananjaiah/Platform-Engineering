# Lab 01-05 - Plugins, custom plugins, and scorecards (Tech Insights)

## Goal
Install community plugins (Kubernetes, ArgoCD, GitHub Actions, PagerDuty, SonarQube, cost), build a custom frontend plugin and a custom backend plugin on the new backend system, and measure service maturity with Tech Insights scorecards.

## Prereqs
Labs 01-03, `acme-backstage` running. Plugin APIs change: for each install, read the plugin's README for current `app-config` keys and frontend wiring, and run it against the Backstage version in `backstage.json`.

## Part A - Community plugins (the install pattern)
Every plugin = (1) frontend package + (2) backend package/module + (3) config + (4) entity annotations + (5) entity page wiring. Pattern with Kubernetes:
```bash
yarn --cwd packages/app add @backstage/plugin-kubernetes
yarn --cwd packages/backend add @backstage/plugin-kubernetes-backend
```
```ts
// packages/backend/src/index.ts
backend.add(import('@backstage/plugin-kubernetes-backend'));
```
```yaml
kubernetes:
  serviceLocatorMethod: { type: multiTenant }
  clusterLocatorMethods:
    - type: config
      clusters:
        - name: dev
          url: https://dev-control-plane:6443
          authProvider: serviceAccount
          skipTLSVerify: true                # lab only
          serviceAccountToken: ${K8S_DEV_TOKEN}
```
Create a **read-only** ServiceAccount for Backstage per cluster (never cluster-admin):
```bash
kubectl --context kind-dev create sa backstage-reader -n backstage-system --dry-run=client -o yaml | kubectl --context kind-dev apply -f - 2>/dev/null || (kubectl --context kind-dev create ns backstage-system && kubectl --context kind-dev -n backstage-system create sa backstage-reader)
kubectl --context kind-dev create clusterrolebinding backstage-reader --clusterrole=view --serviceaccount=backstage-system:backstage-reader
export K8S_DEV_TOKEN=$(kubectl --context kind-dev -n backstage-system create token backstage-reader --duration=24h)
```
Production: AKS uses `authProvider: azure`/`aks` with the user's Entra token so developers see only what their RBAC allows, or a service account with a custom ClusterRole (pods, deployments, events, HPAs, ingresses; no secrets). Entity wiring: annotations `backstage.io/kubernetes-id` and `backstage.io/kubernetes-namespace` (Lab 02) and the `EntityKubernetesContent` tab on the entity page.

Other plugins (same pattern; check each README):
| Plugin | Annotation(s) | Auth / notes |
|---|---|---|
| ArgoCD (Roadie `@roadiehq/backstage-plugin-argo-cd`) | `argocd/app-name` | ArgoCD API token with read-only role (`role:readonly` limited by project) via proxy/backend, never exposing the token to the browser |
| GitHub Actions (`@backstage-community/plugin-github-actions`) | `github.com/project-slug` | GitHub integration token / App |
| PagerDuty | `pagerduty.com/service-id` (or integration key) | API token in backend; shows on-call, incidents |
| SonarQube (`@backstage-community/plugin-sonarqube`) | `sonarqube.org/project-key` | token in backend config |
| Cost | `opencost` via custom backend proxy or the Cost Insights plugin | OpenCost API behind `proxy:` endpoint (Lab 04-09) |
Proxy pattern (keep secrets server-side):
```yaml
proxy:
  endpoints:
    '/opencost':
      target: http://opencost.opencost.svc:9090
      changeOrigin: true
```
Install discipline: pilot with one team; read the code for what data it sends and where tokens live; pin versions; own an upgrade plan; remove unused plugins. Every plugin you add is something you must upgrade monthly.

## Part B - A custom frontend plugin (React)
Scaffold with the CLI (`yarn new` -> `frontend-plugin`, id `team-health`):
```bash
yarn new            # choose: frontend-plugin, id: team-health
ls plugins/team-health/src
```
Goal: an entity tab "Service Health" showing last deploy time and alerts for the entity, using Backstage APIs. Component sketch:
```tsx
// plugins/team-health/src/components/HealthCard/HealthCard.tsx
import React from 'react';
import { useEntity } from '@backstage/plugin-catalog-react';
import { InfoCard, Progress, ResponseErrorPanel } from '@backstage/core-components';
import { useApi, configApiRef, fetchApiRef } from '@backstage/core-plugin-api';
import useAsync from 'react-use/lib/useAsync';

export const HealthCard = () => {
  const { entity } = useEntity();
  const backendUrl = useApi(configApiRef).getString('backend.baseUrl');
  const { fetch } = useApi(fetchApiRef);          // fetchApi injects the Backstage auth token
  const name = entity.metadata.name;

  const { value, loading, error } = useAsync(async () => {
    const res = await fetch(`${backendUrl}/api/team-health/services/${name}`);
    if (!res.ok) throw new Error(`team-health backend: ${res.status}`);
    return (await res.json()) as { lastDeploy: string; firingAlerts: number; sloBurn: number };
  }, [name]);

  if (loading) return <Progress />;
  if (error) return <ResponseErrorPanel error={error} />;
  return (
    <InfoCard title="Service health">
      <div>Last deploy: {value?.lastDeploy}</div>
      <div>Firing alerts: {value?.firingAlerts}</div>
      <div>SLO burn (1h): {value?.sloBurn}</div>
    </InfoCard>
  );
};
```
Use `fetchApiRef` not raw `fetch` (so auth tokens are attached); handle loading/error; keep presentation separate from data fetching; use the design system (`@backstage/core-components`, MUI themes) so it matches the portal; `isTeamHealthAvailable(entity)` to render only for entities with the right annotation; add tests with `renderInTestApp` and `msw`.
Wiring (check your Backstage version's frontend system): legacy `EntityLayout.Route path="/health" title="Health"` with `<EntitySwitch>` guard in `packages/app/src/components/catalog/EntityPage.tsx`; in the new frontend system the plugin exports an extension (`EntityContentBlueprint`) and is enabled by adding the package.

## Part C - A custom backend plugin (new backend system)
```bash
yarn new            # choose: backend-plugin, id: team-health
```
```ts
// plugins/team-health-backend/src/plugin.ts
import { coreServices, createBackendPlugin } from '@backstage/backend-plugin-api';
import { createRouter } from './router';

export const teamHealthPlugin = createBackendPlugin({
  pluginId: 'team-health',
  register(env) {
    env.registerInit({
      deps: {
        httpRouter: coreServices.httpRouter,
        logger: coreServices.logger,
        config: coreServices.rootConfig,
        httpAuth: coreServices.httpAuth,
        cache: coreServices.cache,
      },
      async init({ httpRouter, logger, config, httpAuth, cache }) {
        httpRouter.use(await createRouter({ logger, config, httpAuth, cache }));
        // Everything under /api/team-health requires a valid Backstage user token by default.
      },
    });
  },
});
```
```ts
// plugins/team-health-backend/src/router.ts
import express from 'express';
import Router from 'express-promise-router';
import { LoggerService, HttpAuthService, CacheService, RootConfigService } from '@backstage/backend-plugin-api';

export async function createRouter(o: { logger: LoggerService; config: RootConfigService; httpAuth: HttpAuthService; cache: CacheService }) {
  const router = Router();
  router.use(express.json());
  const prom = o.config.getString('teamHealth.prometheusUrl');

  router.get('/services/:name', async (req, res) => {
    await o.httpAuth.credentials(req, { allow: ['user', 'service'] });   // identity of the caller
    const name = req.params.name;
    const cached = await o.cache.get<string>(`health:${name}`);
    if (cached) return res.json(JSON.parse(cached));

    const q = encodeURIComponent(`sum(ALERTS{alertstate="firing",service="${name.replace(/[^a-z0-9-]/g, '')}"})`);
    const r = await fetch(`${prom}/api/v1/query?query=${q}`);
    const j: any = await r.json();
    const firingAlerts = Number(j?.data?.result?.[0]?.value?.[1] ?? 0);
    const body = { lastDeploy: 'n/a', firingAlerts, sloBurn: 0 };
    await o.cache.set(`health:${name}`, JSON.stringify(body), { ttl: 30_000 });
    res.json(body);
  });
  router.get('/health', (_req, res) => res.json({ status: 'ok' }));
  return router;
}
```
```ts
backend.add(import('@internal/backstage-plugin-team-health-backend'));
```
```yaml
teamHealth:
  prometheusUrl: http://kps-kube-prometheus-stack-prometheus.monitoring:9090
```
Principles: sanitise inputs before building queries (see the regex), use `httpAuth` to require identity, cache expensive calls (`CacheService`), use `scheduler` service for periodic work, `database` service for state, `urlReader` for fetching from SCM with integrations, and never put secrets in the frontend. Test with `supertest` against `createRouter` and a mocked `fetch`; backend test utils (`mockServices`, `startTestBackend`) simulate the backend.
Verify:
```bash
curl -s localhost:7007/api/team-health/health
```
(Needs a token unless `health` is allowed unauthenticated via `httpRouter.addAuthPolicy({ path: '/health', allow: 'unauthenticated' })`.)

## Part D - Scorecards with Tech Insights
Goal: measure service maturity: has owner, has docs, has on-call, has recent base image, has SLO, runs on supported template version.
```bash
yarn --cwd packages/app add @backstage-community/plugin-tech-insights
yarn --cwd packages/backend add @backstage-community/plugin-tech-insights-backend @backstage-community/plugin-tech-insights-backend-module-jsonfc @backstage-community/plugin-tech-insights-node
```
(Package scope moved to `@backstage-community`; verify current names.) Fact retriever (custom) producing facts per entity, e.g. "hasDocs" from annotations, "hasOnCall" from `pagerduty.com/service-id`, "baseImageAge" from a GitHub/registry query:
```ts
// facts
const entityMetadataFactRetriever: FactRetriever = {
  id: 'acmeEntityMetadata',
  version: '1.0.0',
  entityFilter: [{ kind: 'component' }],
  schema: {
    hasOwnerGroup: { type: 'boolean', description: 'Owner is a Group' },
    hasDocs: { type: 'boolean', description: 'techdocs-ref set' },
    hasOnCall: { type: 'boolean', description: 'pagerduty service id set' },
    onLatestTemplate: { type: 'boolean', description: 'template-version is current' },
  },
  handler: async ({ discovery, auth }) => {
    // fetch components from the catalog and compute booleans, return [{ entity, facts }]
    return [];
  },
};
```
Checks (JSON Rules Engine) and a scorecard definition:
```ts
const checks = [
  { id: 'hasDocs', type: 'json-rules-engine', name: 'Has TechDocs', description: 'Docs exist', factIds: ['acmeEntityMetadata'],
    rule: { conditions: { all: [{ fact: 'hasDocs', operator: 'equal', value: true }] } } },
  { id: 'hasOnCall', type: 'json-rules-engine', name: 'Has on-call', description: 'PagerDuty service linked', factIds: ['acmeEntityMetadata'],
    rule: { conditions: { all: [{ fact: 'hasOnCall', operator: 'equal', value: true }] } } },
];
```
Run the retrievers on a schedule (`cron: '*/30 * * * *'`), store results in Postgres, render `EntityTechInsightsScorecardContent` per entity and an org-level leaderboard (custom page querying `/api/tech-insights`).
Scorecard design (what makes them work):
1. **Few, meaningful, automatable checks** (6-10). Every check has a "how to fix" link and an owner.
2. **Tiers**: bronze (owner, docs, on-call) -> silver (SLO, scans, supported base image) -> gold (DR tested, load tested). Tier maps to service tier (tier-1 services need gold).
3. **No shaming**: team dashboards; org rollups by domain; trend over time.
4. **Hook into delivery**: low score blocks nothing at first; later required for production promotion of tier-1.
5. **Evidence from systems of record**, not self-attested annotations only (e.g. on-call exists in PagerDuty, SLO exists in Prometheus rules, base image from registry metadata).
6. Review checks quarterly; retire ones everyone passes.

## Break it on purpose
1. Install a plugin that expects an annotation you don't have: it shows "missing annotation" empty state; adopt a `isAnnotationAvailable` guard so tabs hide instead.
2. Misconfigure the proxy to expose a token in a response header; discover why secrets must never be forwarded to the browser; add a test.
3. Make a fact retriever throw: Tech Insights logs errors per entity; confirm other entities continue; add alerting on retriever failures.

## Cleanup
Keep the plugins; they feed the capstone.

## Talking points
Plugin install pattern and cost-of-ownership, read-only per-cluster credentials, server-side proxies for secrets, custom plugin design (fetchApi, auth, caching, sanitising), scorecards as a behaviour-change tool with evidence-based checks and tiers.
