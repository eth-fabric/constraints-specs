# Preconfirmations API Specification

## Abstract

A proposed generic API specification to support Ethereum and Layer 2 preconfirmations. The API extends the existing PBS architecture and [builder-specs](https://github.com/ethereum/builder-specs). This specification allows proposers to offer preconfirmations to users either directly or by delegating the privilege to a Gateway.

## Motivation

Ethereum’s 12 second block time may be restrictive for particular use cases and is especially inhibitive for L2s that rely on fast confirmations. One option is to reduce the block time. However, this is a very large lift and would likely require multiple hard forks. This alternative option extends the existing PBS architecture, which allows proposers or constraint delegates (Gateways) to offer transaction preconfirmations to users.


## API Scope

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
| Preconfer      | A proposer who registers to offer preconfirmations is a preconfer. Any party the proposer delegates preconf authority to is also a preconfer.  |
| Gateway        | A party which has been delegated preconf constraint and commitment submission authority by the proposer.                                       |
| Builder        | A party specialized in constructing execution payloads and proving constraints are followed.                                                   |
| Relay          | A trusted party that facilitates the exchange of execution payloads between Builders and Proposers and validating constraints are followed.    |
| RPC Router     | The component that provides an abstracted EVM RPC API endpoint for users to submit L2 transactions and get preconfirmations.                   |


# API Summary

| **Namespace** | **Endpoint** | **Description** |
| --- | --- | --- |
| `constraints`   | `POST` /delegate           | Endpoint for Proposer to delegate preconfirmation rights, or more accurately, constraint submission rights to a Gateway. |
| `constraints`   | `POST` /constraints        | Endpoint for Proposer or Gateway to submit a batch of signed constraints to the Relay. |
| `constraints`   | `GET` /header_with_proofs  | Endpoint for Proposer to request a builder bid with proof of constraint validity. |
| `constraints`   | `GET` /delegations         | Endpoint to retrieve the signed delegations for the proposer of a given slot, if it exists. |
| `constraints`   | `GET` /constraints         | Endpoint to retrieve the signed constraints for a given slot. |
| `constraints`   | `GET` /constraints_stream  | Endpoint to retrieve an SSE stream of signed constraints. |
| `constraints`   | `POST` /blocks_with_proofs | Endpoint for Builder to submit a block with inclusion proofs to the Relay. |

---
![image.png](../img/preconf-api-diagram.png)
---

# Constraints API Endpoints

### Endpoint: `/constraints/v0/relay/delegate`

Endpoint for a Proposer to delegate constraint submission rights to a Gateway. 

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
    proposer: BLSPubkey
    delegate: BLSPubkey 
    slasher: Address
    valid_until: Slot
    metadata: Bytes
```

- **Description**
    
    A proposer can delegate preconfirmations rights by signing a `Delegate` message with their `proposer` BLS private key. A `SignedDelegation` binds the `proposer` public key to the `delegate` public key and a `slasher` contract until after the `valid_until` slot elapses. During this time, the `delegate` can submit constraints to the relay on behalf of the `proposer`.

    - `proposer`: The BLS public key of the proposer who is delegating preconfirmation rights.
    - `delegate`: The BLS public key of the gateway who is receiving preconfirmation rights.
    - `slasher`: The address of a slasher contract containing a slashing function to penalize a proposer who has violated constraints (i.e., reneged on a preconf).
    - `valid_until`: slot number (inclusive) that a `SignedDelegation` is considered valid until
    - `metadata`: Additional opaque byte array reserved for and interpreted by the slashing function and/or the gateway (e.g., gas limit, blob limit, chain id, preconf type)

    While the Constraints API aims to be unopinionated about how slasher contracts are implemented, it's assumed that `SignedDelegation` messages are part of the evidence used to slash a proposer.
---

### Endpoint: `/constraints/v0/relay/constraints`

Endpoint for submitting a batch of constraints to the relay. The constraints are expected to be signed by a `delegate` BLS private key, whose corresponding public key is specified in a `SignedDelegation` message.

- Method: `POST`
- Response: Empty
- Headers:
    - `Content-Type: application/json`
- Body: JSON object of type `SignedConstraints[]`

**Schema**

```python
# A signed "bundle" of constraints.
class SignedConstraints(Container):
    message: ConstraintsMessage
    signature: BLSSignature

# A "bundle" of constraints for a specific slot.
class ConstraintsMessage(Container):
    delegate: BLSPubkey
    slot: uint64
    contraints: List[Constraint]

# A contraint for a transactions
class Constraint(Container):
    commitmentType: uint64
    payload: Bytes
```

- **Description**
    
    For each `Preconfirmation` the delegate signs, they will need to create a matching `Constraint`. Collectively, a `SignedConstraints` message is posted to the relay. 
    
    - `commitmentType`: unsigned 64-bit number between `0` and `0xffffffffffffffff` that represents the type of the proposer commitment
    - `payload`: opaque byte array whose interpretation is dependent on the `commitmentType`

    Particularly each `commitmentType` would have a corresponding spec that defines:
    - a schema for a `Preconfirmation` and `SignedPreconfirmation` message
    - how a `Constraint.payload` is interpreted
    - how a `Constraint.payload` is created given a `SignedPreconfirmation`
    - the ordering of `constraints[]`
    - how to build a valid block given a `ConstraintsMessage`
    - how to generate proofs of constraint validity
    - how to verify proofs of constraint validity
---

### Endpoint: **`/constraints/v0/relay/header_with_proofs/{slot}/{parent_hash}/{pubkey}`**

Endpoint for requesting a builder bid with constraint proofs.

- **Method:** `GET`
- **Response:** `VersionedSignedBuilderBidWithProofs`
- **Parameters:**
    - `slot`: `string` (regex `[0-9]+`)
    - `parent_hash`: `string` (regex `0x[a-fA-F0-9]+`)
    - `pubkey`: `string` (regex `0x[a-fA-F0-9]+`)
- **Body:** Empty

**Schema**

```python
class VersionedSignedBuilderBidWithProofs:
    ... # All regular fields from VersionedSignedBuilderBid, additionally
    proofs: ConstraintProofs

class ConstraintProofs(Container):
  commitmentTypes: List[uint64, MAX_CONSTRAINTS_PER_SLOT]
  payloads: List[Bytes, MAX_CONSTRAINTS_PER_SLOT]
```

- **Description**
    
    The `VersionedSignedBuilderBidWithProofs` schema extends `VersionedSignedBuilderBid` from the [original builder specs](https://ethereum.github.io/builder-specs/#/Builder/getHeader) to include proofs of constraint validity. Without leaking the block's contents, a Proposer can verify that the block satisfies the constraints by checking the `proofs` against the block header. To support a wide range of constraint types with different proving requirements, `ConstraintProofs` is left open-ended to allow for future flexibility.

    - `commitmentTypes`: list of unsigned 64-bit numbers between `0` and `0xffffffffffffffff` that represents the type of the proposer commitment (not required to be homogenous)
    - `payloads`: list of opaque byte arrays whose interpretation is dependent on the `commitmentTypes`

- **Requirements**: 
    - each `commitmentType` has a spec that defines how builders can generate `proofs` for their block
    - each `commitmentType` has a spec that defines how relays and proposers can verify `proofs`
    - When serializing, the `proofs` field must be present in `data`, at the same level of `signature` and `message`. See the example below.
    - The length of `commitmentTypes` and `payloads` must be the same

- **Example Payload**
    ```python
    # commitmentType = 0x00
    class InclusionProof(Container):
        tx_hash: Bytes32
        index: uint64
        merkle_hashes: List[Bytes32]
    
    # example inclusion proofs
    proof_0 = InclusionProof(
        tx_hash="0xcf8e...", index=7, merkle_hashes=["0xa7bc...", "0xd912...", ...]
    ).ssz_encode()

    proof_1 = InclusionProof(
        tx_hash="0x9fbb...", index=9, merkle_hashes=["0xeeab...", "0x1a2c...", ...]
    ).ssz_encode()

    # example envelope for mulitple proofs
    proofs = ConstraintProofs(
        commitmentTypes=[0x00, 0x00],
        payloads=[
            proof_0,
            proof_1,
        ],
    )

    ```

- **Example Response**
    ```json
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
                "commitmentTypes": [0x00, 0x00],
                "payloads": ["0x5097...", "0x932587..."]
            },
            "signature": "0x1b66ac1fb663c9bc59509846d6ec05345bd908eda73e670af888da41af171505cc411d61252fb6cb3fa0017b679f8bb2305b26a285fa2737f175668d0dff91cc1b66ac1fb663c9bc59509846d6ec05345bd908eda73e670af888da41af171505"
        }
    }
    ```
---
    
### Endpoint: `/constraints/v0/relay/delegations?slot={slot}`

Return the active delegations for the proposer of this slot, if they exist.

- **Method:** `GET`
- **Response:** `SignedDelegation[]`
- **Parameters:**
    - `slot`: `string` (regex `[0-9]+`)
- **Body:** Empty

- **Example Response**
    ```json
    [
        {
            "message": {
                "proposer": "0x93247f2209abcacf57b75a51dafae777f9dd38bc7053d1af526f220a7489a6d3a2753e5f3e8b1cfe39b56f43611df74a",
                "delegate": "0x84e47f2209abcacf57b75a51dafae777f9dd38bc7053d1af526f220a7489a6d3a2753e5f3e8b1cfe39b56f43611df74b",
                "slasher": "0xabcf8e0d4e9587369b2301d0790347320302cc09",
                "valid_until": "12345",
                "metadata": "0xe9587369b2301d0790347320302cc069b2301d0790347320302cc0943d5a1884560367e8208d920f2e9587369b2301de9587369b2301d0790347320302cc0"
            },
            "signature": "0x1b66ac1fb663c9bc59509846d6ec05345bd908eda73e670af888da41af171505cc411d61252fb6cb3fa0017b679f8bb2305b26a285fa2737f175668d0dff91cc1b66ac1fb663c9bc59509846d6ec05345bd908eda73e670af888da41af171505"
        }
    ]
    ```

- **Description**

    The Relay should only return signed delegations that were signed by the proposer.

---

### Endpoint: `/constraints/v0/relay/constraints?slot={slot}`

Returns all signed constraints for a given slot, if they exist.

- **Method:** `GET`
- **Response:** `SignedConstraints[]`
- **Parameters:**
    - `slot`: `string` (regex `[0-9]+`)
- **Body:** Empty

- **Example Response**
    ```json
    [
        {
            "message": {
                "delegate": "0x93247f2209abcacf57b75a51dafae777f9dd38bc7053d1af526f220a7489a6d3a2753e5f3e8b1cfe39b56f43611df74a",
                "slot": "12345",
                "constraints": [
                    {
                        "commitmentType": "0x00",
                        "payload": "0x301d0790347320302cc0943d5a1884560367e8208d920f2e9587369b2301de9587369b2301d0790347320302cc0"
                    },
                    {
                        "commitmentType": "0x01",
                        "payload": "0x367e8208d920f2e9587369b2301de9587369b2301d0790347320302cc0301d0790347320302cc0943d5a1884560367e8208d920f2e958"
                    }
                ]
            },
            "signature": "0x1b66ac1fb663c9bc59509846d6ec05345bd908eda73e670af888da41af171505cc411d61252fb6cb3fa0017b679f8bb2305b26a285fa2737f175668d0dff91cc1b66ac1fb663c9bc59509846d6ec05345bd908eda73e670af888da41af171505"
        }
    ]
    ```

- **Description**

    The Relay should only return signed constraints that were signed by the proposer or a gateway that was delegated to by the proposer.

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
