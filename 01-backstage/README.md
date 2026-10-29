# 01 - Backstage and developer portal engineering

Treat the portal as a **product** with users, adoption metrics and a roadmap, not a deployed tool.

| Lab | Topic |
|-----|-------|
| [01](lab-01-deploy/README.md) | Create the app, Postgres, GitHub/OIDC auth, permissions (RBAC), container image, Kubernetes deploy |
| [02](lab-02-catalog/README.md) | Entity model, `catalog-info.yaml` standards, GitHub discovery, Entra/LDAP org sync, validation, orphan detection |
| [03](lab-03-scaffolder/README.md) | Golden-path template (repo + CI + catalog + ArgoCD + Terraform PR), custom TypeScript action, template drift strategy |
| [04](lab-04-techdocs/README.md) | TechDocs with MkDocs, CI build + external storage |
| [05](lab-05-plugins-scorecards/README.md) | Community plugins, custom frontend + backend plugin (new backend system), Tech Insights scorecards |
| [06](lab-06-operating/README.md) | Upgrades, HA/performance, observability, product metrics and roadmap |

[Scenario interview Q&A](interview.md)

## Versions note
Backstage releases monthly and moves fast. The labs use the **new backend system** (`createBackend`, `backend.add(import(...))`) and the current frontend system patterns. Always create the app with the current `@backstage/create-app`, record its version, and read the release notes before copying plugin snippets: package names and options drift. Where a step depends on a plugin's current config keys, check the plugin README.

## The portal-as-product framing
- Users: developers (daily), SRE/security (scorecards), engineering managers (ownership, DORA), new hires (onboarding).
- Jobs to be done: "create a service in minutes", "find who owns X", "see my deploys and health", "read the docs".
- Metrics: weekly active users / total engineers, templates used per week, time-to-first-deploy, catalog completeness (% entities with owner/docs/on-call), search success, NPS.
- Roadmap: quarterly themes tied to developer pain (survey + support tickets), not plugin fashion.

## Say it in 2 minutes
"Backstage is our front door: catalog as the source of truth for ownership and relationships, scaffolder golden-path templates that create the repo, CI, catalog entry, ArgoCD app and an infra PR in one form, TechDocs built in CI, and plugins surfacing Kubernetes, ArgoCD, CI, PagerDuty, cost and scorecards. We run it on Kubernetes with Postgres, SSO, permission policies, a monthly upgrade cadence with canary, and treat adoption and catalog quality as KPIs. Template drift is handled by versioned templates plus automated update PRs."
