# Title

CLI skeleton: subcommand router, global flags, exit codes, bilingual printer

## Summary

Implement `internal/cliutil` and wire `cmd/keepsake/main.go`: a stdlib-`flag` subcommand router (with nested subcommands), the global flags with defined precedence, the exit-code constants, a Japanese/English message printer with an enumerated initial catalog, and error-returning prompt helpers.

## Context

DESIGN §8 fixes the command surface and global behavior; ADR-001's stdlib-flag consequence forbids cobra. Recipients see this UI at the worst moment of their lives — wizard ergonomics and bilingual output are product requirements.

## Scope

- `internal/cliutil/router.go`, `printer.go`, `messages.go`, `prompt.go` (+ tests)
- `cmd/keepsake/main.go` registering stub commands: init, status, seal, open, verify, guide, checkin, monitor, trigger (nested: init/update), drill, version

## Detailed Requirements

1. Router (exact contracts):
   ```go
   type Context struct {
       Vault string; Lang string; JSON, Yes, ShowPII bool
       Stdout, Stderr io.Writer; Stdin io.Reader
       P *Printer
       // Lazy config: first call loads+caches; warnings returned once
       VaultConfig func(now time.Time) (cfg *config.VaultConfig, warnings []config.FieldError, err error)
   }
   type Command struct {
       Name, Short string
       Flags func(fs *flag.FlagSet, ctx *Context)   // registers command flags
       Run   func(ctx *Context, args []string) int  // args = positional remainder
       Sub   map[string]*Command                    // nested (trigger init/update)
   }
   ```
   Dispatch: `keepsake [global flags] <cmd> [<subcmd>] [flags] [args]`. Global flags (`--vault --lang --json --yes --show-pii`) are registered into EVERY FlagSet via `AddGlobalFlags(fs, ctx)`, so they are accepted both before and after the (sub)command (DESIGN §8). Unknown command → usage to stderr + exit 2. `keepsake help [cmd [subcmd]]` prints usage.
2. Global flag semantics: `--vault` precedence flag > `KEEPSAKE_VAULT` > `~/KeepsakeVault`; `~` expanded via `os.UserHomeDir`, path `filepath.Clean`ed. `--lang` default: `ja` if `LC_ALL` else `LANG` (that precedence) contains `ja` case-insensitively, else `en`.
3. Exit codes as constants: `ExitOK=0, ExitError=1, ExitUsage=2, ExitValidation=3, ExitIntegrity=4, ExitNetwork=5` (DESIGN §8).
4. Printer:
   ```go
   type MsgID string
   type Printer struct { Lang string; Out, ErrW io.Writer }
   func (p *Printer) Msg(id MsgID, args ...any)   // stdout
   func (p *Printer) Err(id MsgID, args ...any)   // stderr
   func (p *Printer) JSON(v any) error            // stdout, encoding/json, SetEscapeHTML(false), trailing \n
   ```
   Catalog in `messages.go`: `var messages = map[MsgID]map[string]string{...}` with BOTH `ja` and `en` for every ID. Initial catalog (exact IDs, text drafted in this issue, refined by 09/16 content review): `usage.header`, `usage.command`, `err.unknown_command`, `err.not_implemented`, `err.config_load`, `err.no_tty`, `prompt.confirm_default_no`, `prompt.share_input`, `version.human`, `help.global_flags`, `warn.config`, `stub.placeholder`. Missing-translation = test failure; unknown MsgID at runtime = panic in tests, fallback literal in production.
5. Prompts (all return errors; stdin injected via Context):
   ```go
   func (p *Printer) Confirm(ctx *Context, id MsgID) (bool, error)   // --yes → true; non-TTY without --yes → error E-CLI-NOTTY
   func (p *Printer) ReadLine(ctx *Context, id MsgID) (string, error)
   func (p *Printer) ReadSecret(ctx *Context, id MsgID) (string, error) // no echo via golang.org/x/term (allowlisted, DESIGN §10.6); non-TTY → plain ReadLine + warning
   ```
6. `version` command: human output `keepsake <Version> (<Commit>)`; `--json` → `{"version":"...","commit":""}` + newline via Printer.JSON (empty commit = empty string).
7. Output discipline: all user-visible output goes through Printer. `.golangci.yml` gains the exact forbidigo rule:
   ```yaml
   forbidigo:
     forbid:
       - p: ^fmt\.Print(f|ln)?$
         msg: use cliutil.Printer
   ```
   with path exclusions for `internal/cliutil/` and `cmd/keepsake/main.go` bootstrap error path. A committed lint fixture note in the PR demonstrates the rule firing.
8. Stub commands print `err.not_implemented` and exit 1.

## Acceptance Criteria

- [ ] Golden usage-output fixtures for `keepsake`, `keepsake help`, `keepsake help seal`, `keepsake help trigger init` (stdout/stderr split + exit codes asserted via `os/exec` table test on the built binary).
- [ ] `keepsake version --json` and `keepsake --json version` both work (flag-position test).
- [ ] Catalog completeness test (every MsgID has ja+en); LC_ALL-over-LANG detection tests; `--vault` precedence tests incl. `~` expansion.
- [ ] Prompt tests: `--yes` bypass; non-TTY Confirm error; ReadSecret non-TTY fallback warning.
- [ ] forbidigo rule present in `.golangci.yml` and `make lint` green.

## Validation

CI; manual `make build && ./keepsake help` smoke in PR description.

## Dependencies

01, 07.

## Non-goals

Real command logic (10, 13–16, 18, 23, 25, 26, 32), colors/TUI, shell completion (v2).

## Design References

DESIGN §8 (global flags incl. `--show-pii`, exit codes), §2.3, §10.3 rule 1, §10.6 (x/term); ADR-001 (stdlib flag).
