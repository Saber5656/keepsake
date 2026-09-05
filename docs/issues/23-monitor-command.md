# Title

`keepsake monitor`: trigger-side orchestration command

## Summary

Implement the `monitor` command in `internal/trigger` per DESIGN §11.3 (rev): fail-closed config validation, forbidden-file scan, two-channel check-in facts (API failures fail-closed toward no-release), Evaluate → pre-release final check → send → Apply → heartbeat commit, with drill branching before any secret read.

## Context

This binary run IS the dead man's switch. It executes unattended daily for years: every failure path must self-heal next run or surface loudly, and no failure may cause a wrongful release (§11.3, F1–F5, F15).

## Scope

- `internal/trigger/monitor.go` (+ fixture-repo integration tests), command wiring with flags `--repo-dir`, `--now` (RFC3339, refused when `GITHUB_ACTIONS=true`), `--drill`, `--dry-run`.

## Detailed Requirements

1. Sequence = DESIGN §11.3 steps 1–9 verbatim (that section is the contract; highlights and codes below). Each step emits one structured Printer line: `step=<name> status=<ok|fail|skip> detail=<short>` — no secrets, ever.
2. Step specifics (normative):
   - Config: `config.LoadTrigger` — invalid → best-effort `config_error_owner` mail (only if SMTP env parses) + exit 3; NO release path reachable.
   - Forbidden scan [`E-MON-FORBIDDEN`]: `git ls-files` (tracked files only), basename matched case-insensitively against `kit.age`, `*.age`, `share-R*` → fail-closed as above.
   - Facts: signed statement via 17 (invalid/missing signature → treated as absent; owner warning mail once, flagged via `invalid_sig_warned`); dispatch runs via 22 filtered `strings.EqualFold(actor, owner.github_login)`; window since `last_run_at − 48h` (first run: since `initialized_at`). **Actions API error (incl. `ErrWorkflowNotFound`): continue with signed-only facts, set `apiDegraded=true` — Evaluate runs but release-tier actions are stripped from the Plan this run [`E-MON-GHAPI` warning mail + exit non-zero].**
   - Env: `--drill` branches BEFORE any secret read (`KEEPSAKE_SHARE_G` untouched — instrumented test); otherwise share_G validated as role G via sharetext [`E-MON-SHARE` → owner mail + exit 4]; `GITHUB_REPOSITORY` equality with `keepsake.yaml` `repo` enforced only when `GITHUB_ACTIONS=true` [`E-MON-REPO`].
   - Pre-release final check (step 6): only when `plan.ReleaseTier`: `git fetch` + re-read signed statement + re-query dispatch; newer valid check-in → strip release actions, log `release_aborted_by_fresh_checkin`.
   - Send: templates data per issue 21; share_G injected into `release`/`postrelease_confirm` data at send time only; per-recipient `SendResult`s.
   - Apply + persist: canonical state write (19); commit exactly `state/state.json`; author `keepsake-monitor <keepsake-monitor@users.noreply.github.com>`; message `keepsake-monitor: <phase> day <elapsed>`; `GIT_TERMINAL_PROMPT=0`; push; race → fetch/rebase once with the merge rule (union of per-recipient send successes; max of timestamps/counters — implemented as re-Apply onto the remote state) → push again → else exit 5 (next run recovers; duplicates tolerated per send-then-persist).
3. `--dry-run`: steps 1–6 + printed action plan; no sends, no commits; repo tree byte-identical (hash-tree assertion in tests). Combinable with `--drill`.
4. `--now` injection: RFC3339 UTC; presence with `GITHUB_ACTIONS=true` → usage error exit 2 (production foot-gun guard).
5. Test-support envs (documented as test/e2e hooks, honored anywhere): `KEEPSAKE_GH_API_BASE` (issue 22 baseURL), `KEEPSAKE_SMTP_TEST_ROOTCA` (issue 20). Neither weakens verification when unset.
6. Drill semantics beyond the branch point are issue 26's (this issue lands the flag + branch returning "drill not implemented" until 26 merges).

## Acceptance Criteria

- [ ] Fixture-repo integration matrix (local git + httptest GH API + local SMTP): fresh unarmed run (no escalation; setup-warning after 7 fixture-days); ACTIVE→REMINDING with send; same-day re-run dedup; dispatch check-in via API stub resets phase; invalid signature warning once; RELEASED flow with per-recipient partial SMTP failure and next-run retry; push-race recovery preserving the other run's send records; invalid config, forbidden file, wrong-role share_G, API-failure (release stripped) — each fail-closed with the exact code.
- [ ] Pre-release final check: fixture where a fresh check-in lands between evaluate and send → no release mail, log line asserted; and the inverse (stale check-in → release proceeds).
- [ ] Timeline replay: §9.2 (days 0→70 daily `--now` invocations) asserting the exact mail sequence via SMTP capture — fixture-level scope (full-CLI e2e belongs to 27).
- [ ] `--dry-run` byte-identical tree; `--now` + `GITHUB_ACTIONS=true` → exit 2.
- [ ] Secret-hygiene grep across all tests' stdout/stderr/commits/state files: planted share_G/SMTP password appear ONLY in release-mail bodies captured by the SMTP fixture.

## Validation

CI integration suite (< 2 min); real-repo validation in 24/26.

## Dependencies

08, 17, 19, 20, 21, 22 (and 07 transitively).

## Non-goals

Workflow YAML generation (24), drill mail specifics (26), state schema changes (19 owns), share rotation (runbook).

## Design References

DESIGN §11.3 (rev — the step contract), §9, §10.2, §10.3, §13.1 F1–F5/F15.
