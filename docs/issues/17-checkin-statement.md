# Title

Check-in statement format and SSH signature sign/verify

## Summary

Implement `internal/checkin`: the JSON check-in statement (DESIGN §6.7), SSHSIG signing with the owner's SSH key, and verification against `allowed_signers` with replay protection (monotonic counter + timestamp window).

## Context

Check-in authenticity is the defense against forged liveness (abuse A1/A2, ADR-005). Verification runs inside the monitor on every Actions run; signing runs on the owner's machine.

## Scope

- `internal/checkin/statement.go`, `sign.go`, `verify.go` (+ tests, fixtures)

## Detailed Requirements

1. Statement: exact bytes of the committed file are the signed message (no canonicalization step): `{"version":1,"counter":N,"timestamp":"RFC3339Z","note":"..."}` + trailing newline; writer produces this exact serialization (fixed field order via struct marshaling, no HTML escaping); size cap 4 KiB on read [E-CIN-SIZE].
2. Signing: SSHSIG (the `ssh-keygen -Y sign` format), namespace `keepsake-checkin`, using `golang.org/x/crypto/ssh` primitives. Support ed25519 and ecdsa-p256 private keys, OpenSSH format, with passphrase prompt via cliutil when encrypted (callback injected — package itself does no I/O). Output: armored signature file content (`-----BEGIN SSH SIGNATURE-----`).
   - Implementation note: x/crypto/ssh has no high-level SSHSIG API; implement the SSHSIG blob format per OpenSSH `PROTOCOL.sshsig` (magic `SSHSIG`, version 1, reserved, namespace, hash `sha512`) — this is ~150 lines and MUST include cross-verification fixtures generated with real `ssh-keygen -Y sign` (committed under `testdata/` with the generating commands in a README).
3. Verification: `Verify(statementBytes, sigBytes []byte, allowedSigners []string) (Statement, error)` — parse allowed_signers lines (principal, key type, base64 key; options ignored with warning), check signature against ANY listed key [E-CIN-SIG], parse statement [E-CIN-JSON], enforce version [E-CIN-VER].
4. Replay/freshness policy applied by a separate pure function `Accept(st Statement, lastCounter int, now time.Time) error`: require `st.Counter > lastCounter` [E-CIN-REPLAY] and `st.Timestamp ≤ now + 15min` [E-CIN-FUTURE]. (No lower bound on timestamp — an old-but-higher-counter statement is acceptable; elapsed-time math uses the statement timestamp.)
5. No network, no git in this package (18/23 own that).

## Acceptance Criteria

- [ ] Round-trip: sign fixture key → verify OK; wrong key, tampered byte, wrong namespace, truncated armor each fail with E-CIN-SIG.
- [ ] Cross-tool fixtures: statements signed by real `ssh-keygen -Y sign` (ed25519 + ecdsa) verify; our signatures verify with `ssh-keygen -Y verify` (documented manual step + committed transcript).
- [ ] Replay matrix: counter ≤ last rejected; future timestamp > +15min rejected; boundary cases (equal counter, exactly +15min) covered.
- [ ] Encrypted-key signing path with passphrase callback tested.
- [ ] Fuzz: `FuzzVerify` over statement+sig corpora, 30s, no panics.

## Validation

CI unit + fuzz smoke; cross-tool verification transcript in PR.

## Dependencies

01.

## Non-goals

Git commit/push (18), dispatch-channel verification (22/23), key rotation UX (30 runbook F9).

## Design References

DESIGN §6.7, §10.4, §10.5 A1/A2; ADR-005.
