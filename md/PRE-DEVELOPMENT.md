# Pre-Development Blueprint

> What a DevArts Lab project must produce before the first line of
> application code. Companion to [MEGA-PLAN.md](MEGA-PLAN.md), which holds
> the platform research, the foundation decisions D1 to D10, guardrails,
> governance, cost model, and the master repository layout. This document
> turns that plan into a per-project production list with document outlines
> and exit criteria.

| Field        | Value                                              |
| ------------ | -------------------------------------------------- |
| Status       | Living document, revision 2                        |
| Audience     | Project leads, lab engineers, delivery agents      |
| Scope        | Research, design, architecture, planning artifacts |
| Not in scope | Code, configuration, implementation                |
| Depends on   | MEGA-PLAN revision 2 (sections cited inline)       |

---

## Table of contents

1. How the phases map to artifacts
2. Phase 1: Research and discovery
3. Phase 2: Design and UX
4. Phase 3: Architecture and tech stack (Cloudflare-native)
5. Phase 4: Agent, workflow, and implementation planning
6. Phase 5: Guardrails, governance, and security planning
7. Document locations in the master layout
8. Document outlines
9. Pre-development exit criteria
10. Tier-based minimum sets

---

## 1. How the phases map to artifacts

The lab's delivery lifecycle (MEGA-PLAN Section 8) has Research, Design,
Development, Launch, and Operate phases. Pre-development covers Research and
Design and ends at the Design gate. Each pre-development artifact below is
durable (stored in the repository under `md/` and mirrored to R2 by the
Delivery Workflow) and has an owner, a reviewer, and an exit criterion.

| Blueprint section | Lifecycle phase | Gate that consumes it                     |
| ----------------- | --------------- | ----------------------------------------- |
| 2 Research        | Research        | Research gate: problem statement approved |
| 3 Design and UX   | Design          | Design gate: architecture review          |
| 4 Architecture    | Design          | Design gate                               |
| 5 Agent planning  | Design          | Design gate                               |
| 6 Guardrails      | Design          | Design gate and Launch checklist          |

## 2. Phase 1: Research and discovery

Goal: understand the problem, the users, the market, and the constraints
well enough to choose a tier and write a PRD.

| Artifact                | Content                                                                                         | Location                      |
| ----------------------- | ----------------------------------------------------------------------------------------------- | ----------------------------- |
| Intake brief            | Client or internal sponsor, goals, success definition, budget band, timeline, known constraints | `md/research/intake.md`       |
| Market research         | Existing solutions, segments, opportunities, pricing norms                                      | `md/research/market.md`       |
| Competitor analysis     | Strengths, weaknesses, feature matrix, positioning gaps                                         | `md/research/competitors.md`  |
| User personas           | Roles, goals, pains, environment, accessibility needs                                           | `md/research/personas.md`     |
| Requirements draft      | Functional and non-functional requirements with IDs; traceable into PRD and ADRs                | `md/research/requirements.md` |
| Risk register (initial) | Project risks with likelihood, impact, owner; seeded from MEGA-PLAN Section 18                  | `md/research/risks.md`        |
| Tier proposal           | T0 to T3 recommendation with justification (MEGA-PLAN Section 5.3)                              | `md/research/tier.md`         |

Research methods the lab uses: desk research with cited sources, structured
interviews, Browser Run based scans where rendering is required, and the
lab's own research agents (MEGA-PLAN Section 15) with human review of every
claim that will reach a client.

## 3. Phase 2: Design and UX

Goal: visualize the user journey and the system before implementation.

| Artifact              | Content                                                                                   | Location                  |
| --------------------- | ----------------------------------------------------------------------------------------- | ------------------------- |
| PRD                   | Scope, core features, success metrics, non-goals, release criteria; links requirement IDs | `md/design/prd.md`        |
| User flows            | Step-by-step flows per persona, including agent-assisted and approval flows               | `md/design/user-flows.md` |
| Wireframes            | Low-fidelity layouts; reference to the design tool file                                   | `md/design/wireframes.md` |
| Visual design         | High-fidelity direction, brand, accessibility targets                                     | `md/design/visual.md`     |
| Content and messaging | Voice, key copy, error and empty states, email templates                                  | `md/design/content.md`    |

## 4. Phase 3: Architecture and tech stack (Cloudflare-native)

Goal: make the deliberate technical decisions and record them. Production
targets Cloudflare; local development runs the same `workerd` runtime via
`wrangler dev` or the Vite plugin (MEGA-PLAN Section 6).

| Artifact                  | Content                                                                                                                                                                                 | Location                          |
| ------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------- |
| Tech stack selection      | Frontend (Static Assets, Vite plugin, or vinext), backend (Workers), agents (Agents SDK), data (D1, DO SQLite, KV, R2, Hyperdrive), each with justification against MEGA-PLAN Section 4 | `md/architecture/tech-stack.md`   |
| System architecture (C4)  | Context, container, component diagrams; the Cloudflare products appear as containers                                                                                                    | `md/architecture/c4.md`           |
| Data models               | Entity-relationship diagrams and schemas, each entity mapped to its storage product and data class; D1 split strategy if growth could approach 10 GB                                    | `md/architecture/data-models.md`  |
| Environment map           | dev, preview, prod: which bindings exist in each, which are isolated, what the `previews` block declares (no production bindings), Access policy per environment                        | `md/architecture/environments.md` |
| Capacity and limits check | Which MEGA-PLAN Section 13 limits the design approaches, with re-verification date                                                                                                      | `md/architecture/capacity.md`     |
| Cost estimate             | Requests, CPU, storage, AI spend by month; budget threshold for approval gates (MEGA-PLAN Section 12)                                                                                   | `md/architecture/cost.md`         |
| ADRs                      | One per foundation decision D1 to D10 plus project-specific decisions, using MEGA-PLAN Appendix A                                                                                       | `md/adr/`                         |

### 4.1 Required foundation ADRs

Every project instantiates these from MEGA-PLAN Section 9 before
development. "Accept default" is a valid decision but must still be
recorded with the date the defaults were checked.

| ADR      | Foundation decision          | Lab default (summary)                                                                                                                     |
| -------- | ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| ADR-0001 | D1 Client dispatch model     | Tenant Durable Object inside one Worker; hybrid with Workers for Platforms when a flagship client needs it; Workflows stay in lab Workers |
| ADR-0002 | D2 Agent state model         | Durable Object per agent session plus Workflow per process; intent written before side effects                                            |
| ADR-0003 | D3 Storage selection         | D1 relational, DO SQLite per entity, R2 files, KV read-heavy config only                                                                  |
| ADR-0004 | D4 Configuration and tooling | `wrangler.jsonc`, pinned Wrangler, today's compatibility date                                                                             |
| ADR-0005 | D5 Frontend stack            | Static Assets with Vite plugin; vinext for Next.js                                                                                        |
| ADR-0006 | D6 Code execution isolation  | Dynamic Workers with egress blocked; Containers only for Linux needs                                                                      |
| ADR-0007 | D7 Observability pipeline    | Logs and Traces on; lab Tail Worker; Logpush to R2; Analytics Engine metering                                                             |
| ADR-0008 | D8 Secrets                   | Worker secrets, Secrets Store for shared, `.dev.vars` local                                                                               |
| ADR-0009 | D9 Identity and access       | Access on Previews and internal surfaces; end-user auth per project                                                                       |
| ADR-0010 | D10 CI/CD and release        | Workers Builds, Previews per branch, approval gate to production                                                                          |

## 5. Phase 4: Agent, workflow, and implementation planning

Goal: define module responsibilities and the agentic model so the codebase
starts with clean separation.

| Artifact               | Content                                                                                                                                        | Location                           |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------- |
| Frontend plan          | UI components, state management, interaction logic, accessibility plan                                                                         | `md/architecture/frontend-plan.md` |
| Backend plan           | API surface, domain modules, data access patterns, queue consumers                                                                             | `md/architecture/backend-plan.md`  |
| Agent cards            | One per agent: purpose, tools (MCP servers and scopes), state schema, schedule, approval points, audit events, failure modes                   | `md/agents/<name>.md`              |
| Workflow definitions   | Each business process as steps, retry policy, gates, idempotency keys, artifact storage; the project's own Delivery Workflow mapping           | `md/architecture/workflows.md`     |
| API and tool contracts | OpenAPI for HTTP, AsyncAPI for queues and events, MCP tool schemas with least-privilege scope declarations                                     | `md/api/`                          |
| Test strategy          | Workers Vitest integration coverage, DO and Workflow test APIs, Preview smoke tests, agent evaluation plan (tool-call success, regression set) | `md/architecture/testing.md`       |

Agent evaluation note: evaluation criteria are project-specific (for
example latency for a voice agent, extraction accuracy for a document
agent); the plan must name the metrics, the fixture set, and the pass
threshold.

## 6. Phase 5: Guardrails, governance, and security planning

Goal: design guardrails in, not bolt them on. Each row maps to MEGA-PLAN
Sections 10, 11, and 14.

| Artifact                     | Content                                                                                                                                         | Location                          |
| ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------- |
| Data classification register | Every dataset with class (public, internal, client-confidential, regulated), owner, storage product, residency note                             | `md/governance/classification.md` |
| Retention schedule           | Period per class; deletion Workflow design; platform retention facts (Logs 7 days, Workflow state 30 days, Queues 14 days, Time Travel 30 days) | `md/governance/retention.md`      |
| Audit event catalog          | MEGA-PLAN Appendix B baseline plus project events; which Worker emits each                                                                      | `md/governance/audit-events.md`   |
| Guardrail assignment         | For each MEGA-PLAN Section 10 guardrail: the mechanism, the configuration owner, the verification step                                          | `md/governance/guardrails.md`     |
| Threat model                 | MEGA-PLAN Section 14 table extended with project-specific threats and controls                                                                  | `md/security/threat-model.md`     |
| Access and identity plan     | Access applications and policies (Previews, admin, internal), token scopes for CI and agents, end-user auth decision                            | `md/security/access.md`           |
| Secret inventory             | Names, owners, storage (Worker secret or Secrets Store), rotation cadence; never values                                                         | `md/security/secrets.md`          |
| AI data policy               | Which prompt and response classes may be logged in AI Gateway; DLP and Guardrails settings; provider retention choices                          | `md/governance/ai-data.md`        |

## 7. Document locations in the master layout

The layout is defined once in MEGA-PLAN Section 16. The pre-development
artifacts occupy these directories:

```text
md/
  MEGA-PLAN.md            adopted revision or pointer
  PRE-DEVELOPMENT.md      this blueprint with the project's filled checklist
  adr/                    ADR-0001 to ADR-0010 plus project ADRs
  research/               intake, market, competitors, personas, requirements, risks, tier
  design/                 prd, user-flows, wireframes, visual, content
  architecture/           tech-stack, c4, data-models, environments, capacity, cost,
                          frontend-plan, backend-plan, workflows, testing
  agents/                 one card per agent
  governance/             classification, retention, audit-events, guardrails, ai-data
  security/               threat-model, access, secrets
  api/                    openapi, asyncapi, mcp tool schemas
  runbooks/               created during Development; launch, rollback, restore
```

Repository root files (`AGENTS.md`, `README.md`, `REPO_INDEX.md`,
`LICENSE`, `SECURITY.md`, `.gitignore`, `.dev.vars.example`,
`wrangler.jsonc`) are listed in MEGA-PLAN Section 16 and created by the
scaffold.

## 8. Document outlines

Minimum headings for the artifacts that most often come back incomplete.

### 8.1 PRD (`md/design/prd.md`)

1. Problem statement and sponsor
2. Goals and success metrics (measurable, dated)
3. Personas served
4. Scope: features in, features out, non-goals
5. Requirement traceability (IDs from `md/research/requirements.md`)
6. Tier and foundation defaults accepted or deviated (link ADRs)
7. Release criteria and launch definition
8. Open questions

### 8.2 C4 (`md/architecture/c4.md`)

1. System context: users, external systems, AI providers, client systems
2. Containers: each Cloudflare product in use as a container; data flows;
   environments annotated
3. Components: Worker modules, Durable Object classes, Workflow classes,
   agents, queues
4. Deployment view: dev, preview, prod; Previews isolation noted per binding

### 8.3 Data classification register (`md/governance/classification.md`)

Table columns: dataset, description, class, owner, storage product,
tenant-scoped (yes or no), residency requirement, retention period, audit
events emitted, AI exposure (may be sent to a model: yes or no).

### 8.4 Agent card (`md/agents/<name>.md`)

1. Purpose and population (lab operations or delivery)
2. Trigger: user, schedule, event, webhook, email
3. State schema and where it persists
4. Tools: MCP server, tool name, scope, why needed
5. Model path: AI Gateway name, provider order, spend threshold
6. Approval points (human-in-the-loop)
7. Audit events emitted
8. Failure modes and recovery (resume, retry, escalate)
9. Evaluation metrics and fixtures

### 8.5 Environment map (`md/architecture/environments.md`)

Table columns: binding, dev (local), preview (isolated or shared, resource
name), prod (resource name), notes. Followed by: Access policy per
environment, Workflow instance ID prefixing scheme, queue consumer testing
approach (consumers cannot target Previews).

### 8.6 Threat model (`md/security/threat-model.md`)

Table columns: threat, vector, asset and data class, foundation control
(MEGA-PLAN Section 14), project control, owner, verification. Followed by
accepted risks with sign-off.

## 9. Pre-development exit criteria

A project may enter Development when all of the following are true and
recorded in the Design gate approval (MEGA-PLAN Section 8):

1. PRD approved with measurable success metrics.
2. Tier recorded; every product outside the tier has an ADR.
3. ADR-0001 to ADR-0010 exist (accepted default or deviation), with the date
   the MEGA-PLAN defaults and relevant limits were re-verified.
4. C4 and data models committed, every entity mapped to a storage product
   and a data class.
5. Environment map complete; `previews` block declares no production
   bindings; Access policy defined for Previews and internal surfaces.
6. Cost estimate complete with an approval threshold for spend gates.
7. Agent cards exist for every agent in scope with tool scopes and approval
   points.
8. Workflow definitions sketched with steps, gates, retry policy, and
   idempotency approach.
9. Guardrail assignment complete: AI Gateway routing, MCP scopes, approval
   gates, audit events, CPU limits, secrets plan.
10. Data classification register, retention schedule, and AI data policy
    complete.
11. Threat model reviewed and accepted risks signed.
12. Test strategy written, including Preview smoke tests and agent
    evaluation metrics.
13. Linear issue tree created and linked to the engagement Workflow instance
    ID.

## 10. Tier-based minimum sets

Not every project needs every artifact. The tier (MEGA-PLAN Section 5.3)
sets the minimum.

| Artifact group                              | T0                  | T1  | T2  | T3  |
| ------------------------------------------- | ------------------- | --- | --- | --- |
| Intake brief, tier, risks                   | x                   | x   | x   | x   |
| Market, competitors, personas               |                     | x   | x   | x   |
| PRD, user flows                             | x                   | x   | x   | x   |
| Wireframes, visual, content                 |                     | x   | x   | x   |
| Tech stack, C4                              | x                   | x   | x   | x   |
| Data models, environment map                |                     | x   | x   | x   |
| Capacity check, cost estimate               | x                   | x   | x   | x   |
| ADR-0001 to ADR-0010                        | x (accept defaults) | x   | x   | x   |
| Frontend and backend plans                  |                     | x   | x   | x   |
| Agent cards, workflow definitions           |                     |     | x   | x   |
| API and MCP contracts                       |                     | x   | x   | x   |
| Test strategy                               | x                   | x   | x   | x   |
| Classification, retention, audit catalog    |                     | x   | x   | x   |
| Guardrail assignment, AI data policy        |                     |     | x   | x   |
| Threat model, access plan, secret inventory | x (short)           | x   | x   | x   |
| Per-client metering and dispatch design     |                     |     |     | x   |

---

## Revision log

| Revision | Date       | Change                                                                                                                                                                   |
| -------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1        | 2026-10-21 | Initial blueprint aligned with MEGA-PLAN revision 1                                                                                                                      |
| 2        | 2026-10-21 | Aligned with MEGA-PLAN revision 2: single master layout under `md/`, ADR-0001 to ADR-0010, document outlines, tier minimum sets, consistent numbering, ASCII punctuation |
