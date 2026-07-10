# Title

KEK service: generate, split, combine, fingerprint

## Summary

Implement `internal/crypto/keys`: KEK generation from `crypto/rand`, Shamir 2-of-2 split into consistency-guaranteed Share values, recombination with role/length enforcement, passphrase encoding, KEK fingerprinting, and best-effort zeroization with full anti-leak redaction.

## Context

Ties together shamir (03) and sharetext (04) into the key hierarchy of DESIGN §5.1. Identical share_R across recipients is a deliberate anti-collusion property (ADR-002 decision 4).

## Scope

- `internal/crypto/keys/keys.go`, `_test.go`

## Detailed Requirements

1. API (exact; Share fields UNEXPORTED so Raw/Text can never disagree):
   ```go
   type KEK struct{ /* unexported raw [32]byte */ }
   type Share struct{ /* unexported: role sharetext.Role; raw []byte; text string */ }
   func (s Share) Role() sharetext.Role
   func (s Share) Raw() []byte      // defensive copy
   func (s Share) Text() string
   func NewShare(role sharetext.Role, raw []byte) (Share, error)  // validates role∈{R,G} + len==33; derives text
   func ParseShare(text string) (Share, error)                    // sharetext.Decode; role∈{R,G} enforced
   func NewKEK() (*KEK, error)                                    // crypto/rand only
   func newKEKFromBytes(b []byte) (*KEK, error)                   // unexported, tests only; len!=32 → error
   func (k *KEK) Passphrase() string                              // sharetext.Encode(RoleK, raw)
   func (k *KEK) Fingerprint() string                             // hex(SHA-256(raw)[:8]), DESIGN §6.6
   func (k *KEK) Split() (shareR, shareG Share, err error)        // shamir.Split(raw, 2, 2)
   func Combine(a, b Share) (*KEK, error)
   func (k *KEK) Zero()
   var ErrShareRole = errors.New("keys: shares must be one R and one G")
   var ErrShareLen  = errors.New("keys: share must be 33 bytes")
   ```
2. `Combine`: order-insensitive; requires roles == {R,G} exactly (`errors.Is(err, ErrShareRole)`); length re-validated (`ErrShareLen`). It does NOT validate the result (Shamir has no integrity) — callers compare `Fingerprint()` against a manifest; documented prominently in the package doc.
3. Zeroization: `Zero()` wipes the raw array; `Split`/`Combine` wipe only their internal temporary buffers ("non-returned copies only" — returned key material lifetime is caller-owned; GC-language limits per ADR-001 documented).
4. Redaction: `String()` AND `GoString()` on both `KEK` and `Share` return `"<redacted>"`; tests assert `fmt.Sprintf("%v %s %+v %#v", ...)` contains no key bytes, hex, or sharetext substrings.
5. Secret-hygiene: package has no logging; error values never embed raw/text content (planted-value grep test over all error strings).

## Acceptance Criteria

- [ ] Round-trip: NewKEK → Split → Combine (both orders) → identical raw/Passphrase/Fingerprint.
- [ ] Same-role pair → `ErrShareRole`; truncated raw via NewShare → `ErrShareLen`; ParseShare of role-K text → `ErrShareRole`.
- [ ] Corruption test: flip a PAYLOAD byte (`raw[0]`, never the final x-coordinate byte) of shareG → Combine succeeds, Fingerprint differs (no-integrity property test, aligned with issue 03).
- [ ] Redaction matrix over `%v %s %+v %#v` for KEK and Share.
- [ ] Share texts decode back to identical raw via sharetext; `newKEKFromBytes` length error.
- [ ] Planted-value grep over error paths.

## Validation

Unit tests in CI; reviewer greps package for `log`/`fmt.Print` (must be absent).

## Dependencies

03, 04.

## Non-goals

Where shares are stored/printed (12, 13, 16), k-of-n (v2), rotation logic (runbook, 30), agefile usage (13 wires encryption).

## Design References

DESIGN §5.1, §5.3, §6.6, §10.3 rule 1, §10.5 A3; ADR-001 (consequences), ADR-002.
