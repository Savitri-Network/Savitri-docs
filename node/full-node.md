# Full Node Specification

## Overview

A Savitri Network full node is a complete network participant that maintains the full blockchain state, participates in consensus (if bonded), validates all transactions and blocks, and provides network services to other nodes. Full nodes are the backbone of the Savitri Network, ensuring security, decentralization, and data availability.

## Technology Choice Rationale

### Why Full Node Architecture

**Problem Statement**: Blockchain networks require nodes that can maintain complete state, validate all operations, and provide services while remaining economically viable and performant.

**Chosen Solution**: Modular full node architecture with optional consensus participation, efficient storage, and comprehensive service capabilities.

**Rationale**:
- **Complete Validation**: Full state enables comprehensive transaction and block validation
- **Service Provision**: Full nodes provide essential services to light nodes and applications
- **Consensus Option**: Optional validator participation allows flexible node operation
- **Storage Efficiency**: Monolith storage reduces storage requirements while maintaining full state
- **Network Health**: Full nodes ensure network decentralization and security

**Expected Results**:
- Complete blockchain state validation and storage
- Network service provision (RPC, P2P, data availability)
- Optional consensus participation with economic incentives
- Efficient storage through Monolith compression
- High availability and network reliability

## Core Architecture

### System Components

```rust
pub struct FullNode {
    // Blockchain components
    pub storage: Arc<Storage>,                  // Blockchain storage
    pub consensus_engine: ConsensusEngine,       // Consensus engine
    pub mempool: Arc<Mempool>,                  // Transaction pool
    pub executor: Arc<Executor>,                // Transaction executor
    
    // P2P components
    pub p2p_service: P2PService,                // P2P service
    pub peer_manager: PeerManager,              // Peer management
    pub message_handler: MessageHandler,        // Message handling
    
    // Service components
    pub rpc_service: RpcService,                // RPC service
    pub sync_service: SyncService,              // Sync service
    pub api_service: ApiService,                // API service
    
    // Monitoring components
    pub metrics_collector: MetricsCollector,   // Metrics collection
    pub health_checker: HealthChecker,          // Health checking
    pub performance_monitor: PerformanceMonitor, // Performance monitoring
}
```

### Node Configuration

```rust
pub struct FullNodeConfig {
    // Network configuration
    pub listen_port: u16,                      // P2P listen port
    pub rpc_port: u16,                          // RPC port
    pub max_peers: usize,                       // Maximum peers
    pub bootstrap_nodes: Vec<String>,          // Bootstrap nodes
    
    // Consensus configuration
    pub enable_consensus: bool,                 // Enable consensus
    pub bond_amount: Option<u128>,             // Bond amount
    pub validator_key: Option<Keypair>,        // Validator key
    
    // Storage configuration
    pub db_path: String,                        // Database path
    pub cache_size: usize,                      // Cache size
    pub monolith_policy: MonolithPolicy,       // Monolith policy
    
    // Service configuration
    pub enable_rpc: bool,                       // Enable RPC
    pub enable_api: bool,                       // Enable API
    pub enable_metrics: bool,                   // Enable metrics
    
    // Performance configuration
    pub worker_threads: usize,                  // Worker threads
    pub batch_size: usize,                      // Batch size
    pub timeout: Duration,                      // Request timeout
}
```

### Recommended Specifications

**CPU**: 16 cores, 3.0GHz+ (Intel/AMD x64)
- **Benefits**: Higher TPS, better consensus performance, parallel processing

**Memory**: 32GB RAM
- **Benefits**: Larger state cache, higher mempool capacity, improved performance

**Storage**: 1TB NVMe SSD
- **Benefits**: Extended historical data, better I/O performance

**Network**: 1Gbps+ symmetric connection
- **Benefits**: Faster block propagation, better RPC service

## Software Requirements

### Operating System

**Supported Systems**:
- Ubuntu 22.04 LTS (Recommended)
- Debian 12
- CentOS 9 / RHEL 9
- macOS 13+ (Development only)
- Windows 11 (Development only)

**Rationale**: Linux provides best performance, stability, and tooling support for production deployments.

### Dependencies

**Core Dependencies**:
```toml
[dependencies]
# Core blockchain
savitri-node = "1.0.0"
rocksdb = "8.0"
libp2p = "0.53"
tokio = "1.35"

# Cryptography
ring = "0.17"
sha2 = "0.10"
blake3 = "1.5"
ed25519-dalek = "2.0"

# Serialization
serde = "1.0"
bincode = "1.3"
borsh = "1.0"

# Database
rocksdb = "8.0"

# Networking
tonic = "0.8"
hyper = "0.14"
reqwest = "0.11"

# RPC
jsonrpc = "0.17"
serde_json = "1.0"

# Metrics
prometheus = "0.13"
tracing = "0.1"

# Async
futures = "0.3"
async-trait = "0.1"

# Utilities
anyhow = "1.0"
thiserror = "1.0"
clap = "4.0"
```

**Development Dependencies**:
```toml
[dev-dependencies]
criterion = "0.5"
tempfile = "3.0"
test-case = "3.0"
mockall = "0.11"
```

## Node Operations

### Blockchain Operations

```rust
impl FullNode {
    pub fn start_consensus(&mut self) -> Result<(), NodeError> {
        // 1. Initialize consensus engine
        self.consensus_engine.initialize()?;
        
        // 2. Start slot scheduler
        self.consensus_engine.start_slot_scheduler()?;
        
        // 3. Join consensus network
        self.consensus_engine.join_consensus_network()?;
        
        // 4. Start block production
        if self.config.enable_consensus {
            self.consensus_engine.start_block_production()?;
        }
        
        Ok(())
    }
    
    pub fn process_block(&mut self, block: Block) -> Result<BlockResult, NodeError> {
        // 1. Validate block
        self.validate_block(&block)?;
        
        // 2. Execute transactions
        let execution_result = self.executor.execute_block(&block)?;
        
        // 3. Update state
        self.storage.apply_block(&block, &execution_result)?;
        
        // 4. Broadcast block
        self.p2p_service.broadcast_block(&block)?;
        
        // 5. Update metrics
        self.metrics_collector.record_block_processed(&block)?;
        
        Ok(BlockResult {
            block_hash: block.hash(),
            execution_result,
            gas_used: execution_result.gas_used,
            timestamp: current_timestamp(),
        })
    }
    
    pub fn validate_transaction(&self, tx: &SignedTx) -> Result<ValidationResult, NodeError> {
        // 1. Basic validation
        self.validate_transaction_basic(tx)?;
        
        // 2. Signature verification
        self.verify_transaction_signature(tx)?;
        
        // 3. Account validation
        self.validate_transaction_account(tx)?;
        
        // 4. Nonce validation
        self.validate_transaction_nonce(tx)?;
        
        // 5. Balance validation
        self.validate_transaction_balance(tx)?;
        
        Ok(ValidationResult::Valid)
    }
}
```

### P2P Operations

```rust
impl FullNode {
    pub fn start_p2p_service(&mut self) -> Result<(), NodeError> {
        // 1. Initialize P2P service
        self.p2p_service.initialize(&self.config)?;
        
        // 2. Start listening
        self.p2p_service.start_listening(self.config.listen_port)?;
        
        // 3. Connect to bootstrap nodes
        for bootstrap in &self.config.bootstrap_nodes {
            self.p2p_service.connect_to_peer(bootstrap)?;
        }
        
        // 4. Start peer discovery
        self.p2p_service.start_peer_discovery()?;
        
        // 5. Start message handling
        self.p2p_service.start_message_handling()?;
        
        Ok(())
    }
    
    pub fn handle_peer_message(&mut self, peer_id: PeerId, message: P2PMessage) -> Result<(), NodeError> {
        match message {
            P2PMessage::Block(block) => {
                self.handle_block_message(peer_id, block)
            },
            P2PMessage::Transaction(tx) => {
                self.handle_transaction_message(peer_id, tx)
            },
            P2PMessage::Consensus(msg) => {
                self.handle_consensus_message(peer_id, msg)
            },
            P2PMessage::Ping(ping) => {
                self.handle_ping_message(peer_id, ping)
            },
            P2PMessage::Pong(pong) => {
                self.handle_pong_message(peer_id, pong)
            },
        }
    }
}
```

### Service Operations

```rust
impl FullNode {
    pub fn start_rpc_service(&mut self) -> Result<(), NodeError> {
        // 1. Initialize RPC service
        self.rpc_service.initialize(&self.config)?;
        
        // 2. Start JSON-RPC server
        self.rpc_service.start_server(self.config.rpc_port)?;
        
        // 3. Register RPC methods
        self.register_rpc_methods()?;
        
        // 4. Start WebSocket server
        self.rpc_service.start_websocket_server()?;
        
        Ok(())
    }
    
    fn register_rpc_methods(&mut self) -> Result<(), NodeError> {
        // Blockchain methods
        self.rpc_service.register_method("eth_blockNumber", Box::new(self.get_block_number()))?;
        self.rpc_service.register_method("eth_getBlockByNumber", Box::new(self.get_block_by_number()))?;
        self.rpc_service.register_method("eth_getBlockByHash", Box::new(self.get_block_by_hash()))?;
        
        // Transaction methods
        self.rpc_service.register_method("eth_sendRawTransaction", Box::new(self.send_raw_transaction()))?;
        self.rpc_service.register_method("eth_getTransactionReceipt", Box::new(self.get_transaction_receipt()))?;
        
        // Account methods
        self.rpc_service.register_method("eth_getBalance", Box::new(self.get_balance()))?;
        self.rpc_service.register_method("eth_getTransactionCount", Box::new(self.get_transaction_count()))?;
        
        // Contract methods
        self.rpc_service.register_method("eth_call", Box::new(self.eth_call()))?;
        self.rpc_service.register_method("eth_estimateGas", Box::new(self.estimate_gas()))?;
        
        Ok(())
    }
}
```
libp2p = { version = "0.53", features = ["gossipsub", "mdns", "noise", "tcp", "yamux"] }
```

**System Libraries**:
- OpenSSL 3.0+
- LLVM/Clang 15+ (for SIMD optimizations)
- Git 2.30+

## Node Architecture

### Core Components

```rust
pub struct FullNode {
    // Core blockchain components
    pub storage: Arc<Storage>,              // RocksDB storage
    pub consensus: Arc<ConsensusEngine>,    // BFT consensus
    pub executor: Arc<ExecutionEngine>,    // Transaction execution
    pub p2p: Arc<P2PNetwork>,              // P2P networking
    pub rpc: Arc<RPCServer>,               // RPC API server
    
    // State management
    pub state_manager: Arc<StateManager>,  // State synchronization
    pub mempool: Arc<Mempool>,             // Transaction pool
    pub block_processor: Arc<BlockProcessor>, // Block processing
    
    // Configuration
    pub config: NodeConfig,                 // Node configuration
    pub key_manager: Arc<KeyManager>,     // Key management
    pub metrics: Arc<MetricsCollector>,    // Performance metrics
}
```

### Storage Architecture

**RocksDB Column Families**:
```rust
// Core blockchain state
CF_STATE      // Account state, contract storage
CF_BLOCKS     // Block headers and bodies
CF_TRANSACTIONS // Transaction data and receipts
CF_MONOLITHS  // Monolith block storage
CF_CONSENSUS  // Consensus state and votes

// Network and protocol
CF_PEERS      // Peer information and reputation
CF_GOSSIP     // Message gossip tracking
CF_EVIDENCE   // Consensus evidence

// Governance and staking
CF_BONDS      // Validator bonds
CF_GOVERNANCE // Governance proposals and votes
CF_VOTE_TOKENS // Non-transferable vote tokens

// Performance and monitoring
CF_FEE_METRICS // Transaction fee statistics
CF_PERFORMANCE // Node performance metrics
```

### Network Stack

**P2P Protocols**:
```rust
pub struct P2PNetwork {
    // Core protocols
    pub gossipsub: GossipsubBehaviour,      // Message propagation
    pub identify: IdentifyBehaviour,        // Node identification
    pub ping: PingBehaviour,                // Health checking
    pub kademlia: KademliaBehaviour,        // DHT for peer discovery
    
    // Savitri-specific protocols
    pub block_sync: BlockSyncBehaviour,     // Block synchronization
    pub tx_gossip: TxGossipBehaviour,       // Transaction gossip
    pub consensus: ConsensusBehaviour,     // Consensus messaging
    pub state_sync: StateSyncBehaviour,     // State synchronization
}
```

## Node Operations

### Startup Sequence

```rust
impl FullNode {
    pub async fn start(config: NodeConfig) -> Result<NodeHandle, NodeError> {
        // 1. Initialize storage
        let storage = Arc::new(Storage::new(&config.db_path)?);
        
        // 2. Load blockchain state
        let state = storage.load_state().await?;
        
        // 3. Initialize consensus engine
        let consensus = Arc::new(ConsensusEngine::new(
            state.clone(),
            config.consensus_config,
        )?);
        
        // 4. Initialize execution engine
        let executor = Arc::new(ExecutionEngine::new(
            storage.clone(),
            state.clone(),
            config.execution_config,
        )?);
        
        // 5. Initialize P2P network
        let p2p = Arc::new(P2PNetwork::new(
            config.network_config,
            consensus.clone(),
        ).await?);
        
        // 6. Initialize RPC server
        let rpc = Arc::new(RPCServer::new(
            config.rpc_config,
            storage.clone(),
            consensus.clone(),
            executor.clone(),
        )?);
        
        // 7. Start all services
        let handle = NodeHandle {
            storage,
            consensus,
            executor,
            p2p,
            rpc,
        };
        
        handle.start_services().await?;
        Ok(handle)
    }
}
```

### Block Processing

```rust
impl BlockProcessor {
    pub async fn process_block(&self, block: Block) -> Result<BlockResult, ProcessingError> {
        // 1. Validate block header
        self.validate_block_header(&block.header)?;
        
        // 2. Verify consensus certificate
        self.verify_consensus_certificate(&block.header, &block.certificate)?;
        
        // 3. Execute transactions
        let execution_result = self.executor.execute_block(&block).await?;
        
        // 4. Update state
        let state_root = self.update_state(&execution_result).await?;
        
        // 5. Store block
        self.storage.store_block(&block, &execution_result).await?;
        
        // 6. Update consensus state
        self.consensus.update_state(&block.header, state_root).await?;
        
        // 7. Notify peers
        self.p2p.broadcast_block(&block).await?;
        
        Ok(BlockResult {
            execution_result,
            state_root,
            gas_used: execution_result.gas_used,
        })
    }
}
```

### Transaction Processing

```rust
impl Mempool {
    pub async fn add_transaction(&self, tx: SignedTransaction) -> Result<TxHash, MempoolError> {
        // 1. Validate transaction
        self.validate_transaction(&tx)?;
        
        // 2. Check nonce
        let current_nonce = self.storage.get_nonce(&tx.signer)?;
        if tx.nonce != current_nonce + 1 {
            return Err(MempoolError::InvalidNonce);
        }
        
        // 3. Check balance
        let balance = self.storage.get_balance(&tx.signer)?;
        let required_balance = tx.value + (tx.gas_price * tx.gas_limit);
        if balance < required_balance {
            return Err(MempoolError::InsufficientBalance);
        }
        
        // 4. Calculate transaction score
        let score = self.calculate_tx_score(&tx)?;
        
        // 5. Add to mempool
        let tx_hash = tx.hash();
        self.transactions.insert(tx_hash, MempoolTx {
            tx,
            score,
            inserted_at: Instant::now(),
        });
        
        // 6. Update sender nonce
        self.pending_nonces.insert(tx.signer, tx.nonce);
        
        // 7. Gossip transaction
        self.p2p.gossip_transaction(&tx).await?;
        
        Ok(tx_hash)
    }
}
```

## Configuration

### Node Configuration

```toml
[node]
# Node identity
data_dir = "/var/lib/savitri"
log_level = "info"

# Network configuration
[network]
listen_addr = "0.0.0.0:30333"
bootstrap_nodes = [
    "/ip4/104.131.131.82/tcp/30333/p2p/12D3KooWJvyP3UJ6ApXPArq9CWKijKp7N2QzMKK4bL2X9G1XQz1",
    "/ip4/104.131.131.83/tcp/30333/p2p/12D3KooWJvyP3UJ6ApXPArq9CWKijKp7N2QzMKK4bL2X9G1XQz2",
]

max_peers = 50
boot_delay = 5

# RPC configuration
[rpc]
enabled = true
listen_addr = "127.0.0.1:8545"
max_connections = 100
rate_limit = 1000

# Consensus configuration
[consensus]
enabled = true
bond_amount = "1000000000000000"  # 1M tokens
validator = false

# Storage configuration
[storage]
cache_size = "8GB"
write_buffer_size = "128MB"
max_open_files = 1000

# Performance configuration
[performance]
sync_batch_size = 100
max_concurrent_blocks = 5
gc_interval = 3600
```

### Environment Variables

```bash
# Node configuration
SAVITRI_DATA_DIR=/var/lib/savitri
SAVITRI_LOG_LEVEL=info
SAVITRI_NETWORK_ID=mainnet

# Security
SAVITRI_PRIVATE_KEY_PATH=/etc/savitri/private.key
SAVITRI_PASSWORD_FILE=/etc/savitri/password

# Performance
SAVITRI_CACHE_SIZE=8GB
SAVITRI_MAX_PEERS=50
SAVITRI_RPC_ENABLED=true
```

## Monitoring and Maintenance

### Metrics Collection

**Key Metrics**:
```rust
pub struct NodeMetrics {
    // Blockchain metrics
    pub block_height: u64,
    pub block_time: Duration,
    pub transaction_count: u64,
    pub gas_used: u64,
    
    // Network metrics
    pub peer_count: usize,
    pub network_latency: Duration,
    pub bandwidth_usage: BandwidthMetrics,
    
    // Performance metrics
    pub cpu_usage: f64,
    pub memory_usage: MemoryUsage,
    pub disk_usage: DiskUsage,
    pub tps: f64,
    
    // Consensus metrics
    pub consensus_round: u64,
    pub vote_count: usize,
    pub finality_time: Duration,
}
```

### Health Checks

```rust
impl HealthChecker {
    pub async fn check_health(&self) -> HealthStatus {
        let mut status = HealthStatus::Healthy;
        
        // Check blockchain sync
        if !self.is_synced().await {
            status = HealthStatus::Degraded("Not synced");
        }
        
        // Check peer connectivity
        if self.peer_count() < MIN_PEERS {
            status = HealthStatus::Degraded("Low peer count");
        }
        
        // Check disk space
        if self.disk_usage() > 0.9 {
            status = HealthStatus::Critical("Low disk space");
        }
        
        // Check memory usage
        if self.memory_usage() > 0.9 {
            status = HealthStatus::Critical("High memory usage");
        }
        
        status
    }
}
```

### Maintenance Tasks

**Daily Tasks**:
- Backup configuration files
- Rotate log files
- Check disk space usage
- Monitor peer connectivity
- Update block metrics

**Weekly Tasks**:
- Database compaction
- Performance analysis
- Security updates
- Backup verification

**Monthly Tasks**:
- Full system backup
- Performance tuning
- Capacity planning
- Security audit

## Security Considerations

### Key Management

**Private Key Protection**:
```rust
pub struct KeyManager {
    private_key: Secp256k1SecretKey,
    encryption_key: ChaCha20Poly1305Key,
    key_file_path: PathBuf,
}

impl KeyManager {
    pub fn load_or_create(path: &Path) -> Result<Self, KeyManagerError> {
        let key_file = path.join("private_key.enc");
        
        if key_file.exists() {
            // Load existing encrypted key
            let encrypted_key = fs::read(&key_file)?;
            let password = Self::prompt_password()?;
            let private_key = Self::decrypt_key(&encrypted_key, &password)?;
            Ok(Self::new(private_key, key_file))
        } else {
            // Generate new key
            let private_key = Secp256k1SecretKey::new(&mut rand::thread_rng());
            let password = Self::prompt_password()?;
            let encrypted_key = Self::encrypt_key(&private_key, &password)?;
            fs::write(&key_file, encrypted_key)?;
            Ok(Self::new(private_key, key_file))
        }
    }
}
```

### Network Security

**Peer Validation**:
```rust
impl PeerValidator {
    pub fn validate_peer(&self, peer_info: &PeerInfo) -> ValidationResult {
        // 1. Check peer reputation
        if self.get_peer_reputation(&peer_info.peer_id) < MIN_REPUTATION {
            return ValidationResult::Reject("Low reputation");
        }
        
        // 2. Check connection limits
        if self.peer_count() >= MAX_PEERS {
            return ValidationResult::Reject("Peer limit reached");
        }
        
        // 3. Check geographic distribution
        if self.is_geo_concentrated(&peer_info.peer_id) {
            return ValidationResult::Reject("Geographic concentration");
        }
        
        ValidationResult::Accept
    }
}
```

### Access Control

**RPC Authentication**:
```rust
pub struct RPCAuth {
    jwt_secret: Vec<u8>,
    rate_limiter: Arc<RateLimiter>,
}

impl RPCAuth {
    pub fn authenticate_request(&self, request: &RPCRequest) -> Result<AuthResult, AuthError> {
        // 1. Check rate limits
        if !self.rate_limiter.check(&request.client_ip) {
            return Err(AuthError::RateLimited);
        }
        
        // 2. Validate JWT token
        let claims = self.validate_jwt(&request.authorization)?;
        
        // 3. Check permissions
        if !self.has_permission(&claims.role, &request.method) {
            return Err(AuthError::Unauthorized);
        }
        
        Ok(AuthResult {
            user_id: claims.user_id,
            permissions: claims.permissions,
        })
    }
}
```

## Troubleshooting

### Common Issues

**1. Node Not Syncing**
```bash
# Check peer connectivity
curl -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"net_peerCount","params":[],"id":1}'

# Check sync status
curl -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}'

# Restart with different bootstrap nodes
savitri-node --bootstrap-nodes <node1>,<node2>,<node3>
```

**2. High Memory Usage**
```bash
# Check memory usage
ps aux | grep savitri-node

# Reduce cache size
savitri-node --cache-size 4GB

# Enable memory compaction
savitri-node --enable-memory-compaction
```

**3. Slow Performance**
```bash
# Check CPU usage
top -p $(pgrep savitri-node)

# Check disk I/O
iostat -x 1

# Enable performance profiling
savitri-node --enable-profiling --profile-output /tmp/profile.data
```

### Recovery Procedures

**Database Corruption**:
```bash
# Stop node
sudo systemctl stop savitri-node

# Backup current data
cp -r /var/lib/savitri /var/lib/savitri.backup

# Restore from backup
cp -r /var/lib/savitri.backup /var/lib/savitri

# Restart node
sudo systemctl start savitri-node
```

**Network Partition**:
```bash
# Check network connectivity
ping -c 3 8.8.8.8

# Reset peer connections
curl -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"admin_resetPeers","params":[],"id":1}'

# Reconnect to bootstrap nodes
savitri-node --force-bootstrap
```

## Performance Optimization

### Database Optimization

**RocksDB Configuration**:
```rust
pub fn optimize_rocksdb_config() -> rocksdb::Options {
    let mut opts = rocksdb::Options::default();
    
    // Memory optimization
    opts.set_increase_parallelism(4);
    opts.set_max_background_compactions(4);
    opts.set_max_background_flushes(2);
    
    // I/O optimization
    opts.set_use_direct_io_for_flush_and_compaction(true);
    opts.set_use_direct_reads(true);
    
    // Compression
    opts.set_compression_type(rocksdb::DBCompressionType::Lz4);
    
    // Cache configuration
    opts.set_row_cache(8 * 1024 * 1024 * 1024); // 8GB
    opts.set_block_cache_size(2 * 1024 * 1024 * 1024); // 2GB
    
    opts
}
```

### Network Optimization

**P2P Configuration**:
```rust
pub fn optimize_p2p_config() -> NetworkConfig {
    NetworkConfig {
        max_peers: 100,
        peer_timeout: Duration::from_secs(30),
        connection_timeout: Duration::from_secs(10),
        keep_alive_interval: Duration::from_secs(20),
        
        // Bandwidth optimization
        max_upload_bandwidth: 1_000_000, // 1MB/s
        max_download_bandwidth: 2_000_000, // 2MB/s
        
        // Performance optimization
        enable_gossipsub_v12: true,
        enable_mplexing: true,
        enable_identify: true,
    }
}
```

This full node specification provides comprehensive guidance for deploying and operating Savitri Network full nodes with optimal performance, security, and reliability.
