# Lab 01-02 - Software Catalog: modeling, conventions, discovery, org sync, quality

## Goal
Model Acme in the catalog, auto-discover repos from GitHub, sync users/groups, enforce `catalog-info.yaml` standards in CI and at ingestion, and detect orphans/stale entities.

## Prereqs
Lab 01 app running locally (`yarn start`) with a GitHub token (`GITHUB_TOKEN`, fine-grained, read-only contents/metadata/org members).

## Step 1 - The entity model (draw it)
```
Domain (payments) -> System (checkout) -> Component (payments-api) -> API (payments-openapi)
                                      \-> Resource (payments-db)
Group (team-payments) owns Components; User belongs to Group
```
Relations: `spec.owner` -> `ownedBy`; `spec.system` -> `partOf`; `spec.providesApis` -> `providesApi`; `spec.consumesApis` -> `consumesApi`; `spec.dependsOn` -> `dependsOn`/`dependencyOf`. Kinds mean: **Component** = deployable/library with a lifecycle; **API** = contract; **Resource** = infra it uses; **System** = group of components shipped together; **Domain** = business area; **Group/User** = org.
Decisions to state: owner is always a **Group** (people leave), `lifecycle` in {experimental, production, deprecated}, `system` mandatory, `type` in a closed list (service, website, library, job).

## Step 2 - A complete `catalog-info.yaml` for payments-api
```bash
cd ../payments-api
cat > catalog-info.yaml <<'EOF'
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: payments-api
  description: Card payments authorization API
  tags: [java, tier-1]
  annotations:
    github.com/project-slug: acme-lab-<you>/payments-api
    backstage.io/techdocs-ref: dir:.
    backstage.io/kubernetes-id: payments
    backstage.io/kubernetes-namespace: payments-dev
    argocd/app-name: payments-dev
    pagerduty.com/service-id: PXXXXXX
    sonarqube.org/project-key: acme-lab_payments-api
    grafana/dashboard-selector: "tags @> 'payments'"
  links:
    - { url: https://grafana.example/d/payments, title: Dashboard }
spec:
  type: service
  lifecycle: production
  owner: group:default/team-payments
  system: checkout
  providesApis: [payments-openapi]
  dependsOn: [resource:default/payments-db]
---
apiVersion: backstage.io/v1alpha1
kind: API
metadata: { name: payments-openapi, description: Payments REST API }
spec:
  type: openapi
  lifecycle: production
  owner: group:default/team-payments
  system: checkout
  definition:
    $text: ./openapi.yaml
---
apiVersion: backstage.io/v1alpha1
kind: Resource
metadata: { name: payments-db, description: Postgres for payments }
spec:
  type: database
  owner: group:default/team-payments
  system: checkout
EOF
```
The annotations are the **integration contract** with every plugin: document them, validate them, and generate them from templates, never by hand.

## Step 3 - Validate in CI (don't wait for Backstage to reject it)
```bash
npx @roadiehq/backstage-entity-validator catalog-info.yaml
```
Org-specific rules on top (JSON-schema or a small script run in the reusable workflow):
```bash
yq -o=json catalog-info.yaml | jq -e '
  select(.kind=="Component") |
  (.spec.owner | test("^group:")) and
  (.spec.lifecycle | IN("experimental","production","deprecated")) and
  (.metadata.annotations["github.com/project-slug"] != null) and
  (.spec.system != null)' >/dev/null && echo OK || echo "catalog-info violates org rules"
```
Add as a platform workflow job (`validate-catalog`) required via ruleset (Module 06).

## Step 4 - Register a single location, then automate discovery
Manual (one-off) registration: UI "Register existing component" or `catalog.locations` in config. At scale use a **provider**:
```bash
yarn --cwd packages/backend add @backstage/plugin-catalog-backend-module-github @backstage/plugin-catalog-backend-module-github-org
```
```ts
// packages/backend/src/index.ts
backend.add(import('@backstage/plugin-catalog-backend-module-github'));
backend.add(import('@backstage/plugin-catalog-backend-module-github-org'));
```
```yaml
# app-config.yaml
integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}      # prefer a GitHub App (apps: appId, privateKey, webhookSecret) in production
catalog:
  rules:
    - allow: [Component, System, API, Resource, Location, Domain, Group, User]
  providers:
    github:
      acme:
        organization: acme-lab-<you>
        catalogPath: /catalog-info.yaml
        filters:
          branch: main
          repository: '.*'
          topic:
            exclude: [no-catalog]
        schedule:
          frequency: { minutes: 30 }
          timeout: { minutes: 3 }
    githubOrg:
      id: acme-org
      githubUrl: https://github.com
      orgs: [acme-lab-<you>]
      schedule:
        frequency: { hours: 1 }
        timeout: { minutes: 15 }
```
Restart and verify: `curl -s localhost:7007/api/catalog/entities?filter=kind=component | jq '.[].metadata.name'` (requires a token if auth is on; in dev use the UI). Expected: all repos with a `catalog-info.yaml` appear; GitHub org teams become `Group`s and members `User`s (so `usernameMatchingUserEntityName` works, Lab 01).
Scale details to mention: GitHub API rate limits (use a GitHub App; use **webhooks/events** via `@backstage/plugin-events-backend-module-github` so changes appear in seconds instead of waiting for the poll), `catalogPath` conventions, monorepo multi-entity files, `filters` and allow-rules to prevent random repos injecting `kind: Group`/`User` (locations found by providers are subject to `catalog.rules`).

## Step 5 - Entra ID (Azure AD) / LDAP org sync
Entra (Microsoft Graph provider):
```bash
yarn --cwd packages/backend add @backstage/plugin-catalog-backend-module-msgraph
```
```ts
backend.add(import('@backstage/plugin-catalog-backend-module-msgraph'));
```
```yaml
catalog:
  providers:
    microsoftGraphOrg:
      default:
        tenantId: ${AZURE_TENANT_ID}
        clientId: ${AZURE_CLIENT_ID}
        clientSecret: ${AZURE_CLIENT_SECRET}      # or use certificate / federated credential
        user:
          filter: accountEnabled eq true and userType eq 'member'
        group:
          filter: securityEnabled eq true and startswith(displayName,'team-')
        schedule: { frequency: { hours: 1 }, timeout: { minutes: 50 } }
```
App registration needs Graph `User.Read.All`, `Group.Read.All`, `GroupMember.Read.All` (application, admin-consented). Decisions: sync only groups that represent teams (naming prefix/security flag), transform to set `spec.profile`, `spec.memberOf`; disabled users filtered so offboarded people disappear; keep GitHub org sync **or** Entra sync as the user source of truth, not both (duplicate/mismatched entity names). LDAP: `@backstage/plugin-catalog-backend-module-ldap` with `ldapOrg` provider (`target: ldaps://...`, `users: { dn, options.filter, map }`, `groups: {...}`).
Local lab without a tenant: run OpenLDAP (`docker run -d -p 1389:1389 bitnami/openldap` or `osixia/openldap`) and the `ldapOrg` provider to see users/groups flow.

## Step 6 - Keeping ownership accurate (the hard part)
1. **Required at creation**: scaffolder templates require an owner picker (`OwnerPicker`) restricted to Groups.
2. **Validated on every PR**: Step 3 check; ruleset requires it.
3. **Resolved against reality**: a nightly job lists components whose owner Group no longer exists or has 0 members:
```bash
curl -s -H "Authorization: Bearer $BACKSTAGE_TOKEN" "$BACKSTAGE/api/catalog/entities?filter=kind=component" \
 | jq -r '.[] | select(.relations | map(select(.type=="ownedBy")) | length == 0) | .metadata.name'
```
   Backstage marks `ownedBy` relations to missing entities as unresolved; also use the catalog **orphan** state: entities whose source was deleted get `backstage.io/orphan: 'true'` annotation. Configure `catalog.orphanStrategy: delete` so they are removed automatically (default `keep`).
```yaml
catalog:
  orphanStrategy: delete
```
4. **Ownership review cadence**: each Group lead gets a monthly digest ("you own 14 components; 3 lack docs; 1 has no on-call"). Reorgs: team renames handled by a `Group` alias mapping and a bulk PR/bot updating `spec.owner`.
5. **Metrics**: catalog completeness %, % entities with valid owner, # orphaned, # stale (no commit > 12 months and lifecycle production).

## Step 7 - Custom processor to enforce quality at ingestion
Reject or flag bad entities in the backend (new backend system module):
```ts
// packages/backend/src/catalogQuality.ts
import { createBackendModule } from '@backstage/backend-plugin-api';
import { catalogProcessingExtensionPoint } from '@backstage/plugin-catalog-node/alpha';
import { CatalogProcessor } from '@backstage/plugin-catalog-node';
import { Entity } from '@backstage/catalog-model';
import { InputError } from '@backstage/errors';

class RequireOwnerProcessor implements CatalogProcessor {
  getProcessorName() { return 'RequireOwnerProcessor'; }
  async validateEntityKind(entity: Entity): Promise<boolean> {
    if (entity.kind === 'Component') {
      const owner = (entity.spec as any)?.owner;
      if (!owner || !String(owner).startsWith('group:')) {
        throw new InputError(`Component ${entity.metadata.name}: spec.owner must be a group reference like group:default/team-x`);
      }
    }
    return false; // false = don't claim kind validation, just add our check
  }
}

export default createBackendModule({
  pluginId: 'catalog',
  moduleId: 'acme-quality',
  register(env) {
    env.registerInit({
      deps: { catalog: catalogProcessingExtensionPoint },
      async init({ catalog }) { catalog.addProcessor(new RequireOwnerProcessor()); },
    });
  },
});
```
```ts
backend.add(import('./catalogQuality'));
```
Verify: push a component with `owner: alice` (a user, or a bare name); the catalog shows it under "Failed to process" with the error. Trade-off: hard-failing ingestion removes visibility of the entity entirely; for soft enforcement, add a `backstage.io/quality: failing` annotation or relation and surface it in a scorecard (Lab 05). Staff answer: *hard-fail for new entities (via CI), soft-warn + scorecard for legacy*, so you don't hide 200 services during rollout.

## Step 8 - Inspect and debug
```bash
curl -s "localhost:7007/api/catalog/entities/by-name/component/default/payments-api" | jq '.relations'
curl -s "localhost:7007/api/catalog/entities?filter=kind=component&fields=metadata.name,spec.owner" | jq
```
UI: entity -> "Dependencies"/"Ownership" graph; Catalog Import page; `/catalog-graph`. Debug ingestion with logs (`catalog` plugin logs include processing errors); refresh an entity via UI or `POST /api/catalog/refresh`.

## Break it on purpose
1. Delete a repo's `catalog-info.yaml` and wait for the provider cycle (or trigger): the entity becomes orphan then is removed (with `orphanStrategy: delete`).
2. Create two components with the same `metadata.name` in different repos: the second fails with a conflict; discuss naming conventions/`namespace` use.
3. Typo in `spec.system` (system doesn't exist): relation unresolved (`partOf` missing) but entity ingests; detect via nightly check.

## Cleanup
Keep configuration; later labs reuse the catalog.

## Talking points
Entity model rationale, Group-as-owner, provider vs static locations, events for freshness, org sync source of truth, orphan strategy, hard vs soft enforcement, ownership as an ongoing process with metrics.
