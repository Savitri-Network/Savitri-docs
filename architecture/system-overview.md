# System Architecture Overview

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Savitri Network                          │
├─────────────────────────────────────────────────────────────────┤
│  Application Layer                                               │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ │
│  │   RPC API   │ │   Wallet    │ │   Explorer  │ │   dApps     │ │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│  Protocol Layer                                                  │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ │
│  │ Consensus   │ │   Mempool   │ │  Executor   │ │  Contracts  │ │
│  │   Engine    │ │   Manager   │ │   Engine     │ │  Platform   │ │
│  │ (PoU-BFT)   │ │             │ │             │ │             │ │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│  Network Layer                                                   │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ │
│  │  P2P Stack  │ │  Message    │ │  Peer       │ │  Discovery  │ │
│  │  (libp2p)   │ │  Propagation│ │  Management │ │  Service    │ │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│  Storage Layer                                                   │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ │
│  │  RocksDB    │ │  Column     │ │  State      │ │  Monolith   │ │
│  │  Engine     │ │  Families   │ │  Trie       │ │  Storage    │ │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│  Cryptographic Layer                                             │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ │
│  │  Signatures │ │  Hash       │ │  ZKP        │ │  Key        │ │
│  │  (Ed25519)  │ │  Functions  │ │  System     │ │  Management │ │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

## Component Interactions

### Data Flow: Transaction to Block
```
User Transaction → RPC API → Mempool → Executor → Consensus → Storage → Network
```

1. **Transaction Submission**
   - User submits transaction via RPC API
   - Transaction validated and added to mempool
   - Transaction propagated via P2P network

2. **Transaction Processing**
   - Executor selects transactions from mempool
   - SIMD optimization for transaction scoring
   - Adaptive weight scheduling for optimal ordering

3. **Consensus Agreement**
   - Consensus engine proposes block
   - Validators vote using PoU scoring
   - Certificate generated upon agreement

4. **Block Finalization**
   - Block stored in RocksDB
   - State trie updated
   - Block propagated to network

### Data Flow: Block Verification
```
Network Block → P2P Layer → Consensus → Executor → Storage → State Update
```

## Core Components

### Consensus Engine (`src/consensus/`)
```rust
pub struct ConsensusEngine {
    pub committee: Vec<ValidatorInfo>,    // Active validators
    pub slot_scheduler: SlotScheduler,     // Slot assignment
    pub pou_scorer: PoUScorer,             // Proof of Unity scoring
    pub certificate_generator: CertGenerator, // Certificate generation
    pub evidence_collector: EvidenceCollector, // Evidence collection
}
```

**Responsibilities:**
- Leader election using PoU scoring (U/L/I/R/P components)
- BFT consensus with $2f+1$ threshold
- Certificate generation for finality
- Evidence collection and slashing for misbehavior
- Round state management and timeout handling
- Engine events emission for runtime coordination

### Execution Engine (`src/executor/`)
```rust
pub struct ExecutionDispatcher {
    pub score_cache: Arc<Mutex<ScoreCache>>, // Score caching
    pub adaptive_weights: AdaptiveWeights,    // Dynamic scheduling
    pub simd_processor: SimdProcessor,       // SIMD optimization
    pub transaction_validator: TransactionValidator, // Transaction validation
    pub nonce_resolver: AtomicNonceResolver, // Nonce conflict resolution
}
```

**Responsibilities:**
- Transaction scheduling with adaptive weights (fee/class-based)
- SIMD optimization for transaction scoring (x86_64 AVX2, ARM NEON)
- Score caching with LRU eviction and TTL
- Atomic nonce resolution for out-of-order transactions
- Transaction validation and conflict resolution
- Performance metrics collection and optimization

### Storage Layer (`src/storage/`)
```rust
pub struct Storage {
    pub db: DB,                              // RocksDB instance
    pub column_families: ColumnFamilies,      // CF management
    pub state_trie: StateTrie,               // State management
    pub monolith_manager: MonolithManager,    // Monolith handling
}
```

**Column Families:**
- `CF_DEFAULT`: Default RocksDB column family
- `CF_BLOCKS`: Block headers and metadata
- `CF_TX`: Transaction history
- `CF_ACCOUNTS`: Account state data
- `CF_RECEIPTS`: Transaction receipts
- `CF_META`: Blockchain metadata
- `CF_ORPHANS`: Orphaned blocks
- `CF_MISSING`: Missing blocks tracking
- `CF_MONOLITHS`: Monolith block storage
- `CF_FEE_METRICS`: Fee statistics
- `CF_VOTE_TOKENS`: Governance vote tokens
- `CF_TREASURY`: Treasury data
- `CF_GOVERNANCE`: Governance proposals
- `CF_VESTING`: Vesting schedules
- `CF_SUPPLY_METRICS`: Token supply metrics

### P2P Network (`src/p2p/`)
```rust
pub struct P2PService {
    pub swarm: Swarm<Behaviour>,              // libp2p swarm
    pub gossipsub: Gossipsub,                 // Message propagation
    pub peer_manager: PeerManager,            // Peer management
    pub message_handler: MessageHandler,      // Message processing
}
```

**Protocols:**
- Gossipsub for message propagation (consensus, transactions, blocks)
- Request-Response for direct queries (state, data requests)
- Stream for large data transfers (monoliths, bulk data)
- Identify for peer information exchange
- Ping for connectivity monitoring
- Kademlia DHT for peer discovery

**Message Types:**
- `ConsensusMessage`: Proposal, vote, certificate
- `BlockMessage`: Block propagation and sync
- `TransactionMessage`: Transaction gossip
- `MonolithMessage`: Monolith distribution
- `StateRequest`: State queries and proofs

## Node Types

### Full Node
**Components:** All system components
**Storage:** Full blockchain state
**Network:** 50 peers max
**Resources:** 16GB RAM, 4+ CPU cores, 1TB SSD

### Validator Node
**Components:** Full node + consensus participation
**Storage:** Full blockchain + consensus data
**Network:** 50 peers + validator connections
**Resources:** 32GB RAM, 8+ CPU cores, 2TB NVMe SSD
**Requirements:** 1M token bond (1,000,000,000,000,000 tokens)
**Role:** Block proposal, voting, certificate generation

### Light Node
**Components:** Header chain, state proofs, selective sync
**Storage:** Block headers + state proofs + monolith headers
**Network:** 10 peers max
**Resources:** 512MB RAM, mobile CPU, <100MB storage
**Role:** Transaction verification, state queries, mobile access
**Limitations:** No consensus participation, observer-only

### Archive Node
**Components:** Full node + historical data retention
**Storage:** Complete blockchain history + all states
**Network:** 50 peers max
**Resources:** 64GB RAM, 8+ CPU cores, 4TB NVMe SSD
**Role:** Historical queries, analytics, data services

### Masternode
**Components:** Full node + advanced services
**Storage:** Full blockchain + enhanced indexing
**Network:** 100 peers max
**Resources:** 64GB RAM, 12+ CPU cores, 4TB NVMe SSD
**Role:** Block certification, advanced services, network infrastructure

## Performance Architecture

### SIMD Optimization Pipeline
```
Transaction Batch → Score Cache → SIMD Processing → Result Aggregation
```

**Stages:**
1. **Cache Lookup**: Check for cached scores (O(1))
2. **SIMD Processing**: Vectorized computation for misses
3. **Result Combination**: Merge cached and computed results
4. **Cache Update**: Store new scores for future use

### Adaptive Weight Scheduling
```
Mempool Analysis → Weight Calculation → Transaction Ordering → Execution
```

**Feedback Loop:**
1. Analyze current mempool state
2. Calculate optimal fee-to-block-space ratio
3. Adjust transaction selection weights
4. Execute with updated parameters

### Monolith Architecture
```
Block Headers → Headers Commit → ZKP Proof → Monolith Header → Storage
```

**Benefits:**
- Light node verification without full state
- Cryptographic commitment to block history
- Efficient storage of historical data

## Security Architecture

### Cryptographic Security
- **Signatures**: Ed25519 for all signed data (transactions, votes, certificates)
- **Hashes**: SHA512 for blocks, Blake3 for performance-critical operations
- **ZKP**: PLONK for monolith verification and state proofs
- **Key Management**: Secure key derivation and storage
- **Domain Separation**: Unique domain separators for different contexts

### Network Security
- **TLS 1.3**: All P2P connections encrypted
- **Peer Authentication**: Certificate-based verification
- **Rate Limiting**: DoS protection at protocol level
- **Reputation System**: Peer behavior scoring

### Consensus Security
- **BFT Safety**: Tolerates $f$ Byzantine faults with $2f+1$ threshold
- **Slashing**: Economic penalties for misbehavior (50% slash for equivocation)
- **Finality**: Cryptographic certificates with $2f+1$ signatures
- **Evidence Collection**: Automatic misbehavior detection and reporting
- **PoU Integrity**: Meritocratic validator selection prevents centralization

## Scalability Architecture

### Horizontal Scaling
- **P2P Groups**: Dynamic group formation for validator coordination
- **Geographic Distribution**: Region-based peer clustering
- **Load Balancing**: Request distribution across nodes
- **Parallel Processing**: Multi-threaded transaction execution

### Vertical Scaling
- **SIMD Optimization**: Hardware acceleration utilization
- **Memory Efficiency**: Bounded caches and LRU eviction
- **Storage Optimization**: Column family organization
- **CPU Optimization**: Multi-core utilization

### Performance Targets
- **TPS**: 1,000+ transactions per second (current implementation)
- **Latency**: <3 seconds to finality
- **Block Time**: 1-2 seconds per block
- **Sync Time**: <30 seconds for mobile nodes
- **Storage Efficiency**: <100MB for light nodes

## Reliability Architecture

### Fault Tolerance
- **BFT Consensus**: Continues with $f$ faulty validators
- **Network Partition Handling**: Automatic recovery
- **Graceful Degradation**: Reduced functionality under stress
- **Component Isolation**: Failure containment

### Recovery Mechanisms
- **State Snapshots**: Periodic state backups
- **Peer Reconnection**: Automatic peer recovery
- **Transaction Replay**: Mempool recovery after restart
- **Consensus Recovery**: Certificate-based state restoration

### Monitoring and Alerting
- **Health Checks**: Component health monitoring
- **Performance Metrics**: Real-time performance tracking
- **Error Tracking**: Comprehensive error logging
- **Alert System**: Threshold-based notifications

## Configuration Architecture

### Runtime Configuration
```rust
pub struct NodeConfig {
    pub network: NetworkConfig,          // Network settings
    pub consensus: ConsensusConfig,      // Consensus parameters
    pub storage: StorageConfig,          // Storage settings
    pub execution: ExecutionConfig,      // Execution parameters
}
```

### Environment-Specific Configs
- **Development**: Relaxed security, verbose logging
- **Testing**: Deterministic behavior, mock components
- **Staging**: Production-like settings, limited scale
- **Production**: Maximum security, optimized performance

## Integration Points

### External Integrations
- **RPC API**: JSON-RPC 2.0 interface
- **WebSocket**: Real-time event streaming
- **REST API**: HTTP-based service interface
- **SDK Libraries**: Language-specific client libraries

### Internal Integrations
- **Metrics System**: Prometheus-compatible metrics
- **Logging System**: Structured logging with correlation
- **Configuration System**: Hot-reloadable configuration
- **Event System**: Async event bus for component communication

This architecture provides a comprehensive foundation for a scalable, secure, and performant blockchain network.
