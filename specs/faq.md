# FAQ
### Is this spec limited to preconfs?
No! The spec is intentionally designed to support generic proposer commitments. Preconfirmations are just one flavor of this that we're just scratching the surface on today.

### What is the purpose of delegation?
Ethereum validators are the lifeblood of the network. Delegation, introduced through proposer-builder separation, was designed to keep proposers "dumb" and outsource complex responsibilities to external experts. In the same spirit, the constraints API allows validators to delegate the creation of sophisticated constraints to specialized third parties.

### What do delegations do?
`Delegation` messages do not directly commit a proposer to any protocol’s slashing conditions. Instead, they inform Relays and Gateways which keys are authorized to issue constraints and commitments on the proposer’s behalf.

### What if I don’t want to delegate to a Gateway?
Delegation is entirely optional. You can run your own Gateway, self-delegate, and remain fully compatible with the spec.

### Why not support slot ranges in delegations?
Slot ranges introduce vendor lock-in and timing-based cancellation games that increase proposer risk. This design deliberately avoids those pitfalls. Long-term delegations (e.g., for a year) offer minimal benefits over just-in-time (JIT) delegations given how frequent L1 validators propose blocks.

### Why is the `GET /constraints` a gated endpoint?
If constraints were public, a proposer could bypass Gateways and Builders using their constraints to build blocks themselves without compensation. This is especially problematic for non-contentious features like inclusion preconfirmations. To avoid this, Gateways specify who can view the constraints via the `SignedConstraints.message.receivers` field. If the `receivers` field is left unset, anyone can view the `SignedConstraints`. Of course [delegation is entirely optional](#what-if-i-dont-want-to-delegate-to-a-gateway).

### Doesn’t this mean proposers can’t always verify proofs?
Correct. That’s why the spec reuses the existing `GET /header` endpoint instead of requiring proposers to verify proofs. Proposers already trust the Relay, so it’s more efficient to have the Relay handle verification. Plus, simpler methods like block simulation can often replace expensive proofs (e.g., ZKPs).

### Does trusting the Relay mean it can arbitrarily slash?
Not exactly. Preconfirmations define their own slashing logic outside of this spec. The Relay’s role is to act as a data availability (DA) layer for `SignedConstraints`, and optionally publish them on-chain to aid fault attribution—not to enforce slashing unilaterally. See the [fault attribution guidelines](./fault-attribution.md) more more details.

### Can proof verification be made optional?
Yes. Proposers can use the extra bytes field in `Delegation.metadata` to request that Builders bypass proof generation, as described [here](fault-attribution.md#optimistic-relaying).

### Would consolidation between Gateways, Builders, and Relays affect fault attribution?
Even if some entities consolidate, there will still be multiple Gateways and Relays in the ecosystem. If one Relay is unwilling to slash its affiliated Gateway, another Relay can still do so.

### Does this increase the barrier to entry or reliance on Relays?
No. This approach does not materially change the role of the Relay or increase reliance on it. The operational, security, and reputation requirements remain consistent with what has always been expected for relays. Any team capable of running a relay today should be capable of supporting this model without additional overhead.

### Will there be open-source implementations of a Relay and Gateway that support this spec?
Yes. Multiple teams are actively building implementations, and some have already open-sourced their work. Fabric and all related components are also fully open-source.

###Does this system introduce a single point of failure? What are the fallback mechanisms?
No. The trust assumptions around Relays remain largely the same as those in existing PBS designs. The main new component is the Gateway, which acts as a trusted entity—but there are important guardrails in place:
- Self-delegation is always possible: Proposers can act as their own Gateway, maintaining full control and avoiding external dependencies.
- Gateways are expected to be collateralized: This provides economic accountability.
- Fault attribution mechanisms exist: The spec includes tools for mapping failures from proposers to Gateways, enabling slashing or penalties when Gateways misbehave. See the fault attribution documentation for details.

### Is DVT (Distributed Validator Technology) supported?
Yes. `Delegation` messages are intentionally signed with BLS keys which are compatible with DVT. 