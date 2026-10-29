# Module 07 - Staff-level IC: scenario-based interview Q&A

---

### 1. "Tell me about a time you got teams to adopt something they resisted."
**Answer structure.** Lead with the outcome. Then: context (stakes, scale), why they resisted (specific, charitable), what *you* did to understand the objection, how you reduced their cost (automation, pairing, parity period), how you created pull (features only on the new path), how you used data/leadership at the right time (sunset dates after the new path was proven), result with numbers, and what you learned. Use story #1 from `stories.md`. Avoid: heroics, blaming the team, "I mandated it".

### 2. You have 8 engineers and 40 requests. How do you build the roadmap?
**Answer.** Cluster the requests into problems; quantify each (teams affected, hours lost, incidents, risk); score transparently (value x confidence x strategic fit / effort); look at dependencies (ownership data unlocks scorecards and cost showback); reserve capacity (20 % toil/ops, 10 % unplanned); define 3-4 quarterly themes with outcomes and metrics; publish "not doing" with reasons; review with a stakeholder group quarterly. Make trade-offs explicit to leadership instead of silently overloading the team.

### 3. How do you influence a senior engineer in another org who disagrees with your design?
**Answer.** Understand their constraint first; find the shared goal; bring data/prototype instead of opinion; invite them to co-own the risky part (design partner); use an RFC so the debate is written and time-boxed; accept being wrong (change the design if they're right); if still stuck, agree on the decision owner and a revisit trigger, then disagree and commit. Never go around them to their manager without telling them.

### 4. Explain "why GitOps" to a VP in one minute, and to a skeptical engineer in two.
**Answer.** VP: incidents from untracked change, audit evidence, rollback speed, security (no CI credentials to clusters), costs and timeline. Engineer: PR is the deploy, drift correction, revert to roll back, daily-use benefits, how it feels on Monday, how to escape in emergencies. (See Module 07 Lab 03 Step 5.)

### 5. Your platform's biggest adopter is unhappy and threatens to leave the platform
**Answer.** Treat as a design-partner signal: meet quickly, listen for specifics, reproduce their pain, give a dated plan for fixes or an interim workaround, offer an engineer pair for a sprint, and agree on a review. Where requirements are legitimately different, define a supported deviation (exception with expiry) or an extension point. Share the learning with the roadmap. If they leave anyway, ensure security baseline remains enforced and document why.

### 6. How do you make architecture decisions when you're not the expert?
**Answer.** Identify the experts and decision criteria; run a time-boxed spike or POC with the real scenarios; write the alternatives and trade-offs in an RFC/ADR; get review from specialists (security, SRE, data); choose reversible options when uncertain (two-way doors); define a review trigger; communicate assumptions. Staff engineers are judged by decision quality and speed of learning, not by already knowing the answer.

### 7. A team ships a workaround that violates platform standards and it hits production. What do you do?
**Answer.** First stabilise: is there security or reliability risk now? Fix or mitigate with the owner. Then ask why the standard path didn't serve them (missing capability, slow approval, unawareness). Add the capability or fix the process; add a guardrail (policy) if the risk is real, in audit mode first; record an exception with expiry if needed; share the lesson without blame. Track deviations as roadmap input.

### 8. Describe how you run a design review that people don't dread.
**Answer.** Pre-read, written questions beforehand, start from the problem and constraints, separate blocking from advisory, ask questions instead of dictating, time-box, summarise decisions and owners in the doc, follow up. Make it a forum for learning with a rotating chair; recognise good designs publicly. (Lab 07-04.)

### 9. How would you structure a platform team of 8? What roles and interaction modes?
**Answer.** Product-minded lead/PM, 2 engineers on developer experience (Backstage, templates), 2 on delivery (CI/CD, ArgoCD), 2 on runtime/infrastructure (AKS, Terraform, policy), 1 SRE/observability; shared on-call for platform-critical systems; X-as-a-Service for most, time-boxed collaboration for new capabilities; rotating "support engineer" to protect focus; embed security partner. Metrics owned by the team; roadmap reviewed with an advisory group. Avoid single owners (bus factor).

### 10. The company is cutting costs; platform headcount is at risk. What do you do?
**Answer.** Make the value visible: savings (retired tools, cloud cost reductions), time saved, incident reduction, audit effort avoided; show the cost of not having it (return to ticket queues). Offer a prioritised plan at several funding levels (what we stop doing at each), propose revenue-neutral changes (consolidate tools, buy where cheaper), and partner with finance on numbers. Protect the highest-leverage components (CI/CD, GitOps, security guardrails), and communicate transparently with the team.

### 11. How do you keep standards from becoming bureaucracy?
**Answer.** Automate compliance (policy as code, templates), keep contracts minimal (interfaces and security, not implementation), review standards yearly and retire dead ones, give a fast exception path, measure friction (survey, deviation reasons), and ask "which incident or outcome does this protect?" for every rule. Standards without enforcement via defaults become documentation nobody reads.

### 12. Walk me through how you'd handle a major platform incident as the senior engineer in the room.
**Answer.** Establish an incident commander (not necessarily me), communicate cadence, separate roles (investigate, communicate, scribe), stabilise before root cause (rollback, failover, disable), keep a timeline, avoid blame, protect responders from overload, communicate to stakeholders in plain language; afterwards write the post-incident review with contributing factors and tracked follow-ups with owners; share broadly; check follow-ups completed in 30 days.

### 13. How do you decide when to build something vs adopt open source or buy?
**Answer.** Differentiation (does it make us better than others?), total cost of ownership, team capability, time-to-value, risk and exit options. Buy/adopt for commodity; build for unique workflows; always keep data in open formats and git; revisit at fixed intervals. (Module 02 Lab 05.)

### 14. Tell me about a time you failed.
**Answer.** Choose a real, meaningful failure; own it plainly; explain the impact, what you learned and the systemic fix (not just "I'll be more careful"); show you changed how the team works. Interviewers look for ownership and learning; avoid humble-brags.

### 15. How do you stay technically deep while leading through influence?
**Answer.** Ship: take a hard technical slice each quarter (prototype, spike, critical review, on-call rotation); read incident reviews; pair with engineers; run POCs for decisions; keep a learning plan (CNCF projects, upgrade notes); delegate depth appropriately but verify key assumptions. The goal is credibility and judgment, not hands-on-keyboard for everything.
