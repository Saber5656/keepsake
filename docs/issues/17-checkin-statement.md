# Title

Check-in statement format and SSH signature sign/verify

## Summary

Implement `internal/checkin`: the JSON check-in statement (DESIGN §6.7), SSHSIG signing with the owner's SSH key, and verification against `allowed_signers` with replay protection (monotonic counter + timestamp window).

## Context

Check-in authenticity defends against forged liveness (abuse A1/A2, ADR-005). Verification runs inside the monitor daily; signing runs on the owner's machine.

## Scope

- `internal/checkin/statement.go`, `sshsig.go`, `sign.go`, `verify.go` (+ tests, cross-tool fixtures)

## Detailed Requirements

1. Statement: compact single-line JSON `{"version":1,"counter":N,"timestamp":"2006-01-02T15:04:05Z","note":"..."}` + exactly one trailing LF, produced with `json.Encoder` + `SetEscapeHTML(false)`. Timestamp: UTC `Z` layout only, no fractional seconds — anything else `E-CIN-TIME`. The exact file bytes are the signed message (verifier never re-serializes). Read caps enforced BEFORE parsing: statement 4 KiB [`E-CIN-SIZE`], armored signature 16 KiB [`E-CIN-SIGSIZE`].
2. SSHSIG implementation (`sshsig.go`, ~150 lines over `golang.org/x/crypto/ssh`), normative per OpenSSH `PROTOCOL.sshsig` v1:
   - Armored envelope (`-----BEGIN SSH SIGNATURE-----` base64): `byte[6] MAGIC="SSHSIG"` ‖ `uint32 version=1` ‖ `string publickey (SSH wire encoding)` ‖ `string namespace` ‖ `string reserved (empty)` ‖ `string hash_algorithm` ‖ `string signature (SSH wire signature)`.
   - Signed blob (what the inner signature covers): `MAGIC` ‖ `string namespace` ‖ `string reserved` ‖ `string hash_algorithm` ‖ `string H(message)`.
   - namespace `keepsake-checkin`; hash_algorithm `sha512` (accept `sha256` on verify for cross-tool tolerance); message = the statement bytes.
3. Signing: `Sign(statement []byte, privateKeyPEM []byte, passphrase PassphraseFunc) (armored []byte, error)` where `type PassphraseFunc func() ([]byte, error)` is invoked only for encrypted OpenSSH keys (pure callback — this package does NO terminal I/O; issue 18 wires cliutil). Key types: ed25519 and ecdsa-p256 ONLY [`E-CIN-KEYTYPE`].
4. Verification: `Verify(statement, armored []byte, allowedSigners []string) (Statement, error)`:
   - Parse allowed_signers lines: `principal keytype base64key [comment]`; keytype ∈ {ssh-ed25519, ecdsa-sha2-nistp256}; any option token (containing `=` before keytype) → `E-CIN-SIGNERLINE` (REJECTED, aligned with issue 07); principals are parsed but NOT matched (config carries only owner keys — documented rationale).
   - Envelope checks: magic/version/namespace exact [`E-CIN-SIG`]; embedded publickey must equal one of the allowed keys (byte comparison of wire encodings) [`E-CIN-SIG`]; inner signature verifies over the signed blob [`E-CIN-SIG`]; statement JSON parses with `version==1` [`E-CIN-JSON`/`E-CIN-VER`].
5. Acceptance policy as a separate pure function: `Accept(st Statement, lastCounter int, now time.Time) error` — require `st.Counter > lastCounter` [`E-CIN-REPLAY`] and `st.Timestamp ≤ now+15min` [`E-CIN-FUTURE`]. No lower timestamp bound (an old-dated but higher-counter statement is valid; elapsed-time math uses the statement timestamp).
6. No network, no git, no logging in this package.

## Acceptance Criteria

- [ ] Round-trip: sign (ed25519 + ecdsa fixtures, plain + passphrase-encrypted) → verify OK; wrong key, tampered statement byte, tampered signature byte, wrong namespace, truncated armor → `E-CIN-SIG`.
- [ ] Cross-tool fixtures: statements signed by real `ssh-keygen -Y sign -n keepsake-checkin` (both key types) verify with our `Verify`; our armored output verifies with `ssh-keygen -Y verify` — generation commands + transcript committed under `testdata/README.md`.
- [ ] allowed_signers negative table: malformed line, unsupported keytype (ssh-rsa, p384), option-bearing line, duplicate keys (allowed — any match wins), empty list.
- [ ] Replay/freshness matrix incl. boundaries (counter equal → reject; exactly +15min → accept; +15min+1s → reject).
- [ ] Size-cap tests fire before parsing (oversized inputs produce cap errors, not parse errors).
- [ ] Timestamp format negatives: offset form `+09:00`, fractional seconds, lowercase `z` → `E-CIN-TIME`.
- [ ] `FuzzVerify` (exact name; referenced by issue 28) over statement+armor corpora, 30s smoke, no panics.

## Validation

CI unit + fuzz smoke; cross-tool transcript in PR.

## Dependencies

01.

## Non-goals

Git commit/push (18), dispatch-channel verification (22/23), key rotation UX (30 runbook F9).

## Design References

DESIGN §6.7 (exact bytes, LF, key types), §10.4, §10.5 A1/A2; ADR-005.
