# Honest Proposer Specification

## Table of Contents
- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
  - [Definitions](#definitions)
  - [Constants](#constants)
    - [Domain types](#domain-types)
    - [URC parameters](#urc-parameters)
- [Proposer Registration](#proposer-registration)
    - [Registering to the URC](#registering-to-the-urc)
        - [Preparing registrations](#preparing-registrations)
        - [Signing and submitting a registration](#signing-and-submitting-a-registration)
    - [Registering via the Builder API](#registering-via-the-builder-api)
    - [Delegating to Gateways](#delegating-to-gateways)
        - [Preparing a delegation](#preparing-a-delegation)
        - [Signing and submitting a delegation](#signing-and-submitting-a-delegation)
        - [Delegation dissemination](#delegation-dissemination)
        - [How does this relate to slashing?](#how-does-this-relate-to-slashing)
    - [Opting in to Slasher contracts on-chain (Optional)](#opting-in-to-slasher-contracts-on-chain-optional)
- [Block Proposals](#block-proposals)
    - [Constructing the BeaconBlockBody](#constructing-the-beaconblockbody)
        - [ExecutionPayload](#executionpayload)
        - [Bid processing](#bid-processing)
- [Proposer Deregistration](#proposer-deregistration)
    - [Undelegating from Gateways](#undelegating-from-gateways)
    - [Unregistering from the URC](#unregistering-from-the-urc)
        - [Calling unregister()](#calling-unregister)
        - [Calling claimCollateral()](#calling-claimcollateral)
        - [Special case: calling claimSlashedCollateral()](#special-case-claimslashedcollateral)
- [Slashing](#slashing)
    - [Liveness faults](#liveness-faults)
    - [Safety faults](#safety-faults)
        - [Equivocation in PoS](#equivocation-in-pos)
        - [Safety faults in proposer commitment protocols](#safety-faults-in-proposer-commitment-protocols)
        - [Proposer initiated](#proposer-initiated)
        - [Gateway initiated](#gateway-initiated)
        - [Slashing from invalid URC registrations](#slashing-from-invalid-urc-registrations)
        - [Slashing from equivocating delegations](#slashing-from-equivocating-delegations)
    - [Understanding the URC's slashCommitment() function](#understanding-the-urcs-slashcommitment-function)
        - [Signing commitments](#signing-commitments)
        - [Slashing a broken commitment](#slashing-a-broken-commitment)

## Introduction

This document explains the way in which an honest Proposer is expected to use the [Constraints API](https://eth-fabric.github.io/constraints-specs/) and [Universal Registry Contract](https://github.com/eth-fabric/urc) (URC) to issue proposer commitments. The language and format of this document is meant to mirror the original [Honest Validator guide in the Builder Specs](https://github.com/ethereum/builder-specs/blob/main/specs/bellatrix/validator.md#bellatrix----honest-validator). At a high-level, a proposer will mirror the flow of the Honest Validator guide in the Builder Specs but with a few additional registration. 

There is a one-time registration step where proposers post collateral and register their BLS keys on-chain to the URC. After a fraud-proof window, proposers will sign off-chain `Delegation` messages for Gateways to know which slots they are responsible for issuing commitments for. 

Gateways are then responsible for issuing proposer commitments on behalf of proposers via the [Commitments API](https://github.com/eth-fabric/commitments-specs) and then instructing Builders how to construct a valid L1 block via signed `Constraint` messages. When it is the proposer’s turn to propose the next block, they proceed as normal for the Builder Specs (i.e., calling `GET /header`). In the event that a proposer commitment is broken, the proposer’s collateral can be slashed by supplying evidence to the URC (see the [fault attribution guidelines](fault-attribution.md) file for more information). 

## Prerequisites
This document assumes knowledge of the terminology, definitions, and other material in the Builder spec, Constraints API, Commitments API, and URC.

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
Note the following constants are subject to change prior to the launch of the URC.

### Domain types

| Name | Value |
| - | - |
| `DELEGATION_DOMAIN_SEPARATOR` | `DomainType('0x0044656c')` |
| `REGISTRATION_DOMAIN_SEPARATOR` | `DomainType('0x00435255')` |

### URC parameters

| Name | Value |
| - | - |
| `MIN_COLLATERAL` | 0.1 ether |
| `UNREGISTRATION_DELAY` | 86400 seconds |
| `FRAUD_PROOF_WINDOW` |  86400 seconds |
| `SLASH_WINDOW` | 86400 seconds |
| `OPT_IN_DELAY` | 86400 seconds |

## Proposer Registration
Proposers will follow the standard registration process in the [Builder Spec](https://github.com/ethereum/builder-specs/blob/main/specs/bellatrix/builder.md#validator-registration-processing), specifically validators will still sign `ValidatorRegistration` messages to register to begin working with Relays and Builders. Additionally there is a new on-chain registration with the URC and off-chain delegations to Gateway.

### Registering to the URC
The proposer will register to the URC by submitting `Registration` messages for each of the validator BLS keys they wish to register as well as Ether collateral.

#### **Preparing registrations**
1. The proposer selects an `owner` execution layer address to be their admin account for the URC. This can be an EOA or a smart contract address (i.e., a multi-sig).

2. The proposer generates a signature with the BLS key they wish to register.

    ```python
    def get_urc_registration_signature(
        owner: Address,
        privkey: int
    ) -> BLSSignature:
        # note: abi-encoded, not SSZ
        message = abi.encode(owner)
        return BLS.sign(message, privkey, REGISTRATION_DOMAIN_SEPARATOR)
    ```

3. The `signature` is placed in a `SignedRegistration` object with the BLS public key.

    ```python
    class SignedRegistration(Container):
        pubkey: BLS.G1Point # note the encoding matches URC not beacon specs
        signature: BLS.G2Point # note the encoding matches URC not beacon specs
    ```

#### **Signing and submitting a registration**
The proposer will repeat steps 2-3 for each BLS key they wish to register. Once all registrations are prepared, the proposer will package the `SignedRegistration` objects.

```python
def get_signed_registration(
    sigs: List[BLSSignature],
    pubkeys: List[BLSPubkey]
) -> List[SignedRegistration]:
    # encode to BLS.G1Point and BLS.G2Point
    return [
        SignedRegistration(to_g1_point(pk), to_g2_point(sig))
        for pk, sig in zip(sigs, pubkeys)
    ]
```

They will then submit them to the URC via the `register()` function.
```Solidity
function register(SignedRegistration[] registrations, address owner)
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
The proposer disseminates `SignedDelegation` messages to instruct Relays and Gateways about which keys have the rights to issue constraints and commitments on behalf of their BLS keys.

#### **Preparing a delegation**
The proposer assembles a `Delegation` object for their BLS key with the [following information](./constraints-api.md#delegation):
```python
class Delegation(Container):
    # The proposer's BLS public key
    proposer: BLS.G1Point
    # The delegate's BLS public key for Constraints API
    delegate: BLS.G1Point
    # The address of the delegate's ECDSA key for signing commitments
    committer: Address
    # The L1 slot number the delegation is valid for
    slot: Slot
    # Arbitrary metadata reserved for future use
    metadata: Bytes
```

It is not required but is assumed that the `delegate` and `committer` private keys belong to the Gateway.

#### **Signing and submitting a delegation**
1. The proposer generates a `signature` with the BLS key they wish to delegate.

    ```Python
    def get_delegation_signature(
        delegation: Delegation,
        privkey: int
    ) -> SignedDelegation:
        # note: abi-encoded, not SSZ
        message = abi.encode(delegation)
        signature = BLS.sign(message, privkey, DELEGATION_DOMAIN_SEPARATOR)
        return SignedDelegation(message=delegation, signature=signature)
    ```

    Note RLP encoding is used instead of SSZ for simpler on-chain verification.

2. The `signature` is placed in a `SignedDelegation` object with the BLS public key.
    ```Python
    class SignedDelegation(Container):
        message: Delegation
        signature: BLS.G2Point
    ```

#### **Delegation dissemination**
Proposers are expected to send their `SignedDelegation` messages to Relays using the `POST /delegate` endpoint in the [Constraints API](./constraints-api.md#delegation).

- Delegations are expected to be disseminated via Relays.
- Proposers should submit valid Delegations ahead of any their block proposal duties to ensure Gateways have time to submit constraints and commitments on their behalf.
- Delegations should only be signed at most once per slot as it is a slashable offense for a proposer to sign multiple delegations for the same slot.
- Delegations are not cancellable but are only valid for a single slot.

#### How does this relate to slashing?
`Delegation` messages do not directly commit a proposer to any proposer commitment protocol's slashing conditions. Instead, they are used to instruct Relays and Gateways about which keys have the rights to issue constraints and commitments on behalf of the proposer. 

It is left to the `committer` to opt in to proposer commitment protocols but the spec does not mandate any specific way to do so. Alternatively, the proposer can directly opt in to a protocol's `Slasher` contract via the URC as described in the next section.

### Opting in to Slasher contracts on-chain (Optional)
The URC optionally allows an on-chain way for proposers to opt in to a proposer commitment protocol's `Slasher` contract, which are valid-until-cancelled.

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
Block proposal mirror the Builder Spec. The following links are here for reference:
- [Constructing the `BeaconBlockBody`](https://github.com/ethereum/builder-specs/blob/main/specs/bellatrix/validator.md#constructing-the-beaconblockbody)
- [Bid processing](https://github.com/ethereum/builder-specs/blob/main/specs/bellatrix/validator.md#bid-processing)

## Proposer Deregistration
Unlike the Builder spec, proposer commitments require collateral to be posted to the URC so there is a need to define the deregistration process.

### Undelegating from Gateways
The spec does not support undelegating from Gateways as it introduces race conditions surrounding slashing as well as a way to bypass equivocation slashing. Therefore, once signed, a `Delegation` is final and cannot be invalidated until the `slot` has elapsed.

### Unregistering from the URC
Unregistering from the URC is a two-step process:

#### Calling `unregister()`
The proposer's `owner` address in the URC can call `unregister()` to initiate the deregistration process, saving the block timestamp that it was called.

```Solidity
function unregister(bytes32 registrationRoot) external;
```

#### Calling `claimCollateral()`
The `owner` address can call `claimCollateral()` to retrieve their collateral after `UNREGISTRATION_DELAY` seconds have elapsed. 

```Solidity
function claimCollateral(bytes32 registrationRoot) external;
```

The proposer's collateral is transferred to their `owner` address.

#### Special case: `claimSlashedCollateral()`

If the proposer was slashed, the `owner` address can call `claimSlashedCollateral()` after `SLASH_WINDOW` seconds have elapsed to retrieve their remaining collateral.

```Solidity
function claimSlashedCollateral(bytes32 registrationRoot) external;
```

The proposer's remaining collateral is transferred to the `owner` address.

### Opting out of slashing (Optional)
If a proposer previously opted in to a slasher contract [as described above](#opting-in-to-slasher-contracts-on-chain-optional), they can opt out by calling `optOutOfSlasher()` after `OPT_IN_DELAY` seconds have elapsed.

```Solidity
function optOutOfSlasher(bytes32 registrationRoot, address slasher) external
```

## Slashing
There are different ways in which a proposer commitment can be broken. See the [fault attribution document](fault-attribution.md) to learn more on techniques for proposer commitment protocols to correctly attribute fault. 

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

An example of this is if the Gateway fails to disclose all of the constraints to the Relays and Builders resulting in a block that does not satisfy the issued commitments.

#### Slashing from invalid URC registrations
For efficiency, the URC optimistically assumes that all `SignedRegistration` messages are valid. Therefore, if a registration is found to be invalid within the `FRAUD_PROOF_WINDOW`, a challenger can slash the proposer by submitting a fraud proof to the URC's `slashRegistration()` function.

Slashing can be avoided by ensuring that the `SignedRegistration.signature` is generated according to the spec [outlined above](#registering-to-the-urc).

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
    RegistrationProof calldata proof,
    ISlasher.SignedDelegation calldata delegation,
    ISlasher.SignedCommitment calldata commitment,
    bytes calldata evidence
) external returns (uint256 slashAmountWei);
```

The parameters are as follows:
- `proof`: Proof that owner previously called the `register()` function for this BLS key
- `delegation`: the `SignedDelegation` message that was signed by the proposer's BLS key
- `commitment`: the `SignedCommitment` message that was signed by the `committer` address delegated to within the `Delegation` message
- `evidence`: arbitrary data that can be used to provide additional information about the slashing to the `Slasher` contract

The function will ensure that the `SignedDelegation` is valid and from a registered proposer. It will then verify that the `SignedCommitment` is valid and from a delegated `committer` address. The function will the call into the committed `Slasher` contract using the standardized `ISlasher.slash()` function:

```Solidity
function slash(
    Delegation calldata delegation,
    Commitment calldata commitment,
    address committer,
    bytes calldata evidence,
    address challenger
) external returns (uint256 slashAmountWei);
```

Each `Slasher` contract will define their own slashing logic and opt-in conditions but will all return `slashAmountGwei` which is the amount of Ether collateral to be burned from the proposer's collateral back at the URC. 

Note, `slashCommitment()` is overloaded with two implementations. The version described above is meant for slashing based on off-chain delegations. The second version functions identically except it doesn't verify `SignedDelegation` as the `committer` address is already known to the URC from [previous calls](#opting-in-to-slasher-contracts-on-chain-optional) to `optInToSlasher()`.