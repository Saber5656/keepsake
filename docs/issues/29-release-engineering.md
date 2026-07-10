# Title

Release engineering: goreleaser, checksums, version embedding, install docs

## Summary

Add reproducible release builds via goreleaser: 4-platform static binaries with embedded version, `checksums.txt`, GitHub Releases publishing workflow on tags, and installation documentation — the artifact contract consumed by `seal` tools/ and `trigger init` vendoring (DESIGN §8.1).

## Context

`internal/dist` (13) downloads `keepsake_<ver>_<os>_<arch>[.exe]` + `checksums.txt` by exact name; this issue makes releases produce exactly that. Long-term recipients run these binaries from USB years later — static, dependency-free builds are a product requirement (§5.4).

## Scope

- `.goreleaser.yaml`, `.github/workflows/release.yml`, README install section, `Makefile release-snapshot` target.

## Detailed Requirements

1. `.goreleaser.yaml`: builds for `darwin/arm64, darwin/amd64, windows/amd64, linux/amd64`; `CGO_ENABLED=0`, `-trimpath`, ldflags setting `internal/version.Version={{.Version}}` and `.Commit={{.ShortCommit}}`; binary name `keepsake`; **archive format: binary** (no tar/zip — dist and recipients need bare binaries) with `name_template: keepsake_{{.Version}}_{{.Os}}_{{.Arch}}`; checksum file `checksums.txt` (sha256, format `<hex>  <filename>`).
2. `release.yml`: trigger `push: tags: ["v*"]`; `permissions: contents: write` ONLY; pinned actions; runs `make ci` first (full gate before publish), then goreleaser; no other secrets (GITHUB_TOKEN suffices). Draft=false, prerelease auto from semver (`-rc`, `-beta` suffixes).
3. Artifact name contract asserted by a test in `internal/dist` (13 coordination): a table test pins the exact naming template — breaking it fails CI here AND there.
4. `make release-snapshot`: local `goreleaser release --snapshot --clean` for testing; document in README dev section.
5. README "Install" section: download table per OS, sha256 verification instructions (mac/win/linux one-liners), Gatekeeper/SmartScreen notes (KU-5 wording from guide copy), `go install` alternative for developers.
6. Version discipline: tags `vX.Y.Z`; `keepsake version` output matches tag exactly (release workflow asserts by running the built linux binary).
7. NO signing/notarization in v1 (explicit v2 note in .goreleaser.yaml comments referencing ISSUE_PLAN §7).

## Acceptance Criteria

- [ ] `make release-snapshot` produces 4 binaries + checksums.txt with the exact naming contract (asserted by script).
- [ ] Tag `v0.1.0-rc.1` on a scratch branch publishes a prerelease with all artifacts; `internal/dist.Ensure` (13) downloads and verifies it successfully (transcript in PR).
- [ ] Built binaries: `keepsake version` prints the tag; linux binary is static (`ldd` reports not dynamic).
- [ ] Release workflow permissions minimal; actions pinned (02 grep passes).

## Validation

Scratch prerelease end-to-end with dist download (the real contract test); CI snapshot job keeps the config from rotting.

## Dependencies

01, 02.

## Non-goals

Homebrew/scoop packaging (v2), cosign/SLSA provenance (v2), auto-update (never — pinning is the security model, ADR-003).

## Design References

DESIGN §8.1, §5.4, §10.6; ISSUE_PLAN §7; ADR-003.
