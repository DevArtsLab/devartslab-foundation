# LAB-THREAT-MODEL - DevArts Lab baseline threat model (all repositories)

Status: revision 1, baseline confirmed by the lab owner 2026-10-27. Applies to every DevArtsLab repository. This file is the source of truth; the Devin security scan profile "DevArtsLab org baseline threat model" is a copy and is re-synced when this file changes. Facts marked **[charter]** come from this repository: `md/MEGA-PLAN.md` (Sections 4.9, 10, 14, D9, D13, D15), `md/CONFIG-AND-SECRETS.md` (Sections 3, 7.3, 8) and `md/adr/LAB-ADR-0001-centralized-identity-layer.md`. The repository inventory comes from `DevArtsLab/devartslab` `LAB_INDEX.md`. Everything else was inferred from repository descriptions.

Per-repository convention: each repository extends this baseline in its own `md/security/threat-model.md` (MEGA-PLAN Section 14, PRE-DEVELOPMENT Section 8.6), and its system-specific text wins where the two differ. Example: `DevArtsLab/tool-universal-ai-config` `md/security/threat-model.md`.

## System overview
DevArts Lab is a small lab running many repositories. The platform doctrine is Cloudflare-native in production, local-first in development, and agent-orchestrated across delivery [charter]. Production surfaces are Cloudflare Workers on lab domains: `devartslab.com` (devartslab-site), `docs.devartslab.com` (devartslab-notion proxy), `data.devartslab.com` (tool-github-export Worker + R2), `mail.devartslab.com` (devarts-mail) and `voice-quote.devartslab.com` (voice-quote-proxy). Netlify hosts voice-quote, and Vercel is also used [LAB_INDEX]. Many other repositories are local CLIs, agent tooling, docs, demos or stale ML/product prototypes.

## Components (repository classes)
| Name | Covers | Examples |
|---|---|---|
| **Edge services** | Cloudflare Workers, Pages, D1/R2/KV/DO bindings, `wrangler.*` | devartslab-site, devartslab-notion, devarts-mail, voice-quote-proxy, tool-github-export (Worker) |
| **Web apps** | Next.js / Angular / React apps with server routes or APIs | career-tracker, voice-quote, city-scrapers-meetings-viewer, guaranteed-income-program, ai-starter-kits, health-pulse (web) |
| **Local tools & CLIs** | CLIs and scripts run on the developer's machine, installers, Homebrew tap | tool-universal-ai-config, tool-github-actions-audit, tool-pnpm-fleet, tool-readme-links, tool-airtable-export, saystop, sesame-voice-escort, homebrew-tap |
| **Agent tooling** | Skills, prompts, plugins, agent configs, MCP wiring, scheduled agents | agent-skills, agent-research-kit, devin-plugin-devartslab, sesame-link-runbook, org-intelligence-revenue-system (n8n, AI) |
| **ML & data APIs** | Python model-serving APIs and pipelines | drug-admet-predictor, virtual-screener-ai-drug-discovery, sentiment-analyzer-ml-pipeline, documind (Spring Boot + AWS CDK) |
| **CI & release** | `.github/workflows/*`, publish pipelines, GitHub Actions in every repository | all repositories |
| **Docs-only** | Repositories with no executable code | devartslab-foundation, devarts-business, waykeeper, healthcare-ai-validation, american-community-network |

## Trust boundaries
1. **Public internet to edge services / web apps**: any anonymous caller. Every Preview URL, internal tool and admin surface is meant to sit behind Cloudflare Access [charter D9].
2. **End user / admin / machine identity planes**: kept separate. End-user auth is IdP-first with no shared credential database, admin access goes through Cloudflare Access, and machines use scoped API tokens and OAuth MCP [charter D13, LAB-ADR-0001]. Authorization is server-side only, and tenant isolation is structural (a per-tenant DO or database) [charter D13].
3. **Repository to secret stores**: repositories hold secret references, never values. One store per plane: SOPS+age as the source of truth, synced one way to Worker secrets / Secrets Store, GitHub Actions secrets and Devin secrets [charter CONFIG-AND-SECRETS 3, 7.3].
4. **Untrusted content to agents/models**: web pages, documents, emails and repository content fed to a model [charter 14].
5. **Developer machine**: local tools run as the user, with access to `~/.agents/`, provider configs, MCP tokens and the `config/` PII store (local, gitignored) [LAB_INDEX].
6. **Supply chain**: npm/pnpm, PyPI, GitHub Actions, Homebrew tap, install scripts.

## Assets (lab owner: all weighted equally)
- **Secrets and tokens**: Cloudflare API tokens, Secrets Store, GitHub tokens, MCP/OAuth tokens (Gmail, Google Calendar, Linear, Slack, Sentry, Vercel), AI-provider keys.
- **PII**: never committed anywhere, including private repositories [charter CONFIG-AND-SECRETS principle 5]. Examples: career-tracker job-search data (Gmail/Calendar/LinkedIn ingestion), devarts-mail mailboxes, health-pulse health data, nonprofit donor data (org-intelligence-revenue-system).
- **Client and tenant data**: no cross-tenant leakage [charter 14].
- **Integrity of lab domains and Workers**, and of agent configuration and skills (they steer agents that have write access to repositories).
- **Cost**: denial-of-wallet through Workers CPU, AI Gateway spend and runaway agents [charter 10, 14].

## Attacker profiles
- **A1 Anonymous internet user**: hits public Workers and web-app routes, Previews and forms.
- **A2 Malicious content / prompt injection**: crafted email, web page, document or repository content that reaches an agent or model, or a local tool that consumes it [charter 14].
- **A3 Authenticated user of another tenant / lower role**: escalation or cross-tenant access [charter D13, 14].
- **A4 Supply-chain attacker**: compromised dependency, brand-new package version, GitHub Action or install script [charter 14].
- **A5 Local user on a shared machine**: relevant to the local tools; in scope (lab owner).
- Not modeled: the lab owner; physical access.

## Existing controls [charter]
- Cloudflare Access on every Preview, internal tool and admin surface; no production bindings in Previews.
- Secrets Store / Worker secrets; scoped per-Worker tokens; secret scanning before push; no secrets in agent context or sandboxes.
- AI Gateway for every model call (rate limits, spend limits, Guardrails, DLP); no direct provider endpoints in code.
- Dynamic Workers / Containers with egress disabled for generated code.
- MCP least-privilege scopes and per-agent tool allow-lists; human approval for client-visible output, spend, deletion and production deploys.
- `limits.cpu_ms`, WAF rate limits, Turnstile with server-side Siteverify.
- Lab rule: prefer package versions published at least 7 days ago; lockfiles; pnpm fleet governance (tool-pnpm-fleet).

Note: these are the charter's **intended** controls. A finding showing that a repository does not implement one of them is a real finding, not a duplicate of the control.

## Out of scope / accepted risk
- None (lab owner). Stale product repositories and demo apps with no real users are in scope and ranked normally.

## Lab owner decisions (2026-10-27)
- Secrets/tokens, PII, client/tenant data and cost (denial-of-wallet) all matter equally; none is ranked above the others.
- Stale product repositories (documind, health-pulse, healthcare-ai-validation, drug-admet-predictor, virtual-screener, sentiment-analyzer): in scope, ranked normally.
- Shared multi-user machines (A5) are in scope for the local tools.
- Demo apps (e.g. voice-quote, ai-starter-kits): ranked normally.
- Every attacker profile A1-A5 matters, so scanner severities generally stand.

## Controls vs. reality (refinement input)

Gaps where security scans show a repository does not yet implement a [charter] control. Each row becomes a refinement item or an ADR in the implementing repository.

| Repository | Charter control | Observed gap | Source |
| --- | --- | --- | --- |
| tool-universal-ai-config | Supply chain: versions published at least 7 days ago, lockfiles (Section 14) | `install.sh` downloads the unpinned `main` branch tarball with no checksum | threat model review 2026-10-27 |
| tool-universal-ai-config | Untrusted content does not steer agents; MCP least privilege (Sections 10, 14) | `sync --project` writes MCP servers, rules and skills from a cloned repo's `.ai/` into provider configs, and writes through symlinks outside the project root (scan finding, medium) | security scan 2026-10-27 |
| tool-universal-ai-config | Secrets never in repositories (CONFIG-AND-SECRETS principle 1) | `sync --project` can copy global MCP server secrets into project-level provider files (scan finding, low) | security scan 2026-10-27 |
