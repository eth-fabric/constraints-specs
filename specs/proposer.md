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

```python
    class Registration(Container):
        pubkey: BLS.G1Point
        signature: BLS.G2Point
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

    Note RLP encoding is used instead of SSZ for simpler on-chain verification.

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

#### How does this relate to slashing?
`Delegation` messages do not directly commit a proposer to any proposer commitment protocol's slashing conditions. Instead, they are used to instruct the external builder network about which keys have the rights to issue constraints and commitments on behalf of the proposer. 

It is left to the `committer` to opt in to proposer commitment protocols but the spec does not mandate any specific way to do so. Alternatively, the proposer can directly opt in to a protocol's `Slasher` contract via the URC as described in the next section.

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

## Proposer Deregistration
Unlike the Builder spec, proposer commitments require collateral to be posted to the URC so there is a need to define the deregistration process.

### Undelegating from Gateways
The spec does not support undelegating from Gateways as it introduces race conditions surrounding slashing as well as a way to bypass equivocation slashing. Therefore, once signed, a `Delegation` is final and cannot be invalidated until the `slot` has elapsed.

### Unregistering from the URC
Unregistering from the URC is a two-step process:

#### Calling `unregister()`
The `owner` address of the URC can call `unregister()` to initiate the deregistration process, saving the block number that it was called.

```Solidity
function unregister(bytes32 registrationRoot) external;
```

#### Calling `claimCollateral()`
The `owner` address can call `claimCollateral()` to retrieve their collateral after `UNREGISTRATION_DELAY` blocks have elapsed. 

```Solidity
function claimCollateral(bytes32 registrationRoot) external;
```

The proposer's collateral is transferred to the `owner` address.

#### Special case: `claimSlashedCollateral()`

If the proposer was slashed, the `owner` address can call `claimSlashedCollateral()` after `SLASH_WINDOW` blocks have elapsed to retrieve their remaining collateral.

```Solidity
function claimSlashedCollateral(bytes32 registrationRoot) external;
```

The proposer's remaining collateral is transferred to the `owner` address.

### Opting out of slashing (Optional)
If a proposer previously opted in to a slasher contract [as described above](#opt-in-to-slasher-contracts-optional), they can opt out by calling `optOutOfSlasher()` after `OPT_IN_DELAY` blocks have elapsed.

```Solidity
function optOutOfSlasher(bytes32 registrationRoot, address slasher) external
```

## Slashing
There are different ways in which a proposer commitment can be broken.

### Liveness faults
A liveness fault is when a proposer fails to submit a block during their slot. They are subject to penalties from PoS and potential slashing from the proposer commitment protocols they or their delegated `committer` opted into. 

### Safety faults
#### Equivocation in PoS
A safety fault in PoS is when the proposer equivocates when producing a block or attesting to other blocks. They are subject to penalties defined in the [Eth2 specs](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/validator.md#how-to-avoid-slashing).

#### Safety faults in proposer commitment protocols
A safety fault is when a proposer submits a block that breaks the commitments they made. They are subject to penalties defined in the `Slasher` contracts they've opted in to.

#### Proposer initiated
A proposer can trigger a safety fault by submitting a block that breaks the commitments they made.

The trivial case is by proposing a self-built block that ignores all commitments issued by their delegated Gateway.

#### Gateway initiated
A Gateway can trigger a safety fault by issuing commitments that are not satisfied by the proposer's block.

An example of this is if the Gateway fails to disseminate all of the constraints to the external builder network resulting in a block that does not satisfy the issued commitments.

#### Slashing from invalid URC registrations
For efficiency, the URC optimistically assumes that all `Registration` messages are valid. Therefore, if a registration is found to be invalid within the `FRAUD_PROOF_WINDOW`, a challenger can slash the proposer by submitting a fraud proof to the URC's `slashRegistration()` function.

Slashing can be avoided by ensuring that the `Registration.signature` is generated according to the spec [outlined above](#registering-to-the-urc).

#### Slashing from equivocating delegations
The URC mandates that `Delegation` messages are signed at most once per slot per BLS key. If a proposer is caught signing multiple delegations for the same slot, they can be slashed by submitting the conflicting delegations to the URC's `slashEquivocation()` function.

To avoid slashing, sidecars should ensure that `Delegation` messages are only signed once per slot per BLS key (e.g., following in the footsteps of [EIP-3076](https://eips.ethereum.org/EIPS/eip-3076)).

### Understanding the URC's `slashCommitment()` function
#### Signing commitments
The `committer` address delegated to within `Delegation` messages is authorized to sign `Commitment` messages on behalf of the proposer.

```python
class Commitment(Container):
    commitmentType: uint64
    payload: Bytes
    slasher: Address
```
A `signature` is generated with the `committer`'s private key, where the `committer` is a standard execution layer address:

```python
message = keccak256(abi.encode(commitment))
signature = ECDSA.sign(message, committer_private_key)
```

Note RLP encoding is used instead of SSZ for simpler on-chain verification.

The `signature` is placed in a `SignedCommitment` object with the `Commitment`.

```python
class SignedCommitment(Container):
    commitment: Commitment
    signature: Bytes
```

#### Slashing a broken commitment
To slash a proposer, anyone can submit the necessary evidence to the URC's `slashCommitment()` function.
```Solidity
function slashCommitment(
    bytes32 registrationRoot,
    BLS.G2Point calldata registrationSignature,
    bytes32[] calldata proof,
    uint256 leafIndex,
    ISlasher.SignedDelegation calldata delegation,
    ISlasher.SignedCommitment calldata commitment,
    bytes calldata evidence
) external returns (uint256 slashAmountGwei);
```

The parameters are as follows:
- `registrationRoot`: the unique identifier for the proposer saved in the URC during the `register()` function
- `registrationSignature`: the signature that opted the proposer's BLS key into the URC
- `proof`: a Merkle proof connecting the BLS key to the `registrationRoot`
- `leafIndex`: the index of the proposer's registration in the Merkle proof
- `delegation`: the `SignedDelegation` message that was signed by the proposer's BLS key
- `commitment`: the `SignedCommitment` message that was signed by the `committer` address delegated to within the `Delegation` message
- `evidence`: arbitrary data that can be used to provide additional information about the slashing to the `Slasher` contract

The function will ensure that the `SignedDelegation` is valid and from a registered proposer. It will then verify that the `SignedCommitment` is valid and from a delegated `committer` address. The function will the call into the committed `Slasher` contract using the standardized `ISlasher.slash()` function:

```Solidity
function slash(
    Delegation calldata delegation,
    Commitment calldata commitment,
    bytes calldata evidence,
    address challenger
) external returns (uint256 slashAmountGwei);
```

Each `Slasher` contract will define their own slashing logic and opt-in conditions but will all return `slashAmountGwei` which is the amount of Ether collateral to be burned from the proposer's collateral back at the URC. 



Note, the `slashCommitmentFromOptIn()` functions identically to `slashCommitment()` except it doesn't verify `SignedDelegation` as the `committer` address is already known to the URC from [previous calls](./constraints-api.md#opt-in-to-slasher-contracts-optional) to `optInToSlasher()`.