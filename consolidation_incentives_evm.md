# One time consolidation incentives in the EVM

## Abstract

A consolidated set of validators is beneficial for a beacon chain system to lower its resource 
consumption, allow scaling and more expensive slots like SSF. There's on-going research to 
incentivize a permanently consolidated set of validators ([Consolidation incentives in Orbit/Vorbit SSF](https://ethresear.ch/t/consolidation-incentives-in-orbit-vorbit-ssf/21593)).
This is optimal but require consensus changes.

What if we could issue a one-time reward to incentivize and existing set of non-consolidated 
validators to consolidate without consensus changes?

## Background

First, a refresher on consolidations. For a consolidation to be scheduled as pending and then be 
executed, it must satisfy this conditions:

- `assert source_pubkey != target_pubkey`
- `assert source.withdrawal_credential has sender address`
- `assert source is active`
- `assert target is active`
- `assert source.exit_epoch == FAR_FUTURE`
- `assert target.exit_epoch == FAR_FUTURE`
- `assert target.withdrawal_credential is compounding`

Then, once the consolidation is executed this fields change in the source validator:

- `source.exit_epoch = something`
- `source.withdrawable_epoch = something`

In effect, consolidations are an operation that can happen only once, controlled by the validator
record field `exit_epoch`.

## Spec

To prevent economical gamification of the incentive, we can only reward validators that became
active before announcing it. Then it's possible to proof that a certain validator was the source
of a consolidation, and reward its withdrawal credential address.

We can and must check inside a smart contract the following conditions:

- There existed some time in the past a consolidation in the state for some source index
- This source index activated before the start of the program
- This source index has not been rewarded yet
- The address to be rewarded is the withdrawal credentials of the source validator

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

This approach allows validators to consolidate also after announcing the program. A validator can
consolidate anytime and then request the payout by crafting a proof transaction. If a consolidation
exists anyone can craft the proof transaction to issue rewards, so this can be an automated system.
The maximum payout of the program can be computed in advance, so the payout contract is never
insolvent.
