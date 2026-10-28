# A6-RECOVERY-PATH - owner credential recovery runbook

Status: final, 2026-10-28. Implements the "documented recovery path" item of DEV-120 (A6 owner-credential hardening, LAB-THREAT-MODEL revision 3). Owner decisions recorded at the bottom.

## Scope

Recovery paths for loss or compromise of the owner identity on: GitHub, Cloudflare, Google, Linear, and the developer machine's session. One-person lab: no second admin exists, so every path assumes solo recovery.

## Prevention baseline (from the accepted control)

- Two hardware security keys registered per provider where supported: primary (on person) and backup (stored at home, in a separate location from the primary key and the machine).
- Recovery codes generated at 2FA enrollment, stored in the password manager.
- Recovery email and phone on Google, and recovery email on GitHub and Cloudflare: the values currently on file at each provider are the accepted recovery contacts; do not "clean them up" without updating this doc.
- GitHub fallback: SSH key + `gh` OAuth token on the machine; do not rely on these as the sole recovery path since a stolen machine kills both.

## Recovery procedures

### Lost hardware key (account still accessible)

1. Sign in with the backup key or a stored recovery code.
2. Remove the lost key from the provider's security-key list.
3. Register a replacement key; restore the two-key baseline.
4. Open an incident issue in Linear (label `Incident`); record provider, date, and what was rotated.

### Lost all hardware keys, recovery codes available

1. Sign in with a recovery code from the password manager.
2. Rotate all active sessions (provider "sign out all sessions").
3. Register new keys; regenerate recovery codes and update the password manager.
4. Open an incident issue in Linear (label `Incident`).

### Lost all keys and recovery codes (full lockout)

- **GitHub**: account recovery via the on-file recovery email; support fallback requires verified account details.
- **Cloudflare**: recovery via the on-file account email; if email unreachable, Cloudflare support verification flow.
- **Google**: accounts.google.com recovery flow: on-file recovery email, recovery phone, device-based verification.
- **Linear**: single-owner workspace; recovery via workspace email reset; contact Linear support if email also lost.

Open an incident issue (label `Incident`) once access is restored.

### Key suspected stolen or credential replayed (compromise, not just loss)

1. Treat as an incident, not a lockout. Sign out all sessions on every provider from a known-clean device.
2. Revoke all OAuth apps, API tokens, PATs and SSH keys; re-issue only what is needed, short-lived where supported. `tool-github-actions-audit/token_audit.py` enumerates Actions secrets and Cloudflare tokens to drive the rotation list.
3. Rotate SOPS+age-encrypted secrets reachable through the compromised identity (CONFIG-AND-SECRETS Section 8 rotation procedure).
4. Review audit trail (hash-chained event log, GitHub audit log, Cloudflare audit log, Linear history) for actions taken during the compromise window.
5. Post-incident issue in Linear (label `Incident`) linked to DEV-120.

## Owner decisions (2026-10-28)

- Backup hardware key: stored at home, separate location from primary key and machine.
- Recovery codes: password manager.
- Recovery contacts: current on-file values at each provider; recorded as accepted, not enumerated here.
- Event logging: dedicated Linear issue per event under the `Incident` label (not folded into the monthly review issue).
