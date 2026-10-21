# Project Pre-Development Architecture & Planning Blueprint

This comprehensive blueprint outlines the best practices for structuring, categorizing, and documenting the pre-development phase of any project. Use this framework to align with successful engineering and design methodologies before writing a single line of code.

> **Aligned with:** [MEGA-PLAN.md](MEGA-PLAN.md) — the lab-wide foundation covering Cloudflare-native production architecture, the client dispatch model (D1), the agent state model (D2), guardrails, and data governance. Every project scaffold inherits from that plan; this document defines what to produce _before_ implementation begins.

## 1. Research & Discovery

The foundation of any successful project begins with understanding the problem, the users, and the market.

- **Market Research**: Analysis of existing solutions, target audience segmentation, and market opportunities.
- **User Personas**: Detailed profiles representing the ideal users of your application.
- **Competitor Analysis**: Strengths and weaknesses of similar products in the ecosystem.
- **Product Requirements Document (PRD)**: A living document defining the product's scope, core features, and success metrics.

## 2. Design & UX

Before implementation, visualize the user journey and system interactions.

- **User Flows**: Step-by-step diagrams showing how users navigate through the application.
- **Wireframes**: Low-fidelity sketches or blueprints of the user interface layout.
- **Visual Design**: High-fidelity mockups defining the look, feel, and branding.

## 3. Architecture & Tech Stack (Cloudflare-Native)

Map out the technical infrastructure and make deliberate decisions. Per the foundation plan, production targets Cloudflare and local development runs the same runtime via `wrangler dev` / Miniflare.

- **Tech Stack Selection**: Documentation covering frontend (Workers Static Assets / vinext), backend (Workers), agent logic (Agents SDK), and data choices (D1, KV, R2, Durable Objects), with justifications for each.
- **System Architecture (C4 Model)**: High-level diagrams contextualizing the system within its environment, detailing containers, components, and code.
- **Data Models**: Entity-relationship diagrams and database schema definitions, mapped to the chosen Cloudflare storage product by data shape.
- **Environment Parity**: `wrangler.toml` environment definitions (`dev`, `preview`, `prod`) showing how bindings swap per environment with no code changes.
- **Architecture Decision Records (ADRs)**: Logs capturing significant architectural choices, constraints, and the rationale behind choosing them.

### Required foundation decisions (from MEGA-PLAN)

Every project records these two decisions as ADRs before development:

- **D1 - Client dispatch model**: Workers for Platforms (per-client Worker, dispatch namespace, custom domains) vs. single multi-tenant Worker with per-tenant Durable Object. Default: tenant-DO until a second flagship client proves the need.
- **D2 - Agent state model**: Durable Object per agent session + Workflow per business process (production default) vs. stateless calls (only for one-shot, read-only assistance).

## 4. Agentic & Full-Stack Implementation Planning

Organize the codebase structure and module responsibilities to maintain a clean separation of concerns.

- **Frontend**: UI components, state management, and user interaction logic.
- **Backend**: API services, data processing, and business logic.
- **Agent Logic**: Specialized modules dictating the behavior, roles, and collaboration frameworks for AI agents.
- **Delivery Pipeline**: Model the project's research -> design -> development -> launch lifecycle as a durable Workflow with checkpointed phases and approval gates at phase transitions.

## 5. Guardrails & Data Governance (pre-development checklist)

Defined before implementation so they are built in, not bolted on:

- **AI traffic control**: All model calls routed through AI Gateway (rate limits, spend caps, logging, provider failover).
- **Capability scoping**: Agents receive tools via MCP with least-privilege scopes; document which tools each agent may call.
- **Approval gates**: Which phase transitions and client-visible outputs require human or policy-based approval.
- **Audit trail**: Tail Worker records every tool call and state change; define which events are audited.
- **Data classification**: Classify every dataset as public, internal, client-confidential, or regulated.
- **Client boundaries**: Per-client data separation model (dedicated Worker or DO namespace) recorded in ADR-D1.
- **Secrets**: `wrangler secret` / Secrets Store for production; `.dev.vars` locally and gitignored; never in code, CI, or agent context.
- **Execution isolation**: Generated and third-party code paths run in Sandbox SDK / Dynamic Workers environments, never inline in request handlers.

---

## Recommended Directory Structure

Map your planning documents to this hierarchical folder structure to maintain organization for hundreds of Markdown files:

```text
/my-project-root/
│
├── .gitignore
├── wrangler.toml              # env.dev / env.preview / env.prod bindings
├── .dev.vars.example          # local config template (never real secrets)
├── src/                       # Worker entrypoint + domain modules
├── AGENTS.md                  # agent-facing conventions for the repo
└── docs/
    │
    ├── 01_research/
    │   ├── market_research.md
    │   ├── competitor_analysis.md
    │   └── user_personas.md
    │
    ├── 02_design/
    │   ├── prd.md
    │   ├── user_flows.md
    │   └── wireframes.md
    │
    ├── 03_architecture/
    │   ├── tech_stack.md
    │   ├── system_architecture_c4.md
    │   ├── data_models.md
    │   └── environment_parity.md   # wrangler.toml env map
    │
    ├── 04_adrs/
    │   ├── adr_001_client_dispatch_model.md     # D1 decision
    │   ├── adr_002_agent_state_model.md         # D2 decision
    │   └── adr_003_data_governance.md
    │
    └── 05_implementation/
        ├── frontend_plan.md
        ├── backend_plan.md
        ├── agents_plan.md
        └── delivery_pipeline.md     # Workflow phases + approval gates
```

---

## 6. Advanced Technical Specs & Component Deep-Dives

- **`docs/technical/architecture/`**: Low-level system schematics, component interaction data flows, and concurrency state management.
- **`docs/technical/api-specs/`**: OpenAPI, AsyncAPI, and gRPC contracts defining message formats and endpoint schemas.
- **`docs/technical/mcp-schemas/`**: JSON payload definitions and tool schemas for Model Context Protocol interactions, including least-privilege scope declarations per tool.
- **`docs/technical/security/`**: Cryptographic standards, JWT payload structures, TLS configurations, and secret rotation policies.
- **`docs/technical/governance/`**: Data classification register, retention policies, client-boundary rules, and audit event catalog.

---

## 7. Extended Foundation Structure for Production-Grade Agentic Apps (Devartslab Standard)

To support fully realized, scalable production applications across the Devartslab ecosystem, the root repository architecture expands to include these operational and agentic directories:

- **`/agents/`**: System prompts, behavioral guardrails, agent persona definitions, multi-agent communication topologies, and chain-of-thought evaluation scripts.
- **`/infra/`**: Infrastructure-as-code scripts, Cloudflare Worker configurations, Terraform for declared Cloudflare resources, and automated environment provisioning.
- **`/evaluation/`**: Automated test suites for voice latency, intent classification accuracy, tool-calling success rates, and regression metrics.
- **`/schemas/`**: Database definitions, vector embeddings models, state persistence structures, and migration scripts.
- **`/workflows/`**: Delivery pipeline definitions: Workflow classes, Queue consumers, phase gate policies, and approval transition handlers.

### Optional tooling

- **docker-agent (cagent)**: Local, YAML-defined agent prototyping on Docker. Use for fast local iteration on agent flows; the production runtime is always Cloudflare Agents SDK on Durable Objects. cagent output is prototype material, not a deployable unit.

---

## 8. Pre-Development Exit Criteria

A project may enter development when all of the following exist:

- PRD approved with success metrics defined.
- Architecture C4 + data models committed, mapped to Cloudflare products.
- ADRs recorded for D1 (dispatch), D2 (agent state), and data governance.
- Guardrails checklist items assigned: AI Gateway routing, MCP scopes, approval gates, audit events.
- Delivery pipeline sketched: Workflow phases and their approval transitions.
- `wrangler.toml` environments drafted with dev/preview/prod bindings.
