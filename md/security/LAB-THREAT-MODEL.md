# LAB-THREAT-MODEL - DevArts Lab baseline threat model (all repositories)

Status: revision 2, 2026-10-28. Baseline confirmed by the lab owner 2026-10-27; revision 2 adds attacker profile A6, the agents-and-tooling boundary, the per-boundary threat enumeration, the controls-vs-reality lifecycle, and the MEGA-PLAN revision 3d controls. Applies to every DevArtsLab repository. This file is the source of truth; the Devin security scan profile "DevArtsLab org baseline threat model" is a copy and is re-synced when this file changes, and its carried revision number must match this file's (a mismatch is itself a finding). Facts marked **[charter]** come from this repository: `md/MEGA-PLAN.md` (Sections 4.9, 10, 14, D9, D13, D15, Appendix B), `md/CONFIG-AND-SECRETS.md` (Sections 3, 7.3, 8), `md/PII-IN-REPOSITORIES.md` and `md/adr/LAB-ADR-0001-centralized-identity-layer.md`. The repository inventory comes from `DevArtsLab/devartslab` `LAB_INDEX.md`. Everything else was inferred from repository descriptions.

Per-repository convention: each repository extends this baseline in its own `md/security/threat-model.md` (MEGA-PLAN Section 14, PRE-DEVELOPMENT Section 8.6), and its system-specific text wins where the two differ. Example: `DevArtsLab/tool-universal-ai-config` `md/security/threat-model.md`.

## System overview

DevArts Lab is a small lab running many repositories. The platform doctrine is Cloudflare-native in production, local-first in development, and agent-orchestrated across delivery [charter]. Production surfaces are Cloudflare Workers on lab domains: `devartslab.com` (devartslab-site), `docs.devartslab.com` (devartslab-notion proxy), `data.devartslab.com` (tool-github-export Worker + R2), `mail.devartslab.com` (devarts-mail) and `voice-quote.devartslab.com` (voice-quote-proxy). Netlify hosts voice-quote, and Vercel is also used [LAB_INDEX]. Many other repositories are local CLIs, agent tooling, docs, demos or stale ML/product prototypes.

## Components (repository classes)

| Name                   | Covers                                                                       | Examples                                                                                                                                                  |
| ---------------------- | ---------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Edge services**      | Cloudflare Workers, Pages, D1/R2/KV/DO bindings, `wrangler.*`                | devartslab-site, devartslab-notion, devarts-mail, voice-quote-proxy, tool-github-export (Worker)                                                          |
| **Web apps**           | Next.js / Angular / React apps with server routes or APIs                    | career-tracker, voice-quote, city-scrapers-meetings-viewer, guaranteed-income-program, ai-starter-kits, health-pulse (web)                                |
| **Local tools & CLIs** | CLIs and scripts run on the developer's machine, installers, Homebrew tap    | tool-universal-ai-config, tool-github-actions-audit, tool-pnpm-fleet, tool-readme-links, tool-airtable-export, saystop, sesame-voice-escort, homebrew-tap |
| **Agent tooling**      | Skills, prompts, plugins, agent configs, MCP wiring, scheduled agents        | agent-skills, agent-research-kit, devin-plugin-devartslab, sesame-link-runbook, org-intelligence-revenue-system (n8n, AI)                                 |
| **ML & data APIs**     | Python model-serving APIs and pipelines                                      | drug-admet-predictor, virtual-screener-ai-drug-discovery, sentiment-analyzer-ml-pipeline, documind (Spring Boot + AWS CDK)                                |
| **CI & release**       | `.github/workflows/*`, publish pipelines, GitHub Actions in every repository | all repositories                                                                                                                                          |
| **Docs-only**          | Repositories with no executable code                                         | devartslab-foundation, devarts-business, waykeeper, healthcare-ai-validation, american-community-network                                                  |

## Trust boundaries

1. **Public internet to edge services / web apps**: any anonymous caller. Every Preview URL, internal tool and admin surface is meant to sit behind Cloudflare Access [charter D9].
2. **End user / admin / machine identity planes**: kept separate. End-user auth is IdP-first with no shared credential database, admin access goes through Cloudflare Access, and machines use scoped API tokens and OAuth MCP [charter D13, LAB-ADR-0001]. Authorization is server-side only, and tenant isolation is structural (a per-tenant DO or database) [charter D13].
3. **Repository to secret stores**: repositories hold secret references, never values. One store per plane: SOPS+age as the source of truth, synced one way to Worker secrets / Secrets Store, GitHub Actions secrets and Devin secrets [charter CONFIG-AND-SECRETS 3, 7.3].
4. **Untrusted content to agents/models**: web pages, documents, emails and repository content fed to a model [charter 14].
5. **Developer machine**: local tools run as the user, with access to `~/.agents/`, provider configs, MCP tokens and the `config/` PII store (local, gitignored) [LAB_INDEX].
6. **Supply chain**: npm/pnpm, PyPI, GitHub Actions, Homebrew tap, install scripts.
7. **Agents and their tooling**: lab and delivery agents authenticate with scoped MCP and API tokens, read and write repositories, and act unsupervised; skills and MCP servers are themselves supply-chain items (pinned, reviewed) [charter 10, 14]; agent actions must be auditable [charter Appendix B]; facts an agent derives from untrusted input are quarantined until promoted [charter 14].

## Assets (lab owner: all weighted equally)

- **Secrets and tokens**: Cloudflare API tokens, Secrets Store, GitHub tokens, MCP/OAuth tokens (Gmail, Google Calendar, Linear, Slack, Sentry, Vercel), AI-provider keys.
- **PII**: never committed anywhere, including private repositories [charter CONFIG-AND-SECRETS principle 5]. Examples: career-tracker job-search data (Gmail/Calendar/LinkedIn ingestion), devarts-mail mailboxes, health-pulse health data, nonprofit donor data (org-intelligence-revenue-system).
- **Client and tenant data**: no cross-tenant leakage [charter 14].
- **Integrity of lab domains and Workers**, of agent configuration and skills (they steer agents that have write access to repositories), and of the audit and decision record (the hash-chained event log and the Linear history that prove everything else).
- **Cost**: denial-of-wallet through Workers CPU, AI Gateway spend and runaway agents [charter 10, 14].

## Attacker profiles

- **A1 Anonymous internet user**: hits public Workers and web-app routes, Previews and forms.
- **A2 Malicious content / prompt injection**: crafted email, web page, document or repository content that reaches an agent or model, or a local tool that consumes it [charter 14].
- **A3 Authenticated user of another tenant / lower role**: escalation or cross-tenant access [charter D13, 14].
- **A4 Supply-chain attacker**: compromised dependency, brand-new package version, GitHub Action, install script, or a poisoned MCP server or agent skill (agent tooling is supply chain too) [charter 14].
- **A5 Local user on a shared machine**: relevant to the local tools; in scope (lab owner).
- **A6 Compromised owner credential**: a phished, stolen, or replayed owner identity (GitHub, Cloudflare, Linear, the developer machine's session). Distinct from A5 because it needs no local access and inherits the owner's full reach across every boundary. In a one-person lab this is the highest-leverage profile: it subsumes most others. Added revision 2.
- Not modeled: the lab owner acting with intent; physical access. Scenario chains are expected (A2 content steers an agent into doing A6-level damage; A4 tooling delivers A6 capability).

## Threat enumeration (per boundary)

Coverage check: every boundary and every attacker profile maps to at least one threat and one control; a threat with no control is a gap and joins the "Controls vs. reality" table.

| Boundary                    | Threat                                                                                  | Attacker | Primary control [charter unless noted]                                                                                                                                                       |
| --------------------------- | --------------------------------------------------------------------------------------- | -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Public internet to edge     | Unauthenticated abuse of public endpoints; denial-of-wallet                             | A1       | WAF rate limits, Turnstile with Siteverify, `limits.cpu_ms`, Access on every non-public surface                                                                                              |
| Public internet to edge     | Preview URL exposes real data or production bindings                                    | A1       | Access on all Previews; no production bindings in the `previews` block                                                                                                                       |
| Identity planes             | Cross-tenant read or privilege escalation                                               | A3       | Per-tenant DO or database (structural isolation); server-side authorization; IdP-first, no shared credential database (LAB-ADR-0001)                                                         |
| Repository to secret stores | Secret committed to git or leaked into agent context                                    | A4, A6   | Repositories hold references never values; gitleaks pre-push and CI; scoped tokens; no secrets in prompts or sandboxes                                                                       |
| Repository to secret stores | PII committed to a documentation repo (including git history)                           | A2, A6   | `{{PII:<TOKEN>}}` placeholders, committed registry plus gitignored overlay, sentinel scans on edit and pre-publish, history scrub rules (PII-IN-REPOSITORIES)                                |
| Untrusted content to agents | Prompt injection redirects an agent or exfiltrates via a tool                           | A2       | Envelope separation of instructions and data; per-agent tool allow-lists; human approval for side effects; injection eval suites                                                             |
| Untrusted content to agents | Memory and context poisoning: untrusted or inferred facts silently become authoritative | A2       | Trust levels and provenance on every fact; quarantine until human-promoted; invalidation preserves history instead of overwriting                                                            |
| Untrusted content to agents | Automated access violates a source's terms (complaint, block, legal exposure)           | A1, A2   | One source card per source: dated terms status (allowed, restricted, forbidden, unknown); forbidden means import-only; per-source rate limits and kill switches                              |
| Developer machine           | Local user or shared-machine read of the `config/` store and agent configs              | A5       | SOPS+age ciphertext at rest; age key in OS keychain; least-privilege agent scopes                                                                                                            |
| Developer machine           | Owner credential compromised off-machine (phishing, token theft, replay)                | A6       | Gap: no dedicated control today; least-privilege scoping limits blast radius; see open proposals below                                                                                       |
| Supply chain                | Compromised or brand-new dependency, Action, MCP server, or skill                       | A4       | 7-day minimum release age; lockfiles; pre-push review; license gate (permissive in-process, copyleft as separate services, `THIRD_PARTY.md`); pinned MCP and skill digests with review notes |
| Agents and tooling          | Rogue or runaway agent: unbounded actions or spend                                      | A2, A4   | Bounded steps, seconds, and spend per agent; kill switches; per-agent tokens at least privilege; approval gates for side effects                                                             |
| Agents and tooling          | Audit tampering: events altered, dropped, or forged                                     | A3, A6   | CloudEvents envelope; per-tenant hash chain verified on schedule; digests not payloads; "no event, no result" write rule (Appendix B)                                                        |

## Existing controls [charter]

- Cloudflare Access on every Preview, internal tool and admin surface; no production bindings in Previews.
- Secrets Store / Worker secrets; scoped per-Worker tokens; secret scanning before push; no secrets in agent context or sandboxes.
- AI Gateway for every model call (rate limits, spend limits, Guardrails, DLP); no direct provider endpoints in code.
- Dynamic Workers / Containers with egress disabled for generated code.
- MCP least-privilege scopes and per-agent tool allow-lists; human approval for client-visible output, spend, deletion and production deploys.
- `limits.cpu_ms`, WAF rate limits, Turnstile with server-side Siteverify.
- Lab rule: prefer package versions published at least 7 days ago; lockfiles; pnpm fleet governance (tool-pnpm-fleet).
- Source cards govern third-party access: a dated terms status, rate limit, and trust level per source; forbidden sources are import-only (Section 10, revision 3d).
- License gate: in-process dependencies restricted to permissive licenses; copyleft and source-available components run as separate services; every third-party component listed in `THIRD_PARTY.md`; model licenses recorded per model (Section 10, revision 3d).
- Audit events are tamper-evident: CloudEvents envelope, per-tenant hash chain verified on schedule, payloads referenced by digest, synchronous "no event, no result" writes (Appendix B, revision 3d).
- PII in committed documentation is placeholdered, not embedded: `{{PII:<TOKEN>}}` tokens, a committed registry plus gitignored overlay, sentinel scans on edit and pre-publish, history scrub treated as assume-breached (`md/PII-IN-REPOSITORIES.md`, revision 3d).
- Facts from untrusted content and agent inference are quarantined with trust level and provenance until human-promoted (Section 14, revision 3d).

Note: these are the charter's **intended** controls. A finding showing that a repository does not implement one of them is a real finding, not a duplicate of the control.

## Out of scope / accepted risk

- None (lab owner). Stale product repositories and demo apps with no real users are in scope and ranked normally.

## Lab owner decisions (2026-10-27)

- Secrets/tokens, PII, client/tenant data and cost (denial-of-wallet) all matter equally; none is ranked above the others.
- Stale product repositories (documind, health-pulse, healthcare-ai-validation, drug-admet-predictor, virtual-screener, sentiment-analyzer): in scope, ranked normally.
- Shared multi-user machines (A5) are in scope for the local tools.
- Demo apps (e.g. voice-quote, ai-starter-kits): ranked normally.
- Every attacker profile A1-A5 matters, so scanner severities generally stand.
- (Revision 2 note: A6 was added after this decision; its severity mapping is an open proposal pending owner confirmation.)

## Controls vs. reality (refinement input)

Gaps where security scans show a repository does not yet implement a [charter] control. Each row becomes a refinement item or an ADR in the implementing repository.

Row lifecycle (added revision 2): every row carries a status (`open`, `fixed`, `accepted`) and a date. A row is `fixed` when the remediation lands in an ADR or a merged change; fixed rows move to a resolved subsection (or are removed) on the next revision so this table always reads as current exposure, never as history. Rows untouched for 90 days are re-verified during the DEV-109 refinement passes.

| Repository               | Charter control                                                                | Observed gap                                                                                                                                                                         | Source                         | Status |
| ------------------------ | ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------ | ------ |
| tool-universal-ai-config | Supply chain: versions published at least 7 days ago, lockfiles (Section 14)   | `install.sh` downloads the unpinned `main` branch tarball with no checksum                                                                                                           | threat model review 2026-10-27 | open   |
| tool-universal-ai-config | Untrusted content does not steer agents; MCP least privilege (Sections 10, 14) | `sync --project` writes MCP servers, rules and skills from a cloned repo's `.ai/` into provider configs, and writes through symlinks outside the project root (scan finding, medium) | security scan 2026-10-27       | open   |
| tool-universal-ai-config | Secrets never in repositories (CONFIG-AND-SECRETS principle 1)                 | `sync --project` can copy global MCP server secrets into project-level provider files (scan finding, low)                                                                            | security scan 2026-10-27       | open   |
| lab-wide                 | Owner credential compromise has no dedicated control (A6)                      | Owner identity grants full reach across every boundary; mitigations today are incidental (token scoping, audit trail)                                                                | threat enumeration 2026-10-28  | open   |

## Open proposals (pending owner decision)

Items that need the same explicit confirmation the 2026-10-27 decisions received:

1. **Owner-credential hardening control for A6.** Candidate: hardware security key on GitHub, Cloudflare, and Google; short-lived tokens for CI and agents; a quarterly review of sessions and authorized apps; documented recovery path. Accept, modify, or reject.
2. **Findings ranked by blast radius.** Equal asset weighting governs whether a finding counts (owner decision, unchanged). A blast-radius estimate per finding would order the remediation queue without reopening the weighting question. Accept or reject.
3. **Owner device attestation.** The `config/` store and `~/.agents/` give one machine lab-wide reach; full-disk encryption and a screen-lock policy verified periodically. Accept or reject.
