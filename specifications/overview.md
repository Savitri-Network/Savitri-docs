# Technical Specification Overview

## System Architecture

Savitri Network is an enterprise-grade Layer 1 blockchain implementing a custom execution environment with BFT consensus, SIMD-optimized transaction processing, and adaptive weight scheduling.

### Core Components (Workspace Structure)

**Consensus Layer** (`savitri-consensus/`)
- `src/consensus/` - BFT consensus engine with Proof of Unity (PoU) scoring
- `src/pou/` - PoU score calculation with fixed-point arithmetic
- `src/lib.rs` - Main consensus library entry point
- Committee size: 4-256 validators with bond requirements (1M tokens minimum)
- Features: BFT consensus, leader rotation, evidence collection, cross-shard finality

**Execution Layer** (`src/executor/`)
- `src/executor/dispatcher.rs` - ExecutionDispatcher with adaptive weights
- `src/executor/score_cache.rs` - Thread-safe caching for cross-batch optimization
- `src/executor/fixed_point_simd.rs` - SIMD-optimized transaction scoring
- SIMD optimization: AVX2/FMA for x86_64, NEON for ARM

**Storage Layer** (`savitri-storage/`)
- RocksDB-based persistence with column families
- `src/storage/` - Storage management with monolith support
- Column families: CF_ACCOUNTS, CF_TRANSACTIONS, CF_BLOCKS, CF_CONSENSUS, CF_BONDS
- State root computation and overlay management

**Network Layer** (`savitri-p2p/`)
- `src/p2p/` - libp2p-based networking with gossipsub message propagation
- `src/p2p/messages.rs` - Protocol message definitions and serialization
- `src/p2p/gossip.rs` - GossipSub protocol implementation
- `src/p2p/discovery.rs` - Peer discovery system
- `src/networking/` - Connection management and compression

**Core Library** (`savitri-core/`)
- `src/core/` - Base types, slot scheduler, monolith management
- `src/crypto/` - Ed25519 signatures, Blake3 hashing, curve25519 operations
- `src/utils/` - Mathematical utilities and time management
- `src/metrics/` - Metrics export and monitoring

**Zero-Knowledge Proofs** (`savitri-zkp/`)
- PLONK-based zero-knowledge proof system for monolith verification
- Mock and real backend support with feature flags

**Node Implementations**
- `lightnode/` - Light node implementation for mobile devices
- `masternode/` - Masternode with advanced services
- `guardian/` - Guardian node for network security

### Module Dependencies (Workspace Structure)

```
savitri-consensus (BFT + PoU)
    ↓
savitri-core (types, crypto, slot scheduler)
    ↓        ↓
savitri-storage ← savitri-p2p (networking)
    ↓
src/executor (transaction execution)
    ↓
src/mempool → src/consensus
```

**Workspace Members:**
- `masternode/` - Masternode binary
- `lightnode/` - Light node binary  
- `guardian/` - Guardian node binary
- `savitri-consensus/` - Consensus library crate
- `savitri-core/` - Core types and utilities crate
- `savitri-storage/` - Storage layer crate
- `savitri-zkp/` - ZKP proof system crate
- `savitri-p2p/` - P2P networking crate

## Key Architectural Decisions

**1. Custom Execution Environment**
- Non-EVM compatible virtual machine with custom bytecode
- Function selector: keccak256(function_signature)[0:4]
- Storage layout: Reserved slots 0-99 for BaseContract, custom slots 100+

**2. SIMD Optimization**
- Transaction scoring vectorization using AVX2/NEON intrinsics
- Runtime feature detection with automatic scalar fallback
- 2-3x theoretical speedup on x86_64, 1.5-2x on ARM

**3. Adaptive Weight Scheduling**
- Dynamic fee-to-block-space ratio adjustment based on mempool conditions
- Feedback mechanism with configurable update intervals
- Cross-batch optimization through score caching

**4. Monolith Architecture**
- Block aggregation with cryptographic commitment to headers
- Zero-knowledge proof verification for state consistency
- Light node optimized verification without full state download

## Performance Characteristics

**Throughput Targets**
- TPS: 10,000+ with SIMD optimization
- Block time: 500ms target
- Finality: 2-3 blocks (1-1.5 seconds)

**Resource Requirements**
- Full node: 16GB RAM, 4+ CPU cores, 1TB SSD
- Validator: 32GB RAM, 8+ CPU cores, 2TB NVMe SSD
- Light node: 512MB RAM, mobile CPU optimized

**Network Specifications**
- P2P protocol: libp2p with gossipsub
- Message serialization: bincode with Blake3 integrity checks
- Peer discovery: mDNS + bootstrap nodes
- Maximum peers: 50 per node

## Security Model

**Consensus Safety**
- BFT with $f = \lfloor(n-1)/3\rfloor$ fault tolerance
- Slashing: 50% bond penalty for equivocation
- Finality: Cryptographic certificates with $2f+1$ validator signatures

**Cryptographic Guarantees**
- Signature scheme: Ed25519 with batch verification
- Hash functions: Blake3 for performance, SHA3 for compatibility
- ZKP system: PLONK-based for monolith verification

**Network Security**
- TLS encryption for all P2P connections
- Peer reputation scoring with automatic misbehavior detection
- Rate limiting and DoS protection at protocol level

## Implementation Status

**Completed Components**
- [x] Core consensus engine with PoU scoring
- [x] SIMD-optimized transaction dispatcher
- [x] RocksDB storage layer with monolith support
- [x] libp2p networking with message propagation
- [x] Basic smart contract platform

**In Progress**
- [ ] Advanced governance system
- [ ] Cross-chain bridge implementation
- [ ] Sharding architecture
- [ ] Mobile wallet SDK

**Known Limitations**
- EVM compatibility requires bridge implementation
- Mobile validator participation not supported (bond requirements)
- Cross-shard transaction coordination under development

## Validation Criteria

**Determinism Requirements**
- All transaction execution must be deterministic across architectures
- SIMD vs scalar computation divergence < $10^{-10}$
- State root computation must be cryptographically verifiable

**Performance Benchmarks**
- SIMD speedup: ≥1.25x for batches ≥32 transactions
- Cache hit rate: ≥80% for typical workloads
- Network latency: ≤100ms for intra-region propagation

**Security Guarantees**
- Zero knowledge proof verification for all monoliths
- BFT consensus safety under $f$ Byzantine faults
- Cryptographic integrity for all stored data

This specification serves as the foundation for all implementation decisions and audit requirements.
