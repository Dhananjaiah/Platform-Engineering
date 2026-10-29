# Lab 07-02 - Driving adoption: a migration campaign with data

## Goal
Run a measurable migration (kubectl/Jenkins deploys -> GitOps via ArgoCD, or Jenkins -> GitHub Actions) as a campaign: baseline, segmentation, waves, tooling, comms, burn-down, sunset. Produce the tracking data with commands.

## Prereqs
GitHub org with several repos (or your employer's org). `gh`, `jq`.

## Step 1 - Baseline (you cannot show progress without it)
```bash
ORG=acme-lab-<you>
gh repo list $ORG --limit 500 --json name,isArchived,pushedAt --jq '[.[]|select(.isArchived|not)]|length'

# who is still on the legacy path?
gh search code --owner $ORG --filename Jenkinsfile --json repository --jq '[.[].repository.nameWithOwner]|unique' > legacy-jenkins.json
gh search code --owner $ORG 'kubectl apply' --json repository --jq '[.[].repository.nameWithOwner]|unique' > legacy-kubectl.json
# who is on the target path?
gh search code --owner $ORG 'acme-platform-workflows/.github/workflows/build-image.yml' --json repository --jq '[.[].repository.nameWithOwner]|unique' > on-platform.json
jq -s '{legacy_jenkins:(.[0]|length), legacy_kubectl:(.[1]|length), on_platform:(.[2]|length)}' legacy-jenkins.json legacy-kubectl.json on-platform.json
```
Record the numbers in the RFC and put them on a dashboard (a CSV in git + Grafana/Sheets chart). Baseline also: deploy frequency, lead time, rollback time, incident count tied to legacy deploys.

## Step 2 - Segment the population (not all repos are equal)
```bash
gh repo list $ORG --limit 500 --json nameWithOwner,pushedAt --jq '.[] | [.nameWithOwner, .pushedAt[:10]] | @tsv' | sort -k2 > repos-by-activity.tsv
awk -F'\t' -v cutoff=$(date -d '-180 days' +%F) '$2 < cutoff {dead++} $2 >= cutoff {live++} END {print "dead(no push 180d):", dead, "live:", live}' repos-by-activity.tsv
```
Segments: **dead** (archive, don't migrate), **simple/standard** (80 %: migrate by automation), **complex/special** (custom needs: design partner approach), **business-critical** (migrate last with extra safety, or first for a safety win depending on the risk profile). Assign each to an owning team via the catalog (`spec.owner`).

## Step 3 - Make migration cheap (reduce activation energy)
Adoption fails when migration costs more than perceived benefit. Build:
1. **Automated PR bot** that opens a migration PR per repo: adds the caller workflow, `catalog-info.yaml`, gitops overlay, with a checklist and a revert plan.
```bash
for r in $(jq -r '.[]' legacy-jenkins.json | head -3); do
  gh repo clone $r /tmp/mig/$(basename $r) -- --depth 1 &&
  ( cd /tmp/mig/$(basename $r) && git checkout -b migrate/platform-ci \
    && mkdir -p .github/workflows \
    && curl -fsSL https://raw.githubusercontent.com/$ORG/acme-templates/main/migrations/ci-caller.yml -o .github/workflows/ci.yml \
    && git add -A && git commit -m "ci: migrate to platform workflows" \
    && git push -u origin HEAD \
    && gh pr create -t "Migrate CI to platform workflow (see migration guide)" \
         -b "Automated by platform team. What changes: <list>. How to verify: <steps>. Rollback: revert this PR. Help: #platform-help, office hours Tue/Thu." )
done
```
2. **Migration guide** with a 10-minute "happy path" and a troubleshooting FAQ.
3. **Parity checks**: compare old and new pipeline results for the first N runs.
4. **White-glove for complex cases**: platform engineer pairs with the team for a day.
5. **Office hours** and a dedicated channel; fast response SLA.

## Step 4 - Waves and communication plan
| Wave | Who | When | Entry criteria | Exit criteria |
|---|---|---|---|---|
| 0 pilot | 3 friendly teams (design partners) | Q1 | guide drafted | 10 consecutive green runs, feedback incorporated |
| 1 | 15 volunteer teams | Q2 | pilot learnings | owners sign off, Jenkins job read-only |
| 2 | next 40 repos (bot PRs) | Q3 | tooling proven | parity verified |
| 3 | long tail + hold-outs | Q4 | exec-backed date announced | legacy credentials revoked |
Communications (written in advance): kickoff announcement (why, what's in it for you, timeline), monthly progress with a burn-down chart, per-wave reminders 4 weeks / 1 week / day-of, "what to do if blocked" path, a public FAQ, and a hall-of-fame for early adopters. Use multiple channels (email, Slack, team leads' forum, Backstage banner and scorecard).
Segment messaging: developers (what gets easier), managers (what you need to schedule and what it costs: "2 hours, we'll do the PR for you"), executives (risk and cost).

## Step 5 - Track and publish the burn-down
```bash
echo "date,legacy_jenkins,legacy_kubectl,on_platform" > burndown.csv
echo "$(date +%F),$(jq length legacy-jenkins.json),$(jq length legacy-kubectl.json),$(jq length on-platform.json)" >> burndown.csv
cat burndown.csv
```
Automate weekly (scheduled workflow committing the CSV), chart it, and share it in the monthly update. Track: % migrated, # blocked and why (top reasons), time per migration, support load, incidents attributable to migration, satisfaction of migrated teams.
Blocked-teams table: team, blocker, owner, ETA. Remove blockers publicly; they're product feedback.

## Step 6 - Create pull, then apply push
**Pull**: faster pipelines, free security scanning/evidence, on-call visibility, scorecard improvements, new features only on the new path (canary, previews).
**Push** (only after the new path is demonstrably better): freeze new jobs on the legacy system, announce a firm sunset date with executive sponsorship, make legacy slower/less supported on purpose (no new features, best-effort support), and for stragglers remove default credentials/secrets access.
Handling resistance: listen for the root cause (fear of regression, no time, missing feature, "not invented here"), offer a trade ("we do the migration; you review and test"), recruit champions, share peers' results, be willing to fix the platform. Never shame teams on dashboards; show trend and help.

## Step 7 - Sunset checklist (decommission properly)
1. Legacy system read-only for 30 days; archive configs/history to git/object storage.
2. Rotate or revoke every credential the legacy system held (cloud keys, kubeconfigs, tokens).
3. Remove DNS/ingress, shut down agents, delete VMs, cancel licences (track the saving).
4. Update docs and Backstage; remove the old template/golden path.
5. Retrospective with data: duration, effort, savings, lessons; publish.

## Break it on purpose
1. Skip the baseline and try to prove success after: you cannot. Write down the three numbers you wish you had.
2. Mandate a date without pull: simulate resistance (role-play with a peer as a skeptical team lead) and practise responding to "this slows us down".

## Talking points
Baseline first; segment (dead, standard, special, critical); automate migration to near-zero effort; waves with entry/exit criteria; burn-down transparency; pull before push; sunset with credential revocation; resistance as product feedback.
