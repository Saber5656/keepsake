# Title

Config schemas and strict validation for vault config.yaml and trigger keepsake.yaml

## Summary

Implement `internal/config`: Go structs, strict YAML loading with full-error collection, and single-pass validation for the two config files (DESIGN §6.4/§6.5), including the timing sanity rules the state machine's safety depends on.

## Context

Config validation is a security boundary (DESIGN §10.4): timing rules guarantee the family-intervention window; email validation prevents header injection; strict decoding prevents silent typo-misconfiguration of a life-critical switch.

## Scope

- `internal/config/config.go` (types), `load.go`, `validate.go`, `_test.go`, `testdata/` fixtures

## Detailed Requirements

1. Types mirror DESIGN §6.4 (`VaultConfig`) and §6.5 (`TriggerConfig`) exactly, sharing sub-structs (`Owner`, `Recipient`, `Timing`, `Mail`, `Binary`). `TriggerConfig` additionally has top-level `Repo string` (DESIGN §6.5). The full normative `TriggerConfig` YAML shape:
   ```yaml
   version: 1
   repo: "Saber5656/keepsake-switch"
   owner: {name: "...", email: "...", github_login: "..."}
   allowed_signers: ["..."]
   recipients: [{name: "...", email: "..."}]   # relation omitted in trigger config
   timing: {...}                                # same struct as vault
   mail: {from: "...", subject_prefix: "...", reply_to: "..."}
   binary: {path: "bin/keepsake-linux-amd64", sha256: "<64 hex>", version: "vX.Y.Z"}
   ```
2. Loading API (exact):
   ```go
   type Result[T any] struct { Config *T; Warnings []FieldError; Errors []FieldError }
   func LoadVault(path string, now time.Time) Result[VaultConfig]
   func LoadTrigger(path string, now time.Time) Result[TriggerConfig]
   func LoadVaultBytes(b []byte, now time.Time) Result[VaultConfig]     // fuzz/test entry
   func LoadTriggerBytes(b []byte, now time.Time) Result[TriggerConfig] // fuzz/test entry
   type FieldError struct { Path, Code, Msg string }
   ```
   `now` is injected for `pause_until` validation (no clock reads inside).
3. Unknown-field detection: decode to `yaml.Node` first and walk mappings against per-struct known-key sets, collecting `E-CFG-UNKNOWN` with full paths; THEN strict-decode into structs. This keeps "collect ALL errors" semantics that `KnownFields(true)` alone cannot provide. File size cap 1 MiB (`E-CFG-SIZE`); `version: 1` required (`E-CFG-VER`); YAML syntax errors → single `E-CFG-YAML` (position in Msg).
4. Field rules (validation collects everything; codes in brackets). Required unless stated:
   - `owner.name` 1..100 runes [E-CFG-NAME]; `owner.github_login` matches `^[a-zA-Z0-9](?:-?[a-zA-Z0-9]){0,38}$` (no leading/trailing/double hyphen) [E-CFG-LOGIN]
   - `owner.ssh_signing_key` (vault only): non-empty path, `~` allowed [E-CFG-KEYPATH]
   - addr-spec fields (`owner.email`, `recipients[].email`, `mail.reply_to` optional): printable-ASCII local part without whitespace/`<>"`, domain with ≥1 dot, ≤254 total [E-CFG-MAIL]; CR/LF anywhere in ANY string field [E-CFG-CRLF]
   - `mail.from`: RFC 5322 mailbox — either bare addr-spec or `display-name <addr-spec>`; display-name printable, no CR/LF; the addr-spec part follows the rule above [E-CFG-FROM]
   - `mail.subject_prefix` ≤ 30 runes, no control chars [E-CFG-SUBJ]
   - recipients 1..10 [E-CFG-RCPT]; duplicate emails (case-insensitive) [E-CFG-DUP]; `name` 1..100 runes; `relation` (vault only) ≤ 50 runes, optional
   - carrier-domain warning W-MAIL-1 (recipients only): docomo.ne.jp, ezweb.ne.jp, au.com, softbank.ne.jp, i.softbank.jp, ymobile.ne.jp
   - timing: `7 ≤ remind_after_days` [E-CFG-T1]; `remind_after_days < alert_recipients_after_days < release_after_days` [E-CFG-T2]; `release_after_days − alert_recipients_after_days ≥ 7` [E-CFG-T3]; `release_after_days ≥ 21` [E-CFG-T4]; `remind_every_days ≥ 1 ∧ alert_every_days ≥ 1` [E-CFG-T5]; `postrelease_confirm_every_days ≥ 1 ∧ 0 ≤ postrelease_confirm_count ≤ 12` [E-CFG-T6]; `checkin_interval_days ≥ 1` [E-CFG-T7]
   - `pause_until`: null or RFC 3339 timestamp (`2006-01-02T15:04:05Z07:00`); if set: ≤ now+90d [E-CFG-PAUSE]
   - `allowed_signers`: ≥1 lines; each = `principal keytype base64key [comment]`; keytype ∈ {ssh-ed25519, ecdsa-sha2-nistp256}; base64 decodes; lines with options (any token containing `=` before the keytype, e.g. `valid-after=`) are REJECTED [E-CFG-SIGNER] (v1 policy, aligned with issue 17)
   - `trigger.repo` / top-level `repo`: `^[A-Za-z0-9][A-Za-z0-9-]*/[A-Za-z0-9._-]+$` [E-CFG-REPO]; `trigger.local_path` non-empty (vault) [E-CFG-PATH]; `binary.version` matches `^v\d+\.\d+\.\d+(-[0-9A-Za-z.-]+)?$` [E-CFG-BINVER]; `binary.sha256` 64 hex [E-CFG-SHA]; `binary.path` = `bin/keepsake-linux-amd64` literal in v1 [E-CFG-BINPATH]
   - placeholder detection: any string field equal to or containing `REPLACE_ME` [E-CFG-PLACEHOLDER] (init-generated configs use these markers so unedited fields fail loudly)
5. `DefaultVaultConfig()` returns the documented defaults: timing = DESIGN §6.4 values; every identity field set to a `REPLACE_ME…` placeholder that intentionally fails validation; used by issue 10's generator (which emits YAML comments from a template, not from this struct).
6. Redaction helper: `func MarshalRedactedJSON(v any, showPII bool) ([]byte, error)` masking email local parts (`h***@example.com`) unless showPII (DESIGN §10.7).
7. No logging; errors never echo full raw config content (only paths/codes/short values ≤ 40 runes).

## Acceptance Criteria

- [ ] One fully-valid fixture per config kind loads with zero errors/warnings.
- [ ] ≥16 invalid fixtures per kind, table-driven, each asserting the EXACT code set (multi-fault fixture proves all-errors collection: 3 independent faults → 3 codes).
- [ ] Boundary fixtures: unknown key (nested path reported), YAML syntax error, >1 MiB file, missing version, type mismatch (string where int), duplicate emails differing by case, CR/LF in name/email/subject, `mail.from` with display-name (valid) and with CR/LF (invalid), pause_until +91d, option-bearing allowed_signers line, ssh-rsa key line (rejected), placeholder field.
- [ ] W-MAIL-1 fires for recipient carrier address, not for owner address.
- [ ] `now` injection proves pause validation is deterministic (fixed fixture time).
- [ ] go.mod gains only `gopkg.in/yaml.v3`.

## Validation

CI unit tests; `LoadVaultBytes`/`LoadTriggerBytes` are the issue-28 fuzz entry points (targets `FuzzLoadVaultBytes`, `FuzzLoadTriggerBytes` defined here with 10s smoke in PR CI).

## Dependencies

01.

## Non-goals

File generation (10, 24), state.json schema (19), SMTP URL parsing (20 — credentials never appear in config files).

## Design References

DESIGN §6.4, §6.5, §9.3, §10.4, §10.7; research 03 (carrier deliverability).
