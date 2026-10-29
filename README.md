# Platform Engineering: Staff-Level Interview Course (hands-on)

A practicals-only course for a Staff-level Platform Engineer interview (18+ years experience):
Backstage, Internal Developer Platform, GitOps/ArgoCD, Kubernetes (AKS/EKS/GKE), Terraform, GitHub Actions, and Staff-IC leadership.

## Rules of this course
1. **Every command is typed by hand.** No wrapper scripts, no Makefile, no one-shot bootstrap. Each step is: command, what it does, expected output, how to verify. You should be able to narrate every line in an interview.
2. **Production-grade only.** Pinned versions, least privilege, OIDC instead of long-lived secrets, policy as code, rollback and failure drills.
3. **Break it on purpose.** Each lab has a failure drill so you can say "here is how it fails and how I debug it".
4. **Every module ends with scenario-based production interview Q&A** (`interview.md`), written as incidents, design reviews and adoption stories with model answers.

## The running example: "Acme Platform"
One reference platform is built across modules, ending in the end-to-end flow you whiteboard in the interview:

```
Developer -> Backstage template -> GitHub repo (code + catalog-info + reusable CI)
          -> Terraform/Crossplane (cloud resources)
          -> CI: build, test, scan, SBOM, sign, push -> update GitOps repo
          -> ArgoCD ApplicationSets -> cluster(s) -> Argo Rollouts canary
          -> Backstage shows owner, docs, deploys, health, cost, scorecard
          -> DORA + DX metrics drive the roadmap
```

## Course map (build order)
| # | Module | Status |
|---|--------|--------|
| 00 | [Foundations: lab environment, repo topology, standards](00-foundations/README.md) | done |
| 01 | Backstage and developer portal engineering | planned |
| 02 | Internal Developer Platform | planned |
| 03 | GitOps and ArgoCD | planned |
| 04 | Kubernetes and cloud-native platform | planned |
| 05 | [Terraform and IaC](05-terraform/README.md) | done |
| 06 | [GitHub Actions / CI-CD platform](06-github-actions/README.md) | done |
| 07 | Staff-level IC: influence, adoption, roadmap | planned |
| 08 | Capstone: end-to-end golden path | planned |

Recommended order for learning: 00, 05, 06, 03, 04, 01, 02, 07, 08.

## Lab layout
```
NN-module/
  README.md        concepts + the interview narrative for the module
  lab-XX-name/
    README.md      Goal, Prereqs, Step 1..N (command / why / expected / verify), Break it, Cleanup, Talking points
    *.yaml, *.tf   config files that the steps apply by hand
  interview.md     scenario-based production Q&A for the module
```

## Interview delivery
Each module README ends with a "say it in 2 minutes" narrative. Practice drawing the end-to-end flow above from memory, then defend each box with a trade-off from the labs.
