# ADR-002: age scrypt-mode encryption with a Shamir-split random KEK

Status: Accepted · 2026-07-10

## Context

Threat model (all four selected by owner): recipients must not decrypt pre-trigger; stolen media must be safe; no single third-party service may be able to decrypt; the kit must never become undecryptable (availability). See DESIGN §5, §10; research 02.

## Decision

1. Payload encrypted as a **standard age v1 file** with a single **ScryptRecipient**; work factor pinned to 18.
2. The passphrase is the sharetext encoding (DESIGN §7) of a random 32-byte **KEK** — full-entropy, so scrypt cost is not security-load-bearing.
3. KEK split with **Shamir 2-of-2** (GF(2^8), Vault implementation): `share_R` (all recipients, physical) + `share_G` (GitHub Actions secret).
4. All recipients hold the **identical** `share_R` (prevents recipient-collusion decryption by construction; revocation = rotation).
5. Shamir code **vendored from `hashicorp/vault` tag `v1.14.8`** — the last MPL-2.0 release (current Vault is BUSL-1.1 and not redistributable in an OSS project). Files keep MPL-2.0 headers; `LICENSES/MPL-2.0.txt` + provenance in `NOTICE`.
6. A printed **Recovery Sheet** (the passphrase itself) in the owner's safe is the availability backstop and the keepsake-free decryption path (`age -d`).

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| age X25519 identities per recipient | recipients must guard private keys for years; not paper-friendly |
| Custom AES-GCM container | format dies with the project; violates availability requirement |
| OpenPGP | UX and implementation sprawl |
| Import `github.com/hashicorp/vault/shamir` as module | current versions BUSL-1.1; module drags the full Vault dependency tree |
| Write our own Shamir | hand-rolled crypto in a decade-scale product |
| XOR 2-of-2 split | equal v1 capability but no growth path to k-of-n trustees (v2) |

## Consequences

- CI must include an interop gate: a kit sealed by keepsake decrypts with the pinned `age` CLI, and vice-versa test vector (DESIGN §5.4, KU-7).
- Project license (Apache-2.0, confirmed 2026-07-10) coexists with vendored MPL-2.0 files — standard file-level MPL compliance.
- 33-byte shares → 53-char sharetext strings + 4-char checksum; QR + typed entry both supported.
