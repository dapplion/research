# One time consolidation incentives in the EVM

## Abstract

A consolidated validator set benefits a beacon chain by reducing resource consumption, improving 
scalability, and enabling more expensive designs such as SSF. There is ongoing research on permanent 
consolidation incentives (e.g. [Orbit/Vorbit SSF](https://ethresear.ch/t/consolidation-incentives-in-orbit-vorbit-ssf/21593)), but those approaches require consensus changes.

This note explores a one-time, retroactive consolidation incentive that can be implemented entirely 
in the EVM, without modifying the consensus layer.

The goal is not to enforce long-term consolidation incentives, but to clean up an existing validator
set by subsidizing a one-time transition.

## Background

A consolidation schedules one active validator (“source”) to transfer its balance into another
active validator (“target”). A consolidation can only be scheduled if:

- `source_pubkey != target_pubkey`
- `source.withdrawal_credentials` are execution-address based
- `source.is_active && target.is_active`
- `source.exit_epoch == FAR_FUTURE`
- `target.exit_epoch == FAR_FUTURE`
- `target.withdrawal_credentials` are compounding

Once executed, consolidation irreversibly mutates the source validator:

- `source.exit_epoch` is set
- `source.withdrawable_epoch` is set

As a consequence, each validator can act as the source of a consolidation at most once, enforced by
the exit_epoch invariant.

## Spec

To prevent economic gaming, the incentive is restricted to validators that were already active
before the program is announced. Eligibility is therefore fixed at a snapshot epoch MAX_EPOCH.

The contract verifies, via Merkle proofs against a finalized beacon state root, that:

- A consolidation exists in the beacon state with `source_index` as the source
- The source validator activated before `MAX_EPOCH`
- The source validator has not claimed rewards before
- The reward is paid to the withdrawal credentials of the source validator

```python
def reward(
	block_number,
	consolidation_index,
	source_index,
	activation_epoch,
	merkle_proof_consolidation,
	merkle_proof_credentials,
	merkle_proof_activation_epoch,
	source_credentials,
):
    assert not rewarded_source_indices[source_index]

    # EIP-4788 (Beacon block root in the EVM)
    block_root = get_block_root_in_evm(block_number)

    # Proofs `source_index` was the source of at least one consolidation
    assert_proof(
    	root=block_root,
    	gindex=compute_gindex($"state_root.pending_consolidations[{consolidation_index}].source_index"),
    	leaf=source_index,
    	proof=merkle_proof_consolidation,
    )

    # Proofs that `source_credentials` are the withdrawal credentials of 
    # the validator record at `source_index`
    assert_proof(
    	root=block_root,
    	gindex=compute_gindex($"state_root.validators[{source_index}].withdrawal_credentials"),
        leaf=source_credentials,
        proof=merkle_proof_credentials,
    )

    # Proofs that the source validator activate before a certain epoch
    assert_proof(
    	root=block_root,
    	gindex=compute_gindex($"state_root.validators[{source_index}].activation_epoch"),
    	leaf=activation_epoch,
    	proof=merkle_proof_activation_epoch
    )
    assert activation_epoch < MAX_EPOCH

    rewarded_source_indices[source_index] = True
    send_reward(strip_address(source_credentials))
```

Consolidations may occur at any time after the program is announced. Once a consolidation exists
on-chain, anyone may submit a proof transaction to trigger payment. This enables permissionless and
automated claiming.

Because the eligibility set is fixed at MAX_EPOCH, the maximum payout is known in advance, and the
contract cannot become insolvent.

## Security considerations and attack analysis

This section analyzes potential attack vectors and explains how the proposed design guards against them.


### Double-claiming

A validator might appear multiple times across beacon states or in multiple `pending_consolidations[...]` entries.

**Defense:**
- Each validator index can only be rewarded once, enforced by checking that it has not been previously rewarded.
- At the consensus layer, a validator can only be the source of a consolidation once, since consolidation sets its `exit_epoch`.

These two layers mutually reinforce the invariant.


### Sybil validator farming

An attacker could attempt to create new validators purely to consolidate them and claim rewards.

**Defense:**  
Eligibility is fixed by requiring the source validator to have activated before a snapshot epoch set strictly prior to announcing the program.

New validators created after that snapshot can never qualify.

Eliminated if the snapshot epoch is pre-announcement and immutable.


### Fake withdrawal recipients

An attacker could try to redirect rewards to an arbitrary address.

**Defense:**  
The reward recipient is derived directly from the proven withdrawal credentials of the source validator record. The caller has no control over the payout destination.

Non-custodial and non-stealable.


### Consolidation-only-for-reward behavior

Validators may choose to consolidate exclusively to trigger the one-time incentive, without committing to long-term consolidation.

This is intended. The incentive explicitly targets one-time validator set cleanup rather than permanent behavioral enforcement.

Acceptable economic behavior, not an exploit.


### Credential-type edge cases

If non–execution-address withdrawal credentials are allowed, converting withdrawal credentials into an EVM address may be undefined or unsafe.

**Mitigation (recommended):**
- Require execution-address–based withdrawal credentials, or
- Explicitly reject unsupported credential types.

This avoids ambiguity and silent mis-direction of rewards.


### Liveness and permissionless claiming

Anyone may submit the reward transaction once a qualifying consolidation exists.

This enables:
- automated claiming,
- third-party liveness guarantees.

Because the reward address is fixed by consensus data, front-running or claim hijacking is impossible.


### Finality and reorg safety

The function `get_block_root_in_evm` relies on **EIP-4788 (Beacon block root in the EVM)** to
retrieve beacon state roots.

While EIP-4788 exposes recent beacon roots without a full light client, it is still necessary to ensure that the referenced root is not subject to reorgs.

This can be enforced by requiring the provided `block_number` to be:
- sufficiently far in the past (e.g. several epochs),
- such that the corresponding beacon block is finalized with overwhelming probability.

Because consolidations remain in the pending queue for at least  
`MIN_VALIDATOR_WITHDRAWABILITY_DELAY` epochs, a consolidation will appear in **multiple consecutive beacon states**. Therefore, even if a conservative delay is required before claiming, the consolidation will still be provable from a later, safe root.

**Effect:**
- reorg-based payout claims are prevented,
- no reliance on short-lived or unstable state roots.

Safe under conservative “blocks-in-the-past” constraints using EIP-4788.


## Summary

This design enables one-time consolidation incentives without consensus changes by:

- fixing eligibility at a pre-announcement snapshot,
- limiting rewards to one per validator index,
- anchoring claims to beacon state roots exposed via EIP-4788,
- relying on the fact that consolidations persist in the pending queue for at least `MIN_VALIDATOR_WITHDRAWABILITY_DELAY` epochs,
- and paying directly to consensus-owned withdrawal credentials.

The remaining risks are limited to conservative root selection and explicit handling of withdrawal
credential types. There should be no viable economic or cryptographic exploit that allows repeated,
inflated, or misdirected payouts under these assumptions (otherwise DM me, feedback welcome!).
