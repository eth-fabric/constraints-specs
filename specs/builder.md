# Builder Specification

## Table of Contents
- [Background](#background)
  - [Definitions](#definitions)
  - [Constants](#constants)
    - [Domain types](#domain-types)
    - [Time parameters](#time-parameters)
  - [Containers](#containers)
    - [Constraint](#constraint)
    - [ConstraintsMessage](#constraintsmessage)
    - [SignedConstraints](#signedconstraints)
    - [VersionSubmitBlockRequestWithProofs](#versionedsubmitblockrequestwithproofs)
    - [ConstraintProofs](#constraintproofs)
    - [BuilderBid](#builderbid)
    - [SignedBuilderBidWithProofs](#signedbuilderbidwithproofs)
- [Signing](#signing)
- [Validator Registration Processing](#validator-registration-processing)
- [Constraint Processing](#constraint-processing)
    - [verify_builder_slot_signature](#verify_builder_slot_signature)
    - [verify_constraint_signature](#verify_constraint_signature)
    - [process_constraints](#process_constraints)
- [Building](#building)
    - [Bidding](#bidding)
        - [Constructing the BuilderBidWithProofs](#constructing-the-builderbidwithproofs)
    - [Submitting a block](#submitting-a-block)

## Background
### Definitions

| Name | Definition |
| --- | --- |
| **Proposer**   | An Ethereum validator with the rights to propose an L1 block. |
| **Builder**    | An entity specialized in building L1 blocks. |
| **Relay**      | A trusted entity that aggregates blocks from Builders for Proposers. |
| **Commitment** | A binding message committing the proposer to perform an action as part of their block proposal duties. |
| **Constraint** | Instructions for block builders to build blocks that adhere to proposer commitments. |
| **Gateway**    | Third party with Constraint and Commitment submission authority granted by the Proposer. |
| **Committer**  | An entity that makes commitment. |
| **Preconfer**  | A Committer dealing preconfirmations. |
| **Slasher Contract**  | A protocol-specific smart contract that contains the logic to slash a proposer for breaking their commitment. |

A note on definitions:

- Teams commonly refer to **Proposers** as being **Preconfers** and **Gateways** as being **Delegated Preconfers**. Since the spec generalizes to cover *all proposer commitments*, we’ll stick to the term **Gateway** so as to not limit imaginations to just preconfs.

Some nuances:

- Proposers can self-delegate, in which case they act as their own **Gateway**.
- Proposers can be slashed for equivocation if they sign multiple delegations during the same slot, effectively limiting them to a single Gateway at a time.

### Constants

#### Domain types

| Name | Value |
| - | - |
| `DOMAIN_APPLICATION_BUILDER` | `DomainType('0x00000001')` |
| `DOMAIN_APPLICATION_GATEWAY` | TBD |
| `DELEGATION_DOMAIN_SEPARATOR` | `DomainType('0x0044656c')` |

#### Constraints parameters

| Name | Value |
| - | - |
| `CONSTRAINTS_SUBMISSION_DEADLINE` | 8 seconds |
| `MAX_CONSTRAINTS_PER_SLOT` | `uint64(256)` |

### Containers

Consider the following definitions supplementary to the definitions in `consensus-specs`. For information on how containers are signed, see [Signing](#signing).

#### `Constraint`
```python
class Constraint(Container):
    constraintType: uint64
    payload: Bytes
```

#### `ConstraintsMessage` 
```python
class ConstraintsMessage(Container):
    proposer: BLSPubkey
    delegate: BLSPubkey
    slot: uint64
    constraints: List[Constraint, MAX_CONSTRAINTS_PER_SLOT]
    receivers: List[BLSPubkey]
```

#### `SignedConstraints`
```python
class SignedConstraints(Container):
    message: ConstraintsMessage
    signature: BLSSignature
```

#### `VersionedSubmitBlockRequestWithProofs`
```python
class VersionedSubmitBlockRequestWithProofs:
    ... # All regular fields from VersionedSubmitBlockRequest, additionally
    proofs: ConstraintProofs
```

#### `ConstraintProofs`
```python
class ConstraintProofs(Container):
    constraintTypes: List[uint64, MAX_CONSTRAINTS_PER_SLOT]
    payloads: List[Bytes, MAX_CONSTRAINTS_PER_SLOT]
```

#### `BuilderBid`
```python
class BuilderBid(Container):
    header: ExecutionPayloadHeader
    blob_kzg_commitments: List[KZGCommitment, MAX_BLOB_COMMITMENTS_PER_BLOCK]
    execution_requests: ExecutionRequests
    value: uint256
    pubkey: BLSPubkey
```

#### `SignedBuilderBidWithProofs`
```python
class SignedBuilderBidWithProofs(Container):
    message: BuilderBid
    signature: BLSSignature
    proofs: ConstraintProofs
```

## Signing

All signature operations should follow the [standard BLS operations](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#bls-signatures) interface defined in `consensus-specs`.

To assist in signing, we use a function from the [consensus specs](https://github.com/ethereum/consensus-specs):
[`compute_domain`](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#compute_domain)

There are three types of data to sign over in the Builder API:
* In-protocol messages, e.g. `BlindedBeaconBlock`, which should compute the signing root using `compute_signing_root` and use the domain specified for beacon block proposals.
* Builder API messages, e.g. validator registration and signed builder bid with proofs, which should compute the signing root using `compute_signing_root` with domain given by `compute_domain(DOMAIN_APPLICATION_BUILDER)`.
* Gateway API messages, e.g. constraints, which should compute the signing root using `compute_signing_root` with domain given by `compute_domain(DOMAIN_APPLICATION_GATEWAY)`.

## Validator registration processing
The regstration spec is unchanged from the [Builder Spec](https://github.com/ethereum/builder-specs/blob/main/specs/bellatrix/builder.md#validator-registration-processing), specifically validators will still sign `ValidatorRegistration` messages to register to begin working with Relays and Builders.

## Constraint Processing
`SignedConstraints` are expected to be submitted within the `CONSTRAINTS_SUBMISSION_DEADLINE`, after which a Builder can retrieve constraints from Relays by calling the `getConstraints` [endpoint](https://eth-fabric.github.io/constraints-specs/#/Constraints%20API/getConstraints).

For authorization, the Builder will sign the upcoming `slot` number (head + 1) with their BLS private key as follows: 
```python
domain = compute_domain(DOMAIN_APPLICATION_BUILDER)
signing_root = compute_signing_root(slot, domain)
signature = bls.sign(builder_private_key, signing_root)
```

The Builder will include the `signature` in the request header of the `getConstraints` which the relay will check against the BLS public keys in the `receivers` field of the `ConstraintsMessage`.

The Relay is required to abort the request if the signature is invalid or the Builder's BLS public key is not in the `ConstraintsMessage.receivers` list.

### `verify_builder_slot_signature`
```python
def verify_builder_slot_signature(slot: uint64, builder_pubkey: BLSPubkey, signature: BLSSignature) -> bool:
    domain = compute_domain(DOMAIN_APPLICATION_BUILDER)
    signing_root = compute_signing_root(slot, domain)
    return bls.Verify(builder_pubkey, signing_root, signature)
```

### `verify_constraint_signature`
```python
def verify_constraint_signature(signed_constraints: SignedConstraints) -> bool:
    domain = compute_domain(DOMAIN_APPLICATION_GATEWAY)
    signing_root = compute_signing_root(signed_constraints.message, domain)
    return bls.Verify(signed_constraints.message.delegate, signing_root, signed_constraints.signature)
```

### `process_constraints`
A `SignedConstraints` is considered valid if the following function completes without raising any assertions:

```python
def process_constraints(state: BeaconState,
                       signed_constraints: SignedConstraints,
                       delegations: Dict[BLSPubkey, Delegation]) -> bool:
    constraints = signed_constraints.message
    
    # Verify delegate has authority from proposer
    assert constraints.proposer in delegations
    delegation = delegations[constraints.proposer]
    assert delegation.delegate == constraints.delegate
    
    # Verify constraint count
    assert len(constraints.constraints) <= MAX_CONSTRAINTS_PER_SLOT
    
    # Verify constraint signature
    assert verify_constraint_signature(signed_constraints)
    
    # Verify slot matches delegation
    assert constraints.slot == delegation.slot
    
    return True
```

## Building

The builder builds execution payloads for registered validators and submits them to an auction that happens each slot. The key difference from the original builder spec is that builders must now also generate proofs that their blocks satisfy any constraints issued for that slot.

### Bidding

To assist in bidding, we use the following functions from the [consensus specs](https://github.com/ethereum/consensus-specs):
* [`get_beacon_proposer_index`](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#get_beacon_proposer_index)
* [`hash_tree_root`](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#hash_tree_root)

#### Constructing the `BuilderBidWithProofs`

```python
def get_bid_signature(bid: BuilderBid, privkey: int) -> BLSSignature:
    domain = compute_domain(DOMAIN_APPLICATION_BUILDER)
    signing_root = compute_signing_root(bid, domain)
    return bls.Sign(privkey, signing_root)
```

```python
# Electra
def get_bid_with_proofs(
    payload: ExecutionPayload,
    value: uint256,
    pubkey: BLSPubkey,
    constraints: SignedConstraints
) -> SignedBuilderBidWithProofs:
    header = ExecutionPayloadHeader(
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
        excess_blob_gas=payload.excess_blob_gas,
        block_hash=payload.block_hash,
        transactions_root=hash_tree_root(payload.transactions),
        withdrawals_root=hash_tree_root(payload.withdrawals),
    )
    
    # Generate proofs for each constraint
    proofs = generate_constraint_proofs(execution_payload, constraints)

    builder_bid = BuilderBid(header=header, value=value, pubkey=pubkey)
    
    return SignedBuilderBidWithProofs(
        message=builder_bid,
        signature=get_bid_signature(builder_bid, privkey),
        proofs=proofs
    )
```

The exact method for generating proofs depends on the `constraintType`. Each `constraintType` must define:

1. How to generate proofs given an `ExecutionPayload`
2. How to verify proofs against an `ExecutionPayloadHeader`
3. The format of the proof payload

### Submitting a block
The Builder can submit their block and proofs to the relay by posting to the `submitBlocksWithProofs` [endpoint](https://eth-fabric.github.io/constraints-specs/#/Constraints%20API/submitBlocksWithProofs).