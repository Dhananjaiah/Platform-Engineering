# 05 - Terraform and Infrastructure as Code (platform-grade)

Focus: IaC that *other teams consume*. You are a module product owner, not a script author.

## Labs
| Lab | Topic | Needs cloud? |
|-----|-------|--------------|
| [01](lab-01-module-design/README.md) | Reusable module design and composition ("standard service" product module) | No (validate/plan only) |
| [02](lab-02-versioning/README.md) | Semver, release by git tag, consumer upgrade path, deprecation | No (GitHub) |
| [03](lab-03-state/README.md) | Remote state, layout, `import`, `moved`, safe refactor, state lock incident | Optional (uses local MinIO or Azure Storage) |
| [04](lab-04-env-governance/README.md) | Dir-per-env vs workspaces vs Terragrunt, naming/tagging | No |
| [05](lab-05-policy/README.md) | Policy as code: Conftest/OPA, Checkov, Trivy, tflint with must-fail fixtures | No |
| [06](lab-06-automation/README.md) | Plan on PR, apply on merge, OIDC, drift detection, Atlantis | GitHub (+ optional Azure) |
| [07](lab-07-testing/README.md) | `terraform validate`, native `terraform test`, Terratest | No (mock providers) |
| [08](lab-08-alternatives/README.md) | OpenTofu, Crossplane, Pulumi, and the Backstage-to-Terraform PR link | kind |

[Scenario interview Q&A](interview.md)

## Principles to state in the interview
1. **Small modules, composed.** Primitives (network, identity, keyvault, postgres) are composed into product modules ("standard service"). Consumers use the product module and never see 40 inputs.
2. **Sensible secure defaults.** Private endpoints on, public access off, TLS min 1.2, tags required. Opting out is explicit and visible in review.
3. **Contract = inputs, outputs, version.** Breaking a contract means a major version and a migration note.
4. **State is blast radius.** One state per environment per component. Never a giant state.
5. **Policy in the pipeline, not the wiki.**
6. **Everything via PR, apply from CI with OIDC.** No laptop applies in shared envs.

## Say it in 2 minutes
"I publish a small catalog of versioned modules and a few product modules that compose them. State is split per env and component in remote backends with locking. Consumers pin versions by git tag; I ship a changelog and `moved` blocks so upgrades don't recreate resources. CI runs fmt, validate, tflint, Checkov, Conftest, then plan on PR and apply on merge via OIDC. Drift detection runs nightly. Backstage templates open PRs against the infra repo instead of calling apply directly."
