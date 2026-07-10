# Title

KEK service: generate, split, combine, fingerprint

## Summary

Implement `internal/crypto/keys`: KEK generation from `crypto/rand`, Shamir 2-of-2 split into typed share_R/share_G values, recombination, passphrase encoding, KEK fingerprinting, and best-effort zeroization.

## Context

Ties together shamir (03), sharetext (04), and agefile (05) into the key hierarchy of DESIGN §5.1. Identical share_R across recipients is a deliberate anti-collusion property (ADR-002 decision 4).

## Scope

- `internal/crypto/keys/keys.go`, `_test.go`

## Detailed Requirements

1. API:
   ```go
   type KEK struct{ /* unexported raw [32]byte */ }
   type Share struct { Role sharetext.Role; Raw []byte; Text string }
   func NewKEK() (*KEK, error)                                   // crypto/rand only
   func (k *KEK) Passphrase() string                             // sharetext Encode(RoleK, raw)
   func (k *KEK) Fingerprint() string                            // hex of SHA-256(raw)[:8], DESIGN §6.6
   func (k *KEK) Split() (shareR, shareG Share, err error)       // shamir.Split(raw, 2, 2); role-tagged; Text filled
   func Combine(a, b Share) (*KEK, error)                        // order-insensitive; errors on same-role pair (E-KEY-ROLE)
   func (k *KEK) Zero()                                          // best-effort wipe
   ```
2. `Combine` MUST verify roles are {R,G} exactly; identical-role input is an error (prevents combining two recipient shares — belt-and-suspenders on top of the identical-share design).
3. `Combine` does NOT validate correctness of the result (Shamir has no integrity); callers compare `Fingerprint()` against a manifest — document prominently.
4. Zeroization: `Zero()` overwrites the raw array; `Split`/`Combine` wipe temporary buffers before return; document GC-language limits (ADR-001 consequences).
5. No logging anywhere in the package; `String()` methods on KEK/Share return `"<redacted>"` (defense against accidental fmt printing — with a test).
6. Deterministic test mode: `func newKEKFromBytes(b []byte)` unexported, used by tests only.

## Acceptance Criteria

- [ ] Round-trip: NewKEK → Split → Combine (both orders) → identical raw, identical Passphrase, identical Fingerprint.
- [ ] Same-role combine returns E-KEY-ROLE error.
- [ ] Corrupted shareG raw (one byte flipped) → Combine succeeds but Fingerprint differs (test documents the no-integrity property).
- [ ] `fmt.Sprintf("%v %s", kek, share)` contains no key bytes (redaction test).
- [ ] share Text values decode back through sharetext to the same raw bytes.

## Validation

Unit tests in CI; reviewer greps package for `log`, `fmt.Print` (must be absent except redaction).

## Dependencies

03, 04, 05 (module path exposure of `sharetext.Role`; agefile only referenced by doc comments).

## Non-goals

Where shares are stored/printed (12, 13, 16), k-of-n (v2), share rotation logic (runbook, 30).

## Design References

DESIGN §5.1, §5.3, §6.6, §10.5 A3; ADR-002.
