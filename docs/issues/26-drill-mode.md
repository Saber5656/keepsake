# Title

Drill mode: test-fire semantics in monitor and the `keepsake drill` command

## Summary

Implement DESIGN §11.4: `monitor --drill` walks the full notification pipeline with a deterministic placeholder share and `[DRILL]`-marked mails (1 owner + N safety-check + N release-template), plus the owner-side `keepsake drill` command (in `internal/vault`) that dispatches and verifies a drill run remotely via a small `internal/gh` extension.

## Context

The annual drill is the primary mitigation for platform drift (KU-1), deliverability rot (KU-2/F8), and binary/workflow breakage (F4) — proving owner→Actions→SMTP→every-recipient while the owner is alive, with zero secret exposure.

## Scope

- Drill branch in `internal/trigger/monitor.go` (23 plumbed the flag), `internal/vault/drill.go`, `internal/gh/rundetail.go` extension (+ tests).

## Detailed Requirements

1. `monitor --drill` (branching BEFORE any secret read — issue 23 §2):
   - Placeholder share (deterministic constant): raw = `SHA-256("keepsake-drill-placeholder")[:32] || 0x01` (33 bytes) → `sharetext.Encode(RoleG, raw)`; the resulting string is committed as a constant with a golden test. `KEEPSAKE_SHARE_G` env is never read (instrumented getenv wrapper in tests proves zero reads).
   - Mails sent: exactly `1 + N + N` for N recipients — one `remind_owner`, one `alert_recipients` per recipient, one `release` per recipient — all with Drill flag ([DRILL] prefix + explanation block + recipient confirm-receipt footer, issue 21 §6).
   - Per-recipient results printed as a structured table (Printer) AND exit code = non-zero iff any send failed (exit summarizes; details are in output — codes per DESIGN §8).
   - State: parsed-semantic diff vs prior state must show ONLY a `DRILL_RUN` history entry (with per-recipient result summary in `detail`); committed as `keepsake-monitor: drill <RFC3339>` (heartbeat side-effect).
   - `--dry-run --drill`: prints the drill plan; no SMTP connections (asserted via listener fixture), no commits, tree byte-identical.
2. `internal/gh` extension (declared in issue 22's non-goals): `RunsForWorkflowSince(ctx, workflowFile string, event string, since time.Time) (Result, error)` — same validation/pagination rules as `DispatchRuns` but parameterized event; reused for drill correlation.
3. `keepsake drill` (owner machine):
   - Token source precedence: `GITHUB_TOKEN` env, else `gh auth token` (exec, arg vector); neither → `E-DRL-AUTH` with both remedies; token never printed.
   - If `gh` CLI present: record `t0 = now`, run `gh workflow run monitor.yml -f drill=true -R <repo>`, then poll (10s interval, 10 min timeout) `RunsForWorkflowSince(workflowFile="monitor.yml", event="workflow_dispatch", since=t0−30s)` for the FIRST run with `created_at ≥ t0` (scheduled runs are excluded by the event filter; residual ambiguity with a concurrent manual run is documented — earliest-created wins), then poll that run to completion and report conclusion + HTML URL.
   - If `gh` absent: print the literal manual fallback (5 numbered steps: open github.com/<repo>/actions → monitor.yml → Run workflow → check `drill` → watch the run; then the same epilogue checklist).
   - Epilogue checklist (Printer, ja): 「全受領者に届いたか電話等で確認」「届かない場合: 迷惑メール確認 → アドレス修正 → config 更新 → trigger update → 再 drill」「次回 drill 日をカレンダーへ登録」.
4. Secret hygiene: extended grep harness over drill stdout/stderr/commits/state/run output for planted SMTP password, GitHub token, and the REAL share_G fixture value (which must be absent everywhere — only the placeholder appears).

## Acceptance Criteria

- [ ] Integration test (fixture repo + SMTP capture): exact 1+N+N mails with [DRILL] subjects + confirm-receipt footer; placeholder share golden-matched; real share env proven unread; semantic state diff = one DRILL_RUN entry; commit message format.
- [ ] Partial-failure fixture: non-zero exit, per-recipient table shows the failed address, successful sends still delivered.
- [ ] `--dry-run --drill`: zero SMTP connections, byte-identical tree.
- [ ] `keepsake drill` with a PATH-shimmed `gh` + httptest API: happy path (correlates the right run among decoys created before t0), timeout path, gh-absent path (manual steps golden).
- [ ] Token precedence tests; `E-DRL-AUTH` message; token-absence from output (grep).
- [ ] Extended secret grep green.

## Validation

CI integration tests; the FIRST REAL DRILL against the owner's actual trigger repo + real recipients is the final v1 acceptance step (recorded in 30's QA log).

## Dependencies

23, 24 (and 22 for the extension point).

## Non-goals

Scheduling drills (owner's calendar), drilling decryption with real bundles (owner runs `verify --deep`), state-machine changes.

## Design References

DESIGN §11.4, §8 (drill row), §13.1 F4/F8, §13.2 KU-1/KU-2, §14.
