# Lab 07-03 - Roadmap, prioritisation, and executive communication

## Goal
Produce a multi-quarter platform roadmap with explicit prioritisation, outcomes and trade-offs; compute ROI in terms leadership accepts; write the one-pager and the "why GitOps" explanation for two audiences.

## Prereqs
Data from Module 02 Lab 04 (DORA) and Lab 01 (capability inventory). A text editor and a spreadsheet or `awk`.

## Step 1 - Gather inputs (problems before solutions)
Inputs: developer survey free-text, support channel top questions, ticket categories, incident postmortems (contributing factors), DORA numbers, security/audit findings, cloud bill, executive priorities (e.g. "ship product X faster", "reduce cloud cost 15 %", "SOC2"). Tag each pain with: who, how many teams, severity, evidence.
```bash
cat > pains.csv <<'EOF'
id,pain,teams_affected,hours_per_month_lost,incident_link,evidence
P1,No self-service DB provisioning (5 day tickets),45,900,no,ticket data
P2,Mixed CI systems and slow pipelines (p95 25 min),60,1500,no,CI metrics
P3,Prod config drift and manual hotfixes,38,300,yes,11 incidents
P4,No consistent service ownership data,70,400,yes,MTTR +20 min
P5,K8s version upgrades take 6 weeks each,12,500,yes,deprecation incident
P6,Cloud cost growth 30% YoY unattributed,70,200,no,finance report
EOF
column -s, -t pains.csv
```

## Step 2 - Prioritise with a transparent scoring model
RICE-like: Value = hours saved (x loaded rate) + risk reduction; Confidence; Effort (person-months); plus strategic multipliers. Compute with awk so the reasoning is visible and re-runnable:
```bash
cat > scoring.csv <<'EOF'
initiative,hours_saved_month,teams,risk_reduction_1to5,confidence_pct,effort_pm,strategic_1to3
Self-service DB (Crossplane/TF),900,45,3,80,6,2
CI consolidation to GH Actions,1500,60,3,70,10,3
GitOps promotion + rollback,300,38,5,80,8,3
Catalog ownership accuracy,400,70,4,60,4,2
K8s upgrade automation,500,12,4,70,5,2
Cost showback,200,70,2,80,3,2
EOF
awk -F, 'NR>1 { value=($2*0.5)+($3*1)+($4*100); score=(value*$5/100*$7)/$6; printf "%-35s score=%7.1f\n", $1, score }' scoring.csv | sort -t= -k2 -nr
```
State the model's limits: numbers are estimates, so the output *starts* the discussion. Apply judgment: dependencies (catalog ownership accuracy unlocks scorecards and cost showback), sequencing (CI consolidation before GitOps promotion), capacity (platform of 8 engineers), and risk (quick wins build credibility).

## Step 3 - Shape the roadmap (themes, outcomes, not features)
```markdown
# Platform roadmap FY27 (themes -> outcomes -> initiatives)
## Q1: Paved road foundation (outcome: time-to-first-deploy p50 from 12 days to 2 days)
- Golden path v1 (Java/Node), Backstage GA, GitOps for pilot teams
## Q2: Self-service and safety (outcome: provisioning tickets -70 %; rollback p50 < 5 min)
- DB/namespace self-service, GitOps promotion + Rollouts, ownership accuracy to 85 %
## Q3: Scale adoption (outcome: 70 % of services on golden path; CI p95 < 10 min)
- CI migration waves, scorecards, cost showback
## Q4: Resilience and efficiency (outcome: upgrade cadence quarterly; cloud cost -12 %)
- K8s upgrade automation, right-sizing programme, DR drills
## Not doing (and why): custom internal CLI, ML platform, multi-cloud abstraction.
```
Principles: now / next / later with confidence decreasing; outcomes with metrics; explicit "not doing"; capacity reserved (20 % for operations/toil, 10 % for unplanned); dependencies and owners; review quarterly with stakeholders; publish it. Tie each theme to a business outcome (faster delivery, fewer incidents, lower cost, audit readiness).

## Step 4 - ROI and the executive one-pager
Hand calculation (state assumptions):
```bash
cat > roi.sh.txt <<'EOF'
Engineers affected:                 420
Hours lost to friction/mo (survey+ticket data): 4.5 h/engineer/month -> 1,890 h/month
Target reduction after platform:    40 %  -> 756 h/month saved
Loaded cost per hour:               $90   -> $68,040/month = ~$816k/year
Incident cost avoided (11 incidents x 6 h x 6 people x $90 x 50% reduction): ~$18k/year (+ customer impact not counted)
Cloud savings (right-sizing 8% of $3M/year):  $240k/year
Retired tools (Jenkins licences/hosting):     $120k/year
Total annual benefit:               ~$1.19M
Platform cost (6 engineers x $200k + tooling $150k): $1.35M/year
Payback: below break-even on time-saved alone year 1; positive when delivery speed (revenue) and risk are included. Be honest.
EOF
cat roi.sh.txt
```
Honesty is the point: leaders trust numbers with stated assumptions and ranges (low/likely/high), and a plan to validate them with measured data after 2 quarters. The other half: non-financial outcomes (time to market, audit, retention/dev satisfaction).
One-pager structure (fits one screen):
1. **Ask / decision needed** (headcount, sponsorship, date).
2. **Why now** (3 bullets with numbers).
3. **What we will deliver** (3 outcomes, quarter by quarter).
4. **How we'll measure** (DORA, time-to-first-deploy, adoption).
5. **Cost and ROI** (range).
6. **Risks and mitigations** (top 3).
7. **What we are not doing**.

## Step 5 - Explain "why GitOps" to two audiences
**To an engineer (2 minutes):** "Today a deploy is someone running a script with credentials from their laptop or CI, and nobody can tell exactly what's in production or who changed it. With GitOps, a pull request *is* the deploy. You get review, history, easy rollback with a revert, and the cluster continuously corrects drift. CI never needs cluster credentials, so a leaked CI token can't touch prod. On Monday: you merge, ArgoCD shows rollout progress, and if it goes wrong you revert."
**To a VP (60 seconds):** "We've had 11 production incidents from untracked changes in six months, and audits ask us for change evidence we assemble by hand. GitOps makes every production change reviewed, recorded and reversible by design, cutting rollback from 40 minutes to under 5 and removing a whole class of incidents. It also takes cluster credentials out of CI, a security risk the auditors flagged. Cost: two engineers for two quarters; we'll show lead time and incident numbers improving by Q3."
Notice: engineer = mechanism and daily benefit; VP = outcomes, risk, money, timeline.

## Step 6 - Quarterly business review (QBR) template
Slides or doc: **What we said we'd do / did / didn't** (honest), **outcome metrics vs targets** (trend graphs), **adoption**, **customer feedback quotes + survey**, **incidents and learnings**, **next quarter plan + asks**. Keep it to 10 minutes plus Q&A. Share ahead of time.

## Break it on purpose
1. Re-run the scoring with different `effort` and `confidence` assumptions and see which initiatives are robust to changes: those are the safe bets.
2. Write the one-pager without numbers and ask a colleague to explain the ask back to you; if they can't, rewrite.

## Talking points
Problems before solutions, scoring that's transparent but not sacred, roadmap as outcomes with "not doing", honest ROI with ranges, audience-specific language, regular QBRs with data.
