# Savitri Network: Engineering Specification Manifesto

## Architecture is Implementation

We reject probabilistic consensus in favor of BFT finality with explicit synchrony assumptions. Every architectural decision is an implementation trade-off. The choice between fixed-point and floating-point arithmetic is not merely performance optimization; it is a statement about system determinism under adversarial conditions. The selection of data structures determines the actual throughput under contention. The design of the batching mechanism encodes our values of efficiency and scalability.

## BFT Finality with Partial Synchrony

A blockchain system achieves deterministic finality when BFT certificates are issued under partial synchrony assumptions. Formally:

$$\text{Finality} = \text{Certificate} \quad \text{where} \quad |\text{votes}| \geq 2f + 1 \land \text{GST} < \Delta$$

Where $f$ is the number of Byzantine faults tolerated, $\text{GST}$ is Global Stabilization Time, and $\Delta$ is the observed network upper bound after GST. This requires:

- **Partial synchrony**: Network behaves asynchronously until GST, then synchronously
- **Byzantine threshold**: $n \geq 3f + 1$ validators
- **Timeout adaptation**: Dynamic timeout based on observed network conditions

This contrasts with probabilistic systems where finality is never absolute.

## Fairness as Sybil-Resistant Scoring

Fairness in PoU scoring is defined as deterministic fixed-point calculation with sybil resistance:

$$\text{PoU}_{score} = \frac{w_U \cdot U + w_L \cdot L + w_I \cdot I + w_R \cdot R}{w_U + w_L + w_I + w_R}$$

where weights are implemented as basis points: $w_U = 3000$ (30%), $w_L = 1000$ (10%), $w_I = 2500$ (25%), $w_R = 2000$ (20%), with SCALE = 1,000,000 for 6-decimal precision.

**Sybil Resistance Mechanism:**
- **Identity cost**: Non-economic identity binding through cryptographic commitment (detailed in separate identity spec)
- **Measurement oracle**: On-chain telemetry with challenge-response verification
- **Score decay**: Temporal decay prevents permanent advantage accumulation

**MEV Mitigation:**
$$\text{MEV}_{opportunity} \leq \epsilon \quad \text{where} \quad \epsilon = \text{network\_delay} \times \text{propagation\_variance}$$

Scheduler ensures transaction ordering through cryptographic commitment, not timing advantages.

## Formal Verification Requirements

Mathematical proof is required for critical properties. Every invariant must be specified and verified:

- **Safety**: $\square \neg \text{equivocation}$ through evidence collection and slashing
- **Liveness**: $\diamond \text{certificate}$ through timeout adaptation and leader rotation
- **Fairness**: $\square (proposal_i \rightarrow \diamond response_i)$ through deterministic slot assignment

**Verification Stack:**
- **Model checking**: TLA+ for consensus algorithm specification
- **Theorem proving**: Rust type system for memory safety invariants
- **Property testing**: QuickCheck for network layer properties

## Fixed-Point Arithmetic for Determinism

Deterministic arithmetic is required for cross-platform consensus. Fixed-point with SCALE = 1,000,000 eliminates platform divergence:

$$\forall x: \text{FixedPoint}, \quad 0 \leq x.value \leq \text{SCALE}$$

**Implementation Invariants:**
- **Overflow protection**: All operations use checked arithmetic
- **Deterministic rounding**: Consistent rounding rules across platforms
- **Performance**: $C_{fixed\_point} = C_{integer} + C_{verification}$ where $C_{verification} = 0$ at runtime

## Adaptive Systems with WCET Guarantees

We pursue performance through adaptive optimization with worst-case execution time guarantees for consensus critical path:

$$\forall f: \text{critical\_function}, \quad \text{WCET}(f) \leq B_f \cdot \text{adaptation\_factor}$$

**Adaptation Algorithm:**
$$\text{batch\_size}_{new} = \text{batch\_size}_{current} \times \begin{cases} 
1.2 & \text{if } \text{P99}_{latency} < \text{target} \land \text{CPU}_{util} < 0.7 \\
0.8 & \text{if } \text{P99}_{latency} > 1.2 \times \text{target} \\
1.0 & \text{otherwise}
\end{cases}$$

**Real-time Constraints:**
- **Scheduling**: EDF (Earliest Deadline First) with priority inheritance
- **Memory**: Lock-free data structures with epoch-based reclamation
- **Network**: io_uring with kernel bypass for high-priority messages

## Hardware-Aware Performance Engineering

Multi-core scaling requires explicit hardware management. All shared data structures account for:

$$\forall s: \text{shared\_struct}, \quad \text{align}(s) = 64 \text{ bytes} \land \text{false\_sharing\_free}$$

**NUMA Optimization:**
- **Memory allocation**: Thread-local allocators with NUMA-aware placement
- **CPU affinity**: Core pinning for critical path operations
- **Cache hierarchy**: Explicit prefetch for predictable access patterns

**Performance Model:**
$$T_{system} = \min(T_{cpu}, T_{memory}, T_{network}, T_{storage})$$

where each component is measured with P99.9 latency bounds under adversarial load.

## Measured Performance Properties

System performance is characterized by observable metrics, not theoretical bounds:

**Throughput Model:**
$$T = \frac{\text{transactions\_processed}}{\text{time\_window}} \times \text{efficiency\_factor}$$

**Efficiency Factor:**
$$\text{efficiency} = \frac{\text{actual\_throughput}}{\text{theoretical\_peak}} \times \frac{\text{target\_batch\_size}}{\text{actual\_batch\_size}}$$

**State Growth Management:**
$$\text{state\_size}(t) = \text{state}_0 + \int_0^t (\text{tx\_rate} - \text{prune\_rate}) \, dt$$

**Determinism Guarantees:**
- **Fixed-point arithmetic**: Eliminates floating-point divergence
- **Deterministic ordering**: Cryptographic commitment prevents timing games
- **Bounded execution**: WCET analysis for critical path functions

## Threat Model

**Adversarial Capabilities:**
- **Network**: Can delay messages up to $\Delta$ after GST
- **Computation**: Can control up to $f$ Byzantine validators
- **Timing**: Can influence scheduling through transaction ordering
- **State**: Can attempt state growth attacks

**Defenses:**
- **Slashing**: Economic penalty for provable misbehavior
- **Rate limiting**: Transaction admission control per identity
- **State pruning**: Periodic compaction with Merkle proofs
- **Timeout adaptation**: Dynamic adjustment based on observed conditions

## Consensus Algorithm Specification

**Roles:**
- **Proposer**: Selected by deterministic slot scheduler
- **Validator**: Votes on proposals, collects evidence
- **Executor**: Processes committed blocks deterministically

**Algorithm Sketch:**
1. **Slot Assignment**: $slot(t) = \text{hash}(t, \text{validator\_set}) \mod n$
2. **Proposal**: Proposer creates block with cryptographic commitment
3. **Voting**: Validators broadcast votes with timeout $\tau = 2\Delta$
4. **Certificate**: Block committed when $2f+1$ votes collected
5. **Execution**: Deterministic state transition with fixed-point arithmetic

## What is Impossible by Design

**Guarantees:**
- **No finality reversals**: BFT certificates are irreversible
- **No floating-point divergence**: All arithmetic is deterministic
- **No timing-based MEV**: Ordering through cryptographic commitment
- **No unbounded state growth**: Pruning with Merkle proofs

**Trade-offs:**
- **Throughput vs latency**: Adaptive batching optimizes for P99
- **Decentralization vs performance**: Fixed validator set with rotation
- **Complexity vs security**: Formal verification for critical components

## Conclusion

Engineering reality requires explicit assumptions and measurable guarantees. We reject theoretical purity that cannot be implemented. We embrace complexity only when verified through formal methods.

Architecture is specification. Determinism is law. Performance is measurable.

---

*This manifesto represents the engineering specifications required for peer review and audit. It is a living technical specification, evolving with implementation details and formal verification results.*
