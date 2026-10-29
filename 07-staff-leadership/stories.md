# STAR stories bank (12) - fill with your real experience

Format: **Situation** (context, scale, stakes) -> **Task** (your responsibility) -> **Action** (what *you* did: 3-5 concrete steps, with trade-offs) -> **Result** (numbers, what changed, what you learned). Keep each to 2 minutes. The examples below are templates tied to this course's labs; replace the facts with true ones from your career, never invent. Prepare numbers you can defend.

Each story should be tagged by the competency it proves: `[influence] [adoption] [strategy] [stakeholders] [communication] [architecture] [mentoring] [incident] [failure]`.

## 1. Got a resistant team to adopt something new [influence][adoption]
**S:** 40 services deployed via Jenkins `kubectl apply`; a critical payments team opposed GitOps ("it will slow us down and we have PCI deadlines").
**T:** Own the GitOps migration across the org; I had no authority over that team.
**A:** (1) Interviewed the lead to find the real objection: fear of losing hotfix speed and audit risk. (2) Built a prod-safe emergency path (pre-approved revert workflow) addressing hotfix speed. (3) Paired with them for a day to migrate one service; automated the conversion. (4) Showed their own data: 3 incidents from unreviewed changes. (5) Turned audit evidence into a benefit: PRs are the change record. (6) Made them design partners and credited them in the all-hands.
**R:** Their lead time fell from 2 days to 6 hours; they became the strongest advocate; org reached 85 % GitOps in 3 quarters. **Learning:** objections are product feedback.

## 2. Led a major migration end to end [adoption][strategy]
Jenkins to GitHub Actions for 300 jobs: baseline, segmentation (30 % dead), reusable workflows, bot PRs, waves, burn-down, sunset and credential revocation. Result: CI p95 25 -> 9 min, $120k/yr saved, 0 incidents attributable. (See Module 07 Lab 02, Module 06 Lab 06.)

## 3. Wrote an RFC that changed direction [strategy][communication]
Proposed GitOps promotion via PR over Image Updater; pre-wired security and SRE, documented alternatives, ran a 10-day review, recorded dissent; accepted with the decision owner's call; 6-month review confirmed lead time -30 %. (Lab 07-01.)

## 4. Built the portal as a product [strategy][influence]
Backstage with 15 % weekly usage; interviews, metrics, golden path as the only new-service route, catalog bot PRs; WAU to 62 % in 2 quarters; time-to-first-deploy 12 days -> 4 hours. (Module 01.)

## 5. Production incident you led or contributed to [incident]
Admission webhook outage froze deployments: diagnosed `failurePolicy: Fail` with a down policy engine, patched the webhook config, restored in 18 minutes; follow-ups: HA, excludes, alerts, game-day. Include what you'd do differently. (Module 04 interview Q6.)

## 6. A decision you got wrong / a failure [failure]
Pick a real one: e.g. mandated a scan step that doubled pipeline time; developers bypassed it; you measured, rolled back to a nightly deep scan plus a fast PR gate, apologised publicly, and added a performance SLO for platform changes. Shows ownership and learning.

## 7. Said no to a request / managed scope [strategy][stakeholders]
Declined a bespoke plugin for one team; offered inner-source with review and a template; documented in the RFC; team shipped it in 2 weeks; platform avoided permanent cost.

## 8. Influenced security/compliance without friction [stakeholders]
Embedded signing and SBOM in the reusable workflow, Kyverno audit -> enforce in waves, exceptions with expiry; developers did nothing; audit evidence automated; time to audit evidence 3 weeks -> 1 day.

## 9. Mentored someone to promotion [mentoring]
Delegated ownership of ArgoCD notifications/DORA events with an RFC and demo; weekly 1:1s; pre-reviewed drafts; they presented to leadership; promoted next cycle; bus factor on the system 1 -> 3.

## 10. Cross-region stakeholders [communication][stakeholders]
Platform team across 3 time zones: async RFCs, rotating meeting times, regional champions, follow-the-sun office hours; resolved a conflict between two regions' standards via data and a compromise ADR.

## 11. Technical depth moment [architecture]
Choose a hard problem: ArgoCD controller sharding at 2,000 apps; or template drift strategy; or Terraform state split of a 3,000-resource state with zero downtime. Walk through diagnosis, options, trade-offs, validation, rollout. (Module 03 Lab 08, Module 01 Lab 03, Module 05 Lab 03.)

## 12. Used data to change leadership's mind [communication][strategy]
Showed lead-time breakdown (70 % waiting for manual prod approval) and CFR 8 %; proposed automated approvals with canary and SLO gates; got CAB exemption for low-risk changes; lead time p50 2.1 -> 0.6 days with CFR unchanged.

## How to deliver
- Lead with the result in one sentence, then tell the story.
- "I" for your actions, "we" for outcomes; be precise on your contribution.
- Quantify; include trade-offs and what you'd do differently.
- Expect follow-ups: "What did the other side say?", "What was the hardest part?", "What would you do now?".
