# Fault Attribution

## Table of Contents

## Introduction
This document outlines a recommended approach for handling fault attribution in proposer commitment protocols that adopt the Constraints API and the Universal Registry Contract (URC). As block production increasingly involves multiple parties (Gateways, Builders, Relays, and Proposers), some of which delegated, it becomes essential to clearly identify which actor is responsible in the event of a safety fault.

The goal of this proposal is to offer a practical pattern for fault resolution. In particular, it provides a mechanism for distinguishing between proposer, builder, relay and gateway faults, enabling implementations to apply slashing with greater precision.

This approach reflects ongoing discussions across teams building Gateways, Relays, and preconf protocols, and is intended as a shared reference point for implementations.

- *Importantly, it should be noted that this approach remains compatible with self-delegation, where a Proposer operates their own Gateway.*

- *This document is not part of the base spec, rather is a reference to help better understand fault-attribution in the context of the Constraints API.*

## Background
The Constraints API was originally based off of a narrow, focused goal: to enable L1 inclusion preconfirmations issued directly by the L1 proposer. In this non-delegated model, the proposer created commitments and constraints themselves, retaining full visibility and enabling trustless verification via proofs against the block header.

Since then, the role of the Constraints API has expanded beyond simple inclusion preconfs to encompass generic proposer commitments. As a result, teams have increasingly leaned toward delegation as the default model, where Gateways act on behalf of the proposer to issue commitments and constraints.

While delegation helps reduce proposer sophistication, it introduces ambiguity in fault attribution. If a safety fault occurs, who is actually responsible? Should the proposer be slashed? The Gateway? Can we even tell?

This document assumes familiarity with the end-to-end block building flow, including how commitments and constraints are generated, disseminated, and enforced. For those details, readers should refer to the [proposer spec](proposer.md), [gateway spec](gateway.md), and [builder spec](builder.md).

## Problem
The original security model of the Constraints API assumes the proposer has complete visibility into all commitments and constraints they issue. This allowed the proposer to verify that a Builder's block satisfies their constraints without leaking the block's contents by checking proofs against the block header. In the non-delegated model, this verification is both trustless and sound.

However, this model breaks down under delegation, where a Gateway issues commitments and constraints on behalf of the proposer. In this case, the proposer may no longer have full knowledge of all commitments made, nor visibility into the complete set of constraints disseminated to Builders. Similarly, Builders are unaware if the constraints they receive cover all commitments that were made.

The following example illustrates the failure case:

1.	The proposer delegates to a Gateway.
2.	The Gateway signs two commitments for L1 inclusion preconfirmations: `TX1` and `TX2`.
3.	The Gateway only posts the constraint for `TX1` to the Relay, withholding the constraint for `TX2`.
4.	The Builder receives the constraints, includes `TX1` in the block, and generates a valid Merkle inclusion proof.
5.	The proposer, unaware of the missing constraint, verifies the proof and proposes the block.
6.	The proposer is slashed for omitting `TX2`, despite acting honestly and according to protocol.

This reveals a critical flaw: in the delegated setting, valid proofs can still be unsound if the Gateway selectively withholds constraints. Builders cannot be held accountable if they never receive all relevant constraints, and proposers are exposed to slashing despite having no way to detect the omission.

As a result, proofs no longer offer trustless guarantees. Their soundness becomes entirely dependent on the honesty of the Gateway, eroding the foundational security model of the protocol and highlighting the need for a revised approach to fault attribution.

## Implemented Solution
To resolve the ambiguity introduced by delegation, we propose a fault attribution mechanism based on the fact that **Relays are already trusted entities in PBS today**. This approach works in both the delegated and non-delegated case, minimizes latency, and enables slashing decisions to be made with clear attribution.

### Assumptions
The core idea is simple: the Relay plays two essential roles in the pipeline - as a filter and as a data availability (DA) layer for `SignedConstraints` messages.

#### 1. Filter
The Relay is assumed to filter out invalid blocks. Specifically, it must reject any block for which it cannot verify that the Builder met the required constraints. This can involve:
- **Optimistic relaying**, where the Builder commits to a `SignedConstraints` message and the Relay checks the signature.
- **Pessimistic relaying**, where the Relay must verify constraint satisfaction through block simulation, Merkle inclusion proofs, TEEs, ZKPs, or other means.

#### 2. Data Availability Layer
The Relay is assumed to act as a DA layer for `SignedConstraints`:
- It accepts and saves `SignedConstraints` messages from Gateways.
- It enforces a cutoff time for accepting `SignedConstraints` per slot. This ensures Builders can rely on a consistent set of constraints when building blocks and prevents Gateways from submitting constraints after the fact to avoid fault.
- After the target slot has elapsed, the Relay should make the `SignedConstraints` available via the Relay Data API, irrespective of the `SignedConstraints.message.receivers` field.

## Fault Attribution
Understanding who is at fault requires knowing the mapping from `Commitment` to `Constraint`, which must be publicly accessible. From this, slashing conditions are as follows:
- **Proposer**: Can only be slashed if they delegate and then self-build (i.e., bypass Builder/Relay constraints).
- **Gateway**: Can only be slashed if their `SignedConstraints` is incomplete, i.e., their `SignedConstraints` is missing a `Constraint` for a `Commitment` they issued.
- **Builder** (Optimistic case only): Can be slashed if they build an **invalid** block on top of **complete** `SignedConstraints`.
- **Relay** (Pessimistic case only): Can be (socially) slashed if they relay an **invalid** block built on top of **complete** `SignedConstraints`.

Fault attribution is very dependent on the *completeness* of `SignedConstraints`, hence the importance of the Relay to make them available and only accept new `SignedConstraints` up to a certain cutoff time.

## Potential modes of relaying
The following modes serve as implementation guidance rather than prescriptive rules. From the perspective of the spec, these modes are indistinguishable—each simply represents a different strategy for leveraging the `ConstraintProofs` field. This is similar to how “optimistic relaying” emerged as an optimization within PBS: adopted in practice but not enshrined in the spec.

### Optimistic relaying
This mode optimizes for performance over safety. By omitting heavy proof generation and verification, more time is available to build the block, potentially increasing the value of the block. However, this introduces the possibility that an invalid block is relayed to the Proposer, in which case the Builder should be penalized. The Proposer should decide whether to enable optimistic relaying, i.e., in their `Delegation.metadata` field.
1.	Gateway posts `SignedConstraints` to the Relay.
2.	Builder constructs a block that satisfies all the constraints they received. In the `VersionedSubmitBlockRequestWithProofs.proofs` field, the Builder includes their signature over the `SignedContraints` object, effectively attesting that the block complies with the constraints they were given.
3.	Relay includes the block in their auction if the Builder's signature is valid over the `SignedConstraints`.
4.	Proposer receives the signed header and proposes the block.

### Pessimistic relaying
Conversely, this mode optimizes for safety over performance. Builders are required to generate proofs of constraint satisfaction which comes with the associated compute costs. It is advised that this is the default setting that needs to be overriden by Proposers, i.e., in their `Delegation.metadata` field.
1.	Gateway posts `SignedConstraints` to the Relay.
2.	Builder constructs a block that satisfies all the constraints they received. The Builder includes proofs of constraint satisfaction in the `VersionedSubmitBlockRequestWithProofs.proofs` field.
3.	Relay includes the block in their auction if the Builder's proofs are valid over the `SignedConstraints`.
4.	Proposer receives the signed header and proposes the block.

### Mapping faults
The following table summarizes the different outcomes. Note for brevity the table omits when a proposer self-builds, in which case they are liable if there is a safety fault.
| Relaying Type | SignedConstraints are complete | Block satisfies SignedConstraints | Block relayed to proposer | Outcome         |
|---------------|--------------------------------------|-----------------------------------|----------------------------|------------------|
| Optimistic    | No                                   | No                                | No                         | Slot missed      |
| Optimistic    | No                                   | No                                | Yes                        | Gateway slashed  |
| Optimistic    | No                                   | Yes                               | No                         | Slot missed      |
| Optimistic    | No                                   | Yes                               | Yes                        | Gateway slashed  |
| Optimistic    | Yes                                  | No                                | No                         | Slot missed      |
| Optimistic    | Yes                                  | No                                | Yes                        | Builder slashed  |
| Optimistic    | Yes                                  | Yes                               | No                         | Slot missed      |
| Optimistic    | Yes                                  | Yes                               | Yes                        | Happy case       |
| Pessimistic   | No                                   | No                                | No                         | Slot missed      |
| Pessimistic   | No                                   | No                                | Yes                        | Gateway slashed  |
| Pessimistic   | No                                   | Yes                               | No                         | Slot missed      |
| Pessimistic   | No                                   | Yes                               | Yes                        | Gateway slashed  |
| Pessimistic   | Yes                                  | No                                | No                         | Slot missed      |
| Pessimistic   | Yes                                  | No                                | Yes                        | Relay slashed    |
| Pessimistic   | Yes                                  | Yes                               | No                         | Slot missed      |
| Pessimistic   | Yes                                  | Yes                               | Yes                        | Happy case       |