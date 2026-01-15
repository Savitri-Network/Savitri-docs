# Storage Model Architecture

## Storage Stack Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                        Application Layer                         │
├──────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ │
│  │  Account    │ │ Transaction │ │   Block     │ │  Contract   │ │
│  │  Manager    │ │   Manager   │ │   Manager   │ │   Manager   │ │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘ │
├──────────────────────────────────────────────────────────────────┤
│                        Storage Engine                            │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ │
│  │  State      │ │  Monolith   │ │  Column     │ │  Cache      │ │
│  │   Trie      │ │  Manager    │ │  Families   │ │  Manager    │ │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘ │
├──────────────────────────────────────────────────────────────────┤
│                        Database Layer                            │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                    RocksDB Engine                           │ │
│  │  • SST Files  • WAL  • MemTables  • Column Families         │ │
│  └─────────────────────────────────────────────────────────────┘ │
├──────────────────────────────────────────────────────────────────┤
│                        File System                               │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ │
│  │  Data Files │ │  WAL Files  │ │  Index Files│ │  Config     │ │
│  │  (.sst)     │ │  (.log)     │ │  (.ldb)     │ │  Files      │ │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

## RocksDB Configuration

### Database Engine Setup
```rust
pub struct StorageConfig {
    pub db_path: PathBuf,                       // Database path
    pub max_open_files: i32,                    // Maximum open files
    pub write_buffer_size: usize,                // Write buffer size
    pub max_write_buffer_number: i32,            // Maximum write buffers
    pub target_file_size_base: usize,            // Target SST file size
    pub max_background_compactions: i32,         // Background compactions
    pub max_background_flushes: i32,             // Background flushes
}

impl Default for StorageConfig {
    fn default() -> Self {
        Self {
            db_path: PathBuf::from("./data"),
            max_open_files: 1000,
            write_buffer_size: 64 * 1024 * 1024,  // 64MB
            max_write_buffer_number: 3,
            target_file_size_base: 64 * 1024 * 1024,  // 64MB
            max_background_compactions: 4,
            max_background_flushes: 2,
        }
    }
}
```

### Column Family Definitions
```rust
pub struct ColumnFamilies {
    pub accounts: ColumnFamily,                  // CF_ACCOUNTS
    pub transactions: ColumnFamily,              // CF_TRANSACTIONS
    pub blocks: ColumnFamily,                    // CF_BLOCKS
    pub consensus: ColumnFamily,                 // CF_CONSENSUS
    pub bonds: ColumnFamily,                     // CF_BONDS
    pub contracts: ColumnFamily,                 // CF_CONTRACTS
    pub monoliths: ColumnFamily,                 // CF_MONOLITHS
    pub metadata: ColumnFamily,                  // CF_METADATA
}

impl ColumnFamilies {
    pub fn create_cf_descriptors() -> Vec<ColumnFamilyDescriptor> {
        vec![
            ColumnFamilyDescriptor::new("accounts", 
                Options::default().set_merge_operator("uint64add", "uint64add")),
            ColumnFamilyDescriptor::new("transactions", Options::default()),
            ColumnFamilyDescriptor::new("blocks", Options::default()),
            ColumnFamilyDescriptor::new("consensus", Options::default()),
            ColumnFamilyDescriptor::new("bonds", Options::default()),
            ColumnFamilyDescriptor::new("contracts", Options::default()),
            ColumnFamilyDescriptor::new("monoliths", Options::default()),
            ColumnFamilyDescriptor::new("metadata", Options::default()),
        ]
    }
}
```

## Account Storage

### Account Data Structure
```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Account {
    pub balance: u128,                           // Account balance
    pub nonce: u64,                              // Transaction nonce
    pub code_hash: Option<[u8; 64]>,             // Contract code hash
    pub storage_root: [u8; 64],                   // Storage trie root
    pub created_at: u64,                         // Creation timestamp
    pub last_updated: u64,                       // Last update timestamp
}
```

### Account Storage Operations
```rust
impl Storage {
    pub fn get_account(&self, address: &[u8; 32]) -> Result<Option<Account>, StorageError> {
        let key = address;
        match self.cf_accounts.get(key)? {
            Some(data) => {
                let account = bincode::deserialize(&data)?;
                Ok(Some(account))
            }
            None => Ok(None),
        }
    }
    
    pub fn put_account(&self, address: &[u8; 32], account: &Account) -> Result<(), StorageError> {
        let key = address;
        let data = bincode::serialize(account)?;
        self.cf_accounts.put(key, data)?;
        Ok(())
    }
    
    pub fn update_account_balance(&self, address: &[u8; 32], delta: i128) -> Result<u128, StorageError> {
        let mut account = self.get_account(address)?.unwrap_or_default();
        
        if delta >= 0 {
            account.balance = account.balance.saturating_add(delta as u128);
        } else {
            if account.balance < (-delta) as u128 {
                return Err(StorageError::InsufficientBalance);
            }
            account.balance = account.balance.saturating_sub((-delta) as u128);
        }
        
        account.last_updated = current_timestamp();
        self.put_account(address, &account)?;
        Ok(account.balance)
    }
}
```

## State Trie Storage

### Merkle Trie Implementation
```rust
pub struct StateTrie {
    pub db: Arc<DB>,                            // Database reference
    pub root: [u8; 64],                         // Current root hash
    pub cache: LruCache<[u8; 32], Vec<u8>>,      // Node cache
}

impl StateTrie {
    pub fn get(&self, key: &[u8; 32]) -> Result<Option<Vec<u8>>, TrieError> {
        let path = self.compute_path(key);
        let mut current_hash = self.root;
        
        for nibble in path {
            let node_key = current_hash;
            let node_data = self.get_node(&node_key)?;
            
            if let Some(value) = self.extract_value_from_node(&node_data, nibble) {
                return Ok(value);
            }
            
            current_hash = self.extract_child_hash(&node_data, nibble)?;
        }
        
        Ok(None)
    }
    
    pub fn put(&mut self, key: &[u8; 32], value: Vec<u8>) -> Result<(), TrieError> {
        let path = self.compute_path(key);
        let mut nodes_to_update = Vec::new();
        let mut current_hash = self.root;
        
        // Traverse and collect nodes to update
        for nibble in path {
            let node_key = current_hash;
            let mut node_data = self.get_node(&node_key)?;
            
            self.update_node(&mut node_data, nibble, &value);
            nodes_to_update.push((node_key, node_data));
            
            current_hash = self.extract_child_hash(&node_data, nibble)?;
        }
        
        // Batch update nodes
        for (key, data) in nodes_to_update {
            self.put_node(&key, &data)?;
        }
        
        // Update root hash
        self.root = self.compute_new_root()?;
        Ok(())
    }
    
    pub fn compute_root(&self) -> Result<[u8; 64], TrieError> {
        let mut hasher = Blake3::new();
        
        // Hash all trie nodes
        for node_key in self.get_all_node_keys()? {
            let node_data = self.get_node(&node_key)?;
            hasher.update(&node_key);
            hasher.update(&node_data);
        }
        
        let mut root = [0u8; 64];
        root.copy_from_slice(hasher.finalize().as_bytes());
        Ok(root)
    }
}
```

### Trie Path Computation
```rust
impl StateTrie {
    fn compute_path(&self, key: &[u8; 32]) -> Vec<u8> {
        let mut path = Vec::with_capacity(64);
        
        for byte in key {
            path.push(byte >> 4);      // High nibble
            path.push(byte & 0x0F);     // Low nibble
        }
        
        path
    }
    
    fn get_node(&self, hash: &[u8; 64]) -> Result<Vec<u8>, TrieError> {
        if let Some(cached) = self.cache.get(hash) {
            return Ok(cached);
        }
        
        let key = hash;
        match self.cf_accounts.get(key)? {
            Some(data) => {
                self.cache.put(*hash, data.clone());
                Ok(data)
            }
            None => Err(TrieError::NodeNotFound(*hash)),
        }
    }
}
```

## Transaction Storage

### Transaction Indexing
```rust
pub struct TransactionIndex {
    pub by_hash: HashMap<[u8; 64], TransactionLocation>,  // Hash → Location
    pub by_block: HashMap<u64, Vec<[u8; 64]>>,              // Block → Transactions
    pub by_sender: HashMap<[u8; 32], Vec<[u8; 64]>>,       // Sender → Transactions
    pub by_nonce: HashMap<([u8; 32], u64), [u8; 64]>,      // (Sender, Nonce) → Hash
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TransactionLocation {
    pub block_hash: [u8; 64],                  // Block hash
    pub block_height: u64,                     // Block height
    pub index: u32,                            // Transaction index
    pub timestamp: u64,                       // Block timestamp
}
```

### Transaction Storage Operations
```rust
impl Storage {
    pub fn put_transaction(&self, tx: &SignedTx, location: TransactionLocation) -> Result<(), StorageError> {
        let tx_hash = compute_tx_hash(tx);
        let tx_data = bincode::serialize(tx)?;
        
        // Store transaction data
        self.cf_transactions.put(tx_hash, tx_data)?;
        
        // Update indexes
        self.update_transaction_indexes(&tx_hash, &location, tx)?;
        
        Ok(())
    }
    
    pub fn get_transaction(&self, tx_hash: &[u8; 64]) -> Result<Option<SignedTx>, StorageError> {
        match self.cf_transactions.get(tx_hash)? {
            Some(data) => {
                let tx = bincode::deserialize(&data)?;
                Ok(Some(tx))
            }
            None => Ok(None),
        }
    }
    
    pub fn get_transactions_by_block(&self, block_hash: &[u8; 64]) -> Result<Vec<SignedTx>, StorageError> {
        let index_key = format!("block:{}", hex::encode(block_hash));
        
        match self.cf_metadata.get(index_key.as_bytes())? {
            Some(data) => {
                let tx_hashes: Vec<[u8; 64]> = bincode::deserialize(&data)?;
                let mut transactions = Vec::new();
                
                for tx_hash in tx_hashes {
                    if let Some(tx) = self.get_transaction(&tx_hash)? {
                        transactions.push(tx);
                    }
                }
                
                Ok(transactions)
            }
            None => Ok(Vec::new()),
        }
    }
}
```

## Block Storage

### Block Header Storage
```rust
impl Storage {
    pub fn put_block(&self, block: &Block) -> Result<(), StorageError> {
        let block_hash = compute_block_hash(&block.header);
        let header_data = bincode::serialize(&block.header)?;
        
        // Store block header
        self.cf_blocks.put(block_hash, header_data)?;
        
        // Store block metadata
        let metadata = BlockMetadata {
            hash: block_hash,
            height: block.header.exec_height,
            timestamp: block.header.timestamp,
            tx_count: block.transactions.len() as u32,
            size: bincode::serialize(block)?.len() as u32,
        };
        
        let metadata_key = format!("block:{}", block.header.exec_height);
        let metadata_data = bincode::serialize(&metadata)?;
        self.cf_metadata.put(metadata_key.as_bytes(), metadata_data)?;
        
        // Update latest block
        self.cf_metadata.put(b"latest_block", block_hash)?;
        
        Ok(())
    }
    
    pub fn get_block(&self, block_hash: &[u8; 64]) -> Result<Option<Block>, StorageError> {
        match self.cf_blocks.get(block_hash)? {
            Some(header_data) => {
                let header: BlockHeader = bincode::deserialize(&header_data)?;
                let transactions = self.get_transactions_by_block(block_hash)?;
                
                Ok(Some(Block { header, transactions }))
            }
            None => Ok(None),
        }
    }
    
    pub fn get_latest_block(&self) -> Result<Option<Block>, StorageError> {
        match self.cf_metadata.get(b"latest_block")? {
            Some(block_hash) => {
                let hash: [u8; 64] = block_hash.try_into()
                    .map_err(|_| StorageError::InvalidData)?;
                self.get_block(&hash)
            }
            None => Ok(None),
        }
    }
}
```

## Monolith Storage

### Monolith Structure
```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct MonolithHeader {
    pub version: u8,                            // Monolith version
    pub epoch_id: u64,                          // Epoch identifier
    pub exec_height: u64,                       // Execution height
    pub timestamp: u64,                         // Creation timestamp
    pub parent_id: [u8; 64],                    // Parent monolith hash
    pub headers_commit: [u8; 64],                // Headers commitment
    pub state_commit: [u8; 64],                  // State commitment
    pub proof: Vec<u8>,                         // ZKP proof
    pub id: [u8; 64],                          // Monolith ID (computed)
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct MonolithData {
    pub header: MonolithHeader,                 // Monolith header
    pub block_headers: Vec<BlockHeader>,        // Block headers
    pub block_count: u32,                       // Number of blocks
    pub total_size: u64,                        // Total size in bytes
}
```

### Monolith Storage Operations
```rust
impl Storage {
    pub fn put_monolith(&self, monolith: &MonolithData) -> Result<(), StorageError> {
        let monolith_data = bincode::serialize(monolith)?;
        
        // Store monolith data
        self.cf_monoliths.put(monolith.header.id, monolith_data)?;
        
        // Update monolith index
        let index_key = format!("monolith:{}", monolith.header.epoch_id);
        self.cf_metadata.put(index_key.as_bytes(), monolith.header.id)?;
        
        // Update latest monolith
        self.cf_metadata.put(b"latest_monolith", monolith.header.id)?;
        
        Ok(())
    }
    
    pub fn get_monolith(&self, monolith_id: &[u8; 64]) -> Result<Option<MonolithData>, StorageError> {
        match self.cf_monoliths.get(monolith_id)? {
            Some(data) => {
                let monolith = bincode::deserialize(&data)?;
                Ok(Some(monolith))
            }
            None => Ok(None),
        }
    }
    
    pub fn get_monolith_by_epoch(&self, epoch_id: u64) -> Result<Option<MonolithData>, StorageError> {
        let index_key = format!("monolith:{}", epoch_id);
        
        match self.cf_metadata.get(index_key.as_bytes())? {
            Some(monolith_id) => {
                let id: [u8; 64] = monolith_id.try_into()
                    .map_err(|_| StorageError::InvalidData)?;
                self.get_monolith(&id)
            }
            None => Ok(None),
        }
    }
}
```

## Cache Management

### Multi-Level Cache
```rust
pub struct StorageCache {
    pub l1_cache: LruCache<[u8; 32], Vec<u8>>,    // Hot data cache
    pub l2_cache: LruCache<[u8; 32], Vec<u8>>,    // Warm data cache
    pub l1_size: usize,                           // L1 cache size
    pub l2_size: usize,                           // L2 cache size
    pub l1_hits: AtomicU64,                       // L1 hit counter
    pub l2_hits: AtomicU64,                       // L2 hit counter
    pub misses: AtomicU64,                         // Miss counter
}

impl StorageCache {
    pub fn get(&mut self, key: &[u8; 32]) -> Option<Vec<u8>> {
        // Check L1 cache first
        if let Some(data) = self.l1_cache.get(key) {
            self.l1_hits.fetch_add(1, Ordering::Relaxed);
            return Some(data.clone());
        }
        
        // Check L2 cache
        if let Some(data) = self.l2_cache.get(key) {
            self.l2_hits.fetch_add(1, Ordering::Relaxed);
            // Promote to L1
            self.l1_cache.put(*key, data.clone());
            return Some(data);
        }
        
        self.misses.fetch_add(1, Ordering::Relaxed);
        None
    }
    
    pub fn put(&mut self, key: [u8; 32], data: Vec<u8>) {
        // Store in L1 cache
        self.l1_cache.put(key, data.clone());
        
        // If L1 is full, move to L2
        if self.l1_cache.len() > self.l1_size {
            if let Some((evicted_key, evicted_data)) = self.l1_cache.pop_lru() {
                self.l2_cache.put(evicted_key, evicted_data);
            }
        }
    }
}
```

## Performance Optimization

### Batch Operations
```rust
pub struct BatchWriter {
    pub batch: WriteBatch,                       // RocksDB batch
    pub max_batch_size: usize,                   // Maximum batch size
    pub flush_interval: Duration,                 // Flush interval
}

impl BatchWriter {
    pub fn put_account(&mut self, address: [u8; 32], account: &Account) -> Result<(), StorageError> {
        let data = bincode::serialize(account)?;
        self.batch.put_cf(self.cf_accounts, &address, data);
        
        if self.batch.len() >= self.max_batch_size {
            self.flush()?;
        }
        
        Ok(())
    }
    
    pub fn flush(&mut self) -> Result<(), StorageError> {
        self.db.write(self.batch.clone())?;
        self.batch.clear();
        Ok(())
    }
}
```

### Compaction Strategy
```rust
pub struct CompactionManager {
    pub compaction_style: CompactionStyle,       // Compaction style
    pub level_compaction_dynamic_level_bytes: bool, // Dynamic level sizing
    pub target_file_size_base: usize,            // Target file size
    pub max_background_compactions: i32,         // Background compactions
}

impl CompactionManager {
    pub fn configure_compaction(&self, opts: &mut Options) {
        opts.set_compaction_style(self.compaction_style);
        opts.set_level_compaction_dynamic_level_bytes(
            self.level_compaction_dynamic_level_bytes
        );
        opts.set_target_file_size_base(self.target_file_size_base);
        opts.set_max_background_compactions(self.max_background_compactions);
    }
}
```

## Storage Metrics

### Performance Metrics
```rust
pub struct StorageMetrics {
    pub read_ops: u64,                          // Read operations
    pub write_ops: u64,                         // Write operations
    pub bytes_read: u64,                        // Bytes read
    pub bytes_written: u64,                     // Bytes written
    pub cache_hit_rate: f64,                     // Cache hit rate
    pub compaction_time: Duration,               // Compaction time
    pub disk_usage: u64,                        // Disk usage
    pub sst_files: usize,                       // SST file count
}
```

### Monitoring
```rust
impl Storage {
    pub fn get_metrics(&self) -> StorageMetrics {
        let stats = self.db.statistics();
        
        StorageMetrics {
            read_ops: stats.get_number_entries_read(),
            write_ops: stats.get_number_entries_written(),
            bytes_read: stats.get_bytes_read(),
            bytes_written: stats.get_bytes_written(),
            cache_hit_rate: self.cache.get_hit_rate(),
            compaction_time: self.get_compaction_time(),
            disk_usage: self.get_disk_usage(),
            sst_files: self.get_sst_file_count(),
        }
    }
}
```

## Backup and Recovery

### Snapshot Management
```rust
pub struct SnapshotManager {
    pub snapshot_interval: Duration,              // Snapshot interval
    pub max_snapshots: usize,                   // Maximum snapshots
    pub compression: bool,                       // Enable compression
}

impl SnapshotManager {
    pub fn create_snapshot(&self, db: &DB) -> Result<PathBuf, StorageError> {
        let timestamp = current_timestamp();
        let snapshot_path = self.get_snapshot_path(timestamp);
        
        // Create incremental snapshot
        db.create_snapshot(&snapshot_path)?;
        
        // Compress if enabled
        if self.compression {
            self.compress_snapshot(&snapshot_path)?;
        }
        
        // Cleanup old snapshots
        self.cleanup_old_snapshots()?;
        
        Ok(snapshot_path)
    }
}
```

This storage model provides a robust, high-performance foundation for all data persistence needs in Savitri Network.
