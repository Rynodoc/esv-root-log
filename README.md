# EpiSAPShadowVault — root log

Published checkpoints for the ESV staging environment: one JSON Lines file per environment, one line per Bitcoin-attested sealed epoch. Fetch the file from HERE, not from whoever handed you a proof bundle.

| File | Environment |
|---|---|
| `staging.jsonl` | staging (vault.staging.episapvault.com) |
| `releases.jsonl` | verifier release hashes (added when the first wheel is published) |

## What it is

One public file per environment, JSON Lines, one line per **Bitcoin-attested**
sealed epoch. Each line carries that epoch's signed head, its joint
commitment, and the OpenTimestamps proof resolving the commitment to a
Bitcoin block. The first line is the genesis head (epoch 1) and the
environment's keys.

It exists because R6's independence has to come from somewhere. A
counterparty who pins only the signer key can never rise above
`anchor-unverified`; a checkpoint they took from the server they are
checking proves nothing. The root log is the checkpoint's channel.

## Why it is independent, and why it is hosted where it is

Independence does not come from the hosting. It comes from three things,
and the hosting only serves the first:

1. **Publicity and consistency** — many parties fetch the same file from a
   third-party host (a public GitHub repository, mirrored to the offsite
   object store), so the operator cannot show one counterparty a different
   value than another without a discoverable record. A repository's history
   *can* be rewritten by its owner; that is why the log is not the trust.
2. **The OTS proof on every line** — the trust. Anyone can check a line
   against Bitcoin with the standard `ots` client and no software of ours.
3. **The reader obtains it from the public place**, not from whoever handed
   them the bundle.

Consequences that follow, and are not negotiable:

- Only **attested** epochs are published — never the last sealed one, which
  the operator could still re-sign before Bitcoin sees it.
- **No second signature.** The signed head already carries the signer's
  ML-DSA-65 signature; signing the line again with the same key is
  circular, and a separate "release key" is one more thing to distribute
  for no evidentiary gain. The Bitcoin attestation is the signature that
  matters. Signed git commits are hygiene, not evidence.
- The verifier and the bundle must **not** arrive together from the same
  sender with the checkpoint inside — that is self-certification wearing a
  hat.

## Format

Each line is a **superset of the verifier's anchor document**, so any line pastes straight into
`--anchor`, and the verifier's `--root-log FILE.jsonl` picks the right
line itself:

```json
{"root_log_format":1,"env_id_hex":"…","signer_public_key_hex":"…",
 "checkpoint_epoch":4200,"checkpoint_commitment_hex":"…","sealed_at":"2026-09-13T…Z",
 "signed_head_hex":"…","ots_proof_hex":"…",
 "bitcoin_attestation":{"height":912345,"all_heights":[912345]},
 "published_at":"2026-09-20T…+00:00"}
```

`bitcoin_attestation` is the block height the OTS proof attests (OTS
references blocks by height; the block hash is what a reader confirms
against their own view of Bitcoin). It is deliberately **not** named
`bitcoin_checkpoint`: the verifier reserves that key for the reader's own
SPV trust root, which must be *deeper* than the anchors it confirms.

## Consuming a line (counterparty)

```
esv-verify esv-proof-<id>.bundle --root-log staging.jsonl --record-key <64 hex>
```

The verifier picks the newest published checkpoint at or before the
bundle's epoch, prints which line it pinned, and refuses a line for
another environment or one newer than the bundle. To trust nothing of
ours, confirm the chosen line's attestation with `ots verify` against the
`ots_proof_hex` and the block height.

## `releases.jsonl` — the verifier's own provenance

Beside the checkpoint files, one line per verifier release
(`python -m tools.verifier_release <wheel>`): artefact name, version,
SHA-256, size, the source commit it was built from. The wheel itself is
handed over privately until counsel clears publishing the source; the
HASH is public here so any two counterparties can confirm they hold the
same bytes, and the build is reproducible so anyone with the source can
rebuild and match it. No operator signature over the wheel — that would be
circular; the line sits in the same public history as the checkpoints.

## What it does not do

- It does not reach bare `verified` in-tool: served bundles carry no
  Bitcoin anchor material today (a separate ruling).
- It does not detect equivocation by itself; it makes it *detectable* by
  anyone comparing what they hold against what is published.
- If the operator stops publishing, nothing breaks and nothing is hidden:
  the ceiling stops rising, and the last line's date says so.
