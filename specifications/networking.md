# Networking Specification

## Protocol Stack

### Layer 1: Transport (TCP/TLS)
- Protocol: TCP with TLS 1.3 encryption
- Port: 8333 (default), configurable
- Certificate: Self-signed with pinning
- Cipher Suites: TLS_AES_256_GCM_SHA384, TLS_CHACHA20_POLY1305_SHA256

### Layer 2: libp2p
- Implementation: `rust-libp2p` crate
- Transport: TCP + Noise protocol
- Multiplexing: Yamux
- Identification: Ed25519 peer keys

### Layer 3: Application Protocols
- Gossipsub: Message propagation
- Request-Response: Direct queries
- Stream: Large data transfer

## Peer Discovery

### Bootstrap Nodes
```rust
pub struct BootstrapNode {
    pub address: Multiaddr,          // Network address
    pub peer_id: PeerId,             // libp2p peer ID
    pub public_key: [u8; 32],       // Ed25519 public key
}
```

**Default Bootstrap Nodes:**
- `/ip4/203.0.113.1/tcp/8333/p2p/12D3KooW...`
- `/ip4/203.0.113.2/tcp/8333/p2p/12D3KooW...`
- `/ip6/2001:db8::1/tcp/8333/p2p/12D3KooW...`

### mDNS Discovery
```rust
pub struct MdnsConfig {
    pub enable: bool,                 // Enable mDNS
    pub query_interval: Duration,     // Query interval
    pub ttl: Duration,                // Record TTL
    pub service_name: String,         // Service name
}
```

**Default Configuration:**
- Enable: true
- Query Interval: 10 seconds
- TTL: 60 seconds
- Service Name: "_savitri._tcp.local"

### DHT (Distributed Hash Table)
- Type: Kademlia DHT
- Key Space: 256-bit
- Bucket Size: 20 peers
- Refresh Interval: 1 hour

## Message Propagation

### Gossipsub Configuration
```rust
pub struct GossipsubConfig {
    pub heartbeat_interval: Duration,     // Heartbeat interval
    pub history_gossip: usize,            // History gossip factor
    pub history_length: usize,            // History length
    pub mesh_n: usize,                   // Mesh degree
    pub mesh_n_low: usize,                // Low mesh degree
    pub mesh_n_high: usize,               // High mesh degree
    pub duplicate_cache_time: Duration,   // Duplicate cache TTL
}
```

**Default Parameters:**
- Heartbeat Interval: 1 second
- History Gossip: 3
- History Length: 5
- Mesh N: 8
- Mesh N Low: 6
- Mesh N High: 12
- Duplicate Cache Time: 1 minute

### Message Types
```rust
pub enum GossipsubMessage {
    Transaction(TransactionGossip),
    Block(BlockGossip),
    Consensus(ConsensusGossip),
    PeerExchange(PeerExchange),
}
```

### Topic Structure
```rust
pub const TOPIC_TRANSACTIONS: &str = "/savitri/tx/1.0.0";
pub const TOPIC_BLOCKS: &str = "/savitri/blocks/1.0.0";
pub const TOPIC_CONSENSUS: &str = "/savitri/consensus/1.0.0";
pub const TOPIC_PEERS: &str = "/savitri/peers/1.0.0";
```

## Message Formats

### Transaction Gossip
```rust
pub struct TransactionGossip {
    pub tx_hash: [u8; 64],          // Transaction hash
    pub tx_size: u32,                // Transaction size
    pub fee_rate: u64,               // Fee rate (gas/byte)
    pub first_seen: u64,             // First seen timestamp
    pub sender: PeerId,              // Propagating peer
}
```

**Binary Format:**
- TX Hash: 64 bytes
- TX Size: 4 bytes little-endian
- Fee Rate: 8 bytes little-endian
- First Seen: 8 bytes little-endian
- Sender: Variable length peer ID
- Total: 84 + peer_id_length bytes

### Block Gossip
```rust
pub struct BlockGossip {
    pub block_hash: [u8; 64],        // Block hash
    pub height: u64,                  // Block height
    pub timestamp: u64,              // Block timestamp
    pub proposer: [u8; 32],          // Proposer address
    pub tx_count: u32,               // Transaction count
    pub size: u32,                   // Block size
}
```

**Binary Format:**
- Block Hash: 64 bytes
- Height: 8 bytes little-endian
- Timestamp: 8 bytes little-endian
- Proposer: 32 bytes
- TX Count: 4 bytes little-endian
- Size: 4 bytes little-endian
- Total: 120 bytes

### Consensus Gossip
```rust
pub struct ConsensusGossip {
    pub message_type: u8,            // Message type
    pub round: u64,                  // Consensus round
    pub height: u64,                 // Block height
    pub block_hash: [u8; 64],        // Block hash
    pub validator: [u8; 32],          // Validator address
    pub timestamp: u64,              // Message timestamp
}
```

**Binary Format:**
- Message Type: 1 byte
- Round: 8 bytes little-endian
- Height: 8 bytes little-endian
- Block Hash: 64 bytes
- Validator: 32 bytes
- Timestamp: 8 bytes little-endian
- Total: 121 bytes

## Request-Response Protocol

### RPC Methods
```rust
pub enum RpcMethod {
    GetBlock(GetBlockRequest),
    GetTransaction(GetTransactionRequest),
    GetAccount(GetAccountRequest),
    GetPeers(GetPeersRequest),
    Ping(PingRequest),
}
```

### GetBlock Request
```rust
pub struct GetBlockRequest {
    pub block_hash: Option<[u8; 64]>, // Block hash (optional)
    pub height: Option<u64>,           // Height (optional)
    pub include_tx: bool,              // Include transactions
}
```

### GetBlock Response
```rust
pub struct GetBlockResponse {
    pub block: Option<Block>,          // Block data
    pub found: bool,                   // Block found
}
```

### Protocol Handlers
```rust
pub struct RpcHandler {
    pub timeout: Duration,             // Request timeout
    pub max_concurrent: usize,         // Max concurrent requests
    pub rate_limit: u32,               // Requests per second
}
```

**Default Configuration:**
- Timeout: 30 seconds
- Max Concurrent: 100
- Rate Limit: 10 requests/second

## Peer Management

### Connection Limits
```rust
pub struct ConnectionLimits {
    pub max_peers: usize,              // Maximum peer count
    pub max_outgoing: usize,           // Maximum outgoing connections
    pub max_incoming: usize,           // Maximum incoming connections
    pub min_peers: usize,              // Minimum peer count for routing
}
```

**Default Limits:**
- Max Peers: 50
- Max Outgoing: 40
- Max Incoming: 20
- Min Peers: 8

### Peer Scoring
```rust
pub struct PeerScore {
    pub reputation: f64,               // Reputation score (-1.0 to 1.0)
    pub latency: Duration,             // Average latency
    pub success_rate: f64,             // Request success rate
    pub uptime: Duration,              // Connection uptime
    pub last_seen: u64,                // Last seen timestamp
}
```

**Scoring Formula:**
$$score = 0.4 \cdot reputation + 0.3 \cdot (1 - \frac{latency}{max_{latency}}) + 0.2 \cdot success_{rate} + 0.1 \cdot \frac{uptime}{max_{uptime}}$$

### Peer Selection
```rust
pub fn select_peers_for_request(
    peers: &BTreeMap<PeerId, PeerScore>,
    count: usize,
    exclude: &[PeerId],
) -> Vec<PeerId> {
    peers.iter()
        .filter(|(id, _)| !exclude.contains(id))
        .sorted_by(|(_, a), (_, b)| b.reputation.partial_cmp(&a.reputation).unwrap())
        .take(count)
        .map(|(id, _)| *id)
        .collect()
}
```

## Security Measures

### TLS Configuration
```rust
pub struct TlsConfig {
    pub certificate: Vec<u8>,         // Server certificate
    pub private_key: Vec<u8>,          // Server private key
    pub ca_certificates: Vec<Vec<u8>>, // CA certificates
    pub verify_peers: bool,            // Verify peer certificates
}
```

### Rate Limiting
```rust
pub struct RateLimiter {
    pub requests_per_second: u32,      // Requests per second
    pub burst_size: u32,               // Burst size
    pub cleanup_interval: Duration,    // Cleanup interval
}
```

**Implementation:** Token bucket algorithm

### DoS Protection
```rust
pub struct DoSProtection {
    pub max_message_size: usize,       // Maximum message size
    pub max_requests_per_minute: u32,  // Requests per minute
    pub blacklist_duration: Duration,  // Blacklist duration
    pub whitelist: Vec<PeerId>,        // Whitelisted peers
}
```

**Default Parameters:**
- Max Message Size: 1MB
- Max Requests/Minute: 100
- Blacklist Duration: 1 hour
- Whitelist: Bootstrap nodes

## Performance Optimization

### Connection Pooling
```rust
pub struct ConnectionPool {
    pub max_idle: usize,               // Maximum idle connections
    pub idle_timeout: Duration,        // Idle timeout
    pub max_lifetime: Duration,        // Connection lifetime
}
```

### Message Batching
```rust
pub struct MessageBatch {
    pub max_batch_size: usize,         // Maximum batch size
    pub batch_timeout: Duration,       // Batch timeout
    pub compression: bool,             // Enable compression
}
```

**Compression Algorithm:** Zstandard (zstd)

### Caching
```rust
pub struct NetworkCache {
    pub block_cache: LruCache<[u8; 64], Block>,  // Block cache
    pub tx_cache: LruCache<[u8; 64], Transaction>, // Transaction cache
    pub peer_cache: LruCache<PeerId, PeerInfo>,   // Peer info cache
}
```

## Monitoring and Metrics

### Network Metrics
```rust
pub struct NetworkMetrics {
    pub peers_connected: usize,        // Connected peers
    pub messages_sent: u64,            // Messages sent
    pub messages_received: u64,        // Messages received
    pub bytes_sent: u64,               // Bytes sent
    pub bytes_received: u64,           // Bytes received
    pub latency_avg: Duration,         // Average latency
    pub success_rate: f64,             // Success rate
}
```

### Health Checks
```rust
pub struct HealthCheck {
    pub peer_connectivity: bool,       // Peer connectivity
    pub message_propagation: bool,     // Message propagation
    pub disk_space: bool,              // Disk space available
    pub cpu_usage: f64,                // CPU usage percentage
    pub memory_usage: f64,             // Memory usage percentage
}
```

## Configuration

### Network Configuration
```toml
[network]
listen_addr = "0.0.0.0:8333"
max_peers = 50
bootstrap_nodes = [
    "/ip4/203.0.113.1/tcp/8333/p2p/12D3KooW...",
    "/ip4/203.0.113.2/tcp/8333/p2p/12D3KooW..."
]

[network.tls]
enable = true
certificate_file = "cert.pem"
private_key_file = "key.pem"

[network.gossipsub]
heartbeat_interval = "1s"
mesh_n = 8
mesh_n_low = 6
mesh_n_high = 12

[network.rpc]
enable = true
timeout = "30s"
max_concurrent = 100
```

## Error Handling

### Network Errors
```rust
pub enum NetworkError {
    ConnectionFailed(PeerId),          // Connection failed
    MessageTooLarge(usize),             // Message too large
    InvalidFormat(String),              // Invalid format
    Timeout(PeerId),                    // Request timeout
    RateLimited(PeerId),                // Rate limited
    ProtocolError(String),             // Protocol error
}
```

### Recovery Strategies
- Exponential backoff for reconnections
- Peer rotation for failed connections
- Graceful degradation under load
- Circuit breaker for failing peers

This networking specification provides the foundation for secure, efficient peer-to-peer communication in Savitri Network.
