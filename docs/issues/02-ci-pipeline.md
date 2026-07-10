# Title

CI pipeline: build, test, lint, vulnerability and secret scanning, pinned actions

## Summary

Add GitHub Actions CI for the OSS repo running build/test/lint/govulncheck/gitleaks on every PR and push to main, with least-privilege permissions and all actions pinned to commit SHAs.

## Context

DESIGN §10.6 makes supply-chain hygiene a v1 security requirement: this repo produces a binary that later runs with mail credentials and share_G in scope. CI is also the enforcement point for later gates (interop test 05, secret-hygiene gate 28).

## Scope

- `.github/workflows/ci.yml`
- `.github/dependabot.yml` (gomod + github-actions, weekly)
- `.gitleaks.toml` (default rules; allowlist for test fixtures directory with justification comments)

## Detailed Requirements

1. `ci.yml`: top-level `permissions: contents: read`; jobs:
   - `test`: matrix `ubuntu-latest` + `macos-latest`; steps: checkout, setup-go (version from go.mod), `make build`, `make test` (with `-race`), upload no artifacts.
   - `lint`: golangci-lint official action, version pinned.
   - `vuln`: `govulncheck ./...`.
   - `secrets`: gitleaks action over full history (`fetch-depth: 0`).
   - `cross`: `make cross` on ubuntu only, assert 4 files exist.
2. Every `uses:` pinned to a full commit SHA with a trailing `# vX.Y.Z` comment.
3. `concurrency: {group: ci-${{ github.ref }}, cancel-in-progress: true}`.
4. Workflow must not use `pull_request_target`, must not expose secrets to PR jobs (no secrets needed at all in CI v1).
5. Add a `make ci` target chaining build/test/lint/vuln for local parity.
6. Add status badge to README.

## Acceptance Criteria

- [ ] CI green on a no-op PR; red when a test is made to fail (demonstrate once in PR description, then revert).
- [ ] `grep -R "uses:" .github/workflows | grep -v "@[0-9a-f]\{40\}"` returns nothing.
- [ ] gitleaks passes on current history.
- [ ] Dependabot config validates (GitHub UI shows both ecosystems).

## Validation

Open a scratch PR exercising all jobs; link run URLs in the issue/PR. Verify permissions block by checking the run's "Set up job" log shows read-only token.

## Dependencies

01.

## Non-goals

Release builds (29), CodeQL/rulesets/branch protection (31), trigger-repo workflows (24).

## Design References

DESIGN §10.6; research 03 C7; ISSUE_PLAN §6 "Supply chain".
