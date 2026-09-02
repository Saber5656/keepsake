# Review resolution record

- Repository: `Saber5656/keepsake`
- Pull request: #1
- Parent head observed before this addendum: `000104f0dde6592d716b34aa4cac1f128f70d344`
- Scope: existing review threads only; no new Bot review is requested.
- This document records design-level resolutions and focused verification gates. It does not claim implementation or test completion.

## Thread `PRRT_kwDOTNkH5s6QAWN7`

### Fold previous check-in into monitor facts

- Finding: The existing review thread `PRRT_kwDOTNkH5s6QAWN7` identifies this contract gap.
- Normative resolution: Persist the latest accepted check-in facts in monitor state and derive `last_valid_checkin` from that state plus the current dispatch result; replay rejection of the same signed statement must not erase the last-valid fact.
- Focused verification before resolving this thread: Apply one valid check-in, run the monitor again after replay rejection and outside the dispatch window, and assert the monitor still sees the persisted last-valid check-in.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkH5s6QAWN-`

### Allow Recovery Sheet passphrase in guide output

- Finding: The existing review thread `PRRT_kwDOTNkH5s6QAWN-` identifies this contract gap.
- Normative resolution: Treat the Recovery Sheet passphrase as an intentionally printed owner-held secret: permit it only in the owner-safe guide/QR output, while excluding it from `share_G` and `state/seal-manifest.json`; keep other secret-leak checks strict.
- Focused verification before resolving this thread: Generate the guide, share, and manifest fixtures and assert the passphrase appears only in the approved guide locations and never in GitHub-held artifacts.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkH5s6QAWOA`

### Expose dry-run in the generated monitor workflow

- Finding: The existing review thread `PRRT_kwDOTNkH5s6QAWOA` identifies this contract gap.
- Normative resolution: Add an explicit `dry_run` workflow input to the generated monitor workflow and pass it to `monitor --dry-run`; distinguish this from `drill` and keep the default production path unchanged.
- Focused verification before resolving this thread: Parse the generated workflow and execute its input matrix, asserting dry-run is selectable and the production schedule never silently becomes dry-run.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkH5s6QAWOD`

### Accept tar directory entries produced by the writer

- Finding: The existing review thread `PRRT_kwDOTNkH5s6QAWOD` identifies this contract gap.
- Normative resolution: Allow canonical ancestor directory entries emitted by `WriteTar` even though the manifest enumerates files; validate their type, path containment, and absence of unlisted file entries.
- Focused verification before resolving this thread: Round-trip a payload with nested directories and assert writer output extracts successfully while an unlisted file or unsafe directory is rejected.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkH5s6QAWOG`

### Ignore forbidden kit artifacts before they are committed

- Finding: The existing review thread `PRRT_kwDOTNkH5s6QAWOG` identifies this contract gap.
- Normative resolution: Add repository ignore/pre-commit protection for `kit.age`, `share-R.txt`, and equivalent generated recovery artifacts, and retain a pre-commit tracked-file guard so forbidden files are rejected before publication.
- Focused verification before resolving this thread: Create each forbidden artifact in a fixture, run the staging/guard path, and assert it cannot enter the trigger repository.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkH5s6QAWOK`

### Compare full remote check-in on ambiguous push

- Finding: The existing review thread `PRRT_kwDOTNkH5s6QAWOK` identifies this contract gap.
- Normative resolution: When push outcome is ambiguous, fetch and validate the complete remote check-in document, signature, owner identity, counter, and canonical digest; counter equality alone is insufficient.
- Focused verification before resolving this thread: Simulate a racing same-counter but different/invalid remote document and assert it is not accepted as the caller's successful check-in.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkH5s6QAWOM`

### Normalize downloaded release artifacts for bundle tools

- Finding: The existing review thread `PRRT_kwDOTNkH5s6QAWOM` identifies this contract gap.
- Normative resolution: Make `dist.Ensure` normalize release filenames and layout into the plain tool names required by `bundle.Build`, with a manifest of expected platform artifacts and no ambiguous aliases.
- Focused verification before resolving this thread: Download/extract release-named artifacts into a cold tools directory and assert bundle discovery finds only the normalized names.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkH5s6QAWON`

### Require all later CI gates in the ruleset

- Finding: The existing review thread `PRRT_kwDOTNkH5s6QAWON` identifies this contract gap.
- Normative resolution: Define one versioned required-check inventory that includes later security and release gates such as age `interop` and `release-check`, and update the ruleset contract whenever the plan adds a gate.
- Focused verification before resolving this thread: Compare the ruleset inventory with every issue-plan gate and assert no security-critical or release check is omitted.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkH5s6QAWOR`

### Reject non-zero padding bits in sharetext

- Finding: The existing review thread `PRRT_kwDOTNkH5s6QAWOR` identifies this contract gap.
- Normative resolution: Require canonical Base32 decoding with all unused trailing bits zero for each supported share length; reject otherwise checksum-valid but non-canonical encodings.
- Focused verification before resolving this thread: Mutate only the unused padding bits for 32-byte and 33-byte shares and assert both encodings are rejected.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkH5s6QAWOS`

### Replace the weak checksum anti-phishing token

- Finding: The existing review thread `PRRT_kwDOTNkH5s6QAWOS` identifies this contract gap.
- Normative resolution: Use the dedicated 8-character mail-auth code as the paper-to-mail anti-phishing value; the four-character share checksum is display/integrity metadata only and never an authentication token.
- Focused verification before resolving this thread: Generate a mail and Recovery Sheet and assert the eight-character code is compared while the four-character checksum cannot authorize delivery.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTNkH5s6QAWOV`

### Bound SMTP retry budget to the monitor timeout

- Finding: The existing review thread `PRRT_kwDOTNkH5s6QAWOV` identifies this contract gap.
- Normative resolution: Enforce one monitor-wide deadline and per-recipient attempt budget across primary/secondary SMTP paths; backoff is capped and remaining recipients are stopped or marked pending before the job timeout.
- Focused verification before resolving this thread: Run an outage with ten recipients and delayed providers and assert total work stays within the monitor deadline with deterministic retry outcomes.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Bot review policy

The existing Bot review is not re-triggered for this PR. Replies and thread resolution are performed only after the focused verification conditions above are recorded.