# Research: GitHub Actions as a Dead Man's Switch Substrate — Constraints

Status: verified 2026-07-10. This research materially affects DESIGN.md sections "Trigger Architecture" and "Failure Modes", and ADR-003.

## Why GitHub Actions

The trigger must keep running **after the owner stops paying attention (or stops existing)**. Self-hosted infrastructure fails this (see research 01, "self-hosting paradox"). GitHub Actions on a free-plan private repository has no per-owner billing action tied to continued operation, is operated by a company with decade-scale continuity expectations, and provides scheduled execution, secrets storage, and an audit trail.

## Verified constraints and their design consequences

| # | Constraint (verified) | Source | Design consequence |
|---|---|---|---|
| C1 | "In a **public** repository, scheduled workflows are automatically disabled when no repository activity has occurred in 60 days." Private repositories are not covered by this statement. | [GitHub Docs](https://docs.github.com/actions/managing-workflow-runs/disabling-and-enabling-a-workflow) | Trigger repo MUST be private. As defense-in-depth against policy change (and because community reports are inconsistent, e.g. [discussion #32197](https://github.com/orgs/community/discussions/32197)), the monitor workflow commits a heartbeat (updated `state/state.json`) on every run, which also resets any activity clock. Community keepalive patterns: [dev.to guide](https://dev.to/gautamkrishnar/how-to-prevent-github-from-suspending-your-cronjob-based-triggers-knf), [keepalive marketplace action](https://github.com/marketplace/actions/keep-scheduled-workflow-activity). We implement the heartbeat ourselves (a commit) rather than depend on a third-party action. |
| C2 | Official docs say "repository activity" without enumerating qualifying events; community documentation reports that in practice only **new commits** reliably reset the inactivity clock (releases, tags, issues reportedly do not). | [dev.to guide](https://dev.to/gautamkrishnar/how-to-prevent-github-from-suspending-your-cronjob-based-triggers-knf) (community source; official docs are non-specific) | Conservative interpretation adopted: heartbeat must be a commit, not an API touch. |
| C3 | `schedule` events can be delayed or dropped during high-load periods; cron granularity is best-effort. | GitHub Docs (schedule event), widely observed | State machine is **day-granular and idempotent**: every run recomputes phase from `now - last_valid_checkin`; a missed run is caught up by the next. No logic may depend on exact-time execution. Schedule daily; missing several runs must never cause a skipped notification tier or a double release. |
| C4 | Free plan includes 2,000 Actions minutes/month for private repos (account-wide pool); a daily 1–2 minute job uses <5% of that. Over-quota private-repo usage is **blocked** for accounts without a valid payment method until the monthly reset. | [GitHub billing docs](https://docs.github.com/billing/managing-billing-for-github-actions/about-billing-for-github-actions) | Near-zero-cost operation, but the quota is shared with the owner's OTHER private repos → failure mode F15 (DESIGN §13.1): monitor may be blocked late in a month and resumes after reset; day-granular recompute absorbs the gap. Runbook: keep the trigger repo on an account with minimal other private Actions usage. Keep the monitor job under ~2 minutes (single static binary, no build step). |
| C5 | Workflows in a fork are disabled by default; disabling/enabling is manual per workflow. | [GitHub Docs](https://docs.github.com/actions/managing-workflow-runs/disabling-and-enabling-a-workflow) | Owner runbook: never operate the switch from a fork; after any manual disable, re-enable and run a drill. |
| C6 | `workflow_dispatch` can be triggered from the GitHub mobile app / mobile web UI; the run records `triggering_actor` server-side, which cannot be forged by repo contents. | GitHub Docs (workflow_dispatch, Actions API) | The phone check-in path is a `workflow_dispatch` workflow. The monitor verifies dispatch check-ins via the **Actions API** (`GET /repos/{owner}/{repo}/actions/runs?event=workflow_dispatch`), matching `triggering_actor` to the configured owner login — NOT via files committed to the repo (commits are forgeable with push access; API run records are not). |
| C7 | `GITHUB_TOKEN` supports least-privilege `permissions:` blocks per workflow; secrets are exposed only to workflows on the repo. | GitHub Docs | `monitor.yml` needs `contents: write` (heartbeat/state commit) + `actions: read` (dispatch verification). `checkin-button.yml` needs no secrets and no write except a no-op (see design). Mail credentials and the GitHub-side Shamir share (`KEEPSAKE_SHARE_G`) live in Actions secrets, set manually by the owner. |
| C8 | Scheduled workflows run on the default branch's workflow file. | GitHub Docs | The trigger repo has a single protected default branch (`main`); `keepsake trigger update` operates on it directly. |

## Residual platform risks (documented, not solved in v1)

| Risk | Horizon | Mitigation |
|---|---|---|
| GitHub changes free-tier Actions policy or cron semantics | Years | Annual **drill** (test-fire with TEST-marked mails) is a mandatory runbook item; a failed drill reveals platform drift while the owner is alive. Trigger logic lives in the `keepsake` binary, so migrating to another scheduler (GitLab CI, a relative's machine) is a config exercise, not a redesign. Pluggable substrates are a v2 item. |
| Owner's GitHub account is compromised | Any time | See DESIGN.md §10 (abuse cases A5/A6): 2FA with hardware key recommended; worst case leaks only `KEEPSAKE_SHARE_G` (useless without the physical bundle) or DoSes the switch (recoverable while owner alive; paper-safe fallback after death). |
| Owner's GitHub account flagged dormant / ToS action | Years | GitHub has no published dormant-account deletion policy for accounts with content; drill detects. Paper-safe fallback (recovery sheet in owner's safe, reachable via probate) is the ultimate backstop. |
| Actions runner image changes break the binary | Years | Static linux-amd64 Go binary (CGO_ENABLED=0), vendored INTO the trigger repo and checksum-verified at each run; no build step at trigger time. |

## Email delivery from Actions

- Common practice is SMTP from a workflow step (e.g. [action-send-mail](https://github.com/marketplace/actions/send-e-mail-smtp)); we instead send from within the `keepsake` binary (go-mail, ADR-006) to avoid third-party action supply-chain exposure and to unit-test delivery logic.
- Deliverability to Japanese carrier addresses (docomo/au/SoftBank) is historically hostile to unknown SMTP senders. Consequences: (a) recipients SHOULD be regular mailbox addresses (Gmail/iCloud) rather than carrier addresses — enforced as a config lint warning; (b) the annual drill sends real mail end-to-end to every recipient address; (c) optional secondary SMTP provider config as fallback. Known unknown: KU-2 in DESIGN.md §13.

## Sources

- https://docs.github.com/actions/managing-workflow-runs/disabling-and-enabling-a-workflow
- https://github.com/orgs/community/discussions/32197
- https://dev.to/gautamkrishnar/how-to-prevent-github-from-suspending-your-cronjob-based-triggers-knf
- https://github.com/marketplace/actions/keep-scheduled-workflow-activity
- https://github.com/marketplace/actions/send-e-mail-smtp
- https://cronjobpro.com/guides/monitor-github-actions-scheduled-workflows
