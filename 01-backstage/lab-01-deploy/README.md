# Lab 01-01 - Build, secure and deploy Backstage on Kubernetes with PostgreSQL

## Goal
Create the Backstage app, run it locally against Postgres, add GitHub SSO and a permission policy, build a production image, and deploy to the `hub` cluster with CloudNativePG (Postgres), config from Secrets, and ingress.

## Prereqs
Node 20 or 22 (`node -v`), `yarn` (Backstage uses Yarn 4: `corepack enable`), Docker, `kubectl`, `helm`, kind `hub` cluster, GitHub org (Module 00).

## Step 1 - Create the app
```bash
corepack enable
npx @backstage/create-app@latest --path acme-backstage
```
Answer the app name `acme-backstage`. Then:
```bash
cd acme-backstage
cat backstage.json                     # records the Backstage release you are on
ls packages plugins
yarn start                             # frontend :3000, backend :7007 (SQLite in-memory)
```
Verify: open `http://localhost:3000`, see the catalog with example entities. Stop it (Ctrl-C). Commit to a new repo `acme-backstage`:
```bash
git init -b main && git add -A && git commit -m "chore: scaffold backstage" 
gh repo create acme-lab-<you>/acme-backstage --private --source=. --push
```
Why record the version: upgrades (Lab 06) use it, and plugin compatibility is tied to it.

## Step 2 - Postgres locally
```bash
docker run -d --name bs-pg -e POSTGRES_PASSWORD=bs-lab-pass -e POSTGRES_USER=backstage -p 5432:5432 postgres:16
yarn --cwd packages/backend add pg
```
Edit `app-config.local.yaml` (git-ignored, for developer-local overrides):
```bash
cat > app-config.local.yaml <<'EOF'
backend:
  database:
    client: pg
    connection:
      host: localhost
      port: 5432
      user: backstage
      password: bs-lab-pass
EOF
yarn start
```
Verify: `docker exec bs-pg psql -U backstage -c '\l'` shows databases per plugin (`backstage_plugin_catalog`, `..._auth`, `..._scaffolder`...). Interview point: Backstage creates **one database per plugin** (needs `CREATEDB` privilege), so each plugin's migrations are isolated, and you can scale/restore selectively.

## Step 3 - GitHub SSO and sign-in resolver
Create a GitHub OAuth App (org settings -> Developer settings): homepage `http://localhost:3000`, callback `http://localhost:7007/api/auth/github/handler/frame`.
```bash
yarn --cwd packages/backend add @backstage/plugin-auth-backend-module-github-provider
```
In `packages/backend/src/index.ts` add:
```ts
backend.add(import('@backstage/plugin-auth-backend-module-github-provider'));
```
Config (`app-config.yaml`):
```yaml
auth:
  environment: development
  providers:
    github:
      development:
        clientId: ${AUTH_GITHUB_CLIENT_ID}
        clientSecret: ${AUTH_GITHUB_CLIENT_SECRET}
        signIn:
          resolvers:
            - resolver: usernameMatchingUserEntityName   # GitHub login must match a User entity name
```
Then the frontend sign-in page: replace the guest provider with GitHub per the current docs ("Sign-in" under Auth) (`SignInPage` with `githubAuthApiRef`; in the new frontend system use the SignIn extension). Run with env vars:
```bash
export AUTH_GITHUB_CLIENT_ID=... AUTH_GITHUB_CLIENT_SECRET=...
yarn start
```
Expected: login works only if a `User` entity named like your GitHub login exists (Lab 02 provides them via GitHub org sync). Interview point: **resolver choice is the security decision**: `usernameMatchingUserEntityName` or `emailMatchingUserEntityProfileEmail` ties identity to the catalog; do not use a resolver that auto-creates users (or `dangerouslyAllowSignInWithoutUserInCatalog`) unless you have deliberately accepted that anyone who authenticates with the IdP gets in. For enterprise: Microsoft/Entra provider with `emailMatchingUserEntityProfileEmail` and the groups come from the Entra org sync (Lab 02). Add `backend.auth.externalAccess` for service-to-service/static tokens for CI (scoped, expiring).

## Step 4 - Permissions (RBAC) policy as a backend module
```bash
yarn --cwd packages/backend add @backstage/plugin-permission-backend @backstage/plugin-permission-node @backstage/plugin-permission-common @backstage/plugin-catalog-backend
```
Create `packages/backend/src/permissionPolicy.ts`:
```ts
import { createBackendModule } from '@backstage/backend-plugin-api';
import { AuthorizeResult, PolicyDecision } from '@backstage/plugin-permission-common';
import { PermissionPolicy, PolicyQuery, PolicyQueryUser } from '@backstage/plugin-permission-node';
import { policyExtensionPoint } from '@backstage/plugin-permission-node/alpha';

class AcmePermissionPolicy implements PermissionPolicy {
  async handle(request: PolicyQuery, user?: PolicyQueryUser): Promise<PolicyDecision> {
    const owns = user?.info.ownershipEntityRefs ?? [];
    const isPlatform = owns.includes('group:default/platform-admins');

    // Only platform admins may delete catalog entities or unregister locations.
    if (request.permission.name === 'catalog.entity.delete' ||
        request.permission.name === 'catalog.location.delete') {
      return { result: isPlatform ? AuthorizeResult.ALLOW : AuthorizeResult.DENY };
    }
    return { result: AuthorizeResult.ALLOW };
  }
}

export default createBackendModule({
  pluginId: 'permission',
  moduleId: 'acme-policy',
  register(reg) {
    reg.registerInit({
      deps: { policy: policyExtensionPoint },
      async init({ policy }) {
        policy.setPolicy(new AcmePermissionPolicy());
      },
    });
  },
});
```
Register it and enable:
```ts
// packages/backend/src/index.ts
backend.add(import('@backstage/plugin-permission-backend'));
backend.add(import('./permissionPolicy'));
```
```yaml
# app-config.yaml
permission:
  enabled: true
```
Verify: log in as a non-platform user, try "Unregister entity": denied; as a `platform-admins` member: allowed. Production pattern: policy reads a config/YAML or uses the RBAC plugin (Red Hat `backstage-plugin-rbac`) for UI-managed roles; keep conditional rules (e.g. "owners can edit their own entities") with `createCatalogConditionalDecision`.

## Step 5 - Production image
Backstage's recommended host-build approach (build on the host/CI, copy into a slim image):
```bash
yarn install --immutable
yarn tsc
yarn build:backend
docker build . -f packages/backend/Dockerfile --tag acme-backstage:0.1.0
docker run --rm acme-backstage:0.1.0 node --version
```
Explain: `yarn build:backend` produces `skeleton.tar.gz` and `bundle.tar.gz`; the Dockerfile installs only production dependencies and runs `node packages/backend --config app-config.yaml --config app-config.production.yaml`. Serve the frontend from the backend container (single container) via `app-backend` plugin: simplest to operate; separate CDN/nginx only if you need edge caching.
`app-config.production.yaml`:
```yaml
app:
  baseUrl: http://backstage.127.0.0.1.nip.io:8080
backend:
  baseUrl: http://backstage.127.0.0.1.nip.io:8080
  listen: { port: 7007 }
  database:
    client: pg
    connection:
      host: ${POSTGRES_HOST}
      port: ${POSTGRES_PORT}
      user: ${POSTGRES_USER}
      password: ${POSTGRES_PASSWORD}
auth:
  environment: production
  providers:
    github:
      production:
        clientId: ${AUTH_GITHUB_CLIENT_ID}
        clientSecret: ${AUTH_GITHUB_CLIENT_SECRET}
        signIn: { resolvers: [{ resolver: usernameMatchingUserEntityName }] }
```
(Use the `production` key under `auth.providers.github` to match `auth.environment`.) Later in CI (Module 06 reusable build workflow) this image is built, scanned, signed and its digest committed to GitOps.

## Step 6 - Postgres on Kubernetes with CloudNativePG
```bash
kubectl config use-context kind-hub
helm repo add cnpg https://cloudnative-pg.github.io/charts && helm repo update
helm install cnpg cnpg/cloudnative-pg -n cnpg-system --create-namespace
kubectl -n cnpg-system rollout status deploy/cnpg-cloudnative-pg
kubectl create namespace backstage
cat <<'EOF' | kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata: { name: backstage-pg, namespace: backstage }
spec:
  instances: 2
  storage: { size: 5Gi }
  bootstrap:
    initdb: { database: app, owner: backstage }
  postgresql:
    parameters: { max_connections: "200" }
  resources: { requests: { cpu: 100m, memory: 256Mi }, limits: { memory: 512Mi } }
EOF
kubectl -n backstage get cluster backstage-pg -w
kubectl -n backstage get secret backstage-pg-app -o jsonpath='{.data.password}' | base64 -d; echo
```
Expected: `Cluster in healthy state`, 2 instances (primary + replica), services `backstage-pg-rw` (primary) and `-ro`. The `backstage` role needs `CREATEDB` for per-plugin DBs:
```bash
kubectl -n backstage exec backstage-pg-1 -- psql -U postgres -c "ALTER ROLE backstage CREATEDB;"
```
Production: managed Postgres (Azure Database for PostgreSQL Flexible Server from Module 05) with private endpoint, PITR backups, HA; CNPG here shows the operator pattern. Back up the DB: Backstage state is mostly *derived* (catalog rebuilds from sources) except scaffolder task history, permissions state, plugin data (Tech Insights results, etc.): do back it up.

## Step 7 - Deploy with the Backstage Helm chart
```bash
kind load docker-image acme-backstage:0.1.0 --name hub
kubectl -n backstage create secret generic backstage-secrets \
  --from-literal=AUTH_GITHUB_CLIENT_ID=... --from-literal=AUTH_GITHUB_CLIENT_SECRET=... \
  --from-literal=GITHUB_TOKEN=... --from-literal=BACKEND_SECRET=$(openssl rand -hex 32)
helm repo add backstage https://backstage.github.io/charts && helm repo update
cat > backstage-values.yaml <<'EOF'
backstage:
  image: { registry: "", repository: acme-backstage, tag: "0.1.0", pullPolicy: IfNotPresent }
  extraEnvVarsSecrets: [backstage-secrets]
  extraEnvVars:
    - { name: POSTGRES_HOST, value: backstage-pg-rw }
    - { name: POSTGRES_PORT, value: "5432" }
    - name: POSTGRES_USER
      valueFrom: { secretKeyRef: { name: backstage-pg-app, key: username } }
    - name: POSTGRES_PASSWORD
      valueFrom: { secretKeyRef: { name: backstage-pg-app, key: password } }
  command: ["node", "packages/backend", "--config", "app-config.yaml", "--config", "app-config.production.yaml"]
  resources: { requests: { cpu: 250m, memory: 512Mi }, limits: { memory: 1Gi } }
  podSecurityContext: { runAsNonRoot: true, runAsUser: 1000, seccompProfile: { type: RuntimeDefault } }
  containerSecurityContext: { allowPrivilegeEscalation: false, capabilities: { drop: [ALL] } }
ingress:
  enabled: true
  className: nginx
  host: backstage.127.0.0.1.nip.io
postgresql: { enabled: false }
EOF
helm install backstage backstage/backstage -n backstage -f backstage-values.yaml
kubectl -n backstage rollout status deploy/backstage
kubectl -n backstage logs deploy/backstage | head -30
```
Verify: `http://backstage.127.0.0.1.nip.io:8080`. Check `/.backstage/health/v1/readiness` and `/liveness`: use these as probes. Note: the chart's value names evolve; run `helm show values backstage/backstage` and adjust. Production wiring: image by digest, config in ConfigMap + secrets via External Secrets (Module 03 Lab 07), HPA (2+ replicas; the backend is stateless apart from caches; scheduled tasks use DB locks so multiple replicas are safe), PDB, NetworkPolicy, OTel.

## Break it on purpose
1. Remove `CREATEDB` from the role; Backstage fails on start with `permission denied to create database`; restore it.
2. Set the wrong `app.baseUrl`/`backend.baseUrl`; sign-in redirects fail (callback mismatch). Typical production incident after moving hosts.
3. Run 2 replicas with `backend.cache` unset; observe fine behaviour (DB-based locks) but memory cache per pod; discuss Redis for cache at scale.
4. Set `permission.enabled: true` with no policy module installed: default policy allows all (a silent security gap): add a test that checks a delete is denied.

## Cleanup
```bash
helm uninstall backstage -n backstage; docker rm -f bs-pg
```
(Keep the CNPG cluster and namespace for later labs.)

## Talking points
New backend system, database-per-plugin, sign-in resolver as the identity contract, permission policy as backend module, host-build image, operator-run Postgres vs managed, stateless replicas, probes, secrets via ESO, backups of non-derivable data.
