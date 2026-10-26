# LAB-ADR-0001: Centralized identity layer, no shared auth database

Status: accepted
Date: 2026-10-25
Scope: lab-wide (internal products and client services)
Foundation decision: D13 (authentication and authorization from day one)
Resolves: MEGA-PLAN open question Q7

Provenance: originally authored as `ADR-0002-centralized-auth.md` in the
career-tracker repository (commit `0ce979d`, 2026-10-25) and promoted here
on 2026-10-26 because its scope is lab-wide, not product-specific. Git
history for the original file remains in that repository; cross-repository
moves cannot carry history forward.

## Context

The lab runs many projects. A question came up: should one shared
authentication database hold all project IDs and user logins so that a
master admin view can observe every product?

The answer determines whether authentication is a lab-operated service or a
federated concern, and it has to be settled before any product ships auth,
because retrofitting identity is expensive and because client data
ownership is contractual rather than technical.

## Options

| Option                                      | How it works                                                                              | Pros                                                                  | Cons                                                                                                                      |
| ------------------------------------------- | ----------------------------------------------------------------------------------------- | --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| A. Shared lab auth database                 | One lab-operated credential store holds users for every product and client                | Single admin view; one integration to build                           | One point of compromise across all clients; lab holds client end-user credentials; blocks clean client handoff; highest liability |
| B. Centralized identity layer, federated storage | One identity provider per scope; each app keeps its own profile table keyed by the provider's subject claim | No credential store to build or operate; clean per-client isolation and handoff | Admin observability must come from telemetry rather than from one database; per-scope provider configuration |
| C. Fully independent auth per product       | Each product implements its own authentication                                             | Maximum autonomy                                                      | Reimplements the same security-critical code repeatedly; a vulnerability must be fixed in many places                      |

## Decision

Option B. No shared authentication database. Centralize the identity layer
instead.

- One identity provider per scope:
  - Internal products: SSO through the workspace identity provider plus
    Cloudflare Access. No user database to build or operate.
  - Client services: a multi-tenant identity provider with one isolated
    tenant per client. Provider selection per engagement is a project ADR.
- Each application keeps its own user and profile table keyed by the
  identity provider's subject claim. No application reads another
  application's auth data.
- Administrative observability comes from identity provider audit logs and
  error tracking, not from co-locating credentials.

Deciding requirement: client end-user data belongs to the client, and a
lab-operated master credential store would make clean offboarding
impossible while concentrating breach impact.

## Consequences

What becomes easier:

- No credential storage, hashing, reset flow, or breach surface is
  lab-operated.
- Client offboarding is an export and a tenant deletion, not a data
  separation project.
- A vulnerability in the integration layer is fixed once, in the shared
  module distributed through the component and package registry (D11).

What becomes harder:

- Administrative observability is a telemetry problem and must be designed
  as one; the administrative interface (MEGA-PLAN Section 23) mirrors
  identity provider status rather than querying one database.
- Each scope carries its own provider configuration and its own failure
  mode.

Monitor:

- Per-provider pricing as user counts grow, since identity provider cost
  scales with monthly active users.
- Whether a read-only administrative plane aggregating organization and
  membership metadata becomes necessary. If built, it stores no
  credentials.

Revisit when:

- A client contract requires lab-operated identity.
- A provider's tenancy model stops guaranteeing per-client isolation.

## Compliance

- Secrets (D8, D14): credentials live only in the identity provider;
  applications hold no passwords.
- Authorization (D13): authorization is still evaluated server-side per
  application; federated identity settles authentication only.
- Data governance (MEGA-PLAN Section 11): per-tenant isolation gives a
  clean export and handoff path for client offboarding.
- Audit (MEGA-PLAN Appendix B): `authz.denied` and privilege changes are
  emitted by the application; authentication events come from the provider's
  audit log.

## Notes on current products

career-tracker has no authentication today (JSON record store, per its
`AGENTS.md`). When auth lands there it follows this decision, and the
product-specific provider choice is recorded as a project ADR in that
repository (numbering per MEGA-PLAN Section 16).
