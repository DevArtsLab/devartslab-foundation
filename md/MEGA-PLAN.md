# DevArts Lab Foundation: Mega Plan

> The starting point for every DevArts Lab repository and every client
> application the lab delivers. Cloudflare-native in production, local-first
> in development, agent-orchestrated across the full delivery lifecycle:
> research, design, development, and launch.

Status: living document | Audience: lab engineers, client developers, agents
License: TBD (intended to become a public repository)

---

## 1. Purpose

This framework defines the shared foundation that every project inherits:

- One architecture model that scales from a single landing page to a
  multi-client platform.
- One local development loop that behaves exactly like production.
- One agent orchestration layer that runs lab operations and assists client
  delivery end to end.
- One set of guardrails and data governance rules applied uniformly, so
  quality and compliance are properties of the platform, not of individual
  diligence.

Every new repository starts from this foundation rather than from scratch.

## 2. Design principles

| #   | Principle                                     | Consequence                                                                                                |
| --- | --------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| 1   | Same runtime locally and in production        | `wrangler dev` and Miniflare run real `workerd`; parity is a property, not a promise                       |
| 2   | Environment differences live in configuration | Bindings swap per environment in `wrangler.toml`; no code branches per environment                         |
| 3   | Durable state before agent memory             | Every decision, artifact, and step lives in D1, R2, or Durable Object storage, never only in agent context |
| 4   | Every action is auditable                     | A Tail Worker records agent and API activity; nothing writes state outside the audit path                  |
| 5   | Isolation is layered                          | Per-Worker dispatch, per-entity Durable Objects, isolated execution environments for generated code        |
| 6   | Smallest product that fits                    | Add a Cloudflare product only when it serves a concrete requirement                                        |
| 7   | Secrets never touch files in production       | `wrangler secret` and Secrets Store in prod; `.dev.vars` locally and gitignored                            |

## 3. Architecture map

```
┌─ Development (local) ───────────────────────────────────────────────┐
│ repo checkout (foundation scaffold)                                  │
│ ├── wrangler dev / Miniflare: workerd + local D1, KV, R2, DO, Queues │
│ ├── .dev.vars for local configuration                                │
│ └── optional: docker-agent (cagent) for local agent prototyping      │
└──────────────────────────────┬───────────────────────────────────────┘
                               │ git push / wrangler deploy
┌─ Production (Cloudflare) ────┴───────────────────────────────────────┐
│ Edge: Workers + Static Assets  (API, frontend, webhooks)             │
│ State: D1 · KV · R2 · Durable Objects                                │
│ Flow:  Queues · Workflows  (delivery pipeline, agent jobs)           │
│ Agents: Agents SDK · Workers AI · AI Gateway · Sandbox SDK           │
│ Clients: Workers for Platforms · Cloudflare for SaaS                 │
│ Ops:   Workers Builds · Previews · Tail Workers · Logpush            │
└──────────────────────────────────────────────────────────────────────┘
```

### Product map

| Concern           | Local development            | Production                                      |
| ----------------- | ---------------------------- | ----------------------------------------------- |
| Runtime           | `wrangler dev` (workerd)     | Workers on the edge                             |
| Frontend          | same code                    | Workers Static Assets / vinext (Next.js)        |
| Relational data   | Miniflare D1 (local SQLite)  | D1                                              |
| Key-value, config | Miniflare KV                 | KV                                              |
| Objects, files    | Miniflare R2                 | R2                                              |
| Coordination      | Miniflare Durable Objects    | Durable Objects                                 |
| Async jobs        | Miniflare Queues             | Queues, Workflows                               |
| Agent framework   | Agents SDK + local emulation | Agents SDK on Durable Objects                   |
| Code execution    | local toolchain              | Sandbox SDK (isolated Linux) / Dynamic Workers  |
| AI traffic        | AI Gateway (same endpoint)   | AI Gateway: rate limits, logging, cost controls |
| Secrets           | `.dev.vars`                  | `wrangler secret`, Secrets Store                |
| CI/CD             | `wrangler deploy`            | Workers Builds, preview per branch              |
| Observability     | local logs                   | Workers Logs, Tail Workers, Logpush             |

## 4. Delivery pipeline: research, design, development, launch

The lab's delivery lifecycle is modeled as a durable **Workflow** per project,
with each phase a checkpointed step:

| Phase       | Workflow step                                        | Produced artifact                         |
| ----------- | ---------------------------------------------------- | ----------------------------------------- |
| Research    | gather context, competitive scan, requirements draft | research dossier in R2 + D1 record        |
| Design      | architecture proposal, data model, UX direction      | design pack in R2, reviewed and versioned |
| Development | scaffold from this foundation, implement, preview    | preview deployments per branch            |
| Launch      | promote preview, DNS/custom domain, handover pack    | production release + client runbook       |

Each step emits events to the audit Tail Worker. Human or agent reviewers
approve phase transitions through defined gates (see Section 7).

## 5. Decision D1: client dispatch model

How client applications are dispatched and isolated on the platform.

| Option | Model                                                                          | Choose when                                                                                                   |
| ------ | ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------- |
| A      | **Workers for Platforms**: each client app is a Worker in a dispatch namespace | Clients receive real applications with per-client resource isolation, independent deploys, and custom domains |
| B      | **Single multi-tenant Worker + per-tenant Durable Object**                     | Standard-tier offerings, internal tools, and early-stage client work                                          |

**Default decision:** Option B for standard engagements; Option A once a
second flagship client proves the need. The two can coexist: WfP for flagship
delivery, tenant-DO model for everything else.

**Decision criteria:** client data separation requirements, independent
deploy schedules, per-client metering/billing needs, custom domains, and
whether clients run their own code.

## 6. Decision D2: agent state model

How agent work is made durable, resumable, and reviewable.

| Option | Model                                                       | Choose when                             |
| ------ | ----------------------------------------------------------- | --------------------------------------- |
| A      | **Durable Object per agent session + Workflow per process** | Always for production agent work        |
| B      | Stateless function calls with context passed each time      | Only for one-shot, read-only assistance |

**Default decision:** Option A. One Durable Object per agent session owns
mutable state; one Workflow per business process owns multi-step execution.
Every state transition is persisted before the next model call, so any agent
run can be resumed, replayed, or audited.

**Rules:**

- No step may mutate external state before writing its intent to durable
  storage (idempotency by construction).
- Retries re-read state rather than re-executing blindly.
- Long-running work checkpoints after every phase.

## 7. Guardrails

Guardrails are platform features, not conventions.

| Guardrail          | Mechanism                                                                                  | Applies to                                   |
| ------------------ | ------------------------------------------------------------------------------------------ | -------------------------------------------- |
| AI traffic control | AI Gateway: rate limits, spend caps, logging, provider failover                            | every model call                             |
| Code execution     | Sandbox SDK / Dynamic Workers: isolated Linux or isolate environments with egress controls | generated and third-party code paths         |
| Capability scoping | Agents receive tools via MCP with least-privilege scopes                                   | all agents                                   |
| Approval gates     | Workflow steps pause for human or policy-based approval at defined transitions             | phase changes, client-visible outputs, spend |
| Audit trail        | Tail Worker writes every tool call and state change to R2 + Logpush                        | everything                                   |
| Resource limits    | Per-DO and per-namespace quotas; Queues backpressure                                       | multi-tenant workloads                       |
| Secrets handling   | Secrets Store bindings; no secrets in sandbox environment variables                        | all environments                             |

## 8. Data governance

| Area              | Rule                                                                                                           |
| ----------------- | -------------------------------------------------------------------------------------------------------------- |
| Classification    | Every dataset carries a class: public, internal, client-confidential, or regulated                             |
| Client boundaries | Client data lives in per-client resources (dedicated Worker or DO namespace); no cross-client reads by default |
| Residency         | Regional and localization requirements use Data Localization Suite where contracted                            |
| Retention         | Retention periods defined per data class; deletion is a workflow step, not a script                            |
| Auditability      | Reads and writes to client-confidential data are audit-logged events                                           |
| Backups           | D1 point-in-time recovery + R2 versioning for artifacts; restore tested per launch gate                        |
| Secrets           | Never stored in code, CI files, or agent context; rotation through Secrets Store                               |

## 9. Optional: docker-agent (cagent) for local agent development

[cagent](https://github.com/docker/docker-agent) is Docker Engineering's agent
builder and runtime: YAML-defined agents with MCP tool support, running locally
on Docker.

**Where it fits:** development-time agent prototyping. Define and exercise
agent flows locally with cagent, then implement the production version on
Cloudflare Agents SDK + Durable Objects.

**Where it does not fit:** production. cagent is a local/desktop runtime;
Cloudflare Agents SDK is the production runtime. The two share agent design
concepts and MCP tooling but are different runtimes, so treat cagent output as
prototype artifacts, not deployable units.

**Adoption rule:** optional. Use it when agent flow design benefits from fast
local iteration; skip it when the Agents SDK emulator is sufficient.

## 10. Repository scaffold

Every new repository created from this foundation starts with:

```
<repo>/
├── wrangler.toml            # env.dev / env.preview / env.prod bindings
├── .dev.vars.example        # local configuration template (never real secrets)
├── src/                     # Worker entrypoint + domain modules
├── md/                      # this document set: plan, ADRs, runbooks
│   ├── MEGA-PLAN.md         # link to foundation
│   ├── PRE-DEVELOPMENT.md   # link to pre-development blueprint
│   └── adr/                 # architecture decision records
├── .github/workflows/       # Workers Builds wiring when applicable
└── AGENTS.md                # agent-facing conventions for the repo
```

Scaffold commands (target): a template repo or `npm create` style generator
that stamps out the above with the project's name and chosen dispatch model.

## 11. Phased roadmap

| Phase               | Deliverable                                                                      | Exit criteria                                           |
| ------------------- | -------------------------------------------------------------------------------- | ------------------------------------------------------- |
| 1. Spine            | One Worker + one Durable Object + Queues + D1 running dev-to-prod                | `wrangler dev` and deploy both green; preview URL works |
| 2. First client app | A real application delivered on the spine                                        | Client served on custom domain; audit log verified      |
| 3. Agent layer      | Agents SDK + AI Gateway + one Workflow (e.g., client intake to project scaffold) | Agent run resumable, audited, and within spend caps     |
| 4. Client dispatch  | Workers for Platforms for flagship clients                                       | Second client app deployed with per-client isolation    |
| 5. Public framework | This document set + scaffold generator published                                 | External developer can scaffold and deploy unaided      |

## 12. What this foundation deliberately excludes

- Vendor-neutral abstraction layers: bindings are already the portability
  layer; extra indirection adds cost without benefit.
- General-purpose containers for every workload: Containers and Sandbox SDK
  are used only where Workers primitives do not fit.
- Agent autonomy without audit: no phase transition, client-visible output,
  or spend happens outside the guardrail path.

## 13. Open questions

- Pricing model for per-client metering once Workers for Platforms is live.
- Which AI providers anchor the Agents SDK stack and their failover order.
- Whether client code execution (Sandbox SDK) is offered to clients or stays
  internal-only.
- Public license choice for the framework repository.

---

_Living document: revise after each phase exit. Decisions become ADRs in_
_`md/adr/` of implementing repositories._
