# Title

Dead man's switch state machine engine (Evaluate/Apply) and state.json schema

## Summary

Implement `internal/statemachine`: the pure two-step contract of DESIGN §9 — `Evaluate` produces a Plan of actions without mutating state; `Apply` folds send results into the durable state — plus ownership of the `state/state.json` schema (DESIGN §6.8), with an exhaustive table-driven suite.

## Context

Safety-critical core: wrongful release and silent non-release are both catastrophic (§2.4 principle 4). Purity (no clock/rand/I-O) makes it exhaustively testable and substrate-portable (ADR-003). This issue is the single owner of the state schema; other issues amend it only here.

## Scope

- `internal/statemachine/state.go` (schema + strict (de)serialization), `evaluate.go`, `apply.go`, `_test.go`

## Detailed Requirements

1. Types (exact):
   ```go
   type Phase string // "ACTIVE","REMINDING","ALERTING","RELEASED"
   type CheckinFacts struct {
       LastValid     time.Time // zero = no valid check-in ever (unarmed)
       Channel       string    // "cli" | "dispatch" | ""
       SignedCounter int       // highest verified signed counter this run; 0 if none
   }
   type ActionKind string // SendOwnerMail, SendRecipientMail, SendReleaseMail
   type Action struct {
       Kind       ActionKind
       Recipient  string            // exactly one address per action (owner or recipient)
       TemplateID string            // issue 21 IDs
       Data       map[string]string // template data; NEVER key material (share_G is injected by the monitor at send time)
   }
   type Plan struct { Phase Phase; Actions []Action; ReleaseTier bool } // ReleaseTier=true iff any SendReleaseMail present
   func Evaluate(t config.Timing, recipients []config.Recipient, ownerEmail string,
                 prev State, facts CheckinFacts, now time.Time) (Plan, error)
   type ActionResult struct { Action Action; OK bool }
   func Apply(prev State, plan Plan, results []ActionResult, facts CheckinFacts, now time.Time) State
   ```
2. `Evaluate` semantics per DESIGN §9.1 rows T-1..T6 (incl. rev: unarmed, pause history via edges, per-recipient release/confirm tracking, T6 weekly cap) with:
   - `elapsed = int(now.Sub(facts.LastValid).Hours() / 24)`, UTC
   - cadence rule: "send if `last_*_sent_at` is null or ≥N days ago" (never day-index arithmetic)
   - per-recipient release: actions only for recipients not in `release.sent`; confirms per `release.confirm[email]` count/cadence
   - `error` return: timing-invariant re-validation failure (§9.3) — the caller fail-closes; `Evaluate` emits NO release actions on any error
   - Determinism: no clock/rand reads; same inputs ⇒ deep-equal outputs.
3. `Apply` semantics (the ONLY mutation path): set `armed` on first valid check-in; update `last_valid_checkin/channel`, `accepted_counter = max(prev, facts.SignedCounter)` (signed only); advance `last_remind_sent_at` / `last_alert_sent_at` / `release.sent[email]` / `release.confirm[email]` / `release.last_alive_mail_at` / `last_unarmed_warning_at` ONLY for `results[i].OK` actions of the matching kind; set `last_run_at = now`; append history entries (PHASE_CHANGE, PAUSE_START/PAUSE_END via `pause_active` edge, CHECKIN, RELEASE_SENT per recipient, DRILL_RUN is appended by the monitor); cap history at 500 (drop oldest).
4. Schema: exactly DESIGN §6.8 (rev): strict JSON decode (unknown fields error `E-ST-UNKNOWN`), `schema:1` required, canonical writer (SetEscapeHTML(false), struct order, RFC3339 UTC, trailing LF) so state files are byte-stable for tests.
5. Failed sends: dedup/sent records MUST NOT advance (send-then-persist; retried next run). Test-enforced.
6. Zero imports beyond stdlib + `internal/config`.

## Acceptance Criteria

- [ ] Boundary table: every phase edge day (13/14, 27/28, 41/42 with defaults) × armed/unarmed × paused.
- [ ] Unarmed: no escalation ever without a first check-in; setup-warning cadence (7d) verified; arming via T0 sets `armed` irreversibly.
- [ ] Missed-cron gaps: runs on days 14, 19 → reminder fires on 19; release-day gap → release fires on next run.
- [ ] Per-recipient partial failure: failed release recipient absent from `release.sent`, re-planned next run; successful one not re-sent (3-consecutive-runs idempotency test).
- [ ] Confirm schedule exhaustion per recipient; T6 weekly cap via `release.last_alive_mail_at`.
- [ ] Pause: entry/exit history exactly once per edge; no mails while paused; release NOT evaluated during pause; pause cannot un-release.
- [ ] Invalid-timing config → error, no actions.
- [ ] Full-timeline test: daily Evaluate/Apply day 0..85 with silence from day 0 → exact ordered action sequence of §9.2, then no-ops after confirm exhaustion (days 71+).
- [ ] Determinism (double-run deep-equal); canonical serialization golden; `E-ST-UNKNOWN` fixture.
- [ ] 100% statement coverage on evaluate.go + apply.go.
- [ ] `FuzzStateJSON` (exact name; issue 28) round-trips the decoder without panics.

## Validation

CI; issue 27 replays the same scenario tables through the real binary.

## Dependencies

07.

## Non-goals

I/O, git, mail sending, API calls, drill orchestration (23/26 — drill only appends a history entry through the monitor).

## Design References

DESIGN §6.8 (schema owner), §9 (T-1..T6, Plan/Apply contract), §9.3, §10.3 rule 3; ADR-003.
