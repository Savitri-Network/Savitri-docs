# Light Node Overview

## Overview

A Savitri Network light node is a resource-efficient blockchain client designed for mobile devices, embedded systems, and applications with limited resources. Light nodes provide essential blockchain functionality while maintaining minimal storage, bandwidth, and computational requirements through selective data synchronization and cryptographic verification.

**Implementation Location:** `lightnode/` crate in the workspace
**Main Entry Point:** `lightnode/src/main.rs`
**Key Modules:**
- `src/p2p/` - P2P networking integration
- `src/availability.rs` - Availability tracking
- `src/latency_service.rs` - Latency monitoring
- `src/integrity.rs` - Integrity verification

## Technology Choice Rationale

### Why Light Node Architecture

**Problem Statement**: Mobile and embedded applications need blockchain access without the resource requirements of full nodes, while maintaining security and trust in the network.

**Chosen Solution**: Selective synchronization architecture with Monolith verification, header validation, and state proofs.

**Rationale**:
- **Resource Efficiency**: Minimal storage and bandwidth requirements
- **Security**: Cryptographic verification of all received data
- **Mobile Optimization**: Battery-friendly and network-efficient
- **Trust Model**: Verifies data without storing full blockchain
- **Developer Friendly**: Simple API for mobile applications

**Expected Results**:
- Blockchain access on mobile devices with <100MB storage
- Fast synchronization times (<5 minutes initial sync)
- Battery-efficient operation with minimal background activity
- Secure verification of all blockchain data
- Simple integration for mobile and embedded applications

## Architecture Overview

### Core Components

```rust
pub struct LightNode {
    // Core blockchain components
    pub header_chain: Arc<HeaderChain>,        // Block header chain
    pub monolith_verifier: Arc<MonolithVerifier>, // Monolith verification
    pub state_prover: Arc<StateProver>,        // State proof verification
    pub wallet: Arc<Wallet>,                   // Wallet functionality
    
    // Network components
    pub p2p_client: Arc<P2PClient>,            // P2P networking
    pub rpc_client: Arc<RPCClient>,            // RPC communication
    
    // Storage components
    pub header_storage: Arc<HeaderStorage>,    // Header storage
    pub wallet_storage: Arc<WalletStorage>,    // Wallet storage
    
    // Configuration
    pub config: LightNodeConfig,               // Node configuration
    pub sync_manager: Arc<SyncManager>,       // Synchronization management
}
```

### Data Flow Architecture

**Synchronization Flow**:
```rust
impl LightNode {
    pub async fn synchronize(&mut self) -> Result<SyncResult, SyncError> {
        // 1. Connect to trusted peers
        let peers = self.p2p_client.connect_to_peers(5).await?;
        
        // 2. Download latest headers
        let latest_headers = self.download_latest_headers(&peers).await?;
        
        // 3. Verify header chain
        self.verify_header_chain(&latest_headers)?;
        
        // 4. Download Monolith headers
        let monolith_headers = self.download_monolith_headers(&peers).await?;
        
        // 5. Verify Monolith integrity
        self.verify_monolith_integrity(&monolith_headers)?;
        
        // 6. Update local state
        self.update_local_state(&latest_headers, &monolith_headers).await?;
        
        Ok(SyncResult {
            headers_synced: latest_headers.len(),
            monoliths_verified: monolith_headers.len(),
            sync_time: Instant::now() - sync_start,
        })
    }
}
```

## Resource Requirements

### Hardware Requirements

**Minimum Requirements**:
- **CPU**: 2 cores, 1.5GHz+ (ARM/x64)
- **Memory**: 512MB RAM
- **Storage**: 100MB available space
- **Network**: 3G/4G mobile or WiFi

**Recommended Requirements**:
- **CPU**: 4 cores, 2.0GHz+ (ARM/x64)
- **Memory**: 1GB RAM
- **Storage**: 200MB available space
- **Network**: 4G/5G mobile or WiFi

### Software Requirements

**Supported Platforms**:
- **Mobile**: iOS 14+, Android 8+
- **Desktop**: Windows 10+, macOS 10.15+, Linux
- **Embedded**: Raspberry Pi 4+, ARM64 devices

**Dependencies**:
```toml
[dependencies]
# Core blockchain
savitri-light-client = "1.0.0"
tokio = { version = "1.35", features = ["rt-multi-thread"] }

# Cryptography
ring = "0.17"
sha2 = "0.10"
blake3 = "1.5"

# Serialization
serde = { version = "1.0", features = ["derive"] }
bincode = "1.3"

# Networking
reqwest = { version = "0.11", features = ["json"] }
websockets = "0.3"

# Mobile-specific
jni = "0.21"  # Android
objc = "0.2"   # iOS
```

## Security Model

### Trust Assumptions

**Trust Model**:
```rust
pub struct TrustModel {
    pub trusted_peers: Vec<PeerId>,           // Trusted peer list
    pub minimum_peers: usize,                  // Minimum peer connections
    pub verification_threshold: f64,          // Verification threshold
    pub sync_depth: u64,                      // Sync depth for verification
}

impl TrustModel {
    pub fn verify_data_integrity(&self, data: &BlockchainData) -> Result<VerificationResult, VerificationError> {
        // 1. Verify cryptographic proofs
        let proof_verification = self.verify_cryptographic_proofs(data)?;
        
        // 2. Check peer consensus
        let peer_consensus = self.check_peer_consensus(data)?;
        
        // 3. Verify chain continuity
        let chain_continuity = self.verify_chain_continuity(data)?;
        
        // 4. Validate state transitions
        let state_validation = self.validate_state_transitions(data)?;
        
        // 5. Combine verification results
        let overall_score = (proof_verification.score * 0.4) +
                           (peer_consensus.score * 0.3) +
                           (chain_continuity.score * 0.2) +
                           (state_validation.score * 0.1);
        
        if overall_score >= self.verification_threshold {
            Ok(VerificationResult::Verified { score: overall_score })
        } else {
            Ok(VerificationResult::Unverified { score: overall_score })
        }
    }
}
```

### Verification Mechanisms

**Header Verification**:
```rust
impl HeaderVerifier {
    pub fn verify_block_header(&self, header: &BlockHeader, parent: &BlockHeader) -> Result<bool, VerificationError> {
        // 1. Verify timestamp sequence
        if header.timestamp <= parent.timestamp {
            return Ok(false);
        }
        
        // 2. Verify height sequence
        if header.exec_height != parent.exec_height + 1 {
            return Ok(false);
        }
        
        // 3. Verify parent hash
        if header.parent_exec_hash != parent.hash {
            return Ok(false);
        }
        
        // 4. Verify proposer signature
        if !self.verify_proposer_signature(header)? {
            return Ok(false);
        }
        
        // 5. Verify consensus certificate
        if !self.verify_consensus_certificate(header)? {
            return Ok(false);
        }
        
        Ok(true)
    }
}
```

**Monolith Verification**:
```rust
impl MonolithVerifier {
    pub fn verify_monolith_integrity(&self, monolith: &Monolith, header: &BlockHeader) -> Result<bool, VerificationError> {
        // 1. Verify Monolith header
        if monolith.header != header.monolith_header {
            return Ok(false);
        }
        
        // 2. Verify Merkle root
        let calculated_root = self.calculate_merkle_root(&monolith.transactions)?;
        if calculated_root != header.tx_root {
            return Ok(false);
        }
        
        // 3. Verify state root
        let calculated_state_root = self.calculate_state_root(&monolith.state_changes)?;
        if calculated_state_root != header.state_root {
            return Ok(false);
        }
        
        // 4. Verify transaction inclusion proofs
        for tx in &monolith.transactions {
            if !self.verify_transaction_inclusion(tx, &monolith.proofs)? {
                return Ok(false);
            }
        }
        
        Ok(true)
    }
}
```

## Performance Optimization

### Battery Optimization

**Power Management**:
```rust
pub struct PowerManager {
    pub sync_interval: Duration,                // Sync interval
    pub background_sync: bool,                 // Background sync enabled
    pub battery_threshold: f64,                 // Battery threshold
    pub network_type: NetworkType,              // Network type
}

impl PowerManager {
    pub fn should_sync(&self) -> bool {
        // 1. Check battery level
        if self.battery_level < self.battery_threshold {
            return false;
        }
        
        // 2. Check network type
        match self.network_type {
            NetworkType::WiFi => true,
            NetworkType::Cellular => self.background_sync,
            NetworkType::None => false,
        }
    }
    
    pub fn optimize_sync_frequency(&mut self) {
        match self.network_type {
            NetworkType::WiFi => {
                self.sync_interval = Duration::from_secs(30);
            },
            NetworkType::Cellular => {
                self.sync_interval = Duration::from_secs(300); // 5 minutes
            },
            NetworkType::None => {
                self.sync_interval = Duration::from_secs(3600); // 1 hour
            },
        }
    }
}
```

### Network Optimization

**Bandwidth Management**:
```rust
pub struct BandwidthManager {
    pub max_bandwidth: u64,                     // Maximum bandwidth
    pub compression_enabled: bool,             // Compression enabled
    pub delta_sync: bool,                       // Delta sync enabled
    pub prefetch_enabled: bool,                 // Prefetch enabled
}

impl BandwidthManager {
    pub fn optimize_data_transfer(&self, data: &BlockchainData) -> Result<OptimizedData, OptimizationError> {
        let mut optimized = OptimizedData::new();
        
        // 1. Apply compression
        if self.compression_enabled {
            optimized.compressed_data = self.compress_data(data)?;
        } else {
            optimized.compressed_data = data.clone();
        }
        
        // 2. Apply delta sync
        if self.delta_sync {
            optimized.delta_data = self.calculate_delta(data)?;
        }
        
        // 3. Apply prefetch optimization
        if self.prefetch_enabled {
            optimized.prefetch_data = self.generate_prefetch_data(data)?;
        }
        
        Ok(optimized)
    }
}
```

## API Interface

### Mobile API

**Simplified API for Mobile**:
```rust
pub struct LightNodeAPI {
    pub light_node: Arc<LightNode>,
    pub event_emitter: Arc<EventEmitter>,
}

impl LightNodeAPI {
    // Basic blockchain queries
    pub async fn get_balance(&self, address: &Address) -> Result<u128, APIError> {
        let state_proof = self.light_node.get_state_proof(address).await?;
        Ok(state_proof.balance)
    }
    
    pub async fn get_transaction(&self, tx_hash: &TxHash) -> Result<Transaction, APIError> {
        let tx_proof = self.light_node.get_transaction_proof(tx_hash).await?;
        Ok(tx_proof.transaction)
    }
    
    pub async fn send_transaction(&self, tx: &SignedTransaction) -> Result<TxHash, APIError> {
        // 1. Verify transaction locally
        self.light_node.verify_transaction(tx)?;
        
        // 2. Submit to network
        let tx_hash = self.light_node.submit_transaction(tx).await?;
        
        // 3. Wait for confirmation
        self.light_node.wait_for_confirmation(&tx_hash).await?;
        
        Ok(tx_hash)
    }
    
    // Event subscriptions
    pub async fn subscribe_to_blocks(&self) -> Result<Subscription<BlockEvent>, APIError> {
        let subscription = self.event_emitter.subscribe(BlockEvent::new()).await?;
        Ok(subscription)
    }
    
    pub async fn subscribe_to_transactions(&self, address: &Address) -> Result<Subscription<TransactionEvent>, APIError> {
        let subscription = self.event_emitter.subscribe(TransactionEvent::new(address.clone())).await?;
        Ok(subscription)
    }
}
```

### Platform-Specific APIs

**Android API**:
```java
public class SavitriLightNode {
    private LightNodeAPI api;
    
    public void initialize(Context context, LightNodeConfig config) {
        // Initialize native light node
        this.api = new LightNodeAPI(context, config);
    }
    
    public CompletableFuture<BigInteger> getBalance(String address) {
        return api.getBalance(Address.fromHex(address));
    }
    
    public CompletableFuture<String> sendTransaction(String signedTx) {
        return api.sendTransaction(SignedTransaction.fromHex(signedTx));
    }
    
    public void subscribeToBlocks(BlockCallback callback) {
        api.subscribeToBlocks().thenAccept(subscription -> {
            subscription.onEvent(callback::onBlock);
        });
    }
}
```

**iOS API**:
```swift
class SavitriLightNode {
    private let api: LightNodeAPI
    
    init(config: LightNodeConfig) {
        self.api = LightNodeAPI(config: config)
    }
    
    func getBalance(address: String) -> Promise<UInt256> {
        return api.getBalance(address: Address(hex: address))
    }
    
    func sendTransaction(signedTx: String) -> Promise<String> {
        return api.sendTransaction(tx: SignedTransaction(hex: signedTx))
    }
    
    func subscribeToBlocks(callback: @escaping (Block) -> Void) -> Promise<Subscription<BlockEvent>> {
        return api.subscribeToBlocks().then { subscription in
            subscription.onEvent(callback)
            return subscription
        }
    }
}
```

## Configuration

### Mobile Configuration

**Default Mobile Config**:
```toml
[light_node]
# Basic configuration
network_id = "mainnet"
sync_interval = 300  # 5 minutes
max_headers = 10000
max_monoliths = 100

# Resource limits
max_memory = "512MB"
max_storage = "100MB"
max_bandwidth = "10MB/day"

# Power management
battery_threshold = 0.2  # 20%
background_sync = true
wifi_only_sync = false

# Network configuration
trusted_peers = [
    "/ip4/104.131.131.82/tcp/30333/p2p/12D3KooWJvyP3UJ6ApXPArq9CWKijKp7N2QzMKK4bL2X9G1XQz1",
    "/ip4/104.131.131.83/tcp/30333/p2p/12D3KooWJvyP3UJ6ApXPArq9CWKijKp7N2QzMKK4bL2X9G1XQz2",
]

# Security configuration
verify_headers = true
verify_monoliths = true
require_consensus = true

# Performance configuration
compression_enabled = true
delta_sync_enabled = true
prefetch_enabled = false
```

### Platform-Specific Configuration

**Android Configuration**:
```xml
<!-- AndroidManifest.xml -->
<application>
    <service android:name=".SavitriLightNodeService" />
    
    <receiver android:name=".NetworkChangeReceiver">
        <intent-filter>
            <action android:name="android.net.conn.CONNECTIVITY_CHANGE" />
        </intent-filter>
    </receiver>
    
    <receiver android:name=".BatteryChangeReceiver">
        <intent-filter>
            <action android:name="android.intent.action.BATTERY_CHANGED" />
        </intent-filter>
    </receiver>
</application>
```

**iOS Configuration**:
```swift
// Info.plist
<key>UIBackgroundModes</key>
<array>
    <string>background-processing</string>
    <string>background-fetch</string>
</array>

<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <true/>
</dict>
```

## Use Cases

### Mobile Wallets

**Wallet Integration**:
```rust
pub struct MobileWallet {
    pub light_node: Arc<LightNode>,
    pub key_manager: Arc<KeyManager>,
    pub ui_manager: Arc<UIManager>,
}

impl MobileWallet {
    pub async fn create_wallet(&self, password: &str) -> Result<WalletAddress, WalletError> {
        // 1. Generate new key pair
        let key_pair = self.key_manager.generate_key_pair(password)?;
        
        // 2. Initialize wallet
        let wallet_address = Address::from_public_key(&key_pair.public_key);
        
        // 3. Store wallet securely
        self.key_manager.store_wallet(wallet_address, key_pair, password)?;
        
        // 4. Start background sync
        self.light_node.start_background_sync().await?;
        
        Ok(wallet_address)
    }
    
    pub async def send_transaction(&self, to: &Address, amount: u128, password: &str) -> Result<TxHash, WalletError> {
        // 1. Get current balance
        let balance = self.light_node.get_balance(&self.get_address()).await?;
        
        // 2. Check sufficient balance
        if balance < amount {
            return Err(WalletError::InsufficientBalance);
        }
        
        // 3. Create transaction
        let tx = self.create_transaction(to, amount)?;
        
        // 4. Sign transaction
        let signed_tx = self.key_manager.sign_transaction(&tx, password)?;
        
        // 5. Send transaction
        let tx_hash = self.light_node.send_transaction(&signed_tx).await?;
        
        // 6. Update UI
        self.ui_manager.show_transaction_sent(tx_hash);
        
        Ok(tx_hash)
    }
}
```

### DApps and DeFi

**DApp Integration**:
```rust
pub struct DAppConnector {
    pub light_node: Arc<LightNode>,
    pub contract_interactor: Arc<ContractInteractor>,
    pub event_listener: Arc<EventListener>,
}

impl DAppConnector {
    pub async fn call_contract(&self, contract: &Address, method: &str, params: &[Value]) -> Result<ContractResult, DAppError> {
        // 1. Get contract state
        let contract_state = self.light_node.get_contract_state(contract).await?;
        
        // 2. Prepare call data
        let call_data = self.contract_interactor.prepare_call(method, params)?;
        
        // 3. Execute call locally
        let result = self.contract_interactor.execute_call(&contract_state, &call_data)?;
        
        // 4. Verify result
        self.light_node.verify_contract_result(contract, &result).await?;
        
        Ok(result)
    }
    
    pub async fn subscribe_to_contract_events(&self, contract: &Address, event_name: &str) -> Result<EventSubscription, DAppError> {
        let subscription = self.event_listener.subscribe_to_contract_events(contract, event_name).await?;
        Ok(subscription)
    }
}
```

This light node overview provides comprehensive guidance for implementing and deploying resource-efficient Savitri Network light nodes optimized for mobile and embedded applications.
