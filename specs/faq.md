# FAQ
### Is this spec limited to preconfs?
No! The spec is intentionally designed to support generic proposer commitments. Preconfirmations are just one flavor of this that we're just scratching the surface on today.

### What do delegations do?
`Delegation` messages do not directly commit a proposer to any protocol’s slashing conditions. Instead, they inform Relays and Gateways which keys are authorized to issue constraints and commitments on the proposer’s behalf.

### What if I don’t want to delegate to a Gateway?
Delegation is entirely optional. You can run your own Gateway, self-delegate, and remain fully compatible with the spec.

### Why not support slot ranges in delegations?
Slot ranges introduce vendor lock-in and timing-based cancellation games that increase proposer risk. This design deliberately avoids those pitfalls. Long-term delegations (e.g., for a year) offer minimal benefits over just-in-time (JIT) delegations given how frequent L1 validators propose blocks.

### Why is the `GET /constraints` a gated endpoint?
If constraints were public, a proposer could bypass Gateways and Builders using their constraints to build blocks themselves without compensation. This is especially problematic for non-contentious features like inclusion preconfirmations. To avoid this, Gateways specify who can view the constraints via the `SignedConstraints.message.receivers` field.

### Doesn’t this mean proposers can’t always verify proofs?
Correct. That’s why the spec reuses the existing `GET /header` endpoint instead of requiring proposers to verify proofs. Proposers already trust the Relay, so it’s more efficient to have the Relay handle verification. Plus, simpler methods like block simulation can often replace expensive proofs (e.g., ZKPs).

### Does trusting the Relay mean it can arbitrarily slash?
Not exactly. Preconfirmations define their own slashing logic outside of this spec. The Relay’s role is to act as a data availability (DA) layer for `SignedConstraints`, and optionally publish them on-chain to aid fault attribution—not to enforce slashing unilaterally. See the [fault attribution guidelines](./fault-attribution.md) more more details.

### Can proof verification be made optional?
Yes. Proposers can use the extra bytes field in `Delegation.metadata` to request that Builders bypass proof generation, as described [here](fault-attribution.md#optimistic-relaying).

### Would consolidation between Gateways, Builders, and Relays affect fault attribution?
Even if some entities consolidate, there will still be multiple Gateways and Relays in the ecosystem. If one Relay is unwilling to slash its affiliated Gateway, another Relay can still do so.

### Is DVT (Distributed Validator Technology) supported?
Yes. `Delegation` messages are intentionally signed with BLS keys which are compatible with DVT. 