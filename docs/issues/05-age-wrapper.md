# Title

age encryption wrapper: scrypt recipient, streaming seal/open, CLI interop gate

## Summary

Implement `internal/crypto/agefile`: thin streaming wrappers over `filippo.io/age` for passphrase (scrypt) encryption/decryption, plus a CI interop gate proving byte-level compatibility with the pinned standard `age` CLI in both directions.

## Context

kit.age must be a bit-standard age v1 file so the Recovery-Sheet path stays decryptable with third-party tools given the passphrase P (DESIGN §5.2, §5.4; ADR-002). This is the only package allowed to import `filippo.io/age`.

## Scope

- `internal/crypto/agefile/agefile.go`, `_test.go`
- `internal/crypto/agefile/interop_test.go` (build tag `interop`) + `testdata/interop/` fixtures
- `interop` CI job added to `.github/workflows/ci.yml` (extends issue 02's workflow — dependency listed)

## Detailed Requirements

1. API (exact):
   ```go
   func Encrypt(dst io.Writer, src io.Reader, passphrase string) error
   func Decrypt(dst io.Writer, src io.Reader, passphrase string) error
   var ErrWrongPassphraseOrCorrupt = errors.New("age: wrong passphrase or corrupted file")
   ```
   - `Encrypt`: `age.Encrypt(dst, recipient)` with `age.NewScryptRecipient(passphrase)` + `SetWorkFactor(18)` (comment cites ADR-002: full-entropy passphrase makes the factor UX-only), `io.Copy` src → age writer, then `Close()` the age writer (REQUIRED — the final chunk is written on close); close/copy errors propagate.
   - `Decrypt`: `age.Decrypt(src, age.NewScryptIdentity(passphrase))`, then `io.Copy` to dst reading to EOF (age authenticates during read; a truncated read can miss tampering).
   - Error mapping: `age.NoIdentityMatchError` and header/MAC parse failures wrap `ErrWrongPassphraseOrCorrupt` (callers cannot distinguish by design); genuine I/O errors from src/dst are returned unwrapped (NOT collapsed) so infrastructure faults stay diagnosable.
2. go.mod: add `filippo.io/age` pinned to the latest v1.x at implementation time (record the exact version in the PR description; v1.2.x expected). No other new direct deps.
3. Streaming only: no `io.ReadAll`, no whole-file buffers (reviewed criterion + adversarial test below).
4. Import-boundary gate: add a `depguard` (or equivalent golangci) rule permitting `filippo.io/age` imports ONLY under `internal/crypto/agefile` — rule text included in this issue and appended to `.golangci.yml`.
5. Interop gate (build tag `interop`; runs as a separate CI job on linux):
   - **Fixture (decrypt direction)**: `testdata/interop/cli-generated.age` was generated ONCE with the standard `age` CLI. `testdata/interop/README.md` records: age version, exact generation command, the passphrase (a NON-SECRET fixed test string `keepsake-interop-fixture-passphrase`), and SHA-256 of the plaintext. Test: our `Decrypt` recovers plaintext with matching hash.
   - **Live gate (encrypt direction)**: CI installs the pinned `age` CLI (exact version + release-asset SHA-256 recorded in the workflow; downloaded from the age GitHub release) and `expect` (`apt-get install -y expect`). Our `Encrypt` output is decrypted by driving `age -d` through the provided expect script (age has no non-interactive passphrase flag; the exact expect script is committed at `testdata/interop/age-decrypt.expect` and included in this issue's PR). Assert plaintext hash equality.
   - KU-7 note in `testdata/interop/README.md`: fixture regeneration procedure when the pinned age version is bumped.
6. Package doc documents the single-recipient scrypt-only policy (no X25519 in v1, ADR-002).

## Acceptance Criteria

- [ ] Round-trip unit test; wrong passphrase → `errors.Is(err, ErrWrongPassphraseOrCorrupt)`.
- [ ] I/O error preservation test: failing writer (returns `errFixture` after N bytes) surfaces `errFixture`, not `ErrWrongPassphraseOrCorrupt`.
- [ ] Streaming proof: encrypt from an `io.Reader` that produces 64 MiB via `io.LimitReader` of a repeating pattern into a counting writer, with `testing.T`-checked ceiling on live heap growth via an adversarial `Read` implementation that fails the test if a single `Read` request exceeds 4 MiB (proves bounded chunking, portable across CI).
- [ ] Tamper matrix (each read to EOF): flip byte in header, first payload chunk, and final 16 bytes → decrypt error for all three.
- [ ] Interop fixture decrypts; live expect-driven `age -d` decrypts our output (CI transcript linked).
- [ ] Exactly one new DIRECT dependency (`filippo.io/age`); depguard rule active (demonstrated by a compile-fixture note in the PR).
- [ ] Planted-passphrase hygiene test: passphrase string never appears in any error text or test-captured output.

## Validation

CI `interop` job (linux) with pinned age CLI + expect; unit tests everywhere.

## Dependencies

01, 02.

## Non-goals

KEK/share handling (06), tar packing (11), X25519 identities (rejected, ADR-002), compression.

## Design References

DESIGN §5.2, §5.4, §10.3 rule 1, §10.6, KU-7; ADR-002; research 02 §1.
