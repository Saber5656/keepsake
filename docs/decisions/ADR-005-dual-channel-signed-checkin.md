# ADR-005: Dual-channel check-in (signed CLI + workflow_dispatch) with server-side verification

Status: Accepted · 2026-07-10 · Confirmed with owner

## Context

Check-in ergonomics directly control the false-fire rate (hospital stays, travel), while check-in **authenticity** controls the forged-liveness attack (an attacker indefinitely suppressing release) — DESIGN §10.5 A1/A2.

## Decision

Two independent check-in channels, both verified by the monitor:

1. **CLI (`keepsake checkin`)**: a JSON statement `{version, counter, timestamp, note}` signed as an **SSH signature** (SSHSIG, namespace `keepsake-checkin`), committed and pushed to the trigger repo. Verified against `allowed_signers` in config; replay-proof via monotonic counter + timestamp window.
2. **Phone (`checkin-button.yml` workflow_dispatch)**: tapping "Run workflow" in the GitHub mobile app/web. The monitor verifies these via the **Actions API run records** (`triggering_actor == owner.github_login`) — server-side facts that cannot be forged by pushing files, so the button workflow itself needs **zero permissions and zero logic**.

`last_valid_checkin = max(valid signed statement, valid dispatch run)`.

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| Email-link check-in | needs an inbound endpoint + token-in-link is replayable by mailbox observers (forged-liveness risk) |
| Unsigned commit as check-in | stolen push credential (PAT in some tool) forges liveness silently |
| Dispatch events recorded via committed files | commits are forgeable with write access; the Actions API record is not |
| GPG signatures | GPG toolchain dependency; owners already have SSH keys; SSHSIG verifiable with `golang.org/x/crypto/ssh` |

## Consequences

- SSH signing key ≠ security boundary against full account compromise (that is A6, mitigated elsewhere); it defends the narrower stolen-push-credential case and provides tamper-evident history.
- Lost signing key is a documented runbook (F9): rotate `allowed_signers`, `trigger update`, push.
- Monitor needs `actions: read`; button workflow needs `permissions: {}`.
