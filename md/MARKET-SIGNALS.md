# MARKET-SIGNALS.md

> Lab-wide market alignment contract, revision 1.
> What hiring managers, execs, and partners measurably value; what the lab
> collects to prove it; how roadmaps, resumes, and positioning stay aligned.
>
> **Last scan:** 2026-10-28 | **Driver:** Linear DEV-125 (living research ticket)
> **Decay policy:** every fact carries a date + source. Facts older than 2
> quarters are stale until re-verified by the next DEV-125 pass.

## Status

Living document. Does NOT self-update. Refresh path:

1. DEV-125 pass reruns the bounded web scan (3-6 queries: hiring keywords,
   exec-valued engineering metrics, AI/agentic market numbers).
2. Findings updated in place in DEV-125; decayed claims marked
   `(stale, replaced by X)`.
3. Deltas synced here and into Linear doc
   "Contribution Ledger: Conventions + Stats" -> Market signal lens.
4. Cadence: monthly while applications/partnerships are active, quarterly
   otherwise.

## Verified signals (2026-10-28 scan)

### Metrics that reviewers and execs check

| Signal                          | Verified benchmark / adoption                                  | Source            |
| ------------------------------- | -------------------------------------------------------------- | ----------------- |
| Deploy frequency                | Elite >1 deploy/service/day                                    | LinearB 2025      |
| Cycle / lead time               | Elite <26h                                                     | LinearB 2025      |
| Change failure rate             | Elite <1%                                                      | LinearB 2025      |
| MTTR                            | Elite <6h                                                      | LinearB 2025      |
| SPACE framework                 | >50% of dev teams; top productivity frame 2025                 | Atlassian DevEx   |
| DORA + satisfaction blend       | deploy freq, perf, lead time, satisfaction each ~34-35% of orgs | JetBrains DevEx   |
| Revenue per Engineer            | Median $892K; top quartile $1.5M+                               | DX Core 4 (2025)  |
| AI impact measures              | time saved, PR throughput, CFR, engaged users, innovation rate  | DX survey of 18   |

### Exec pain: unclear AI ROI

- ~2/3 of software firms deployed genAI; only 10-15% productivity lift;
  gains evaporate when saved time is not redirected (Bain 2025).
- Coding is ~25-35% of idea-to-launch time: SDLC-wide AI beats codegen-only
  (Bain 2025).
- Unclear ROI is the top CIO/CTO adoption hurdle; fix pattern = scorecards +
  developer satisfaction (BCG 2025).

### Keyword watchlist (growth-verified)

| Keyword                              | Verified growth                                |
| ------------------------------------ | ---------------------------------------------- |
| AI agents / agentic / multi-agent    | +2043% YoY postings (Dice 2025)                |
| Edge intelligence                    | +608% YoY                                      |
| RAG                                  | +475% YoY                                      |
| Enterprise AI infra (LangChain etc.) | >100% YoY                                      |
| MCP server                           | ~2000 registry entries, +407% since 2025-09    |
| Agentic-systems roles                | median salary $225K (frontierroles.com)        |
| AI skills in postings                | 36% of US tech postings (Dice 2025)            |

Adjacent vocabulary to use: evals + LLM observability, context engineering,
AI safety + guardrails, platform engineering, edge/serverless, FinOps,
SDLC-wide AI.

### Market position shift

Builders -> Orchestrators. AI engineer responsibility share (Northzone):
prompt/retrieval optimization 90%, model eval 80%, LLM stack
(RAG/chains/agents) 70%, fine-tuning 50%, infra 35%, training models 5%.
DevArts Lab operates in the orchestration tier.

## Collection contract (what every ledger entry must carry)

`## Impact` numbers draw from this set, not ad-hoc counts:

- deploys/week, lead time, cycle time
- change failure rate, MTTR, uptime, p99 latency
- tests added/passing, coverage delta
- endpoints live, users served, records processed
- cost saved or unit-cost basis (RpE proxy where estimable)
- time saved vs manual baseline (AI ROI narrative)

`## Evidence` is mandatory: every number links to a click-verifiable source
(commit SHA, PR, live URL, dashboard).

Work descriptions tag market vocabulary on the way in (agentic AI, MCP
server, RAG, evals, guardrails, platform engineering, edge/serverless,
FinOps) so label-and-keyword queries surface the right bullets.

## Alignment rules (bind every lab product + pitch)

1. Position as SDLC-wide orchestration (plan, build, test, deploy, observe),
   never codegen-only. Matches where Bain says real returns come from.
2. Lead every product story with a quantified scorecard: cadence, failure
   rate, time saved, cost basis. This is the artifact execs say is missing.
3. Resume/ledger framing maps each bullet to DORA/SPACE/Core 4 vocabulary
   first, proprietary terms second.
4. Vision/mission framing: healthy growth = measurable leverage per
   engineer, not headcount. We sell orchestration and verified impact.
5. New roadmap items check the keyword watchlist: does the item compound an
   in-demand signal or chase a decaying one? Note the answer in the issue.

## Source register (2026-10-28)

Dice AI Hiring Guide 2025 | LinearB 2025 Software Engineering Benchmarks |
DX Core 4 Revenue-per-Engineer benchmarks | DX "How 18 companies measure AI
impact" | Bain "From Pilots to Payoff" 2025 | BCG "AI-Enabled Engineering
Excellence" 2025 | Northzone "On The Pulse: AI Engineer Hiring" | MCP
anniversary post 2025-11 | frontierroles.com agentic salary data |
InfoWorld "9 AI development skills" | Atlassian + JetBrains DevEx surveys.
