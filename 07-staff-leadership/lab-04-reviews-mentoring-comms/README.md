# Lab 07-04 - Design reviews, mentoring, office hours, demos, global stakeholders

## Goal
Run the daily mechanics of technical leadership: a design review that improves outcomes, a mentoring plan, office hours and demos that drive adoption, and a cadence that works across time zones. Each section has an artifact to create.

## Step 1 - Design review checklist (run it on a real design)
Create `design-review-checklist.md` and apply it to a design doc from this course (e.g. the Postgres-as-a-Service API from Module 02):
```bash
mkdir -p docs && cat > docs/design-review-checklist.md <<'EOF'
# Design review checklist (platform)
## Problem and fit
- [ ] Problem and users are clear; evidence provided. Is this the right problem?
- [ ] Does it fit the platform strategy and TVP? What are we *not* building?
## Design
- [ ] Interfaces/contracts defined (API, CRD, workflow inputs/outputs); versioning plan
- [ ] Failure modes: what happens when each dependency is down? Blast radius?
- [ ] Day-2: upgrade, rollback, backup/restore, scaling limits, capacity
- [ ] Idempotency, retries, ordering, concurrency, data consistency
- [ ] Simplest thing that works? What would we delete?
## Security and compliance
- [ ] AuthN/Z, least privilege, secrets handling, audit logging, supply chain
- [ ] Data classification and residency; policy enforcement points
## Operability
- [ ] SLOs/SLIs, alerts with runbooks, dashboards; who is on call?
- [ ] Debuggability: logs, traces, status conditions that tell users what's wrong
## Adoption and cost
- [ ] Migration/rollout plan, pilot users, docs, deprecation of old path
- [ ] Cost (infra, people, opportunity); success metrics
## Decision
- [ ] Reviewers' concerns captured; decision owner; follow-ups with owners and dates
EOF
```
How to *run* the review (what interviewers want to hear):
1. Read the doc beforehand; send written questions 24 hours earlier so the meeting is for discussion.
2. Start with the problem and constraints; ask "what would make this fail?" and "what are you least sure about?".
3. Separate **blocking** concerns (security, data loss, irreversible choices) from **suggestions**; label them.
4. Ask questions rather than prescribe: help the author arrive at the design; the author owns it.
5. Close with explicit outcomes: approved / approved with changes / revisit, owners, dates. Write a summary in the doc.
6. Run a lightweight architecture review forum (weekly 45 minutes, 2 designs max) with a rotating chair; publish decisions.
Mentoring angle: review the *thinking* (trade-offs, alternatives considered), not just the diagram.

## Step 2 - Mentoring and growth plan
Create a plan for two people (e.g. a mid-level and a senior engineer on the platform team):
```bash
cat > docs/mentoring-plan.md <<'EOF'
# Growth plan: <name> (quarterly)
Current level / target: mid -> senior (or senior -> staff)
## Strengths to use
## Growth areas (2-3 max): e.g. system design, influence without authority, operational ownership
## Stretch assignments
- Own the design and rollout of <ArgoCD notifications for DORA events> end to end, including the RFC and a demo
## Support from me
- Weekly 1:1 (30 min), pre-review of their RFC draft, shadow-then-lead for incident commander, introductions to security/SRE partners
## Evidence of progress (check at 6 and 12 weeks)
- RFC accepted with measurable outcome; led a design review; mentored an intern
EOF
```
Principles: delegate **ownership**, not tasks (outcome + context + guardrails); give feedback quickly and specifically (situation-behaviour-impact); share credit publicly; ask "what would you do?" before giving answers; sponsor, don't just mentor (nominate them for visible work); build **bus-factor** resilience (pairs on each critical system); document tribal knowledge as runbooks they write. Measure: people promoted, systems with two+ confident owners, quality of RFCs from the team.

## Step 3 - Office hours (adoption engine)
Set: twice weekly, 45 minutes, rotating host, a visible calendar link in Backstage and the platform channel, agenda doc with a "walk-up" and "deep dive" format. Keep a log (`docs/office-hours-log.md`): question, team, resolution, doc/product follow-up.
```bash
cat >> docs/office-hours-log.md <<'EOF'
| date | team | question | outcome | follow-up |
|---|---|---|---|---|
| 2026-11-05 | orders | netpol blocks metrics scrape | added allow in template | PR #212 updates golden path |
EOF
```
Every 10 questions on the same topic = a doc, a template fix, or a feature. Track attendance and satisfaction; use as a source for the roadmap.

## Step 4 - Demos (show, don't tell)
Monthly 20-minute demo: "from zero to production in 15 minutes", live, with a safety net (pre-recorded fallback). Structure: problem (30 s) -> demo (8 min) -> result with numbers (1 min) -> what's next + how to get help (1 min). Record, post in Backstage news. Demo for different audiences: developers (end-to-end flow), security (signed images blocked when unsigned), execs (before/after metrics).
Practice the capstone demo script in Module 08.

## Step 5 - Global stakeholders across time zones
- **Async-first**: decisions in written RFCs/PRs; summaries after meetings; recorded demos.
- **Overlap windows**: 2 meeting slots per week rotated to share the pain of inconvenient hours.
- **Follow-the-sun support**: office hours in two regions; regional champions; handoff notes.
- **Stakeholder map** (create it): for each group list interest, influence, what they need, communication channel and frequency.
```bash
cat > docs/stakeholders.md <<'EOF'
| Group | Cares about | Needs from us | Channel / cadence |
|---|---|---|---|
| App teams | speed, fewer toil tasks | easy paths, help, stability | office hours, Slack, demos |
| Security | controls, evidence, vulnerabilities | enforceable policies, audit trails | monthly sync, shared dashboards |
| SRE | reliability, on-call load | SLOs, runbooks, safe rollouts | weekly sync, incident reviews |
| Architecture | coherence, standards | RFCs, ADRs | architecture forum |
| Product/Finance | delivery speed, cost | outcomes and ROI | QBR |
| Leadership | strategy, risk | roadmap, metrics | QBR, one-pager |
EOF
```

## Step 6 - Handling conflict and escalation
Pattern: (1) understand the other party's goal and constraint; (2) find shared goal; (3) put data on the table; (4) offer options with trade-offs; (5) if stuck, escalate **together** with a written summary of options, not behind their back. Disagree and commit: record the decision and revisit trigger. Example: security wants a 2-day change freeze for scans vs platform wants continuous delivery; resolve by moving scans earlier in the pipeline and adding admission enforcement, so both goals are met.

## Break it on purpose
1. Run a design review where you only criticise; ask a colleague to rate how safe it felt. Re-run with questions first.
2. Schedule all meetings in your own time zone for a month; record absent teammates' participation: shows the cost.

## Talking points
Design reviews as teaching and risk reduction, blocking vs advisory feedback, delegating ownership, office hours as roadmap input, demos with numbers, async-first global working, stakeholder map and conflict resolution with data.
