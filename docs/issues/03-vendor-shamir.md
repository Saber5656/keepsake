# Title

Vendor Shamir secret sharing implementation from Vault v1.14.8 (MPL-2.0)

## Summary

Copy HashiCorp Vault's `shamir` package (last MPL-2.0 release, tag `v1.14.8`) into `internal/shamir/` with license headers, provenance record, and its upstream tests, then wrap it behind a minimal internal API.

## Context

Current Vault is BUSL-1.1 and not redistributable here; v1.14.8 is the final MPL-2.0 release (research 02 §2). The implementation is the industry-standard GF(2^8) split used to protect Vault master keys. ADR-002 decision 5.

## Scope

- `internal/shamir/shamir.go`, `internal/shamir/shamir_test.go` — verbatim from upstream except package name and import-path adjustments
- `internal/shamir/PROVENANCE.md`
- `LICENSES/MPL-2.0.txt` (full license text), `NOTICE` entry

## Detailed Requirements

1. Source: `https://github.com/hashicorp/vault` tag `v1.14.8`, directory `shamir/`. Record in `PROVENANCE.md`: upstream URL, tag, the tag's commit SHA, retrieval date, list of copied files with their upstream SHA-256, and a statement of local modifications (package rename only).
2. Keep the MPL-2.0 header comment at the top of each copied file; do NOT reformat or "improve" upstream code (gofmt allowed only if upstream already gofmt-clean; otherwise exclude from lint).
3. If the upstream files import Vault-internal packages, replace only those imports with stdlib equivalents and document each replacement in PROVENANCE.md (expected: the package is stdlib-only; verify).
4. Exclude `internal/shamir` from golangci-lint strict rules via `.golangci.yml` path exclusion (vendored code).
5. `NOTICE`: add section "internal/shamir — Copyright HashiCorp, Inc. — MPL-2.0 — see LICENSES/MPL-2.0.txt and internal/shamir/PROVENANCE.md".
6. Public surface consumed by issue 06 must be exactly: `Split(secret []byte, parts, threshold int) ([][]byte, error)` and `Combine(parts [][]byte) ([]byte, error)`.
7. Add one additional local test file `shamir_local_test.go` asserting: 32-byte secret → 2-of-2 shares are 33 bytes; combining both recovers the secret; combining one share with a corrupted share yields a wrong secret (not an error — document this property: Shamir gives no integrity, which is why kek_fingerprint exists, DESIGN §6.6).

## Acceptance Criteria

- [ ] Upstream tests pass unmodified (`go test ./internal/shamir/`).
- [ ] PROVENANCE.md contains tag commit SHA and per-file upstream hashes (verifiable by re-download).
- [ ] LICENSE headers intact; NOTICE updated; `make lint` green with the exclusion.
- [ ] No new `go.mod` requirements.

## Validation

Reviewer re-fetches the upstream files at the recorded tag and diffs against `internal/shamir/` (only the documented modifications may differ). CI runs the tests.

## Dependencies

01.

## Non-goals

KEK orchestration (06), share text encoding (04), any k>2 UX.

## Design References

DESIGN §5.1, §10.6; ADR-002; research 02 §2.
