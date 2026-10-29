# 02 - Internal Developer Platform (IDP)

Backstage is the UI. The IDP is everything behind it: golden paths, self-service APIs, platform architecture, abstraction design, measurement, and the build-vs-buy decision.

| Lab | Topic |
|-----|-------|
| [01](lab-01-architecture-ttp/README.md) | Platform architecture (CNCF planes), capability inventory, Team Topologies "thinnest viable platform" |
| [02](lab-02-golden-paths/README.md) | Golden paths: define, contract-test, support levels, deviation process |
| [03](lab-03-self-service-abstractions/README.md) | Self-service resources with Crossplane; abstraction design: CRD vs Score vs Helm values |
| [04](lab-04-dora-metrics/README.md) | DORA + developer-experience metrics pipeline and dashboards |
| [05](lab-05-build-vs-buy/README.md) | Build vs buy: decision matrix, TCO, ADR |

[Scenario interview Q&A](interview.md)

## Mental model
```
Developer control plane   Backstage / CLI / API
Integration & delivery    GitHub, Actions, ArgoCD, Rollouts
Resource plane            Terraform, Crossplane, clusters, managed services
Monitoring & logging      Prometheus, Grafana, Loki, OTel
Security                  Identity, policy, secrets, signing
```
Principles: platform as a product (users, roadmap, SLOs), thinnest viable platform, golden paths not golden cages, self-service with guardrails, measure outcomes (DORA, time-to-first-deploy), make the right way the easy way.

## Say it in 2 minutes
"The IDP is a set of products with contracts: a service golden path, a database, a namespace, secrets, DNS, environments. Each has a self-service interface (template, claim, API), guardrails by default, and an SLO. We choose abstractions deliberately: raw YAML for experts, a small spec for the 90 %, with an escape hatch. We measure DORA and time-to-first-deploy and survey developers, and we publish a roadmap. We bought where differentiation was low and built where our workflows are unique."
