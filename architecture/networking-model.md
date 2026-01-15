# Networking Model

## Overview

This document describes the networking architecture, topology, and communication patterns in Savitri Network.

## Network Topology

### Physical Topology

```
                    Internet
                       │
        ┌──────────────┼──────────────┐
        │              │              │
    Validator      Validator      Validator
    (Region A)     (Region B)     (Region C)
        │              │              │
        └──────────────┼──────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
     Full Node      Full Node      Full Node
        │              │              │
        └──────────────┼──────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
    Light Node     Light Node     Light Node
```

### Logical Topology

#### Validator Network
- **Structure**: Full mesh (all-to-all)
- **Connections**: Each validator connects to all others
- **Purpose**: Fast consensus communication
- **Latency**: <100ms target

#### Full Node Network
- **Structure**: Partial mesh
- **Connections**: 20-50 peers per node
- **Purpose**: Block and transaction propagation
- **Latency**: <500ms target

#### Light Node Network
- **Structure**: Star topology
- **Connections**: 3-5 full nodes per light node
- **Purpose**: Data queries and verification
- **Latency**: <1s target

## Message Propagation

### GossipSub Protocol

#### Topic Structure
```
/blocks/v1.0          - Block propagation
/transactions/v1.0    - Transaction broadcast
/consensus/v1.0       - Consensus messages
/state/v1.0           - State synchronization
```

#### Message Flow
```
1. Message created
   ↓
2. Published to topic
   ↓
3. GossipSub routing
   ↓
4. Peers receive message
   ↓
5. Message validated
   ↓
6. Valid messages forwarded
   ↓
7. Invalid messages dropped
```

### Propagation Strategies

#### Block Propagation
- **Strategy**: Flooding with deduplication
- **Optimization**: Header-first, then body
- **Priority**: Validator blocks prioritized
- **Deduplication**: Hash-based

#### Transaction Propagation
- **Strategy**: Epidemic flooding
- **Optimization**: Transaction pool management
- **Priority**: Fee-based prioritization
- **Deduplication**: Transaction hash

#### Consensus Messages
- **Strategy**: Direct validator communication
- **Optimization**: Aggregated signatures
- **Priority**: High priority channel
- **Reliability**: Guaranteed delivery

## Peer Management

### Peer Discovery

#### Kademlia DHT
- **Purpose**: Find peers by node ID
- **Algorithm**: Kademlia distance metric
- **Bootstrap**: Hardcoded bootstrap nodes
- **Refresh**: Periodic table refresh

#### mDNS
- **Purpose**: Local network discovery
- **Scope**: Local network only
- **Use Case**: Development, testing
- **Limitation**: Not for production

### Peer Scoring

#### Scoring Factors
```
score = w1 * uptime_score +
        w2 * latency_score +
        w3 * bandwidth_score +
        w4 * behavior_score
```

#### Score Components
- **Uptime**: Connection stability (0-1)
- **Latency**: Response time (inverse)
- **Bandwidth**: Data transfer rate (normalized)
- **Behavior**: Protocol compliance (0-1)

#### Score Usage
- Peer selection priority
- Connection management
- Banning decisions

### Connection Management

#### Connection Limits
- **Max Incoming**: 50 connections
- **Max Outgoing**: 25 connections
- **Total**: 75 connections per node

#### Connection Lifecycle
```
1. Peer discovery
   ↓
2. Connection attempt
   ↓
3. Handshake
   ↓
4. Authentication
   ↓
5. Active connection
   ↓
6. Message exchange
   ↓
7. Connection monitoring
   ↓
8. Disconnection (if needed)
```

## Network Protocols

### Transport Protocols

#### TCP/IP
- **Default**: Primary transport
- **Port**: 30303 (configurable)
- **Reliability**: Guaranteed delivery
- **Features**: Connection-oriented

#### QUIC
- **Optional**: Alternative transport
- **Port**: 30304 (configurable)
- **Benefits**: Lower latency, multiplexing
- **Status**: Experimental

### Application Protocols

#### libp2p
- **Purpose**: P2P networking framework
- **Features**: Multi-transport, multi-protocol
- **Benefits**: Modular, extensible
- **Status**: Production ready

#### GossipSub
- **Purpose**: Message propagation
- **Features**: Efficient flooding, mesh management
- **Benefits**: Low overhead, high reliability
- **Status**: Production ready

## Message Types

### Block Messages

#### Block Announcement
```
BlockAnnounce {
    block_hash: [32 bytes]
    block_number: uint64
    validator: [20 bytes]
}
```

#### Block Request
```
BlockRequest {
    block_hash: [32 bytes]
    block_number: uint64
}
```

#### Block Response
```
BlockResponse {
    block: Block
}
```

### Transaction Messages

#### Transaction Broadcast
```
TxBroadcast {
    transaction: SignedTx
}
```

#### Transaction Request
```
TxRequest {
    tx_hash: [32 bytes]
}
```

#### Transaction Response
```
TxResponse {
    transaction: SignedTx
}
```

### Consensus Messages

#### Block Proposal
```
BlockProposal {
    block: Block
    validator: [20 bytes]
    signature: [96 bytes]
}
```

#### Vote
```
Vote {
    block_hash: [32 bytes]
    validator: [20 bytes]
    signature: [96 bytes]
}
```

#### Certificate
```
Certificate {
    block_hash: [32 bytes]
    signatures: [96 bytes][]  // Aggregated BLS signatures
    validators: [20 bytes][]  // Validator addresses
}
```

## Rate Limiting

### Per-Peer Limits
- **Block Requests**: 10 per second
- **Transaction Requests**: 100 per second
- **State Requests**: 5 per second
- **Bandwidth**: Configurable (default: 10 MB/s)

### Global Limits
- **Incoming Connections**: 50 max
- **Outgoing Connections**: 25 max
- **Message Rate**: 1000 messages/second

### Rate Limiting Algorithm
- **Method**: Token bucket
- **Refill Rate**: Configurable
- **Bucket Size**: Configurable
- **Action on Violation**: Warning → Disconnect

## Network Security

### Peer Authentication
- **Method**: Ed25519 signatures
- **Process**: Challenge-response
- **Verification**: Public key validation
- **Result**: Authenticated connection

### Message Authentication
- **Blocks**: BLS signature from proposer
- **Transactions**: ECDSA signature from sender
- **Consensus**: BLS signature from validator
- **Verification**: Signature validation required

### Encryption
- **Optional**: TLS 1.3 or QUIC encryption
- **Use Case**: Sensitive data
- **Default**: Unencrypted (public blockchain)
- **Future**: End-to-end encryption

## Network Monitoring

### Metrics
- **Peer Count**: Number of connected peers
- **Message Latency**: Average propagation time
- **Bandwidth Usage**: Data transfer rate
- **Connection Stability**: Uptime percentage
- **Message Loss**: Dropped message rate

### Monitoring Tools
- **Prometheus**: Metrics collection
- **Grafana**: Visualization
- **Alerting**: Anomaly detection

## Performance Optimization

### Message Batching
- **Purpose**: Reduce overhead
- **Method**: Group multiple messages
- **Benefit**: Lower latency, higher throughput
- **Trade-off**: Slight delay

### Compression
- **Algorithm**: Snappy or gzip
- **Use Case**: Large messages
- **Benefit**: Reduced bandwidth
- **Trade-off**: CPU usage

### Caching
- **Recent Blocks**: Cache recent blocks
- **Transaction Pool**: Cache transactions
- **Peer State**: Cache peer information
- **Benefit**: Faster access

## Fault Tolerance

### Network Partitions
- **Detection**: Connection loss monitoring
- **Recovery**: Automatic reconnection
- **Handling**: Continue with available peers
- **Merge**: Automatic when partition resolves

### Peer Failures
- **Detection**: Timeout-based
- **Recovery**: Remove from peer list
- **Replacement**: Discover new peers
- **Impact**: Minimal (redundancy)

### Message Loss
- **Detection**: Sequence numbers
- **Recovery**: Retransmission
- **Handling**: Request missing messages
- **Reliability**: High (GossipSub)

## Future Enhancements

### Planned Improvements
- Advanced routing algorithms
- Better load balancing
- Improved fault tolerance
- Enhanced security

### Research Areas
- Network coding
- Advanced gossip protocols
- Quantum-resistant networking
- Edge computing integration

---

*The networking model ensures reliable and efficient communication across Savitri Network. Continuous optimization maintains low latency and high throughput.*


