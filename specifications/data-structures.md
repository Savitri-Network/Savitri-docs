# Data Structures Specification

## Core Data Types

### Account
```rust
pub struct Account {
    pub balance: u128,           // Account balance in smallest unit
    pub nonce: u64,              // Transaction counter
    pub code_hash: Option<[u8; 64]>, // Contract code hash (None for EOA)
    pub storage_root: [u8; 64],  // Merkle root of contract storage
}
```

**Binary Format:**
- Balance: 16 bytes little-endian
- Nonce: 8 bytes little-endian  
- Code Hash: 64 bytes (or all zeros for EOA)
- Storage Root: 64 bytes
- Total: 142 bytes

**Storage Key:** 32-byte address = Blake3(public_key)[0:32]

### Transaction
```rust
pub struct SignedTx {
    pub signature: [u8; 64],    // Ed25519 signature
    pub signer: [u8; 32],       // Signer address
    pub nonce: u64,              // Account nonce
    pub fee: u128,               // Transaction fee
    pub call_data: Vec<u8>,      // Contract call data
}
```

**Binary Format:**
- Signature: 64 bytes
- Signer: 32 bytes
- Nonce: 8 bytes little-endian
- Fee: 16 bytes little-endian
- Call Data Length: 4 bytes little-endian
- Call Data: Variable length
- Total: 124 + call_data_length bytes

**Transaction Hash:** Blake3(serialized_transaction)

### Block Header
```rust
pub struct BlockHeader {
    pub version: u8,                     // Block version
    pub exec_height: u64,                // Execution height
    pub timestamp: u64,                  // Unix timestamp
    pub parent_exec_hash: [u8; 64],     // Parent block hash
    pub parent_ref_hash: Option<[u8; 64]>, // Reference hash
    pub state_root: [u8; 64],           // State trie root
    pub tx_root: [u8; 64],              // Transaction trie root
    pub proposer: [u8; 32],             // Proposer address
    pub hash: [u8; 64],                 // Block hash (computed)
}
```

**Binary Format:**
- Version: 1 byte
- Exec Height: 8 bytes little-endian
- Timestamp: 8 bytes little-endian
- Parent Exec Hash: 64 bytes
- Parent Ref Hash: 64 bytes (or all zeros if None)
- State Root: 64 bytes
- TX Root: 64 bytes
- Proposer: 32 bytes
- Hash: 64 bytes (computed, not stored)
- Total: 305 bytes (excluding hash)

**Block Hash:** SHA512("BLK" || serialized_header)

### Monolith Header
```rust
pub struct MonolithHeader {
    pub version: u8,                    // Monolith version
    pub epoch_id: u64,                  // Epoch identifier
    pub exec_height: u64,                // Execution height
    pub timestamp: u64,                  // Creation timestamp
    pub parent_id: [u8; 64],            // Parent monolith hash
    pub headers_commit: [u8; 64],        // Commitment to headers
    pub state_commit: [u8; 64],          // Commitment to state
    pub proof: Vec<u8>,                  // ZKP proof
    pub id: [u8; 64],                   // Monolith ID (computed)
}
```

**Binary Format:**
- Version: 1 byte
- Epoch ID: 8 bytes little-endian
- Exec Height: 8 bytes little-endian
- Timestamp: 8 bytes little-endian
- Parent ID: 64 bytes
- Headers Commit: 64 bytes
- State Commit: 64 bytes
- Proof Length: 4 bytes little-endian
- Proof: Variable length
- ID: 64 bytes (computed, not stored)
- Total: 221 + proof_length bytes

**Monolith ID:** Blake3(serialized_header)

## Consensus Data Structures

### Consensus Proposal
```rust
pub struct ConsensusProposal {
    pub block_hash: [u8; 64],          // Proposed block hash
    pub round: u64,                     // Consensus round
    pub height: u64,                    // Block height
    pub proposer: [u8; 32],             // Proposer address
    pub timestamp: u64,                 // Proposal timestamp
    pub signature: [u8; 64],           // Proposer signature
}
```

**Binary Format:**
- Block Hash: 64 bytes
- Round: 8 bytes little-endian
- Height: 8 bytes little-endian
- Proposer: 32 bytes
- Timestamp: 8 bytes little-endian
- Signature: 64 bytes
- Total: 184 bytes

### Consensus Vote
```rust
pub struct ConsensusVote {
    pub block_hash: [u8; 64],          // Voted block hash
    pub round: u64,                     // Consensus round
    pub height: u64,                    // Block height
    pub voter: [u8; 32],               // Voter address
    pub timestamp: u64,                // Vote timestamp
    pub signature: [u8; 64],           // Voter signature
}
```

**Binary Format:** Same as ConsensusProposal (184 bytes)

### Consensus Certificate
```rust
pub struct ConsensusCertificate {
    pub block_hash: [u8; 64],          // Certified block hash
    pub round: u64,                     // Finality round
    pub height: u64,                    // Block height
    pub signatures: Vec<[u8; 64]>,      // Validator signatures
    pub timestamp: u64,                 // Certificate timestamp
}
```

**Binary Format:**
- Block Hash: 64 bytes
- Round: 8 bytes little-endian
- Height: 8 bytes little-endian
- Signature Count: 4 bytes little-endian
- Signatures: 64 bytes × count
- Timestamp: 8 bytes little-endian
- Total: 92 + (64 × signature_count) bytes

## Network Data Structures

### Peer ID
```rust
pub type PeerId = [u8; 32];  // libp2p peer identifier
```

**Format:** 32-byte multihash encoding

### Network Message
```rust
pub struct NetworkMessage {
    pub message_type: u16,              // Message type identifier
    pub payload: Vec<u8>,               // Message payload
    pub sender: PeerId,                 // Sender peer ID
    pub timestamp: u64,                 // Message timestamp
}
```

**Binary Format:**
- Message Type: 2 bytes big-endian
- Payload Length: 4 bytes big-endian
- Payload: Variable length
- Sender: 32 bytes
- Timestamp: 8 bytes little-endian
- Total: 46 + payload_length bytes

## Storage Data Structures

### RocksDB Column Families

**CF_ACCOUNTS**
- Key: 32-byte account address
- Value: Serialized Account (142 bytes)

**CF_TRANSACTIONS**  
- Key: 64-byte transaction hash
- Value: Serialized SignedTx

**CF_BLOCKS**
- Key: 64-byte block hash
- Value: Serialized BlockHeader (305 bytes)

**CF_CONSENSUS**
- Key: "latest_certificate" (constant)
- Value: Serialized ConsensusCertificate

**CF_BONDS**
- Key: 32-byte validator address
- Value: BondInfo structure

### BondInfo
```rust
pub struct BondInfo {
    pub amount: u128,                   // Bond amount
    pub created_at: u64,                // Creation timestamp
    pub expires_at: Option<u64>,        // Expiration timestamp
    pub status: BondStatus,              // Bond status
}
```

**Binary Format:**
- Amount: 16 bytes little-endian
- Created At: 8 bytes little-endian
- Expires At: 8 bytes little-endian (or all zeros if None)
- Status: 1 byte
- Total: 33 bytes

## Smart Contract Data Structures

### Contract Info
```rust
pub struct ContractInfo {
    pub code: Vec<u8>,                  // Contract bytecode
    pub code_hash: [u8; 64],            // Code hash
    pub version: u64,                   // Contract version
    pub owner: [u8; 32],                // Contract owner
}
```

**Binary Format:**
- Code Length: 4 bytes little-endian
- Code: Variable length
- Code Hash: 64 bytes
- Version: 8 bytes little-endian
- Owner: 32 bytes
- Total: 108 + code_length bytes

### Call Frame
```rust
pub struct CallFrame {
    pub contract_address: [u8; 32],    // Called contract
    pub caller: [u8; 32],               // Caller address
    pub value: u128,                    // Transferred value
    pub calldata: Vec<u8>,              // Call data
    pub return_data: Vec<u8>,           // Return data
    pub gas_remaining: u64,             // Remaining gas
    pub depth: u8,                      // Call depth
    pub storage_snapshot: [u8; 64],     // Storage root snapshot
}
```

**Binary Format:**
- Contract Address: 32 bytes
- Caller: 32 bytes
- Value: 16 bytes little-endian
- Calldata Length: 4 bytes little-endian
- Calldata: Variable length
- Return Data Length: 4 bytes little-endian
- Return Data: Variable length
- Gas Remaining: 8 bytes little-endian
- Depth: 1 byte
- Storage Snapshot: 64 bytes
- Total: 161 + calldata_length + return_data_length bytes

## Performance Data Structures

### Score Cache Entry
```rust
pub struct CacheEntry {
    pub score: f64,                     // Cached score
    pub timestamp: u64,                 // Cache timestamp
    pub block_height: u64,              // Height when cached
}
```

**Binary Format:**
- Score: 8 bytes IEEE 754
- Timestamp: 8 bytes little-endian
- Block Height: 8 bytes little-endian
- Total: 24 bytes

### PoU Score Components
```rust
pub struct ScoreComponentsStored {
    pub availability: u32,              // U component (fixed-point)
    pub latency: u32,                   // L component (fixed-point)
    pub integrity: u32,                  // I component (fixed-point)
    pub reputation: u32,                // R component (fixed-point)
    pub participation: u32,             // P component (fixed-point)
    pub total: u32,                     // Total score (fixed-point)
}
```

**Binary Format:**
- Each component: 4 bytes little-endian
- Total: 24 bytes

**Fixed-Point Format:** Q16.16 (16 integer bits, 16 fractional bits)

## Cryptographic Data Structures

### ZKP Statement
```rust
pub struct Statement {
    pub parent_root: [u8; 32],          // Parent state root
    pub headers_commit: [u8; 32],      // Headers commitment
    pub state_commit: [u8; 32],        // State commitment
    pub exec_height: u64,              // Execution height
    pub epoch_id: u64,                 // Epoch identifier
}
```

**Binary Format:**
- Parent Root: 32 bytes
- Headers Commit: 32 bytes
- State Commit: 32 bytes
- Exec Height: 8 bytes little-endian
- Epoch ID: 8 bytes little-endian
- Total: 112 bytes

### Merkle Proof
```rust
pub struct MerkleProof {
    pub leaf: [u8; 32],                 // Leaf hash
    pub proof: Vec<[u8; 32]>,          // Proof hashes
    pub path: Vec<bool>,               // Path bits (left/right)
}
```

**Binary Format:**
- Leaf: 32 bytes
- Proof Length: 4 bytes little-endian
- Proof: 32 bytes × length
- Path Length: 4 bytes little-endian
- Path: 1 byte × length (packed)
- Total: 40 + (32 × proof_length) + path_length bytes

## Constants and Limits

**Maximum Values:**
- Max Call Depth: 64
- Max Contract Size: 24 KB
- Max Transaction Size: 1 MB
- Max Block Size: 10 MB
- Max Peer Count: 50

**Time Constants:**
- Block Time: 500ms
- Slot Duration: 500ms
- Cache TTL: 100 blocks
- Bond Min Duration: 30 days

**Cryptographic Constants:**
- Address Size: 32 bytes
- Hash Size (Blake3): 32 bytes
- Hash Size (SHA512): 64 bytes
- Signature Size: 64 bytes
- Public Key Size: 32 bytes

This specification provides exact binary formats for all data structures used in Savitri Network implementation.
