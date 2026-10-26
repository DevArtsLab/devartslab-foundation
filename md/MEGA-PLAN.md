# DevArts Lab Foundation: Research Mega Plan

> The research base and architectural charter for every DevArts Lab repository
> and every client application the lab delivers. Cloudflare-native in
> production, local-first in development, agent-orchestrated across the full
> delivery lifecycle: research, design, development, launch, operate.

| Field          | Value                                                                  |
| -------------- | ---------------------------------------------------------------------- |
| Status         | Living document, revision 3 (reuse and control edition)                |
| Audience       | Lab engineers, client developers, agents (human and machine readers)   |
| Scope          | Platform research, architecture, decisions, guardrails, governance     |
| Not in scope   | Code, configuration files, implementation guides (see PRE-DEVELOPMENT) |
| Facts verified | 2026-10-21 against developers.cloudflare.com (see Section 22)          |
| License        | TBD (open question Q4); intended to become a public repository         |
| Companion      | [PRE-DEVELOPMENT.md](PRE-DEVELOPMENT.md): what to produce before code  |

---

## Table of contents

0. How to use this document
1. Purpose and scope
2. Research method and fact decay policy
3. Design principles
4. Platform research: the Cloudflare product catalog by concern
5. Reference architecture and project tiers
6. Local development and environment parity model
7. Environment model: dev, preview, production
8. Delivery lifecycle as a durable Workflow
9. Foundation decisions D1 to D15
10. Guardrails
11. Data governance
12. Cost model
13. Limits and capacity cheat sheet
14. Threat model for agentic applications
15. Agent orchestration model
16. Repository scaffold (master layout)
17. Phased roadmap
18. Risks and mitigations
19. Deliberately excluded
20. Open questions
21. Glossary
22. Source register
23. Administrative management interface

- Appendix A: ADR template
- Appendix B: Audit event catalog (baseline)
- Appendix C: Project start checklist

Companion research documents in `md/`:

- [CONFIG-AND-SECRETS.md](CONFIG-AND-SECRETS.md): the centralized
  configuration, PII, and secret architecture across local, staging,
  production, and distributed products, plus the per-platform secret-store
  inventory. Expands D14.

---

## 0. How to use this document

- Starting a new project: read Sections 1, 3, 5, 9, 16, then run Appendix C
  and switch to [PRE-DEVELOPMENT.md](PRE-DEVELOPMENT.md).
- Choosing a product for a requirement: Section 4 (by concern) and Section 13
  (limits). Confirm against the live docs listed in Section 22 before relying
  on any number.
- Making an architecture decision: Section 9 holds the lab defaults; record
  project deviations as ADRs using Appendix A.
- Reviewing for compliance or security: Sections 10, 11, 14, Appendix B.
- Estimating cost: Section 12.
- Agents reading this file: treat Section 9 defaults as constraints, treat
  Section 13 numbers as hints that must be re-verified, and never write code
  from this document; it describes intent, not implementation.

Conventions: ASCII punctuation only; product names follow Cloudflare's
current naming (Section 21 maps older names); every quantitative claim has a
source row in Section 22.

## 1. Purpose and scope

This foundation defines the shared base that every DevArts Lab project
inherits, so that quality, compliance, and operability are properties of the
platform rather than of individual diligence.

It provides:

- One architecture model that scales from a single landing page to a
  multi-client agentic platform without a rewrite (Section 5 tiers).
- One local development loop that runs the same runtime as production.
- One agent orchestration layer that runs lab operations and assists client
  delivery end to end (Section 15).
- One set of guardrails and governance rules applied uniformly (Sections 10,
  11, 14).
- One repository layout and documentation set (Section 16).
- One cost and capacity model so proposals and estimates are grounded
  (Sections 12, 13).

It does not provide code, templates, or configuration. Those live in the
scaffold repository that implements this plan (roadmap Phase 5).

### 1.1 What "Cloudflare-native" means here

Production runs on Cloudflare's developer platform. The deciding factors
behind this choice, in order:

1. Runtime parity: `wrangler dev` and Miniflare execute the same `workerd`
   runtime that production uses, so local behavior is representative rather
   than approximated.
2. Bindings as the integration layer: storage, queues, AI, and secrets are
   reached through configuration-declared bindings instead of SDK clients and
   connection strings, which removes a class of environment drift.
3. A complete primitive set: compute, static assets, SQL, KV, object storage,
   coordination (Durable Objects), queues, durable workflows, agents,
   inference, gateway, sandboxing, multi-tenant dispatch, secrets, logs, and
   CI/CD exist in one account with one billing relationship.
4. Cost shape: a Workers Paid subscription starts at USD 5 per month with
   generous included usage and no egress or bandwidth charges (Section 12).

The trade-offs accepted with this choice are listed in Section 18.

### 1.2 Platform doctrine: defaults are revisable

Cloudflare-native is the lab's default for hosted production work. It is a
default, not a boundary. The lab is not restricted to Cloudflare, or to any
named platform, framework, or agent runtime, and this plan is the operating
system for choosing tools rather than a mandate for one stack.

Rules that keep flexibility honest:

- Every engagement selects tools on evidence, not habit. The deciding
  requirement is written down before the tool is named; this is the same
  pattern used throughout Section 4.
- Section 4 is the research catalog for the Cloudflare platform. When the
  lab adopts a second platform at decision grade (Neuro-SAN for self-hosted
  agent runtime, n8n for client workflow automation, Docker Compose for
  product distribution), it earns a parallel catalog section with the same
  discipline: verified facts, limits, cost shape, deciding requirements.
- A platform enters the plan through a defined path: living research
  ticket, then a decision record, then a catalog entry. It never enters by
  silent adoption.
- The delivery lifecycle in Section 8 is stack-agnostic. Research, Design,
  Development, Launch, and Operate phases, their artifacts, and their
  approval gates hold whether the runtime is Workers, containers, a user's
  machine, or a client-managed service.
- Deviation is cheap when it is recorded. An engagement that picks a
  non-default platform files one ADR naming the deciding requirement and
  the exit or fallback path; nothing else changes.
- The discipline that does not flex: Linear is the system of record,
  secrets never touch repositories, documents are the interface between
  humans and agents, and nothing falls back silently (Principle 13). These
  hold under any stack.

The result is one SDLC with a preferred spine, not a walled garden: the
plan navigates to the best tool for the requirement at hand and records
why, so the next engagement inherits the research rather than repeating it.

## 2. Research method and fact decay policy

### 2.1 Method

- Primary sources are the official Cloudflare developer docs, read in their
  markdown form (`index.md` endpoints) on the verification date.
- Secondary sources are the Cloudflare-maintained agent skills bundle
  (`cloudflare/skills`), which encodes product selection guidance and
  anti-patterns, and the Agents SDK repository docs.
- No vendor blog posts, third-party tutorials, or model memory were used for
  quantitative claims.
- Each claim that can change (limit, price, plan availability, beta status)
  carries a row in the Section 22 source register.

### 2.2 Decay policy

Cloudflare ships platform changes weekly. The following classes decay at
different rates and must be re-verified on the stated cadence:

| Class                          | Examples                                                                  | Re-verify                                 |
| ------------------------------ | ------------------------------------------------------------------------- | ----------------------------------------- |
| Prices and included allotments | Workers requests and CPU-ms, Logs, Logpush                                | Quarterly, and before any client estimate |
| Hard limits                    | D1 10 GB, Queues 128 KB, Workflows step result                            | Quarterly                                 |
| Beta and preview status        | Secrets Store, Sandbox SDK, Containers policies, `cf` CLI, Artifacts      | Monthly                                   |
| Recommended defaults           | `wrangler.jsonc`, vinext over OpenNext, Previews over Version URL aliases | Quarterly                                 |
| Product names                  | Browser Rendering to Browser Run, Pipelines to Basin, Sandbox naming      | On encounter                              |
| Deprecations                   | AI Gateway Unified API for single-model calls, Pages for new projects     | Quarterly                                 |

Verification is a Section 22 update plus a note in the revision log at the
end of this file. A refinement pass should also be logged in Linear.

## 3. Design principles

Each principle states the rule, the reason it exists, the concrete
consequence for projects, and how compliance is checked.

| #   | Principle                                             | Why                                                                                        | Consequence                                                                                                                                  | How checked                                                                  |
| --- | ----------------------------------------------------- | ------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| 1   | Same runtime locally and in production                | Most "works on my machine" defects are runtime divergence                                  | `wrangler dev` or the Vite plugin on `workerd`; local D1, KV, R2, DO, Queues emulated by Miniflare                                           | Phase 1 exit: identical test suite passes locally and in Preview             |
| 2   | Environment differences live in configuration         | Branching code on environment is untestable and leaks                                      | Bindings swap per environment and per Preview in `wrangler.jsonc`; no `if (prod)` in application code                                        | Code search for environment conditionals in review                           |
| 3   | Durable state before agent memory                     | Agent context is ephemeral and unauditable                                                 | Every decision, artifact, and step lives in D1, R2, DO storage, or Workflow state, never only in a prompt                                    | Resumability test: kill and resume an agent run                              |
| 4   | Every action is auditable                             | Compliance and incident response need a complete trail                                     | Tail Worker plus Workers Logs capture tool calls and state changes; Appendix B event catalog                                                 | Audit log verification at Phase 2 and 3 exits                                |
| 5   | Isolation is layered                                  | One isolation mechanism is never enough for untrusted code and multi-tenant data           | Per-Worker dispatch, per-entity Durable Objects, sandboxed execution for generated code                                                      | Threat model review (Section 14) per project                                 |
| 6   | Smallest product set that fits                        | Each added product is a cost, a limit surface, and a learning burden                       | Add a Cloudflare product only against a named requirement; prefer Workers Cache before KV caching                                            | ADR required for any product outside the tier default                        |
| 7   | Secrets never touch files in production               | File-borne secrets leak through VCS, CI logs, and agent context                            | `wrangler secret` or Secrets Store bindings in prod; `.dev.vars` locally and gitignored                                                      | Pre-push secret scan (lab `devartslab-code-review` skill)                    |
| 8   | Prefer bindings over REST calls                       | REST adds auth, latency, and rate-limit exposure (1200 requests per 5 minutes per account) | Reach D1, KV, R2, Queues, AI, Secrets through bindings; REST only for control-plane automation                                               | Review flag per `workers-best-practices` anti-patterns                       |
| 9   | Protect against runaway spend                         | Agents and public endpoints can create denial-of-wallet                                    | `limits.cpu_ms` set deliberately; AI Gateway spend limits; Queues backpressure; WAF rate limits                                              | Cost guardrail checklist at launch gate                                      |
| 10  | Documents are the interface between humans and agents | Agents act on what is written; undocumented intent is lost                                 | `md/` document set in every repo, this plan as root; ADRs for every deviation                                                                | REPO_INDEX and ADR presence in repository audit                              |
| 11  | Build once, reuse everywhere                          | Work rebuilt per project is work paid for twice and maintained in parallel                 | Libraries, packages, UI components, and agent rules ship through the lab registry (D11); per-app copies need an ADR                          | Duplication review: a second implementation of the same concern fails review |
| 12  | Identity is a variable, never a literal               | A business name, domain, or brand hardcoded into the estate makes a rename a rewrite       | One identity manifest per product; brand strings resolve from it (D12)                                                                       | CI check for forbidden literals outside the manifest                         |
| 13  | No silent fallback                                    | A degradation nobody can see is a defect that persists                                     | Every automatic switch (provider, cache, limit, feature) surfaces in the administrative interface and emits an audit event (Section 23, D15) | Fallback test: trigger each one and observe it in the interface              |

## 4. Platform research: the Cloudflare product catalog by concern

This section is the research core. Each concern lists the products that
serve it, the deciding requirement between them, maturity and plan notes,
and the lab default. Numbers are summarized in Section 13 and sourced in
Section 22.

### 4.1 Compute and request handling

| Product               | Role                                                           | Notes (verified 2026-10-26)                                                                                                                         |
| --------------------- | -------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| Workers               | HTTP handlers, APIs, webhooks, scheduled jobs, queue consumers | 128 MB memory per isolate; CPU 30 s default, configurable to 5 min on Paid; 10,000 subrequests per request on Paid; 500 Workers per account on Paid |
| Workers Static Assets | Serve frontend files from the same Worker                      | Requests to static assets are free and unlimited; 100,000 files per version on Paid; 25 MiB per file                                                |
| Cron Triggers         | Scheduled invocations                                          | 250 per account on Paid; 15 min wall time; combine with Queues or Workflows for the work itself                                                     |
| Smart Placement       | Run the Worker near its backends rather than the user          | Use when a Worker makes several round trips to one region (Hyperdrive, external API)                                                                |
| Snippets and Rules    | Small edge transforms without a Worker                         | Prefer Rules for redirects and header rewrites; keep application logic in Workers                                                                   |
| Pages                 | Legacy static hosting                                          | Not for new projects; Workers with Static Assets is the recommendation; preserve existing Pages during maintenance only                             |

Deciding requirement: anything that is application logic runs in a Worker.
Static files ride along as Static Assets. Pages is excluded for new work.

### 4.2 Storage and data

Cloudflare publishes a storage selection guide; the lab condenses it into
the following decision matrix.

| Need                                                      | Product                                   | Deciding requirement                                                                             | Key constraints                                                                                                                                                |
| --------------------------------------------------------- | ----------------------------------------- | ------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Relational application data, SQL, cross-entity queries    | D1                                        | Read-heavy, globally read, lightweight RDBMS without ops                                         | 10 GB per database (not raisable); single-threaded per database; 50,000 databases per account on Paid (raisable by request); read replication available; Time Travel 30 days on Paid |
| Per-entity state with strict consistency and coordination | Durable Objects (SQLite)                  | Coordination atom per tenant, session, room, or agent; transactional storage; WebSockets; alarms | 10 GB per object; unlimited objects; soft limit about 1,000 requests per second per object; 500 classes per account on Paid                                    |
| Configuration, routing metadata, sessions, feature data   | Workers KV                                | Read-heavy, eventually consistent, hot keys served from cache in roughly 0.5 to 10 ms            | 1 write per second per key; not for counters or anything needing immediate consistency                                                                         |
| Files, uploads, artifacts, exports, logs                  | R2                                        | Object storage, S3-compatible, no egress fees, per-object strong consistency                     | Pair with D1 for searchable metadata; versioning available for artifact retention                                                                              |
| Existing Postgres or MySQL                                | Hyperdrive                                | Keep an existing database or need single databases above D1's ceiling                            | Connection pooling and caching from Workers; works with Neon, Supabase, PlanetScale                                                                            |
| Semantic search, embeddings                               | Vectorize                                 | Custom retrieval with control over embeddings and indexes                                        | Pair with Workers AI embeddings; use AI Search when a managed RAG pipeline is enough                                                                           |
| Custom usage metrics, tenant metering                     | Analytics Engine                          | High-cardinality time series written from Workers, queried by SQL                                | Natural fit for per-client metering (open question Q1)                                                                                                         |
| Durable event log with independent readers                | K2 Streams                                | Replayable events, multiple consumers, archival                                                  | Newer product; verify availability before promising it                                                                                                         |
| Streaming ingestion into analytics tables                 | Basin Pipelines, Basin Catalog, Basin SQL | Clickstream, telemetry into Iceberg tables in R2                                                 | Formerly Pipelines, R2 Data Catalog, R2 SQL                                                                                                                    |
| Versioned file trees, agent checkpoints                   | Artifacts                                 | Git-compatible versioned storage                                                                 | Closed beta; do not depend on it in client commitments                                                                                                         |

Lab guidance:

- Start every project with D1 for relational data and one Durable Object
  class per coordination atom. Add R2 the moment files or large artifacts
  appear. Add KV only for read-heavy configuration or sessions.
- Design D1 schemas for horizontal split from day one: per-tenant or
  per-domain databases are cheap (pricing is per query and storage, not per
  database), and 10 GB per database is a hard ceiling.
- Use `batch()` for D1 throughput; per-request overhead, not query time,
  is usually the throughput limiter.
- Treat Durable Object storage as the system of record for anything an
  agent session mutates (Section 9, D2), and D1 as the system of record for
  cross-entity reporting.

### 4.3 Coordination, asynchronous work, and durable execution

| Product         | Role                                                               | When to choose                                                                                        | Constraints                                                                                                                                                                                                                                      |
| --------------- | ------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Durable Objects | Single-threaded coordination with transactional storage and alarms | Shared state per entity, WebSockets, ordering, scheduled per-entity work                              | One alarm per object (set replaces existing); alarm handler 15 min wall; do not build one global object                                                                                                                                          |
| Queues          | Decouple producers and consumers; absorb bursts                    | Background jobs, fan-out, retries with backoff, dead-letter handling                                  | 128 KB message; 100 retries; batch up to 100; 5,000 messages per second per queue; retention up to 14 days; 25 GB backlog; consumer 15 min wall; 250 concurrent push consumers                                                                   |
| Workflows       | Durable multi-step execution with retries, sleeps, events          | Business processes that must resume after failure: delivery pipeline, onboarding, approvals           | 1 MiB per step result and event payload; 1 GB persisted state per instance on Paid; 10,000 steps default (25,000 max); `step.sleep` up to 365 days; completed state retained 30 days on Paid; not deployable in Workers for Platforms namespaces |
| Agents SDK      | Durable Object based agent runtime                                 | Stateful agents with RPC, scheduling, built-in queue, retries, `AgentWorkflow`, MCP client and server | Requires `nodejs_compat`; each agent class is its own DO binding plus SQLite migration                                                                                                                                                           |
| Cron Triggers   | Time-based entry points                                            | Scheduled kick-off of Queues or Workflows                                                             | Keep the cron handler thin; the work belongs to a Workflow or queue consumer                                                                                                                                                                     |

Design rule: Queues for "do this later, maybe many times"; Workflows for
"do these steps in order and survive failure"; Durable Objects for "keep
this entity consistent"; Agents SDK when the entity is an agent session.

Workflows detail that shapes the lab's lifecycle model (Section 8):

- Steps are the persistence and retry boundary. A step's return value is
  stored and replayed; side effects inside a step can run twice, so external
  writes must be idempotent.
- Waiting instances (sleeping, waiting for an event, awaiting retry) do not
  count toward concurrency, so long approval waits are cheap.
- Large outputs belong in R2 with a reference returned from the step.
- Completed instance state is retained 30 days; durable records must be
  exported to D1 or R2 before expiry (Section 11).

### 4.4 AI layer: inference, gateway, agents, tools

| Product        | Role                                                 | Capabilities relied on by this plan                                                                                                                                                                                                                                                                                                                                                       |
| -------------- | ---------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| AI Gateway     | Control plane for every model call                   | Caching, rate limiting, spend limits, Guardrails (content policy), DLP (sensitive data detection), BYOK provider keys, Unified Billing with Zero Data Retention option, logging with payload storage controls, Logpush, custom metadata, custom costs, dynamic routing (fallbacks, conditional routes, traffic splits)                                                                    |
| Workers AI     | Managed inference on Cloudflare GPUs                 | Text generation, embeddings, image, speech; bound through `ai` binding; route through AI Gateway                                                                                                                                                                                                                                                                                          |
| Agents SDK     | Agent runtime on Durable Objects                     | Persistent SQLite-backed state synced to clients, `@callable` RPC, scheduling (delay, cron, interval), `AgentWorkflow`, `runFiber` and `stash` for durable execution, built-in FIFO queue, retries with backoff, MCP client and server, email channel, chat agents with resumable streaming, human-in-the-loop approvals, diagnostics-channel observability, webhooks, push notifications |
| MCP on Workers | Tool exposure to agents                              | `createMcpHandler` servers with Streamable HTTP or SSE transport; OAuth-secured servers; least-privilege tool scopes                                                                                                                                                                                                                                                                      |
| AI Search      | Managed RAG                                          | Use before building Vectorize pipelines when the requirement is "answers over our content"                                                                                                                                                                                                                                                                                                |
| Flagship       | Feature flags with targeting and percentage rollouts | Gate agent capabilities and client-visible features independently of deploys                                                                                                                                                                                                                                                                                                              |
| Browser Run    | Headless browser automation                          | Research-phase scraping, screenshots, rendered extraction; formerly Browser Rendering                                                                                                                                                                                                                                                                                                     |

Notes that affect design:

- AI Gateway's legacy Unified API is deprecated for single-model calls but
  still required for dynamic routes. Projects needing failover must plan for
  this split.
- Gateway authentication and upstream provider credentials are separate
  concerns; provider keys belong in Secrets Store or BYOK, never in Worker
  variables.
- Agents SDK marks Voice, Browser tools, Think, and Code Mode as
  experimental; treat them as research spikes, not client commitments.

### 4.5 Executing generated or untrusted code

Cloudflare offers two sandbox environments, both reached through a Worker
(available on Workers Paid):

| Environment              | What it is                                                                                                           | Choose when                                                                                                   | Controls                                                                                        |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| Dynamic Workers          | A Worker spawned at runtime from supplied JavaScript, Python, or Wasm via a Worker Loader binding (`worker_loaders`) | Code only needs to call methods your Worker exposes; one-shot evaluation (Code Mode) or cached-by-id modules  | `globalOutbound: null` blocks all egress; explicit binding allow-list; isolate-level separation |
| Containers (Sandbox SDK) | A microVM with its own kernel and network, started from an image by a Durable Object                                 | Code needs Linux, a shell, a filesystem, long-running processes, or coding agents (Claude Code, Codex, Devin) | `enableInternet: false`; credentials stay in the Worker; R2 mounts for workspaces; snapshots    |

Rules:

- Generated code never runs inline in a request handler.
- Default to Dynamic Workers; escalate to Containers only when Linux is a
  requirement.
- The `@cloudflare/sandbox` package line matters: 0.x and the current SDK
  differ; check the installed version before reading docs.

### 4.6 Multi-tenancy and client dispatch

| Product                    | Role                                                                            | Facts that drive the decision                                                                                                                                                                                                                                                                                              |
| -------------------------- | ------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Workers for Platforms      | Dispatch namespaces holding unlimited customer Workers behind a dispatch Worker | Untrusted mode by default (no `request.cf`, per-Worker cache); custom CPU and subrequest limits per customer; optional Outbound Worker for egress control (does not intercept DO or mTLS fetches; disables TCP `connect()`); up to 8 tags per Worker; no gradual deployments; Workflows cannot be deployed into namespaces |
| Cloudflare for SaaS        | Custom hostnames and TLS for client domains                                     | Pairs with WfP or with a single multi-tenant Worker; check apex-domain plan requirements                                                                                                                                                                                                                                   |
| Durable Objects per tenant | Isolation inside one Worker                                                     | Per-tenant SQLite state with strict serializability; simplest model for standard engagements                                                                                                                                                                                                                               |
| Previews                   | Isolation per branch, not per tenant                                            | Not a tenancy mechanism; listed here to avoid confusion                                                                                                                                                                                                                                                                    |

The Workflows constraint is decisive: if a client application's business
processes are modeled as Workflows, they must live in the lab's own Workers,
not inside a dispatch namespace. See Section 9, D1 for the resulting hybrid.

### 4.7 Delivery tooling: configuration, CI/CD, previews, releases

| Tool or feature          | Role                                          | Facts                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| ------------------------ | --------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Wrangler                 | CLI for dev, deploy, resources, secrets       | `wrangler.jsonc` recommended for new projects; some newer features require JSON config; compatibility date should be today's date for new Workers                                                                                                                                                                                                                                                                                                           |
| Wrangler environments    | Named configurations under `env.<name>`       | Most keys inherit; bindings do not and must be declared per environment; with the Vite plugin, select via `CLOUDFLARE_ENV` at build time, not deploy time                                                                                                                                                                                                                                                                                                   |
| Previews                 | Isolated branch environments under one Worker | Requires Wrangler 4.135.0 or later; own variables, secrets, bindings, URL, observability; DO namespace and container instances auto-provisioned per Preview; KV, D1, R2 isolated only when bound to separate resources; Workflow bindings reuse existing Workflows; service bindings call production; routes and crons target production; queue consumers cannot target a Preview; public by default (protect with Access); 500 Previews per Worker on Paid |
| Versions and deployments | Upload, gradual rollout, rollback             | Rollback restores code only; connected resources and their data are not rolled back                                                                                                                                                                                                                                                                                                                                                                         |
| Workers Builds           | Git-connected build and deploy                | Default CI path; external CI (GitHub Actions) remains valid when repos need custom pipelines                                                                                                                                                                                                                                                                                                                                                                |
| C3 and framework guides  | Project scaffolding                           | `npm create cloudflare` and framework templates; vinext for Next.js                                                                                                                                                                                                                                                                                                                                                                                         |
| `cf` CLI                 | Next-generation CLI (beta)                    | Uses `cloudflare.config.ts`; do not mix with Wrangler in one project; watch item, not a lab default yet                                                                                                                                                                                                                                                                                                                                                     |
| Terraform or Pulumi      | Infrastructure as code for account resources  | Use for zones, DNS, Access policies, and namespaces that are not expressed in `wrangler.jsonc`                                                                                                                                                                                                                                                                                                                                                              |

Secrets handling facts that affect process:

- `wrangler secret put` and `secret delete` create and deploy a new version
  immediately; use the `wrangler versions secret` workflow when a secret
  change must be staged with code.
- Local secret files are not uploaded; `.dev.vars` is local only.
- The `wrangler login` OAuth flow does not support granular scopes; CI and
  agents use account-owned API tokens with the narrowest role.

### 4.8 Observability

| Signal or tool                  | Role                                                             | Facts                                                                                                                                                                                  |
| ------------------------------- | ---------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Workers Logs                    | Stored, searchable invocation logs                               | Paid: 20 million events per month included, then USD 0.60 per million; 7-day retention. Free: 200,000 per day, 3 days. Pricing moves to Cloudflare Observability pricing on 2026-12-01 |
| Workers Traces                  | Request flow and dependency latency                              | Enable `observability.traces.enabled` explicitly; the top-level flag alone does not enable traces                                                                                      |
| Real-time logs, `wrangler tail` | Live debugging                                                   | Not a persistence mechanism                                                                                                                                                            |
| Tail Workers                    | Custom processing of trace events (filter, redact, route)        | Receives events from dynamic dispatch and user Workers too; use for the lab audit trail (Appendix B)                                                                                   |
| Workers Logpush                 | Deliver trace events to external storage                         | Paid only; 10 million requests per month included, then USD 0.05 per million; charged after filtering and sampling                                                                     |
| OpenTelemetry export            | Push logs and traces to an OTel destination                      | Check first whether this replaces a custom Tail Worker                                                                                                                                 |
| Analytics Engine                | Custom metrics and per-tenant usage                              | SQL-queryable; use for metering and spend dashboards                                                                                                                                   |
| Agents SDK diagnostics          | `diagnostics_channel` events for state, RPC, schedule, lifecycle | Feed into the same audit path                                                                                                                                                          |

### 4.9 Security and access

| Product or feature                             | Role                                                       | Notes                                                                                                                                                           |
| ---------------------------------------------- | ---------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Worker secrets                                 | Per-Worker encrypted values                                | Default for credentials owned by one Worker                                                                                                                     |
| Secrets Store                                  | Account-level secrets shared across Workers and AI Gateway | Binding type `secrets_store_secrets`; binding requires Super Administrator or Secrets Store Deployer role; each secret carries scopes (`workers`, `ai-gateway`) |
| Cloudflare Access                              | Identity-aware access to apps, Previews, internal tools    | Protect every Preview URL and every internal lab surface                                                                                                        |
| Turnstile                                      | Bot verification for forms                                 | Server-side Siteverify required                                                                                                                                 |
| WAF, rate limiting, Bot Management, API Shield | Edge protection                                            | Rate limit public agent endpoints; schema-validate APIs                                                                                                         |
| Roles, scopes, API tokens                      | Least privilege for humans, CI, and agents                 | Scope tokens to the Worker or product; never paste token values into chat or logs                                                                               |
| Workers security model                         | Isolate-based separation                                   | The basis for Dynamic Workers; read before trusting any sandbox claim                                                                                           |

### 4.10 Data governance products

| Capability                      | Product or feature                                                                      | Facts                                                                                                             |
| ------------------------------- | --------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| Point-in-time recovery, SQL     | D1 Time Travel                                                                          | Always on; restore to any minute in the last 30 days on Paid (7 on Free); 10 restores per 10 minutes per database |
| Object versioning               | R2 versioning                                                                           | Retain prior versions of artifacts and exports                                                                    |
| Regional processing and storage | Data Localization Suite                                                                 | Evaluate against actual contractual requirements; enterprise feature set                                          |
| Prompt and response privacy     | AI Gateway DLP, Guardrails, logging controls, Zero Data Retention under Unified Billing | Decide what may be stored before enabling payload logging                                                         |
| Log retention                   | Workers Logs 7 days; Logpush to R2 for longer                                           | Retention beyond 7 days requires export                                                                           |
| Workflow history                | 30 days completed state retention on Paid                                               | Export durable records before expiry                                                                              |

### 4.11 Frontend

| Need                   | Product                   | Notes                                                                                                                  |
| ---------------------- | ------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Static site or SPA     | Workers Static Assets     | Free asset requests; SPA not-found handling available                                                                  |
| React or framework app | Cloudflare Vite plugin    | Runs the Worker in `workerd` during Vite dev; generates output config at build                                         |
| Next.js                | vinext                    | Recommended over OpenNext for new projects; reimplements the Next.js API surface on Vite; check feature parity per app |
| Images, video          | Cloudflare Images, Stream | Only when the requirement exists (Principle 6)                                                                         |

### 4.12 Communications

| Need                       | Product         | Notes                                                        |
| -------------------------- | --------------- | ------------------------------------------------------------ |
| Transactional email        | Email Service   | Application-generated messages                               |
| Inbound routing            | Email Routing   | Domain addresses to mailboxes                                |
| Inbound processing in code | Email Workers   | Agents SDK has an email channel with secure reply resolution |
| Web push                   | Agents SDK push | VAPID-based from agents                                      |

## 5. Reference architecture and project tiers

### 5.1 Architecture map

```text
+-- Development (local) -------------------------------------------------+
| repository checkout (foundation scaffold, Section 16)                   |
|   wrangler dev or Vite plugin: workerd + local D1, KV, R2, DO, Queues   |
|   .dev.vars for local configuration (gitignored)                        |
|   optional: cagent for agent flow prototyping (Section 15.4)            |
+------------------------------------+------------------------------------+
                                     | git push -> Workers Builds -> Preview per branch
                                     | merge to main -> production deploy
+-- Production (Cloudflare) ----------+------------------------------------+
| Edge      Workers + Static Assets (API, frontend, webhooks)             |
| State     D1 | KV | R2 | Durable Objects (SQLite)                       |
| Flow      Queues | Workflows | Cron Triggers                            |
| Agents    Agents SDK | Workers AI | AI Gateway | Dynamic Workers | Containers |
| Clients   Durable Object per tenant | Workers for Platforms | Cloudflare for SaaS |
| Ops       Workers Builds | Previews | Workers Logs and Traces | Tail Worker | Logpush | Analytics Engine |
| Security  Access | Secrets Store | WAF and rate limiting | Turnstile    |
+-------------------------------------------------------------------------+
```

### 5.2 Request and event paths

- Synchronous user request: Edge Worker authenticates, routes to a tenant
  Durable Object or D1 query, returns; static files bypass the Worker.
- Asynchronous job: Worker enqueues to Queues; consumer Worker processes in
  batches; failures retry with backoff; dead letters land in a second queue.
- Business process: Worker or agent starts a Workflow instance; steps
  checkpoint; approvals pause the instance via `waitForEvent`; completion
  writes a durable record to D1 and artifacts to R2.
- Agent session: client connects to an Agents SDK Durable Object; state
  syncs over WebSocket; tool calls go through MCP with scoped tokens; model
  calls go through AI Gateway; generated code executes in a Dynamic Worker.
- Audit: every Worker in the account has the lab Tail Worker attached; it
  redacts, filters, and writes to R2 plus Analytics Engine; Logpush delivers
  raw trace events for retention.

### 5.3 Project tiers

Tiers keep Principle 6 honest: a tier fixes the default product set, and
anything beyond the tier needs an ADR.

| Tier | Name                  | Typical project                                   | Default product set                                                                                                              | Not included by default             |
| ---- | --------------------- | ------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------- |
| T0   | Static or landing     | Marketing site, docs site, portfolio              | Workers + Static Assets, Workers Logs, Access for previews                                                                       | Any storage, any AI                 |
| T1   | Application           | Form-backed site, dashboard, small SaaS           | T0 plus D1, KV (config), R2 (uploads), one DO class if coordination exists, Queues for background jobs, Turnstile, Email Service | Workflows, AI, WfP                  |
| T2   | Agentic application   | Assistant, automation, internal agent tool        | T1 plus Agents SDK, AI Gateway, Workers AI or BYOK providers, Workflows, Dynamic Workers, Tail Worker audit, Analytics Engine    | Containers, WfP                     |
| T3   | Multi-client platform | Flagship client delivery with independent deploys | T2 plus Workers for Platforms, Cloudflare for SaaS, Outbound Worker, per-client metering, Secrets Store                          | Nothing excluded; everything by ADR |

Lab operations itself is a T2 system that grows into T3 once it dispatches
client applications.

## 6. Local development and environment parity model

### 6.1 What parity means and does not mean

- Runtime parity holds: local `workerd` is the production runtime.
- Data parity is simulated: Miniflare emulates D1, KV, R2, Durable Objects,
  Queues, and Workflows locally with local persistence. Semantics match;
  scale and latency do not.
- Network parity is partial: AI Gateway, external providers, and Access are
  real network services. Remote bindings let a local Worker talk to real
  resources; this is powerful and dangerous (writes go to real data).
- Policy parity is absent locally: WAF, rate limiting, Access, and Turnstile
  do not run in `wrangler dev`. They are verified in Preview.

### 6.2 Local rules

1. `.dev.vars` holds local configuration and test credentials; it is
   gitignored and never contains production secrets. A committed
   `.dev.vars.example` documents required keys with placeholder values.
2. Local data persists in the project's Wrangler state directory and is
   disposable. Seed scripts recreate it (implementation lives in the
   scaffold, not here).
3. Remote bindings are off by default; enabling one is a deliberate,
   documented act and never targets production resources.
4. Tests run with the Workers Vitest integration against the same runtime;
   Durable Object and Workflow tests use the official test APIs.
5. Compatibility date is pinned per project and advanced deliberately with a
   test run.

### 6.3 Why not Docker for the application

Containers add a second runtime with its own drift. The Workers runtime is
already reproducible locally. Docker is reserved for cagent prototyping
(Section 15.4) and for Containers-based sandboxes, which are a production
product in their own right.

## 7. Environment model: dev, preview, production

| Environment | Mechanism                                | Resources                                                                                                                 | Access                            | Who deploys                             |
| ----------- | ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- | --------------------------------- | --------------------------------------- |
| dev         | `wrangler dev` on the developer machine  | Local Miniflare state; remote bindings off                                                                                | Localhost                         | Developer or agent                      |
| preview     | Workers Previews, one per branch or PR   | Preview-specific DO namespaces and containers automatically; separate D1, KV, R2 bound explicitly in the `previews` block | Public URL; must be behind Access | Workers Builds on push                  |
| prod        | Production deployment of the same Worker | Production bindings                                                                                                       | Public, custom domain             | Workers Builds on merge; humans approve |

Rules derived from the Previews facts in Section 4.7:

- Never bind a Preview to a production D1, KV, or R2 resource. Isolation is
  only real when the Preview block declares its own resources.
- Workflows are shared across Previews; a Preview that starts a Workflow
  starts it in the real Workflow namespace. Design Workflow instance IDs to
  carry an environment prefix so Preview runs are identifiable and purgeable.
- Queue consumers cannot target a Preview; test consumers locally and in
  production with feature flags (Flagship), not in Previews.
- Service bindings from a Preview call production; a Preview of a frontend
  Worker will talk to the production API Worker unless the API also exposes a
  Preview and the frontend is pointed at it explicitly.
- Preview URLs on `workers.dev` carry `X-Robots-Tag: noindex`; custom-domain
  Previews need Access to stay unindexed and private.
- Wrangler environments (`env.<name>`) remain the mechanism for persistent,
  separately deployed Workers such as a long-lived staging. Previews are for
  branches. The lab uses Previews for branches and does not run a permanent
  staging unless a client contract requires one.

## 8. Delivery lifecycle as a durable Workflow

The lab models each project engagement as one Workflow instance with five
phases. Each phase is a step group that checkpoints its outputs and pauses
on a `waitForEvent` approval gate.

| Phase       | Inputs                       | Work (human and agent)                                                      | Produced artifacts (durable)                                                                  | Gate to exit                                                    |
| ----------- | ---------------------------- | --------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| Research    | Intake brief, client context | Context gathering, competitive scan, requirements draft, risk register      | Research dossier in R2; D1 project record; Linear issue tree                                  | Client and lab lead approve problem statement and scope         |
| Design      | Dossier                      | Architecture proposal, data model, tier selection, ADR drafts, UX direction | Design pack in R2 (versioned); ADRs D1 to D10 instantiated; PRE-DEVELOPMENT exit criteria met | Architecture review passes; cost estimate within budget         |
| Development | Design pack                  | Scaffold from foundation, implement, test, preview per branch               | Preview deployments; test reports; audit log samples                                          | All exit criteria of PRE-DEVELOPMENT Section 8; security review |
| Launch      | Approved preview             | Promote to production, DNS and custom domain, Access policies, runbook      | Production release record; client runbook; restore test evidence                              | Launch checklist complete; monitoring live; rollback rehearsed  |
| Operate     | Production                   | Monitoring, cost review, dependency updates, quarterly fact re-verification | Monthly operations report in R2 and D1                                                        | Continuous; feeds back into Research for the next engagement    |

Mechanics:

- The Workflow instance ID is the project slug plus engagement number, so
  any agent or human can locate state deterministically.
- Each step writes its intent to D1 before performing external side effects
  (idempotency by construction; Section 9, D2).
- Approval events carry the approver identity from Access and are written
  to the audit trail (Appendix B, `gate.approved`).
- Artifacts over 1 MiB are stored in R2 with the key returned by the step.
- Because completed Workflow state is retained for 30 days, the final step
  exports a project summary record to D1 and the full step log to R2.

## 9. Foundation decisions D1 to D15

Each decision has a default, the alternatives, the criteria that would
change it, and the facts that drove it. Projects instantiate these as ADRs
(Appendix A) and record deviations.

### D1. Client dispatch model

| Option | Model                                                                             | Choose when                                                                                                   |
| ------ | --------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| A      | Workers for Platforms: each client app is a Worker in a dispatch namespace        | Clients need independent deploys, per-client CPU and subrequest limits, custom domains, or run their own code |
| B      | Single multi-tenant Worker with a Durable Object per tenant                       | Standard engagements, internal tools, early-stage client work                                                 |
| C      | Hybrid: WfP for client-facing Workers, lab-owned Workers for Workflows and agents | Any time a WfP client needs durable processes, because Workflows cannot run inside dispatch namespaces        |

Default: B for standard engagements; C once a second flagship client proves
the need. Pure A is not an option for agentic clients because of the
Workflows constraint. Decision criteria: data separation requirements,
deploy independence, metering and billing, custom domains, whether clients
run their own code, whether the client needs durable processes.

Consequences of C: the dispatch Worker and all Workflows live in the lab's
account Workers; client Workers call back into lab services through service
bindings or authenticated HTTP; the Outbound Worker governs client egress
but does not intercept Durable Object or mTLS fetches.

### D2. Agent state model

| Option | Model                                                      | Choose when                         |
| ------ | ---------------------------------------------------------- | ----------------------------------- |
| A      | Durable Object per agent session plus Workflow per process | Always for production agent work    |
| B      | Stateless calls with context passed each time              | Only one-shot, read-only assistance |

Default: A. One Agents SDK Durable Object per session owns mutable state in
SQLite; one Workflow per business process owns multi-step execution. Rules:
no step mutates external state before writing its intent to durable storage;
retries re-read state rather than re-executing blindly; long-running work
checkpoints after every phase; `runFiber` and `stash` are used for work that
must survive Durable Object eviction.

### D3. Storage selection

Default: D1 for relational and reporting data; Durable Object SQLite for
per-entity state; R2 for files and artifacts; KV only for read-heavy
configuration. Hyperdrive only when a client brings an existing database.
Vectorize only after AI Search is shown insufficient. Change triggers: a
single dataset projected above 10 GB (split D1 or use Hyperdrive), write
rates above 1 per second per key (KV is wrong), need for strict ordering
(Durable Object).

### D4. Configuration format and tooling

Default: `wrangler.jsonc` with environments declared per binding; Wrangler
pinned per project; compatibility date set at project creation. `cf` CLI is
a watch item and not adopted until stable. Terraform for account-level
resources not expressible in Wrangler.

### D5. Frontend stack

Default: Workers Static Assets with the Cloudflare Vite plugin. Next.js
projects use vinext. OpenNext is maintained only where it already exists.
Pages is not used for new work.

### D6. Code execution isolation

Default: Dynamic Workers with egress blocked and an explicit binding
allow-list. Containers only when Linux, shell, filesystem, or long-running
processes are required. Never inline execution of generated code.

### D7. Observability pipeline

Default: Workers Logs and Traces enabled on every Worker; one lab Tail
Worker attached to every Worker for the audit trail (Appendix B); Logpush to
R2 for retention beyond 7 days; Analytics Engine for metering. OTel export
is evaluated per client when they operate their own observability stack.

### D8. Secrets

Default: Worker secrets for single-Worker credentials; Secrets Store for
shared credentials (AI provider keys used by AI Gateway and multiple
Workers); `.dev.vars` locally. Staged secret changes use the versions
workflow. Rotation is a Workflow step with an audit event.

### D9. Identity and access

Default: Cloudflare Access in front of every Preview, internal tool, and
admin surface; Access identity propagated as the approver identity in
Workflow gates; API tokens scoped per Worker for CI and agents. Client
end-user authentication is a per-project ADR (OpenAuth on KV is the noted
reference pattern).

### D10. CI/CD and release

Default: Workers Builds connected to the repository; Previews per branch;
production deploy on merge to main with human approval in the Workflow gate;
gradual deployments for lab-owned Workers (not available in WfP namespaces,
where rollout is implemented in the dispatch Worker); rollback rehearsed
before launch with the understanding that data is not rolled back.

### D11. Design system and component distribution

Default: shadcn/ui on Tailwind CSS. Components are copied into the
repository and owned by the lab, not consumed as a black-box dependency, so
every app can evolve them without forking a vendor.

| Option                            | Why not the default                                                                                                               |
| --------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| Atlassian Design System           | Strong for dense table and form workloads, but enterprise-opinionated, heavy, and hard to rebrand per client (conflicts with D12) |
| MUI, Chakra, Mantine              | Runtime weight and a recognizable generic look; theming fights the library instead of using it                                    |
| Radix or Base UI with bespoke CSS | Maximum control, but rebuilds what shadcn already provides; more agent effort per component                                       |
| Hand-written CSS per app          | No leverage; this is the state being retired                                                                                      |

Deciding requirements, in order: components must be ownable and reusable
across every lab app and client product (D12); the stack must be
rebrandable through tokens rather than overrides; and the component corpus
must be one that code-generating agents write fluently, because frontend
work is agent-assisted by design (Section 15).

Distribution is the load-bearing half of this decision. The lab runs its
own private shadcn registry so one component set serves every application:

- A namespaced registry (for example `@devartslab`) is declared in each
  project's `components.json`; components install with
  `npx shadcn add @devartslab/<item>`.
- Registries are authenticated with a bearer token or API key supplied
  through an environment variable, never committed (see D14). Namespaces
  allow a public tier, an internal tier, and a client-licensed tier side by
  side.
- A registry item can carry components, hooks, pages, config, and agent
  rules, so the registry distributes conventions and not only UI.
- Community registries from the shadcn Registry Directory may be used for
  advanced surfaces, subject to the supply-chain rule in Section 14: review
  the code on install and prefer versions published at least 7 days ago.

Any other design system or component library requires an ADR naming the
specific use case it serves and why the lab registry cannot.

### D12. Product identity and white-label readiness

Default: no business name, brand string, domain, legal entity, or contact
address is hardcoded anywhere in code, configuration, documentation
templates, email bodies, or UI. All of it resolves from one identity
manifest per product.

The requirement this satisfies: if the lab operates under a new name, or a
new venture starts, the accumulated research, design, code, and
documentation must be reusable without recreation. `devartslab` is a
variable, not a constant.

| Concern                 | Rule                                                                                                                                                                                          |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Identity manifest       | One file per product resolves legal name, display name, slug, primary domain, email domain, support addresses, social handles, logo and favicon asset paths, theme tokens, and license holder |
| Code and UI             | Read identity through a typed accessor; literal brand strings in components are a review failure                                                                                              |
| Theme                   | Brand expression lives in design tokens (D11), so rebranding is a token swap plus an asset swap                                                                                               |
| Documents and templates | Rendered from the manifest, following the existing org and consultant template pattern already proven in the RTS repository                                                                   |
| Infrastructure          | Domains, account names, and bucket or database names are derived from the slug, never typed twice                                                                                             |
| Verification            | A CI check greps the tree for forbidden literals (current business name, current domain) outside the manifest and asset files                                                                 |

Consequence: the scaffold generator (Phase 5) takes the identity manifest
as its primary input. The same generator produces a lab product and a
client-branded product from identical sources.

### D13. Authentication and authorization from day one

Default: every project declares its authentication and authorization model
before its first route exists, and implements it from the lab's shared auth
package rather than per-app reinvention.

Three planes are designed separately and never conflated:

| Plane          | Subjects                                | Default mechanism                                                                                                                                                                                     |
| -------------- | --------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| End user       | Customers and product users             | Identity-provider first per LAB-ADR-0001: no shared credential database; each app keeps its own profile table keyed by the IdP subject claim; the integration module ships through the registry (D11) |
| Administrative | Lab staff, operators, internal surfaces | Cloudflare Access in front of every admin and internal surface (D9); Access identity is the approver identity in Workflow gates                                                                       |
| Machine        | Agents, CI, service-to-service          | Scoped API tokens and OAuth-secured MCP servers with least-privilege tool scopes (Section 10)                                                                                                         |

Authorization rules that hold regardless of provider:

- Authorization is evaluated server-side, always. A client-side check is a
  display concern, never a control.
- Permissions are expressed as roles plus scopes against resources, not as
  boolean `isAdmin` flags.
- Tenant isolation is structural, per D1 and D3: a request carries a tenant
  identity and reaches only that tenant's Durable Object or database.
- Every authorization denial and every privilege change emits an audit
  event (Appendix B).
- Session, token, and refresh handling live in the shared module, so a
  vulnerability is fixed once for all products.

The identity layer is centralized while credential storage is not. This is
the substance of [LAB-ADR-0001](adr/LAB-ADR-0001-centralized-identity-layer.md)
(accepted 2026-10-25, lab-wide):

- Internal products federate through SSO plus Cloudflare Access, so there
  is no user database to build or operate.
- Client services use a multi-tenant identity provider with one isolated
  tenant per client, which keeps client end-user data with the client and
  preserves a clean handoff path.
- A single shared authentication database is explicitly rejected: it is one
  point of compromise across every client, and it blocks offboarding.
- Administrative observability is a telemetry concern, met with identity
  provider audit logs and error tracking, not by co-locating credentials.

Consequence: auth is a Phase 1 spine concern, not a pre-launch retrofit.
What ships through the registry (D11) is the integration module, session
handling, and authorization policy evaluation, which makes it a reusable
asset under D12 rather than per-product work. Credentials themselves never
live in lab-built tables.

### D14. Configuration and secret management

Default: one centralized configuration and secret store per environment
plane, with repositories holding references and never values.

| Plane                       | Store                                                                                                     | Rule                                                                                                                 |
| --------------------------- | --------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| Local development           | A single workspace-root store outside every repository, resolved by relative path                         | Deleting any repository must not lose its secrets; moving the workspace root must not break resolution               |
| Staging and production      | Platform-native stores: Worker secrets per Worker, Secrets Store for shared and account-level values (D8) | One source of truth, synced outward; never hand-copied between dashboards                                            |
| Distributed and self-hosted | The user's machine or deployment                                                                          | The lab holds nothing; credentials never traverse lab infrastructure (self-hosted products, career-tracker ADR-0016) |

Non-negotiables:

- PII is never committed to any repository, including private and
  local-only repositories, because a remote can be added later.
- Long-lived plaintext secrets on disk are a finding, not a convention.
  Material at rest is encrypted (SOPS with age for file-based config, the
  OS keychain or a password manager for human-held material).
- Secrets never enter agent context, prompts, or logs (Principle 7).
- Every secret has a named owner, a scope, and a rotation trigger.

Full architecture, platform-by-platform secret-store inventory, and the
staging and production research strategy live in
[CONFIG-AND-SECRETS.md](CONFIG-AND-SECRETS.md).

### D15. Model provider and local inference policy

Default for lab and delivery engineering work: the Devin team-plan quota is
the primary agent capacity. A local model (Ollama or equivalent) is used
only when that quota is exhausted.

| Rule                 | Statement                                                                                                                        |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| Primary              | Devin sessions, within the team plan, for agentic development and research work                                                  |
| Fallback             | Local inference engages only on quota exhaustion, not as a cost optimization and not as a default                                |
| Never silent         | Switching providers is an explicit, visible, administered state change (Section 23), and emits an audit event                    |
| Product inference    | In-app AI calls route through AI Gateway (Section 4.4) with spend limits, independent of the engineering-capacity policy above   |
| Self-hosted products | Products distributed to users may run local inference as their primary path when a project ADR says so (career-tracker ADR-0016) |

Rationale: quality of agentic engineering output is the constraint that
matters most, and a local fallback that activates quietly would degrade it
without anyone noticing. Visibility is the point, which is why this
decision depends on Section 23 rather than on a code-level fallback chain.

## 10. Guardrails

Guardrails are platform features, not conventions. Each row names the
mechanism precisely enough to be verified in configuration review.

| Guardrail          | Mechanism                                                                                                                  | Applies to                                       | Verification                                                              |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------ | ------------------------------------------------------------------------- |
| AI traffic control | AI Gateway: rate limits, spend limits, Guardrails, DLP, logging policy, dynamic routing for failover                       | Every model call, including Workers AI           | No direct provider endpoints in code; gateway dashboard shows all traffic |
| Code execution     | Dynamic Workers with `globalOutbound` disabled and minimal bindings; Containers with internet disabled unless required     | Generated and third-party code                   | Egress test from sandbox fails by default                                 |
| Capability scoping | MCP tools exposed with least-privilege scopes; per-agent tool allow-lists; OAuth-secured MCP servers                       | All agents                                       | Tool inventory per agent in `md/agents/`                                  |
| Approval gates     | Workflow `waitForEvent` steps with Access-authenticated approvers; Agents SDK human-in-the-loop for tool calls             | Phase transitions, client-visible outputs, spend | Gate events present in audit log                                          |
| Audit trail        | Tail Worker on every Worker; Agents SDK diagnostics events; Logpush for retention                                          | Everything                                       | Appendix B events observed end to end                                     |
| Resource limits    | `limits.cpu_ms` set per Worker; per-customer limits in WfP dispatch; Queues backpressure; WAF rate limits on public routes | Public and multi-tenant workloads                | Limits present in config; load test hits limits cleanly                   |
| Secrets handling   | Secrets Store bindings with scopes; no secrets in sandbox environments; secret scanning before push                        | All environments                                 | Pre-push gate; config review                                              |
| Preview exposure   | Access policy on all Previews; no production bindings in `previews` block                                                  | Every branch deployment                          | Access policy listed; Preview binding diff                                |
| Denial-of-wallet   | CPU limits, AI Gateway spend caps, request rate limits, Queues backlog alarms                                              | Public endpoints and agents                      | Cost guardrail checklist at launch                                        |

## 11. Data governance

| Area              | Rule                                                                                                                                | Platform facts                                                                                     |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| Classification    | Every dataset carries one class: public, internal, client-confidential, regulated. Class is recorded in `md/governance/`            | Drives logging policy, AI Gateway DLP, residency                                                   |
| Client boundaries | Client data lives in per-client resources (dedicated Worker, DO namespace, or D1 database); no cross-client reads by default        | WfP untrusted mode isolates cache and `request.cf`; DO per tenant isolates state                   |
| Residency         | Regional requirements use Data Localization Suite where contracted; otherwise documented as global                                  | Enterprise feature set; evaluate per contract                                                      |
| Retention         | Periods defined per class; deletion is a Workflow step with an audit event, not an ad hoc script                                    | Workers Logs 7 days; Workflow state 30 days; Queues messages up to 14 days; D1 Time Travel 30 days |
| Auditability      | Reads and writes to client-confidential or regulated data emit audit events                                                         | Tail Worker plus application events (Appendix B)                                                   |
| Backups           | D1 Time Travel plus periodic export to R2; R2 versioning for artifacts; restore tested at every launch gate                         | Time Travel is always on; 10 restores per 10 minutes per database                                  |
| AI data           | Prompts and responses classified like any dataset; payload logging in AI Gateway only for classes that permit it; ZDR where offered | Provider retention is not controlled by gateway log settings                                       |
| Secrets           | Never in code, CI files, or agent context; rotation through Secrets Store with an audit event                                       | Scoped secrets; deployer role required to bind                                                     |

## 12. Cost model

All figures USD, verified 2026-10-26, Workers Paid plan (Standard usage
model). Re-verify before quoting (Section 2.2).

| Item                                                      | Included per month          | Overage                                                                  |
| --------------------------------------------------------- | --------------------------- | ------------------------------------------------------------------------ |
| Subscription                                              | n/a                         | 5.00 minimum per account                                                 |
| Worker requests                                           | 10 million                  | 0.30 per additional million                                              |
| Worker CPU time                                           | 30 million CPU-ms           | 0.02 per additional million CPU-ms                                       |
| Static asset requests                                     | Unlimited                   | Free                                                                     |
| Egress and bandwidth                                      | Unlimited                   | Free                                                                     |
| Workers Logs                                              | 20 million events           | 0.60 per additional million; changes 2026-12-01 to Observability pricing |
| Workers Logpush                                           | 10 million requests         | 0.05 per additional million                                              |
| D1, KV, R2, DO, Queues, Workflows, AI Gateway, Workers AI | Product-specific allotments | See each product's pricing page (Section 22)                             |

Worked estimates (requests and CPU only; storage and AI billed separately):

| Scenario                                        | Requests          | CPU per request | Estimate                                            |
| ----------------------------------------------- | ----------------- | --------------- | --------------------------------------------------- |
| T0 site, 15 million requests, 80 percent static | 3 million dynamic | 7 ms            | 5.00 (all within included)                          |
| T1 app, 15 million dynamic requests             | 15 million        | 7 ms            | 8.00 (5.00 + 1.50 requests + 1.50 CPU)              |
| Hourly cron agent, 3 minutes CPU per run        | 720               | 180,000 ms      | 6.99                                                |
| T2 agent platform, 100 million requests         | 100 million       | 7 ms            | 45.40; 34.20 with 80 percent Workers Cache hit rate |

Cost controls that are part of the foundation: `limits.cpu_ms` per Worker,
AI Gateway spend limits per gateway, Analytics Engine metering per tenant,
monthly cost review in the Operate phase, and a per-project budget recorded
in the Design ADR set.

## 13. Limits and capacity cheat sheet

Consolidated from Section 4; every number is a hint to re-verify.

| Product               | Limit                                        | Workers Paid value                                          |
| --------------------- | -------------------------------------------- | ----------------------------------------------------------- |
| Workers               | CPU per request                              | 30 s default, 5 min max                                     |
| Workers               | Memory per isolate                           | 128 MB                                                      |
| Workers               | Subrequests per request                      | 10,000                                                      |
| Workers               | Workers per account                          | 500                                                         |
| Workers               | Worker size                                  | 64 MiB                                                      |
| Workers               | Cron Triggers per account                    | 250                                                         |
| Workers               | Wall time: HTTP                              | Unlimited while client connected                            |
| Workers               | Wall time: cron, queue consumer, DO alarm    | 15 min                                                      |
| Workers               | `waitUntil` after response                   | 30 s                                                        |
| Static Assets         | Files per version; file size                 | 100,000; 25 MiB                                             |
| D1                    | Database size                                | 10 GB (not raisable)                                        |
| D1                    | Databases per account; storage per account   | 50,000 (raisable); 1 TB (raisable)                          |
| D1                    | Time Travel                                  | 30 days; 10 restores per 10 min per database                |
| D1                    | Queries per invocation                       | 1,000                                                       |
| D1                    | Columns; bound params; statement length      | 100; 100; 100 KB                                            |
| D1                    | Max row, string, or BLOB size                | 2 MB                                                        |
| D1                    | Query duration                               | 30 s                                                        |
| Durable Objects       | Storage per object                           | 10 GB (SQLite)                                              |
| Durable Objects       | Classes per account                          | 500                                                         |
| Durable Objects       | Throughput per object                        | about 1,000 requests per second (soft)                      |
| Durable Objects       | WebSocket message (received)                 | 32 MiB                                                      |
| Queues                | Queues per account                           | 10,000                                                      |
| Queues                | Message size; retries; batch                 | 128 KB; 100; 100 messages                                   |
| Queues                | Throughput per queue                         | 5,000 messages per second                                   |
| Queues                | Retention; backlog                           | up to 14 days; 25 GB                                        |
| Queues                | Concurrent push consumers                    | 250                                                         |
| Workflows             | Step result; event payload                   | 1 MiB each                                                  |
| Workflows             | Persisted state per instance                 | 1 GB                                                        |
| Workflows             | Steps per instance                           | 10,000 default, 25,000 max                                  |
| Workflows             | Sleep                                        | 365 days                                                    |
| Workflows             | Completed state retention                    | 30 days                                                     |
| Workflows             | Concurrent instances per account             | 50,000 (waiting instances excluded)                         |
| Workflows             | Instance creation rate                       | 300 per second per account; 100 per second per workflow     |
| Workflows             | Where it cannot run                          | Workers for Platforms namespaces                            |
| Previews              | Previews per Worker; deployments per Preview | 500; 100                                                    |
| Workers for Platforms | Workers per namespace; tags per Worker       | Unlimited; 8                                                |
| Workers for Platforms | Gradual deployments                          | Not supported                                               |
| Workers Logs          | Retention                                    | 7 days                                                      |
| Cloudflare API        | Rate limit                                   | 1,200 requests per 5 min per account; 200 per second per IP |
| Request body          | Maximum (zone plan dependent)                | 100 MB Free and Pro; 200 MB Business; up to 5 GB Enterprise |

## 14. Threat model for agentic applications

Threats specific to agent-driven systems, with the foundation control that
addresses each. Projects extend this in `md/security/threat-model.md`.

| Threat                                   | Vector                                                  | Foundation control                                                                                     |
| ---------------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Prompt injection via retrieved content   | Web pages, documents, emails fed to the model           | Tool allow-lists per agent; human approval for client-visible outputs and spend; AI Gateway Guardrails |
| Tool misuse or privilege escalation      | Agent calls a tool outside its intended scope           | MCP least-privilege scopes; OAuth-secured MCP servers; audit of every tool call                        |
| Data exfiltration through generated code | Code attempts outbound network calls                    | Dynamic Workers with egress blocked; Containers with internet disabled; Outbound Worker in WfP         |
| Secrets exposure to the model or sandbox | Credentials in prompts or sandbox environment variables | Secrets stay in the Worker; Secrets Store bindings; no secrets in agent context (Principle 7)          |
| Cross-tenant data leakage                | Shared cache, shared DO, or shared D1 across clients    | Per-tenant DO or database; WfP untrusted mode; classification-driven access rules                      |
| Runaway cost                             | Loops, retries, abusive public callers                  | `limits.cpu_ms`; AI Gateway spend limits; rate limiting; Queues retry caps (100)                       |
| Replay or duplicate side effects         | Workflow step retried after a committed external write  | Intent-before-effect rule; idempotency keys; retries re-read state                                     |
| Unaudited state change                   | Direct writes bypassing the audit path                  | Tail Worker on every Worker; application events for data access; review flag for direct REST writes    |
| Preview exposure                         | Public Preview URL with real data                       | Access on all Previews; no production bindings in Previews                                             |
| Supply chain                             | Compromised dependency or brand-new package version     | Lab rule: prefer versions published at least 7 days ago; lockfiles; pre-push review                    |
| Over-trust in sandbox claims             | Assuming isolation that the runtime does not provide    | Read the Workers security model and sandbox security docs before each new execution pattern            |

## 15. Agent orchestration model

### 15.1 Two agent populations

| Population     | Purpose                                                           | Runtime                                                                              | Examples                                                       |
| -------------- | ----------------------------------------------------------------- | ------------------------------------------------------------------------------------ | -------------------------------------------------------------- |
| Lab operations | Run the lab: intake, research, estimates, indexes, Linear hygiene | Agents SDK on lab Workers; local CLI agents (Devin, Claude Code) for repository work | Intake agent, research agent, index regenerator, cost reviewer |
| Delivery       | Assist client projects end to end                                 | Agents SDK inside the project (T2) or lab-owned agents acting on client repos        | Scaffold agent, migration agent, QA agent, runbook writer      |

### 15.2 Shared contracts

- Every agent has a written card in `md/agents/<name>.md`: purpose, tools
  (MCP servers and scopes), state schema, approval points, audit events,
  failure modes.
- Agents never hold the only copy of state (Principle 3).
- Agents reach external systems only through MCP servers or bindings;
  never through ad hoc HTTP with embedded credentials.
- Human approval is mandatory for: phase transitions, client-visible
  outputs, spend above a per-project threshold, deletion of any data class
  above internal, and production deploys.

### 15.3 Work tracking

Linear is the system of record for activity (lab rule). Every agent run that
changes a repository or a deployment creates or updates a Linear issue; the
Delivery Workflow mirrors its phase into the issue state. GitHub Issues are
not used.

### 15.4 Optional: cagent for local agent prototyping

cagent is Docker's agent builder and runtime: YAML-defined agents with MCP
tool support, running locally. It fits development-time prototyping of agent
flows and tool sets. It does not fit production; the production runtime is
the Agents SDK on Durable Objects. Treat cagent definitions as design
artifacts that inform the agent card, not as deployable units. Adoption is
optional per project.

Repository: <https://github.com/docker/docker-agent> (formerly
`docker/cagent`; the old URL redirects).

### 15.5 Agent-facing conventions in every repository

- `AGENTS.md` at the repository root states how agents should work in the
  repo: commands, verification steps, forbidden actions, Linear tracking.
- `REPO_INDEX.md` lists what lives where and is regenerated on change.
- `md/` holds the plan, ADRs, agent cards, governance register, runbooks.
- Skills resolve by name from the user-global skills directory
  (`~/.agents/skills/`), never by vendoring copies into repositories. A
  repository declares the skills it needs in `AGENTS.md`. Installed copies
  under tool-specific directories (for example `~/.claude/skills/`) are
  generated downstream artifacts, not a second source of truth, and drift
  between them is checked, not assumed away. A repository that must be
  self-contained distributes its skills through a registry, not through a
  copied file.

## 16. Repository scaffold (master layout)

One layout for every repository created from this foundation. This
reconciles the earlier `md/` and `docs/` variants into a single tree;
`md/` is the documentation root.

```text
<repo>/
  AGENTS.md                     agent-facing conventions for this repo
  README.md                     human entry point; links to md/
  REPO_INDEX.md                 what lives where (regenerated on change)
  LICENSE                       per Q4; public repos must carry one
  SECURITY.md                   disclosure contact and supported versions
  .gitignore                    env files, keys, build output, OS junk
  .dev.vars.example             local configuration template; never real secrets
  wrangler.jsonc                environments, previews block, bindings, limits
  src/                          Worker entrypoint, domain modules, agents, workflows
  test/                         Workers Vitest integration tests
  md/
    MEGA-PLAN.md                copy or pointer to this plan at the adopted revision
    PRE-DEVELOPMENT.md          pointer to the blueprint; filled checklist for this project
    adr/                        ADR-0001 to ADR-0015 reserved for foundation decisions D1 to D15 in order; project-specific ADRs start at ADR-0016 (see numbering rule below)
    research/                   market, competitors, personas, requirements (Research phase)
    design/                     PRD, user flows, wireframes, visual direction
    architecture/               C4 diagrams, data models, environment map, cost estimate
    agents/                     one card per agent: tools, scopes, state, approvals
    governance/                 data classification register, retention schedule, audit event catalog
    security/                   threat model, access policies, secret inventory (names only)
    runbooks/                   launch checklist, rollback, restore test, on-call notes
    api/                        OpenAPI, AsyncAPI, MCP tool schemas
  .github/workflows/            only when external CI is needed beyond Workers Builds
```

Directories are created when the tier needs them; a T0 site has no
`agents/`. The PRE-DEVELOPMENT blueprint defines what each document
contains and when it is required.

### 16.1 ADR numbering and placement

Numbers are allocated once, lab-wide, so that an ADR reference means the
same thing in every repository.

| Range                     | Holds                                                                                                                  | Lives in                                |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------- | --------------------------------------- |
| `ADR-0001` to `ADR-0015`  | One per foundation decision D1 to D15, in order, recording the project's instantiation (accepted default or deviation) | The implementing repository's `md/adr/` |
| `ADR-0016` and upward     | Project-specific decisions with no foundation counterpart                                                              | The implementing repository's `md/adr/` |
| `LAB-ADR-0001` and upward | Lab-wide decisions that bind every project but are not themselves foundation decisions                                 | This repository's `md/adr/`             |

Rules:

- The reserved block is reserved even when empty. A project that accepts
  the D7 default still files `ADR-0007` stating that it accepted it, so the
  absence of a file always means "not yet decided" rather than "decided
  silently".
- A decision that binds more than one project is a `LAB-ADR` in the
  foundation, never an ADR inside one product. A lab-wide decision stored
  in a product repository is invisible to every other project and ships
  with that product if it is ever distributed.
- When a decision's scope grows from project to lab-wide, it is promoted:
  moved to this repository, renumbered in the `LAB-ADR` range, and its
  provenance (origin repository and commit) recorded in the file, because a
  cross-repository move cannot carry git history.
- When a `LAB-ADR` answers an open question in Section 20, the question is
  marked resolved with a pointer to it.
- Extending the foundation decision set (a new D-number) extends the
  reserved range; it does not renumber existing project ADRs.

### 16.2 Documentation platform readiness

The lab operates a public documentation platform. Publication is a status,
not an assumption: every document is internal by default, and joining the
platform is an explicit, recorded transition.

| Status      | Meaning                                      |
| ----------- | -------------------------------------------- |
| `refining`  | Under active refinement; not for publication |
| `ready`     | Reviewed and cleared; eligible to publish    |
| `published` | Live on the platform                         |

Rules:

- Readiness is tracked once, in `LAB_INDEX.md` at the workspace root, not
  duplicated into each repository and not duplicated inside this document.
- Research and UX documentation under refinement stays internal until its
  repo owner marks it ready; a living document (like this plan or a DEV-47
  style research ticket) may carry per-section readiness since sections
  finish at different times.
- A document that names private paths, orgs, people, or client details is
  published only after the identity and privacy check implied by D12 and
  Section 11.
- Git submodule and vendoring approaches to shared documentation are
  rejected: repos link to a single authoritative copy at a pinned adopted
  revision in `AGENTS.md`, they do not embed it.

Scaffold generation (roadmap Phase 5) is a template repository or a
`create` style generator that stamps this tree with the project name, tier,
and chosen dispatch model. Its implementation is out of scope here.

## 17. Phased roadmap

| Phase               | Deliverable                                                                                                                                                           | Exit criteria (measurable)                                                                                                                             |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1. Spine            | One Worker with Static Assets, one Durable Object class, one Queue, one D1 database, Workers Logs and Traces on, Tail Worker attached, running dev to preview to prod | `wrangler dev` and production both green; Preview URL behind Access; test suite passes locally and in Preview; audit events visible for one request    |
| 2. First client app | A real T1 application delivered on the spine                                                                                                                          | Client served on a custom domain; restore from Time Travel rehearsed; cost within estimate for first month; runbook delivered                          |
| 3. Agent layer      | Agents SDK plus AI Gateway plus the Delivery Workflow (intake to scaffold)                                                                                            | Agent run killed mid-process resumes correctly; every tool call present in audit log; spend limit triggers and is observed; approval gate exercised    |
| 4. Client dispatch  | Workers for Platforms for a flagship client under D1 option C                                                                                                         | Second client deployed with per-client CPU limits and Outbound Worker; no cross-client data path found in review; metering visible in Analytics Engine |
| 5. Public framework | This document set plus the scaffold generator published under the chosen license                                                                                      | An external developer scaffolds and deploys a T1 app unaided in under one day; license, SECURITY.md, and contribution guide present                    |

## 18. Risks and mitigations

| Risk                                                   | Likelihood | Impact | Mitigation                                                                                                                       |
| ------------------------------------------------------ | ---------- | ------ | -------------------------------------------------------------------------------------------------------------------------------- |
| Platform facts drift (prices, limits, betas)           | High       | Medium | Section 2.2 decay policy; quarterly verification; Section 22 register                                                            |
| Vendor concentration on Cloudflare                     | Medium     | High   | Bindings keep application code portable in shape; R2 is S3-compatible; D1 and DO use SQLite; documented export paths per product |
| Beta dependency in a client commitment                 | Medium     | High   | Tier defaults use GA products; betas need an ADR with a fallback                                                                 |
| Workflows not available in WfP namespaces              | Certain    | Medium | D1 option C hybrid; design dispatch so processes live in lab Workers                                                             |
| D1 10 GB ceiling reached                               | Medium     | Medium | Per-tenant or per-domain databases from day one; Hyperdrive escape hatch                                                         |
| Preview leaks production data                          | Medium     | High   | Rule: no production bindings in Previews; Access on all Previews; review diff                                                    |
| Agent cost runaway                                     | Medium     | Medium | AI Gateway spend limits; `limits.cpu_ms`; retry caps; monthly cost review                                                        |
| Observability pricing change (2026-12-01)              | Certain    | Low    | Re-estimate log volume before December; sampling policy ready                                                                    |
| Deprecation of AI Gateway Unified API for single calls | Certain    | Low    | Use provider-native or binding endpoints for single calls; Unified only for dynamic routes                                       |
| Over-engineering small projects                        | Medium     | Medium | Tier system; ADR required to exceed tier                                                                                         |

## 19. Deliberately excluded

- Vendor-neutral abstraction layers over bindings: bindings are already the
  portability seam; extra indirection adds cost without benefit.
- General-purpose containers for every workload: Containers and Sandbox are
  used only where Workers primitives do not fit.
- Agent autonomy without audit: no phase transition, client-visible output,
  or spend happens outside the guardrail path.
- Pages for new projects.
- A permanent staging environment by default: Previews cover branch testing;
  staging exists only under contract.
- GitHub Issues for tracking: Linear is the source of truth.
- Code and configuration in this document.

## 20. Open questions

| ID  | Question                                                                                                                                                                                                                                                                             | Owner          | Needed by                                       |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------- | ----------------------------------------------- |
| Q1  | Per-client metering and pricing model once Workers for Platforms is live (Analytics Engine as meter?)                                                                                                                                                                                | Lab lead       | Phase 4 design                                  |
| Q2  | Which AI providers anchor the stack, in what failover order, BYOK or Unified Billing                                                                                                                                                                                                 | Lab lead       | Phase 3 design                                  |
| Q3  | Is client code execution (Dynamic Workers or Containers) offered to clients or internal only                                                                                                                                                                                         | Lab lead       | Phase 4 design                                  |
| Q4  | Public license for the framework repository                                                                                                                                                                                                                                          | Lab lead       | Before Phase 5 and before any push of this repo |
| Q5  | OTel export versus Tail Worker as the primary audit path when a client has its own stack                                                                                                                                                                                             | Per client ADR | Phase 2                                         |
| Q6  | Adopt the `cf` CLI once stable, and migration plan from Wrangler                                                                                                                                                                                                                     | Lab lead       | Re-check monthly                                |
| Q7  | ~~End-user authentication default~~ Resolved 2026-10-25 by LAB-ADR-0001: identity-provider first, no shared credential database; SSO plus Access for internal, multi-tenant IdP per client for client services. Remaining per-project choice is which IdP, recorded as a project ADR | Resolved       | Resolved                                        |

## 21. Glossary

| Term               | Meaning                                                                                      |
| ------------------ | -------------------------------------------------------------------------------------------- |
| Binding            | Configuration-declared handle through which a Worker reaches a resource (`env.DB`, `env.AI`) |
| Coordination atom  | The entity a Durable Object instance represents: tenant, session, room, agent                |
| Dispatch namespace | Workers for Platforms container for customer Workers                                         |
| Dynamic Worker     | Worker created at runtime by a Worker Loader binding                                         |
| Gate               | Workflow step that waits for an approval event                                               |
| Preview            | Isolated branch environment under the same Worker (distinct from Wrangler environments)      |
| Tail Worker        | Worker that receives trace events from other Workers                                         |
| Time Travel        | D1's always-on point-in-time recovery                                                        |
| Tier               | T0 to T3 default product set (Section 5.3)                                                   |
| Browser Run        | Current name of Browser Rendering                                                            |
| Basin              | Current name of Pipelines, R2 Data Catalog, and R2 SQL                                       |
| Sandbox            | Umbrella for Dynamic Workers and Containers used to run untrusted code                       |

## 22. Source register

All sources read on 2026-10-21 in markdown form. Re-verify per Section 2.2.

| Topic                            | URL                                                                                     |
| -------------------------------- | --------------------------------------------------------------------------------------- |
| Workers limits                   | <https://developers.cloudflare.com/workers/platform/limits/>                            |
| Workers pricing                  | <https://developers.cloudflare.com/workers/platform/pricing/>                           |
| Storage selection                | <https://developers.cloudflare.com/workers/platform/storage-options/>                   |
| Wrangler configuration           | <https://developers.cloudflare.com/workers/wrangler/configuration/>                     |
| Previews                         | <https://developers.cloudflare.com/workers/previews/>                                   |
| Previews resources and limits    | <https://developers.cloudflare.com/workers/previews/resources/>                         |
| Workflows limits                 | <https://developers.cloudflare.com/workflows/reference/limits/>                         |
| Rules of Workflows               | <https://developers.cloudflare.com/workflows/build/rules-of-workflows/>                 |
| Queues limits                    | <https://developers.cloudflare.com/queues/platform/limits/>                             |
| D1 limits                        | <https://developers.cloudflare.com/d1/platform/limits/>                                 |
| D1 Time Travel                   | <https://developers.cloudflare.com/d1/reference/time-travel/>                           |
| Durable Objects limits           | <https://developers.cloudflare.com/durable-objects/platform/limits/>                    |
| Durable Objects best practices   | <https://developers.cloudflare.com/durable-objects/best-practices/>                     |
| Agents SDK                       | <https://developers.cloudflare.com/agents/>                                             |
| MCP on Cloudflare                | <https://developers.cloudflare.com/agents/model-context-protocol/>                      |
| AI Gateway features              | <https://developers.cloudflare.com/ai-gateway/>                                         |
| Sandboxes overview               | <https://developers.cloudflare.com/sandbox/>                                            |
| Dynamic Workers                  | <https://developers.cloudflare.com/dynamic-workers/>                                    |
| Containers                       | <https://developers.cloudflare.com/containers/>                                         |
| Workers for Platforms            | <https://developers.cloudflare.com/cloudflare-for-platforms/workers-for-platforms/>     |
| Cloudflare for SaaS              | <https://developers.cloudflare.com/cloudflare-for-platforms/cloudflare-for-saas/>       |
| Secrets Store, Workers           | <https://developers.cloudflare.com/secrets-store/integrations/workers/>                 |
| Workers secrets                  | <https://developers.cloudflare.com/workers/configuration/secrets/>                      |
| Workers Logs                     | <https://developers.cloudflare.com/workers/observability/logs/workers-logs/>            |
| Tail Workers                     | <https://developers.cloudflare.com/workers/observability/logs/tail-workers/>            |
| Workers Logpush                  | <https://developers.cloudflare.com/workers/observability/logs/logpush/>                 |
| OpenTelemetry export             | <https://developers.cloudflare.com/workers/observability/exporting-opentelemetry-data/> |
| Analytics Engine                 | <https://developers.cloudflare.com/analytics/analytics-engine/>                         |
| Workers Builds                   | <https://developers.cloudflare.com/workers/ci-cd/builds/>                               |
| Versions, deployments, rollbacks | <https://developers.cloudflare.com/workers/versions-and-deployments/>                   |
| Static Assets                    | <https://developers.cloudflare.com/workers/static-assets/>                              |
| Next.js on Workers (vinext)      | <https://developers.cloudflare.com/workers/framework-guides/web-apps/nextjs/>           |
| Access                           | <https://developers.cloudflare.com/cloudflare-one/access-controls/>                     |
| Data Localization Suite          | <https://developers.cloudflare.com/data-localization/>                                  |
| Workers security model           | <https://developers.cloudflare.com/workers/reference/security-model/>                   |
| Cloudflare API limits            | <https://developers.cloudflare.com/fundamentals/api/reference/limits/>                  |
| Cloudflare skills bundle         | <https://github.com/cloudflare/skills>                                                  |
| cagent (docker-agent)            | <https://github.com/docker/docker-agent>                                                |

Added revision 3 (read 2026-10-25):

| Topic                               | URL                                                                      |
| ----------------------------------- | ------------------------------------------------------------------------ |
| shadcn registry directory           | <https://ui.shadcn.com/docs/directory>                                   |
| shadcn registry authentication      | <https://ui.shadcn.com/docs/registry/authentication>                     |
| shadcn registry namespaces          | <https://ui.shadcn.com/docs/registry/namespace>                          |
| Cloudflare Secrets Store (overview) | <https://developers.cloudflare.com/secrets-store/>                       |
| Flagship feature flags              | <https://developers.cloudflare.com/flagship/>                            |
| GitHub Actions secrets              | <https://docs.github.com/en/actions/concepts/security/secrets>           |
| Devin secrets and site cookies      | <https://docs.devin.ai/product-guides/secrets>                           |
| Devin environment best practices    | <https://docs.devin.ai/enterprise/environment-management/best-practices> |
| SOPS                                | <https://github.com/age-sops/sops>                                       |
| 1Password service accounts          | <https://www.1password.dev/service-accounts>                             |

Re-verified 2026-10-26 (DEV-109 daily pass; Workers limits, Workers pricing,
D1 limits, Durable Objects limits, Queues limits, Workflows limits, Previews;
URLs unchanged from the register above). Corrections applied: D1 databases per
account is raisable by request; cheat sheet rows added for D1 2 MB row size,
Queues per account, and Workflows concurrent instances.

---

## 23. Administrative management interface

The lab is operated by looking at it, not by reading documentation. This
section defines the visual control plane that makes platform state
observable and changeable.

### 23.1 Why this exists as a foundation concern

Every decision in this plan creates state that someone must see and
control: which features are live, which provider is serving inference
(D15), which agents are scheduled (Section 15), what spend has accrued
(Section 12), which secrets exist (D14). Without one interface, that state
is spread across vendor dashboards, config files, and log queries, and the
only way to answer "what is the system doing right now" is to read code.

The governing rule: **no silent fallback anywhere**. Any automatic
degradation (provider switch, cache bypass, queue backpressure, feature
disable) must surface as visible state in this interface and emit an audit
event. Fallback behavior is permitted; invisible fallback behavior is not.

### 23.2 Control surfaces

| Surface             | What it shows                                                                        | What it controls                                                               |
| ------------------- | ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------ |
| Feature flags       | Every flag, scope, current value, targeting rule, rollout percentage                 | Toggle on and off; target a tenant, user, or percentage; emergency kill switch |
| Model and provider  | Active provider, Devin quota consumed and remaining, fallback state (D15)            | Explicit provider selection; enable or disable local inference                 |
| Agents              | Registered agents, schedules, last run, outcome, session cost (Section 15)           | Enable, disable, pause, run now, change schedule                               |
| Delivery lifecycle  | Each engagement's Workflow instance, current phase, pending gates (Section 8)        | Approve or reject gates with Access identity recorded                          |
| Environments        | Worker versions per environment, Preview inventory, binding map, drift warnings      | Promote, roll back, delete stale Previews                                      |
| Cost and limits     | Spend by product, AI Gateway spend against caps, limits approached (Sections 12, 13) | Adjust spend caps and CPU limits; set alert thresholds                         |
| Audit feed          | Live stream of Appendix B events, filterable by tenant, actor, and event             | Nothing; read-only by design                                                   |
| Secret inventory    | Secret names, scopes, owners, age, rotation due (D14, never values)                  | Trigger a rotation Workflow                                                    |
| Tenants and clients | Per-client status, isolation model, metering (D1, Section 11)                        | Suspend, resume, adjust per-client limits                                      |
| Data governance     | Classification register, retention schedules, restore-test recency (Section 11)      | Trigger a restore test or retention run                                        |

### 23.3 Toggle contract

Every toggle in the interface declares the same metadata, so the interface
is generated from the registry of toggles rather than hand-built:

| Field          | Meaning                                                                     |
| -------------- | --------------------------------------------------------------------------- |
| `id`           | Stable identifier used in code and in audit events                          |
| `scope`        | lab, project, tenant, or user                                               |
| `default`      | Value when no override exists                                               |
| `owner`        | Who is accountable for its state                                            |
| `kind`         | Release flag (temporary), operational toggle (permanent), or kill switch    |
| `blast_radius` | What breaks when it is off; shown in the UI before the switch is thrown     |
| `audit_event`  | Event emitted on change (Appendix B `flag.changed`)                         |
| `expires_at`   | Required for release flags, so temporary flags do not become permanent debt |

### 23.4 Implementation model

- Flag state lives in Flagship, which provides targeting and percentage
  rollouts independent of deploys (Section 4.4), so toggling never requires
  a deployment.
- The interface itself is a lab-owned Worker with Static Assets, built from
  the D11 component registry, behind Cloudflare Access (D9). It is a T2
  surface and inherits every guardrail in Section 10.
- Reads come from Analytics Engine (metrics and metering), Workers Logs and
  the audit R2 bucket (events), D1 (project, tenant, and toggle registry),
  and the Cloudflare API for version, Preview, and binding state.
- Writes go through the same approval path as everything else: a change
  with a blast radius above a declared threshold opens a Workflow gate
  rather than applying immediately.
- Operating it is itself auditable: the interface is not a back door around
  Section 10.

### 23.5 Division of responsibility with vendor consoles

The lab interface does not attempt to replace vendor administration. The
split is explicit so there is no ambiguity about where to act:

| Concern                                                                                      | Where it is administered                                           |
| -------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| Agent sessions, machine access, agent secrets, infrastructure and analytics for agentic work | The Devin Administration space (access, infrastructure, analytics) |
| Cloudflare account resources, DNS, zones, Access policies                                    | The Cloudflare dashboard, or Terraform for declared resources      |
| Product feature state, tenant state, lifecycle gates, lab-wide toggles                       | The lab administrative interface defined in this section           |
| Work tracking and decisions                                                                  | Linear (Section 15.3)                                              |

Where a vendor console is authoritative, the lab interface links to it and
mirrors read-only status rather than duplicating control.

### 23.6 Roadmap placement

This interface is built incrementally alongside the phases in Section 17:
the audit feed and environment view accompany Phase 1, flags and the model
and provider toggle accompany Phase 3, and tenant and metering views
accompany Phase 4. A static read-only version is acceptable in Phase 1;
write paths require the approval gates described above.

---

## Appendix A: ADR template

```text
# ADR-NNNN: <title>

Status: proposed | accepted | superseded by ADR-MMMM
Date: YYYY-MM-DD
Foundation decision: D<n> (or "project-specific")
Tier: T0 | T1 | T2 | T3

## Context
What requirement or constraint forces a decision. Cite MEGA-PLAN sections
and the live docs consulted (with date).

## Options
Table: option, how it works, pros, cons, platform facts that matter.

## Decision
The chosen option and the deciding requirement.

## Consequences
What becomes easier, what becomes harder, what must be monitored, what
would trigger revisiting.

## Compliance
Which guardrails (Section 10) and governance rules (Section 11) this touches
and how they are satisfied.
```

## Appendix B: Audit event catalog (baseline)

Every project emits at least these events through the Tail Worker or
application logging. Field names are illustrative; the schema is defined in
the scaffold.

| Event                            | Emitted when                                                               | Minimum fields                                               |
| -------------------------------- | -------------------------------------------------------------------------- | ------------------------------------------------------------ |
| `request.handled`                | Any Worker invocation completes                                            | worker, environment, outcome, cpu_ms, wall_ms, tenant        |
| `agent.session.started`          | Agents SDK instance created or resumed                                     | agent, session_id, tenant, trigger                           |
| `agent.tool.called`              | Any MCP or binding tool invocation                                         | agent, session_id, tool, scope, duration_ms, outcome         |
| `agent.model.called`             | Any AI Gateway request                                                     | gateway, provider, model, tokens_in, tokens_out, cost, cache |
| `agent.state.changed`            | `setState` or SQL write in an agent                                        | agent, session_id, keys_changed                              |
| `workflow.step.completed`        | Workflow step returns                                                      | workflow, instance_id, step, attempt, duration_ms            |
| `gate.requested`                 | Workflow waits for approval                                                | workflow, instance_id, gate, requested_of                    |
| `gate.approved`, `gate.rejected` | Approval event received                                                    | workflow, instance_id, gate, approver (Access identity)      |
| `data.read`, `data.write`        | Access to client-confidential or regulated data                            | dataset, class, tenant, actor, operation                     |
| `data.deleted`                   | Retention or deletion Workflow step runs                                   | dataset, class, tenant, policy, count                        |
| `secret.rotated`                 | Secrets Store value replaced                                               | secret_name (never value), actor                             |
| `flag.changed`                   | Any toggle changed in the admin interface (Section 23)                     | flag_id, scope, from, to, actor, blast_radius                |
| `provider.switched`              | Model or inference provider changed, manually or on quota exhaustion (D15) | from, to, reason, actor or "automatic"                       |
| `authz.denied`                   | An authorization check rejects a request (D13)                             | subject, resource, action, policy, tenant                    |
| `identity.resolved`              | Product identity manifest loaded at build (D12)                            | product_slug, display_name, manifest_version                 |
| `deploy.completed`               | Production or Preview deployment                                           | worker, version, environment, actor                          |
| `sandbox.executed`               | Dynamic Worker or Container run                                            | kind, egress_allowed, duration_ms, outcome                   |
| `limit.hit`                      | CPU, spend, rate, or backlog limit reached                                 | limit, worker or gateway, value                              |

## Appendix C: Project start checklist

1. Classify the project into a tier (Section 5.3) and record it.
2. Create the Linear issue tree for the engagement; link the Workflow
   instance ID.
3. Instantiate ADRs for D1 to D15 from the defaults; mark any deviation.
4. Re-verify Section 13 numbers that the design depends on (record the date).
5. Produce the cost estimate (Section 12) and budget threshold for approvals.
6. Build the data classification register and retention schedule.
7. Write agent cards for every agent in scope, with tool scopes.
8. Draft the threat model from Section 14.
9. Draft `wrangler.jsonc` environments and the `previews` block (no
   production bindings in Previews).
10. Confirm Access policy for Previews and internal surfaces.
11. Create the product identity manifest (D12); confirm no brand literals
    outside it.
12. Declare the authentication and authorization model for all three planes
    before the first route (D13).
13. Register the project's configuration and secret locations in the
    central store; add no secret to the repository (D14,
    CONFIG-AND-SECRETS).
14. Register the project's toggles in the administrative interface
    (Section 23), each with scope, owner, and blast radius.
15. Complete PRE-DEVELOPMENT exit criteria; then development begins.

---

## Revision log

| Revision | Date       | Change                                                                                                                                                                                                                                                                                                                                                                                    |
| -------- | ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1        | 2026-10-21 | Initial plan: principles, map, D1 and D2, guardrails, governance, scaffold, roadmap                                                                                                                                                                                                                                                                                                       |
| 2        | 2026-10-21 | Research edition: product catalog by concern, tiers, parity and environment model, D1 to D10, cost and limits, threat model, orchestration model, single scaffold, source register; audit findings DEV-99 resolved                                                                                                                                                                        |
| 3        | 2026-10-25 | Reuse and control edition: D11 design system and private component registry, D12 product identity and white-label readiness, D13 auth and authz from day one, D14 configuration and secret management, D15 model provider policy; Section 23 administrative management interface; principles 11 to 13; CONFIG-AND-SECRETS companion document; new audit events; source register additions |
| 3a       | 2026-10-26 | Amendment: platform doctrine Section 1.2 (defaults are revisable, adoption path for second platforms, stack-agnostic lifecycle); skills sourcing rule in 15.5; ADR numbering Section 16.1; documentation platform readiness Section 16.2; Q7 resolved by LAB-ADR-0001                                                                                                                     |
| 3b       | 2026-10-26 | DEV-109 daily verification: Sections 4.1-4.3, 4.7, 12, 13 re-verified against official docs; D1 databases-per-account marked raisable; cheat sheet gained D1 row size, Queues per account, Workflows concurrent instances                                                                                                                 |

_Living document: revise after each phase exit and each quarterly fact
verification. Decisions become ADRs in `md/adr/` of implementing
repositories._
