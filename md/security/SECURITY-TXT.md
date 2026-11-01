# SECURITY-TXT - vulnerability disclosure file convention (RFC 9116)

Status: revision 1, 2026-11-01. Convention for `security.txt` across
`{{ORG_NAME}}` public domains. Applies to every org-controlled domain serving
public traffic.

Placeholders: `{{ORG_NAME}}`, `{{ORG_DOMAIN}}`, `{{SECURITY_EMAIL}}`,
`{{SITE_REPO}}`. Values resolve from `config/identity/<slug>.json` (D12
manifest), never hardcoded per repo. Current instantiation:
`config/identity/devartslab.json`.

## Convention

Each public domain serves `/.well-known/security.txt` (RFC 9116) as
`text/plain` with, at minimum:

```text
Contact: mailto:{{SECURITY_EMAIL}}
Expires: <ISO-8601, max 1 year ahead>
Preferred-Languages: en
```

Rules:

- `Expires` is required; stale files are treated as absent by scanners. Renew
  annually; current file expires 2027-11-01.
- `Contact` must be a monitored mailbox. `security@{{ORG_DOMAIN}}` is the
  convention; verify the mailbox exists before publishing a new domain.
- Serve inline from the edge Worker (no asset dependency) so path handling
  stays explicit. Pattern: branch on `/.well-known/security.txt` before the
  catch-all page response.
  Current implementation: `{{SITE_REPO}}` `src/index.ts`, `securityTxt()`.
- New public domains adopt this file at launch; add the same branch in that
  Worker's fetch handler.

## Verification

```bash
curl -s https://{{ORG_DOMAIN}}/.well-known/security.txt
```

Must return `text/plain` body, not the site HTML (SPA catch-alls produce false
positives on a bare status check).
