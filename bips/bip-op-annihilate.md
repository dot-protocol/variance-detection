# BIP-??: Ephemeral Single-Use Covenants (ESU) — `OP_ANNIHILATE`

```
  BIP: ??
  Layer: Consensus (soft fork)
  Title: Ephemeral Single-Use Covenants (OP_ANNIHILATE)
  Author: The Harrier Room (DOT Protocol)
  Status: Draft-Sketch (pre-specification)
  Type: Standards Track
  Created: 2026-04-22
  License: BSD-3-Clause
  Requires: BIP-119 (CTV), BIP-348 (CSFS), BIP-341 (Taproot)
```

> **Status warning.** This document is a *sketch*, not a specification.
> Normative sections are deliberately marked **TBD**. Do not implement
> against this text. The goal here is to fix vocabulary and shape,
> so later drafts can argue against a concrete target instead of
> against vapor.

---

## Abstract

This BIP introduces a covenant opcode, provisionally named
`OP_ANNIHILATE`, that lets a UTXO commit to being spent **exactly once
against a named off-chain artifact**. The spending witness must carry a
cryptographic proof that the referenced artifact has been *annihilated*
(rendered unspendable / unredeemable) at its point of origin. Combined
with `OP_CHECKTEMPLATEVERIFY` (BIP-119) and `OP_CHECKSIGFROMSTACK`
(BIP-348), this enables UTXOs to serve as self-describing single-use
packets — the primitive required by the DOT Protocol's unit model.

Consensus already guarantees that a UTXO cannot be spent twice. This
BIP is **not** about preventing double-spend of the UTXO itself. It is
about binding the UTXO's *one* allowed spend to the consumption of an
external commitment, so the unit carries its own non-copyability
rather than inheriting it from global ledger consensus.

## Motivation

Bitcoin Script today can express:

- who may spend a UTXO (keys, scripts, Taproot leaves),
- *how* the spending transaction must be shaped (via CTV),
- signatures over arbitrary stack data (via CSFS).

It cannot natively express: *"this UTXO's single spend is bound to the
destruction of a specific off-chain object, and the spend is invalid
unless that destruction is proven."*

Concretely, the DOT Protocol needs a unit that carries, in one object:

1. **Its own address** — the Taproot output key.
2. **Its own authorization** — a Tapscript leaf.
3. **Its own single-use destruction rule** — this BIP.

Without (3), a DOT wrapper referenced by a UTXO can be duplicated
off-chain and re-presented in other contexts even though the UTXO
itself spends only once. `OP_ANNIHILATE` closes that gap by making the
Bitcoin spend *semantically coupled* to the off-chain wrapper's
nullification.

### Non-goals

- Replacing or weakening the existing double-spend guarantees of Bitcoin.
- Introducing statefulness to Bitcoin Script beyond what CTV already implies.
- Prescribing the format of the off-chain artifact being annihilated.
  The opcode treats the artifact as an opaque hash commitment.

## Specification (TBD — sketch only)

### New opcode

```
OP_ANNIHILATE_COMMIT  (repurposes OP_SUCCESSx, exact slot TBD)
```

Stack behavior (*sketch*):

```
Before: ... <dot_hash> <annihilation_proof>
After:  ... 1   (or script fails)
```

Semantics (*sketch*):

1. Pop `annihilation_proof` and `dot_hash` from the stack.
2. Verify `annihilation_proof` against `dot_hash` under a schema
   defined in a companion BIP (**TBD**). The schema MUST commit to:
   - the input's `outpoint`,
   - the spending transaction's `txid` (or a CTV-equivalent template hash),
   - the `dot_hash` itself.
3. If verification fails, script fails. Otherwise push `1`.

The proof format is deliberately left open in this sketch. Candidate
constructions include:

- **Signature-based nullifier.** A Schnorr signature by a
  commitment key over `(outpoint || dot_hash)`, verifiable by
  a key committed in the Taproot internal key.
  Cheapest; requires a trusted/federated origin.
- **Adaptor-witness nullifier.** Uses an adaptor signature whose
  completion publishes a nullifier; makes annihilation
  observable off-chain without a trusted party.
- **Succinct proof.** A zero-knowledge proof of
  membership-and-removal against an accumulator maintained
  off-chain. Most general, highest verification cost.

Only the signature-based construction is in scope for the first
normative draft. The others are listed so the opcode's shape does
not foreclose them.

### Example script

```
script_pubkey =
    OP_ANNIHILATE_COMMIT <dot_hash>   // commits to DOT wrapper
    OP_CHECKTEMPLATEVERIFY <template> // BIP-119
    OP_CHECKSIGFROMSTACK              // BIP-348

spend_witness =
    <signature> <annihilation_proof> <dot_wrapper>
```

The CTV clause pins the spend's shape so that the annihilation proof
cannot be replayed across differently-shaped spends of the same
UTXO (which consensus already forbids, but matters for mempool-layer
replacements — see *RBF interaction* below).

## Rationale

### Why a new opcode rather than a Tapscript pattern?

An in-script emulation using CSFS alone can check a signature over
`dot_hash`, but it cannot enforce that the signed object is *globally*
consumed — the signature is local data. `OP_ANNIHILATE_COMMIT` exists
to make the *semantic linkage* between the on-chain spend and the
off-chain destruction a consensus-visible fact, not a convention.

### Why compose with CTV and CSFS?

- **CTV** fixes the spending template. Without it, an annihilation
  proof might be valid for multiple candidate spend transactions,
  and only one of them can ultimately confirm — the others still
  "used" the proof in mempool and may leak through RBF paths.
- **CSFS** lets the Tapscript validate a signature whose message is
  assembled from stack items, which is what a nullifier-style
  annihilation proof fundamentally is.

The three together form the minimum kit. `OP_ANNIHILATE_COMMIT` alone
would be underspecified; CTV + CSFS alone cannot express single-use
semantics tied to an external artifact.

### Why "annihilate" and not "burn"?

"Burn" in Bitcoin already colloquially means sending to an
unspendable output. This BIP is about destroying an *off-chain*
commitment at its origin, not about locking satoshis. A distinct
term avoids overloading.

## Backwards Compatibility

Deployed as a soft fork by repurposing a Tapscript `OP_SUCCESSx`
opcode, following the same path as BIP-342. Pre-activation, the
opcode is a no-op success; post-activation, it carries the semantics
above. Outputs created before activation are unaffected; outputs
authored to require `OP_ANNIHILATE_COMMIT` before activation are
trivially spendable and should not be authored.

Activation mechanism: **TBD**. The draft takes no position between
BIP-8, BIP-9, and speedy-trial-style flag days.

## Security Considerations

Because the content below is all TBD in different ways, it is
structured as a checklist for later drafts rather than a
definitive analysis.

- **Proof forgery.** Under the signature-based construction, forgery
  reduces to Schnorr EUF-CMA; standard. Under ZK constructions, the
  proof's soundness assumption becomes part of Bitcoin's consensus
  security envelope — undesirable without overwhelming justification.
- **Replay across chains / forks.** The proof schema MUST bind to a
  chain/network identifier. Otherwise an annihilation proof valid on
  mainnet is replayable on testnet/signet, and vice-versa.
- **Mempool pinning via RBF.** Since the annihilation proof commits
  to the spend template (via CTV), an RBF replacement with a
  different template is invalid, so the proof cannot be re-used
  across replacements. Feerate bumps that preserve the template
  (e.g. via anchor outputs) remain possible. **Open:** interaction
  with package RBF still needs analysis.
- **Proof-of-custody.** The annihilation proof asserts destruction
  at origin but does not assert that the destroyed artifact was
  ever *legitimate*. Legitimacy is a property of the DOT wrapper's
  issuance scheme, which is out of scope.
- **Fee sniping / reorg.** On reorg, the UTXO reappears but the
  off-chain artifact has already been annihilated. The spec MUST
  either (a) allow re-spend with the original proof, or
  (b) accept that deep reorgs can brick such outputs. The draft
  leans toward (a) but this is **TBD**.

## Reference Implementation

Not yet written. A reference implementation will live in
`./reference/` in a follow-up commit and will consist of:

1. A patch against `bitcoin/bitcoin` implementing
   `OP_ANNIHILATE_COMMIT` behind a fork-signal flag.
2. A Python `bip-op-annihilate` module under
   `bitcoin/test/functional/` with:
   - construction of the signature-based proof,
   - CTV template derivation matching the proof,
   - a test vector set covering the security-considerations checklist.
3. An end-to-end functional test spending a DOT-wrapper UTXO.

The 2-week estimate in the seed document refers to steps (1) and (2)
with the signature-based proof only. Steps toward ZK proofs are not
estimated.

## Open Questions

These are the items blocking promotion from Draft-Sketch to Draft.

1. **Proof schema.** Exact bytes of `annihilation_proof`. What
   does it commit to, and in what order? Must be pinned before
   any implementation work.
2. **RBF composition.** Does package RBF introduce a pinning vector
   when the annihilation proof is on an ancestor rather than the
   replaced transaction?
3. **Soft-fork safety.** Repurposing `OP_SUCCESSx` is safe in
   principle, but the CTV-dependency means this BIP cannot activate
   before BIP-119. Sequence and fallback **TBD**.
4. **Lightning HTLCs.** HTLC success/timeout transactions spend
   outputs whose script is fixed at channel-funding time. If an
   HTLC output ever wanted annihilation semantics, both parties
   would need to pre-commit to the DOT wrapper at channel open.
   Viable but restrictive; document the pattern, don't prescribe it.
5. **Reorg handling.** See security considerations.
6. **Opcode budget.** CSFS + a nullifier check inside one Tapscript
   leaf needs a costing table. **TBD**.

## Prior Art

- **RPOW (Hal Finney, 2004).** Reusable Proofs of Work; a
  centralized server issued single-use tokens backed by hashcash.
  The primitive — a token whose use is globally observable and
  unrepeatable — is the exact ancestor of what this BIP seeks to
  express in Script. RPOW was sidelined, not refuted, by the
  arrival of Bitcoin's UTXO-based double-spend prevention.
- **BIP-119 — `OP_CHECKTEMPLATEVERIFY`** (Jeremy Rubin). Template
  commitments used here to bind the proof to a spend shape.
- **BIP-348 — `OP_CHECKSIGFROMSTACK`**. Signature verification
  over arbitrary stack messages; the verification primitive for
  the signature-based proof construction.
- **BIP-341 — Taproot.** Assumed as the deployment substrate.

## Next Step

Full mathematical treatment of the signature-based proof
construction, followed by a reference patch against
`bitcoin/bitcoin`. Estimate: ~2 weeks of focused work, followed
by a draft PR to `bitcoin/bips` for community review.

---

*// end draft-sketch*
