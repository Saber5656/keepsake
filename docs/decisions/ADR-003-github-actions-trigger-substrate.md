# ADR-003: GitHub Actions private repo as the dead man's switch substrate

Status: Accepted · 2026-07-10 · Confirmed with owner

## Context

The trigger must keep evaluating liveness and be able to send mail **after the owner stops maintaining anything** — including after death. Self-hosted options die with the owner (research 01 "self-hosting paradox"); paid subscriptions lapse (Bitwarden paradox). See research 03 for verified platform constraints.

## Decision

- The switch runs as a **daily scheduled workflow in a private GitHub repository** on the free plan (no *payment* event tied to owner survival; the monthly free-minutes quota resets automatically, and quota exhaustion by other repos only delays runs until reset — failure mode F15).
- The `keepsake` **linux-amd64 binary is vendored into the trigger repo** (committed, with `.sha256` verified in-workflow before execution). No build step, no runtime download — release must not depend on the OSS repo's availability years later.
- The monitor **commits a state heartbeat every run**, which simultaneously provides idempotency records, an audit trail, and keepalive against the 60-day scheduled-workflow disablement (which officially applies to public repos only — the trigger repo is private; heartbeat is defense-in-depth against policy drift).
- The state machine is **day-granular and idempotent** because Actions cron is best-effort (delays/skips absorbed by recomputation).
- GitHub is trusted with: `share_G`, config, recipient PII, liveness state. It can never obtain ciphertext or `share_R` (DESIGN §10.2), so platform compromise ≠ content compromise.

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| Home server / VPS cron | dies before or with the owner (power, billing, hardware) |
| Hosted DMS SaaS | operator trust + company-lifetime risk |
| GitLab CI / other CI | viable; deferred as v2 pluggable substrate — GitHub chosen for owner's existing account, mobile app dispatch button, and Actions API run records used for check-in verification |
| Download binary from Releases at run time | adds a live dependency on the OSS repo + network at fire time |

## Consequences

- Residual platform risks (policy drift, account dormancy) are accepted and mitigated by the **annual drill** runbook and the paper Recovery Sheet backstop (DESIGN §13).
- Trigger repo must remain private; monitor fail-closes if forbidden artifacts (kit.age, share-R) appear in it.
- 2,000 free minutes/month (account-wide) vs ~60 used: no cost, but shared quota with the owner's other private repos can delay runs near month-end (F15; absorbed by day-granular recompute).
