# INDEX.md - devartslab-foundation

> Single source of truth for what lives in this repository.
> Regenerate when files or folders are added, removed, or substantially rewritten.
>
> **Last updated:** 2026-10-28

## Top-level

| Path            | What it is                                                      |
| --------------- | --------------------------------------------------------------- |
| `README.md`     | Entry point: what the foundation is, how to read it, status     |
| `INDEX.md` | This index                                                      |
| `.gitignore`    | Hygiene rules (env files, keys, OS junk, editor state)          |
| `md/`           | Framework documentation set (documentation only, no code)       |
| `md/adr/`       | Lab-wide decision records (`LAB-ADR-*`) that bind every project |
| `md/security/`  | Lab-wide security baseline that every project extends           |

## md/

| File                     | What it is                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `MEGA-PLAN.md`           | Research mega plan, revision 3: research method and fact decay policy, 13 design principles, Cloudflare product catalog by concern, reference architecture and tiers T0-T3, local parity and environment model, delivery lifecycle as a Workflow, foundation decisions D1-D15, guardrails, data governance, cost model, limits cheat sheet, threat model, agent orchestration model, master repository scaffold, roadmap, risks, open questions, glossary, source register, administrative management interface, ADR template, audit event catalog, start checklist |
| `PRE-DEVELOPMENT.md`     | Pre-development blueprint, revision 3: artifacts per phase (research, design, architecture, agent planning, guardrails), required ADR-0001 to ADR-0015, reserved-block numbering rule, document locations in the master layout, document outlines, exit criteria, tier-based minimum sets                                                                                                                                                                                                                                                                           |
| `CONFIG-AND-SECRETS.md`  | Configuration, PII, and secret architecture, revision 1 (expands D14): current-state audit, four-plane model, root-anchored local store and resolution contract, protection at rest, three-tier staging and production recommendation, per-platform secret-store inventory, research strategy and open questions S1-S6                                                                                                                                                                                                                                              |
| `PII-IN-REPOSITORIES.md` | PII in committed documentation, revision 1 (expands the D14 PII rule and Section 11): three-tier enforcement model, `{{PII:<TOKEN>}}` placeholder contract, committed registry plus gitignored overlay, sentinel agent spec, evaluated detection tooling (Presidio + GLiNER, recall floors), git-history scrub rules; generalized from the waykeeper operating system                                                                                                                                                                                               |

## md/adr/

| File                                         | What it is                                                                                                                                         |
| -------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `LAB-ADR-0001-centralized-identity-layer.md` | Lab-wide: centralized identity layer, no shared auth database. Resolves MEGA-PLAN Q7 and underpins D13. Promoted from career-tracker on 2026-10-26 |

## md/security/

| File                  | What it is                                                                                                                                                                                                                                                                            |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `LAB-THREAT-MODEL.md` | Lab-wide baseline threat model, revision 2: repository classes, trust boundaries (incl. agents and tooling), assets, attacker profiles A1-A6, per-boundary threat enumeration, charter controls, owner decisions, controls-vs-reality gap register with row lifecycle, open proposals |

## Not yet present (tracked as open items)

| Item          | Why it matters                                                                | Reference              |
| ------------- | ----------------------------------------------------------------------------- | ---------------------- |
| `LICENSE`     | Repository is intended to become public                                       | MEGA-PLAN Q4           |
| `SECURITY.md` | Disclosure contact for a public repo                                          | MEGA-PLAN Section 16   |
| `AGENTS.md`   | Agent-facing conventions; required by the scaffold this repo defines          | MEGA-PLAN Section 15.5 |
| Git remote    | `DevArtsLab/devartslab-foundation` on `master`; local commits ahead of origin | LAB_INDEX housekeeping |
