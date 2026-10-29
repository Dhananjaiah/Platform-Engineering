# Rapid-fire (aim: 20 seconds each)

## Backstage
1. Which entity kinds exist? Component, API, Resource, System, Domain, Group, User, Location, Template.
2. Why owner = Group? People move; groups persist; avoids orphaned ownership.
3. Entity provider vs processor? Provider brings entities in; processor validates/enriches/emits relations.
4. Scaffolder drift? Fire-and-forget; solve with thin templates, version stamping, update-PR bots.
5. TechDocs builder in prod? `external`: CI builds and publishes to object storage.
6. Backstage DB layout? One database per plugin.
7. New backend system in one line? Plugins/modules declare service deps via `createBackendPlugin/Module` and are composed with `backend.add`.
8. Permission framework? Plugins declare permissions; a policy module decides ALLOW/DENY/conditional.

## IDP
9. Golden path vs cage? Strict on interfaces/security, flexible inside.
10. TVP? Smallest platform that removes the most cognitive load.
11. DORA four? Deploy frequency, lead time, change failure rate, time to restore.
12. Goodhart's law? A measure that becomes a target stops being a good measure.
13. Portal vs orchestrator? UI/catalog vs resource/config engine.

## GitOps / ArgoCD
14. App-of-apps vs ApplicationSet? Hand-written Applications vs generated from generators.
15. Auto-sync flags? `prune` deletes removed resources; `selfHeal` reverts manual drift.
16. Rollback in GitOps? `git revert` (record of truth); Argo UI rollback breaks with auto-sync.
17. Sync waves vs hooks? Ordering within sync vs actions at phases (PreSync/PostSync).
18. Hub-and-spoke risk? Credential concentration and deploy SPOF.
19. Secrets options? ESO (external store), Sealed Secrets (encrypted in git), SOPS.
20. Controller scaling? Shard by cluster, webhooks, manifest-generate-paths, exclusions.
21. AppProject purpose? Tenancy boundary: repos, destinations, kinds.

## Kubernetes
22. Requests vs limits? Scheduling/bin-packing and HPA basis vs enforcement ceiling (memory kill, CPU throttle).
23. PDB with 1 replica? Blocks drains forever.
24. Liveness vs readiness vs startup? Restart on deadlock; gate traffic; protect slow starters.
25. Workload Identity? Federated SA token exchanged for cloud token; no secrets.
26. PSS levels? privileged, baseline, restricted.
27. Default-deny + DNS? Always add egress to kube-dns.
28. Kyverno vs Gatekeeper? YAML/CEL with mutate/generate/verifyImages vs Rego with OPA reuse.
29. Cluster Autoscaler vs Karpenter? Scale groups on pending pods vs provision right-sized nodes directly.
30. OOMKilled exit code? 137.
31. Velero covers? Namespaced resources and volumes; use DB-native PITR for databases.

## Terraform
32. `moved` vs `state mv`? Declarative reviewable vs imperative.
33. `count` vs `for_each`? Index shifting vs stable keys.
34. State layout rule? Per env per component; avoid giant state.
35. Policy gates? tflint, Checkov/Trivy, Conftest on plan JSON, Sentinel.
36. Drift detection? Scheduled `plan -detailed-exitcode`.
37. Module versioning? Semver by git tag; `moved` blocks; deprecation policy.
38. OpenTofu? Open-source fork, drop-in, adds state encryption.
39. Crossplane vs Terraform? Continuous reconcile K8s API vs plan-reviewed apply.

## GitHub Actions
40. Reusable vs composite? Whole jobs with own runner/permissions vs step bundles.
41. Why SHA pin? Tags are mutable; supply-chain attacks move them.
42. OIDC benefit? Short-lived tokens, no stored secrets, scoped by sub claim.
43. `pull_request_target` danger? Runs with secrets in base context; never run PR head code.
44. ARC? Autoscaling ephemeral runners on Kubernetes.
45. SLSA? Supply chain levels; provenance attestations, hardened builder.
46. Cosign keyless? Signing with OIDC identity and transparency log (Rekor).
47. Required workflows? Org rulesets forcing platform checks.
48. Concurrency for applies? `cancel-in-progress: false`.

## Staff behaviours
49. RFC vs ADR? Proposal under review vs recorded decision.
50. Pre-wiring? Aligning key stakeholders 1:1 before publishing.
51. Pull vs push adoption? Make new path better first, then set sunset dates.
52. Roadmap form? Themes, outcomes, metrics, "not doing".
53. ROI honesty? State assumptions, ranges, and validate with measured data.
54. Disagree and commit? Decision owner decides; record dissent and revisit trigger.
55. Bus factor? Pair on critical systems and document runbooks.
56. Office hours value? Source of roadmap items and adoption.
57. Blocking vs advisory review feedback? Label them so authors know what must change.
58. Exceptions? Owner, ticket, expiry.
59. Platform says no when? Single-team benefit adding permanent cost, duplicates, conflicts with security.
60. First 90 days? Listen (interviews, data), baseline metrics, pick one painful workflow, ship a thin slice with a pilot team.
