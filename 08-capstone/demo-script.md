# 15-minute demo script ("from form to production")

Audience variants: engineers (full flow), security (supply chain), leadership (outcomes). Rehearse with a timer. Have a pre-recorded fallback video and pre-created tabs: Backstage, GitHub, ArgoCD, Rollouts dashboard, Grafana, terminal.

## 0:00-1:00 Problem (no screen)
"Last year a new service took 12 days and 9 tickets; 40 % of our platform tickets were provisioning. Today I'll show it taking under an hour, with security and compliance built in."

## 1:00-4:00 Create (Backstage)
Open Create, choose the golden path, fill the form (name, owner group via picker, system, DB yes). Point out guardrails: owner must be a Group, name pattern validated, allowed org only. Submit; narrate the task log as it runs: repo created, platform registration, two PRs, catalog registration.
Say: "Nothing here is magic: it makes a repo and two pull requests."

## 4:00-6:00 Review PRs
Show the infra PR: Terraform plan comment and policy results (tags, no public access). Show the GitOps PR (just a folder). Merge both. "Every change is reviewed and recorded; rollback is a revert."

## 6:00-9:00 CI
Push a code change. Show the pipeline: tests, CodeQL/Semgrep, Trivy, SBOM, Cosign signing, provenance. Run `cosign verify` live. "The workflow identity is the only thing allowed to produce a deployable image."

## 9:00-12:00 Deploy
ArgoCD shows the app syncing to dev. Open the Rollout: canary 10 % -> analysis -> 50 % -> 100 %. Show the Prometheus analysis run. Try deploying `nginx:latest` by hand: Kyverno denies. "CI never touches the cluster; admission verifies signatures."

## 12:00-14:00 See and measure
Back in Backstage: entity page with owner, docs, pipeline, Kubernetes, ArgoCD, on-call, cost, scorecard. Open the DORA dashboard: lead time, deploy frequency for this service. "Everything about this service is in one place; the metrics tell us where to invest next."

## 14:00-15:00 Result and ask
"Form to dev in 22 minutes; prod after approvals same day. 70 % of new services now use this path; tickets for provisioning are down 65 %." Close with the ask or next step (feedback form, office hours link, roadmap preview).

## Fallback plan
- If Backstage fails to load: show the template YAML and run the same steps via `gh`/`git` (the flow is git-native).
- If CI is slow: show a completed run and the evidence commands.
- If a cluster is flaky: play the recorded rollout.

## Likely questions after the demo (prepare answers)
- "What if we need something the template doesn't provide?" Deviation process, extension points.
- "How do rollbacks work?" Revert PR; auto-abort in canary.
- "What does it cost to run?" Platform team size, infra cost, ROI numbers.
- "How do you keep old services up to date?" Template stamping, bots, reusable workflows.
- "What if Backstage is down?" Delivery is independent.
