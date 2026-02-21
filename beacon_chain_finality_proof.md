# ZK Proof of Ethereum Finality for Trustless Bridges

## Problem

To build a trustless bridge from Ethereum, the destination chain needs to verify that a specific block was **finalized** — meaning ≥2/3 of active validator stake attested to it through Casper FFG's two-phase justification/finalization process.

Naively proving this in ZK is potentially too expensive. The bottleneck is not BLS signature verification (which has efficient precompiles in modern zkVMs) but **hashing**. The beacon state uses SHA256-based SSZ Merkle trees, and the validator set alone is ~1M entries deep. Any proof that touches the full state transition or rebuilds the validators tree from scratch requires ~16M+ SHA256 hashes in-circuit.

## Core Insight

We don't need to re-execute the beacon chain state transition. We need to prove a **predicate**: "this checkpoint received ≥2/3 of active stake." We can decompose this into two subproblems with radically different cost profiles:

1. **Maintaining the validator set** — tracking who is active and what their balance and public key are. This changes slowly (~200 mutations per epoch due to churn limits and effective balance hysteresis).

2. **Verifying attestation weight** — proving that validators representing ≥2/3 of total active balance signed off on a target checkpoint. This touches ~700k validators but can be done cheaply if the data structure is ZK-friendly.

The key design decision: **maintain a Poseidon Merkle accumulator mirroring the SSZ validator set**. Update it cheaply via diffs against the real SSZ tree. Query it cheaply in the finality proof.

## Architecture

Two proofs, cleanly separated:

### Proof 1 — Epoch Diff (per epoch, ~6.4 min)

Proves that the Poseidon accumulator was correctly updated to reflect validator set changes between two consecutive beacon states.

**Public inputs:**
- `state_root_1` — previous epoch's beacon state root (already verified on-chain)
- `state_root_2` — new epoch's beacon state root
- `accumulator_1` — previous Poseidon Merkle root (already verified on-chain)
- `total_active_balance_1` — previous total (already verified on-chain)

**Public outputs:**
- `accumulator_2` — new Poseidon Merkle root
- `total_active_balance_2` — new total

**Witness:**
- Tree diff: set of mutations with SSZ Merkle siblings from both trees and Poseidon siblings from the accumulator

**Circuit logic:**

The circuit walks both SSZ validator data trees top-down, opening only branches that diverge between `state_root_1` and `state_root_2`. Where subtrees match (identical hash), they are carried over unchanged in both the SSZ verification and the Poseidon accumulator. Where they diverge, the circuit recurses until it reaches mutated leaves.

At each mutated leaf:
1. Verify the old validator data against the SSZ tree under `state_root_1`
2. Verify the new validator data against the SSZ tree under `state_root_2`
3. Verify the old Poseidon leaf against `accumulator_1`
4. Compute and insert the new Poseidon leaf, producing an intermediate root progressing toward `accumulator_2`
5. Derive `is_active` from `activation_epoch` and `exit_epoch` (two integer comparisons)
6. Accumulate the balance delta

The SSZ trees use a fixed-capacity layout (`VALIDATOR_REGISTRY_LIMIT = 2^40`). The list length is mixed in at the top via `sha256(data_tree_root | length)` — one extra hash per tree. New validator deposits appear as mutations from a zeroed leaf to a populated one.

Completeness is guaranteed because the circuit reconstructs both SSZ data tree roots from the same diff walk. Any omitted mutation would cause the reconstructed root to not match `state_root_2`. The Poseidon accumulator root is a **computed output**, not a trusted input.

**Cost: ~4,000 SHA256 + ~4,000 Poseidon hashes for ~200 mutations.** Trivially provable.

### Proof 2 — Finality (per finality event, ~12.8 min)

Proves that a checkpoint received ≥2/3 of active balance in attestations with valid BLS signatures.

**Public inputs:**
- `poseidon_root` — from the latest verified epoch diff proof
- `total_active_balance` — from the latest verified epoch diff proof
- `finalized_checkpoint` — the checkpoint being proven (epoch + block root)

**Witness:**
- Attestations: each with `aggregation_bits`, `target_checkpoint`, `bls_aggregate_signature`, and committee validator indices
- Per attesting validator: Poseidon Merkle proof to the leaf in the accumulator

**Circuit logic:**

```
attesting_balance = 0

for each attestation:
    assert target_checkpoint == claimed_checkpoint
    
    for each participating validator (bit set in aggregation_bits):
        (pubkey, active_eff_balance) = poseidon_merkle_verify(root, index, proof)
        attesting_balance += active_eff_balance
        accumulate pubkey into BLS aggregate
    
    bls_verify(aggregate_pubkey, signing_root(target_checkpoint), signature)

assert attesting_balance * 3 >= total_active_balance * 2
```

Double-counting is impossible by protocol design: a validator voting twice for the same target epoch is a slashable offense (double vote rule). The protocol's economic guarantees give us uniqueness for free — no in-circuit deduplication needed.

**Cost: ~1M Poseidon hashes (rebuild tree once, read attesting validators by index) + BLS verification (precompile).**. The BLS is cheap with zkVM precompiles.

### Poseidon Accumulator

A Merkle tree using Poseidon hashing, mirroring the SSZ validators list structure. Same fixed-size layout, same leaf indices. Each leaf stores only what the finality proof needs:

```
leaf[i] = Poseidon(pubkey_lo, pubkey_hi, active_effective_balance)
```

Where `active_effective_balance = is_active ? effective_balance : 0`. This collapses the activity check into the leaf value, so the finality proof simply sums the third component without branching.

The tree is never stored on-chain or in the beacon state. Only its root hash lives on the destination chain contract, updated by each epoch diff proof.

## Cost Summary

| Component | Frequency | Hashing Cost | Equivalent SHA256 |
|---|---|---|---|
| Epoch diff proof | Every 6.4 min | ~4k SHA256 + ~4k Poseidon | ~4,040 |
| Finality proof | Every 6.4 min | ~1M Poseidon + BLS precompile | ~10,000 |
| **Total per finality** | | | **~14,000 SHA256 equiv** |

Compared to a full state transition proof (~16M+ SHA256), this is a **~1000x reduction**.

## Bootstrap

The initial Poseidon accumulator must be built from the full validator set once. This requires hashing all ~1M validators into the Poseidon tree — approximately 4M Poseidon hashes. This is a one-time cost that can be chunked across multiple recursive proofs. After bootstrap, only the incremental epoch diffs are needed.

## Security Model

- **Finality guarantee**: Same as Ethereum's Casper FFG. A finalized checkpoint cannot be reverted without ≥1/3 of total staked ETH being slashed.
- **Proof soundness**: The ZK proofs guarantee computational integrity. A malicious prover cannot fabricate a finality proof without valid BLS signatures from ≥2/3 of active stake.
- **Accumulator integrity**: The epoch diff proof guarantees completeness by reconstructing both SSZ tree roots from the same diff walk. Any omitted or fabricated mutation causes root mismatch.
- **No sync committee dependency**: Unlike Altair light client approaches (512 validators, ~16k ETH security), this proves against the full validator set (~1M validators, ~34M ETH security).
- **No spec changes required**: The proof works against the existing beacon state format. No need for extra accumulators in the state.

## Confirmation Latency vs Security

Ethereum offers three levels of block confirmation, each with different latency and security properties:

| Level | Latency | Security basis |
|---|---|---|
| Fast confirmation rule | ~12 seconds (1 slot) | Synchrony assumption: honest weight accumulates faster than attacker can compensate |
| Justification | ~6.4 min (1 epoch) | Supermajority signal: ≥2/3 of stake voted for this checkpoint |
| Finalization | ~12.8 min (2 epochs) | Economic: reverting requires ≥1/3 of total stake slashed (~11M ETH) |

**Fast confirmation** relies on a synchrony model. Under bounded message delay, once a block accumulates enough LMD-GHOST weight, honest validators keep reinforcing it faster than any competing branch can grow. The safety is real but conditional — if the synchrony assumption breaks (partition, censorship, large delays), reorgs become possible with no slashing consequence.

**Justification** is a supermajority vote link `source → target` with ≥2/3 of active stake. It signals strong agreement but does not provide economic irreversibility. A conflicting checkpoint can be justified later without any validator being slashed.

**Finalization** requires two consecutive justified checkpoints (`parent → child`). This is the only level that provides economic safety: reverting a finalized checkpoint forces ≥1/3 of stake into slashable votes.

For a trustless bridge, only finalization provides unconditional security. Fast confirmation and justification both have failure modes where the bridge could accept a block that later gets reorged — without any economic recourse.

### Why justification alone is not safe

Casper FFG defines two slashing conditions for validator votes `(source_epoch → target_epoch)`:

1. **Double vote**: two votes with the same target epoch but different targets
2. **Surround vote**: one vote's epoch range strictly contains another's

Crucially, votes that merely *cross* on the epoch line are legal. This is what allows conflicting justified checkpoints to coexist without slashing.

**Concrete example.** Consider two branches diverging from genesis (epoch 0), with conflicting checkpoints X at epoch 2 and Y at epoch 3:

```
        G (epoch 0)
       / \
      /   \
    X(2)  Y(3)
```

A validator can honestly vote for both branches across epochs:

```
Vote 1:  0 -----> 2   (source=0, target=2, justifying X)
Vote 2:     1 -----> 3   (source=1, target=3, justifying Y)
```

Check slashing rules:
- Double vote? Targets 2 ≠ 3 — no.
- Surround? Need `0 < 1 < 3 < 2` — false. The intervals cross but neither contains the other.

Both votes are legal. No custom software or duplicated keys required — this happens naturally when different parts of the network temporarily see different forks. Validators vote according to their local fork-choice view, and that view can shift between epochs as new blocks and attestations arrive.

If both votes reach ≥2/3 participation (which requires overlapping validator sets, guaranteed by the pigeonhole principle: `2/3 + 2/3 - 1 = 1/3` minimum overlap), both X and Y become justified on conflicting branches. Zero slashing.

**Why finalization forces slashing.** To finalize X, the chain needs a second justified checkpoint on top of it — a vote `2 → 3`. But a validator that already voted `1 → 3` (justifying Y) would now have:

```
Existing:  1 → 3
New:       2 → 3
```

Same target epoch (3) — this is a **double vote**, slashable. The two supermajorities must overlap by ≥1/3, so at least one-third of total stake would be provably slashable.

The geometric intuition: single vote intervals can cross freely on the epoch line. But chaining two consecutive intervals on one branch forces alignment with any crossing interval on the other branch, collapsing the crossing into a shared endpoint — which is exactly what the double vote rule forbids.

```
Branch A:  0 ----- 2
                 2 ----- 3   ← forces target=3

Branch B:     1 ----- 3      ← conflicts: same target=3, different source
```

This is why Casper FFG requires two rounds. The first round (justification) allows ambiguity for liveness — validators can change their view across epochs without penalty. The second round (finalization) turns that ambiguity into accountability by forcing the vote geometry into contradiction.

