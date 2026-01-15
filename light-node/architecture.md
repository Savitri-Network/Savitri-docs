# Light Node Architecture

## Overview

The Savitri Network light node architecture is designed for resource-constrained environments while maintaining cryptographic security and trust in the blockchain network. The architecture prioritizes minimal resource usage, battery efficiency, and fast synchronization through selective data verification and intelligent caching strategies.

## Technology Choice Rationale

### Why Light Node Architecture

**Problem Statement**: Mobile and embedded applications need blockchain functionality without the storage, bandwidth, and computational overhead of full nodes, while maintaining security guarantees.

**Chosen Solution**: Selective synchronization architecture with header chain verification, Monolith validation, and state proof verification.

**Rationale**:
- **Resource Efficiency**: Only stores essential blockchain data (headers, proofs)
- **Security**: Cryptographic verification of all received data
- **Performance**: Fast synchronization and minimal background activity
- **Mobile Optimization**: Battery-friendly and network-efficient
- **Scalability**: Supports millions of light nodes without network strain

**Expected Results**:
- <100MB storage requirement for full functionality
- <5 minute initial synchronization time
- <1% battery drain per day for background sync
- Cryptographic verification of all blockchain data
- Support for real-time blockchain interactions

## Core Architecture

### System Components

```rust
pub struct LightNodeArchitecture {
    // Core blockchain components
    pub header_chain: Arc<HeaderChain>,        // Block header chain
    pub monolith_manager: Arc<MonolithManager>, // Monolith management
    pub state_prover: Arc<StateProver>,        // State proof verification
    pub transaction_verifier: Arc<TransactionVerifier>, // Transaction verification
    
    // Network components
    pub peer_manager: Arc<PeerManager>,        // Peer connection management
    pub sync_coordinator: Arc<SyncCoordinator>, // Synchronization coordination
    pub message_handler: Arc<MessageHandler>,   // Message processing
    
    // Storage components
    pub header_storage: Arc<HeaderStorage>,    // Header storage
    pub proof_storage: Arc<ProofStorage>,      // Proof storage
    pub cache_manager: Arc<CacheManager>,      // Cache management
    
    // Mobile optimization
    pub battery_optimizer: BatteryOptimizer,   // Battery optimization
    pub network_optimizer: NetworkOptimizer,   // Network optimization
    pub resource_monitor: ResourceMonitor,     // Resource monitoring
}

### Light Node Configuration

```rust
pub struct LightNodeConfig {
    // Network configuration
    pub listen_port: u16,                      // P2P listen port
    pub max_peers: usize,                       // Maximum peers
    pub bootstrap_nodes: Vec<String>,          // Bootstrap nodes
    pub masternode_peers: Vec<String>,          // Masternode peers
    
    // Synchronization configuration
    pub sync_interval: Duration,                // Sync interval
    pub batch_size: usize,                      // Batch size
    pub max_headers_per_request: usize,        // Max headers per request
    pub max_proofs_per_request: usize,         // Max proofs per request
    
    // Storage configuration
    pub max_headers: usize,                    // Maximum headers stored
    pub max_proofs: usize,                      // Maximum proofs stored
    pub cache_size: usize,                      // Cache size
    pub storage_path: String,                   // Storage path
    
    // Mobile optimization
    pub battery_saver_mode: bool,               // Battery saver mode
    pub wifi_only_sync: bool,                   // WiFi only sync
    pub background_sync: bool,                  // Background sync
    pub adaptive_sync: bool,                    // Adaptive sync
    
    // Resource limits
    pub max_memory_mb: usize,                   // Maximum memory in MB
    pub max_storage_mb: usize,                  // Maximum storage in MB
    pub max_cpu_percent: f64,                  // Maximum CPU percentage
    pub max_network_kb_per_sec: u64,           // Maximum network KB/s
}

### Data Flow Architecture

**Synchronization Pipeline**:
```rust
impl SyncCoordinator {
    pub async fn synchronize(&mut self) -> Result<SyncResult, SyncError> {
        // 1. Initialize synchronization
        let sync_session = self.initialize_sync_session().await?;
        
        // 2. Connect to trusted peers
        let peers = self.peer_manager.connect_to_trusted_peers(5).await?;
        
        // 3. Download latest headers
        let header_batch = self.download_header_batch(&peers, sync_session.current_height + 1).await?;
        
        // 4. Verify header chain
        let verified_headers = self.verify_header_chain(&header_batch)?;
        
        // 5. Download Monolith headers
        let monolith_headers = self.download_monolith_headers(&peers, &verified_headers).await?;
        
        // 6. Verify Monolith integrity
        let verified_monoliths = self.verify_monolith_integrity(&monolith_headers)?;
        
        // 7. Update local state
        self.update_local_state(&verified_headers, &verified_monoliths).await?;
        
        // 8. Update cache
        self.cache_manager.update_cache(&verified_headers, &verified_monoliths).await?;
        
        Ok(SyncResult {
            headers_synced: verified_headers.len(),
            monoliths_verified: verified_monoliths.len(),
            sync_time: Instant::now() - sync_start,
        })
    }
}
```

## Header Chain Management

### Header Storage Architecture

**Header Chain Structure**:
```rust
pub struct HeaderChain {
    pub headers: Vec<BlockHeader>,             // Header chain
    pub verified_height: u64,                  // Verified height
    pub sync_target: u64,                      // Sync target height
    pub verification_cache: LruCache<BlockHash, VerificationResult>, // Verification cache
}

impl HeaderChain {
    pub fn add_header(&mut self, header: BlockHeader) -> Result<(), HeaderError> {
        // 1. Verify header sequence
        if let Some(last_header) = self.headers.last() {
            if header.exec_height != last_header.exec_height + 1 {
                return Err(HeaderError::InvalidSequence);
            }
            
            if header.parent_exec_hash != last_header.hash {
                return Err(HeaderError::InvalidParentHash);
            }
        }
        
        // 2. Verify cryptographic properties
        self.verify_header_cryptography(&header)?;
        
        // 3. Add to chain
        self.headers.push(header);
        self.verified_height = header.exec_height;
        
        // 4. Update cache
        self.verification_cache.put(header.hash, VerificationResult::Verified);
        
        Ok(())
    }
    
    pub fn verify_header_cryptography(&self, header: &BlockHeader) -> Result<(), HeaderError> {
        // 1. Verify proposer signature
        if !self.verify_proposer_signature(header)? {
            return Err(HeaderError::InvalidSignature);
        }
        
        // 2. Verify consensus certificate
        if !self.verify_consensus_certificate(header)? {
            return Err(HeaderError::InvalidCertificate);
        }
        
        // 3. Verify Merkle roots
        if !self.verify_merkle_roots(header)? {
            return Err(HeaderError::InvalidMerkleRoots);
        }
        
        Ok(())
    }
}
```

### Header Verification

**Verification Pipeline**:
```rust
pub struct HeaderVerifier {
    pub validator_set: Arc<ValidatorSet>,      // Current validator set
    pub consensus_rules: Arc<ConsensusRules>,  // Consensus rules
    pub cryptographic_verifier: Arc<CryptoVerifier>, // Cryptographic verification
}

impl HeaderVerifier {
    pub fn verify_header(&self, header: &BlockHeader, parent: &BlockHeader) -> Result<VerificationResult, VerificationError> {
        let mut verification_score = 0.0;
        let mut verification_details = Vec::new();
        
        // 1. Verify timestamp (20% weight)
        if self.verify_timestamp(header, parent)? {
            verification_score += 0.2;
            verification_details.push("Timestamp valid".to_string());
        } else {
            verification_details.push("Timestamp invalid".to_string());
        }
        
        // 2. Verify proposer (25% weight)
        if self.verify_proposer(header)? {
            verification_score += 0.25;
            verification_details.push("Proposer valid".to_string());
        } else {
            verification_details.push("Proposer invalid".to_string());
        }
        
        // 3. Verify consensus certificate (30% weight)
        if self.verify_consensus_certificate(header)? {
            verification_score += 0.3;
            verification_details.push("Consensus certificate valid".to_string());
        } else {
            verification_details.push("Consensus certificate invalid".to_string());
        }
        
        // 4. Verify Merkle roots (25% weight)
        if self.verify_merkle_roots(header)? {
            verification_score += 0.25;
            verification_details.push("Merkle roots valid".to_string());
        } else {
            verification_details.push("Merkle roots invalid".to_string());
        }
        
        Ok(VerificationResult {
            score: verification_score,
            details: verification_details,
            is_valid: verification_score >= 0.8,
        })
    }
}
```

## Monolith Management

### Monolith Verification Architecture

**Monolith Verification**:
```rust
pub struct MonolithManager {
    pub monolith_cache: LruCache<MonolithHash, Monolith>, // Monolith cache
    pub proof_verifier: Arc<ProofVerifier>,      // Proof verification
    pub state_tracker: Arc<StateTracker>,        // State tracking
}

impl MonolithManager {
    pub async fn verify_monolith(&self, monolith_hash: &MonolithHash, header: &BlockHeader) -> Result<MonolithVerificationResult, MonolithError> {
        // 1. Check cache first
        if let Some(cached_monolith) = self.monolith_cache.get(monolith_hash) {
            if self.verify_monolith_integrity(&cached_monolith, header)? {
                return Ok(MonolithVerificationResult::Verified { monolith: cached_monolith });
            }
        }
        
        // 2. Download Monolith
        let monolith = self.download_monolith(monolith_hash).await?;
        
        // 3. Verify Monolith header
        if monolith.header != header.monolith_header {
            return Err(MonolithError::HeaderMismatch);
        }
        
        // 4. Verify Merkle root
        let calculated_root = self.calculate_merkle_root(&monolith.transactions)?;
        if calculated_root != header.tx_root {
            return Err(MonolithError::InvalidMerkleRoot);
        }
        
        // 5. Verify state root
        let calculated_state_root = self.calculate_state_root(&monolith.state_changes)?;
        if calculated_state_root != header.state_root {
            return Err(MonolithError::InvalidStateRoot);
        }
        
        // 6. Verify transaction inclusion proofs
        for (tx, proof) in monolith.transactions.iter().zip(&monolith.proofs) {
            if !self.proof_verifier.verify_transaction_inclusion(tx, proof, &header.tx_root)? {
                return Err(MonolithError::InvalidInclusionProof);
            }
        }
        
        // 7. Cache verified Monolith
        self.monolith_cache.put(*monolith_hash, monolith.clone());
        
        Ok(MonolithVerificationResult::Verified { monolith })
    }
}
```

### State Proof Verification

**State Proof Architecture**:
```rust
pub struct StateProver {
    pub state_cache: LruCache<StateKey, StateValue>, // State cache
    pub proof_verifier: Arc<ProofVerifier>,      // Proof verification
    pub state_tracker: Arc<StateTracker>,        // State tracking
}

impl StateProver {
    pub async fn verify_state_proof(&self, proof: &StateProof, state_root: &StateRoot) -> Result<StateValue, StateProofError> {
        // 1. Verify proof structure
        self.verify_proof_structure(proof)?;
        
        // 2. Verify Merkle path
        if !self.proof_verifier.verify_merkle_path(&proof.key, &proof.value, &proof.proof, state_root)? {
            return Err(StateProofError::InvalidMerklePath);
        }
        
        // 3. Verify state transition
        if let Some(previous_state) = self.state_cache.get(&proof.key) {
            if !self.verify_state_transition(&previous_state, &proof.value, &proof.proof)? {
                return Err(StateProofError::InvalidStateTransition);
            }
        }
        
        // 4. Cache verified state
        self.state_cache.put(proof.key, proof.value);
        
        // 5. Update state tracker
        self.state_tracker.update_state(&proof.key, proof.value);
        
        Ok(proof.value)
    }
}
```

## Network Architecture

### Peer Management

**Peer Connection Architecture**:
```rust
pub struct PeerManager {
    pub trusted_peers: Vec<PeerId>,             // Trusted peer list
    pub active_connections: HashMap<PeerId, PeerConnection>, // Active connections
    pub connection_pool: Arc<ConnectionPool>,   // Connection pool
    pub peer_reputation: HashMap<PeerId, Reputation>, // Peer reputation
}

impl PeerManager {
    pub async fn connect_to_trusted_peers(&mut self, count: usize) -> Result<Vec<PeerId>, PeerError> {
        let mut connected_peers = Vec::new();
        
        for peer_id in self.trusted_peers.iter().take(count) {
            match self.connect_to_peer(peer_id).await {
                Ok(connection) => {
                    self.active_connections.insert(*peer_id, connection);
                    connected_peers.push(*peer_id);
                },
                Err(e) => {
                    log::warn!("Failed to connect to peer {}: {:?}", peer_id, e);
                    // Update reputation
                    self.update_peer_reputation(peer_id, -1.0);
                }
            }
        }
        
        if connected_peers.is_empty() {
            return Err(PeerError::NoConnections);
        }
        
        Ok(connected_peers)
    }
    
    pub async fn connect_to_peer(&self, peer_id: &PeerId) -> Result<PeerConnection, PeerError> {
        // 1. Resolve peer address
        let peer_address = self.resolve_peer_address(peer_id)?;
        
        // 2. Establish connection
        let connection = self.connection_pool.establish_connection(&peer_address).await?;
        
        // 3. Authenticate peer
        self.authenticate_peer(&connection, peer_id)?;
        
        // 4. Verify peer capabilities
        let capabilities = self.negotiate_capabilities(&connection).await?;
        
        // 5. Update peer reputation
        self.update_peer_reputation(peer_id, 1.0);
        
        Ok(PeerConnection {
            peer_id: *peer_id,
            connection,
            capabilities,
            connected_at: Instant::now(),
        })
    }
}
```

### Message Handling

**Message Processing Architecture**:
```rust
pub struct MessageHandler {
    pub header_processor: Arc<HeaderProcessor>, // Header processing
    pub monolith_processor: Arc<MonolithProcessor>, // Monolith processing
    pub transaction_processor: Arc<TransactionProcessor>, // Transaction processing
    pub message_queue: Arc<MessageQueue>,      // Message queue
}

impl MessageHandler {
    pub async fn process_message(&self, message: NetworkMessage) -> Result<ProcessingResult, ProcessingError> {
        match message.message_type {
            MessageType::BlockHeader => {
                let header: BlockHeader = serde_json::from_slice(&message.data)?;
                self.header_processor.process_header(header).await
            },
            
            MessageType::Monolith => {
                let monolith: Monolith = serde_json::from_slice(&message.data)?;
                self.monolith_processor.process_monolith(monolith).await
            },
            
            MessageType::Transaction => {
                let transaction: SignedTransaction = serde_json::from_slice(&message.data)?;
                self.transaction_processor.process_transaction(transaction).await
            },
            
            MessageType::StateProof => {
                let proof: StateProof = serde_json::from_slice(&message.data)?;
                self.process_state_proof(proof).await
            },
            
            _ => Err(ProcessingError::UnknownMessageType),
        }
    }
}
```

## Storage Architecture

### Efficient Storage Design

**Storage Components**:
```rust
pub struct LightNodeStorage {
    pub header_storage: Arc<HeaderStorage>,    // Header storage
    pub proof_storage: Arc<ProofStorage>,      // Proof storage
    pub wallet_storage: Arc<WalletStorage>,    // Wallet storage
    pub cache_storage: Arc<CacheStorage>,      // Cache storage
}

impl LightNodeStorage {
    pub fn initialize_storage(config: &StorageConfig) -> Result<Self, StorageError> {
        // 1. Initialize header storage
        let header_storage = HeaderStorage::new(&config.header_path)?;
        
        // 2. Initialize proof storage
        let proof_storage = ProofStorage::new(&config.proof_path)?;
        
        // 3. Initialize wallet storage
        let wallet_storage = WalletStorage::new(&config.wallet_path)?;
        
        // 4. Initialize cache storage
        let cache_storage = CacheStorage::new(&config.cache_path)?;
        
        Ok(LightNodeStorage {
            header_storage: Arc::new(header_storage),
            proof_storage: Arc::new(proof_storage),
            wallet_storage: Arc::new(wallet_storage),
            cache_storage: Arc::new(cache_storage),
        })
    }
}
```

### Cache Management

**Multi-Level Cache**:
```rust
pub struct CacheManager {
    pub l1_cache: LruCache<CacheKey, CacheValue>, // In-memory cache
    pub l2_cache: Arc<DiskCache>,              // Disk cache
    pub cache_policy: CachePolicy,             // Cache policy
    pub cache_stats: CacheStats,               // Cache statistics
}

impl CacheManager {
    pub async fn get(&self, key: &CacheKey) -> Option<CacheValue> {
        // 1. Try L1 cache
        if let Some(value) = self.l1_cache.get(key) {
            self.cache_stats.record_hit(CacheLevel::L1);
            return Some(value);
        }
        
        // 2. Try L2 cache
        if let Some(value) = self.l2_cache.get(key).await {
            self.cache_stats.record_hit(CacheLevel::L2);
            // Promote to L1
            self.l1_cache.put(key.clone(), value.clone());
            return Some(value);
        }
        
        // 3. Record miss
        self.cache_stats.record_miss();
        None
    }
    
    pub async fn put(&self, key: CacheKey, value: CacheValue) {
        // 1. Store in L1 cache
        self.l1_cache.put(key.clone(), value.clone());
        
        // 2. Store in L2 cache if valuable
        if self.cache_policy.should_cache_l2(&key, &value) {
            self.l2_cache.put(key, value).await;
        }
    }
}
```

## Mobile Optimization

### Power Management

**Battery Optimization**:
```rust
pub struct PowerManager {
    pub current_battery_level: f64,            // Current battery level
    pub battery_threshold: f64,                // Battery threshold
    pub power_state: PowerState,               // Current power state
    pub sync_policy: SyncPolicy,               // Sync policy
}

impl PowerManager {
    pub fn update_battery_level(&mut self, level: f64) {
        self.current_battery_level = level;
        
        // Update power state
        self.power_state = match level {
            l if l < 0.1 => PowerState::Critical,
            l if l < 0.2 => PowerState::Low,
            l if l < 0.5 => PowerState::Medium,
            _ => PowerState::High,
        };
        
        // Update sync policy
        self.update_sync_policy();
    }
    
    pub fn should_sync(&self) -> bool {
        match self.power_state {
            PowerState::Critical => false,
            PowerState::Low => false,
            PowerState::Medium => self.sync_policy.allow_background_sync,
            PowerState::High => true,
        }
    }
    
    fn update_sync_policy(&mut self) {
        match self.power_state {
            PowerState::Critical => {
                self.sync_policy.sync_interval = Duration::from_secs(3600); // 1 hour
                self.sync_policy.allow_background_sync = false;
            },
            PowerState::Low => {
                self.sync_policy.sync_interval = Duration::from_secs(1800); // 30 minutes
                self.sync_policy.allow_background_sync = false;
            },
            PowerState::Medium => {
                self.sync_policy.sync_interval = Duration::from_secs(300); // 5 minutes
                self.sync_policy.allow_background_sync = true;
            },
            PowerState::High => {
                self.sync_policy.sync_interval = Duration::from_secs(60); // 1 minute
                self.sync_policy.allow_background_sync = true;
            },
        }
    }
}
```

### Network Optimization

**Bandwidth Management**:
```rust
pub struct NetworkOptimizer {
    pub network_type: NetworkType,              // Network type
    pub bandwidth_limit: u64,                  // Bandwidth limit
    pub compression_enabled: bool,             // Compression enabled
    pub prefetch_enabled: bool,                 // Prefetch enabled
}

impl NetworkOptimizer {
    pub async fn optimize_data_transfer(&self, data: &BlockchainData) -> Result<OptimizedData, OptimizationError> {
        let mut optimized = OptimimizedData::new();
        
        // 1. Apply compression based on network type
        if self.compression_enabled && self.should_compress(data) {
            optimized.compressed_data = self.compress_data(data)?;
            optimized.compression_ratio = optimized.compressed_data.len() as f64 / data.len() as f64;
        } else {
            optimized.compressed_data = data.clone();
            optimized.compression_ratio = 1.0;
        }
        
        // 2. Apply delta sync if enabled
        if self.should_use_delta_sync(data) {
            optimized.delta_data = self.calculate_delta(data)?;
        }
        
        // 3. Apply prefetch optimization
        if self.prefetch_enabled {
            optimized.prefetch_data = self.generate_prefetch_data(data)?;
        }
        
        // 4. Apply bandwidth limits
        if optimized.compressed_data.len() > self.bandwidth_limit as usize {
            return Err(OptimizationError::BandwidthLimitExceeded);
        }
        
        Ok(optimized)
    }
    
    fn should_compress(&self, data: &BlockchainData) -> bool {
        match self.network_type {
            NetworkType::Cellular => data.len() > 1024, // Compress > 1KB on cellular
            NetworkType::WiFi => data.len() > 4096,   // Compress > 4KB on WiFi
            NetworkType::None => false,
        }
    }
}
```

## Security Architecture

### Cryptographic Security

**Verification Pipeline**:
```rust
pub struct SecurityManager {
    pub cryptographic_verifier: Arc<CryptoVerifier>, // Cryptographic verification
    pub trust_manager: Arc<TrustManager>,      // Trust management
    pub attack_detector: Arc<AttackDetector>,   // Attack detection
}

impl SecurityManager {
    pub fn verify_blockchain_data(&self, data: &BlockchainData) -> Result<SecurityVerification, SecurityError> {
        // 1. Verify cryptographic signatures
        let signature_verification = self.cryptographic_verifier.verify_signatures(data)?;
        
        // 2. Verify trust relationships
        let trust_verification = self.trust_manager.verify_trust(data)?;
        
        // 3. Detect potential attacks
        let attack_detection = self.attack_detector.detect_attacks(data)?;
        
        // 4. Combine verification results
        let overall_security = self.calculate_security_score(&signature_verification, &trust_verification, &attack_detection)?;
        
        if overall_security >= 0.8 {
            Ok(SecurityVerification::Secure { score: overall_security })
        } else {
            Ok(SecurityVerification::Suspicious { score: overall_security })
        }
    }
}
```

This light node architecture provides comprehensive guidance for implementing resource-efficient, secure, and mobile-optimized blockchain clients that maintain full cryptographic verification while minimizing resource usage.
