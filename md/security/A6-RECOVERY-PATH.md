# A6-RECOVERY-PATH - owner credential recovery runbook (draft)

Status: draft, 2026-10-28. Implements the "documented recovery path" item of DEV-120 (A6 owner-credential hardening, LAB-THREAT-MODEL revision 3). Entries marked **[proposed]** await owner confirmation; entries marked **[open]** need owner input before this doc can land.

## Scope

Recovery paths for loss or compromise of the owner identity on: GitHub, Cloudflare, Google, Linear, and the developer machine's session. One-person lab: no second admin exists, so every path assumes solo recovery.

## Prevention baseline (from the accepted control)

- Two hardware security keys registered per provider where supported: primary (on person) and backup (stored off-site) **[proposed]**.
- Recovery codes generated at 2FA enrollment, stored in SOPS+age per CONFIG-AND-SECRETS, with an offline printed copy **[proposed]**.
- A recovery email and phone on the Google account that are not themselves protected only by the same hardware key **[proposed]**.
- GitHub fallback: SSH key + `gh` OAuth token on the machine; do not rely on these as the sole recovery path since a stolen machine kills both **[proposed]**.

## Recovery procedures

### Lost hardware key (account still accessible)

1. Sign in with the backup key or a stored recovery code.
2. Remove the lost key from the provider's security-key list.
3. Register a replacement key; restore the two-key baseline.
4. Record the event in Linear (comment on the current monthly review issue, DEV-121 recurrence).

### Lost all hardware keys, recovery codes available

1. Sign in with a recovery code.
2. Rotate all active sessions (provider "sign out all sessions").
3. Register new keys; regenerate and re-store recovery codes.

### Lost all keys and recovery codes (full lockout)

- **GitHub**: account recovery via recovery email; support fallback requires verified account details **[proposed]**.
- **Cloudflare**: recovery via email on the account; if email unreachable, Cloudflare support verification flow **[proposed]**.
- **Google**: accounts.google.com recovery flow: recovery email, recovery phone, device-based verification **[proposed]**.
- **Linear**: single-owner workspace; recovery via workspace email reset; contact Linear support if email also lost **[proposed]**.

### Key suspected stolen or credential replayed (compromise, not just loss)

1. Treat as an incident, not a lockout. Sign out all sessions on every provider from a known-clean device.
2. Revoke all OAuth apps, API tokens, PATs and SSH keys; re-issue only what is needed, short-lived where supported.
3. Rotate SOPS+age-encrypted secrets reachable through the compromised identity (CONFIG-AND-SECRETS Section 8 rotation procedure).
4. Review audit trail (hash-chained event log, GitHub audit log, Cloudflare audit log, Linear history) for actions taken during the compromise window.
5. Post-incident note on DEV-120 or a new incident issue.

## Decisions needed from owner **[open]**

1. Backup key storage location (off-site definition).
2. Where recovery codes live: SOPS+age only, or also an offline/printed copy.
3. Recovery email and phone for Google; recovery email for GitHub and Cloudflare.
4. Whether the monthly review issue (DEV-121) doubles as the log of key-loss events, or a dedicated incident label is preferred.
