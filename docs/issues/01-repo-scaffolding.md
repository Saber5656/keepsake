# Title

Repository scaffolding: Go module, directory layout, Makefile, licenses

## Summary

Create the Go module `github.com/Saber5656/keepsake` with the normative package layout, build tooling, lint configuration, license files (Apache-2.0, owner-confirmed), and a README skeleton, so every subsequent issue has a fixed place to put code.

## Context

DESIGN §4.3 fixes the module layout so implementation agents do not improvise structure. ADR-001 fixes Go with stdlib-only CLI plumbing. The dependency allowlist (DESIGN §10.6) is enforced starting now.

## Scope

- `go.mod`, directory skeleton, `cmd/keepsake/main.go` stub, `Makefile`, `.golangci.yml`, `LICENSE`, `NOTICE`, `LICENSES/README.md`, `.gitignore`, `.editorconfig`, README update.

## Detailed Requirements

1. `go.mod`: module `github.com/Saber5656/keepsake`; `go` directive = the latest stable minor release at implementation time (check https://go.dev/dl — e.g. `go 1.NN`); `toolchain` directive pinned to the exact patch (`go1.NN.P`). Record the chosen version in the PR description. Zero `require` entries in this issue.
2. Directory skeleton = DESIGN §4.3 exactly. A `doc.go` with a one-paragraph purpose comment is required in each of these EXACT directories (and nowhere else): `internal/cliutil`, `internal/version`, `internal/config`, `internal/content`, `internal/payload`, `internal/crypto/agefile`, `internal/crypto/keys`, `internal/shamir`, `internal/encoding/sharetext`, `internal/bundle`, `internal/guide`, `internal/checkin`, `internal/statemachine`, `internal/mail`, `internal/gh`, `internal/trigger`, `internal/vault`, `internal/dist`, `internal/sectest`.
3. `internal/version/version.go`: `var Version = "dev"`, `var Commit = ""`, `func String() string` → `keepsake <Version> (<Commit>)`.
4. `cmd/keepsake/main.go`: prints `version.String()`, exits 0 (real router in issue 08).
5. `Makefile` targets:
   - `build`: `go build -o keepsake ./cmd/keepsake` (binary at repo root; gitignored)
   - `test`: `go test -race ./...`
   - `lint`: `golangci-lint run`
   - `fmt`: `gofmt -l -w .`
   - `vuln`: `govulncheck ./...`
   - `cross`: loops over `darwin/arm64 darwin/amd64 windows/amd64 linux/amd64` with `CGO_ENABLED=0 go build -trimpath -ldflags "-s -w -X github.com/Saber5656/keepsake/internal/version.Version=$(VERSION) -X github.com/Saber5656/keepsake/internal/version.Commit=$(COMMIT)" -o dist/keepsake_$${os}_$${arch}$${ext}` — **local-dev naming only**; public release artifact names are issue 29's contract (`keepsake_<version>_<os>_<arch>`), documented in a Makefile comment.
6. `.golangci.yml`: enable govet, staticcheck, errcheck, gosec, revive; `lll` explicitly disabled (named); placeholder comment for the forbidigo rule (added by issue 08) and the vendored-shamir exclusion (added by issue 03). Pin the golangci-lint version: choose the latest stable at implementation time, record it in README "Development" AND in a `GOLANGCI_LINT_VERSION` Makefile variable used by `make lint-install`.
7. Licenses: `LICENSE` = full Apache-2.0 text; `NOTICE` first line `keepsake` then `Copyright 2026 The keepsake Authors` (license confirmed by owner 2026-07-10, KU-6 resolved) and a reserved section header `## Vendored components` (filled by issue 03). `LICENSES/README.md` explains the directory holds third-party license texts (no placeholder license files — the MPL text arrives with issue 03).
8. `.gitignore`: `/keepsake`, `/dist/`, editor junk, OS junk. README: keep the existing Japanese one-liner; add English description, "status: pre-alpha — design in `docs/`", links to DESIGN.md / ISSUE_PLAN.md, and a Development section (Go version, golangci-lint version, make targets).

## Acceptance Criteria

- [ ] `make build` produces `./keepsake` printing `keepsake dev ()`.
- [ ] `make cross` produces exactly 4 files with the documented local names; the linux binary reports statically linked (`file`/`ldd` note in PR).
- [ ] `make test` and `make lint` pass on a clean checkout using the pinned golangci-lint.
- [ ] Layout + doc.go presence matches requirement 2 exactly (script check in PR).
- [ ] `go list -m all` shows only the main module (zero deps).
- [ ] LICENSE/NOTICE/LICENSES contents as specified; README updated without deleting the Japanese line.

## Validation

Run all Makefile targets locally and paste output in the PR; issue 02's CI re-runs them permanently.

## Dependencies

None (root issue).

## Non-goals

CLI subcommands (08), CI workflows (02), goreleaser + public artifact naming (29), any crypto.

## Design References

DESIGN §4.3, §10.6; ADR-001; ISSUE_PLAN wave 0, KU-6 (resolved).
