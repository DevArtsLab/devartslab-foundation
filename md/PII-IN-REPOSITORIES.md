# PII in Repositories: Placeholders, Registries, and Sentinels

> Companion to [MEGA-PLAN.md](MEGA-PLAN.md), expanding the PII rule in D14
> and Section 11. Covers how committed content references personal data
> without ever containing it, and how that discipline is enforced.
> Documentation only; no implementation.

| Field        | Value                                                                                           |
| ------------ | ----------------------------------------------------------------------------------------------- |
| Status       | Living document, revision 1                                                                     |
| Audience     | Lab engineers, agents, future operators under a different brand                                 |
| Scope        | PII in committed documentation and research content: references, registries, detection, history |
| Not in scope | Secrets and credentials (CONFIG-AND-SECRETS); datastore classification per project (Section 11) |
| Source       | Working system in the waykeeper repository, operating since 2026-10-04; generalized here        |

---

## Table of contents

1. Why this document exists
2. The three tiers of PII enforcement
3. The placeholder contract
4. Entity types and the allowlist
5. The sentinel agent
6. Detection tooling
7. Git history is the hole in the rule
8. When to apply which parts
9. Open questions

---

## 1. Why this document exists

D14 states the rule: PII is never committed to any repository, including
private and remote-less ones, because a remote can always be added later.
That rule alone is not a mechanism, and research reality works against it:
research and design documents routinely need to discuss real people. Owner
fields name the operator. Study sources name personal accounts and
profiles. Intake notes carry contact details. A rule that says "never"
without saying "instead do this" produces either docs that avoid reality or
quiet violations.

The waykeeper research corpus hit this on day one, because it was designed
to be publishable on the lab documentation platform from the first commit.
The mechanism it built and has been operating since 2026-10-04 is the same
principle the lab already applies to secrets, extended to personal data:
**repositories hold references, never values.** Committed content carries
placeholders; a committed registry describes each placeholder; a gitignored
overlay maps placeholders to real values held in the operator-owned store
described in [CONFIG-AND-SECRETS.md](CONFIG-AND-SECRETS.md) Section 4.

This document records that mechanism as a lab convention, so every
repository intended for the documentation platform, or for public
visibility at any point in its life, inherits it instead of reinventing it.

## 2. The three tiers of PII enforcement

"No PII in the app" is enforced at three tiers, each with its own
mechanism. The model comes from waykeeper `10-security-and-privacy.md`,
generalized.

| Tier              | Guarantee                                                          | Enforcement                                                                                                                                                                    |
| ----------------- | ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| T1 Repository     | Zero PII and zero secrets in git, ever                             | Secret scanning pre-commit and in CI; PII scanning of committed text; the placeholder contract (Section 3); public-by-default posture makes the discipline real                |
| T2 Datastore      | Only scrubbed derivatives and placeholders; no column may hold PII | A firewall step on every ingest path; schema rules; scheduled sweeps over the datastore with alerts; per-tenant isolation (D1, D3)                                             |
| T3 Model boundary | Providers only ever see placeholders                               | A firewall pass on every prompt and tool result before the model call; zero-data-retention provider settings where offered; local inference for the most sensitive steps (D15) |

This document specifies T1 in full and the prompt side of T3. T2 is
declared per project in `md/governance/classification.md` and the project
threat model (Section 14), because the datastore shape varies.

## 3. The placeholder contract

Committed content that would otherwise carry a personal value carries a
placeholder instead: `{{PII:<TOKEN>}}`. Tokens are uppercase, stable, and
impersonal; the grammar is `<SUBJECT>_<KIND>` (for example
`OPERATOR_ZERO_GIVEN_NAME`, `OPERATOR_ZERO_GITHUB`). A token names a role
in the system, never the person, so the registry itself carries no PII.

Three files make the contract work, following the waykeeper layout:

| File                             | Tracked?    | Holds                                                                                                                                                         |
| -------------------------------- | ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `config/pii-registry.yaml`       | Committed   | One entry per token: entity type, masked hint (first character plus asterisks, never more), status, every `file:line` redacted from, notes. Never a raw value |
| `config/pii-registry.local.yaml` | Gitignored  | Token-to-field map into the master store; a literal value only as an explicit, commented override                                                             |
| The master store                 | Outside git | `config/profiles/` in the workspace-root config store (CONFIG-AND-SECRETS Section 4); the single source of truth for real values                              |

Output contract, observed in waykeeper and adopted verbatim:

1. The registry is updated before the content that references it; content
   must never carry a token that is not registered.
2. Every redaction is reversible locally through the overlay; nothing is
   destructively rewritten in a way the operator cannot inspect.
3. Ambiguous cases are flagged for a human decision, never silently kept
   and never silently removed.

Publicly rendered material keeps the placeholder visible. Readers of the
published corpus see `{{PII:OPERATOR_ZERO_GITHUB}}`, which is informative
(the slot exists, the kind is known) without being identifying.

## 4. Entity types and the allowlist

| Entity type    | Examples                                           |
| -------------- | -------------------------------------------------- |
| PERSON         | Names of natural persons, including the operator   |
| EMAIL          | Personal email addresses                           |
| PHONE          | Personal phone numbers                             |
| ADDRESS        | Street addresses, home locations                   |
| ID             | Identity, license, claim, and account numbers      |
| ACCOUNT_HANDLE | Personal GitHub or social handles and profile URLs |
| LOCATION       | Precise personal whereabouts                       |
| FINANCIAL      | Personal bank, card, or benefit account details    |

Allowlist, logged under `reviewed_not_pii` rather than redacted: employer
names, product names, public institution names and their public contact
channels (a government office phone number is a public contact), and the
lab's own organization identity. Recording allowlisted near-misses in the
registry stops repeat findings and makes the judgment auditable.

Two subtleties observed in practice:

- An organization handle is not PII; a personal handle on the same
  platform is. The registry records which is which so agents do not
  re-litigate it per scan.
- Sometimes the right action is rewriting, not tokenizing: when a personal
  handle appeared in a repository path, the fix was moving the repository
  under the lab org rather than placeholdering the path.

## 5. The sentinel agent

Detection is a standing job, not a one-time scrub. Each repo that adopts
this convention runs a sentinel agent with a written spec, following the
serial-code convention in Section 15.5 (`SCAN-###` or the repo's stage
prefix; waykeeper uses `RESEARCH-001`, spec at
`waykeeper/research/RESEARCH-001.md`). The spec file carries:

- Mission: the guarantee it enforces, in one paragraph.
- Triggers: a new committed text file, a material edit to one, a
  pre-publish gate (Section 16.2 readiness), a manual sweep request.
- The entity table and allowlist (Section 4).
- The procedure: scan, assign or reuse tokens, register, replace, update
  the overlay, log the run.
- The output contract (Section 3).
- A run log: every sweep with date, scope, findings, and actions.

The sentinel reports to a human: a findings table (file:line, type,
action) plus anything ambiguous. It never expands its own authority; a
finding it cannot classify is escalated, not resolved.

Registry-of-agents note: repos running more than one such agent keep a
status table (proposed, active, paused, done, retired) in the same
directory as the specs, updated on every run or scope change. Waykeeper's
`research/AGENTS.md` is the working example.

## 6. Detection tooling

Tooling evaluated and adopted in waykeeper; facts here decay per
MEGA-PLAN Section 2.2 and must be re-verified at adoption time.

| Layer      | Tool                                                                                                | Notes                                                                                                                                      |
| ---------- | --------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Secrets    | gitleaks, pre-commit and CI                                                                         | Already the lab convention (Principle 7, `devartslab-code-review` skill)                                                                   |
| PII        | Presidio analyzer and anonymizer (MIT) with a GLiNER recognizer plus regex and checksum recognizers | `nvidia/gliner-PII` (NVIDIA Open Model License; verify terms before redistributing) with Apache-2.0 fallback `urchade/gliner_multi_pii-v1` |
| Evaluation | Labeled corpus: synthetic fixtures plus the operator's own redacted archives, with consent          | Build fails if recall drops below the configured floor: 0.99 for structured types (email, phone, ID), 0.95 for names                       |

Rejected options, with the deciding requirement:

- LLM-only redaction: non-deterministic; a detector that cannot be
  regression-tested against a recall floor is not a control.
- Cloud DLP APIs: the PII would leave the workspace to be detected, which
  defeats T1.

Every detection pass logs counts by entity type, never values. A scan
result is itself an audit event (Appendix B, `pii.redacted`).

## 7. Git history is the hole in the rule

The waykeeper sentinel's most important finding was not in the working
tree. A repo can pass every worktree scan and still carry raw PII in pushed
commit blobs; one `git remote add` later turns that latent exposure into a
permanent one, because history preserves it and force-pushing does not
reach forks or clones.

Operating rules adopted from that incident:

1. Pre-publish gates scan the full history, not just the worktree. The
   Section 16.2 readiness transition includes a history pass.
2. Scrubbing uses `git filter-repo --replace-text` followed by a force
   push, and the scrub is recorded in the sentinel run log.
3. A scrubbed value is still treated as exposed. For credential-class
   material the answer is rotation, not scrubbing (CONFIG-AND-SECRETS
   remediation sequence). For personal data, scrub plus assume-breached is
   the honest posture.
4. The real fix is sequence: scan on add and on edit from the first
   commit, and flip repository visibility only after a clean history pass,
   so the scrub is never needed.

## 8. When to apply which parts

| Repository situation                                    | Apply                                                                                             |
| ------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| Every lab repository                                    | Secret scanning; the "references, never values" rule for PII                                      |
| Repos intended for the documentation platform or public | The full placeholder contract: registry, overlay, sentinel spec, pre-publish history gate         |
| Products that store or process personal data            | Plus T2 and T3 from Section 2, declared in the project's classification register and threat model |
| A repo that has already pushed raw PII                  | Section 7 sequence: history scrub, rotation where applicable, incident note                       |

The placeholder contract costs little in an internal repo and is the only
version that scales safely, which is why it is the default rather than an
opt-in.

## 9. Open questions

| ID  | Question                                                                                                                | Needed by                        |
| --- | ----------------------------------------------------------------------------------------------------------------------- | -------------------------------- |
| P1  | Does the docs platform render placeholders at publish (resolving against the master store) or publish tokens literally? | Documentation platform build     |
| P2  | Is the sentinel distributed as a lab skill (`devartslab-pii-sentinel`) so repos adopt it without copying the spec?      | Next repo that adopts T1 in full |
| P3  | Does the recall-floor eval harness live once in the scaffold, or per repo?                                              | Scaffold generator (Phase 5)     |
