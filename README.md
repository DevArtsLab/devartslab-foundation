# devartslab-foundation

The research base and architectural charter for every DevArts Lab
repository and every client application the lab delivers: Cloudflare-native
in production, local-first in development, agent-orchestrated across the
delivery lifecycle.

This repository is documentation only. It contains no code, templates, or
configuration; those will live in the scaffold generator described in the
roadmap.

## Read in this order

1. [md/MEGA-PLAN.md](md/MEGA-PLAN.md): platform research, design
   principles, product catalog by concern, project tiers, foundation
   decisions D1 to D10, guardrails, governance, cost and limits, threat
   model, agent orchestration, master repository layout, roadmap.
2. [md/PRE-DEVELOPMENT.md](md/PRE-DEVELOPMENT.md): what a project must
   produce before code, with document outlines and exit criteria.
3. [md/CONFIG-AND-SECRETS.md](md/CONFIG-AND-SECRETS.md): where
   configuration, PII, and secrets live across local, staging, production,
   and distributed products, with the per-platform store inventory.
4. [md/security/LAB-THREAT-MODEL.md](md/security/LAB-THREAT-MODEL.md):
   lab-wide baseline threat model (repository classes, trust boundaries,
   assets, attackers, owner decisions, controls vs. reality) that every
   repository extends in its own `md/security/threat-model.md`.
5. [md/PII-IN-REPOSITORIES.md](md/PII-IN-REPOSITORIES.md): how committed
   content references personal data without containing it: placeholders,
   the registry and overlay, the sentinel agent, detection tooling, and
   the git-history rules.
6. [REPO_INDEX.md](REPO_INDEX.md): what lives where in this repository.

## Status

- Revision 3 (reuse and control edition), facts verified 2026-10-21 and
  2026-10-25 against developers.cloudflare.com and the vendor docs in the
  source registers. Quantitative claims decay; see MEGA-PLAN Section 2.2
  for the re-verification policy and Section 22 for sources.
- License: to be decided before this repository is published (MEGA-PLAN
  open question Q4).
- Work is tracked in Linear (DevArts Lab workspace).

## Conventions

- ASCII punctuation only.
- Product names follow Cloudflare's current naming; older names are mapped
  in the MEGA-PLAN glossary.
- Every decision that deviates from the plan is recorded as an ADR in the
  implementing repository.
