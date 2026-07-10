# Title

Drill mode: test-fire semantics in monitor and the `keepsake drill` command

## Summary

Implement DESIGN §11.4: `monitor --drill` walks the full notification pipeline with placeholder key material and `[DRILL]`-marked mails, plus the owner-side `keepsake drill` convenience command that triggers and verifies a drill run remotely.

## Context

The annual drill is the primary mitigation for platform drift (KU-1), deliverability rot (KU-2/F8), and binary/workflow breakage (F4) — it proves the whole chain owner→Actions→SMTP→every recipient while the owner is alive, with zero secret exposure.

## Scope

- Drill branch in `internal/trigger/monitor.go` (23 plumbed the flag), `internal/vault/drill.go` command (+ tests).

## Detailed Requirements

1. `monitor --drill` (also reachable via workflow_dispatch input `drill=true` per §11.1):
   - never reads `KEEPSAKE_SHARE_G` (assert: env var may be present but must remain unread — code path receives a sentinel placeholder `KEEPSAKE-G1-TEST-...` generated to be sharetext-valid for role G with a fixed test payload)
   - sends: one `remind_owner`, one `alert_recipients`, one `release`, each with Drill flag (subject `[DRILL]` + explanation block; 21)
   - per-recipient SendResults reported in run output and as the process exit code (any failure → non-zero)
   - state.json: only a `history` entry `{event:"DRILL_RUN", results:...}` committed (heartbeat side-effect); phase, timestamps, sent-records untouched (assert byte-equality of all other fields)
2. `keepsake drill` (owner machine):
   - if `gh` CLI present: `gh workflow run monitor.yml -f drill=true -R <trigger.repo>`, then poll the run via `internal/gh` (extend with a `RunByID`/latest-run lookup — this issue owns that small addition) until completion (timeout 10 min), report conclusion + link
   - if `gh` absent: print exact manual steps (mobile/web path) and how to verify
   - epilogue checklist: 「全受領者に届いたか電話で確認」「届かない場合の対処 (spam/アドレス変更→config修正→trigger update)」「次回 drill 日をカレンダーへ」
3. Drill mails include a footer asking recipients to confirm receipt to the owner (closing the loop is human, by design).
4. `--dry-run --drill` combination: prints the drill plan without sending (for CI).

## Acceptance Criteria

- [ ] Integration test (fixture repo + SMTP capture): drill sends exactly 1+1+N mails with [DRILL] subjects, placeholder share only, real share env untouched (instrumented getenv wrapper proves no read), state diff limited to history.
- [ ] Non-zero exit on any recipient failure (partial-failure fixture).
- [ ] `keepsake drill` with mocked `gh` (PATH shim) covers happy/timeout/absent-gh paths.
- [ ] Grep harness: no real-share fixture string appears in any drill output.

## Validation

CI integration tests; FIRST REAL DRILL against the owner's actual trigger repo + real recipients is the final v1 acceptance step (recorded in 30's runbook execution log).

## Dependencies

23, 24.

## Non-goals

Scheduling drills automatically (calendar is the owner's), drilling the decrypt step with real bundles (owner does `verify --deep` locally instead), state-machine changes.

## Design References

DESIGN §11.4, §13.1 F4/F8, §13.2 KU-1/KU-2, §14 (drill runbook).
