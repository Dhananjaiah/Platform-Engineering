# 60-minute mock interview (interviewer and candidate sheets)

## Timeline
| Min | Segment | Interviewer prompt |
|---|---|---|
| 0-5 | Intro | "Walk me through a platform you built and what you were personally accountable for." |
| 5-20 | System design (whiteboard) | "Design an internal developer platform for 400 engineers on AKS. Start with the flow from 'developer opens the portal' to 'serving prod traffic'." |
| 20-35 | Deep dive (pick 2 per round) | GitOps/ArgoCD at scale; Terraform module lifecycle; Backstage operations; CI supply chain; K8s multi-tenancy |
| 35-47 | Incident/troubleshooting scenario | Choose from the module scenario banks; ask for commands and reasoning aloud |
| 47-57 | Staff behaviours | "Tell me about a time you got teams to adopt something they resisted." + follow-ups |
| 57-60 | Candidate questions | Candidate asks about team, roadmap, success metrics |

## Interviewer probes (use to push depth)
- "What did you measure, and what changed?"
- "What would break first at 10x?"
- "Who disagreed, and how did you handle it?"
- "What did you give up by choosing that?"
- "How do you roll it back?"
- "How would a new engineer debug this at 3 am?"
- "What is the smallest version you would ship first?"

## Scoring rubric (1-4 per row; 3 = meets Staff bar)
| Dimension | 1 | 2 | 3 | 4 |
|---|---|---|---|---|
| Technical depth | tool recall | works with docs | explains internals and failure modes | has scars, predicts non-obvious issues |
| System design | components only | flow without trade-offs | trade-offs, SLOs, failure modes, scale | ties design to org context, phased rollout |
| Operational thinking | happy path | monitoring mentioned | runbooks, rollback, DR, upgrades | game days, measured RTO/RPO, toil budgets |
| Security | basic | scanning/secrets | supply chain, least privilege, policy | threat-model driven, defence in depth with evidence |
| Product/adoption mindset | tool-centric | some user focus | metrics, roadmap, pull before push | data-driven prioritisation with exec narrative |
| Influence and communication | tells people what to do | explains well | adapts to audience, handles conflict | builds coalitions, writes decisions down |
| Ownership and learning | blames | partial | owns failures, follow-through | systemic fixes, mentors others |

Bar: no 1s, at least four 3s including technical depth, design, and influence.

## Candidate self-review checklist
- Did I clarify before answering? Did I give a recommendation instead of a survey of options?
- Did I name trade-offs and what I would measure?
- Did I mention failure behaviour and rollback?
- Did I use real numbers and "I" vs "we" precisely?
- Did I stay within time and check for follow-ups?
- What did I not know, and how did I handle it? ("I'd verify X by Y" beats bluffing.)

## Questions for the candidate to ask (shows Staff thinking)
1. What are the platform's top three outcomes this year and how are they measured?
2. What is the current adoption of the golden paths and what blocks the rest?
3. How are platform priorities decided and who can say no?
4. What was the last decision the platform team reversed and why?
5. How does platform on-call work, and what is the incident load?
6. How do Staff engineers influence direction here (RFC process, forums)?
