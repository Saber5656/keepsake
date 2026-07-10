# Title

age encryption wrapper: scrypt recipient, streaming seal/open, CLI interop gate

## Summary

Implement `internal/crypto/agefile`: thin streaming wrappers over `filippo.io/age` for passphrase (scrypt) encryption/decryption, plus a CI interop test proving files are decryptable by the standard `age` CLI.

## Context

kit.age must be a bit-standard age v1 file so recipients can always fall back to third-party tools (DESIGN §5.2, §5.4; ADR-002). This is the only package allowed to import `filippo.io/age`.

## Scope

- `internal/crypto/agefile/agefile.go`, `_test.go`
- `internal/crypto/agefile/interop_test.go` (build-tagged `interop`)
- CI job addition (extends issue 02 workflow)

## Detailed Requirements

1. API:
   ```go
   func Encrypt(dst io.Writer, src io.Reader, passphrase string) error   // ScryptRecipient, SetWorkFactor(18)
   func Decrypt(dst io.Writer, src io.Reader, passphrase string) error   // ScryptIdentity
   var ErrWrongPassphraseOrCorrupt = errors.New(...)                     // all age decrypt failures map to this + wrapped cause
   ```
2. Add `filippo.io/age` to go.mod (allowlisted, DESIGN §10.6); pin the current v1.x release; no other new deps.
3. Work factor MUST be set explicitly to 18 with a comment referencing ADR-002 ("not security-load-bearing; UX predictability").
4. Streaming only — no full-file buffering; callers pass files.
5. Error mapping: passphrase mismatch and corrupted ciphertext are indistinguishable to callers by design; message text guides users to "wrong shares or corrupted kit" (worded by callers).
6. Interop test (`//go:build interop`): encrypt a fixture with our wrapper → decrypt via `exec` of the `age` binary with the passphrase on stdin (`age -d` prompts; use `--passphrase`? The CLI has no non-interactive passphrase flag — use the documented `AGE...` approach: run `age -d` with a PTY is overkill; INSTEAD do the reverse direction natively and the forward direction via `rage` which supports `--passphrase-from`? Simplify: use `expect`-free approach — the `age` CLI reads the passphrase from the terminal only. Therefore implement interop as: (a) decrypt-side: our wrapper decrypts a **committed fixture** `testdata/interop-age-cli.age` that was generated ONCE manually with the real `age` CLI (generation command documented in the fixture's README along with age version); (b) encrypt-side: our output is decrypted by a second, independent Go implementation path — spawn the pinned `rage` binary if present in CI, else skip with clear message. CI installs `rage` via cargo-binstall or GitHub release download, pinned version+sha256.)
7. Document KU-7 (CLI drift) in package doc: fixture regeneration procedure.

## Acceptance Criteria

- [ ] Round-trip unit test (encrypt→decrypt, wrong passphrase fails with `ErrWrongPassphraseOrCorrupt`).
- [ ] 1 GiB streaming test uses <100 MiB RSS (use `io.CopyN` from `/dev/zero`-like reader; assert via testing.B or skip on short mode).
- [ ] Interop: committed age-CLI-generated fixture decrypts with our wrapper; our ciphertext decrypts with pinned `rage` in CI.
- [ ] Tampering any single byte of ciphertext body or header → decrypt error.
- [ ] go.mod gains exactly one require (+ its transitive set); `govulncheck` green.

## Validation

CI `interop` job (linux) with pinned rage; fixture README documents regeneration. Reviewer checks work factor constant and streaming (no `io.ReadAll`).

## Dependencies

01.

## Non-goals

KEK/share handling (06), tar packing (11), X25519 identities (rejected, ADR-002).

## Design References

DESIGN §5.2, §5.4, §10.6, KU-7; ADR-002; research 02 §1.
