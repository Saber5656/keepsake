# Title

Dead man's switch state machine engine

## Summary

Implement `internal/statemachine`: the pure `Evaluate` function realizing DESIGN §9 — phase computation, escalation actions with cadence dedup, release stickiness, pause handling, and the state.json schema (§6.8) — with an exhaustive table-driven test suite.

## Context

This is the safety-critical core: wrongful release and silent non-release are both catastrophic (§2.4 principle 4). Purity (no I/O, injected `now`) makes it exhaustively testable and substrate-portable (ADR-003 consequence).

## Scope

- `internal/statemachine/state.go` (State + (de)serialization), `evaluate.go`, `_test.go`

## Detailed Requirements

1. Types:
   ```go
   type Phase string // "ACTIVE","REMINDING","ALERTING","RELEASED"
   type CheckinFacts struct { LastValid time.Time; Channel string; Counter int }
   type Action struct { Kind ActionKind; Recipients []string; TemplateID string; Meta map[string]string }
   // ActionKind: SendOwnerMail, SendRecipientMail, SendReleaseMail, RecordHistory
   func Evaluate(t config.Timing, prev State, facts CheckinFacts, now time.Time) (State, []Action)
   ```
2. Semantics exactly per §9.1 T0–T6 and §6.8 fields, including:
   - day-granular `elapsed = int(now.Sub(facts.LastValid).Hours() / 24)`; UTC everywhere
   - cadence dedup via `last_remind_sent_at`/`last_alert_sent_at` timestamps ("send if null or ≥ N days ago"), NOT day-index arithmetic (missed-cron robustness)
   - release actions per-recipient, skipping those in `release.sent`; postrelease confirms bounded by count/cadence
   - pause: `now < pause_until` → ACTIVE(reason=paused), no mails, but a history entry on transition into/out of pause
   - RELEASED sticky; T6 owner "rotate now" mail with weekly cap via `release.last_alive_mail_at`
   - fail-closed rule §9.3: `Evaluate` never emits release actions if timing config violates invariants (defense against hand-edited trigger config — validate again inside, return an ErrorAction → owner mail)
3. Actions carry template IDs + metadata only; NO mail bodies, NO share material (the monitor injects share_G into the release template at send time — the engine never sees secrets).
4. `State` (de)serialization: strict JSON, schema 1, unknown fields error; history append helper with 500-entry cap.
5. Determinism: same inputs ⇒ same outputs (no clock/rand reads inside).

## Acceptance Criteria

- [ ] Table tests covering: every phase boundary day (13/14, 27/28, 41/42 with default timing), cadence dedup across consecutive days, missed-run gaps (e.g. runs on days 14, 19 → reminder fires on 19), check-in resets from each phase, pause entry/exit, release idempotency across 3 consecutive RELEASED-day runs, postrelease confirm schedule exhaustion, T6 weekly cap, invalid-timing fail-closed.
- [ ] Full-timeline property test: simulate daily runs day 0..80 with no check-in and assert the exact ordered action sequence of §9.2.
- [ ] Same-inputs determinism test (run Evaluate twice, deep-equal).
- [ ] 100% statement coverage on evaluate.go (this package only; enforced via covermode in Makefile note).
- [ ] Zero imports beyond stdlib + internal/config.

## Validation

CI; the e2e test (27) reuses the same scenario tables against the real monitor binary.

## Dependencies

07.

## Non-goals

I/O, git, mail sending, API calls (23), drill semantics (26 — drill bypasses Evaluate's persisted state by design).

## Design References

DESIGN §6.8, §9 (all), §10.3 rule 3 (no secrets in engine); ADR-003.
