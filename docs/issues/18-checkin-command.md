# Title

`keepsake checkin`: sign, commit, and push the liveness statement

## Summary

Implement the `checkin` command in `internal/vault` (statement/signing stays in `internal/checkin`): bump the counter safely, write and sign the statement in the trigger-repo clone, commit and push with bounded retries, and never leave the repo or the counter in an ambiguous state.

## Context

The owner runs this habitually (DESIGN §9 starts from it; ADR-005 channel 1). Failure modes here directly cause false alarms, so ambiguity (push raced, push half-landed) must resolve safely.

## Scope

- `internal/vault/checkin_cmd.go` (+ tests with local bare-repo fixtures), command wiring.

## Detailed Requirements

1. Preconditions (exact codes):
   - `trigger.local_path` exists and is a git worktree [`E-CKN-REPO`]; missing → error message includes the one-time clone command hint (`git clone git@github.com:<repo>.git <path>` and the https alternative). Cloning is a documented manual step (DESIGN §8).
   - `origin` URL must identify `trigger.repo`: normalize by stripping optional trailing `.git` and matching either `git@github.com:<owner>/<name>` or `https://github.com/<owner>/<name>` case-insensitively [`E-CKN-ORIGIN`].
   - Owned paths = exactly `state/checkin.json` + `state/checkin.sig`. `git status --porcelain -- state/checkin.json state/checkin.sig` must be empty (staged/unstaged/untracked all count as dirty) [`E-CKN-DIRTY`].
   - Current branch must have upstream `origin/<branch>` [`E-CKN-BRANCH`].
   - git binary present [`E-CKN-GIT`]; all git invocations use arg vectors, `GIT_TERMINAL_PROMPT=0`.
2. Counter: `next = max(localCounterFile, remote state.json accepted_counter, remote checkin.json counter) + 1` (absent/unparsable sources count as 0 with a warning). Local file `state/checkin-counter` (vault, 0600) updated ONLY after confirmed push (see 5).
3. Flow: `git pull --ff-only` (non-FF → `git fetch` then evaluate: if only owned paths diverge, `git rebase` once; conflict → `git rebase --abort` + `E-CKN-CONFLICT` with clean-tree guarantee) → write statement + signature (17; passphrase prompt wired via cliutil `ReadSecret`) → `git add <owned paths>` → commit `keepsake checkin #<counter>` → `git push`.
4. Push rejection: `git pull --rebase` once, push again. Second rejection → `E-CKN-PUSH`, exit 5, tree clean.
5. Ambiguous push resolution (normative): after ANY push error, `git fetch` and inspect `origin/<branch>:state/checkin.json`; if its counter == ours, the push landed — treat as success (update local counter, exit 0). Otherwise leave the local counter file untouched so the retry reuses a strictly-greater counter (monitor accepts any counter > accepted).
6. `--note "text"`: ≤200 runes, `unicode.IsControl` rejection [`E-CKN-NOTE`].
7. Success output (Printer): counter, timestamp, channel="cli", plus facts from last-known `state/state.json` if readable: phase, `last_valid_checkin` + whole UTC days elapsed (`floor((now − last)/24h)`; null → "初回 check-in（スイッチが有効化されます）"). No local recomputation of phase (monitor owns transitions).

## Acceptance Criteria

- [ ] E2E against a bare-repo fixture: two sequential checkins → counters N, N+1, signatures verify via 17.
- [ ] Counter recovery matrix: local file deleted / remote ahead / both stale — next counter correct.
- [ ] Push-race simulation: remote advanced between pull and push → rebase-retry succeeds; double-race → `E-CKN-PUSH` with clean tree AND unchanged local counter.
- [ ] Ambiguous-push test: push reports failure but remote actually has our commit → detected as success, counter file updated.
- [ ] Rebase-conflict fixture → `E-CKN-CONFLICT`, no mid-rebase state (`git status` clean, no `.git/rebase-merge`).
- [ ] Origin normalization table (ssh/https/.git/case); missing upstream → `E-CKN-BRANCH`; auth failure with `GIT_TERMINAL_PROMPT=0` exits 5 without hanging (fixture remote requiring credentials).
- [ ] Encrypted-key path: passphrase prompt invoked; wrong passphrase → clean error, no commit.
- [ ] `--note` control-char rejection; 200-rune boundary.

## Validation

Integration tests with `git init --bare` fixtures; manual smoke against the real private repo during issue 24 validation.

## Dependencies

07, 08, 17.

## Non-goals

Creating/cloning the trigger repo (24 + runbook), workflow_dispatch channel (GitHub-native), phase computation (19/23).

## Design References

DESIGN §6.7, §8 (checkin row — clone is manual); ADR-005.
