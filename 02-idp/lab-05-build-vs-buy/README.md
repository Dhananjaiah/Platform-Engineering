# Lab 02-05 - Build vs buy: Backstage vs Port, Cortex, Humanitec, Roadie

## Goal
Run a defensible build-vs-buy evaluation: define criteria from your needs, time-boxed proof-of-concepts, a weighted scoring matrix, a TCO model, and an ADR. The interview tests the *method* and your trade-off articulation.

## Prereqs
Your Backstage from Module 01 (the "build" baseline). Trial accounts for one SaaS portal (Port/Cortex free tiers) are helpful but optional; the model below runs in a spreadsheet or a few shell commands.

## Step 1 - Landscape (know what each is)
| Option | What it is | Strength | Watch-outs |
|---|---|---|---|
| Backstage (OSS, CNCF) | portal framework you assemble and run | total flexibility, huge plugin ecosystem, no licence fees | needs a dedicated team, upgrades, you build the product |
| Roadie | managed Backstage SaaS | Backstage ecosystem without running it, faster start | less control over infra, plugin constraints, per-seat cost |
| Port | SaaS portal with flexible data model ("blueprints"), scorecards, self-service actions | fast to value, no-code modeling, good for catalog + actions | proprietary model, SaaS cost, less deep UI customisation than code |
| Cortex | SaaS internal developer portal focused on catalog, scorecards, initiatives | scorecards/governance strength, quick onboarding | less emphasis on scaffolding/workflow depth; cost |
| Humanitec | platform orchestrator (Score, resource definitions, environments) | dynamic config management, environment/resource provisioning API | it is orchestration, not primarily a portal UI; pairs with Backstage/Port |
| DIY (Crossplane+ArgoCD+custom UI) | assemble | exactly fits | highest ongoing cost |
Say clearly: **portal (UI/catalog) and orchestrator (resource/config engine) are different layers**: comparisons must not mix them.

## Step 2 - Requirements (weighted, from interviews and the TVP)
Create `evaluation.csv` by hand:
```bash
cat > evaluation.csv <<'EOF'
criterion,weight,backstage,roadie,port,cortex
Software catalog + ownership (org sync),5,4,4,5,5
Self-service workflows (scaffolding, multi-system),5,5,4,4,3
Customisation / unique internal workflows,4,5,3,3,2
Scorecards / maturity tracking,4,3,3,4,5
Time to first value (weeks),4,2,4,5,5
Ongoing operational burden (5 = low),4,2,4,5,5
Security / SSO / RBAC / audit,5,4,4,4,4
Ecosystem / plugin availability,3,5,4,3,3
Cost predictability at 400 engineers (5 = low/predictable),3,4,3,3,3
Vendor risk / lock-in (5 = low),3,5,3,2,2
Team skills fit (TypeScript/React in-house),3,5,3,3,3
EOF
awk -F, 'NR>1 {w=$2; for(i=3;i<=6;i++) s[i]+=w*$i; tw+=w} END {split("backstage roadie port cortex",n," "); for(i=3;i<=6;i++) printf "%-10s %.2f / 5\n", n[i-2], s[i]/tw}' evaluation.csv
```
(The scores above are illustrative: **replace them with your own, evidence-based scores** from the POC. In the interview, say the *weights* come from your context.) Sensitivity analysis: change the weights for "operational burden" and "time to value" and see if the winner flips; if it flips easily, the decision is close and you should pick the lower-regret option.

## Step 3 - Time-boxed POC (2 weeks per candidate, same scenario)
Same scripted scenarios for every candidate:
1. Ingest 20 real repos with owners from GitHub + org from Entra: how long, how much custom code?
2. Self-service: create a service from the golden path (repo + CI + infra PR + ArgoCD PR) end to end.
3. Scorecard: "has on-call, has docs, supported base image" across the 20 repos.
4. Integrations: ArgoCD, Kubernetes, PagerDuty, GitHub Actions.
5. Governance: SSO, RBAC per team, audit logs.
6. Day-2: upgrade story, backup/restore, support responsiveness.
Record **effort in person-days** and the **gaps** (what required hacks). Involve 5 developers from 3 teams in the evaluation and capture their feedback.

## Step 4 - TCO model (3 years)
```bash
cat > tco.sh.txt <<'EOF'
Backstage (build+run):   3 FTE x $200k loaded = $600k/yr  + infra $30k/yr    = ~$630k/yr  (year 1 includes +1 FTE x 6 months bootstrap)
Roadie (managed Backstage): 1.5 FTE x $200k = $300k/yr + SaaS $120k/yr (400 seats)  = ~$420k/yr
Port/Cortex (SaaS):      1 FTE x $200k = $200k/yr + SaaS $150k/yr                      = ~$350k/yr
EOF
cat tco.sh.txt
```
(Figures are illustrative placeholders. Use your own numbers.) Add opportunity cost: the 3 FTE could build golden paths instead; and risk costs: vendor price increases, lock-in exit cost (data export of the catalog/scorecards), and "we cannot do X" blockers. A useful metric: **cost per active developer per year** and **time-to-value**.

## Step 5 - Decision heuristics (what a Staff engineer says)
- **Build (Backstage)** if: >300 engineers, strong TypeScript/platform capacity (3+ people for 2+ years), many bespoke workflows/integrations, wish to own the data model, desire for ecosystem plugins.
- **Buy a SaaS portal** if: <200 engineers or small platform team, need value in weeks, standard catalog+scorecards+actions suffice, comfortable with SaaS data and pricing.
- **Managed Backstage** if you love the ecosystem but don't want to operate it.
- **Add an orchestrator (Humanitec or Crossplane)** when the pain is environment/resource/config management, regardless of portal choice.
- Hedge against lock-in: keep **source of truth in git** (`catalog-info.yaml`, manifests, Terraform), expose capabilities through APIs, so the portal is a replaceable view.
- Revisit annually with usage data.
Context matters more than the tool: "The portal is 20 % of the value; the golden paths and the platform APIs behind it are 80 %."

## Step 6 - Write the ADR (0004)
```bash
mkdir -p docs/adr && cat > docs/adr/0004-portal-build-vs-buy.md <<'EOF'
# ADR 0004: Developer portal: Backstage (self-run) vs SaaS
Status: accepted | Date: 2026-11-15
## Context
420 engineers, TypeScript-capable platform team of 6, strong need for custom golden paths wired to GitHub, Terraform and ArgoCD, security requires on-prem data residency for catalog metadata.
## Decision
Adopt Backstage, run by the platform team on AKS with Postgres. Revisit at 12 months.
## Alternatives
Port, Cortex (fast, but limited workflow customisation and data residency concerns), Roadie (considered; rejected for residency), Humanitec (complementary; deferred, revisit for environment management).
## Consequences
+ full control, plugin ecosystem, no per-seat fees. - requires ~3 FTE ongoing and monthly upgrades; slower to first value (mitigated by a thin first release).
## Success metrics and review
Weekly active users > 60 % by Q3; time-to-first-deploy p50 < 1h; catalog completeness > 90 %; if platform burden > 3.5 FTE or WAU < 30 % at month 12, re-evaluate SaaS.
## Exit plan
All catalog/golden-path content lives in git and open formats; export scripts tested.
EOF
git add -A && git commit -m "docs: ADR 0004 build vs buy"
```

## Break it on purpose
1. Change the weights to favour "time to value" and "operational burden" and note the winner flips: write down what you'd have to believe for each choice.
2. Remove the exit plan from the ADR and ask yourself what happens if the vendor doubles prices.

## Talking points
Portal vs orchestrator distinction, criteria from requirements, POC with identical scenarios, TCO incl. opportunity cost, sensitivity analysis, hedging lock-in through git as source of truth, explicit review triggers.
