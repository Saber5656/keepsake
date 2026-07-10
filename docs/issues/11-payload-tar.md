# Title

Payload manifest and deterministic tar builder

## Summary

Implement `internal/payload`: walk `payload/`, enforce content policy (no symlinks, size caps), produce `payload-manifest.json`, and stream a byte-deterministic USTAR archive for encryption.

## Context

DESIGN §6.2. Determinism makes seal outputs reproducible for identical inputs (auditable, testable); the manifest gives recipients an integrity inventory and `open` its extraction bounds (DESIGN §10.4).

## Scope

- `internal/payload/walk.go`, `manifest.go`, `tar.go`, `_test.go`, `testdata/`

## Detailed Requirements

1. `Scan(dir string, caps Caps) (*Manifest, error)`: lexicographic walk (byte order of `/`-separated relative paths, NFC-normalized names rejected if normalization changes bytes — record E-PAY-UNICODE with the path; simpler: reject any path whose UTF-8 bytes differ from NFC form).
2. Policy errors (all collected): symlink E-PAY-SYMLINK, hardlink-count>1 E-PAY-HARDLINK, non-regular file E-PAY-SPECIAL, single file >1 GiB E-PAY-FILESIZE, total > hard cap E-PAY-TOTAL (default 2 GiB; warn ≥ soft cap 512 MiB W-PAY-SIZE), file count >100k E-PAY-COUNT, empty payload E-PAY-EMPTY, path containing `..` segment or NUL E-PAY-PATH.
3. `Manifest` JSON `{schema:1, created_at, files:[{path,size,sha256}], total_size, file_count}`; hashing streams each file once.
4. `WriteTar(w io.Writer, dir string, m *Manifest, sealTime time.Time) error`: first entry `payload-manifest.json` (serialized canonical: sorted keys via struct order, trailing newline), then files in manifest order; USTAR format; mode 0644/dirs 0755; uid=gid=0; uname/gname empty; mtime = sealTime truncated to 00:00:00 UTC; directory entries included for every ancestor.
5. Re-hash while streaming into the tar; mismatch vs manifest (file changed between Scan and WriteTar) → E-PAY-RACE abort.
6. Determinism contract: same dir bytes + same sealTime ⇒ identical tar bytes (test with two runs + shuffled directory creation order).

## Acceptance Criteria

- [ ] Determinism test green (hash-compare two independent builds).
- [ ] Every policy error has a fixture and exact-code assertion; multi-error collection verified.
- [ ] Round-trip: WriteTar output extracts (stdlib reader) to byte-identical files.
- [ ] Manifest hashes independently verified in test via second hashing pass.
- [ ] Unicode NFC fixture (macOS NFD-style name) rejected with E-PAY-UNICODE.

## Validation

CI unit tests; `testdata/` includes a mixed-tree fixture (nested dirs, multibyte 日本語 filenames).

## Dependencies

01.

## Non-goals

Encryption (05/13), extraction (14 — separate hardened path), compression (age already compresses nothing; payload compression is v2 with size-oracle analysis).

## Design References

DESIGN §6.2, §6.6, §10.4.
