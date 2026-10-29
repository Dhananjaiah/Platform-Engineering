# Lab 01-03 - Software Templates: golden path, custom action, template drift

## Goal
Build a template that, from one form, creates the GitHub repo (code + `catalog-info.yaml` + thin CI caller), opens a PR to `acme-gitops` (ArgoCD app/overlays), opens a PR to `acme-infra` (Terraform), and registers the component. Add a custom TypeScript scaffolder action with a unit test. Then solve **template drift**.

## Prereqs
Labs 01-02, repos `acme-gitops`, `acme-infra`, `acme-platform-workflows` (Modules 03, 05, 06). GitHub integration token with `repo` and `workflow` scopes (use a GitHub App in production).

## Step 1 - Install the modules you need
```bash
cd acme-backstage
yarn --cwd packages/backend add @backstage/plugin-scaffolder-backend-module-github
```
```ts
// packages/backend/src/index.ts (scaffolder core is already added by create-app)
backend.add(import('@backstage/plugin-scaffolder-backend-module-github'));
```
List the available actions at `http://localhost:3000/create/actions`: your documentation for what you can call.

## Step 2 - Template repo layout (`acme-templates`)
```bash
gh repo create acme-lab-<you>/acme-templates --private --clone && cd acme-templates
mkdir -p java-spring-aks/skeleton/{app,.github/workflows,gitops/overlays/dev,infra}
```
Skeleton files use Nunjucks (`${{ values.name }}`):
```bash
cat > java-spring-aks/skeleton/catalog-info.yaml <<'EOF'
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: ${{ values.name | dump }}
  description: ${{ values.description | dump }}
  annotations:
    github.com/project-slug: ${{ values.orgName }}/${{ values.name }}
    backstage.io/techdocs-ref: dir:.
    backstage.io/kubernetes-id: ${{ values.name }}
    argocd/app-name: ${{ values.name }}-dev
    acme.io/template: java-spring-aks
    acme.io/template-version: "1.0.0"
spec:
  type: service
  lifecycle: experimental
  owner: ${{ values.owner | dump }}
  system: ${{ values.system | dump }}
EOF
cat > java-spring-aks/skeleton/.github/workflows/ci.yml <<'EOF'
name: ci
on: { pull_request: {}, push: { branches: [main] } }
permissions: { contents: read }
jobs:
  ci:
    uses: ${{ values.orgName }}/acme-platform-workflows/.github/workflows/ci-java.yml@v1
  image:
    needs: ci
    if: github.ref == 'refs/heads/main'
    permissions: { contents: read, packages: write, id-token: write, attestations: write, security-events: write }
    uses: ${{ values.orgName }}/acme-platform-workflows/.github/workflows/build-image.yml@v1
    with: { image-name: ${{ values.orgName }}/${{ values.name }} }
EOF
cat > java-spring-aks/skeleton/gitops/overlays/dev/kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: ${{ values.name }}-dev
resources: [../../base]
images:
  - name: ghcr.io/${{ values.orgName }}/${{ values.name }}
    digest: sha256:0000000000000000000000000000000000000000000000000000000000000000
EOF
cat > java-spring-aks/skeleton/infra/terragrunt.hcl <<'EOF'
include "root" { path = find_in_parent_folders("root.hcl") }
terraform { source = "git::https://github.com/${{ values.orgName }}/acme-tf-modules.git//modules/standard-service?ref=v2.0.0" }
inputs = {
  service      = "${{ values.name }}"
  env          = "dev"
  owner        = "${{ values.owner | replace('group:default/', '') }}"
  cost_center  = "${{ values.costCenter }}"
  with_database = ${{ values.database }}
}
EOF
```
(A real skeleton also contains the app source: Spring Boot project, Dockerfile, `mkdocs.yml`, `docs/index.md`, `.gitignore`, `CODEOWNERS`.) Directory names containing `${{ }}` work in skeleton paths, but keep paths simple where possible.

## Step 3 - The template definition
```bash
cat > java-spring-aks/template.yaml <<'EOF'
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: java-spring-aks
  title: Java Spring service on AKS
  description: Golden path - repo, CI, catalog, ArgoCD app, Postgres/KeyVault via Terraform
  tags: [java, aks, golden-path, recommended]
  annotations:
    acme.io/template-version: "1.0.0"
spec:
  owner: group:default/platform-admins
  type: service
  parameters:
    - title: Service details
      required: [name, owner, system]
      properties:
        name:
          title: Name
          type: string
          pattern: '^[a-z][a-z0-9-]{2,30}$'
          ui:autofocus: true
        description: { title: Description, type: string }
        owner:
          title: Owner
          type: string
          ui:field: OwnerPicker
          ui:options: { allowedKinds: [Group] }
        system:
          title: System
          type: string
          ui:field: EntityPicker
          ui:options: { catalogFilter: { kind: System } }
    - title: Infrastructure
      properties:
        database: { title: Needs a Postgres database?, type: boolean, default: true }
        costCenter: { title: Cost center, type: string, pattern: '^cc-[0-9]{4}$' }
    - title: Repository
      required: [repoUrl]
      properties:
        repoUrl:
          title: Repository location
          type: string
          ui:field: RepoUrlPicker
          ui:options:
            allowedHosts: [github.com]
            allowedOwners: [acme-lab-<you>]
  steps:
    - id: fetch-app
      name: Generate service skeleton
      action: fetch:template
      input:
        url: ./skeleton
        values:
          name: ${{ parameters.name }}
          description: ${{ parameters.description }}
          owner: ${{ parameters.owner }}
          system: ${{ parameters.system }}
          orgName: acme-lab-<you>
          costCenter: ${{ parameters.costCenter }}
          database: ${{ parameters.database }}
        targetPath: ./out
    - id: publish
      name: Create repository
      action: publish:github
      input:
        repoUrl: ${{ parameters.repoUrl }}
        sourcePath: ./out
        description: ${{ parameters.description }}
        defaultBranch: main
        repoVisibility: private
        topics: [backstage, golden-path]
        requireCodeOwnerReviews: true
    - id: acme-request
      name: Register in platform inventory
      action: acme:resource:request
      input:
        service: ${{ parameters.name }}
        owner: ${{ parameters.owner }}
        costCenter: ${{ parameters.costCenter }}
    - id: gitops-pr
      name: Open GitOps PR (ArgoCD app)
      action: publish:github:pull-request
      input:
        repoUrl: github.com?owner=acme-lab-<you>&repo=acme-gitops
        branchName: onboard-${{ parameters.name }}
        title: "onboard ${{ parameters.name }}"
        description: "Created by Backstage template java-spring-aks. Owner ${{ parameters.owner }}"
        sourcePath: ./out/gitops
        targetPath: apps/${{ parameters.name }}
    - id: infra-pr
      name: Open Terraform PR
      action: publish:github:pull-request
      input:
        repoUrl: github.com?owner=acme-lab-<you>&repo=acme-infra
        branchName: onboard-${{ parameters.name }}
        title: "infra for ${{ parameters.name }}"
        description: "Postgres + KeyVault + identity for ${{ parameters.name }}"
        sourcePath: ./out/infra
        targetPath: live/dev/${{ parameters.name }}
    - id: register
      name: Register in catalog
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps['publish'].output.repoContentsUrl }}
        catalogInfoPath: /catalog-info.yaml
  output:
    links:
      - { title: Repository, url: "${{ steps['publish'].output.remoteUrl }}" }
      - { title: Open in catalog, icon: catalog, entityRef: "${{ steps['register'].output.entityRef }}" }
      - { title: GitOps PR, url: "${{ steps['gitops-pr'].output.remoteUrl }}" }
      - { title: Infra PR, url: "${{ steps['infra-pr'].output.remoteUrl }}" }
EOF
git add -A && git commit -m "template: java-spring-aks 1.0.0" && git push -u origin main
```
Register it: in Backstage config add the location, or as a provider-discovered `template.yaml`:
```yaml
catalog:
  locations:
    - type: url
      target: https://github.com/acme-lab-<you>/acme-templates/blob/main/java-spring-aks/template.yaml
      rules: [{ allow: [Template] }]
```
Run it from `/create` with name `orders-api`. Verify each step's output panel, the new repo, both PRs, catalog entity. Debug failed steps from the task log (each action's logs + dry-run via "Template Editor" -> Dry run).
Why PRs and not direct writes: policy checks and approval on gitops/infra; the PR is the audit trail and the rollback.
Security: `allowedOwners`, `allowedHosts` restrict where templates can publish; control who can run which template with scaffolder permissions (`scaffolderTemplateConditions`); templates run server-side with the integration token, so *template authorship = code execution rights*: review template PRs as code, and CODEOWNERS the template repo.

## Step 4 - Custom scaffolder action in TypeScript
Use case: call the internal Platform API to allocate a service ID, cost-center validation and DNS name before any repo is created. Create a backend module:
```bash
cd ../acme-backstage
mkdir -p plugins/scaffolder-backend-module-acme/src/actions && cd plugins/scaffolder-backend-module-acme
yarn init -y 2>/dev/null; cat package.json | head -5
```
(Preferably generate with `yarn new` -> `scaffolder-backend-module`, which wires lint/test/tsconfig, then replace the example action.) Action code:
```ts
// src/actions/resourceRequest.ts
import { createTemplateAction } from '@backstage/plugin-scaffolder-node';
import { Config } from '@backstage/config';

export function createResourceRequestAction(config: Config) {
  const baseUrl = config.getString('acme.platformApi.baseUrl');
  const token = config.getString('acme.platformApi.token');

  return createTemplateAction({
    id: 'acme:resource:request',
    description: 'Registers a new service with the Acme Platform API and returns its service ID',
    schema: {
      input: {
        service: z => z.string().regex(/^[a-z][a-z0-9-]{2,30}$/),
        owner: z => z.string(),
        costCenter: z => z.string().regex(/^cc-\d{4}$/),
      },
      output: {
        serviceId: z => z.string(),
      },
    },
    async handler(ctx) {
      const { service, owner, costCenter } = ctx.input;
      ctx.logger.info(`Requesting platform registration for ${service}`);

      if (ctx.isDryRun) {
        ctx.output('serviceId', 'dry-run-id');
        return;
      }

      const res = await fetch(`${baseUrl}/v1/services`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${token}`, 'Idempotency-Key': `${ctx.task.id}` },
        body: JSON.stringify({ service, owner, costCenter }),
        signal: ctx.signal,
      });
      if (!res.ok) {
        throw new Error(`Platform API returned ${res.status}: ${await res.text()}`);
      }
      const body = (await res.json()) as { id: string };
      ctx.output('serviceId', body.id);
    },
  });
}
```
Design points to articulate: **idempotency** (scaffolder tasks can be retried: send an Idempotency-Key), **dry-run support** (`ctx.isDryRun`), **abort signal**, **schema validation** (rejects bad input before running), **secrets** via config not template parameters, **least privilege token**, logging without secrets, and **output** consumed by later steps.
Module registration:
```ts
// src/module.ts
import { coreServices, createBackendModule } from '@backstage/backend-plugin-api';
import { scaffolderActionsExtensionPoint } from '@backstage/plugin-scaffolder-node/alpha';
import { createResourceRequestAction } from './actions/resourceRequest';

export const scaffolderModuleAcme = createBackendModule({
  pluginId: 'scaffolder',
  moduleId: 'acme-actions',
  register(env) {
    env.registerInit({
      deps: { scaffolder: scaffolderActionsExtensionPoint, config: coreServices.rootConfig },
      async init({ scaffolder, config }) {
        scaffolder.addActions(createResourceRequestAction(config));
      },
    });
  },
});
export default scaffolderModuleAcme;
```
Wire it:
```ts
backend.add(import('@internal/backstage-plugin-scaffolder-backend-module-acme'));
```
```yaml
acme:
  platformApi:
    baseUrl: http://platform-api.platform.svc
    token: ${PLATFORM_API_TOKEN}
```
Unit test (no network; mock fetch):
```ts
// src/actions/resourceRequest.test.ts
import { createMockActionContext } from '@backstage/plugin-scaffolder-node-test-utils';
import { ConfigReader } from '@backstage/config';
import { createResourceRequestAction } from './resourceRequest';

describe('acme:resource:request', () => {
  const config = new ConfigReader({ acme: { platformApi: { baseUrl: 'http://x', token: 't' } } });

  it('outputs the service id on success', async () => {
    global.fetch = jest.fn().mockResolvedValue({ ok: true, json: async () => ({ id: 'svc-1' }) }) as any;
    const ctx = createMockActionContext({ input: { service: 'orders-api', owner: 'group:default/team-orders', costCenter: 'cc-1234' } });
    await createResourceRequestAction(config).handler(ctx);
    expect(ctx.output).toHaveBeenCalledWith('serviceId', 'svc-1');
  });

  it('fails with the API status on error', async () => {
    global.fetch = jest.fn().mockResolvedValue({ ok: false, status: 409, text: async () => 'exists' }) as any;
    const ctx = createMockActionContext({ input: { service: 'orders-api', owner: 'group:default/team-orders', costCenter: 'cc-1234' } });
    await expect(createResourceRequestAction(config).handler(ctx)).rejects.toThrow('409');
  });
});
```
```bash
yarn workspace @internal/backstage-plugin-scaffolder-backend-module-acme test
```
(Package and test-util names depend on the Backstage version: check `yarn new` output and imports.) Stand in for the API locally with `docker run -p 8085:80 kennethreitz/httpbin` or a 10-line express stub returning `{ "id": "svc-1" }`.

## Step 5 - Template versioning and the drift problem
**Problem.** Template v1.0.0 produced 150 services. v1.4.0 fixes the CI caller, adds a policy file and bumps the Terraform module ref. 150 repos still have v1.0.0 output. Scaffolder is *fire-and-forget*: it does not track or update what it created. Interviewers ask this because naive answers ("re-run the template") are wrong.

**Strategy (layered, from cheapest to most invasive):**
1. **Minimise copied code.** Put logic in things that update centrally: reusable workflows (`@v1` moves for everyone), the golden Helm chart (Renovate bumps its version), Terraform product modules (Renovate bumps `?ref=`), base image via Renovate. Template output should be *thin glue* (caller workflow, catalog-info, a few config files). Most "template changes" then ship without touching repos.
2. **Stamp the output.** Every generated repo has `acme.io/template` and `acme.io/template-version` annotations in `catalog-info.yaml` (see skeleton) so the catalog knows who is on what.
3. **Measure drift.** Scorecard/Tech Insights check "on latest template minor" (Lab 05), and a query for the fleet:
```bash
curl -s "$BACKSTAGE/api/catalog/entities?filter=metadata.annotations.acme.io/template=java-spring-aks&fields=metadata.name,metadata.annotations" \
  | jq -r '.[] | [.metadata.name, .metadata.annotations["acme.io/template-version"]] | @tsv' | sort -k2 | uniq -c -f1
```
4. **Automated update PRs ("template sync bot").** A workflow in the templates repo runs a codemod per template version: for each repo on an older version, apply the delta and open a PR bumping the annotation. Minimal runnable form with `gh` (hand-typed):
```bash
for repo in $(gh search code 'acme.io/template-version: "1.0.0"' --owner acme-lab-<you> --json repository --jq '.[].repository.nameWithOwner' | sort -u); do
  echo "== $repo"
  gh repo clone $repo /tmp/sync/$(basename $repo) -- --depth 1
  ( cd /tmp/sync/$(basename $repo) && git checkout -b template/1.4.0 \
    && cp ~/acme-templates/java-spring-aks/migrations/1.0.0-to-1.4.0/policy.yml .github/ 2>/dev/null \
    && sed -i 's/acme.io\/template-version: "1.0.0"/acme.io\/template-version: "1.4.0"/' catalog-info.yaml \
    && git add -A && git commit -m "chore: apply template java-spring-aks 1.4.0" \
    && git push -u origin HEAD && gh pr create -t "Template update 1.4.0" -b "See CHANGELOG in acme-templates" )
done
```
Production tools for this: `copier update` (template engine with update support and answers file), **Renovate custom managers**, Cruft, OpenRewrite for Java code recipes, or Backstage plugins that call a workflow per repo. Each migration is a small, tested script in `migrations/<from>-to-<to>/` with a CHANGELOG entry and risk level (auto-merge for safe, human review for behavioural).
5. **Policy backstop.** Anything that *must* be uniform (security) is enforced centrally (rulesets, required workflows, Kyverno), not by templates, so drift in non-critical files is tolerable.
6. **Deprecation.** Template marked `spec.lifecycle: deprecated` + hidden from the picker; services on deprecated paths appear on a migration dashboard with a deadline.
**Answer in one sentence:** "I keep templates thin, stamp and measure versions, ship changes through reusable workflows/modules/charts that update centrally, and use codemod PR bots for the remaining file-level drift."

## Step 6 - Tests for templates (catch breakage before users)
- Backstage CLI/Template Editor dry-run in CI: `curl -X POST $BACKSTAGE/api/scaffolder/v2/dry-run` with the template and sample values.
- Render the skeleton and run `kubeconform`, `terraform validate`, `actionlint`, `yamllint` over the output in the templates repo CI.
- Canary: a nightly job creates a service from the template in a sandbox org, waits for CI green and first deploy to dev, then deletes it. Alert on failure. This is a **synthetic transaction for the platform** and measures time-to-first-deploy.

## Break it on purpose
1. Break the Terraform ref in the skeleton and watch the infra PR's CI fail: why the canary matters.
2. Rerun the same template with the same `name`: `publish:github` fails (repo exists); discuss idempotency, naming collisions and a pre-check action calling the platform API.
3. Give the integration token only `contents:read`: `publish:github` 403s. Observe error surfaced in task log.

## Cleanup
Delete test repos created by the template: `gh repo delete acme-lab-<you>/orders-api --yes` and close the PRs.

## Talking points
Golden path as a *composed* flow, PR-based hand-offs, thin templates, custom actions designed for idempotency/dry-run/secrets, templates are privileged code, drift handled by centralisation + stamping + codemod PRs + synthetic canary.
