# REPO_INDEX.md - devartslab-foundation

> Single source of truth for what lives in this repository.
> Regenerate when files or folders are added, removed, or substantially rewritten.
>
> **Last updated:** 2026-10-21

## Top-level

| Path            | What it is                                                  |
| --------------- | ----------------------------------------------------------- |
| `README.md`     | Entry point: what the foundation is, how to read it, status |
| `REPO_INDEX.md` | This index                                                  |
| `.gitignore`    | Hygiene rules (env files, keys, OS junk, editor state)      |
| `md/`           | Framework documentation set (documentation only, no code)   |

## md/

| File                 | What it is                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `MEGA-PLAN.md`       | Research mega plan, revision 2: research method and fact decay policy, design principles, Cloudflare product catalog by concern, reference architecture and tiers T0-T3, local parity and environment model, delivery lifecycle as a Workflow, foundation decisions D1-D10, guardrails, data governance, cost model, limits cheat sheet, threat model, agent orchestration model, master repository scaffold, roadmap, risks, open questions, glossary, source register, ADR template, audit event catalog, start checklist |
| `PRE-DEVELOPMENT.md` | Pre-development blueprint, revision 2: artifacts per phase (research, design, architecture, agent planning, guardrails), required ADR-0001 to ADR-0010, document locations in the master layout, document outlines, exit criteria, tier-based minimum sets                                                                                                                                                                                                                                                                  |

## Not yet present (tracked as open items)

| Item          | Why it matters                                                       | Reference              |
| ------------- | -------------------------------------------------------------------- | ---------------------- |
| `LICENSE`     | Repository is intended to become public                              | MEGA-PLAN Q4           |
| `SECURITY.md` | Disclosure contact for a public repo                                 | MEGA-PLAN Section 16   |
| `AGENTS.md`   | Agent-facing conventions; required by the scaffold this repo defines | MEGA-PLAN Section 15.5 |
| Git remote    | Local-only repository; no push yet                                   | LAB_INDEX housekeeping |
