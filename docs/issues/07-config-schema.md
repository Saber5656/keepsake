# Title

Config schemas and strict validation for vault config.yaml and trigger keepsake.yaml

## Summary

Implement `internal/config`: Go structs, strict YAML loading, and single-pass full validation for the two config files defined in DESIGN §6.4/§6.5, including timing sanity rules that the state machine's safety depends on.

## Context

Config validation is a security boundary (DESIGN §10.4): timing rules guarantee the family-intervention window; email validation prevents header injection; strict decoding prevents silent typo-misconfiguration of a life-critical switch.

## Scope

- `internal/config/config.go` (types + load), `validate.go`, `_test.go`, `testdata/` fixtures

## Detailed Requirements

1. Types mirroring DESIGN §6.4 exactly (`VaultConfig`) and §6.5 (`TriggerConfig`); both embed shared sub-structs (`Owner`, `Recipient`, `Timing`, `Mail`).
2. Loading: `LoadVault(path string) (*VaultConfig, error)`, `LoadTrigger(path string) (*TriggerConfig, error)` using `gopkg.in/yaml.v3` decoder with `KnownFields(true)`; file size cap 1 MiB; `version: 1` required (`E-CFG-VER` otherwise).
3. Validation collects ALL errors into `type ValidationErrors []FieldError` (`FieldError{Path, Code, Msg}`) — never fail-fast on the first.
4. Rules (codes in brackets):
   - owner.name 1..100 chars [E-CFG-NAME]; github_login matches `^[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,38})$` [E-CFG-LOGIN]
   - emails: RFC 5322 addr-spec subset — `local@domain` with printable ASCII local part, no whitespace, no `<>"`, domain has ≥1 dot, total ≤254; reject any CR/LF anywhere in any string field [E-CFG-MAIL / E-CFG-CRLF]
   - recipients 1..10 [E-CFG-RCPT]; duplicate emails rejected [E-CFG-DUP]
   - carrier-domain warning W-MAIL-1 (non-fatal) for: docomo.ne.jp, ezweb.ne.jp, au.com, softbank.ne.jp, i.softbank.jp, ymobile.ne.jp
   - timing: `7 ≤ remind_after_days` [E-CFG-T1]; `remind_after_days < alert_recipients_after_days < release_after_days` [E-CFG-T2]; `release_after_days − alert_recipients_after_days ≥ 7` [E-CFG-T3]; `release_after_days ≥ 21` [E-CFG-T4]; `remind_every_days ≥ 1`, `alert_every_days ≥ 1` [E-CFG-T5]; `postrelease_confirm_every_days ≥ 1`, `0 ≤ postrelease_confirm_count ≤ 12` [E-CFG-T6]
   - pause_until: RFC 3339 date, ≤ 90 days in the future at validation time [E-CFG-PAUSE]
   - allowed_signers: ≥1 line parseable as OpenSSH allowed_signers format (principal + key type + base64; key types ssh-ed25519 / ssh-rsa / ecdsa-*) [E-CFG-SIGNER]
   - trigger.repo matches `owner/name` pattern [E-CFG-REPO]; binary.sha256 is 64 hex chars [E-CFG-SHA]
5. `Warnings() []FieldError` separate from errors; CLI prints warnings but proceeds.
6. Redaction: config String()/JSON dump helper redacts emails unless caller passes ShowPII (DESIGN §10.7).
7. Helper `DefaultVaultConfig()` returning the documented defaults (used by `init`).

## Acceptance Criteria

- [ ] Fixture matrix: one valid vault + trigger config; ≥12 invalid fixtures each producing the exact expected code set (table-driven).
- [ ] Unknown YAML key anywhere → E-CFG-UNKNOWN with the key path.
- [ ] All-errors-at-once verified (fixture with 3 independent faults yields 3 codes).
- [ ] CR/LF injection fixtures (in name, email, subject_prefix) rejected.
- [ ] go.mod gains only `gopkg.in/yaml.v3`.

## Validation

CI unit tests; fuzz seed for the YAML loader deferred to issue 28 (note the hook: exported `LoadVaultBytes` for fuzzing).

## Dependencies

01.

## Non-goals

File generation (10, 24), state.json schema (19/23), SMTP URL parsing (20 — secrets never appear in config files).

## Design References

DESIGN §6.4, §6.5, §9.3, §10.4, §10.7; research 03 (carrier deliverability).
