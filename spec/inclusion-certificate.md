# Label 309 — Inclusion Certificate (informative companion)

## Status

**Informative.** This document describes a portable certificate format that is an
**application of** the normative Label 309 specification ([`label-309.md`](./label-309.md)) —
it composes the existing top-level Merkle list commitment (`merkle[i]`), the
RFC 9162 §2.1.1 inclusion proof, and the COSE verifiable-data-structure encoding
that the normative document already defines. It **does not change the wire
format**: no new on-chain field, no new metadata label, no new algorithm
identifier. A conformant Label 309 verifier needs nothing from this document to
verify an on-chain record. Conversely, the certificate format below can be
implemented and verified byte-for-byte by any third party from the normative
specification plus this companion.

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHOULD**, **MAY**,
and **OPTIONAL** in this document are to be interpreted as described in
[RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and
[RFC 8174](https://www.rfc-editor.org/rfc/rfc8174) when, and only when, they appear
in capitals. Byte strings are shown as lowercase hexadecimal with no `0x` prefix.
CBOR types follow [RFC 8949](https://www.rfc-editor.org/rfc/rfc8949); canonical
encoding is the deterministic profile of [RFC 8949 §4.2.1](https://www.rfc-editor.org/rfc/rfc8949#section-4.2.1).
JSON field names are `snake_case`.

## 1. Purpose

A Label 309 record **MAY** carry a top-level Merkle list commitment
(`merkle[i]` — see the "Top-level list commitments" subsection of
[`label-309.md`](./label-309.md)), binding the transaction to an ordered list of
32-byte leaves through an [RFC 9162](https://www.rfc-editor.org/rfc/rfc9162)
SHA-256 hash tree whose root is published on chain. The on-chain root, together
with the off-chain leaves-list document, is sufficient to prove that any
individual leaf was a member of the committed set as of the transaction's block
time.

That proof, however, is not by itself a portable artifact. An **inclusion
certificate** packages it as one: a self-contained file pinning one or more leaves
to the published root, embedding the sibling path needed to recompute that root,
and naming the Cardano transaction whose block time witnesses the commitment.
The certificate re-verifies offline from the file plus any public Cardano
explorer — it does not require the off-chain leaves-list, the original publisher,
or any server.

It is the Proof-of-Existence analogue of an OpenTimestamps proof, with two
differences: the timestamp authority is the **Cardano block time** rather than a
calendar server, and the proof is produced and verified entirely without a
trusted intermediary. This preserves the standalone-verifiable invariant of
Label 309.

## 2. Trust model

A certificate makes exactly two **cryptographic** claims, each independently
checkable by anyone with no trust in the certificate's producer:

1. **Inclusion.** `leaf` is the leaf at position `index` of an RFC 9162 SHA-256
   Merkle tree of size `tree_size` whose root is `root`. This is proven by
   recomputing the root from `leaf`, `index`, `tree_size`, and the embedded
   sibling path. It is self-contained — it needs only the certificate file.
2. **Anchoring.** `root` appears verbatim in the `merkle[].root` field of the
   Label 309 record carried by transaction `tx_hash` under metadata label 309.
   This is proven by reading that transaction on **any** public Cardano explorer
   and comparing the bytes. It needs the certificate file plus a public explorer.

Together these prove the content committed as `leaf` existed **on or before** the
block time of `tx_hash`.

The **time** value (`block_time`) is asserted by the public blockchain — read
from a Cardano explorer, never recomputed from transaction bytes (transaction
CBOR carries no timestamp; see the "Chain facts: block time and confirmation
depth" subsection of [`label-309.md`](./label-309.md)). This is the same trust
basis as other blockchain timestamps: a verifier trusts the chain's block time.
A certificate is **NOT** an [RFC 3161](https://www.rfc-editor.org/rfc/rfc3161)
timestamp token and is **NOT** an eIDAS "qualified" electronic timestamp; it is a
blockchain-anchored timestamp. The asserting authority **MUST** be attributed to
the Cardano blockchain, never to the certificate's producer.

### 2.1 Why the proof is unsigned

A strict IETF COSE Receipt
([draft-ietf-cose-merkle-tree-proofs](https://datatracker.ietf.org/doc/draft-ietf-cose-merkle-tree-proofs/))
is a `COSE_Sign1` whose payload is the tree root, signed by an authority. Here the
authority is the blockchain, not a key the producer holds. Signing the root with a
producer-held key would reintroduce trust in that producer and violate the
standalone-verifiable invariant. The certificate therefore emits the IETF
inclusion-proof CBOR structure unchanged and carries the **blockchain anchor in
place of the Sign1 signature** (§5). The proof *math* is byte-identical to the
IETF encoding; the proof is deliberately unsigned and blockchain-anchored.

## 3. Construction this composes

The certificate reuses, without modification, the construction defined normatively
in [`label-309.md`](./label-309.md):

- The list-commitment algorithm `rfc9162-sha256`: a binary Merkle tree per
  [RFC 9162 §2.1.1](https://www.rfc-editor.org/rfc/rfc9162#section-2.1.1) using
  SHA-256, with `leaf = SHA-256(0x00 ‖ d)` and
  `internal = SHA-256(0x01 ‖ L ‖ R)`. This identifier is the IANA "COSE Verifiable
  Data Structure Algorithms" registry codepoint **1**.
- The inclusion proof: for a leaf at `index` in a tree of `tree_size` leaves, the
  ordered list of sibling node hashes from the leaf up to the root, leaf→root
  order. A single-leaf tree has an **empty** proof. The proof length is variable
  for unbalanced trees; the authoritative acceptance check is algorithmic, not a
  length comparison.
- The leaves placed in the tree are the producer's ordered list of 32-byte content
  hashes; a `leaf` in the certificate is one such content hash. The leaves-list
  document's advisory `leaf_alg` (default `sha2-256`) names how a file is hashed to
  reproduce a leaf and carries no verification semantics.

The verification math is byte-for-byte that of RFC 9162 §2.1.3.2. The certificate
does not introduce a new tree, a new leaf-hashing rule, or a new proof encoding.

## 4. JSON certificate (`label-309-inclusion-certificate-v1`)

The primary artifact is a single JSON object. It is human- and machine-readable,
covers one OR many leaves, and is self-contained: every item embeds its full
sibling path, so the file re-verifies indefinitely without the off-chain
leaves-list.

```jsonc
{
  "format": "label-309-inclusion-certificate-v1",
  "generated_at": "2026-06-16T12:00:00.000Z",      ; informational only; NOT trusted
  "anchor": {
    "chain": "cardano",
    "network": "mainnet",                           ; e.g. "mainnet" | "preprod"
    "tx_hash": "…64hex…",                           ; 32-byte Cardano transaction id
    "metadata_label": 309,
    "block_time": 1781611200,                       ; POSIX seconds; explorer-asserted
    "block_time_iso": "2026-06-16T12:00:00.000Z",   ; UTC rendering of block_time
    "block_height": 12345678,                       ; OPTIONAL; explorer-asserted
    "slot": 123456789,                              ; OPTIONAL; explorer-asserted
    "confirmations_at_generation": 1024,            ; OPTIONAL snapshot; not a claim
    "explorer_urls": [ "…", "…" ]                   ; OPTIONAL convenience links
  },
  "merkle": {
    "tree_alg": "rfc9162-sha256",
    "root": "…64hex…",                              ; the on-chain merkle[i].root
    "tree_size": 1024,                              ; MUST equal on-chain leaf_count
    "leaves_list_uri": "ar://<txid>",               ; OPTIONAL source reference
    "leaves_list_url": "https://…/<txid>"           ; OPTIONAL convenience mirror
  },
  "items": [
    {
      "leaf": "…64hex…",                            ; the content hash committed as a leaf
      "leaf_alg": "sha2-256",                        ; OPTIONAL; how to hash a file to get `leaf`
      "index": 42,                                  ; 0-based position in the tree
      "proof": ["…64hex…", "…64hex…"],              ; siblings leaf→root; [] for a single-leaf tree
      "verified": true,                             ; producer's recompute result; NOT trusted
      "label": "contract.pdf",                      ; OPTIONAL user note/filename
      "error": "…"                                  ; OPTIONAL; present iff verified is false
    }
  ],
  "claim": "Each listed hash was included in a Merkle tree whose root was published on the Cardano blockchain in the referenced transaction under metadata label 309; therefore each hash provably existed on or before the stated block time.",
  "verification": {
    "method": "RFC 9162 (Certificate Transparency) SHA-256 inclusion proof. For each item, recompute the Merkle root from leaf+index+tree_size+proof and compare to merkle.root; then confirm merkle.root equals the merkle[].root in the Label 309 record of anchor.tx_hash on any public Cardano explorer.",
    "independent_tools": [ "…", "…" ],              ; OPTIONAL independent re-verification tools
    "requires_issuer_trust": false,                 ; OPTIONAL; false for a public-chain anchor
    "time_asserted_by": "Cardano blockchain (block time), via public explorers"
  }
}
```

Field rules:

- `format` — REQUIRED. Exactly `label-309-inclusion-certificate-v1`.
- `anchor.chain` — REQUIRED. `cardano` for this version.
- `anchor.tx_hash` — REQUIRED. The 64-character lowercase-hex Cardano transaction
  id whose label 309 record carries `merkle.root`.
- `anchor.metadata_label` — REQUIRED. The integer `309`.
- `anchor.block_time` — REQUIRED. POSIX seconds (a non-negative integer). It
  **MUST** map to a calendar year in `1 .. 9999`
  (`0 <= block_time < 253402300800`). The value is explorer-asserted and is the
  timestamp the certificate's existence claim rests on.
- `anchor.block_time_iso` — REQUIRED. The UTC `YYYY-MM-DDTHH:MM:SS.000Z` rendering
  of `block_time`. The fixed millisecond `.000Z` form keeps the rendering
  well-defined and identical across producers.
- `merkle.tree_alg` — REQUIRED. `rfc9162-sha256`.
- `merkle.root` — REQUIRED. The 32-byte tree root as 64-character hex; it **MUST**
  equal the on-chain `merkle[i].root`.
- `merkle.tree_size` — REQUIRED. The leaf count; it **MUST** equal the on-chain
  `merkle[i].leaf_count` and lie in `1 .. 2^32 − 1`.
- `items[].leaf`, `items[].proof[]` — REQUIRED. Raw 32-byte values as hex.
  Producers **MUST** emit lowercase hex. Verifiers **MUST** accept either case and
  **MUST** reject any non-hex character (including leading, trailing, or embedded
  whitespace) and any odd-length string.
- `items[].index` — REQUIRED. The 0-based leaf position; it **MUST** satisfy
  `0 <= index < tree_size`.
- `items[].verified` — REQUIRED. The producer's recompute result at generation
  time. A verifier **MUST NOT** trust this boolean and **MUST** recompute the
  proof independently (§6). A target hash absent from the committed set **MUST** be
  recorded as an item with `"verified": false` and an explanatory `"error"` field
  rather than omitted, so the certificate is honest about misses.
- `claim` / `verification` — REQUIRED human-readable framing. The wording **MUST
  NOT** overclaim: it **MUST** attribute the time to the Cardano blockchain and
  **MUST NOT** describe the certificate as a "qualified" timestamp.
- `verification.independent_tools` — OPTIONAL array of strings. Names or
  identifiers of independent tools with which a verifier can recheck the
  certificate's claim (for example a command-line verifier, or any generic
  RFC 9162 / COSE verifiable-data-structure implementation). The list is
  informative: it is neither exhaustive nor authoritative, and a verifier
  **MAY** use any conforming implementation of the algorithm in §6.
- `verification.requires_issuer_trust` — OPTIONAL boolean. Whether validating
  the certificate's claim requires trusting the issuing service. For a
  certificate over a public-blockchain inclusion proof this is `false`: any
  party recomputes the proof from the certificate file and confirms the anchor
  on a public explorer (§6), with no input from the producer. The field is
  informative; trust is established by the recomputation of §6, never by this
  assertion.

The JSON **SHOULD** be emitted with stable key order (as shown above) and 2-space
indentation for the human-readable form; a compact single-line form is also
permitted. Because cross-language JSON serializers differ on whitespace and number
formatting, two independent producers are expected to agree on the JSON's
**semantic content** (the same fields with the same values, in the normative item
key order) rather than on a byte-identical serialization. The byte-exact
interoperability surface is the CBOR proof of §5.

## 5. CBOR inclusion proof (COSE / RFC 9162 aligned)

Each item **MAY** additionally be exported as a compact CBOR artifact whose
proof structure is byte-identical to
[draft-ietf-cose-merkle-tree-proofs](https://datatracker.ietf.org/doc/draft-ietf-cose-merkle-tree-proofs/),
so any RFC 9162 / COSE verifiable-data-structure verifier reads the proof math
directly. It carries no absolute block time and no human-readable claim — those
live in the JSON of §4. It is blockchain-anchored and unsigned (§2.1).

The artifact is a canonical-CBOR map ([RFC 8949 §4.2.1](https://www.rfc-editor.org/rfc/rfc8949#section-4.2.1)):

```cddl
; The bare IETF inclusion proof — extractable on its own for a pure COSE verifier.
inclusion-proof = bstr .cbor [ tree_size: uint, leaf_index: uint, [ + bstr ] ]

inclusion-certificate-proof = {
  "vds":             1,                  ; RFC9162_SHA256 — IANA COSE VDS Algorithms codepoint 1
  "inclusion_proof": inclusion-proof,    ; the IETF bstr .cbor array above
  "root":            bytes .size 32,     ; the value an IETF Receipt carries as the Sign1 payload
  "anchor": {                            ; blockchain anchor in place of a TSA / Sign1 signature
    "chain":          "cardano",
    "network":        tstr,
    "tx_hash":        bytes,             ; 32-byte Cardano transaction id
    "metadata_label": 309
  },
  "leaf":            bytes .size 32,     ; convenience: the committed content hash
  ? "leaf_alg":      tstr
}
```

- `vds` — REQUIRED. The IANA "COSE Verifiable Data Structure Algorithms" codepoint
  `1` (RFC 9162 SHA-256).
- `inclusion_proof` — REQUIRED. The bare IETF inclusion proof, i.e. the CBOR byte
  string whose contents are the canonical CBOR encoding of
  `[tree_size, leaf_index, inclusion_path]`, where `inclusion_path` is the ordered
  array of 32-byte sibling node hashes (leaf→root order; an empty array for a
  single-leaf tree). This nested byte string **MUST** be extractable and verifiable
  on its own by a pure COSE / RFC 9162 verifier.
- `root` — REQUIRED. The 32-byte tree root.
- `anchor` — REQUIRED. The blockchain anchor: `chain`, `network`, the 32-byte
  `tx_hash`, and `metadata_label` (`309`).
- `leaf` — REQUIRED. The committed 32-byte content hash, carried for convenience.
- `leaf_alg` — OPTIONAL. An informative annotation; no verification semantics.

Because the encoding is canonical CBOR, the artifact for a given input is
deterministic: a fixed leaves-list and target set yield byte-identical CBOR across
independent implementations. This CBOR is the certificate's byte-exact
interoperability anchor.

## 6. Verification algorithm

A verifier checks a certificate with no input from its producer:

1. **Structural / range checks.** Confirm `format`, `merkle.tree_alg`, and the
   required fields are present. Decode every `root` / `leaf` / `proof[]` entry as
   even-length hex to exactly 32 bytes, rejecting any non-hex or odd-length value.
   Confirm `tree_size` and each `index` are exact integers with
   `1 <= tree_size <= 2^32 − 1` and `0 <= index < tree_size`. A certificate
   claiming an out-of-range `tree_size` or `index` **MUST NOT** verify (this guards
   against integer-fold truncation in a naïve verifier).
2. **Recompute each item's root** from `leaf`, `index`, `tree_size`, and `proof[]`
   per RFC 9162 §2.1.3.2 (`leaf = SHA-256(0x00 ‖ leaf_digest)`,
   `node = SHA-256(0x01 ‖ L ‖ R)`, splitting at the largest power of two strictly
   below the running subtree size; an empty proof for a single-leaf tree). Compare
   the recomputed root to `merkle.root` **byte-for-byte** (a constant-time compare
   is RECOMMENDED). The item verifies iff they are equal. The verifier **MUST NOT**
   rely on the stored `items[].verified` boolean.
3. **Confirm the anchor on chain.** Fetch `anchor.tx_hash` on any public Cardano
   explorer, read its label 309 metadata record, and confirm `merkle.root` equals
   the record's `merkle[i].root` byte-for-byte. The block time read from the
   explorer for that transaction is the timestamp the certificate asserts; a
   verifier for which block time is load-bearing **SHOULD** cross-check it across
   at least two independent explorers.

Steps 1–2 are self-contained (no network). Step 3 is the single network read, and
it is directed at a verifier-chosen explorer — never at the certificate's producer.
When (and only when) both the inclusion recompute (step 2) and the on-chain root
match (step 3) succeed, the leaf is proven to have existed on or before the
transaction's block time.

For a CBOR proof (§5), step 2 operates on the bare `inclusion_proof` byte string —
decode its `[tree_size, leaf_index, inclusion_path]`, recompute, and compare
against `root`; step 3 reads `anchor.tx_hash` exactly as above.

## 7. Reference implementations

Open-source reference implementations of the build and verify paths exist in the
public Label 309 tooling (TypeScript, Python, and Rust SDKs with byte-parity
across the CBOR proof, a command-line tool, and in-browser verification). They are
not required to produce or check a certificate: this document plus
[`label-309.md`](./label-309.md) fully specify the format, and any independent
implementation that follows §4–§6 interoperates byte-for-byte at the CBOR proof
and field/value-for-value at the JSON certificate.

## License

This companion document is licensed under **CC-BY-4.0** (see
[`../LICENSE-docs`](../LICENSE-docs)), matching the specification prose it
accompanies.
