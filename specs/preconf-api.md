# Preconfirmations API Specification

## Abstract

A proposed generic API specification to support Ethereum and Layer 2 preconfirmations. The API is compatible with the existing PBS architecture and [builder-specs](https://github.com/ethereum/builder-specs). This specification allows proposers to offer preconfirmations to users either directly or by delegating the privilege to a Gateway.

## Motivation

Ethereum’s 12 second block time may be restrictive for particular use cases and is especially inhibitive for L2s that rely on fast confirmations. One option is to reduce the block time. However, this is a very large lift and would likely require multiple hard forks. This alternative option extends the existing PBS architecture, which allows proposers or constraint delegates (Gateways) to offer transaction preconfirmations to users.


### API Scope

**In Scope**

- the delegation from proposers to gateways
- the submission of constraints
- the retrieval of constraints

Also known as the Constraints API

**Out of Scope**
- Any interaction between thrid parties (Users, RPC Router, etc) and the Gatway.
- Commitments from Gateways to third parties

Also known as the Commitments API

# Terminology

| Term           | Description                                                                                                                                    |
|----------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| Preconfer      | A proposer who registers to offer preconfirmations is a preconfer.  Any party the proposer delegates preconf authority to is also a preconfer. |
| Gateway        | A party which has been delegated preconf authority by the proposer                                                                             |
| Preconf Router | The component that provides an abstracted EVM RPC API endpoint for users to submit L2 transactions and get preconfirmations                    |

# API Summary

---

| **Namespace** | **Endpoint** | **Description** |
| --- | --- | --- |
| `constraints`  | `POST` /delegate | Endpoint for proposer to delegate constraint submission rights to a Gateway. |
| `constraints` | `POST` /constraints | Endpoint for submitting a batch of signed constraints from either the proposer or gateway to the relay. |
| `constraints` | `GET` /header_with_proofs | Endpoint for requesting a builder bid with constraint proofs. |
| `constraints` | `GET` /delegations | Returns the active delegations for the proposer of this slot, if it exists. |
| `constraints` | `GET` /constraints | Returns all constraints for a given slot. |
| `constraints`  | `GET` /constraints_stream | Returns an SSE stream of constraints. |
| `constraints`  | `POST` /blocks_with_proofs | Endpoint for submitting blocks with inclusion proofs. |

---
### Overview
![image.png](../img/preconf-api-diagram.png)

# Constraints API: Builder

---

## **Overview**

The Constraints API is the way for **proposers** to communicate with the **gateway** in the PBS pipeline. The constraints-API adds the following new responsibilities:

- Proposers should be able to delegate preconfirmation rights, or more accurately, constraint submission rights to a gateway
- A proposer or gateway should be able to submit constraints through the builder API
- Proposers should be able to get bids with proofs of constraint validity

The Constraints API is also the way for block builders to communicate bids to relays in the PBS pipeline. New responsibilities:

- Getter function for delegations and constraints in a slot.
- Subscription to new constraints using Server Side Events (SSE).
- Implement an updated block submission endpoint with support for inclusion proofs.

## `Constraints` namespace

This namespace defines endpoints that should be called by either the validator or a gateway.


### Endpoint: `/constraints/v0/delegate`

Endpoint for proposer to delegate constraint submission rights to a Gateway. 

- Method: `POST`
- Response: Empty
- Headers:
    - `Content-Type: application/json`

**Schema**

```python
# A signed delegation
class SignedDelegation(Container):
    message: Delegation
    signature: BLSSignature

# A delegation from a proposer to a BLS public key
class Delegation(Container):
    validator_pubkey: BLSPubkey
    delegatee_pubkey: BLSPubkey 
    slasher_address: Address
    metadata: Bytes
    
```

- **Description**
    
    For proposers to delegate preconfirmations to a gateway they must provide a signed delegate message. Delegation expires; this message is required for every proposed block. 
       
    The `metadata` will be made available in the slashing function. The purpose of the metadata is so the proposer has the flexibility to change parameters without needing to change the bytecode. The metadata format is flexible and up to the bytecode function to interpret. 
    
    Metadata example parameters:
    
    - Gas limit
    - Blob limit
    - ChainId
    - Preconf Type (Execution, Inclusion, State lock)

---

### Endpoint: `/constraints/v0/constraints`

Endpoint for submitting a batch of signed constraints from either the proposer or gateway to the relay.

- Method: `POST`
- Response: Empty
- Headers:
    - `Content-Type: application/json`
- Body: JSON object of type `SignedDelegation[]`

**Schema**

```jsx
# A signed "bundle" of constraints.
class SignedConstraints(Container):
    message: ConstraintsMessage
    signature: BLSSignature

# A "bundle" of constraints for a specific slot.
class ConstraintsMessage(Container):
    pubkey: BLSPubkey,
    slot: uint64
    contraints: List[Constraint]

# A contraint for a transactions
class Constraint(Container):
		transaction: Bytes
		slasher_address: Address
		metadata: Bytes
```

- **Description**
    
    For each preconfirmation proposers and gateways provide, they will need to create a matching constraint. These constraints need to be posted to the relay. 
    
    **Note**: Ordering of `contraints[]` does not need to be explicit, though it can be enforced in the bytecode.
    
    The `metadata` will be made available in the slashing function. This is separate to the delegate `metadata` and it’s purpose is to offer transactions-specific commitment parameters because the bytecode is immutable.  The metadata format is flexible and up to the bytecode function to interpret. 
    
    metadata example parameters:
    
    - Index
    - Preconf type

### Endpoint: **`/constraints/header_with_proofs/{slot}/{parent_hash}/{pubkey}`**

Endpoint for requesting a builder bid with constraint proofs.

- **Method:** `GET`
- **Response:** `VersionedSignedBuilderBidWithProofs`
- **Parameters:**
    - `slot`: `string` (regex `[0-9]+`)
    - `parent_hash`: `string` (regex `0x[a-fA-F0-9]+`)
    - `pubkey`: `string` (regex `0x[a-fA-F0-9]+`)
- **Body:** Empty

**Schema**

```jsx
class VersionedSignedBuilderBidWithProofs:
    ... # All regular fields from VersionedSignedBuilderBid, additionally
    proofs: InclusionProofs

# An SSZ Merkle Multiproof for proving inclusion against the transactions_root
class InclusionProofs(Container):
  transaction_hashes: List[Bytes32, MAX_CONSTRAINTS_PER_SLOT]
  generalized_indexes: List[uint64, MAX_CONSTRAINTS_PER_SLOT]
  merkle_hashes: List[List[Bytes32], MAX_CONSTRAINTS_PER_SLOT]
```

- **Description**
    
    `VersionedSignedBuilderBid` is from the [original specs](https://ethereum.github.io/builder-specs/#/Builder/getHeader).  `VersionedSignedBuilderBidWithProofs` just adds a field for proofs of inclusion. Note that `InclusionProofs` is a Merkle multiproof, as defined in the [consensus specs](https://github.com/ethereum/consensus-specs/blob/dev/ssz/merkle-proofs.md#merkle-multiproofs).
    
    When serializing, the `proofs` field must be present in `data`, at the same level of `signature` and `message`. See the example below.
    
- **Example Response**
    
    ```python
    {
        "version": "deneb",
        "data": {
            "message": {
                "header": {
                    "parent_hash": "0xcf8e0d4e9587369b2301d0790347320302cc0943d5a1884560367e8208d920f2",
                    "fee_recipient": "0xabcf8e0d4e9587369b2301d0790347320302cc09",
                    "state_root": "0xcf8e0d4e9587369b2301d0790347320302cc0943d5a1884560367e8208d920f2",
                    "receipts_root": "0xcf8e0d4e9587369b2301d0790347320302cc0943d5a1884560367e8208d920f2",
                    "logs_bloom": "0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
                    "prev_randao": "0xcf8e0d4e9587369b2301d0790347320302cc0943d5a1884560367e8208d920f2",
                    "block_number": "1",
                    "gas_limit": "1",
                    "gas_used": "1",
                    "timestamp": "1",
                    "extra_data": "0xcf8e0d4e9587369b2301d0790347320302cc0943d5a1884560367e8208d920f2",
                    "base_fee_per_gas": "1",
                    "blob_gas_used": "1",
                    "excess_blob_gas": "1",
                    "block_hash": "0xcf8e0d4e9587369b2301d0790347320302cc0943d5a1884560367e8208d920f2",
                    "transactions_root": "0xcf8e0d4e9587369b2301d0790347320302cc0943d5a1884560367e8208d920f2",
                    "withdrawals_root": "0xcf8e0d4e9587369b2301d0790347320302cc0943d5a1884560367e8208d920f2"
                },
                "blob_kzg_commitments": [
                    "0xa94170080872584e54a1cf092d845703b13907f2e6b3b1c0ad573b910530499e3bcd48c6378846b80d2bfa58c81cf3d5"
                ],
                "value": "1",
                "pubkey": "0x93247f2209abcacf57b75a51dafae777f9dd38bc7053d1af526f220a7489a6d3a2753e5f3e8b1cfe39b56f43611df74a"
            },
            "proofs": {
                "transaction_hashes": ["0x1234...", "0x456..."],
                "generalized_indexes": [4, 5],
                "merkle_hashes": ["0x5097...", "0x932587..."]
            },
            "signature": "0x1b66ac1fb663c9bc59509846d6ec05345bd908eda73e670af888da41af171505cc411d61252fb6cb3fa0017b679f8bb2305b26a285fa2737f175668d0dff91cc1b66ac1fb663c9bc59509846d6ec05345bd908eda73e670af888da41af171505"
        }
    }
    ```
    
## Overview




### Endpoint: `/constraints/v0/delegations?slot={slot}`

Return the active delegations for the proposer of this slot, if it exists.

- Method: `GET`
- Parameters:
    - `slot: uint64`
- Headers:
    - `Content-Type: application/json`
- Body: Empty
- Response: JSON object of type `SignedDelegation[]`

**Schema**

```python
TODO: Needs to be updated to support the changes in the /delegate call
```

- **Description**

---

### Endpoint: `/constraints/v0/constraints?slot={slot}`

Returns all constraints for a given slot.

- Method: `GET`
- Parameters:
    - `slot: uint64`
- Headers:
    - `Content-Type: application/json`
- Body: Empty
- Response: JSON object of type `SignedConstraints[]`

**Schema**

```python
# TODO: Needs to be updated to support the changes in the builder/delegate call

class SignedConstraints(Container):
    message: ConstraintsMessage
    signature: BLSSignature

class ConstraintsMessage(Container):
    pubkey: uint64,
    slot: uint64
    top: boolean,
    transactions: List[Bytes, MAX_CONSTRAINTS_PER_SLOT]
```

- **Description**

---

### Endpoint: `/constraints/v0/constraints_stream?slot={slot}`

Returns an SSE stream of constraints.

- Method: `GET`
- Parameters: Empty
- Headers:
    - `Content-Type: application/json`
    - `Connection: keep-alive`
- Body: Empty
- Response: stream of JSON objects of type `SignedConstraints[]`

**Schema**

```python
# TODO: Needs to be updated to support the changes in the builder/delegate call

class SignedConstraints(Container):
    message: ConstraintsMessage
    signature: BLSSignature

class ConstraintsMessage(Container):
    pubkey: BLSPubkey,
    slot: uint64
    top: boolean,
    transactions: List[Bytes, MAX_CONSTRAINTS_PER_SLOT]
```

- **Description**

---

### Endpoint: `/constraints/v0/blocks_with_proofs?cancellations={cancellations}`

Endpoint for submitting blocks with inclusion proofs.

- Method: `POST`
- Parameters: `cancellations: bool` (query)
- Headers:
    - `Content-Type: application/json`
- Body: JSON object of type `VersionedSubmitBlockRequestWithProofs`
- Response: Empty

**Schema**

```jsx
class VersionedSubmitBlockRequestWithProofs(Container):
  ... # All regular fields from VersionedSubmitBlockRequest, additionally
  proofs: InclusionProofs

class InclusionProofs(Container):
  transanction_hashes: List[Bytes32, MAX_CONSTRAINTS_PER_SLOT]
  generalized_indexes: List[uint64, MAX_CONSTRAINTS_PER_SLOT]
  merkle_hashes: List[List[Bytes32], MAX_CONSTRAINTS_PER_SLOT]
```

- **Description**
    
    `VersionedSubmitBlockRequest` is from the [original specs](https://flashbots.github.io/relay-specs/#/Builder/submitBlock). `VersionedSubmitBlockRequestWithProofs` just adds proofs of inclusion. Note that `InclusionProofs` is a Merkle multiproof, as defined in the [consensus specs](https://github.com/ethereum/consensus-specs/blob/dev/ssz/merkle-proofs.md#merkle-multiproofs).
    

## Bytecode Examples

### /Delegate

```solidity
function slash(bytes inputs) public uint256 {
		
		// Verfiy the validator and delagate commited 
		// Verify validator signature 
		// Verify the signature of the delegate
	
}
```
