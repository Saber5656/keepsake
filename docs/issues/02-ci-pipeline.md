# Title

CI pipeline: build, test, lint, vulnerability and secret scanning, pinned actions

## Summary

Add GitHub Actions CI for the OSS repo running build/test/lint/govulncheck/gitleaks on every PR and push to main, with least-privilege read-only permissions, credential-less checkouts, pinned tools, and an automated pinned-actions audit.

## Context

DESIGN §10.6 makes supply-chain hygiene a v1 security requirement: this repo produces a binary that later runs with mail credentials and share_G in scope. CI is also the enforcement point for later gates (interop 05, secret-hygiene 28, e2e 27).

## Scope

- `.github/workflows/ci.yml`, `.github/dependabot.yml`, `.gitleaks.toml`, `scripts/audit-actions.sh`, Makefile additions.

## Detailed Requirements

1. `ci.yml`: top-level `permissions: {contents: read}`; `concurrency: {group: ci-${{ github.ref }}, cancel-in-progress: true}`; triggers `pull_request` + `push: branches: [main]`. Jobs (names are FINAL — issue 31 wires them as required checks; no matrix, so check contexts are stable):
   - `test-ubuntu`, `test-macos`: checkout (`persist-credentials: false`), setup-go with `go-version-file: go.mod` and `check-latest: false`, `make build test`.
   - `lint`: golangci-lint official action, action pinned by SHA AND `with: version: <pinned vX.Y.Z from issue 01's variable>`.
   - `vuln`: install via `go install golang.org/x/vuln/cmd/govulncheck@vX.Y.Z` (exact version recorded in the workflow; no @latest), run `make vuln`.
   - `secrets`: gitleaks action pinned by SHA, `fetch-depth: 0`.
   - `cross`: ubuntu, `make cross`, assert exactly 4 artifacts exist.
   - `audit`: run `scripts/audit-actions.sh` (below).
2. Every `uses:` pinned to a full 40-hex commit SHA with a trailing `# vX.Y.Z` comment. `actions/checkout` everywhere sets `persist-credentials: false` (PR jobs must not expose even the read token to build steps).
3. `scripts/audit-actions.sh`: parses every workflow YAML under `.github/workflows/` and fails unless (a) every external `uses:` ref matches `@[0-9a-f]{40}$`, (b) no `pull_request_target` trigger exists, (c) no `secrets.` context is referenced anywhere in `ci.yml`, (d) top-level permissions are read-only in `ci.yml`. Implemented with `grep`/`awk` only (no yq dependency); exact patterns included in the script.
4. `.gitleaks.toml`: default rules; allowlist ONLY specific named fixture FILES (e.g. `internal/checkin/testdata/*.sig`) each with a justification comment — never whole directories of mixed content.
5. `.github/dependabot.yml` exact values: `version: 2`; ecosystems `gomod` (directory `/`, schedule weekly) and `github-actions` (directory `/`, schedule weekly).
6. Makefile: `make ci` = build+test+lint+vuln+cross (documented as core local parity; gitleaks and audit run in CI or via optional `make secrets` / `make audit` targets that no-op with a hint when the tools are absent).
7. README status badge for `ci.yml`.

## Acceptance Criteria

- [ ] CI green on a no-op PR; a deliberately failing test on a scratch commit turns `test-ubuntu` red (run link in PR, then reverted).
- [ ] `audit` job fails when given a scratch workflow with an unpinned action / `pull_request_target` / write permissions (three negative demonstrations linked in PR, then reverted) and passes on the real tree.
- [ ] gitleaks passes on full history; allowlist entries all point at existing files with comments.
- [ ] Dependabot config matches requirement 5 byte-for-byte (golden in repo).
- [ ] All checkout steps show `persist-credentials: false` (audit-script check (e) added for this).

## Validation

Scratch PR exercising all jobs incl. the negative audit demonstrations; run URLs linked in the PR.

## Dependencies

01.

## Non-goals

Release builds (29), CodeQL/rulesets/required-check wiring (31), interop job (05), nightly fuzz/secret-hygiene gates (28), trigger-repo workflows (24).

## Design References

DESIGN §10.6; research 03 C7; ISSUE_PLAN §6 supply chain.
