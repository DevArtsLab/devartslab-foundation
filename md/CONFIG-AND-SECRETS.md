# Configuration, PII, and Secret Architecture

> Companion to [MEGA-PLAN.md](MEGA-PLAN.md), expanding foundation decision
> D14. Covers four planes: local development, staging, production, and
> products distributed to users. Documentation only; no implementation.

| Field          | Value                                                                                                  |
| -------------- | ------------------------------------------------------------------------------------------------------ |
| Status         | Living document, revision 1                                                                            |
| Audience       | Lab engineers, agents, future operators under a different brand                                        |
| Scope          | Where configuration, PII, and secrets live; how they are resolved, protected, rotated, and inventoried |
| Not in scope   | Secret values, code, provider-specific setup steps                                                     |
| Facts verified | 2026-10-25 (see Section 10)                                                                            |

---

## Table of contents

1. Why this document exists
2. Current state audit
3. Principles
4. Plane 1: local development architecture
5. Resolution contract
6. Protection at rest
7. Plane 2 and 3: staging and production
8. Plane 4: distributed and self-hosted products
9. Platform inventory: every store the lab touches
10. Research strategy and open questions
11. Source register

---

## 1. Why this document exists

The lab's volume is growing faster than ad hoc practice can hold. Four
requirements drive the architecture:

1. **Repository deletion must be non-destructive.** Removing a local
   checkout must never destroy credentials, because gitignored secrets are
   invisible to every backup that follows the repository.
2. **The workspace must be portable.** Moving the workspace root to another
   user profile, machine, or container must preserve every secret and break
   no path.
3. **Nothing brand-specific or person-specific may be hardcoded.** Per
   MEGA-PLAN D12, identity is a variable; the same applies to the operator's
   personal data.
4. **Agents operate this estate.** Unattended agents read configuration and
   must never be able to read a credential they do not need, or leak one
   into a prompt, a log, or a commit.

## 2. Current state audit

Observed in the workspace on 2026-10-25. This is the baseline the
architecture must improve, recorded honestly so progress is measurable.

| Observation                                                                                                                                                                                                     | Assessment                                                                                                                                                                    |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A root-level `config/` directory already exists outside every project repository                                                                                                                                | Correct instinct and the right foundation. The centralization principle is already in practice                                                                                |
| `config/secrets/` is gitignored and organized by platform subdirectory                                                                                                                                          | Good shape: platform grouping scales better than per-repo duplication                                                                                                         |
| Secret material is stored unencrypted at rest: dotenv files, a private certificate, an OAuth client secret document, a database credential file, a git-crypt key, and recovery codes                            | **Finding.** Disk encryption is the only control. Any process, backup, sync client, or agent with filesystem read access reads everything                                     |
| `pii.json` carries full legal name, phone, email, location, and immigration status, and is **tracked** in a git repository                                                                                      | **Finding.** The repository currently has no remote, so exposure is latent rather than actual. One `git remote add` turns latent into permanent, because history preserves it |
| The parent meta-repo gitignores `config/` and also ignores `.env`, `*.env`, `*.pem`, `*.key`, and `secrets/` at any depth                                                                                       | Strong defense in depth; keep it                                                                                                                                              |
| Absolute paths appear in generated artifacts (for example the installed scheduler job in agent-research-kit)                                                                                                    | **Finding.** Blocks requirement 2: moving the workspace root breaks resolution                                                                                                |
| The RTS repository (`org-intelligence-revenue-system/shared/config`) already implements tracked `_template` and `_default` trees with gitignored real data, a documented precedence chain, and a management CLI | **This is the pattern to generalize.** It solves per-entity config correctly; the lab needs the same model one level up                                                       |

The RTS precedence chain, worth lifting verbatim as a convention:
explicit flag, then environment variable, then an active-context file, then
a single existing candidate.

## 3. Principles

| #   | Principle                                              | Consequence                                                                                     |
| --- | ------------------------------------------------------ | ----------------------------------------------------------------------------------------------- |
| 1   | Repositories hold references, never values             | A repository may name `GOOGLE_CLIENT_SECRET`; it may never contain it                           |
| 2   | One store per plane                                    | Local has exactly one root store; each deployed environment has exactly one authoritative store |
| 3   | Resolution is relative, never absolute                 | Paths resolve from a discovered workspace root, so the tree is movable                          |
| 4   | Encrypted at rest, decrypted at use                    | Plaintext exists in memory for the life of a process, not on disk indefinitely                  |
| 5   | PII is never committed anywhere                        | Including private and remote-less repositories; a remote can always be added later              |
| 6   | Every secret has an owner, scope, and rotation trigger | Unowned credentials are the ones that outlive their purpose                                     |
| 7   | Least privilege for agents                             | An agent receives the narrowest scope for the task, and never a store-wide key                  |
| 8   | Inventory is generated, not remembered                 | A registry lists what exists and where, by name only                                            |

## 4. Plane 1: local development architecture

The target tree. The workspace root is the unit of portability: move it,
and nothing breaks. Delete any project directory, and no secret is lost.

```text
<workspace-root>/                     any path; discovered, never hardcoded
|
|-- .workspace.json                   ROOT MARKER (tracked)
|     identifies the root, declares schema version and store layout
|
|-- config/                           the ONLY location for PII and secrets
|   |
|   |-- registry.json                 generated inventory: names, scopes,
|   |                                 owners, rotation due, consumers
|   |                                 (names and metadata only, never values)
|   |
|   |-- identity/<brand>.json          D12 manifests (tracked, non-secret)
|   |
|   |-- profiles/                      person data (ENCRYPTED, tracked as ciphertext)
|   |   |-- operator.sops.json         replaces today's plaintext pii.json
|   |   +-- profile.schema.json        schema (tracked, plaintext)
|   |
|   |-- _shared/                       credentials shared across entities
|   |   +-- <platform>/<env>/secrets/
|   |
|   |-- projects/<slug>/               entity axis: owning project
|   |   +-- <platform>/<env>/
|   |       |-- secrets/               (ENCRYPTED)
|   |       |-- config/                (non-secret settings)
|   |       +-- data/                  (manifests, seed, schemas)
|   |
|   |-- orgs/<slug>/                   partner orgs (multi-tenant entities)
|   |-- consultants/<slug>/
|   |
|   +-- env/<entity-path>/             rendered dotenv, gitignored;
|                                      path mirrors the entity path
|
|-- career-tracker/                    a project; contains NO secret values
|   |-- .env.example                   required key NAMES with placeholders
|   +-- project.json                   declares slug + the secret ids it needs
|
+-- <other projects>/                  same contract
```

Layout rules:

- Partition by owning entity first; providers nest underneath. A new
  project, org, or consultant is one new top dir - no reshuffling.
- Uniform leaf: every `<entity>/<platform>/<env>/` holds the same
  `{secrets, config, data}` triple. `_default` and `_template` dirs hold
  per-entity-type defaults (proven in the RTS repository).
- Encrypt `secrets/` subtrees only; `config/` and `data/` stay tracked
  plaintext. `env/` renders gitignored dotenv at the mirrored entity path.

Data flow for a local run:

```text
  developer or agent runs a project task
            |
            v
  resolve workspace root (Section 5)
            |
            v
  read project.json -> list of required secret ids
            |
            v
  decrypt only those ids under config/projects/<slug>/  (age key from OS keychain)
            |
            v
  inject as environment variables into the child process only
            |
            v
  process exits -> plaintext is gone; nothing written to the project tree
```

Why this satisfies the requirements:

- **Delete a repository safely.** Credentials live under
  `config/projects/<slug>/`, outside the checkout. Removing the
  checkout removes no secret. Re-cloning restores a working project with no
  credential re-entry.
- **Move the workspace.** Every path is relative to the discovered root, so
  relocating to a different user profile, a container mount, or another
  machine changes one thing: where the root happens to be.
- **Rebrand.** Identity manifests are data in `config/identity/`, so the
  brand is swapped, not refactored.
- **Keyed by slug, not path.** Renaming or relocating a project directory
  does not orphan its secrets.

## 5. Resolution contract

Precedence, first hit wins. This mirrors the chain already proven in the
RTS repository:

| Order | Source                                                             | Use                                                  |
| ----- | ------------------------------------------------------------------ | ---------------------------------------------------- |
| 1     | Explicit flag or argument                                          | One-off overrides and tests                          |
| 2     | `DEVARTSLAB_HOME` (generically: `<LAB>_HOME`) environment variable | Containers, CI, non-standard layouts                 |
| 3     | Upward search for `.workspace.json` from the current directory     | The normal path; works from any depth in any project |
| 4     | A single existing candidate in a conventional location             | Convenience for fresh machines                       |

Rules:

- No tool, template, generated artifact, or scheduler entry may contain an
  absolute path to the store. The current scheduler job in
  agent-research-kit violates this and is the reference case for the fix.
- Resolution failure is a hard error with a remediation message, never a
  silent fallback to a default or an empty value (MEGA-PLAN principle 13).
- The root marker carries a schema version so the layout can migrate.

## 6. Protection at rest

| Material                                                   | Protection                                                     | Rationale                                                                                             |
| ---------------------------------------------------------- | -------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Structured config with sensitive values, person profiles   | SOPS with an age identity; values encrypted, keys readable     | Diffable, reviewable, safe to track; the age private key is the only secret to protect                |
| The age identity itself                                    | OS keychain or a password manager; never in the workspace      | A key stored beside its ciphertext provides no protection                                             |
| Human-only material: recovery codes, personal certificates | Password manager, not the workspace                            | These are not application inputs and should not be machine-readable                                   |
| Rendered dotenv files                                      | Gitignored, treated as disposable cache, regenerated on demand | Short-lived plaintext is acceptable; permanent plaintext is not                                       |
| Live API keys with frequent rotation                       | Prefer a runtime store over committed ciphertext               | Git history preserves ciphertext permanently; "encrypted today" is only as strong as the key, forever |

Important limitation to record: encrypted-in-git patterns put ciphertext in
history permanently. If the age key leaks, every historical version is
exposed, and rewriting history does not reach forks or clones. Encryption
is not deletion. High-churn, high-value credentials belong in a runtime
store (Section 7), not in a committed file.

Remediation sequence implied by the Section 2 findings, in priority order:

1. Rotate anything that was ever stored in plaintext, on the assumption
   that plaintext-at-rest is compromised-until-rotated.
2. Move person data out of git tracking, then re-add it as SOPS ciphertext;
   treat existing history as exposed and scrub or abandon the repository.
3. Move human-only material (recovery codes, certificates) to a password
   manager.
4. Replace absolute paths in generated artifacts with root-relative
   resolution.

## 7. Plane 2 and 3: staging and production

### 7.1 The two integration patterns

Every option reduces to one of two shapes:

| Pattern                | How it works                                                                                                                                       | Trade-off                                                                                                              |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Sync into the platform | The store is the source of truth; values are pushed into the platform's own secret store; the app reads ordinary bindings or environment variables | Simple, no runtime dependency, no startup latency; values exist in two places and can drift                            |
| Fetch at runtime       | The platform holds one bootstrap credential; the app pulls the rest at startup                                                                     | One source of truth; adds a runtime dependency, a startup cost, and a bootstrap credential that is now the crown jewel |

Recommendation for the lab: **sync into the platform.** The production
architecture is Cloudflare-native, where bindings are the integration layer
(MEGA-PLAN Principle 8) and a runtime fetch would trade a configuration
concern for an availability dependency. Drift is managed by making the sync
one-directional and automated, never manual dashboard editing.

### 7.2 Options assessed

| Option                                    | Model                                         | Fit for the lab                                                                                                                                                                                                                                            |
| ----------------------------------------- | --------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Cloudflare Secrets Store + Worker secrets | Platform-native, account-level and per-Worker | **Default.** Already the D8 decision; bindings, no extra vendor, no extra bill. Constraint: open beta, and currently integrated with Workers and AI Gateway only, so it is not a whole-estate store                                                        |
| SOPS with age                             | Encrypted files in git                        | **Default for local and for low-churn declarative config.** Zero infrastructure, no vendor. Not for high-churn live keys (Section 6)                                                                                                                       |
| 1Password with service accounts           | Managed, human plus machine                   | **Strong candidate for the human and bootstrap tier.** Already the right home for recovery codes and certificates; service accounts scope machine access per vault, and secrets can be injected into a process for its lifetime only without touching disk |
| Infisical                                 | Open source, self-hostable                    | Best fit if a single cross-platform runtime store becomes necessary; MIT-licensed with a self-host path, so it does not add vendor lock-in                                                                                                                 |
| Doppler                                   | Managed SaaS only                             | Excellent developer experience, but no self-host path and per-user pricing; conflicts with the self-hosted posture in career-tracker ADR-0016                                                                                                              |
| HashiCorp Vault / OpenBao                 | Self-hosted, dynamic secrets, PKI             | Most capable (dynamic credentials, encryption as a service), and the heaviest to operate. Revisit only when dynamic database credentials or internal PKI become real requirements                                                                          |

### 7.3 Recommended target architecture

A three-tier model, chosen because no single store covers every plane well:

```text
  TIER 1  Human and bootstrap material
          Password manager with service accounts
          recovery codes, certificates, the age identity,
          the few bootstrap tokens that unlock other tiers
                        |
                        v
  TIER 2  Source of truth for application configuration
          SOPS + age ciphertext in the config store (Section 4)
          declarative, reviewable, environment-scoped
                        |
                        | one-directional automated sync
                        v
  TIER 3  Runtime stores, per platform
          Cloudflare Worker secrets / Secrets Store  (production runtime)
          GitHub Actions secrets                      (CI)
          Devin secrets                               (agent sessions)
          other platform stores as needed
                        |
                        v
          applications read bindings and environment variables only
```

Rules for the sync:

- One direction only, Tier 2 to Tier 3. A dashboard edit is drift and gets
  overwritten; this is intentional.
- Environment-scoped: a staging value can never be promoted into production
  by accident, because the mapping is declared, not typed.
- Rotation is a tracked operation that updates Tier 2 and re-syncs, and
  emits an audit event (MEGA-PLAN Appendix B `secret.rotated`).
- The secret inventory surfaces in the administrative interface
  (MEGA-PLAN Section 23), by name and metadata only.

Note the deployment mechanics already recorded in MEGA-PLAN Section 4.7:
`wrangler secret put` deploys a new version immediately, so a secret change
that must ship with code uses the staged versions workflow instead.

## 8. Plane 4: distributed and self-hosted products

For products shipped to users under a self-hosted distribution ADR
(career-tracker ADR-0016 is the reference case), the posture inverts: the
lab must hold nothing.

| Concern               | Rule                                                                                                        |
| --------------------- | ----------------------------------------------------------------------------------------------------------- |
| User credentials      | Generated and stored on the user's machine; never transit lab infrastructure                                |
| OAuth client identity | The app's own client registration may ship with the product; user tokens never do                           |
| Default posture       | A fresh install contains no lab credential of any kind                                                      |
| Telemetry             | Off by default; any opt-in path carries its own classification under MEGA-PLAN Section 11                   |
| Lab liability         | Nothing to breach means nothing to disclose; this is the compliance argument behind career-tracker ADR-0016 |

## 9. Platform inventory: every store the lab touches

Each platform the lab uses that can hold a secret, what it is authoritative
for, and the constraint that matters. Verified 2026-10-25.

| Platform                     | Store                                                    | Authoritative for                              | Key facts and constraints                                                                                                                                                                                                                                                                                   |
| ---------------------------- | -------------------------------------------------------- | ---------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Cloudflare                   | Worker secrets                                           | Credentials owned by one Worker                | Encrypted per Worker; `secret put` and `secret delete` deploy a new version immediately; use `versions secret` to stage with code                                                                                                                                                                           |
| Cloudflare                   | Secrets Store (account level)                            | Values shared across Workers and AI Gateway    | Open beta; binding type `secrets_store_secrets`; binding requires Super Administrator or Secrets Store Deployer; secrets carry scopes; unavailable on the China Network                                                                                                                                     |
| Cloudflare                   | AI Gateway BYOK provider keys                            | Upstream model provider credentials            | Keep provider keys here or in Secrets Store, never in Worker variables                                                                                                                                                                                                                                      |
| Cloudflare                   | `.dev.vars`                                              | Local development values only                  | Never uploaded; gitignored; a committed `.dev.vars.example` documents key names                                                                                                                                                                                                                             |
| GitHub                       | Actions secrets (org, repo, environment)                 | CI-only credentials                            | Encrypted client-side with libsodium sealed boxes before reaching GitHub; org secrets can be policy-limited by repository; environment secrets can require reviewer approval; log redaction is best-effort and **not guaranteed**                                                                           |
| GitHub                       | Actions variables                                        | Non-secret configuration                       | Not encrypted and not redacted; never use for sensitive values                                                                                                                                                                                                                                              |
| GitHub                       | Dependabot secrets                                       | Private registry access for dependency updates | Separate store from Actions; easy to forget during rotation                                                                                                                                                                                                                                                 |
| Devin (app.devin.ai)         | Secrets: enterprise, organization, repository, session   | Credentials agents need during sessions        | Encrypted at rest; tiers cascade with the most specific winning; types are key-value, cookie, and TOTP; a `note` field carries usage guidance; all org members can use org secrets while only admins can view or edit; guidance is to define at the highest applicable tier and never place secrets in YAML |
| Devin                        | Administration space (access, infrastructure, analytics) | Agent access, machine identity, session policy | The vendor console for agent administration; the lab interface mirrors status rather than duplicating control (MEGA-PLAN Section 23.5)                                                                                                                                                                      |
| Password manager (1Password) | Vaults and service accounts                              | Human-only material and bootstrap tokens       | Service accounts give non-human authentication scoped per vault; `op run` injects secrets into a subprocess for its lifetime, so values never land on disk; up to 100 service accounts                                                                                                                      |
| Vercel                       | Environment variables per environment                    | Any Vercel-hosted surface                      | Separate store from Cloudflare; a second place to rotate if used                                                                                                                                                                                                                                            |
| Railway                      | Service environment variables                            | Legacy or incidental deployments               | **Treat as compromised-by-default:** the 2026-10-21 breach is tracked in DEV-101; rotate anything that ever lived here and prefer migration off                                                                                                                                                             |
| Google Cloud                 | OAuth client credentials and refresh tokens              | Gmail and Calendar ingestion                   | Restricted scopes for mail content; refresh tokens are the high-value artifact; Testing status caps at 100 users with 7-day refresh token expiry                                                                                                                                                            |
| MongoDB Atlas                | Database users and connection strings                    | Atlas-backed data                              | Connection strings embed credentials; rotate as a unit                                                                                                                                                                                                                                                      |
| Salesforce                   | Connected app secrets, SFDX auth URLs                    | RTS org automation                             | An SFDX auth URL is a full-access credential in a single string; treat as a password                                                                                                                                                                                                                        |
| n8n                          | Credential store                                         | Workflow connections                           | Encrypted with an instance key; the instance key is itself a Tier 1 secret and a single point of failure                                                                                                                                                                                                    |
| Airtable                     | Personal access tokens                                   | CTD and partner exports                        | Scope per base; prefer short-lived where supported                                                                                                                                                                                                                                                          |
| PyPI                         | API tokens and recovery codes                            | Package publishing                             | Prefer trusted publishing (OIDC) over long-lived tokens; recovery codes are Tier 1 human material                                                                                                                                                                                                           |
| Linear                       | API keys, OAuth apps                                     | Work tracking automation                       | Agent-facing; scope narrowly since agents hold it                                                                                                                                                                                                                                                           |
| Cloudflare Access            | Service tokens                                           | Headless access to protected surfaces          | The documented fix for agents blocked by Access (DEV-83); a service token bypasses identity, so scope and rotate deliberately                                                                                                                                                                               |

Cross-cutting observations:

- The estate has **at least eight independent runtime stores**. Without a
  declared source of truth, rotation is guesswork. This is the strongest
  argument for the Tier 2 model in Section 7.3.
- GitHub Actions variables and GitHub Actions secrets are different stores
  with different protections; confusing them is a common and quiet leak.
- Prefer OIDC and short-lived credentials wherever a platform offers them
  (PyPI trusted publishing, GitHub Apps over personal access tokens), since
  a credential that does not persist cannot be exfiltrated later.

## 10. Research strategy and open questions

Re-verification cadence, following the MEGA-PLAN Section 2.2 decay policy:

| Class                                                      | Re-verify |
| ---------------------------------------------------------- | --------- |
| Secrets Store beta status and product integrations         | Monthly   |
| Platform pricing and plan gates for secret features        | Quarterly |
| Breach and advisory status for every platform in Section 9 | Monthly   |
| The inventory itself (has a new store appeared?)           | Quarterly |

Research tasks, in dependency order:

1. Validate the Section 4 tree and the Section 5 resolution chain on a
   second machine or container, since portability is the central claim and
   is currently untested.
2. Prove the Tier 2 to Tier 3 sync for one real credential end to end,
   including rotation, before generalizing.
3. Measure the operational cost of SOPS plus age for an operator working
   alone, and decide whether a runtime store is warranted yet or is
   premature infrastructure.
4. Decide the inventory generator's source: derived from the store, or a
   declared manifest validated against it.
5. Define the agent scoping model concretely: which tier an unattended
   agent may reach, and how a per-task scope is issued and expires.

Open questions:

| ID  | Question                                                                                                                                               | Needed by                                                   |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------- |
| S1  | Does a single cross-platform runtime store (Infisical) earn its operational cost, or is SOPS plus platform-native stores sufficient at current volume? | Before the estate exceeds a maintainable number of projects |
| S2  | Does the lab adopt a password manager as the formal Tier 1, or continue with the OS keychain?                                                          | Before the next credential rotation cycle                   |
| S3  | Secrets Store is in open beta and Workers-and-AI-Gateway-scoped; what is the fallback if it does not reach GA on the needed timeline?                  | Phase 2 design                                              |
| S4  | What is the retention and scrub plan for repositories whose history already contains PII or plaintext secrets?                                         | Immediately; history is permanent                           |
| S5  | How are secrets handled for Previews, given that Preview isolation is only real when the previews block declares its own resources?                    | Phase 1 exit                                                |
| S6  | Does the lab need per-client credential isolation (separate vaults or accounts per client) before the first Workers for Platforms client?              | Phase 4 design                                              |

## 11. Source register

Read 2026-10-25 unless noted.

| Topic                                       | URL                                                                                                       |
| ------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| Cloudflare Secrets Store                    | <https://developers.cloudflare.com/secrets-store/>                                                        |
| Secrets Store, Workers integration          | <https://developers.cloudflare.com/secrets-store/integrations/workers/>                                   |
| Worker secrets                              | <https://developers.cloudflare.com/workers/configuration/secrets/>                                        |
| AI Gateway bring your own keys              | <https://developers.cloudflare.com/ai-gateway/configuration/bring-your-own-keys/>                         |
| GitHub Actions secrets                      | <https://docs.github.com/en/actions/concepts/security/secrets>                                            |
| Devin secrets and site cookies              | <https://docs.devin.ai/product-guides/secrets>                                                            |
| Devin environment management best practices | <https://docs.devin.ai/enterprise/environment-management/best-practices>                                  |
| SOPS                                        | <https://github.com/age-sops/sops>                                                                        |
| 1Password service accounts                  | <https://www.1password.dev/service-accounts>                                                              |
| 1Password `op run`                          | <https://www.1password.dev/cli/reference/commands/run>                                                    |
| OWASP DevSecOps: secrets patterns           | <https://owasp.github.io/DevSecOpsGuideline/2-Process/2-6-Deploy/2-6-2-Secrets-in-Kubernetes-and-GitOps/> |

---

## Revision log

| Revision | Date       | Change                                                                                                                                                                              |
| -------- | ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1        | 2026-10-25 | Initial: current-state audit, four-plane model, local architecture and resolution contract, three-tier staging and production recommendation, platform inventory, research strategy |

_Living document: revise with each platform change and each quarterly
verification pass. Decisions become ADRs in the implementing repository._
