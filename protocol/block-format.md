# Block Format Specification

## Block Structure Overview

```
┌───────────────────────────────────────────────────────────────┐
│                        Block                                  │
├───────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐ ┌─────────────────────────────────────┐  │
│  │   Block Header  │ │         Transactions                │  │
│  │   (305 bytes)   │ │         (Variable Length)           │  │
│  └─────────────────┘ └─────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘
```

## Block Header Format

### Header Structure
```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct BlockHeader {
    pub version: u8,                     // Block version (1 byte)
    pub exec_height: u64,                // Execution height (8 bytes LE)
    pub timestamp: u64,                  // Unix timestamp (8 bytes LE)
    pub parent_exec_hash: [u8; 64],     // Parent block hash (64 bytes)
    pub parent_ref_hash: Option<[u8; 64]>, // Reference hash (64 bytes)
    pub state_root: [u8; 64],           // State trie root (64 bytes)
    pub tx_root: [u8; 64],              // Transaction trie root (64 bytes)
    pub proposer: [u8; 32],             // Proposer address (32 bytes)
    pub hash: [u8; 64],                 // Block hash (64 bytes, computed)
}
```

### Binary Layout
```
Offset  | Size  | Field                | Description
--------|-------|----------------------|-------------
0       | 1     | version              | Block protocol version
1       | 8     | exec_height          | Execution height (little-endian)
9       | 8     | timestamp            | Unix timestamp (little-endian)
17      | 64    | parent_exec_hash     | Parent block hash
81      | 64    | parent_ref_hash      | Reference hash (or zeros)
145     | 64    | state_root           | State trie root
209     | 64    | tx_root              | Transaction trie root
273     | 32    | proposer             | Proposer address
305     | 64    | hash                 | Block hash (SHA512)
```

**Total Size:** 369 bytes (excluding computed hash)

### Field Specifications

#### Version
- **Type**: `u8`
- **Range**: 0-255
- **Current Version**: 1
- **Purpose**: Protocol versioning and compatibility

#### Execution Height
- **Type**: `u64`
- **Encoding**: Little-endian
- **Range**: 0 to $2^{64}-1$
- **Purpose**: Sequential block numbering

#### Timestamp
- **Type**: `u64`
- **Encoding**: Little-endian
- **Range**: Unix timestamp seconds
- **Purpose**: Block creation time

#### Parent Exec Hash
- **Type**: `[u8; 64]`
- **Encoding**: SHA512 hash
- **Purpose**: Link to previous block
- **Validation**: Must be valid block hash

#### Parent Ref Hash
- **Type**: `Option<[u8; 64]>`
- **Encoding**: SHA512 hash or all zeros
- **Purpose**: Optional reference to fork block
- **Validation**: Valid hash or all zeros

#### State Root
- **Type**: `[u8; 64]`
- **Encoding**: Blake3 hash of state trie
- **Purpose**: Cryptographic commitment to account state
- **Validation**: Must match computed state root

#### Transaction Root
- **Type**: `[u8; 64]`
- **Encoding**: Blake3 hash of transaction trie
- **Purpose**: Cryptographic commitment to transactions
- **Validation**: Must match computed transaction root

#### Proposer
- **Type**: `[u8; 32]`
- **Encoding**: Account address
- **Purpose**: Block proposer identifier
- **Validation**: Must be valid validator address

#### Block Hash
- **Type**: `[u8; 64]`
- **Encoding**: SHA512("BLK" || serialized_header)
- **Purpose**: Block identifier and integrity
- **Validation**: Must match computed hash

## Block Hash Computation

### Hash Algorithm
```rust
pub fn compute_block_hash(header: &BlockHeader) -> Result<Hash64, HashError> {
    // 1. Serialize header without hash field
    let mut header_bytes = Vec::new();
    header_bytes.push(header.version);
    header_bytes.extend_from_slice(&header.exec_height.to_le_bytes());
    header_bytes.extend_from_slice(&header.timestamp.to_le_bytes());
    header_bytes.extend_from_slice(&header.parent_exec_hash);
    
    match header.parent_ref_hash {
        Some(ref_hash) => header_bytes.extend_from_slice(&ref_hash),
        None => header_bytes.extend_from_slice(&[0u8; 64]),
    }
    
    header_bytes.extend_from_slice(&header.state_root);
    header_bytes.extend_from_slice(&header.tx_root);
    header_bytes.extend_from_slice(&header.proposer);
    
    // 2. Compute SHA512 with domain separator
    let mut hasher = Sha512::new();
    hasher.update(b"BLK");  // Domain separator
    hasher.update(&header_bytes);
    let hash_bytes = hasher.finalize();
    
    // 3. Convert to 64-byte array
    let mut hash = [0u8; 64];
    hash.copy_from_slice(&hash_bytes);
    
    Ok(Hash64::new(hash))
}
```

### Domain Separation
- **Prefix**: "BLK" (ASCII bytes: 0x42, 0x4C, 0x4B)
- **Purpose**: Prevent hash collision across contexts
- **Security**: Ensures block hashes cannot collide with other hash types

## Transaction Format

### Transaction Structure
```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SignedTx {
    pub signature: [u8; 64],            // Ed25519 signature (64 bytes)
    pub signer: [u8; 32],               // Signer address (32 bytes)
    pub nonce: u64,                     // Account nonce (8 bytes LE)
    pub fee: u128,                      // Transaction fee (16 bytes LE)
    pub call_data: Vec<u8>,              // Call data (variable length)
}
```

### Transaction Binary Layout
```
Offset  | Size  | Field                | Description
--------|-------|----------------------|-------------
0       | 64    | signature            | Ed25519 signature
64      | 32    | signer               | Signer address
96      | 8     | nonce                | Account nonce (little-endian)
104     | 16    | fee                  | Transaction fee (little-endian)
120     | 4     | call_data_length     | Call data length (little-endian)
124     | N     | call_data            | Call data (variable)
```

**Total Size:** 124 + N bytes (where N = call_data_length)

### Transaction Hash
```rust
pub fn compute_transaction_hash(tx: &SignedTx) -> Hash32 {
    let mut hasher = Blake3::new();
    
    // Serialize transaction fields
    hasher.update(&tx.signature);
    hasher.update(&tx.signer);
    hasher.update(&tx.nonce.to_le_bytes());
    hasher.update(&tx.fee.to_le_bytes());
    hasher.update(&(tx.call_data.len() as u32).to_le_bytes());
    hasher.update(&tx.call_data);
    
    let hash_bytes = hasher.finalize();
    let mut hash = [0u8; 32];
    hash.copy_from_slice(&hash_bytes);
    
    Hash32::new(hash)
}
```

## Transaction Trie

### Trie Construction
```rust
pub struct TransactionTrie {
    pub transactions: BTreeMap<[u8; 64], SignedTx>, // Hash → Transaction
    pub root: [u8; 64],                           // Trie root hash
}

impl TransactionTrie {
    pub fn build(transactions: &[SignedTx]) -> Result<TransactionTrie, TrieError> {
        let mut tx_map = BTreeMap::new();
        
        // Insert transactions into map
        for tx in transactions {
            let tx_hash = compute_transaction_hash(tx);
            tx_map.insert(tx_hash.into(), tx.clone());
        }
        
        // Build Merkle trie
        let root = Self::build_merkle_trie(&tx_map)?;
        
        Ok(TransactionTrie {
            transactions: tx_map,
            root,
        })
    }
    
    fn build_merkle_trie(transactions: &BTreeMap<[u8; 64], SignedTx>) -> Result<[u8; 64], TrieError> {
        let mut hasher = Blake3::new();
        
        // Hash all transaction hashes and data
        for (tx_hash, tx) in transactions {
            hasher.update(tx_hash);
            hasher.update(&compute_transaction_hash(tx));
        }
        
        let mut root = [0u8; 64];
        root.copy_from_slice(hasher.finalize().as_bytes());
        Ok(root)
    }
}
```

### Trie Verification
```rust
impl TransactionTrie {
    pub fn verify_inclusion(&self, tx_hash: &[u8; 64], proof: &[MerkleProof]) -> Result<bool, TrieError> {
        // 1. Check if transaction exists
        let tx = self.transactions.get(tx_hash)
            .ok_or(TrieError::TransactionNotFound)?;
        
        // 2. Verify Merkle proof
        let mut computed_hash = compute_transaction_hash(tx);
        
        for proof_node in proof {
            computed_hash = Self::hash_parent(computed_hash, proof_node);
        }
        
        Ok(computed_hash == self.root)
    }
    
    fn hash_parent(left: [u8; 64], right: [u8; 64]) -> [u8; 64] {
        let mut hasher = Blake3::new();
        hasher.update(&left);
        hasher.update(&right);
        let mut result = [0u8; 64];
        result.copy_from_slice(hasher.finalize().as_bytes());
        result
    }
}
```

## Block Validation

### Header Validation
```rust
pub struct BlockValidator {
    pub max_block_size: usize,                  // Maximum block size
    pub max_tx_count: usize,                    // Maximum transaction count
    pub min_timestamp_diff: Duration,          // Minimum timestamp difference
    pub max_timestamp_diff: Duration,          // Maximum timestamp difference
}

impl BlockValidator {
    pub fn validate_header(&self, header: &BlockHeader, parent_header: &BlockHeader) -> Result<(), ValidationError> {
        // 1. Validate version
        if header.version != CURRENT_VERSION {
            return Err(ValidationError::InvalidVersion {
                expected: CURRENT_VERSION,
                actual: header.version,
            });
        }
        
        // 2. Validate height sequence
        if header.exec_height != parent_header.exec_height + 1 {
            return Err(ValidationError::InvalidHeight {
                expected: parent_header.exec_height + 1,
                actual: header.exec_height,
            });
        }
        
        // 3. Validate timestamp
        let time_diff = Duration::from_secs(header.timestamp - parent_header.timestamp);
        if time_diff < self.min_timestamp_diff || time_diff > self.max_timestamp_diff {
            return Err(ValidationError::InvalidTimestamp {
                min_diff: self.min_timestamp_diff,
                max_diff: self.max_timestamp_diff,
                actual: time_diff,
            });
        }
        
        // 4. Validate parent hash
        if header.parent_exec_hash != parent_header.hash {
            return Err(ValidationError::InvalidParentHash {
                expected: parent_header.hash,
                actual: header.parent_exec_hash,
            });
        }
        
        // 5. Validate proposer
        if !self.is_valid_proposer(&header.proposer, header.exec_height)? {
            return Err(ValidationError::InvalidProposer(header.proposer));
        }
        
        // 6. Validate block hash
        let computed_hash = compute_block_hash(header)?;
        if header.hash != computed_hash {
            return Err(ValidationError::InvalidHash {
                expected: computed_hash,
                actual: header.hash,
            });
        }
        
        Ok(())
    }
}
```

### Transaction Validation
```rust
impl BlockValidator {
    pub fn validate_transactions(&self, transactions: &[SignedTx], state: &StateDB) -> Result<(), ValidationError> {
        // 1. Validate transaction count
        if transactions.len() > self.max_tx_count {
            return Err(ValidationError::TooManyTransactions {
                max_allowed: self.max_tx_count,
                actual: transactions.len(),
            });
        }
        
        // 2. Validate block size
        let block_size = self.calculate_block_size(transactions)?;
        if block_size > self.max_block_size {
            return Err(ValidationError::BlockTooLarge {
                max_allowed: self.max_block_size,
                actual: block_size,
            });
        }
        
        // 3. Validate individual transactions
        for tx in transactions {
            self.validate_transaction(tx, state)?;
        }
        
        // 4. Validate transaction root
        let tx_trie = TransactionTrie::build(transactions)?;
        // Note: tx_root validation happens in header validation
        
        Ok(())
    }
    
    fn validate_transaction(&self, tx: &SignedTx, state: &StateDB) -> Result<(), ValidationError> {
        // 1. Validate signature
        if !verify_signature(&tx.signature, &tx.signer, &self.serialize_tx_for_signing(tx)?) {
            return Err(ValidationError::InvalidSignature);
        }
        
        // 2. Validate nonce
        let account = state.get_account(&tx.signer)?
            .ok_or(ValidationError::AccountNotFound)?;
        
        if tx.nonce != account.nonce {
            return Err(ValidationError::InvalidNonce {
                expected: account.nonce,
                actual: tx.nonce,
            });
        }
        
        // 3. Validate balance
        let total_cost = tx.fee + self.get_call_cost(&tx.call_data)?;
        if account.balance < total_cost {
            return Err(ValidationError::InsufficientBalance {
                required: total_cost,
                available: account.balance,
            });
        }
        
        // 4. Validate call data
        self.validate_call_data(&tx.call_data)?;
        
        Ok(())
    }
}
```

## Block Serialization

### Canonical Serialization
```rust
pub trait CanonicalSerialize {
    fn canonical_serialize(&self) -> Result<Vec<u8>, SerializationError>;
}

impl CanonicalSerialize for BlockHeader {
    fn canonical_serialize(&self) -> Result<Vec<u8>, SerializationError> {
        let mut buffer = Vec::with_capacity(305);
        
        // Serialize fields in defined order
        buffer.push(self.version);
        buffer.extend_from_slice(&self.exec_height.to_le_bytes());
        buffer.extend_from_slice(&self.timestamp.to_le_bytes());
        buffer.extend_from_slice(&self.parent_exec_hash);
        
        match self.parent_ref_hash {
            Some(hash) => buffer.extend_from_slice(&hash),
            None => buffer.extend_from_slice(&[0u8; 64]),
        }
        
        buffer.extend_from_slice(&self.state_root);
        buffer.extend_from_slice(&self.tx_root);
        buffer.extend_from_slice(&self.proposer);
        
        Ok(buffer)
    }
}
```

### Deserialization
```rust
pub trait CanonicalDeserialize: Sized {
    fn canonical_deserialize(data: &[u8]) -> Result<Self, DeserializationError>;
}

impl CanonicalDeserialize for BlockHeader {
    fn canonical_deserialize(data: &[u8]) -> Result<Self, DeserializationError> {
        if data.len() < 305 {
            return Err(DeserializationError::InsufficientData);
        }
        
        let mut cursor = 0;
        
        let version = data[cursor];
        cursor += 1;
        
        let exec_height = u64::from_le_bytes([
            data[cursor], data[cursor+1], data[cursor+2], data[cursor+3],
            data[cursor+4], data[cursor+5], data[cursor+6], data[cursor+7],
        ]);
        cursor += 8;
        
        let timestamp = u64::from_le_bytes([
            data[cursor], data[cursor+1], data[cursor+2], data[cursor+3],
            data[cursor+4], data[cursor+5], data[cursor+6], data[cursor+7],
        ]);
        cursor += 8;
        
        let parent_exec_hash: [u8; 64] = data[cursor..cursor+64].try_into()
            .map_err(|_| DeserializationError::InvalidData)?;
        cursor += 64;
        
        let parent_ref_hash: [u8; 64] = data[cursor..cursor+64].try_into()
            .map_err(|_| DeserializationError::InvalidData)?;
        cursor += 64;
        
        let state_root: [u8; 64] = data[cursor..cursor+64].try_into()
            .map_err(|_| DeserializationError::InvalidData)?;
        cursor += 64;
        
        let tx_root: [u8; 64] = data[cursor..cursor+64].try_into()
            .map_err(|_| DeserializationError::InvalidData)?;
        cursor += 64;
        
        let proposer: [u8; 32] = data[cursor..cursor+32].try_into()
            .map_err(|_| DeserializationError::InvalidData)?;
        cursor += 32;
        
        // Compute hash
        let header = BlockHeader {
            version,
            exec_height,
            timestamp,
            parent_exec_hash,
            parent_ref_hash: if parent_ref_hash == [0u8; 64] { None } else { Some(parent_ref_hash) },
            state_root,
            tx_root,
            proposer,
            hash: [0u8; 64], // Will be computed
        };
        
        let hash = compute_block_hash(&header)?;
        
        Ok(BlockHeader {
            hash,
            ..header
        })
    }
}
```

## Block Size Limits

### Configuration
```rust
pub struct BlockSizeConfig {
    pub max_header_size: usize,                // Maximum header size
    pub max_transaction_size: usize,           // Maximum transaction size
    pub max_transactions_per_block: usize,     // Maximum transactions per block
    pub max_block_size: usize,                 // Maximum block size
}

impl Default for BlockSizeConfig {
    fn default() -> Self {
        Self {
            max_header_size: 369,              // Fixed header size
            max_transaction_size: 1_048_576,    // 1MB per transaction
            max_transactions_per_block: 10_000, // 10K transactions
            max_block_size: 10_485_760,        // ~10MB total
        }
    }
}
```

### Size Calculation
```rust
impl BlockSizeConfig {
    pub fn calculate_block_size(&self, transactions: &[SignedTx]) -> Result<usize, SizeError> {
        let header_size = self.max_header_size;
        let transactions_size: usize = transactions.iter()
            .map(|tx| 124 + tx.call_data.len())
            .sum();
        
        let total_size = header_size + transactions_size;
        
        if total_size > self.max_block_size {
            return Err(SizeError::BlockTooLarge {
                max_allowed: self.max_block_size,
                actual: total_size,
            });
        }
        
        Ok(total_size)
    }
}
```

## Block Examples

### Genesis Block
```rust
pub fn create_genesis_block() -> Block {
    let header = BlockHeader {
        version: 1,
        exec_height: 0,
        timestamp: GENESIS_TIMESTAMP,
        parent_exec_hash: [0u8; 64], // No parent
        parent_ref_hash: None,
        state_root: compute_genesis_state_root(),
        tx_root: [0u8; 64], // No transactions
        proposer: GENESIS_PROPOSER,
        hash: [0u8; 64], // Will be computed
    };
    
    let hash = compute_block_hash(&header).unwrap();
    
    Block {
        header: BlockHeader { hash, ..header },
        transactions: Vec::new(),
    }
}
```

### Regular Block
```rust
pub fn create_block(
    parent_hash: [u8; 64],
    height: u64,
    proposer: [u8; 32],
    transactions: Vec<SignedTx>,
    state_root: [u8; 64],
) -> Result<Block, BlockError> {
    // Build transaction trie
    let tx_trie = TransactionTrie::build(&transactions)?;
    
    // Create header
    let header = BlockHeader {
        version: 1,
        exec_height: height,
        timestamp: current_timestamp(),
        parent_exec_hash: parent_hash,
        parent_ref_hash: None,
        state_root,
        tx_root: tx_trie.root,
        proposer,
        hash: [0u8; 64], // Will be computed
    };
    
    // Compute hash
    let hash = compute_block_hash(&header)?;
    
    Ok(Block {
        header: BlockHeader { hash, ..header },
        transactions,
    })
}
```

This block format specification provides a complete, unambiguous definition of all data structures used in Savitri Network blocks.
