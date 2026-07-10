# Specification

This directory holds the single normative document of the **Label 309** standard:
[`label-309.md`](./label-309.md). It is an open standard for **Proof of Existence
(PoE)** anchored on the Cardano blockchain. A publisher hashes content and
records the digest — together with optional metadata — on-chain under Cardano
transaction **metadata label 309**. Anyone holding the transaction reference can
later prove that the content existed on or before the block time, without
trusting the publisher, their domain, or any server.

## The document

[`label-309.md`](./label-309.md) is structured against the
[CIP-0001 template](https://github.com/cardano-foundation/CIPs/blob/master/CIP-0001/README.md)
(preamble header block, Abstract, Motivation, Specification, Rationale, Path to
Active, Copyright), and was submitted to `cardano-foundation/CIPs` and accepted
as [CIP-0190](https://github.com/cardano-foundation/CIPs/tree/master/CIP-0190).

## Status

**Working draft — authoring in progress.**

This is a pre-1.0 working draft. It has been accepted into the Cardano CIP
repository as [CIP-0190](https://github.com/cardano-foundation/CIPs/tree/master/CIP-0190)
(CIP lifecycle status: Proposed). This repository is the reference home for the
standard.

## Machine-readable companions

The prose in `label-309.md` references machine-readable artifacts that are
**normative companions** to the specification — an implementation that disagrees
with them does not conform:

- [`../cddl/`](../cddl) — the CDDL grammar for the on-chain record.
- [`../schemas/`](../schemas) — JSON Schemas for the record and its companions.
- [`../registries/`](../registries) — the algorithm registries (hash, KDF,
  signature, KEM, AEAD) and the structural error-code catalogue.
- [`../conformance/`](../conformance) — the byte-pinned conformance vectors that
  serve as the cross-implementation tie-breaker.

## License

The specification prose in this directory is licensed under **CC-BY-4.0** (see
[`../LICENSE-docs`](../LICENSE-docs)). The machine-readable companions —
CDDL, JSON Schemas, registries, and conformance vectors — are licensed under
**Apache-2.0** (see [`../LICENSE`](../LICENSE)).
