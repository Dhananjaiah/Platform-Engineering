# 06 - GitHub Actions / CI-CD Platform Engineering

You build the pipelines hundreds of repos reuse. The product is the **paved pipeline**: secure by default, fast, versioned.

| Lab | Topic |
|-----|-------|
| [01](lab-01-reusable-workflows/README.md) | Central repo: reusable workflows, composite actions, versioning, SHA pinning |
| [02](lab-02-secure-pipeline/README.md) | Full pipeline: build, test, SAST, deps, Trivy, SBOM, Cosign, SLSA provenance, GitOps handoff |
| [03](lab-03-governance/README.md) | OIDC, least-privilege token, action allow-list, rulesets, required workflows, environments |
| [04](lab-04-arc-runners/README.md) | Actions Runner Controller on Kubernetes, runner groups, caching, speed |
| [05](lab-05-metrics/README.md) | Pipeline metrics: duration, failure rate, queue time, flaky tests |
| [06](lab-06-jenkins-migration/README.md) | Jenkins to GitHub Actions migration playbook |

[Scenario interview Q&A](interview.md)

## CI to CD handoff (draw this from memory)
```
push/PR -> reusable CI (build, test, scan, SBOM, sign, push image@digest)
        -> CI opens/updates a commit in acme-gitops (image digest in the env overlay)
        -> ArgoCD detects, syncs to dev -> promotion PRs to stg/prd (Module 03)
```
CI never holds cluster credentials. It only writes to git and a registry.

## Say it in 2 minutes
"A central `acme-platform-workflows` repo exposes versioned reusable workflows; actions are pinned by SHA and allow-listed. Cloud access uses OIDC with per-environment subjects, and `GITHUB_TOKEN` is read-only by default. Every image is scanned, SBOM'd, signed keyless with Cosign and has SLSA provenance, and admission policy verifies the signature at deploy time. CI hands off by committing the image digest to the GitOps repo. Runners are ephemeral on Kubernetes via ARC. I track duration, queue time, failure and flake rates per repo."
