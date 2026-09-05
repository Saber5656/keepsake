# Title

sharetext codec: Crockford Base32 encoding for key material

## Summary

Implement `internal/encoding/sharetext`: the normative textual format `KEEPSAKE-<ROLE>1-<GROUPS>-<CHECK>` for share_R, share_G, and the recovery passphrase, plus the mail-auth code derivation, with a strictly specified decode pipeline, typo tolerance, checksum verification, and fuzz tests.

## Context

Key material must survive paper, QR, and typing by non-technical Japanese users (research 02 §3). The CHECK group is integrity-only; mail authentication uses the separate 8-char mail-auth code (DESIGN §7, §10.5 A7). Every other component (seal, open, monitor, guides, mails) consumes this codec.

## Scope

- `internal/encoding/sharetext/sharetext.go`, `_test.go`, `fuzz_test.go` (fuzz target defined here; CI wiring is owned by issues 02/28)

## Detailed Requirements

1. Types/API (exact):
   ```go
   type Role byte
   const (
       RoleR Role = 'R' // recipient share, 33 raw bytes
       RoleG Role = 'G' // GitHub share, 33 raw bytes
       RoleK Role = 'K' // KEK / recovery passphrase, 32 raw bytes
   )
   type ParseError struct{ Code string } // Code ∈ E-SHARE-LEN, E-SHARE-CHK, E-SHARE-ROLE, E-SHARE-VER, E-SHARE-CHAR, E-SHARE-SIZE
   func (e *ParseError) Error() string   // returns the code and a short generic sentence; NEVER echoes input content
   func Encode(role Role, raw []byte) (string, error)
   func Decode(s string) (Role, []byte, error)          // error is *ParseError for input faults
   func CheckGroup(role Role, raw []byte) (string, error) // 4-char integrity group
   func MailAuthCode(rawShareG []byte) (string, error)    // first 8 Crockford chars of SHA-256("keepsake-mail-auth" || raw); role-G length enforced
   ```
2. Encoding: 5-bit big-endian bit-packing into the Crockford alphabet `0123456789ABCDEFGHJKMNPQRSTVWXYZ` (uppercase, no padding), dash-separated groups of 4 (last group may be shorter); prefix `KEEPSAKE-<ROLE>1-`; suffix `-<CHECK>` where CHECK = first 4 encoded chars of `SHA-256(roleByte || raw)`. Implementation note: hand-roll the bit-packing (do not use stdlib `base32` even with a custom alphabet — we need position-exact error codes and no-padding semantics; state this rationale in the package doc).
3. Decode pipeline — normative, in this exact order (mirrors DESIGN §7):
   1. `E-SHARE-SIZE` if input >1024 bytes.
   2. Fold full-width forms U+FF01–U+FF5E to ASCII and U+3000 to space (implement as a local 2-case mapping; no external Unicode dependency).
   3. ASCII-uppercase.
   4. Strip every ignored rune: space, `\t`, `\r`, `\n`, `-` (hyphen-minus), U+2010, U+2212.
   5. Require literal prefix `KEEPSAKE`; next rune = role ∈ {R,G,K} else `E-SHARE-ROLE`; next rune = `1` else `E-SHARE-VER`.
   6. Remainder = DATA‖CHECK (CHECK = last 4 runes). Apply alias mapping `O→0, I→1, L→1` to DATA and CHECK only.
   7. Any rune outside the Crockford alphabet (including `U`) → `E-SHARE-CHAR`.
   8. Length check by role (52 data chars for 32 bytes, 53 for 33) → `E-SHARE-LEN`.
   9. Decode; recompute checksum; mismatch → `E-SHARE-CHK`.
4. `Encode` rejects wrong raw lengths for the role; `Decode(Encode(x)) == x` for all valid inputs.
5. Length invariants documented as constants with tests: 32 bytes → 52 data chars; 33 bytes → 53 data chars.
6. No logging, no global state; `ParseError` carries no input bytes (verified by test).

## Acceptance Criteria

- [ ] Golden vectors committed: ≥3 per role with fixed raw bytes and exact expected strings, plus one mail-auth-code vector (vectors frozen forever — file comment says so).
- [ ] Round-trip property test (1000 random inputs per role).
- [ ] Mutation corpus test: a committed, deterministically generated corpus (seed constant, ≥500 cases) of single-rune substitutions/deletions/insertions **within the DATA/CHECK region of the post-normalization string**, excluding alias-equivalent and formatting-only mutations — every case must be rejected with a structural or checksum error. (Checksum is 20-bit: the corpus is fixed precisely so this is deterministic, not probabilistic.)
- [ ] Typo-tolerance table: lowercase input, `O/I/L` substitutions, arbitrary dash/space placement, full-width input (`ＫＥＥＰＳＡＫＥ…`, U+3000), U+2010/U+2212 dashes — all decode to the same raw bytes.
- [ ] Rejection table: `U` char, >1024-byte input, wrong role char, wrong version, truncated CHECK — exact `ParseError.Code` asserted.
- [ ] Error-content test: for a set of malformed secret-like inputs, `err.Error()` output contains no substring of the input (≥6 chars).
- [ ] `go test -fuzz=FuzzDecode -fuzztime=30s` panics-free; seed corpus = goldens + malformed samples. Fuzz function name is exactly `FuzzDecode` (referenced by issue 28).
- [ ] 100% branch coverage on the decode error paths.

## Validation

CI unit + fuzz smoke (30s). Reviewer verifies one golden vector by hand (independent Base32 tool + manual SHA-256).

## Dependencies

01.

## Non-goals

QR rendering (16), share semantics/combination (06), wordlist encodings (rejected, research 02 §3), CI fuzz scheduling (02/28).

## Design References

DESIGN §7 (decode pipeline, mail-auth code), §10.4, §10.5 A7; research 02 §3; ADR-002.
