# Lab 05-02 - Module versioning, distribution and consumer upgrades

## Goal
Publish modules by git tag with semver, consume them pinned, ship a breaking change as a major with a `moved` block so consumers upgrade without recreating resources, and define a deprecation policy.

## Prereqs
Lab 01 repo `~/acme-tf-modules`, `gh` authenticated, GitHub org from Module 00.

## Step 1 - Push the repo and tag v1.0.0
```bash
cd ~/acme-tf-modules
gh repo create acme-lab-<you>/acme-tf-modules --private --source=. --remote=origin --push
git tag -a v1.0.0 -m "standard-service 1.0.0"
git push origin v1.0.0
git ls-remote --tags origin
```
Expected: `refs/tags/v1.0.0` listed. Why tags: immutable, reviewable pins. A monorepo of modules tags the whole repo; for independent versions per module use `modules/<name>/vX.Y.Z` style tags or one repo per module (trade-off: release overhead vs unrelated churn).

## Step 2 - Consume it pinned
Create a consumer root:
```bash
mkdir -p ~/consumer-payments && cd ~/consumer-payments
cat > main.tf <<'EOF'
module "svc" {
  source              = "git::https://github.com/acme-lab-<you>/acme-tf-modules.git//modules/standard-service?ref=v1.0.0"
  service             = "payments"
  env                 = "dev"
  region_short        = "weu"
  resource_group_name = "rg-acme-lab"
  location            = "westeurope"
  tenant_id           = "00000000-0000-0000-0000-000000000000"
  owner               = "team-payments"
  cost_center         = "cc-1234"
}
EOF
terraform init -backend=false
```
Verify: `.terraform/modules/modules.json` contains the ref. Interview point: `//` selects the subdirectory, `?ref=` pins the tag; Renovate understands this and opens upgrade PRs.

## Step 3 - A breaking change, done safely (v2.0.0)
Scenario: rename the internal module `postgres` to `database` for clarity. Naively this destroys and recreates the DB. Use `moved`.
```bash
cd ~/acme-tf-modules
git mv modules/postgres modules/database
sed -i 's#"../postgres"#"../database"#; s#module "postgres"#module "database"#; s#module.postgres#module.database#g' modules/standard-service/main.tf modules/standard-service/outputs.tf
cat >> modules/standard-service/main.tf <<'EOF'

moved {
  from = module.postgres
  to   = module.database
}
EOF
cd examples/basic && terraform init -backend=false -upgrade && terraform validate
```
Why: `moved` blocks live in the module so every consumer's state is re-addressed automatically on plan. Keep them for at least two majors.

## Step 4 - Changelog, release, upgrade path
```bash
cd ~/acme-tf-modules
cat > CHANGELOG.md <<'EOF'
## v2.0.0
BREAKING: internal module `postgres` renamed `database`. A `moved` block migrates state; plan must show "has moved", 0 to destroy.
Upgrade: change ?ref=v1.x to ?ref=v2.0.0, run terraform plan, confirm no destroy.
## v1.0.0
Initial release.
EOF
git add -A && git commit -m "feat!: rename postgres to database (moved block)" 
git tag -a v2.0.0 -m "v2.0.0" && git push origin main --tags
```

## Step 5 - Prove the upgrade is non-destructive
Needs a real subscription or a state fixture. With cloud: apply v1.0.0 in the consumer, bump to `?ref=v2.0.0`, then:
```bash
terraform init -upgrade
terraform plan
```
Expected: `module.svc.module.postgres[0]... has moved to module.svc.module.database[0]...` and `0 to destroy`. Without cloud: describe this step in your own words; it is the interview answer to "how do you evolve modules without breaking consumers".

## Step 6 - Automate upgrade PRs (Renovate)
In the consumer repo:
```bash
cat > renovate.json <<'EOF'
{ "extends": ["config:recommended"], "terraform": { "enabled": true } }
EOF
```
Why: consumers get PRs for new tags with the changelog, plan runs in the PR (Lab 06), humans merge. You, as platform owner, can query who is on old versions: `gh search code 'acme-tf-modules.git//modules/standard-service?ref=v1' --owner acme-lab-<you>`.

## Distribution options (be ready to compare)
| Option | Pros | Cons |
|--------|------|------|
| Git tags | zero infra, simple | no UI/search, auth via git |
| Terraform Cloud/Enterprise private registry | UI, semver constraints `~> 2.1`, no-code provisioning | licence/cost |
| Artifactory/Nexus | one artifact store for everything | more setup |

## Deprecation policy (write it down)
Support N and N-1 majors; deprecate with a CHANGELOG entry and a `precondition`/`check` warning; remove after 2 quarters with a migration guide; track consumers' versions in a Backstage scorecard.

## Break it on purpose
1. Retag: `git tag -f v1.0.0` on a new commit and push with `--force`. Show how a consumer's lock/cache differs. Lesson: never move tags; tag protection rules in GitHub (rulesets) make this impossible.
2. Remove the `moved` block and run plan against v1 state: destroy-and-create of the DB shows.

## Cleanup
```bash
rm -rf ~/consumer-payments
```

## Talking points
Semver contract, `moved` blocks, immutable tags with rulesets, Renovate-driven upgrades, visibility into consumer versions, explicit deprecation.
