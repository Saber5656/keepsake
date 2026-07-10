# Title

Release engineering: goreleaser, checksums, version embedding, install docs

## Summary

Add release builds via goreleaser producing exactly the four artifact names of the `internal/dist` contract plus `checksums.txt`, published from a split validate/publish workflow on tags, with a PR-time `goreleaser check`+snapshot job and installation documentation.

## Context

`internal/dist` (13) downloads `keepsake_<version>_<os>_<arch>[.exe]` + `checksums.txt` by exact name; this issue makes releases produce exactly that and keeps the contract tested. Recipients run these binaries from USB years later — static, dependency-free builds are a product requirement (§5.4).

## Scope

- `.goreleaser.yaml`, `.github/workflows/release.yml`, snapshot job added to CI, README "Install" section, `Makefile release-snapshot`.

## Detailed Requirements

1. `.goreleaser.yaml`:
   - builds: `darwin/arm64, darwin/amd64, windows/amd64, linux/amd64`; `CGO_ENABLED=0`, `-trimpath`, ldflags `-s -w -X github.com/Saber5656/keepsake/internal/version.Version={{.Version}} -X github.com/Saber5656/keepsake/internal/version.Commit={{.ShortCommit}}`.
   - archives: `format: binary`, `name_template: keepsake_{{.Version}}_{{.Os}}_{{.Arch}}` — goreleaser appends `.exe` for windows binary format; the four EXACT uploaded names are asserted (below): `keepsake_<v>_darwin_arm64`, `keepsake_<v>_darwin_amd64`, `keepsake_<v>_windows_amd64.exe`, `keepsake_<v>_linux_amd64`.
   - checksum: `name_template: checksums.txt`, sha256, covering exactly the four binaries (no extra artifacts; source archives disabled).
   - Comment noting signing/notarization is v2 (ISSUE_PLAN §7).
2. `release.yml` (tags `v*`): TWO jobs — `validate` (`permissions: contents: read`) running `make ci`; `publish` (`needs: validate`, `permissions: contents: write`) running goreleaser. Pinned actions; GITHUB_TOKEN only. Prerelease auto-detected from semver suffix.
3. Contract test (lives beside `internal/dist`, this issue depends on 13): a table test pinning the artifact name template AND a snapshot-integration test: `make release-snapshot` output filenames must be parseable/downloadable by `dist.Ensure` pointed at a file:// or httptest mirror of `dist/`.
4. CI addition (extends 02's workflow): `release-check` job on PRs — `goreleaser check` + `goreleaser release --snapshot --clean` + assert the four names + checksums.txt exist (keeps config from rotting).
5. Version discipline: tags `vX.Y.Z[-pre]`; `publish` runs the built linux binary and asserts `version --json` yields `.version == <tag>` and non-empty `.commit`.
6. README "Install": per-OS download table, sha256 verification one-liners (macOS `shasum -a 256 -c`, Windows `CertUtil`, Linux `sha256sum -c`), Gatekeeper/SmartScreen note pointing to the canonical wording in `internal/content/guide/recipient-guide-copy.md` (KU-5), `go install` for developers.

## Acceptance Criteria

- [ ] `make release-snapshot` produces the four exact names + checksums.txt (script-asserted).
- [ ] Scratch prerelease tag (e.g. `v0.1.0-rc.1`): both workflow jobs green with the stated permissions; `dist.Ensure` downloads and verifies from the real release (transcript in PR).
- [ ] Built linux binary: static (`ldd` not-dynamic), `version --json` matches the tag.
- [ ] `release-check` PR job green; contract table test green.
- [ ] Actions pinned (02's audit passes on the new workflows).

## Validation

Scratch prerelease end-to-end with a real `dist.Ensure` download; the PR snapshot job guards continuously.

## Dependencies

01, 02, 13.

## Non-goals

Homebrew/scoop packaging (v2), cosign/SLSA (v2), auto-update (never — ADR-003), reproducible-build attestation (not claimed in v1).

## Design References

DESIGN §8.1 (artifact contract), §5.4, §10.6; ISSUE_PLAN §7; ADR-003.
