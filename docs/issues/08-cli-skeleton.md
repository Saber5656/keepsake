# Title

CLI skeleton: subcommand router, global flags, exit codes, bilingual printer

## Summary

Implement `internal/cliutil` and wire `cmd/keepsake/main.go`: a stdlib-`flag` subcommand router with the global flags, exit-code convention, Japanese/English message printer, and interactive prompt helpers used by every command.

## Context

DESIGN §8 fixes the command surface and global behavior; ADR-001 forbids cobra. Recipients see this UI at the worst moment of their lives — wizard ergonomics and bilingual output are product requirements, not polish.

## Scope

- `internal/cliutil/router.go`, `printer.go`, `prompt.go`, `_test.go`
- `cmd/keepsake/main.go` registering stub commands for: init, status, seal, open, verify, guide, checkin, monitor, trigger (with subcommands init/update), drill, version

## Detailed Requirements

1. Router: `type Command struct { Name, Short string; Flags *flag.FlagSet; Run func(ctx Context) int }`; `keepsake <cmd> [flags]`; unknown command → usage to stderr + exit 2; `keepsake help [cmd]`.
2. Global flags parsed before dispatch: `--vault PATH` (default `~/KeepsakeVault`, env `KEEPSAKE_VAULT`), `--lang ja|en` (default: `ja` if `LANG`/`LC_ALL` contains `ja`, else `en`), `--json`, `--yes`. Exposed via `Context`.
3. Exit codes exactly per DESIGN §8: 0/1/2/3/4/5 as package constants (`ExitOK`...`ExitNetwork`).
4. Printer: `type Printer struct{ Lang string }` with `Msg(id MsgID, args ...any)`; message catalog = Go map `map[MsgID]map[string]string` in `messages.go` (ja + en for every ID; missing translation = test failure). No external i18n dep.
5. Prompts: `Confirm(id) bool` (respects `--yes`), `ReadLine(id) string`, `ReadSecret(id) string` (no echo; use `golang.org/x/term` — allowlisted with go.mod addition documented in DESIGN §10.6 note: x/crypto already pulls x/term? If not, add `golang.org/x/term` and record it in the dependency table via PR description).
6. `version` command: prints `internal/version.String()`; `--json` → `{"version":..., "commit":...}`.
7. All output through Printer (enables the no-secret-logging audit in 28); direct `fmt.Print*` outside cliutil is lint-flagged via `forbidigo` rule added to `.golangci.yml` (allowlist: cliutil itself, main.go bootstrap error path).
8. Stub commands print "not implemented" + exit 1 (replaced by later issues).

## Acceptance Criteria

- [ ] `keepsake`, `keepsake help`, `keepsake help seal`, `keepsake version --json` behave as specified.
- [ ] Message catalog completeness test (every MsgID has ja+en).
- [ ] `--lang` and LANG-detection tests; `--yes` bypasses Confirm.
- [ ] forbidigo rule active (demonstrate by a failing example in tests of the lint config — or a `make lint` fixture note).
- [ ] Exit codes verified via `os/exec` table test on the stub binary.

## Validation

CI; manual `make build && ./keepsake help` smoke in PR description.

## Dependencies

01, 07 (Context carries a lazily-loaded VaultConfig accessor).

## Non-goals

Real command logic (10, 13–16, 18, 23, 25, 26), colors/TUI, shell completion (v2).

## Design References

DESIGN §8, §2.3, §10.3 rule 1; ADR-001.
