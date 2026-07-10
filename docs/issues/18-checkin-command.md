# Title

`keepsake checkin`: sign, commit, and push the liveness statement

## Summary

Implement the `checkin` command: bump the counter, write and sign the statement into the local trigger-repo clone, commit, push with one rebase-retry, and report the switch phase.

## Context

The owner runs this habitually (DESIGN §9 timeline starts from it). It must be fast, unambiguous about success/failure, and never leave the trigger repo in a broken state (ADR-005 channel 1).

## Scope

- `internal/vault/checkin.go` (+ tests with a local bare-repo fixture), command wiring.

## Detailed Requirements

1. Preconditions: `trigger.local_path` exists and is a git repo whose `origin` matches `trigger.repo` [E-CKN-REPO]; working tree clean in `state/` paths (other local edits are outside scope but a dirty `state/` aborts [E-CKN-DIRTY]).
2. Counter source: local `state/checkin-counter` in the vault; on absence or lower value, recover from trigger repo `state/state.json` `accepted_counter` (max+1 rule, DESIGN §6.7).
3. Flow: `git -C <path> pull --ff-only` (fall back to `fetch` + `rebase` only for our own state files; any conflict → abort with instructions [E-CKN-CONFLICT]) → write `state/checkin.json` + `state/checkin.sig` (17) → `git add state/checkin.json state/checkin.sig` → commit message `keepsake checkin #<counter>` → push; on push rejection: pull --rebase once, push again, else fail [E-CKN-PUSH, exit 5].
4. Git execution via `os/exec` argument vectors (no shell), `GIT_TERMINAL_PROMPT=0`; missing git binary → E-CKN-GIT with install hint.
5. Success output: counter, timestamp, and — if `state/state.json` readable — current phase + days since previous check-in ("switch was in REMINDING; now reset" case per §9 T0 is computed by the monitor, not locally; local output only reports facts).
6. `--note "text"` (≤200 chars, no control chars) included in statement.
7. Update local counter file only after successful push (atomicity: failed push must not burn counters — re-run uses same counter+new timestamp; monitor accepts because counter > accepted_counter still holds).

## Acceptance Criteria

- [ ] E2E against a local bare repo: two sequential checkins produce counters N, N+1 with valid signatures (verified via 17's Verify).
- [ ] Counter recovery from remote state.json when local file deleted.
- [ ] Push-race simulation (bare repo advanced between pull and push) → rebase-retry succeeds; double-race → E-CKN-PUSH with clean tree.
- [ ] Dirty `state/`, wrong origin, missing git → specified errors; no commit created.
- [ ] Failed push does not increment the local counter file.

## Validation

Integration tests with `git init --bare` fixtures in t.TempDir(); manual smoke against the real private repo during issue 24's validation.

## Dependencies

08, 17.

## Non-goals

Creating the trigger repo (24), workflow_dispatch channel (GitHub UI native), status rendering beyond facts (08's `status` command reads state.json similarly).

## Design References

DESIGN §6.7, §8 (checkin row); ADR-005.
