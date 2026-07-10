# Title

`keepsake monitor`: trigger-side orchestration command

## Summary

Implement the `monitor` command per DESIGN §11.3: load and fail-closed-validate trigger config, gather check-in facts from both channels, run the state machine, execute mail actions (injecting share_G only at release-send time), persist state, and commit/push the heartbeat.

## Context

This binary run IS the dead man's switch. It executes unattended inside GitHub Actions daily for years: every failure path must either self-heal on the next run or surface loudly to the owner (§11.3, F1–F5).

## Scope

- `internal/trigger/monitor.go` (+ integration-style tests with fixture repos), command wiring with flags `--repo-dir`, `--now` (RFC3339, test-only), `--drill`, `--dry-run`.

## Detailed Requirements

1. Sequence (each numbered step logs a single structured line via Printer; no secrets ever):
   1. Load `keepsake.yaml` (07). Invalid → attempt `config_error_owner` mail IF mail config+secret parse (best effort), exit 3. NO release path reachable on invalid config (§9.3).
   2. Forbidden-file scan: any `kit.age` / `share-R*` glob match in repo tree → same fail-closed handling [E-MON-FORBIDDEN] (§10.2).
   3. Facts: verify `state/checkin.json(.sig)` via 17 (invalid signature → treated as absent + owner-mail warning once per state flag); query dispatch runs via 22 since `state.last_run_at - 48h`, filter actor == owner login (case-insensitive); `last_valid_checkin = max(...)`; update `accepted_counter`.
   4. Read env: `KEEPSAKE_SHARE_G` (validated as sharetext role G — mismatch [E-MON-SHARE] → owner mail + exit 4, fail-closed), `KEEPSAKE_SMTP_URL[_SECONDARY]`, `GITHUB_TOKEN`, `GITHUB_REPOSITORY` (must equal config trigger repo [E-MON-REPO]).
   5. `Evaluate` (19) with `now` = flag or `time.Now().UTC()`.
   6. Execute actions in order; release template data gets share_G injected at compose time only; send-then-persist ordering (§9.1 rules); SendResults folded into state.
   7. Persist state.json; `git add state/state.json && git commit -m "keepsake-monitor: <phase> day <elapsed>" && git push`; on rejection: `pull --rebase` once, re-apply state write (recompute merge: OUR run's results win for send-records, MAX for accepted_counter), push again; second failure → exit 5 (next run recovers).
   8. Exit non-zero iff any action failed (Actions UI redness = extra owner signal).
2. `--dry-run`: steps 1–5 plus printed action plan; no send, no commit.
3. `--drill`: delegate semantics to 26 (this issue lands the flag plumbed but returning "drill not implemented" if 26 not merged yet).
4. Idempotency guarantee documented + tested: running twice same day (manual re-run) sends nothing twice (dedup via state timestamps/sent records).
5. Time injection (`--now`) refused when `GITHUB_ACTIONS=true` env is set (test-only guard, prevents accidental foot-gun in production workflow).

## Acceptance Criteria

- [ ] Fixture-repo integration tests (local git dir + httptest GitHub API + local SMTP fixture) for: fresh ACTIVE run, REMINDING day with send, dedup re-run, dispatch-checkin override, invalid signature warning path, RELEASED full flow with per-recipient partial SMTP failure + next-run retry, push-race recovery, invalid config fail-closed, forbidden-file fail-closed, wrong-role share_G fail-closed.
- [ ] Timeline test: replay §9.2 (days 0→70 daily invocations with `--now`) asserting the exact mail sequence via SMTP fixture capture.
- [ ] No secret material in any log line or committed file across all tests (grep harness).
- [ ] `--dry-run` leaves repo dir byte-identical (hash tree before/after).

## Validation

CI integration suite (the heaviest in the repo — keep under 2 min); real-repo validation happens in 24/26.

## Dependencies

08, 17, 19, 20, 21, 22 (and 07).

## Non-goals

Workflow YAML generation (24), drill semantics (26), state-machine logic changes (19), share_G rotation (runbook).

## Design References

DESIGN §11.3, §9, §10.2, §10.3, §13.1 F1–F5.
