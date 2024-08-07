# Annotated Electra Spec -- The Beacon Chain

**Notice**: This document is a work-in-progress for researchers and implementers.

## Table of contents

<!-- TOC -->
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Constants](#constants)
  - [Misc](#misc)
  - [Withdrawal prefixes](#withdrawal-prefixes)
  - [Domains](#domains)
- [Preset](#preset)
  - [Gwei values](#gwei-values)
  - [Rewards and penalties](#rewards-and-penalties)
  - [State list lengths](#state-list-lengths)
  - [Max operations per block](#max-operations-per-block)
  - [Execution](#execution)
  - [Withdrawals processing](#withdrawals-processing)
- [Configuration](#configuration)
  - [Validator cycle](#validator-cycle)
- [Containers](#containers)
  - [New containers](#new-containers)
    - [`DepositReceipt`](#depositreceipt)
    - [`PendingBalanceDeposit`](#pendingbalancedeposit)
    - [`PendingPartialWithdrawal`](#pendingpartialwithdrawal)
    - [`ExecutionLayerWithdrawalRequest`](#executionlayerwithdrawalrequest)
    - [`Consolidation`](#consolidation)
    - [`SignedConsolidation`](#signedconsolidation)
    - [`PendingConsolidation`](#pendingconsolidation)
  - [Modified Containers](#modified-containers)
    - [`AttesterSlashing`](#attesterslashing)
  - [Extended Containers](#extended-containers)
    - [`Attestation`](#attestation)
    - [`IndexedAttestation`](#indexedattestation)
    - [`BeaconBlockBody`](#beaconblockbody)
    - [`ExecutionPayload`](#executionpayload)
    - [`ExecutionPayloadHeader`](#executionpayloadheader)
    - [`BeaconState`](#beaconstate)
- [Helper functions](#helper-functions)
  - [Predicates](#predicates)
    - [Updated `is_eligible_for_activation_queue`](#updated-is_eligible_for_activation_queue)
    - [New `is_compounding_withdrawal_credential`](#new-is_compounding_withdrawal_credential)
    - [New `has_compounding_withdrawal_credential`](#new-has_compounding_withdrawal_credential)
    - [New `has_execution_withdrawal_credential`](#new-has_execution_withdrawal_credential)
    - [Updated `is_fully_withdrawable_validator`](#updated-is_fully_withdrawable_validator)
    - [Updated `is_partially_withdrawable_validator`](#updated-is_partially_withdrawable_validator)
  - [Misc](#misc-1)
    - [`get_committee_indices`](#get_committee_indices)
    - [`get_validator_max_effective_balance`](#get_validator_max_effective_balance)
  - [Beacon state accessors](#beacon-state-accessors)
    - [New `get_balance_churn_limit`](#new-get_balance_churn_limit)
    - [New `get_activation_exit_churn_limit`](#new-get_activation_exit_churn_limit)
    - [New `get_consolidation_churn_limit`](#new-get_consolidation_churn_limit)
    - [New `get_active_balance`](#new-get_active_balance)
    - [New `get_pending_balance_to_withdraw`](#new-get_pending_balance_to_withdraw)
    - [Modified `get_attesting_indices`](#modified-get_attesting_indices)
  - [Beacon state mutators](#beacon-state-mutators)
    - [Updated  `initiate_validator_exit`](#updated--initiate_validator_exit)
    - [New `switch_to_compounding_validator`](#new-switch_to_compounding_validator)
    - [New `queue_excess_active_balance`](#new-queue_excess_active_balance)
    - [New `queue_entire_balance_and_reset_validator`](#new-queue_entire_balance_and_reset_validator)
    - [New `compute_exit_epoch_and_update_churn`](#new-compute_exit_epoch_and_update_churn)
    - [New `compute_consolidation_epoch_and_update_churn`](#new-compute_consolidation_epoch_and_update_churn)
    - [Updated `slash_validator`](#updated-slash_validator)
- [Beacon chain state transition function](#beacon-chain-state-transition-function)
  - [Epoch processing](#epoch-processing)
    - [Updated `process_epoch`](#updated-process_epoch)
    - [Updated  `process_registry_updates`](#updated--process_registry_updates)
    - [New `process_pending_balance_deposits`](#new-process_pending_balance_deposits)
    - [New `process_pending_consolidations`](#new-process_pending_consolidations)
    - [Updated `process_effective_balance_updates`](#updated-process_effective_balance_updates)
  - [Block processing](#block-processing)
    - [Withdrawals](#withdrawals)
      - [Updated `get_expected_withdrawals`](#updated-get_expected_withdrawals)
      - [Updated `process_withdrawals`](#updated-process_withdrawals)
    - [Execution payload](#execution-payload)
      - [Modified `process_execution_payload`](#modified-process_execution_payload)
    - [Operations](#operations)
      - [Modified `process_operations`](#modified-process_operations)
      - [Attestations](#attestations)
        - [Modified `process_attestation`](#modified-process_attestation)
      - [Deposits](#deposits)
        - [Updated  `apply_deposit`](#updated--apply_deposit)
        - [New `is_valid_deposit_signature`](#new-is_valid_deposit_signature)
        - [Modified `add_validator_to_registry`](#modified-add_validator_to_registry)
        - [Updated `get_validator_from_deposit`](#updated-get_validator_from_deposit)
      - [Voluntary exits](#voluntary-exits)
        - [Updated `process_voluntary_exit`](#updated-process_voluntary_exit)
      - [Execution layer withdrawal requests](#execution-layer-withdrawal-requests)
        - [New `process_execution_layer_withdrawal_request`](#new-process_execution_layer_withdrawal_request)
      - [Deposit receipts](#deposit-receipts)
        - [New `process_deposit_receipt`](#new-process_deposit_receipt)
      - [Consolidations](#consolidations)
        - [New `process_consolidation`](#new-process_consolidation)
- [Testing](#testing)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->
<!-- /TOC -->

## Introduction

Electra is a consensus-layer upgrade containing a number of features. Including:
* [EIP-6110](https://eips.ethereum.org/EIPS/eip-6110): Supply validator deposits on chain
* [EIP-7002](https://eips.ethereum.org/EIPS/eip-7002): Execution layer triggerable exits
* [EIP-7251](https://eips.ethereum.org/EIPS/eip-7251): Increase the MAX_EFFECTIVE_BALANCE
* [EIP-7549](https://eips.ethereum.org/EIPS/eip-7549): Move committee index outside Attestation

*Note:* This specification is built upon [Deneb](../../deneb/beacon_chain.md) and is under active development.

## Deposits and activations

With EIP-7251, validators can now have effective balances ranging from 32 to 2048 ETH. Maintaining weak subjectivity invariants therefore requires us to track the validator set churn by balance, rather than by number of validators. Accordingly, we replace `get_validator_churn_limit` with `get_balance_churn_limit`. Besides this, the deposit and activation flow is quite deeply restructured. The previous flow was the following:
1. `process_deposit` created new validators, with their initial balance being the deposited balance
2. `process_registry_updates` set validators eligible for activation once their balance is at least `MIN_ACTIVATION_BALANCE` (this is unchanged)
3. `process_registry_updates` processed activations of validators whose `activation_eligiblility_epoch` has finalized, up to the churn limit

One possibility with EIP-7251 was to keep this flow unchanged, except for using the balance-based churn in the last step. Ultimately, we moved away from this simpler design in favor of an explicit deposit queue in the state, whose processing is also responsible for enforcing the churn limit:
1. `process_deposit` adds a new`PendingBalanceDeposit` to the queue`state.pending_balance_deposits`. If the deposit is for a new validator, it creates it without any balance.
2. `process_pending_balance_deposits` processes the queue `state.pending_balance_deposits` up to the churn limit. This is where balance increases happen.
3. `process_registry_updates` sets validators eligible for activation once their balance is at least `MIN_ACTIVATION_BALANCE` (unchanged)
4. `process_registry_updates` activates *all* validators validators whose `activation_eligiblility_epoch` has finalized

The upside is that this explicit queue handles all deposits equally, including top-ups, which are therefore now also constrained by the churn limit. As a consequence, we can allow top-ups above the`MIN_ACTIVATION_BALANCE`. Since`MIN_ACTIVATION_BALANCE` and `MAX_EFFECTIVE_BALANCE` do not coincide anymore, this allows a validator to increase their own stake without exiting and activating a new validator, a very useful feature and much requested by stakers.

Note that previously top-ups were not required to go through the churn, because a top-up could not increase a validator's effective balance without a prior, equal loss of balance. In the words of the [phase0 annotated spec](https://github.com/ethereum/annotated-spec/blob/master/phase0/beacon-chain.md#deposits):

> (Note: yes, balance top-ups do kinda get around activation queues, but note that for an attacker to benefit from this, they need to have already lost the ETH that is being topped up since depositing requires 32 ETH and 32 ETH is the maximum effective balance, so it is not an attack vector)

Still, having top-ups also go through the deposit churn does have a beneficial effect on the weak subjectivity period, in that it becomes invariant to changes in the average balance. Previously, [the calculation of the weak subjectivity period](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/weak-subjectivity.md#compute_weak_subjectivity_period) had to take into account the average balance, and the period would get shorter the farther this was from 32 ETH.

### Side effect: freeing up a field in the validator record

While the current design introduces a new queue in the state, it also makes one of `activation_eligibility_epoch` or `activation_epoch` redundant. For example, we could check that a validator is active this way, without ever setting `activation_epoch`.

```python
def is_active_validator(validator: Validator, epoch: Epoch) -> bool:
    """
    Check if ``validator`` is active.
    """
    was_activated = validator.activation_eligibility_epoch <= min(
        state.finalized_checkpoint.epoch,
        compute_activation_exit_epoch(validator.activation_eligiblity_epoch)
    ) 
    return was_activated and epoch < validator.exit_epoch
```

In principle we could then use the `activation_epoch` field in the validator record for something else.

## Constants

The following values are (non-configurable) constants used throughout the specification.

### Misc

| Name | Value | Description |
| - | - | - |
| `UNSET_DEPOSIT_RECEIPTS_START_INDEX` | `uint64(2**64 - 1)` | *[New in Electra:EIP6110]* |
| `FULL_EXIT_REQUEST_AMOUNT` | `uint64(0)` | *[New in Electra:EIP7002]* |

EIP-7002 introduces a new EL request and corresponding CL operation, the `ExecutionLayerWithdrawalRequest`. This encodes either a full exit request, the EL-triggered equivalent to a `VoluntaryExit`, or a partial withdrawal request. An `ExecutionLayerWithdrawalRequest` is interpreted as a full exit request when its `amount` field matches `FULL_EXIT_REQUEST_AMOUNT`.

### Withdrawal prefixes

| Name | Value |
| - | - |
| `BLS_WITHDRAWAL_PREFIX` | `Bytes1('0x00')` |
| `ETH1_ADDRESS_WITHDRAWAL_PREFIX` | `Bytes1('0x01')` |
| `COMPOUNDING_WITHDRAWAL_PREFIX` | `Bytes1('0x02')` |

EIP-7251 adds a third kind of withdrawal credential, only differing from an eth1 credential in the prefix, and which allows validators adopting it to raise their maximum effective balance to `MAX_EFFECTIVE_BALANCE_ELECTRA`, or 2048 ETH. As long as their effective balance is lower than that, their rewards automatically compound.

### Domains

| Name | Value |
| - | - |
| `DOMAIN_CONSOLIDATION` | `DomainType('0x0B000000')` |


## Preset

### Gwei values

| Name | Value |
| - | - |
| `MIN_ACTIVATION_BALANCE` | `Gwei(2**5 * 10**9)`  (= 32,000,000,000) |
| `MAX_EFFECTIVE_BALANCE_ELECTRA` | `Gwei(2**11 * 10**9)` (= 2048,000,000,000) |

### Rewards and penalties

| Name | Value |
| - | - |
| `MIN_SLASHING_PENALTY_QUOTIENT_ELECTRA` | `uint64(2**12)`  (= 4,096) |
| `WHISTLEBLOWER_REWARD_QUOTIENT_ELECTRA` | `uint64(2**12)`  (= 4,096) |

The main goal of EIP-7251 is to allow staking pools to consolidate their validators into ones with larger stake. While this is clearly beneficial to the network in many ways, it does come with some increased risks. In particular, concentrating stake in fewer, larger validators increases the risk that a slashing event might impact a large amount of stake. This would discourage pools from consolidating, reducing the benefits of the EIP. 

To address this, we reduce the `MIN_SLASHING_PENALTY_QUOTIENT` by a factor of 128, from `MIN_SLASHING_PENALTY_QUOTIENT_BELLATRIX = 32` to `MIN_SLASHING_PENALTY_QUOTIENT_ELECTRA = 4096`. With an initial slashing penalty of < 0.025% of one's stake, this ceases to be much of a risk factor. For a 32 ETH validator, the penalty is reduced from the current 1 ETH to < 0.01 ETH. For a 2048 ETH validator it is only 0.5 ETH instead of 64 ETH. 

Readers wonder why it is at all conceivable from a security perspective to eliminate the initial slashing penalty. There are a few reasons for this:
1. The economic security given by Casper-FFG finality has nothing to do with this penalty, and everything to do with the [proportional slashing penalty](https://github.com/ethereum/annotated-spec/blob/master/phase0/beacon-chain.md#aside-anti-correlation-penalties-in-eth2), whose slashed amount is proportional to the amount of slashed stake, to punish large attacks. In other words, economic finality guarantees are fully preserved by this change.
2. Proportional slashing sufficiently punishes *any* attacks where a non-negligible portion of the stake is slashable. For example, even with `MIN_SLASHING_PENALTY_QUOTIENT_BELLATRIX`, the proportional slashing penalty would be larger than the initial slashing penalty in any slashing event involving > 1/96 of the stake. As for attacks involving even less slashable stake, they fall into two categories:
    - The attacking stake and slashed stake roughly coincide. Here the attacker is very small, and just unable to cause much damage. A special case here are proposal equivocations, where a single slashed validator can cause [a decent amount of trouble](https://collective.flashbots.net/t/post-mortem-april-3rd-2023-mev-boost-relay-incident-and-related-timing-issue/1540). A 1 ETH penalty does not do much to prevent this kind of attack.
    - The attacking stake is significantly higher than the slashed stake. This example be the case in ex-ante reorgs or balancing attacks. Such attacks might include one or more proposal equivocations, but otherwise just requires controlling a sizable amount of attesters, without necessarily having them commit slashable offenses. In this case, the maximum slashable stake is anyway negligible compared to the total attacking stake, so a high initial slashing penalty does little to deter the attack.



https://hackmd.io/@dapplion/maxeb_slashing_risks#Initial-slashing-modeling

https://hackmd.io/@5wamg-wlRCCzGh0aoCqR0w/r1aYbH8x0

### State list lengths

| Name | Value | Unit |
| - | - | :-: |
| `PENDING_BALANCE_DEPOSITS_LIMIT` | `uint64(2**27)` (= 134,217,728) | pending balance deposits |
| `PENDING_PARTIAL_WITHDRAWALS_LIMIT` | `uint64(2**27)` (= 134,217,728) | pending partial withdrawals |
| `PENDING_CONSOLIDATIONS_LIMIT` | `uint64(2**18)` (= 262,144) | pending consolidations |

### Max operations per block

| Name | Value |
| - | - |
| `MAX_CONSOLIDATIONS` | `uint64(1)` |

### Execution

| Name | Value | Description |
| - | - | - |
| `MAX_DEPOSIT_RECEIPTS_PER_PAYLOAD` | `uint64(2**13)` (= 8,192) | *[New in Electra:EIP6110]* Maximum number of deposit receipts allowed in each payload |
| `MAX_ATTESTER_SLASHINGS_ELECTRA`   | `2**0` (= 1) | *[New in Electra:EIP7549]* |
| `MAX_ATTESTATIONS_ELECTRA` | `2**3` (= 8) | *[New in Electra:EIP7549]* |
| `MAX_WITHDRAWAL_REQUESTS_PER_PAYLOAD` | `uint64(2**4)` (= 16)| *[New in Electra:EIP7002]* Maximum number of execution layer withdrawal requests in each payload |

### Withdrawals processing

| Name | Value | Description |
| - | - | - |
| `MAX_PENDING_PARTIALS_PER_WITHDRAWALS_SWEEP` | `uint64(2**3)` (= 8)| *[New in Electra:EIP7002]* Maximum number of pending partial withdrawals to process per payload |

## Configuration

### Validator cycle

| Name | Value |
| - | - |
| `MIN_PER_EPOCH_CHURN_LIMIT_ELECTRA` | `Gwei(2**7 * 10**9)` (= 128,000,000,000) | # Equivalent to 4 32 ETH validators
| `MAX_PER_EPOCH_ACTIVATION_EXIT_CHURN_LIMIT` | `Gwei(2**8 * 10**9)` (= 256,000,000,000) |

`MIN_PER_EPOCH_CHURN_LIMIT_ELECTRA` is the equivalent to the previous `MIN_PER_EPOCH_CHURN_LIMIT`, but measured in `Gwei` instead of validators. The minimum churn limit was previously 4 validators, and is therefore now 128 ETH, corresponding to 4 validators with a 32 ETH effective balance. 

`MAX_PER_EPOCH_ACTIVATION_EXIT_CHURN_LIMIT` follows in the footsteps of `MAX_PER_EPOCH_ACTIVATION_CHURN_LIMIT`, [introduced in Deneb](https://github.com/ethereum/consensus-specs/blob/dev/specs/deneb/beacon-chain.md#validator-cycle) and equal to 8 validators, extending the maximum churn limit to exits instead of only to activations. It is set to 256 ETH, corresponding to 8 validators. The reasoning to extend this churn limit cap to exits is related to the introduction of consolidations, which act as both exits and activations at once, and require their own churn limiting. A more in depth discussion is [below](#New-get_consolidation_churn_limit).

## Containers

### New containers

#### `DepositReceipt`

*Note*: The container is new in EIP6110.

```python
class DepositReceipt(Container):
    pubkey: BLSPubkey
    withdrawal_credentials: Bytes32
    amount: Gwei
    signature: BLSSignature
    index: uint64
```

#### `PendingBalanceDeposit`

*Note*: The container is new in EIP7251.

```python
class PendingBalanceDeposit(Container):
    index: ValidatorIndex
    amount: Gwei
```

#### `PendingPartialWithdrawal`

*Note*: The container is new in EIP7251.

```python
class PendingPartialWithdrawal(Container):
    index: ValidatorIndex
    amount: Gwei
    withdrawable_epoch: Epoch
```
#### `ExecutionLayerWithdrawalRequest`

*Note*: The container is new in EIP7251.

```python
class ExecutionLayerWithdrawalRequest(Container):
    source_address: ExecutionAddress
    validator_pubkey: BLSPubkey
    amount: Gwei
```

#### `Consolidation`

*Note*: The container is new in EIP7251.

```python
class Consolidation(Container):
    source_index: ValidatorIndex
    target_index: ValidatorIndex
    epoch: Epoch
```

#### `SignedConsolidation`

*Note*: The container is new in EIP7251.

```python
class SignedConsolidation(Container):
    message: Consolidation
    signature: BLSSignature
```

#### `PendingConsolidation`

*Note*: The container is new in EIP7251.

```python
class PendingConsolidation(Container):
    source_index: ValidatorIndex
    target_index: ValidatorIndex
```

### Modified Containers

#### `AttesterSlashing`

```python
class AttesterSlashing(Container):
    attestation_1: IndexedAttestation  # [Modified in Electra:EIP7549]
    attestation_2: IndexedAttestation  # [Modified in Electra:EIP7549]
```

### Extended Containers

#### `Attestation`

```python
class Attestation(Container):
    aggregation_bits: Bitlist[MAX_VALIDATORS_PER_COMMITTEE * MAX_COMMITTEES_PER_SLOT]  # [Modified in Electra:EIP7549]
    data: AttestationData
    committee_bits: Bitvector[MAX_COMMITTEES_PER_SLOT]  # [New in Electra:EIP7549]
    signature: BLSSignature
```

#### `IndexedAttestation`

```python
class IndexedAttestation(Container):
    # [Modified in Electra:EIP7549]
    attesting_indices: List[ValidatorIndex, MAX_VALIDATORS_PER_COMMITTEE * MAX_COMMITTEES_PER_SLOT]
    data: AttestationData
    signature: BLSSignature
```

#### `BeaconBlockBody`

```python
class BeaconBlockBody(Container):
    randao_reveal: BLSSignature
    eth1_data: Eth1Data  # Eth1 data vote
    graffiti: Bytes32  # Arbitrary data
    # Operations
    proposer_slashings: List[ProposerSlashing, MAX_PROPOSER_SLASHINGS]
    attester_slashings: List[AttesterSlashing, MAX_ATTESTER_SLASHINGS_ELECTRA]  # [Modified in Electra:EIP7549]
    attestations: List[Attestation, MAX_ATTESTATIONS_ELECTRA]  # [Modified in Electra:EIP7549]
    deposits: List[Deposit, MAX_DEPOSITS]
    voluntary_exits: List[SignedVoluntaryExit, MAX_VOLUNTARY_EXITS]
    sync_aggregate: SyncAggregate
    # Execution
    execution_payload: ExecutionPayload  # [Modified in Electra:EIP6110:EIP7002]
    bls_to_execution_changes: List[SignedBLSToExecutionChange, MAX_BLS_TO_EXECUTION_CHANGES]
    blob_kzg_commitments: List[KZGCommitment, MAX_BLOB_COMMITMENTS_PER_BLOCK]
    consolidations: List[SignedConsolidation, MAX_CONSOLIDATIONS]  # [New in Electra:EIP7251]
```

The `BeaconBlockBody` is extended with `consolidations`, a list of `SignedConsolidation` operations. Moreover, the `attestations` and `attester_slashings` fields are modified by EIP-7549, and the `execution_payload` by both EIP-6110 and EIP-7002.

#### `ExecutionPayload`

```python
class ExecutionPayload(Container):
    # Execution block header fields
    parent_hash: Hash32
    fee_recipient: ExecutionAddress
    state_root: Bytes32
    receipts_root: Bytes32
    logs_bloom: ByteVector[BYTES_PER_LOGS_BLOOM]
    prev_randao: Bytes32
    block_number: uint64
    gas_limit: uint64
    gas_used: uint64
    timestamp: uint64
    extra_data: ByteList[MAX_EXTRA_DATA_BYTES]
    base_fee_per_gas: uint256
    # Extra payload fields
    block_hash: Hash32
    transactions: List[Transaction, MAX_TRANSACTIONS_PER_PAYLOAD]
    withdrawals: List[Withdrawal, MAX_WITHDRAWALS_PER_PAYLOAD]
    blob_gas_used: uint64
    excess_blob_gas: uint64
    deposit_receipts: List[DepositReceipt, MAX_DEPOSIT_RECEIPTS_PER_PAYLOAD]  # [New in Electra:EIP6110]
    # [New in Electra:EIP7002:EIP7251]
    withdrawal_requests: List[ExecutionLayerWithdrawalRequest, MAX_WITHDRAWAL_REQUESTS_PER_PAYLOAD]
```

Two new EL -> CL messages are introduced in Electra. EIP-6110 introduces `deposit_receipts`, to bridge validator deposits from the EL to the CL, replacing the previous mechanism based on voting, which relied on an honest majority of validators. EIP-7002 introduces `withdrawal_requests`, which allow a validator with execution layer withdrawal credentials to trigger an exit or partial withdrawal from the EL.

#### `ExecutionPayloadHeader`

```python
class ExecutionPayloadHeader(Container):
    # Execution block header fields
    parent_hash: Hash32
    fee_recipient: ExecutionAddress
    state_root: Bytes32
    receipts_root: Bytes32
    logs_bloom: ByteVector[BYTES_PER_LOGS_BLOOM]
    prev_randao: Bytes32
    block_number: uint64
    gas_limit: uint64
    gas_used: uint64
    timestamp: uint64
    extra_data: ByteList[MAX_EXTRA_DATA_BYTES]
    base_fee_per_gas: uint256
    # Extra payload fields
    block_hash: Hash32
    transactions_root: Root
    withdrawals_root: Root
    blob_gas_used: uint64
    excess_blob_gas: uint64
    deposit_receipts_root: Root  # [New in Electra:EIP6110]
    withdrawal_requests_root: Root  # [New in Electra:EIP7002:EIP7251]
```

#### `BeaconState`

```python
class BeaconState(Container):
    # Versioning
    genesis_time: uint64
    genesis_validators_root: Root
    slot: Slot
    fork: Fork
    # History
    latest_block_header: BeaconBlockHeader
    block_roots: Vector[Root, SLOTS_PER_HISTORICAL_ROOT]
    state_roots: Vector[Root, SLOTS_PER_HISTORICAL_ROOT]
    historical_roots: List[Root, HISTORICAL_ROOTS_LIMIT]
    # Eth1
    eth1_data: Eth1Data
    eth1_data_votes: List[Eth1Data, EPOCHS_PER_ETH1_VOTING_PERIOD * SLOTS_PER_EPOCH]
    eth1_deposit_index: uint64
    # Registry
    validators: List[Validator, VALIDATOR_REGISTRY_LIMIT]
    balances: List[Gwei, VALIDATOR_REGISTRY_LIMIT]
    # Randomness
    randao_mixes: Vector[Bytes32, EPOCHS_PER_HISTORICAL_VECTOR]
    # Slashings
    slashings: Vector[Gwei, EPOCHS_PER_SLASHINGS_VECTOR]  # Per-epoch sums of slashed effective balances
    # Participation
    previous_epoch_participation: List[ParticipationFlags, VALIDATOR_REGISTRY_LIMIT]
    current_epoch_participation: List[ParticipationFlags, VALIDATOR_REGISTRY_LIMIT]
    # Finality
    justification_bits: Bitvector[JUSTIFICATION_BITS_LENGTH]  # Bit set for every recent justified epoch
    previous_justified_checkpoint: Checkpoint
    current_justified_checkpoint: Checkpoint
    finalized_checkpoint: Checkpoint
    # Inactivity
    inactivity_scores: List[uint64, VALIDATOR_REGISTRY_LIMIT]
    # Sync
    current_sync_committee: SyncCommittee
    next_sync_committee: SyncCommittee
    # Execution
    latest_execution_payload_header: ExecutionPayloadHeader  # [Modified in Electra:EIP6110:EIP7002]
    # Withdrawals
    next_withdrawal_index: WithdrawalIndex
    next_withdrawal_validator_index: ValidatorIndex
    # Deep history valid from Capella onwards
    historical_summaries: List[HistoricalSummary, HISTORICAL_ROOTS_LIMIT]
    deposit_receipts_start_index: uint64  # [New in Electra:EIP6110]
    deposit_balance_to_consume: Gwei  # [New in Electra:EIP7251]
    exit_balance_to_consume: Gwei  # [New in Electra:EIP7251]
    earliest_exit_epoch: Epoch  # [New in Electra:EIP7251]
    consolidation_balance_to_consume: Gwei  # [New in Electra:EIP7251]
    earliest_consolidation_epoch: Epoch  # [New in Electra:EIP7251]
    pending_balance_deposits: List[PendingBalanceDeposit, PENDING_BALANCE_DEPOSITS_LIMIT]  # [New in Electra:EIP7251]
    # [New in Electra:EIP7251]
    pending_partial_withdrawals: List[PendingPartialWithdrawal, PENDING_PARTIAL_WITHDRAWALS_LIMIT]
    pending_consolidations: List[PendingConsolidation, PENDING_CONSOLIDATIONS_LIMIT]  # [New in Electra:EIP7251]
```

Three queues are introduced in the `BeaconState`: `pending_balance_deposits`, `pending_partial_withdrawals` and `pending_consolidations`. Moreover, the switch to [balance-based churn-limiting](https://notes.ethereum.org/UmXYMSCORfaTTjRGn5r_EA#Deposits-and-activations) requires tracking unused balance churn that's available in future epochs, which is done through `deposit_balance_to_consume`, `exit_balance_to_consume` and `consolidation_balance_to_consume`. Finally, `earliest_exit_epoch` and`earliest_consolidation_epoch` keep track of the last epochs reached by the exit and consolidation queue, respectively, i.e., the first epochs in which a new exit or consolidation can take place (see [here](#New-compute_exit_epoch_and_update_churn) and [here](#New-compute_consolidation_epoch_and_update_churn) for more details)

## Helper functions

### Predicates

#### Updated `is_eligible_for_activation_queue`

```python
def is_eligible_for_activation_queue(validator: Validator) -> bool:
    """
    Check if ``validator`` is eligible to be placed into the activation queue.
    """
    return (
        validator.activation_eligibility_epoch == FAR_FUTURE_EPOCH
        and validator.effective_balance >= MIN_ACTIVATION_BALANCE  # [Modified in Electra:EIP7251]
    )
```

Here we simply replaced `MAX_EFFECTIVE_BALANCE` with `MIN_ACTIVATION_BALANCE`.

#### New `is_compounding_withdrawal_credential`

```python
def is_compounding_withdrawal_credential(withdrawal_credentials: Bytes32) -> bool:
    return withdrawal_credentials[:1] == COMPOUNDING_WITHDRAWAL_PREFIX
```

#### New `has_compounding_withdrawal_credential`

```python
def has_compounding_withdrawal_credential(validator: Validator) -> bool:
    """
    Check if ``validator`` has an 0x02 prefixed "compounding" withdrawal credential.
    """
    return is_compounding_withdrawal_credential(validator.withdrawal_credentials)
```

#### New `has_execution_withdrawal_credential`

```python
def has_execution_withdrawal_credential(validator: Validator) -> bool:
    """
    Check if ``validator`` has a 0x01 or 0x02 prefixed withdrawal credential.
    """
    return has_compounding_withdrawal_credential(validator) or has_eth1_withdrawal_credential(validator)
```

#### Updated `is_fully_withdrawable_validator`

```python
def is_fully_withdrawable_validator(validator: Validator, balance: Gwei, epoch: Epoch) -> bool:
    """
    Check if ``validator`` is fully withdrawable.
    """
    return (
        has_execution_withdrawal_credential(validator)  # [Modified in Electra:EIP7251]
        and validator.withdrawable_epoch <= epoch
        and balance > 0
    )
```


#### Updated `is_partially_withdrawable_validator`

```python
def is_partially_withdrawable_validator(validator: Validator, balance: Gwei) -> bool:
    """
    Check if ``validator`` is partially withdrawable.
    """
    max_effective_balance = get_validator_max_effective_balance(validator)
    has_max_effective_balance = validator.effective_balance == max_effective_balance  # [Modified in Electra:EIP7251]
    has_excess_balance = balance > max_effective_balance  # [Modified in Electra:EIP7251]
    return (
        has_execution_withdrawal_credential(validator)  # [Modified in Electra:EIP7251]
        and has_max_effective_balance
        and has_excess_balance
    )
```

The max effective balance of a validator can now vary depending on its withdrawal credential. Still, a validator is partially withdrawable whenever it has an execution withdrawal credential, the maximum possible effective balance allowed by its credential, and balance in excess of that.

### Misc

#### `get_committee_indices`

```python
def get_committee_indices(commitee_bits: Bitvector) -> Sequence[CommitteeIndex]:
    return [CommitteeIndex(index) for index, bit in enumerate(commitee_bits) if bit]
```

#### `get_validator_max_effective_balance`

```python
def get_validator_max_effective_balance(validator: Validator) -> Gwei:
    """
    Get max effective balance for ``validator``.
    """
    if has_compounding_withdrawal_credential(validator):
        return MAX_EFFECTIVE_BALANCE_ELECTRA
    else:
        return MIN_ACTIVATION_BALANCE
```
Validators with compounding withdrawal credentials have a maximum effective balance of 2048 ETH, while everyone else keeps the 32 ETH maximum.

### Beacon state accessors

#### New `get_balance_churn_limit`

```python
def get_balance_churn_limit(state: BeaconState) -> Gwei:
    """
    Return the churn limit for the current epoch.
    """
    churn = max(
        MIN_PER_EPOCH_CHURN_LIMIT_ELECTRA,
        get_total_active_balance(state) // CHURN_LIMIT_QUOTIENT
    )
    return churn - churn % EFFECTIVE_BALANCE_INCREMENT
```

The churn limit is now expressed in terms of balance instead of in number of validators. In the second expression we then use the total active balance instead of the validator set size, but maintain the same `CHURN_LIMIT_QUOTIENT`.

#### New `get_activation_exit_churn_limit`

```python
def get_activation_exit_churn_limit(state: BeaconState) -> Gwei:
    """
    Return the churn limit for the current epoch dedicated to activations and exits.
    """
    return min(MAX_PER_EPOCH_ACTIVATION_EXIT_CHURN_LIMIT, get_balance_churn_limit(state))
```

`get_balance_churn_limit` only computes the maximum possible churn limit for activations and exit. This function computes the actual churn limit, which is capped to `MAX_PER_EPOCH_ACTIVATION_EXIT_CHURN_LIMIT`, or 256 ETH. The cap starts taking effect once the total stake is at least $2^{24}$, or about 16M ETH. 

#### New `get_consolidation_churn_limit`

```python
def get_consolidation_churn_limit(state: BeaconState) -> Gwei:
    return get_balance_churn_limit(state) - get_activation_exit_churn_limit(state)
```

A consolidation has the same weak subjectivity impact as an exit and an activation together (see [here](https://notes.ethereum.org/SyjG0FomTxqKki1m2cmU9w#Consolidations-and-churn-limit) for details). Therefore, consolidations also need to be churn-limited. One option would be to have a dedicated churn for consolidations, which is independent of activation and exit churn. The downside is that it would have an impact on the weak subjectivity period. If for example we were to allow as much consolidation churn as activation or exit churn, the weak subjectivity period would be halved, due to a consolidation contributing as much as an exit and an activation. We could set a much smaller consolidation churn and only pay a small weak subjectivity price. Instead, we have opted to cap the churn of both activations and exit at 256 ETH, as discussed [above](#New-get_activation_exit_churn_limit), and to use the "leftover" churn for consolidations. 

If there is less than $2^{24}$ ETH staked, consolidation churn is 0 and there cannot be any consolidation *through this in-protocol feature*, though it is in principle still possible for consolidation to happen just through exits and activations. As long as we have at least $2^{24}$ ETH staked, consolidations are allowed, and the maximum consolidation churn increases the more ETH is staked. At $2^{25}$ ETH staked, which is roughly where we are at the time of writing, the max consolidation churn is 256 ETH per epoch, as for activations and exits. At this level of stake, consolidation of *all* validators would require at least $2^{25} / 2^8 = 2^{17}$ epochs, or ~582 days. More generally, this plot shows the minimum years required for every validator to consolidate versus the total stake.


<iframe src="https://www.desmos.com/calculator/o9uegz2aqr?embed" width="500" height="500" style="border: 1px solid #ccc" frameborder=0></iframe>


#### New `get_active_balance`

```python
def get_active_balance(state: BeaconState, validator_index: ValidatorIndex) -> Gwei:
    max_effective_balance = get_validator_max_effective_balance(state.validators[validator_index])
    return min(state.balances[validator_index], max_effective_balance)
```

This function gets the active balance of a validator, which is the lower of its max effective balance and balance. For example, a validator with execution credentials and a balance of 100 ETH has have active balance of 32 ETH, while one with the same balance and compounding credentials has an active balance of 128 ETH. 
#### New `get_pending_balance_to_withdraw`

```python
def get_pending_balance_to_withdraw(state: BeaconState, validator_index: ValidatorIndex) -> Gwei:
    return sum(
        withdrawal.amount for withdrawal in state.pending_partial_withdrawals if withdrawal.index == validator_index
    )
```

#### Modified `get_attesting_indices`

```python
def get_attesting_indices(state: BeaconState, attestation: Attestation) -> Set[ValidatorIndex]:
    """
    Return the set of attesting indices corresponding to ``aggregation_bits`` and ``committee_bits``.
    """
    output: Set[ValidatorIndex] = set()
    committee_indices = get_committee_indices(attestation.committee_bits)
    committee_offset = 0
    for index in committee_indices:
        committee = get_beacon_committee(state, attestation.data.slot, index)
        committee_attesters = set(
            index for i, index in enumerate(committee) if attestation.aggregation_bits[committee_offset + i])
        output = output.union(committee_attesters)

        committee_offset += len(committee)

    return output
```

### Beacon state mutators

#### Updated  `initiate_validator_exit`

```python
def initiate_validator_exit(state: BeaconState, index: ValidatorIndex) -> None:
    """
    Initiate the exit of the validator with index ``index``.
    """
    # Return if validator already initiated exit
    validator = state.validators[index]
    if validator.exit_epoch != FAR_FUTURE_EPOCH:
        return

    # Compute exit queue epoch [Modified in Electra:EIP7251]
    exit_queue_epoch = compute_exit_epoch_and_update_churn(state, validator.effective_balance)

    # Set validator exit epoch and withdrawable epoch
    validator.exit_epoch = exit_queue_epoch
    validator.withdrawable_epoch = Epoch(validator.exit_epoch + MIN_VALIDATOR_WITHDRAWABILITY_DELAY)
```

The only change here is the computation of the `exit_queue_epoch`, which is offloaded to `compute_exit_epoch_and_update_churn` due to the increased complexity of handling the balance-based churn.

#### New `switch_to_compounding_validator`

```python
def switch_to_compounding_validator(state: BeaconState, index: ValidatorIndex) -> None:
    validator = state.validators[index]
    if has_eth1_withdrawal_credential(validator):
        validator.withdrawal_credentials = COMPOUNDING_WITHDRAWAL_PREFIX + validator.withdrawal_credentials[1:]
        queue_excess_active_balance(state, index)
```

To convert an eth1 withdrawal credential to a compounding credential, all that is required is to change the prefix from `ETH1_ADDRESS_WITHDRAWAL_PREFIX` (`0x01`) to `COMPOUNDING_WITHDRAWAL_PREFIX` (`0x02`), while the withdrawal address stays the same. Just changing the withdrawal credential would be unsafe from a weak-subjectivity perspective, however, because a validator might have balance in excess of `MIN_ACTIVATION_BALANCE` which would become active as soon as the credential change has been made, since the maximum effective balance is immediately raised to `MAX_EFFECTIVE_BALANCE_ELECTRA`. For example, switching a validator with a balance of 2048 ETH to compounding credentials would suddently activate 2016 ETH. Instead, `queue_excess_active_balance` (just below) removes the balance in excess of 32 ETH from the validator and queues it up as a new `PendingBalanceDeposit`, which is going to go through churn limiting in `process_pending_balance_deposits`.

#### New `queue_excess_active_balance`

```python
def queue_excess_active_balance(state: BeaconState, index: ValidatorIndex) -> None:
    balance = state.balances[index]
    if balance > MIN_ACTIVATION_BALANCE:
        excess_balance = balance - MIN_ACTIVATION_BALANCE
        state.balances[index] = MIN_ACTIVATION_BALANCE
        state.pending_balance_deposits.append(
            PendingBalanceDeposit(index=index, amount=excess_balance)
        )
```

Besides being used in `switch_to_compounding_validator` (just above), this function is also used at the fork transition, for validators which have been activated with compounding withdrawal credentials prior to the fork:

```python
for index, validator in enumerate(post.validators):
    if has_compounding_withdrawal_credential(validator):
        queue_excess_active_balance(post, ValidatorIndex(index))
```

The reasoning is the same as above: we do not want the excess balance (anything beyond `MIN_ACTIVATION_BALANCE`) of such validators to all of a sudden become active due to EIP-7251 going live and increasing their maximum effective balance.



#### New `queue_entire_balance_and_reset_validator`
```python
def queue_entire_balance_and_reset_validator(state: BeaconState, index: ValidatorIndex) -> None:
    balance = state.balances[index]
    state.balances[index] = 0
    validator = state.validators[index]
    validator.effective_balance = 0
    validator.activation_eligibility_epoch = FAR_FUTURE_EPOCH
    state.pending_balance_deposits.append(
        PendingBalanceDeposit(index=index, amount=balance)
    )
```

This function is only needed at the fork transition, where we apply it to all validators which are not yet active, by resetting their balance and effective balance to `0` and queueing up a `PendingBalanceDeposit` which will eventually restore their balance. We do this due to the [changes to the deposit and activation processing flow](#Deposits-and-activations), which move the churn limiting from activation processing in `process_registry_updates` to deposit processing in `process_pending_balance_deposits`. This would cause validators whose deposit is processed before the fork but which are not yet active at the fork to instantly be activated by `process_registry_updates`, without any churn limiting. 

https://github.com/ethereum/consensus-specs/pull/3659


#### New `compute_exit_epoch_and_update_churn`

```python
def compute_exit_epoch_and_update_churn(state: BeaconState, exit_balance: Gwei) -> Epoch:
    earliest_exit_epoch = max(state.earliest_exit_epoch, compute_activation_exit_epoch(get_current_epoch(state)))
    per_epoch_churn = get_activation_exit_churn_limit(state)
    # New epoch for exits.
    if state.earliest_exit_epoch < earliest_exit_epoch:
        exit_balance_to_consume = per_epoch_churn
    else:
        exit_balance_to_consume = state.exit_balance_to_consume

    # Exit doesn't fit in the current earliest epoch.
    if exit_balance > exit_balance_to_consume:
        balance_to_process = exit_balance - exit_balance_to_consume
        additional_epochs = (balance_to_process - 1) // per_epoch_churn + 1
        earliest_exit_epoch += additional_epochs
        exit_balance_to_consume += additional_epochs * per_epoch_churn

    # Consume the balance and update state variables.
    state.exit_balance_to_consume = exit_balance_to_consume - exit_balance
    state.earliest_exit_epoch = earliest_exit_epoch

    return state.earliest_exit_epoch
```

This function is called by `initiate_voluntary_exit` whenever an exit is initiated either through a `VoluntaryExit` or an `ExecutionLayerWithdrawalRequest`, in which case the `exit_balance` is the validator's effective balance, as this is what leaves the active set. It is also called in `process_execution_layer_withdrawal_request` when processing a partial withdrawal, because (unlike partial withdrawals through the sweep, which only skim excess balance) EL-initiated partial withdrawals are churn-limited and share the queue with exits. In other words, this is really a withdrawal queue, for the purposes of which exits are just withdrawals of the whole effective balance. 

Previously, `initiate_validator_exit` would keep track of the back of the exit queue simply through the exit epochs in the validator records, easily identifying `exit_queue_epoch`, the earliest epoch in which a new exit can be processed, and `exit_queue_churn`, the already consumed churn in the `exit_queue_epoch`:
```python
    exit_epochs = [v.exit_epoch for v in state.validators if v.exit_epoch != FAR_FUTURE_EPOCH]
    exit_queue_epoch = max(exit_epochs + [compute_activation_exit_epoch(get_current_epoch(state))])
    exit_queue_churn = len([v for v in state.validators if v.exit_epoch == exit_queue_epoch])
```
A consequence of partial withdrawals and exits sharing the same queue is that we cannot rely on these, because partial withdrawals do not set the `exit_epoch`. Therefore, we introduce in the `BeaconState`:
- `earliest_exit_epoch`, representing the earliest epoch whose churn is still at least partially available for consumption (and thus also the earliest `exit_epoch` available to new exits)
- `exit_balance_to_consume`, which tracks the unused churn in the `earliest_exit_epoch`, available to new exits and withdrawals.

Given these and an `exit_balance`, the function computes the new `earliest_exit_epoch` and `exit_balance_to_consume` and updates them in the state. Something to note in the computation is that, when the exit does not fit in the `exit_balance_to_consume` at the current `earliest_exit_epoch`, we compute the additional required epochs as `(balance_to_process - 1) // per_epoch_churn + 1` instead of `balance_to_process // per_epoch_churn + 1`, so that we do not go to the next epoch when `balance_to_process` is exactly a multiple of `per_epoch_churn`. For example, say `earliest_exit_epoch = 10`, `exit_balance_to_consume = 128`, `per_epoch_churn = 256` and `exit_balance = 384`. Then the exit balance fully consumes the remaining churn in this epoch and all of the churn in the next epoch, and we end up with `earliest_exit_epoch = 11` and `exit_balance_to_consume = 0`, rather than with `earliest_exit_epoch = 12` and `exit_balance_to_consume = 256`. The benefit is that the validator can exit in the first epoch with sufficient churn, rather than having to wait an extra epoch.

#### New `compute_consolidation_epoch_and_update_churn`

```python
def compute_consolidation_epoch_and_update_churn(state: BeaconState, consolidation_balance: Gwei) -> Epoch:
    earliest_consolidation_epoch = max(
        state.earliest_consolidation_epoch, compute_activation_exit_epoch(get_current_epoch(state)))
    per_epoch_consolidation_churn = get_consolidation_churn_limit(state)
    # New epoch for consolidations.
    if state.earliest_consolidation_epoch < earliest_consolidation_epoch:
        consolidation_balance_to_consume = per_epoch_consolidation_churn
    else:
        consolidation_balance_to_consume = state.consolidation_balance_to_consume

    # Consolidation doesn't fit in the current earliest epoch.
    if consolidation_balance > consolidation_balance_to_consume:
        balance_to_process = consolidation_balance - consolidation_balance_to_consume
        additional_epochs = (balance_to_process - 1) // per_epoch_consolidation_churn + 1
        earliest_consolidation_epoch += additional_epochs
        consolidation_balance_to_consume += additional_epochs * per_epoch_consolidation_churn

    # Consume the balance and update state variables.
    state.consolidation_balance_to_consume = consolidation_balance_to_consume - consolidation_balance
    state.earliest_consolidation_epoch = earliest_consolidation_epoch

    return state.earliest_consolidation_epoch
```

This function is identical to `compute_exit_epoch_and_update_churn`, except it deals with consolidations instead of exits and partial withdrawals, so it updates `state.consolidation_balance_to_consume` and `state.earliest_consolidation_epoch`.

#### Updated `slash_validator`

```python
def slash_validator(state: BeaconState,
                    slashed_index: ValidatorIndex,
                    whistleblower_index: ValidatorIndex=None) -> None:
    """
    Slash the validator with index ``slashed_index``.
    """
    epoch = get_current_epoch(state)
    initiate_validator_exit(state, slashed_index)
    validator = state.validators[slashed_index]
    validator.slashed = True
    validator.withdrawable_epoch = max(validator.withdrawable_epoch, Epoch(epoch + EPOCHS_PER_SLASHINGS_VECTOR))
    state.slashings[epoch % EPOCHS_PER_SLASHINGS_VECTOR] += validator.effective_balance
    # [Modified in Electra:EIP7251]
    slashing_penalty = validator.effective_balance // MIN_SLASHING_PENALTY_QUOTIENT_ELECTRA
    decrease_balance(state, slashed_index, slashing_penalty)

    # Apply proposer and whistleblower rewards
    proposer_index = get_beacon_proposer_index(state)
    if whistleblower_index is None:
        whistleblower_index = proposer_index
    whistleblower_reward = Gwei(
        validator.effective_balance // WHISTLEBLOWER_REWARD_QUOTIENT_ELECTRA)  # [Modified in Electra:EIP7251]
    proposer_reward = Gwei(whistleblower_reward * PROPOSER_WEIGHT // WEIGHT_DENOMINATOR)
    increase_balance(state, proposer_index, proposer_reward)
    increase_balance(state, whistleblower_index, Gwei(whistleblower_reward - proposer_reward))
```

The only change here is replacing `MIN_SLASHING_PENALTY_QUOTIENT` and `WHISTLEBLOWER_REWARD_QUOTIENT` with new `_Electra` values, as [previously discussed](#Rewards-and-penalties).

## Beacon chain state transition function

### Epoch processing

#### Updated `process_epoch`
```python
def process_epoch(state: BeaconState) -> None:
    process_justification_and_finalization(state)
    process_inactivity_updates(state)
    process_rewards_and_penalties(state)
    process_registry_updates(state)  # [Modified in Electra:EIP7251]
    process_slashings(state)
    process_eth1_data_reset(state)
    process_pending_balance_deposits(state)  # [New in Electra:EIP7251]
    process_pending_consolidations(state)  # [New in Electra:EIP7251]
    process_effective_balance_updates(state)  # [Modified in Electra:EIP7251]
    process_slashings_reset(state)
    process_randao_mixes_reset(state)
    process_historical_summaries_update(state)
    process_participation_flag_updates(state)
    process_sync_committee_updates(state)
```

TODO: check what's important about the order of operations.

#### Updated  `process_registry_updates`

`process_registry_updates` uses the updated definition of `initiate_validator_exit`
and changes how the activation epochs are computed for eligible validators.

```python
def process_registry_updates(state: BeaconState) -> None:
    # Process activation eligibility and ejections
    for index, validator in enumerate(state.validators):
        if is_eligible_for_activation_queue(validator):
            validator.activation_eligibility_epoch = get_current_epoch(state) + 1

        if (
            is_active_validator(validator, get_current_epoch(state))
            and validator.effective_balance <= EJECTION_BALANCE
        ):
            initiate_validator_exit(state, ValidatorIndex(index))

    # Activate all eligible validators
    activation_epoch = compute_activation_exit_epoch(get_current_epoch(state))
    for validator in state.validators:
        if is_eligible_for_activation(state, validator):
            validator.activation_epoch = activation_epoch
```

As [previously discussed](#Deposits-and-activations), we now activate all eligible validators, because the churn limiting is already done as part of `PendingBalanceDeposit` processing, by the function just below.

#### New `process_pending_balance_deposits`

```python

def process_pending_balance_deposits(state: BeaconState) -> None:
    next_epoch = Epoch(get_current_epoch(state) + 1)
	available_for_processing = state.deposit_balance_to_consume + get_activation_exit_churn_limit(state)
	processed_amount = 0
	next_deposit_index = 0
	deposits_to_postpone = []

	for deposit in state.pending_balance_deposits:
            validator = state.validators[deposit.index]
            # Validator is exiting, postpone the deposit until after withdrawable epoch
            if validator.exit_epoch < FAR_FUTURE_EPOCH:
                if next_epoch <= validator.withdrawable_epoch:
                    deposits_to_postpone.append(deposit)
                # Deposited balance will never become active. Increase balance but do not consume churn
                else:
                    increase_balance(state, deposit.index, deposit.amount)
            # Validator is not exiting, attempt to process deposit
            else:
                # Deposit does not fit in the churn, no more deposit processing in this epoch.
                if processed_amount + deposit.amount > available_for_processing:
                    break
                # Deposit fits in the churn, process it. Increase balance and consume churn.
                else:
                    increase_balance(state, deposit.index, deposit.amount)
                    processed_amount += deposit.amount
            # Regardless of how the deposit was handled, we move on in the queue.
            next_deposit_index += 1

	state.pending_balance_deposits = state.pending_balance_deposits[next_deposit_index:]

  
	if len(state.pending_balance_deposits) == 0:
            state.deposit_balance_to_consume = Gwei(0)
	else:
            state.deposit_balance_to_consume = available_for_processing - processed_amount

	state.pending_balance_deposits += deposits_to_postpone

```


This function goes through the list of pending balance deposits and processes as many deposits *to validators that are not exiting or exited* as allowed by the churn , where processing here means just to increase the validator's balance by the deposit amount. Deposits to validators whose exit has been initiated are handled somewhat differently, as we discuss later. 

One complication is that there can be deposits whose amount is larger than the available churn, and thus which need more than one epoch for processing. To deal with such cases, we keep track of `deposit_balance_to_consume` in the state, which accumulates churn contributing towards processing a pending deposit which cannot be fully processed at once.  To this, we add `get_activation_exit_churn_limit(state)`, the churn available in the current epoch, to obtain the total amount of balance which we can process. This way, if a deposit requires multiple epochs of churn to be processed, it just stays at the front of `pending_balance_deposits` until enough epochs have passed that we have accumulated sufficient `deposit_balance_to_consume` to process it. For example, if the churn per epoch is `256` ETH and the deposit at the front of the queue is 512 ETH, we will not process anything and just accumulate this epoch's churn into `deposit_balance_to_consume`, so that by the next epoch we have `512` ETH `available_for_processing` and we can clear the deposit.

An important note here is that we only accumulate churn that contributes towards processing an existing deposit: if instead we successfully process all `pending_balance_deposits`, `deposit_balance_to_consume` is set to `0`, so that unused churn does not accumulate. For example, if there aren't any pending deposits for many epochs, we do not want `deposit_balance_to_consume` to keep growing and eventually allow many many deposits to be processed at once, as this would break the churn invariants underpinning weak subjectivity.

##### Handling of deposits to exiting/exited validators

Let's now discuss why and how we handle differently the pending balance deposits to validators whose exit has already been initiated. Were we to just process these deposits as all others, it would be possible to do the following:
1. Initiate a validator's exit
2. Make a top-up while the validator is still exiting
3. The top-up happens before the exit
4. Once the validator exits, the topped-up balance leaves the validator set without going through the churn

This might not seem like an attack vector, because the top-up does go through the deposit churn, enforced in this very function. Still, there are two problems. One is that [exiting balance has twice as much of a weak-subjectivity impact as depositing the same balance](https://notes.ethereum.org/SyjG0FomTxqKki1m2cmU9w#Consolidations-and-churn-limit), and the strategy above can be used to convert deposit churn into exit churn. More importantly, even if we were to ignore the asymmetry between exits and activations, *churn limit invariants can be violated* even with top-ups going through the deposit churn. The reason for this is subtle: the top-ups can go through the deposit churn slowly, over a long period of time, but *the same balance can then exit much faster*. For example, say the exit queue is one week long, while the activation queue is completely empty, and an attacker executes the strategy above, in particular exiting validators with minimum balance and topping them up to max balance. Crucially, because of the exit queue being full, the large top-ups have one whole week to go through the deposit churn and be processed. Moreover, all exits happen very fast once the week is over, because the validators had minimum balance when the exits were initiated, and so the exits are scheduled very close to each other. With the maximum effective balance being 128x the ejection balance, this can cause up to 128x the exit churn that should be allowed in a given period of time: if 1000 validators with initial balance 16 ETH exit consecutively after being topped-up to 2048 ETH, the period of time in which they exit will experience ~2M ETH of exit churn instead of merely 16k ETH.

As to how we handle such deposits, first note that we cannot just ignore them, because that would effectively burn the deposit. Instead, we simply *postpone* them until after the validator's withdrawable epoch, by moving them at the end of the queue. More precisely, we accumulate the postponed deposits in `deposits_to_postpone` and only append them to the queue at the end. This is to avoid accumulating `deposit_balance_to_consume` when all deposits have been either processed or postponed and there's no ongoing processing of a multi-epoch deposit. After the withdrawable epoch, we increase the validator's balance by the deposit amount *without any churn limiting*. Ignoring consolidations for the moment, it would actually be safe to do this as soon as the validator's exit epoch is reached, because this balance would then go straight to an inactive validator, never impacting the active validator set and eventually just getting withdrawn without any consequence. With consolidations, things are a bit trickier because even the balance of an already exited validator might end up becoming active by being sent to an active consolidation target (which might itself be exiting, with the same effect as the attack we discussed). As we see just below, `process_pending_consolidation` processes the balance transfer from source to target in the epoch transition to the withdrawable epoch of the source validator, i.e., when `source.withdrawable_epoch == next_epoch`. It is therefore safe to process deposits starting from the epoch transition to the following epoch, i.e., when `validator.withdrawable_epoch > next_epoch`, because at this point any consolidation processing would have already happened and the deposited balance can then only be withdrawn. 

To clarify with an example, say that the withdrawable epoch of `validator` is N. In the epoch transition to epoch N (at the end of epoch N-1, when `next_epoch` is N), `process_pending_consolidation` processes any consolidation with `validator` as the source. During epoch N, the balance of `validator` can first be withdrawn. Starting from the subsequent epoch transition, from epoch N to N+1 (`next_epoch` is N+1), `process_pending_balance_deposits` is free to process any deposits to `validator`. At this point, the deposited balance can only be withdrawn, because any existing `pending_consolidation` would have already been processed in the epoch transition to N, and no further `pending_consolidation` can be queued up for an exiting/exited `validator`.

#### New `process_pending_consolidations`

```python
def process_pending_consolidations(state: BeaconState) -> None:
    next_epoch = Epoch(get_current_epoch(state) + 1)
    next_pending_consolidation = 0
    for pending_consolidation in state.pending_consolidations:
        source_validator = state.validators[pending_consolidation.source_index]
        if source_validator.slashed:
            next_pending_consolidation += 1
            continue
        if source_validator.withdrawable_epoch > next_epoch:
            break

        # Churn any target excess active balance of target and raise its max
        switch_to_compounding_validator(state, pending_consolidation.target_index)
        # Move active balance to target. Excess balance is withdrawable.
        active_balance = get_active_balance(state, pending_consolidation.source_index)
        decrease_balance(state, pending_consolidation.source_index, active_balance)
        increase_balance(state, pending_consolidation.target_index, active_balance)
        next_pending_consolidation += 1

    state.pending_consolidations = state.pending_consolidations[next_pending_consolidation:]
```

Compared to`process_pending_balance_deposits`, this function does not do any churn limiting itself, as this already happens in `process_consolidation`. Here we go through the `pending_consolidations` in order and do the following:
1. Skip any pending consolidation whose `source_validator` is slashed.
2. Other than that, process all pending consolidations until `source_validator.withdrawable_epoch > next_epoch`, at which point we stop. Processing a pending consolidation involves switching the target to withdrawal credentials (currently the only way to do so for an existing validator), and then transferring the *active* balance of the source validator to the target.

From a churn limiting perspective, we could instead use the exit epoch as the trigger for the balance transfer from the source validator to the target validator, since `process_consolidation` assigns exit epochs to respect the consolidation churn constraints. We instead use the withdrawable epoch as the trigger, to maintain the property that validators cannot escape slashing if it is processed on chain within `MIN_VALIDATOR_WITHDRAWABILITY_DELAY` epochs after exiting.

 To be precise, we process a `pending_consolidation` in the epoch transition to the withdrawable epoch, i.e., when `source_validator.withdrawable_epoch == next_epoch`. This way, pending consolidation processing happens strictly before any withdrawal, in particular preventing the withdrawal sweep from accidentally withdrawing balance that should instead be consolidated.

Using the withdrawable epoch as the consolidation trigger also explains why we skip consolidations where the source validator is slashed: slashing can increase the withdrawable epoch of a validator, breaking the property that `pending_consolidations` is ordered by increasing withdrawable epoch. Skipping those consolidations leaves us only with ones which respect this order, which simplifies the processing, because we can stop as soon as we reach a withdrawable epoch beyond the current epoch.

#### Updated `process_effective_balance_updates`

`process_effective_balance_updates` is updated with a new limit for the maximum effective balance.

```python
def process_effective_balance_updates(state: BeaconState) -> None:
    # Update effective balances with hysteresis
    for index, validator in enumerate(state.validators):
        balance = state.balances[index]
        HYSTERESIS_INCREMENT = uint64(EFFECTIVE_BALANCE_INCREMENT // HYSTERESIS_QUOTIENT)
        DOWNWARD_THRESHOLD = HYSTERESIS_INCREMENT * HYSTERESIS_DOWNWARD_MULTIPLIER
        UPWARD_THRESHOLD = HYSTERESIS_INCREMENT * HYSTERESIS_UPWARD_MULTIPLIER
        EFFECTIVE_BALANCE_LIMIT = (
            MAX_EFFECTIVE_BALANCE_ELECTRA if has_compounding_withdrawal_credential(validator)
            else MIN_ACTIVATION_BALANCE
        )

        if (
            balance + DOWNWARD_THRESHOLD < validator.effective_balance
            or validator.effective_balance + UPWARD_THRESHOLD < balance
        ):
            validator.effective_balance = min(balance - balance % EFFECTIVE_BALANCE_INCREMENT, EFFECTIVE_BALANCE_LIMIT)
```

### Block processing

```python
def process_block(state: BeaconState, block: BeaconBlock) -> None:
    process_block_header(state, block)
    process_withdrawals(state, block.body.execution_payload)  # [Modified in Electra:EIP7251]
    process_execution_payload(state, block.body, EXECUTION_ENGINE)  # [Modified in Electra:EIP6110]
    process_randao(state, block.body)
    process_eth1_data(state, block.body)
    process_operations(state, block.body)  # [Modified in Electra:EIP6110:EIP7002:EIP7549:EIP7251]
    process_sync_aggregate(state, block.body.sync_aggregate)
```

#### Withdrawals

##### Updated `get_expected_withdrawals`

```python
def get_expected_withdrawals(state: BeaconState) -> Tuple[Sequence[Withdrawal], uint64]:
    epoch = get_current_epoch(state)
    withdrawal_index = state.next_withdrawal_index
    validator_index = state.next_withdrawal_validator_index
    withdrawals: List[Withdrawal] = []

    # [New in Electra:EIP7251] Consume pending partial withdrawals
    for withdrawal in state.pending_partial_withdrawals:
        if withdrawal.withdrawable_epoch > epoch or len(withdrawals) == MAX_PENDING_PARTIALS_PER_WITHDRAWALS_SWEEP:
            break

        validator = state.validators[withdrawal.index]
        has_sufficient_effective_balance = validator.effective_balance >= MIN_ACTIVATION_BALANCE
        has_excess_balance = state.balances[withdrawal.index] > MIN_ACTIVATION_BALANCE
        if validator.exit_epoch == FAR_FUTURE_EPOCH and has_sufficient_effective_balance and has_excess_balance:
            withdrawable_balance = min(state.balances[withdrawal.index] - MIN_ACTIVATION_BALANCE, withdrawal.amount)
            withdrawals.append(Withdrawal(
                index=withdrawal_index,
                validator_index=withdrawal.index,
                address=ExecutionAddress(validator.withdrawal_credentials[12:]),
                amount=withdrawable_balance,
            ))
            withdrawal_index += WithdrawalIndex(1)

    partial_withdrawals_count = len(withdrawals)

    # Sweep for remaining.
    bound = min(len(state.validators), MAX_VALIDATORS_PER_WITHDRAWALS_SWEEP)
    for _ in range(bound):
        validator = state.validators[validator_index]
        balance = state.balances[validator_index]
        if is_fully_withdrawable_validator(validator, balance, epoch):
            withdrawals.append(Withdrawal(
                index=withdrawal_index,
                validator_index=validator_index,
                address=ExecutionAddress(validator.withdrawal_credentials[12:]),
                amount=balance,
            ))
            withdrawal_index += WithdrawalIndex(1)
        elif is_partially_withdrawable_validator(validator, balance):
            withdrawals.append(Withdrawal(
                index=withdrawal_index,
                validator_index=validator_index,
                address=ExecutionAddress(validator.withdrawal_credentials[12:]),
                amount=balance - get_validator_max_effective_balance(validator),  # [Modified in Electra:EIP7251]
            ))
            withdrawal_index += WithdrawalIndex(1)
        if len(withdrawals) == MAX_WITHDRAWALS_PER_PAYLOAD:
            break
        validator_index = ValidatorIndex((validator_index + 1) % len(state.validators))
    return withdrawals, partial_withdrawals_count
```

##### Updated `process_withdrawals`

```python
def process_withdrawals(state: BeaconState, payload: ExecutionPayload) -> None:
    expected_withdrawals, partial_withdrawals_count = get_expected_withdrawals(state)  # [Modified in Electra:EIP7251]

    assert len(payload.withdrawals) == len(expected_withdrawals)

    for expected_withdrawal, withdrawal in zip(expected_withdrawals, payload.withdrawals):
        assert withdrawal == expected_withdrawal
        decrease_balance(state, withdrawal.validator_index, withdrawal.amount)

    # Update pending partial withdrawals [New in Electra:EIP7251]
    state.pending_partial_withdrawals = state.pending_partial_withdrawals[partial_withdrawals_count:]

    # Update the next withdrawal index if this block contained withdrawals
    if len(expected_withdrawals) != 0:
        latest_withdrawal = expected_withdrawals[-1]
        state.next_withdrawal_index = WithdrawalIndex(latest_withdrawal.index + 1)

    # Update the next validator index to start the next withdrawal sweep
    if len(expected_withdrawals) == MAX_WITHDRAWALS_PER_PAYLOAD:
        # Next sweep starts after the latest withdrawal's validator index
        next_validator_index = ValidatorIndex((expected_withdrawals[-1].validator_index + 1) % len(state.validators))
        state.next_withdrawal_validator_index = next_validator_index
    else:
        # Advance sweep by the max length of the sweep if there was not a full set of withdrawals
        next_index = state.next_withdrawal_validator_index + MAX_VALIDATORS_PER_WITHDRAWALS_SWEEP
        next_validator_index = ValidatorIndex(next_index % len(state.validators))
        state.next_withdrawal_validator_index = next_validator_index
```

#### Execution payload

##### Modified `process_execution_payload`

*Note*: The function `process_execution_payload` is modified to use the new `ExecutionPayloadHeader` type.

```python
def process_execution_payload(state: BeaconState, body: BeaconBlockBody, execution_engine: ExecutionEngine) -> None:
    payload = body.execution_payload

    # Verify consistency of the parent hash with respect to the previous execution payload header
    assert payload.parent_hash == state.latest_execution_payload_header.block_hash
    # Verify prev_randao
    assert payload.prev_randao == get_randao_mix(state, get_current_epoch(state))
    # Verify timestamp
    assert payload.timestamp == compute_timestamp_at_slot(state, state.slot)
    # Verify commitments are under limit
    assert len(body.blob_kzg_commitments) <= MAX_BLOBS_PER_BLOCK
    # Verify the execution payload is valid
    versioned_hashes = [kzg_commitment_to_versioned_hash(commitment) for commitment in body.blob_kzg_commitments]
    assert execution_engine.verify_and_notify_new_payload(
        NewPayloadRequest(
            execution_payload=payload,
            versioned_hashes=versioned_hashes,
            parent_beacon_block_root=state.latest_block_header.parent_root,
        )
    )
    # Cache execution payload header
    state.latest_execution_payload_header = ExecutionPayloadHeader(
        parent_hash=payload.parent_hash,
        fee_recipient=payload.fee_recipient,
        state_root=payload.state_root,
        receipts_root=payload.receipts_root,
        logs_bloom=payload.logs_bloom,
        prev_randao=payload.prev_randao,
        block_number=payload.block_number,
        gas_limit=payload.gas_limit,
        gas_used=payload.gas_used,
        timestamp=payload.timestamp,
        extra_data=payload.extra_data,
        base_fee_per_gas=payload.base_fee_per_gas,
        block_hash=payload.block_hash,
        transactions_root=hash_tree_root(payload.transactions),
        withdrawals_root=hash_tree_root(payload.withdrawals),
        blob_gas_used=payload.blob_gas_used,
        excess_blob_gas=payload.excess_blob_gas,
        deposit_receipts_root=hash_tree_root(payload.deposit_receipts),  # [New in Electra:EIP6110]
        withdrawal_requests_root=hash_tree_root(payload.withdrawal_requests),  # [New in Electra:EIP7002:EIP7251]
    )
```

#### Operations

##### Modified `process_operations`

*Note*: The function `process_operations` is modified to support all of the new functionality in Electra.

```python
def process_operations(state: BeaconState, body: BeaconBlockBody) -> None:
    # [Modified in Electra:EIP6110]
    # Disable former deposit mechanism once all prior deposits are processed
    eth1_deposit_index_limit = min(state.eth1_data.deposit_count, state.deposit_receipts_start_index)
    if state.eth1_deposit_index < eth1_deposit_index_limit:
        assert len(body.deposits) == min(MAX_DEPOSITS, eth1_deposit_index_limit - state.eth1_deposit_index)
    else:
        assert len(body.deposits) == 0

    def for_ops(operations: Sequence[Any], fn: Callable[[BeaconState, Any], None]) -> None:
        for operation in operations:
            fn(state, operation)

    for_ops(body.proposer_slashings, process_proposer_slashing)
    for_ops(body.attester_slashings, process_attester_slashing)
    for_ops(body.attestations, process_attestation)  # [Modified in Electra:EIP7549]
    for_ops(body.deposits, process_deposit)  # [Modified in Electra:EIP7251]
    for_ops(body.voluntary_exits, process_voluntary_exit)  # [Modified in Electra:EIP7251]
    for_ops(body.bls_to_execution_changes, process_bls_to_execution_change)
    # [New in Electra:EIP7002:EIP7251]
    for_ops(body.execution_payload.withdrawal_requests, process_execution_layer_withdrawal_request)
    for_ops(body.execution_payload.deposit_receipts, process_deposit_receipt)  # [New in Electra:EIP6110]
    for_ops(body.consolidations, process_consolidation)  # [New in Electra:EIP7251]
```

##### Attestations

###### Modified `process_attestation`

```python
def process_attestation(state: BeaconState, attestation: Attestation) -> None:
    data = attestation.data
    assert data.target.epoch in (get_previous_epoch(state), get_current_epoch(state))
    assert data.target.epoch == compute_epoch_at_slot(data.slot)
    assert data.slot + MIN_ATTESTATION_INCLUSION_DELAY <= state.slot

    # [Modified in Electra:EIP7549]
    assert data.index == 0
    committee_indices = get_committee_indices(attestation.committee_bits)
    participants_count = 0
    for index in committee_indices:
        assert index < get_committee_count_per_slot(state, data.target.epoch)
        committee = get_beacon_committee(state, data.slot, index)
        participants_count += len(committee)

    assert len(attestation.aggregation_bits) == participants_count

    # Participation flag indices
    participation_flag_indices = get_attestation_participation_flag_indices(state, data, state.slot - data.slot)

    # Verify signature
    assert is_valid_indexed_attestation(state, get_indexed_attestation(state, attestation))

    # Update epoch participation flags
    if data.target.epoch == get_current_epoch(state):
        epoch_participation = state.current_epoch_participation
    else:
        epoch_participation = state.previous_epoch_participation

    proposer_reward_numerator = 0
    for index in get_attesting_indices(state, attestation):
        for flag_index, weight in enumerate(PARTICIPATION_FLAG_WEIGHTS):
            if flag_index in participation_flag_indices and not has_flag(epoch_participation[index], flag_index):
                epoch_participation[index] = add_flag(epoch_participation[index], flag_index)
                proposer_reward_numerator += get_base_reward(state, index) * weight

    # Reward proposer
    proposer_reward_denominator = (WEIGHT_DENOMINATOR - PROPOSER_WEIGHT) * WEIGHT_DENOMINATOR // PROPOSER_WEIGHT
    proposer_reward = Gwei(proposer_reward_numerator // proposer_reward_denominator)
    increase_balance(state, get_beacon_proposer_index(state), proposer_reward)
```

##### Deposits

###### Updated  `apply_deposit`

*NOTE*: `process_deposit` is updated with a new definition of `apply_deposit`.

```python
def apply_deposit(state: BeaconState,
                  pubkey: BLSPubkey,
                  withdrawal_credentials: Bytes32,
                  amount: uint64,
                  signature: BLSSignature) -> None:
    validator_pubkeys = [v.pubkey for v in state.validators]
    if pubkey not in validator_pubkeys:
        # Verify the deposit signature (proof of possession) which is not checked by the deposit contract
        if is_valid_deposit_signature(pubkey, withdrawal_credentials, amount, signature):
            add_validator_to_registry(state, pubkey, withdrawal_credentials, amount)
    else:
        # Increase balance by deposit amount
        index = ValidatorIndex(validator_pubkeys.index(pubkey))
        state.pending_balance_deposits.append(
            PendingBalanceDeposit(index=index, amount=amount)
        )  # [Modified in Electra:EIP-7251]
        # Check if valid deposit switch to compounding credentials
        if (
            is_compounding_withdrawal_credential(withdrawal_credentials)
            and has_eth1_withdrawal_credential(state.validators[index])
            and is_valid_deposit_signature(pubkey, withdrawal_credentials, amount, signature)
        ):
            switch_to_compounding_validator(state, index)

```

For both deposits to new validators and top-ups, the balance increase is not immediate anymore. Instead, we append a  `PendingBalanceDeposit` to the queue (for deposits to new validators, this happens in `add_validator_to_registry`), which increases the validator's balance after going through churn, as discussed [here](#deposits-and-activations). 

Top-ups to validators with eth1 withdrawal credentials now can also trigger a switch to compounding withdrawal credentials, serving as a simple mechanism for credential change which does not require an ad-hoc operation. For this, we only require authorization by the validator's signing key, and not by the its withdrawal credentials. The reasoning is that switching from eth1 to compounding credentials does not change the execution address which owns the funds (the withdrawal address). Instead, it only changes the prefix of the credential, wh

For this, we first of all require that  `deposit.withdrawal_credentials` are compounding credentials, so that validators are able to indicate whether they want to trigger the switch to compounding credentials (use the `0x02` prefix) or not (use the `0x01` prefix). We also check the validity of the deposit signature (the proof of possession, which is otherwise not checked for top-ups), to ensure that a switch to compounding credentials is authorized by the 
###### New `is_valid_deposit_signature`

```python
def is_valid_deposit_signature(pubkey: BLSPubkey,
                               withdrawal_credentials: Bytes32,
                               amount: uint64,
                               signature: BLSSignature) -> bool:
    deposit_message = DepositMessage(
        pubkey=pubkey,
        withdrawal_credentials=withdrawal_credentials,
        amount=amount,
    )
    domain = compute_domain(DOMAIN_DEPOSIT)  # Fork-agnostic domain since deposits are valid across forks
    signing_root = compute_signing_root(deposit_message, domain)
    return bls.Verify(pubkey, signing_root, signature)
```

###### Modified `add_validator_to_registry`

```python
def add_validator_to_registry(state: BeaconState,
                              pubkey: BLSPubkey,
                              withdrawal_credentials: Bytes32,
                              amount: uint64) -> None:
    index = get_index_for_new_validator(state)
    validator = get_validator_from_deposit(pubkey, withdrawal_credentials)
    set_or_append_list(state.validators, index, validator)
    set_or_append_list(state.balances, index, 0)  # [Modified in Electra:EIP7251]
    set_or_append_list(state.previous_epoch_participation, index, ParticipationFlags(0b0000_0000))
    set_or_append_list(state.current_epoch_participation, index, ParticipationFlags(0b0000_0000))
    set_or_append_list(state.inactivity_scores, index, uint64(0))
    state.pending_balance_deposits.append(PendingBalanceDeposit(index=index, amount=amount))  # [New in Electra:EIP7251]
```

Previously, a validator's balance was immediately set to the deposit amount. Now, as part of the [redesign of the deposit and activation flow](#deposits-and-activations), we instead add a `PendingBalanceDeposit` to the queue, and only after it has gone through churn is the deposit amount added to the validator's balance, in `process_pending_balance_deposits`. Therefore, the balance is initially set to `0`.
###### Updated `get_validator_from_deposit`

```python
def get_validator_from_deposit(pubkey: BLSPubkey, withdrawal_credentials: Bytes32) -> Validator:
    return Validator(
        pubkey=pubkey,
        withdrawal_credentials=withdrawal_credentials,
        activation_eligibility_epoch=FAR_FUTURE_EPOCH,
        activation_epoch=FAR_FUTURE_EPOCH,
        exit_epoch=FAR_FUTURE_EPOCH,
        withdrawable_epoch=FAR_FUTURE_EPOCH,
        effective_balance=0,  # [Modified in Electra:EIP7251]
    )
```

Just as in the previous function we set a new validator's initial balance to `0`, we do the same with the effective balance.
###### Updated `process_voluntary_exit`

```python
def process_voluntary_exit(state: BeaconState, signed_voluntary_exit: SignedVoluntaryExit) -> None:
    voluntary_exit = signed_voluntary_exit.message
    validator = state.validators[voluntary_exit.validator_index]
    # Verify the validator is active
    assert is_active_validator(validator, get_current_epoch(state))
    # Verify exit has not been initiated
    assert validator.exit_epoch == FAR_FUTURE_EPOCH
    # Exits must specify an epoch when they become valid; they are not valid before then
    assert get_current_epoch(state) >= voluntary_exit.epoch
    # Verify the validator has been active long enough
    assert get_current_epoch(state) >= validator.activation_epoch + SHARD_COMMITTEE_PERIOD
    # Only exit validator if it has no pending withdrawals in the queue
    assert get_pending_balance_to_withdraw(state, voluntary_exit.validator_index) == 0  # [New in Electra:EIP7251]
    # Verify signature
    domain = compute_domain(DOMAIN_VOLUNTARY_EXIT, CAPELLA_FORK_VERSION, state.genesis_validators_root)
    signing_root = compute_signing_root(voluntary_exit, domain)
    assert bls.Verify(validator.pubkey, signing_root, signed_voluntary_exit.signature)
    # Initiate exit
    initiate_validator_exit(state, voluntary_exit.validator_index)
```

The only modification is this assertion `assert get_pending_balance_to_withdraw(state, voluntary_exit.validator_index) == 0 `.  `get_pending_balance_to_withdraw` simply sums the withdrawal amount for every `PendingPartialWithdrawal` that's currently in the queue for the given validator. We require this to be `0` in order to process the exit, to avoid that the same balance can consume churn twice, both by being queued up as a partial withdrawal and as part of an exit.

##### Execution layer withdrawal requests

###### New `process_execution_layer_withdrawal_request`

*Note*: This function is new in Electra following EIP-7002 and EIP-7251.

```python
def process_execution_layer_withdrawal_request(
    state: BeaconState,
    execution_layer_withdrawal_request: ExecutionLayerWithdrawalRequest
) -> None:
    amount = execution_layer_withdrawal_request.amount
    is_full_exit_request = amount == FULL_EXIT_REQUEST_AMOUNT

    # If partial withdrawal queue is full, only full exits are processed
    if len(state.pending_partial_withdrawals) == PENDING_PARTIAL_WITHDRAWALS_LIMIT and not is_full_exit_request:
        return

    validator_pubkeys = [v.pubkey for v in state.validators]
    # Verify pubkey exists
    request_pubkey = execution_layer_withdrawal_request.validator_pubkey
    if request_pubkey not in validator_pubkeys:
        return
    index = ValidatorIndex(validator_pubkeys.index(request_pubkey))
    validator = state.validators[index]

    # Verify withdrawal credentials
    has_correct_credential = has_execution_withdrawal_credential(validator)
    is_correct_source_address = (
        validator.withdrawal_credentials[12:] == execution_layer_withdrawal_request.source_address
    )
    if not (has_correct_credential and is_correct_source_address):
        return
    # Verify the validator is active
    if not is_active_validator(validator, get_current_epoch(state)):
        return
    # Verify exit has not been initiated
    if validator.exit_epoch != FAR_FUTURE_EPOCH:
        return
    # Verify the validator has been active long enough
    if get_current_epoch(state) < validator.activation_epoch + SHARD_COMMITTEE_PERIOD:
        return

    pending_balance_to_withdraw = get_pending_balance_to_withdraw(state, index)

    if is_full_exit_request:
        # Only exit validator if it has no pending withdrawals in the queue
        if pending_balance_to_withdraw == 0:
            initiate_validator_exit(state, index)
        return

    has_sufficient_effective_balance = validator.effective_balance >= MIN_ACTIVATION_BALANCE
    has_excess_balance = state.balances[index] > MIN_ACTIVATION_BALANCE + pending_balance_to_withdraw

    # Only allow partial withdrawals with compounding withdrawal credentials
    if has_compounding_withdrawal_credential(validator) and has_sufficient_effective_balance and has_excess_balance:
        to_withdraw = min(
            state.balances[index] - MIN_ACTIVATION_BALANCE - pending_balance_to_withdraw,
            amount
        )
        exit_queue_epoch = compute_exit_epoch_and_update_churn(state, to_withdraw)
        withdrawable_epoch = Epoch(exit_queue_epoch + MIN_VALIDATOR_WITHDRAWABILITY_DELAY)
        state.pending_partial_withdrawals.append(PendingPartialWithdrawal(
            index=index,
            amount=to_withdraw,
            withdrawable_epoch=withdrawable_epoch,
        ))
```

Electra introduces a new type of EL -> CL request, allowing for EL-triggered exits and partial withdrawals. Note that, because the EL cannot fully validate the requests it receives before forwarding them to the EL (by inclusion in the `ExecutionPayload`), there is nothing in this function which can cause the block to be invalid. We `return` whenever some validation fails, essentially just doing a no-op, instead of failing an assertion. 

To authenticate the operation, we rely on the EL to ensure correctness of the`source_address` in an `execution_layer_withdrawal_request`. With that, we only need to check that it matches the (last 20 bytes of the) withdrawal credential of the validator whose pubkey is included in the request. Other than authentication, a lot of the checks we make are the same as in `process_voluntary_exit`: the validator should be active, have been active long enough, and should not be exiting. 

When the request is a full exit, which is when `amount == 0`, we also check that the validator does not have partial withdrawals already queued up. We do the same in `process_voluntary_exit`, to avoid that the same balance takes up exit churn twice, both as a partial withdrawal and as a full exit. 

Partial withdrawals requests are processed by adding a `PendingPartialWithdrawal` to the `pending_partial_withdrawals` queue. This contains the validator's `index`, the max `amount` to withdraw (at withdrawal time the available balance might be lower) and the `withdrawable_epoch`, determining when the withdrawal will be processed. Computing the latter works exactly as if we were dealing with the exit of a validator with `amount` effective balance: the exit queue epoch is computed by `compute_exit_epoch_and_update_churn` with `exit_balance` set to `amount`, and the `MIN_VALIDATOR_WITHDRAWABILITY_DELAY` is added to that. As for exits, `compute_exit_epoch_and_update_churn` deals with  

We only allow a validator to queue up the withdrawal of their excess balance (anything over `MIN_ACTIVATION_BALANCE`). For the same reason as just above, we also subtract `pending_balance_to_withdraw` from the excess balance, not allowing the withdrawal of balance which is already in the partial withdrawal queue. A validator attempts to withdraw more than they have available, we just 


##### Deposit receipts

###### New `process_deposit_receipt`

*Note*: This function is new in Electra:EIP6110.

```python
def process_deposit_receipt(state: BeaconState, deposit_receipt: DepositReceipt) -> None:
    # Set deposit receipt start index
    if state.deposit_receipts_start_index == UNSET_DEPOSIT_RECEIPTS_START_INDEX:
        state.deposit_receipts_start_index = deposit_receipt.index

    apply_deposit(
        state=state,
        pubkey=deposit_receipt.pubkey,
        withdrawal_credentials=deposit_receipt.withdrawal_credentials,
        amount=deposit_receipt.amount,
        signature=deposit_receipt.signature,
    )
```

##### Consolidations

###### New `process_consolidation`

```python
def process_consolidation(state: BeaconState, signed_consolidation: SignedConsolidation) -> None:
    # If the pending consolidations queue is full, no consolidations are allowed in the block
    assert len(state.pending_consolidations) < PENDING_CONSOLIDATIONS_LIMIT
    # If there is too little available consolidation churn limit, no consolidations are allowed in the block
    assert get_consolidation_churn_limit(state) > MIN_ACTIVATION_BALANCE
    consolidation = signed_consolidation.message
    # Verify that source != target, so a consolidation cannot be used as an exit.
    assert consolidation.source_index != consolidation.target_index

    source_validator = state.validators[consolidation.source_index]
    target_validator = state.validators[consolidation.target_index]
    # Verify the source and the target are active
    current_epoch = get_current_epoch(state)
    assert is_active_validator(source_validator, current_epoch)
    assert is_active_validator(target_validator, current_epoch)
    # Verify exits for source and target have not been initiated
    assert source_validator.exit_epoch == FAR_FUTURE_EPOCH
    assert target_validator.exit_epoch == FAR_FUTURE_EPOCH
    # Consolidations must specify an epoch when they become valid; they are not valid before then
    assert current_epoch >= consolidation.epoch

    # Verify the source and the target have Execution layer withdrawal credentials
    assert has_execution_withdrawal_credential(source_validator)
    assert has_execution_withdrawal_credential(target_validator)
    # Verify the same withdrawal address
    assert source_validator.withdrawal_credentials[12:] == target_validator.withdrawal_credentials[12:]

    # Verify consolidation is signed by the source and the target
    domain = compute_domain(DOMAIN_CONSOLIDATION, genesis_validators_root=state.genesis_validators_root)
    signing_root = compute_signing_root(consolidation, domain)
    pubkeys = [source_validator.pubkey, target_validator.pubkey]
    assert bls.FastAggregateVerify(pubkeys, signing_root, signed_consolidation.signature)

    # Initiate source validator exit and append pending consolidation
    source_validator.exit_epoch = compute_consolidation_epoch_and_update_churn(
        state, source_validator.effective_balance
    )
    source_validator.withdrawable_epoch = Epoch(
        source_validator.exit_epoch + MIN_VALIDATOR_WITHDRAWABILITY_DELAY
    )
    state.pending_consolidations.append(PendingConsolidation(
        source_index=consolidation.source_index,
        target_index=consolidation.target_index
    ))
```

TODO: substitute with EL-triggered consolidations. 
## Testing

*Note*: The function `initialize_beacon_state_from_eth1` is modified for pure Electra testing only.
Modifications include:
1. Use `ELECTRA_FORK_VERSION` as the previous and current fork version.
2. Utilize the Electra `BeaconBlockBody` when constructing the initial `latest_block_header`.
3. *[New in Electra:EIP6110]* Add `deposit_receipts_start_index` variable to the genesis state initialization.
4. *[New in Electra:EIP7251]* Initialize new fields to support increasing the maximum effective balance.

```python
def initialize_beacon_state_from_eth1(eth1_block_hash: Hash32,
                                      eth1_timestamp: uint64,
                                      deposits: Sequence[Deposit],
                                      execution_payload_header: ExecutionPayloadHeader=ExecutionPayloadHeader()
                                      ) -> BeaconState:
    fork = Fork(
        previous_version=ELECTRA_FORK_VERSION,  # [Modified in Electra:EIP6110] for testing only
        current_version=ELECTRA_FORK_VERSION,  # [Modified in Electra:EIP6110]
        epoch=GENESIS_EPOCH,
    )
    state = BeaconState(
        genesis_time=eth1_timestamp + GENESIS_DELAY,
        fork=fork,
        eth1_data=Eth1Data(block_hash=eth1_block_hash, deposit_count=uint64(len(deposits))),
        latest_block_header=BeaconBlockHeader(body_root=hash_tree_root(BeaconBlockBody())),
        randao_mixes=[eth1_block_hash] * EPOCHS_PER_HISTORICAL_VECTOR,  # Seed RANDAO with Eth1 entropy
        deposit_receipts_start_index=UNSET_DEPOSIT_RECEIPTS_START_INDEX,  # [New in Electra:EIP6110]
    )

    # Process deposits
    leaves = list(map(lambda deposit: deposit.data, deposits))
    for index, deposit in enumerate(deposits):
        deposit_data_list = List[DepositData, 2**DEPOSIT_CONTRACT_TREE_DEPTH](*leaves[:index + 1])
        state.eth1_data.deposit_root = hash_tree_root(deposit_data_list)
        process_deposit(state, deposit)

    # Process deposit balance updates
    for deposit in state.pending_balance_deposits:
        increase_balance(state, deposit.index, deposit.amount)
    state.pending_balance_deposits = []

    # Process activations
    for index, validator in enumerate(state.validators):
        balance = state.balances[index]
        validator.effective_balance = min(balance - balance % EFFECTIVE_BALANCE_INCREMENT, MAX_EFFECTIVE_BALANCE)
        if validator.effective_balance == MAX_EFFECTIVE_BALANCE:
            validator.activation_eligibility_epoch = GENESIS_EPOCH
            validator.activation_epoch = GENESIS_EPOCH

    # Set genesis validators root for domain separation and chain versioning
    state.genesis_validators_root = hash_tree_root(state.validators)

    # Fill in sync committees
    # Note: A duplicate committee is assigned for the current and next committee at genesis
    state.current_sync_committee = get_next_sync_committee(state)
    state.next_sync_committee = get_next_sync_committee(state)

    # Initialize the execution payload header
    state.latest_execution_payload_header = execution_payload_header

    return state
```
