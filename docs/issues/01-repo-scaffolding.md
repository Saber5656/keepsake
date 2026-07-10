# Title

Repository scaffolding: Go module, directory layout, Makefile, licenses

## Summary

Create the Go module `github.com/Saber5656/keepsake` with the normative package layout, build tooling, lint configuration, license files, and a README skeleton, so every subsequent issue has a fixed place to put code.

## Context

DESIGN §4.3 fixes the module layout so implementation agents do not improvise structure. ADR-001 fixes Go with stdlib-only CLI plumbing. The dependency allowlist (DESIGN §10.6) is enforced starting now.

## Scope

- `go.mod` (module path above; latest stable Go; `toolchain` directive pinned)
- Directory skeleton with placeholder `doc.go` per package: `cmd/keepsake/`, `internal/cliutil/`, `internal/version/`, `internal/config/`, `internal/payload/`, `internal/crypto/agefile/`, `internal/crypto/keys/`, `internal/shamir/`, `internal/encoding/sharetext/`, `internal/bundle/`, `internal/guide/`, `internal/checkin/`, `internal/statemachine/`, `internal/mail/`, `internal/gh/`, `internal/trigger/`
- `cmd/keepsake/main.go` printing version and exiting 0 (real router in issue 08)
- `Makefile` targets: `build`, `test`, `lint`, `fmt`, `vuln` (govulncheck), `cross` (darwin-arm64/darwin-amd64/windows-amd64/linux-amd64, `CGO_ENABLED=0`, `-trimpath`, `-ldflags "-s -w -X .../internal/version.Version=$(VERSION)"`)
- `.golangci.yml` (enable: govet, staticcheck, errcheck, gosec, revive; line length off)
- `LICENSE` = Apache-2.0 (owner name/year), `LICENSES/MPL-2.0.txt` placeholder dir note, `NOTICE` skeleton
- `.gitignore` (binaries, `dist/`, editor junk), `.editorconfig`
- README.md: keep the existing Japanese one-liner, add: short English description, "status: pre-alpha, design in docs/", links to DESIGN.md/ISSUE_PLAN.md

## Detailed Requirements

1. `internal/version/version.go`: `var Version = "dev"`, `var Commit = ""`; `func String() string` returning `keepsake <Version> (<Commit>)`.
2. Makefile `cross` writes to `dist/keepsake_<os>_<arch>[.exe]`.
3. No third-party dependency may be introduced in this issue (`go.mod` has zero `require` entries).
4. Every `internal/*` package: `doc.go` with one-paragraph purpose comment matching DESIGN §4.3.
5. `make lint` passes on a clean checkout with golangci-lint installed; document the required version in README "Development" section.
6. KU-6: license is Apache-2.0 pending owner confirmation; put `TODO(KU-6)` marker in NOTICE.

## Acceptance Criteria

- [ ] `make build` produces a working `keepsake` binary printing `keepsake dev ()`.
- [ ] `make cross` produces 4 static binaries (verify `file`/no dynamic deps for linux).
- [ ] `make test` runs (no tests yet → pass), `make lint` passes.
- [ ] Layout matches DESIGN §4.3 exactly (names and nesting).
- [ ] LICENSE/NOTICE/LICENSES present; README updated without deleting the Japanese line.

## Validation

CI in issue 02 will re-run these; for this issue run the Makefile targets locally and paste output in the PR. `go mod verify` clean.

## Dependencies

None (root issue).

## Non-goals

CLI subcommands (08), CI workflows (02), goreleaser (29), any crypto.

## Design References

DESIGN §4.3, §10.6; ADR-001; ISSUE_PLAN wave 0.
