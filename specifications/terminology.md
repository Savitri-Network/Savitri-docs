# Technical Terminology

## Core Blockchain Concepts

**Block**
- Structured data container containing transactions, consensus data, and state commitments
- Implemented in `src/p2p/messages.rs` as `Block` struct with `BlockHeader` and transactions
- Block hash: SHA512 with domain separator "BLK"

**Monolith**
- Aggregated block structure containing multiple block headers with ZKP proof
- Defined in `src/core/monolith.rs` with `MonolithHeader` and verification logic
- Enables light node verification without full state download

**Account**
- State entity with balance, nonce, and optional contract code
- Structure in `src/core/types.rs`: `balance: u128`, `nonce: u64`, `code_hash: Option<[u8; 64]>`
- Storage key: 32-byte address derived from public key

**Transaction**
- State transition instruction signed by sender
- `SignedTx` in `src/core/types.rs` with signature, nonce, fee, and call data
- Transaction hash: Blake3 of serialized transaction

## Consensus Terminology

**Proof of Unity (PoU)**
- Custom consensus scoring algorithm combining multiple metrics
- Formula: $Score = w_U \cdot U + w_L \cdot L + w_I \cdot I + w_R \cdot R + w_P \cdot P$
- Components: Availability (U), Latency (L), Integrity (I), Reputation (R), Participation (P)

**Slot**
- Time interval for consensus participation, typically 500ms
- Managed by `src/core/slot_scheduler.rs` with deterministic leader election
- Slot roles: Leader, Follower, Observer

**Validator**
- Node participating in consensus with active bond
- Bond requirement: 1,000,000 tokens (1M tokens minimum)
- Slashing: 50% bond penalty for equivocation

**Certificate**
- Cryptographic proof of block finality with $2f+1$ validator signatures
- Structure in `src/consensus/types.rs`: block_hash, signatures, round, height
- Finality guarantee: Once certificate issued, block cannot be reverted

## Execution Terminology

**ExecutionDispatcher**
- Transaction scheduling and execution engine with SIMD optimization
- Located in `src/executor/dispatcher.rs` with adaptive weight scheduling
- Supports batch processing with score caching

**SIMD Optimization**
- Single Instruction Multiple Data vectorization for transaction scoring
- Uses AVX2 on x86_64, NEON on ARM with runtime detection
- Threshold: 32 transactions minimum for SIMD benefits

**Score Cache**
- Thread-safe caching system for transaction scores across batches
- Implemented in `src/executor/score_cache.rs` with LRU eviction and TTL
- Cache hit rate target: ≥80% for typical workloads

**Adaptive Weights**
- Dynamic scheduling parameters based on mempool conditions
- Fee-to-block-space ratio adjustment with feedback mechanism
- Update interval: Configurable, default 100 blocks

## Network Terminology

**Gossipsub**
- Message propagation protocol using libp2p
- Topic-based message distribution with mesh topology
- Implemented in `src/p2p/gossipsub.rs`

**Peer Reputation**
- Scoring system for peer behavior assessment
- Factors: Message forwarding, response time, misbehavior detection
- Stored in `src/p2p/reputation.rs`

**Message Types**
- `ConsensusProposal`: Block proposal with consensus data
- `ConsensusVote`: Validator vote on proposal
- `ConsensusCertificate`: Finality certificate
- `TransactionGossip`: Transaction propagation

## Storage Terminology

**Column Family**
- RocksDB data partition for efficient access patterns
- CF_ACCOUNTS: Account state data
- CF_TRANSACTIONS: Transaction history
- CF_BLOCKS: Block headers and metadata
- CF_CONSENSUS: Consensus state and certificates
- CF_BONDS: Validator bond information

**State Root**
- Cryptographic hash of entire account state
- Computed using Merkle tree of account storage
- 64-byte hash using Blake3

**Monolith Storage**
- Aggregated storage for historical block data
- Headers commit: Cryptographic commitment to block headers
- State commit: Cryptographic commitment to account state

## Cryptographic Terminology

**Ed25519**
- Digital signature algorithm for transaction signing
- Batch verification support for performance
- Key size: 32 bytes private, 32 bytes public

**Blake3**
- High-performance hash function for general hashing
- Output size: 256 bits (32 bytes)
- Used for transaction hashes and state roots

**SHA512**
- Cryptographic hash for block hashing
- Output size: 512 bits (64 bytes)
- Domain separator: "BLK" for block hash computation

**Zero-Knowledge Proof**
- Cryptographic proof system for monolith verification
- PLONK-based implementation in `src/zkp/`
- Enables verification without revealing underlying data

## Smart Contract Terminology

**BaseContract**
- Standard contract interface with reserved storage slots
- Slots 0-99 reserved for BaseContract data
- Implemented in `src/contracts/base.rs`

**Function Selector**
- 4-byte identifier for contract functions
- Computed as keccak256(function_signature)[0:4]
- Used for routing contract calls

**Gas Meter**
- Resource usage measurement for contract execution
- Implemented in `src/contracts/gas.rs`
- Prevents infinite loops and resource exhaustion

## Performance Terminology

**TPS**
- Transactions Per Second, primary throughput metric
- Target: 10,000+ with SIMD optimization
- Measured under sustained load conditions

**Latency**
- Time from transaction submission to finality
- Target: ≤1.5 seconds (2-3 blocks)
- Components: Network, consensus, execution

**Determinism**
- Property of producing identical results across different environments
- Critical for blockchain consensus safety
- SIMD vs scalar divergence must be < $10^{-10}$

## Security Terminology

**Slashing**
- Penalty mechanism for validator misbehavior
- 50% bond forfeiture for equivocation
- Implemented in `src/consensus/evidence.rs`

**Equivocation**
- Signing conflicting blocks or votes in same round
- Detection through evidence collection
- Automatic slashing upon proof submission

**Finality**
- Irreversible state transition guarantee
- Achieved through certificate generation
- BFT safety property under $f$ Byzantine faults

## Data Structure Formats

**Address**
- 32-byte account identifier
- Derived from public key: Blake3(public_key)[0:32]
- Hexadecimal representation with 0x prefix

**Hash**
- Cryptographic digest of data
- Block hash: 64 bytes (SHA512)
- Transaction hash: 32 bytes (Blake3)
- State root: 64 bytes (Blake3)

**Signature**
- Ed25519 digital signature
- Size: 64 bytes
- Verification: Public key + message + signature

**Nonce**
- Account transaction counter
- Type: u64 (0 to $2^{64}-1$)
- Prevents replay attacks

## Configuration Parameters

**SIMD_THRESHOLD**
- Minimum transaction count for SIMD optimization
- Value: 32 transactions
- Below threshold: scalar processing

**MAX_CALL_DEPTH**
- Maximum nested contract call depth
- Value: 64 levels
- Prevents stack overflow

**TTL_CACHE**
- Time-to-live for score cache entries
- Value: 100 blocks
- Automatic expiration of stale entries

This terminology provides precise definitions for all technical concepts used in Savitri Network implementation and documentation.
