# 03 - GitOps and ArgoCD

Design the deployment model, not just an `Application` manifest.

| Lab | Topic |
|-----|-------|
| [01](lab-01-install-sso-rbac/README.md) | HA install, SSO (Keycloak/OIDC), RBAC, AppProjects per team, repo credentials |
| [02](lab-02-apps-sync/README.md) | Application, sync policies, waves, hooks, health checks, diff customization |
| [03](lab-03-applicationsets/README.md) | ApplicationSets (git, cluster, matrix, list, PR), app-of-apps, hub-and-spoke |
| [04](lab-04-repo-design/README.md) | Config repo design, Helm vs Kustomize, overlays, values hierarchy |
| [05](lab-05-promotion-rollback/README.md) | Promotion dev->stg->prd, Image Updater/Kargo, rollback |
| [06](lab-06-rollouts/README.md) | Argo Rollouts canary + blue-green with Prometheus analysis |
| [07](lab-07-secrets/README.md) | External Secrets, Sealed Secrets, SOPS |
| [08](lab-08-lifecycle-dr/README.md) | Drift, sharding, notifications, disaster recovery of ArgoCD itself |

[Scenario interview Q&A](interview.md)

## Model to defend
- Git is the source of truth; the cluster pulls (no CI credentials to clusters).
- App repo (code, builds image) is separate from config repo (`acme-gitops`, desired state).
- One **hub** ArgoCD manages many **spoke** clusters (trade-offs in Lab 03).
- ApplicationSets generate Applications from folders/clusters/PRs; humans never hand-write 300 Applications.
- AppProjects are the tenancy boundary (repos, destinations, resource kinds allowed).
- Promotion is a git operation (PR) that moves an image digest or chart version up an environment ladder.

## Say it in 2 minutes
"CI writes the image digest into the config repo. A hub ArgoCD with ApplicationSets discovers apps by directory and clusters by label and renders them with Kustomize overlays per environment. Teams get AppProjects limiting repos, namespaces and kinds. Promotion is a PR that bumps the overlay; prod needs approval. Rollouts do canary with Prometheus analysis and abort automatically. Secrets come from External Secrets, never git. Rollback is `git revert`, because Argo's UI rollback conflicts with auto-sync. ArgoCD itself is rebuildable from git plus a backup of its few stateful bits."
