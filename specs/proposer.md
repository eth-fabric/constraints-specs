# Honest Proposer Specification

## Table of Contents
todo - add when doc is finalized

## Introduction

This document explains the way in which an honest Proposer is expected to use the [Constraints API](https://eth-fabric.github.io/constraints-specs/) and [Universal Registry Contract](https://github.com/eth-fabric/urc) (URC) to issue proposer commitments. The language and format of this document is meant to mirror the original [Honest Validator guide in the Builder Specs](https://github.com/ethereum/builder-specs/blob/main/specs/bellatrix/validator.md#bellatrix----honest-validator). 

At a high-level, a proposer will mirror the flow of the Honest Validator guide in the Builder Specs with a few registration and verification steps. There is a one-time registration step where proposers post collateral and register their BLS keys on-chain to the URC. After a fraud-proof window, proposers will sign off-chain `Delegation` messages for Gateways to know which slots they are responsible for issuing commitments for. Gateways are then responsible for issuing proposer commitments on behalf of proposers via the Commitments API and then instructing Builders how to construct a valid L1 block via `Constraint` messages. When it is the proposer’s turn to propose the next block proceed as normal for the Builder Specs except they additionally verify a proof that their block satisfies the constraints. In the event that a proposer commitment is broken, the proposer’s collateral can be slashed by supplying evidence to the URC. 

## Prerequisites
This document assumes knowledge of the terminology, definitions, and other material in the Builder spec, Constraints API, Commitments API, and URC.

### Definitions

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

## Proposer Registration
A proposer will follow the standard registration process in the Builder Spec as well as a new on-chain registration with the URC and off-chain delegations to Gateway.

### Registering to the URC
The proposer will register to the URC by submitting `Registration` messages for each of the validator BLS keys they wish to register as well as Ether collateral.

#### **Preparing registrations**
1. The proposer selects an `owner` execution layer address to be their admin account for the URC. This can be an EOA or a smart contract address (i.e., a multi-sig).

2. The proposer generates a signature with the BLS key they wish to register.

    ```python
    message = abi.encode(owner_address)
    signature = BLS.sign(message, REGISTRATION_DOMAIN_SEPARATOR, bls_private_key)
    ```

    Note that the `REGISTRATION_DOMAIN_SEPARATOR` is defined in the URC as: `"0x00435255"` which is "URC" in little endian.

3. The `signature` is placed in a `Registration` object with the BLS public key.

```Solidity
    struct Registration {
        /// BLS public key
        BLS.G1Point pubkey;
        /// BLS signature
        BLS.G2Point signature;
    }
```

#### **Signing and submitting a registration**
The proposer will repeat steps 2-3 for each BLS key they wish to register. Once all registrations are prepared, the proposer will submit them to the URC via the `register()` function.
```Solidity
function register(Registration[] calldata regs, address owner)
    external payable returns (bytes32 registrationRoot)
```

The proposer is required to send at least `MIN_COLLATERAL` Ether (as defined in the URC) when calling `register()`. 

Registration is a one-time operation and the proposer will not be able to modify their registration (i.e. add or remove BLS keys) once it has been submitted.

The proposer must wait for `FRAUD_PROOF_WINDOW` blocks to elapse before their registration process is finalized to allow for the possibility of a fraud-proof being submitted.

### Registering via the Builder API
The following steps are unchanged as the Constraints spec is meant to accompany the Builder spec. Here the proposer will sign `ValidatorRegistration` messages and submit them via the Builder API.

The registration steps are linked below for reference.

- [Preparing a registration](https://github.com/ethereum/builder-specs/blob/main/specs/bellatrix/validator.md#preparing-a-registration)
- [Signing and submitting a registration](https://github.com/ethereum/builder-specs/blob/main/specs/bellatrix/validator.md#signing-and-submitting-a-registration)
- [Registration dissemination](https://github.com/ethereum/builder-specs/blob/main/specs/bellatrix/validator.md#registration-dissemination)

### Delegating to Gateways
The proposer disseminates `SignedDelegation` messages to instruct the external builder network about which keys have the rights to issue constraints and commitments on behalf of their BLS keys.

#### **Preparing a delegation**
The proposer assembles a `Delegation` object for their BLS key with the [following information](./constraints-api.md#delegation):
- `proposer`: the BLS public key of the validator
- `delegate`: the BLS public key that can issue constraints
- `committer`: an execution layer address that can issue commitments
- `slot`: the slot number of the block that the delegation is for
- `metadata`: arbitrary bytes reserved for future use

It is not required but is assumed that the `delegate` BLS and `committer` ECDSA private key belong to the Gateway.

#### **Signing and submitting a delegation**
1. The proposer generates a `signature` with the BLS key they wish to delegate.

    ```Python
    message = abi.encode(delegation)
    signature = BLS.sign(message, DELEGATION_DOMAIN_SEPARATOR, bls_private_key)
    ```

    Note that the `DELEGATION_DOMAIN_SEPARATOR` is defined in the URC as: `"0x0044656c"` which is "Del" in little endian.

2. The `signature` is placed in a `SignedDelegation` object with the BLS public key.
    ```Python
    class SignedDelegation(Container):
        message: Delegation
        signature: BLSSignature
    ```

#### **Delegation dissemination**
Proposers are expected to send their `SignedDelegation` messages to the external builder network using the `POST /delegate` endpoint in the [Constraints API](./constraints-api.md#delegation).

Delegations are expected to be disseminated across the external builder network via Relays.

Proposers should submit valid Delegations ahead of any their block proposal duties to ensure Gateways have time to submit constraints and commitments on their behalf.

Delegations should only be signed at most once per slot as it is a slashable offense for a proposer to sign multiple delegations for the same slot.

Delegations are not cancellable but are only valid for a single slot.

### Opting in to Slasher contract's on-chain (Optional)
The URC optionally allows an on-chain way for proposers to opt in to a proposer commitment protocol's `Slasher` contract, allowing for valid-until-cancelled commitments.

#### **Preparing the inputs**
1. The proposer will select the `Slasher` contract of the proposer commitment protocol they wish to opt in to.

2. The proposer will choose a `committer` address that is allowed to issue commitments on behalf of the proposer for this proposer commitment protocol.

#### **Updating the URC**
The proposer will call the `optInToSlasher()` function in the URC with the `Slasher` contract address, `committer` address, and the `RegistrationRoot` from the URC registration step.
```Solidity
function optInToSlasher(bytes32 registrationRoot, address slasher, address committer) external
```

This function can only be called after the proposer has registered to the URC and the `FRAUD_PROOF_WINDOW` has elapsed.

## Block Proposals
The following steps largely follow the Builder Spec with step 2 being the only additional verification step to ensure the proposer commitment is satisfied.

### Constructing the `BeaconBlockBody`
#### `ExecutionPayload`

To obtain an execution payload, a block proposer building a block on top of a beacon state in a given slot must take the following actions:

1. Call upstream builder software to get an `ExecutionPayloadHeader` with the required data `slot`, `parent_hash`, and `pubkey`, where:
- `slot` is the proposal's slot
- `parent_hash` is the value `state.latest_execution_payload_header.block_hash`
- `pubkey` is the proposer's public key

2. Verify the `proofs` against the `ExecutionPayloadHeader` to ensure that the constraints are satisfied.

3. Assemble a `BlindedBeaconBlock` according to the process outlined in the Electra specs but with the `ExecutionPayloadHeader` from the prior step in lieu of the full `ExecutionPayload`.

4. The proposer signs the `BlindedBeaconBlock` and assembles a `SignedBlindedBeaconBlock` which is returned to the upstream builder software.

5. The upstream builder software responds with the full `ExecutionPayload`. The proposer can use this payload to assemble a `SignedBeaconBlock` following the rest of the proposal process outlined in the Bellatrix specs.

#### Bid processing
Bids received from step (1) above can be validated with `process_bid` below, where `state` corresponds to the state for the proposal without applying the block (currently under construction) and `fee_recipient` corresponds to the validator's most recently registered fee recipient address:
```Python
def verify_bid_signature(state: BeaconState, signed_bid: SignedBuilderBid) -> bool:
    pubkey = signed_bid.message.pubkey
    domain = compute_domain(DOMAIN_APPLICATION_BUILDER)
    signing_root = compute_signing_root(signed_registration.message, domain)
    return bls.Verify(pubkey, signing_root, signed_bid.signature)
```
A bid is considered valid if the following function completes without raising any assertions:

```Python
def process_bid(state: BeaconState, bid: SignedBuilderBid, fee_recipient: ExecutionAddress):
    # Verify execution payload header
    header = bid.message.header
    assert header.parent_hash == state.latest_execution_payload_header.block_hash
    assert header.fee_recipient == fee_recipient
    
    # Verify bid proofs
    verify_bid_proofs(state, bid)

    # Verify bid signature
    verify_bid_signature(state, bid)
```
