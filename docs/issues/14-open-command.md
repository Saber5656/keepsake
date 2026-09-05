# Title

`keepsake open`: recipient decryption wizard

## Summary

Implement `keepsake open` in `internal/vault`: the interactive Japanese-first wizard that locates kit.age, collects both shares with forgiving input handling, fingerprint-checks the combined KEK, and decrypts through the hardened extractor (`payload.Extract`, issue 11). Must run on a machine with NO vault and NO config (the recipient's PC).

## Context

Executed by a grieving non-technical person following a paper guide (DESIGN §2.2). Every error message must say what to do next in plain Japanese. Extraction hostile-input handling lives in issue 11; this issue owns the UX and the exit-code taxonomy.

## Scope

- `internal/vault/open.go` (+ tests), command wiring.

## Detailed Requirements

1. Invocation modes (DESIGN §8 open row):
   - Wizard: `keepsake open [KIT_PATH]` — positional path optional; else search order for `kit.age`: CWD → executable's directory → executable's parent; found → confirm with user; none → prompt for path (strip surrounding quotes, shell-escape backslashes, trailing whitespace — drag&drop tolerant).
   - Non-interactive: `--kit PATH (--share-r-file PATH | --share-r TEXT) --share-g TEXT --out DIR --yes`.
   - MUST NOT touch vault config: `open` never calls `Context.VaultConfig` (recipient machines have none) — enforced by a test running in an empty HOME.
2. Share collection: auto-detect `share-R.txt` beside the kit and parse via `bundle.ReadShareRFile` (confirm before use); share_G prompted as paste. Sharetext errors map to guidance messages by code (`E-SHARE-CHK` → 「文字の写し間違いがないか確認してください。確認コード（末尾4文字）が一致しません」 etc.); 5 attempts max per share, then exit 3 with the "call your helper" message. All share input errors are validation → **exit 3**.
3. Bundle manifest handling: if `manifest.json` exists beside the kit — schema check; `kit_format > 1` → `E-OPEN-FORMAT` with the upgrade message (exit 4); `kek_fingerprint` compared BEFORE decryption: mismatch → message 「このメールの鍵とこの USB は組み合わせが違います（古い USB か、別のキットのメールかもしれません）」 + no decrypt attempt (exit 4). Absent manifest → proceed with a warning (paper-only scenarios).
4. Size pre-check: kit file size must be ≤ manifest `kit_size` + 64 MiB slack when manifest present (`E-OPEN-SIZE`, exit 4).
5. Decrypt+extract: stream `agefile.Decrypt` into `payload.Extract` (issue 11 owns all safety codes E-OPEN-*); dest = `--out` or `./keepsake-opened/` (DESIGN §8; pre-existing → error with hint to rename). Wrong-passphrase/corrupt (`ErrWrongPassphraseOrCorrupt`) → bilingual "wrong shares or corrupted kit" guidance, exit 4.
6. Post-extract: report file count/bytes; `E-OPEN-HASH` mismatches listed prominently (exit 4 even though files were written — message explains partial trust). Success epilogue: point at `00-README-FIRST.md` if present, else at the output folder; plaintext warning (「この中身は暗号化されていません…」).
7. Exit-code taxonomy (normative): 2 usage; 3 share/input validation (incl. retry exhaustion); 4 = manifest/fingerprint/format/size/extract/decrypt/hash failures; 5 unused here.
8. Language: wizard defaults to `ja`, honors `--lang en`; every message ID exists in both languages (catalog extension in this issue).

## Acceptance Criteria

- [ ] Scripted-stdin wizard transcript test on a sealed fixture bundle: decrypts, extracts, hashes verified, epilogue correct.
- [ ] No-config guarantee: passes in an environment with no vault/HOME config (explicit test).
- [ ] Positional path + autodetect + prompt fallback matrix; drag&drop cleanup cases.
- [ ] Fingerprint-mismatch fixture → specified message ID, no decrypt attempt, exit 4.
- [ ] kit_format=2 fixture → E-OPEN-FORMAT upgrade message, exit 4; missing manifest → warning path works.
- [ ] Share retry limit (5) → exit 3; every E-SHARE-* code maps to a distinct message ID (table test).
- [ ] Oversized kit vs manifest → E-OPEN-SIZE before decryption.
- [ ] Secret-hygiene grep: share texts/passphrase never in stdout/stderr/error strings (planted values).
- [ ] Exit-code table test over the full taxonomy.

## Validation

Unit + fixture suite (hostile-tar cases live in issue 11's suite; this issue adds the wizard-level integration on top). A non-engineer walkthrough with the printed guide is tracked in issue 30's docs QA.

## Dependencies

04, 05, 06, 08, 11, 12.

## Non-goals

Web decryptor (v2), Recovery-Sheet flow (bypasses keepsake via `age`; documented in guides), QR scanning inside the CLI (paper QR is scanned by a phone, text pasted).

## Design References

DESIGN §8 (open row), §10.4 (extraction boundary — implemented in issue 11), §5.4, §2.2.
