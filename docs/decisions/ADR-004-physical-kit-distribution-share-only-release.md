# ADR-004: Physical pre-distribution of ciphertext; release sends only share_G

Status: Accepted · 2026-07-10 · Confirmed with owner

## Context

The ciphertext (kit.age, potentially hundreds of MB with documents/photos) and the key material must reach recipients. The delivery topology determines what each party can do alone (DESIGN §10).

## Decision

- **Ciphertext is pre-distributed physically while the owner is alive**: USB/SD bundle (DESIGN §6.3) + printed guide, handed to each recipient / stored where family can reach it.
- **The dead man's switch releases only `share_G`** — a one-line text secret — by email at trigger time. Email never carries ciphertext.
- The printed guide carries `share_G`'s expected 4-char checksum group as an **anti-phishing token** so recipients can authenticate the release mail against paper they already hold.

## Rationale

| Property | Effect of this topology |
|---|---|
| Third-party operator threat | Mail provider sees share_G only → useless without physical bundle. GitHub holds share_G only → same. No single service ever holds decryptable material. |
| Storage-theft threat | USB thief holds ciphertext + share_R → useless without share_G. |
| Pre-trigger recipient threat | Recipient holds ciphertext + share_R → cannot decrypt until release. |
| Payload size | Unbounded by mail attachment limits; photos/scans OK. |
| Long-horizon availability | Physical media in family custody has no service dependency; email path only needs to move ~200 bytes. |

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| Email everything at trigger time | single-channel compromise = total loss; attachment limits; violates operator threat model |
| Cloud storage + emailed link/key | link rot and provider mortality over 10+ years conflict with availability requirement |
| Give recipients the full key now, ciphertext later | inverts the model; ciphertext delivery after death has no reliable channel |

## Consequences

- Content updates require **re-seal + physical re-distribution** (runbook cadence: annual or on life events). Accepted cost; `keepsake status` surfaces payload drift.
- Media longevity: guide instructs recipients to keep paper (shares/guide are paper-recoverable) and owner to refresh USB media at drill time.
- v1 has exactly one bundle flavor (identical for all recipients) per ADR-002 §4.
