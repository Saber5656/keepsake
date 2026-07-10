# Title

sharetext codec: Crockford Base32 encoding for key material

## Summary

Implement `internal/encoding/sharetext`: the normative textual format `KEEPSAKE-<ROLE>1-<GROUPS>-<CHECK>` for share_R, share_G, and the recovery passphrase, with strict decoding, typo tolerance, checksum verification, and fuzz tests.

## Context

Key material must survive paper, QR, and typing by non-technical Japanese users (research 02 §3). The 4-char CHECK group doubles as the anti-phishing token (DESIGN §7, §10.5 A7). Every other component (seal, open, monitor, guides) consumes this codec.

## Scope

- `internal/encoding/sharetext/sharetext.go`, `_test.go`, `fuzz_test.go`

## Detailed Requirements

1. Types/API:
   ```go
   type Role byte // RoleR = 'R', RoleG = 'G', RoleK = 'K'
   func Encode(role Role, raw []byte) (string, error)         // raw: 33 bytes for R/G, 32 for K
   func Decode(s string) (Role, []byte, error)
   func CheckGroup(role Role, raw []byte) string               // 4-char group, exported for guides/mails
   ```
2. Encoding: Crockford Base32 uppercase (alphabet `0123456789ABCDEFGHJKMNPQRSTVWXYZ`), dash-separated groups of 4 chars (last group may be shorter), prefix `KEEPSAKE-<ROLE>1-`, suffix `-<CHECK>` where CHECK = first 4 Crockford chars of `SHA-256(roleByte || raw)`.
3. Decoding: case-insensitive; strip ALL whitespace and dashes before parsing; map `O→0`, `I→1`, `L→1`; reject `U` and any other non-alphabet char with `E-SHARE-CHAR`; verify prefix/role (`E-SHARE-ROLE`), version `1` (`E-SHARE-VER`), payload length for role (`E-SHARE-LEN`), checksum (`E-SHARE-CHK`). Error type: `type ParseError struct { Code string; Pos int }` implementing `error`; human messages rendered by callers (bilingual, issue 08 printer).
4. Length invariants: 32 bytes → 52 data chars; 33 bytes → 53 data chars (integer math, no padding chars; implement 5-bit big-endian bit-packing; do NOT use stdlib base32 with a custom alphabet because Crockford has no padding and different bit ordering is unnecessary — document the choice in package doc).
5. `Encode` rejects wrong lengths for the role; `Decode(Encode(x)) == x` for all valid inputs.
6. No logging; no global state; allocation-light but clarity over micro-optimization.

## Acceptance Criteria

- [ ] Golden vectors committed: at least 3 per role with fixed raw bytes and exact expected strings (these vectors are frozen forever — comment says so).
- [ ] Property tests: round-trip over random inputs (1000 iters); every single-character substitution/deletion/insertion in a valid string is rejected (checksum or structural error).
- [ ] Typo-mapping tests: lowercase, `O/I/L` substitutions, arbitrary whitespace/dash placement all decode.
- [ ] `go test -fuzz=FuzzDecode -fuzztime=30s` finds no panics; corpus seeded with goldens + malformed samples.
- [ ] 100% branch coverage on Decode error paths.

## Validation

CI unit + fuzz smoke (30s budget in CI). Reviewer verifies a golden vector by hand with an independent Base32 tool.

## Dependencies

01.

## Non-goals

QR rendering (16), share semantics/combination (06), wordlist encodings (rejected, research 02 §3).

## Design References

DESIGN §7, §10.4, §10.5 A7; research 02 §3; ADR-002.
