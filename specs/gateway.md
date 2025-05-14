# Gateway Specification

## Table of Contents
todo - add when doc is finalized

# Introduction

This document explains the way in which a Gateway is expected to use the [Constraints Spec](https://eth-fabric.github.io/constraints-specs/) in conjunction with the existing [Builder Spec](https://ethereum.github.io/builder-specs/#/Builder) as well as the [Commitments Spec](https://github.com/eth-fabric/commitments-specs) and [Universal Registry Contract](https://github.com/eth-fabric/urc) (URC) to facilitate proposer commitments. The language and format of this document is meant to mirror the original [Builder Specs](https://github.com/ethereum/builder-specs/blob/main/specs/bellatrix/builder.md).

At a high-level, a Gateway will issue `Commitment` messages that adhere to the [Commitments Spec](https://github.com/eth-fabric/commitments-specs) to commit to actions on behalf of a proposer. They will also create `Constraint` messages and disseminate them via the [Constraints API](https://eth-fabric.github.io/constraints-specs/) to instruct Builders how to construct a valid L1 block that satisfies every `Commitment`. 

# Background
## Definitions

| Name | Definition |
| --- | --- |
| **Proposer**   | An Ethereum validator with the rights to propose an L1 block. |
| **Commitment** | A binding message committing the proposer to perform an action as part of their block proposal duties. |
| **Constraint** | Instructions for block builders to build blocks that adhere to proposer commitments. |
| **Gateway**    | Third party with Constraint and Commitment submission authority granted by the Proposer. |
| **Committer**  | An entity that makes commitment. |
| **Preconfer**  | A Committer dealing preconfirmations. |
| **Slasher Contract**  | A protocol-specific smart contract that contains the logic to slash a proposer for breaking their commitment. |

A note on definitions:

- Teams commonly refer to **Proposers** as being **Preconfers** and **Gateways** as being **Delegated Preconfers.** 
- Since the spec generalizes to cover *all proposer commitments*, we’ll use the terms **Committer** and **Delegated Committer** respectively so as to not limit imaginations.

Some nuances:

- Proposers can self-delegate, in which case they are considered a **Committer**, otherwise they can delegate to a **Gateway**.
- The URC makes it possible for a **Proposer** and **Gateway** to simultaneously be **Committers**.
- Proposers can be slashed for equivocation if they sign multiple delegations during the same slot, effectively limiting them to a single Gateway at a time.

## Constants
### Domain types

| Name | Value |
| - | - |
| `DOMAIN_APPLICATION_BUILDER` | `DomainType('0x00000001')` |
| `DOMAIN_APPLICATION_GATEWAY` | TBD |
| `DELEGATION_DOMAIN_SEPARATOR` | `DomainType('0x0044656c')` |

### URC parameters

| Name | Value |
| - | - |
| `MIN_COLLATERAL` | 0.1 ether |
| `FRAUD_PROOF_WINDOW` | 7200 blocks |
| `SLASH_WINDOW` | 7200 blocks |


## Containers
### Delegation
```python
class Delegation(Container):
    proposer: BLSPubkey
    delegate: BLSPubkey
    committer: Address
    slot: Slot
    metadata: Bytes
```
### SignedDelegation
```python
class SignedDelegation(Container):
    message: Delegation
    signature: BLSSignature
```

### Constraint
```python
class Constraint(Container):
    constraintType: uint64
    payload: Bytes
```

### ConstraintsMessage
```python
class ConstraintsMessage(Container):
    proposer: BLSPubkey
    delegate: BLSPubkey
    slot: uint64
    constraints: List[Constraint]
    receivers: List[BLSPubkey]
```

### SignedConstraints
```python
class SignedConstraints(Container):
    message: ConstraintsMessage
    signature: BLSSignature
```

### Commitment
```python
class Commitment(Container):
    commitmentType: uint64
    payload: Bytes
    slasher: Address
```

### SignedCommitment
```python
class SignedCommitment(Container):
    commitment: Commitment
    signature: Bytes
```

# Receiving delegations
Proposers will sign and disseminate `Delegation` messages as described in the [Proposer Spec](./proposer.md#delegating-to-gateways).

### **Receiving delegations**
Gateways can check all delegations for a given slot by querying the `getDelegations` endpoint in the [Constraints API](./constraints-api.md#endpoint-constraintsv0relaydelegationsslot).

### **Delegation processing**
To assist in registration processing, we use the following functions from the [consensus specs](https://github.com/ethereum/consensus-specs):

- [get_current_epoch](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#get_current_epoch)
- [is_active_validator](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#is_active_validator)
- [is_slashable_validator](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#is_slashable_attestation_data)

#### is_eligible_for_delegation
```python
def is_eligible_for_delegation(state: BeaconState, validator: Validator) -> bool:
    """
    Check if ``validator`` is active or pending.
    """
    epoch = get_current_epoch(state)
    return is_active_validator(validator, epoch) and not is_slashable_validator(validator, epoch)
```


#### verify_delegation_signature
```python
def verify_delegation_signature(signed_delegation: SignedDelegation) -> bool:
    pubkey = signed_delegation.message.proposer
    message = abi.encode(signed_delegation.message)
    signing_root = keccak256(abi.encode_packed(DELEGATION_DOMAIN_SEPARATOR, message))
    return bls.Verify(pubkey, signing_root, signed_delegation.signature)
```

#### process_delegation
A `delegation` is considered valid if the following function completes without raising any assertions:

```python
def process_delegation(state: BeaconState,
                         signed_delegation: SignedDelegation,
                         delegations: Dict[BLSPubkey, Delegation],
                         current_timestamp: uint64):
    signature = signed_delegation.signature
    delegation = signed_delegation.message

    # Verify BLS public key corresponds to a registered validator
    validator_pubkeys = [v.pubkey for v in state.validators]
    assert delegation.proposer in validator_pubkeys

    index = ValidatorIndex(validator_pubkeys.index(pubkey))
    validator = state.validators[index]

    # Verify validator delegation elibility
    assert is_eligible_for_delegation(state, validator)

    # Verify delegation signature
    assert verify_delegation_signature(signed_delegation)
```

## Issuing commitments
A Gateway controlling the `Delegation.committer` address can issue commitments on behalf of the `Delegation.proposer`. 

### The `commitmentType`
The `commitmentType` is a uint64 that identifies the type of commitment being made and is inspired by [EIP-2718](https://eips.ethereum.org/EIPS/eip-2718). The commitments spec does not define any commitment types, rather they are expected to be defined by protocols. Some guidelines to consider:

- the `commitmentType` defines how the `Commitment.payload` is interpreted.
- the `slasher` address points to a `Slasher` contract capable of interpreting the `payload`.
- the `commitmentType` defines how a corresponding `Constraint` is constructed.


### Receiving commitment requests
The Gateway will receive commitment requests when users post to the postCommitment` endpoint as defined in the [Commitments API](https://github.com/eth-fabric/commitments-specs).

The validity of a commitment request is dependent on the `commitmentType` so is left out of scope for this document.

### Signing commitments
The Gateway will sign commitments using the private key corresponding to the `Delegation.committer` execution layer address.

```python
message = keccak256(abi.encode(commitment))
signature = ECDSA.sign(message, committer_private_key)
```

The Gateway will include the `signature` in the `SignedCommitment` container when responding to the `postCommitment` request.


## Issuing constraints
The Gateway is responsible for issuing constraints to builders.

### The `constraintType`
The `constraintType` is a uint64 that identifies the type of constraint being made and is inspired by [EIP-2718](https://eips.ethereum.org/EIPS/eip-2718). The constraints spec does not define any constraint types, rather they are expected to be defined by protocols. Some guidelines to consider:

- the `constraintType` defines how the `Constraint.payload` is interpreted.
- the ordering of `constraints[]` in the `ConstraintsMessage`
- how to generate proofs of constraint validity
- how to verify proofs of constraint validity

### ConstraintsMessage
The Gateway will package one of more `Constraint`s into a `ConstraintsMessage`. Builders are expected to process the `ConstraintsMessage.constraints[]` in the order received. The Gateway will also specify the `ConstraintsMessage.receivers[]` which is a list of public keys that are authorized to access these constraints, enforced by the Relay. If this list is empty, the constraints are publicly accessible to anyone.

### Signing constraints
The Gateway will sign the `ConstraintsMessage` using the private key corresponding to the `Delegation.delegate` BLS public key.

```python
    domain = compute_domain(DOMAIN_APPLICATION_GATEWAY, fork_version=None, genesis_validators_root=None)
    signing_root = compute_signing_root(constraints_message, domain)
    signature = bls.sign(delegate_bls_private_key, signing_root)
```

The `signature` is included in the `SignedConstraints` container when responding to the `postConstraints` request.


### Disseminating constraints
The Gateway will disseminate constraints by posting the `SignedConstraints` to the `postConstraints` endpoint in the [Constraints API](https://eth-fabric.github.io/constraints-specs/#/Constraints%20API/postConstraints).