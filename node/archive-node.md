# Archive Node Specification

## Overview

A Savitri Network archive node is a specialized full node that maintains complete historical blockchain data, provides comprehensive data access services, and supports analytical workloads. Archive nodes are essential for blockchain explorers, data analytics, historical queries, and compliance applications that require access to complete blockchain history.

## Technology Choice Rationale

### Why Archive Node Architecture

**Problem Statement**: Blockchain applications often require access to historical data, but full nodes typically prune old data to manage storage constraints, creating a gap between data availability and analytical needs.

**Chosen Solution**: Dedicated archive node architecture with optimized historical data storage, efficient indexing, and specialized query capabilities.

**Rationale**:
- **Complete Data Retention**: Maintains all blockchain data from genesis
- **Optimized Storage**: Efficient compression and indexing for historical data
- **Query Performance**: Specialized indexing for fast historical queries
- **Data Integrity**: Complete verification of blockchain history
- **Analytical Support**: Optimized for complex analytical workloads

**Expected Results**:
- Complete blockchain history availability
- Fast historical query performance
- Efficient storage utilization through compression
- Support for complex analytical queries
- Data integrity verification capabilities

## Hardware Requirements

### Minimum Specifications

**CPU**: 12 cores, 2.8GHz+ (Intel/AMD x64)
- **Rationale**: Historical data processing, complex queries, and indexing operations
- **Expected Performance**: 500+ historical queries per second

**Memory**: 64GB RAM
- **Rationale**: Large historical data caches, query processing, and indexing
- **Expected Usage**: 32GB for data cache, 16GB for query processing, 16GB for system

**Storage**: 5TB NVMe SSD + 20TB HDD
- **Rationale**: Hot storage for recent data, cold storage for historical data
- **Expected Growth**: ~10TB per year with current network activity

**Network**: 1Gbps+ symmetric connection
- **Rationale**: Historical data serving, peer synchronization, and API access
- **Expected Bandwidth**: 200GB+ per month for data serving and sync

### Recommended Specifications

**CPU**: 24 cores, 3.2GHz+ (Intel/AMD x64)
- **Benefits**: Higher query throughput, better indexing performance, parallel processing

**Memory**: 128GB RAM
- **Benefits**: Larger data caches, more concurrent queries, improved performance

**Storage**: 10TB NVMe SSD + 50TB HDD
- **Benefits**: Faster hot data access, extended historical retention

**Network**: 10Gbps+ symmetric connection
- **Benefits**: Higher data serving capacity, faster synchronization

## Storage Architecture

### Tiered Storage System

**Hot Storage (NVMe SSD)**:
```rust
pub struct HotStorage {
    // Recent blocks (last 30 days)
    pub recent_blocks: Arc<RocksDB>,          // CF_BLOCKS, CF_TRANSACTIONS
    
    // Active state and indices
    pub active_state: Arc<RocksDB>,           // CF_STATE, CF_MONOLITHS
    pub query_indices: Arc<RocksDB>,          // Custom indices for queries
    
    // Performance optimization
    pub cache_size: usize,                     // 32GB cache
    pub compression: CompressionType,          // LZ4 for speed
}
```

**Cold Storage (HDD)**:
```rust
pub struct ColdStorage {
    // Historical blocks
    pub historical_blocks: Arc<ArchivalDB>,   // Compressed block storage
    
    // Historical indices
    pub historical_indices: Arc<ArchivalDB>, // Time-series, address indices
    
    // Data optimization
    pub compression: CompressionType,          // ZSTD for maximum compression
    pub deduplication: bool,                  // Enable data deduplication
}
```

### Database Schema

**Archive Database Structure**:
```rust
// Hot storage (recent data)
CF_BLOCKS         // Block headers and bodies (last 30 days)
CF_TRANSACTIONS   // Transaction data and receipts (last 30 days)
CF_STATE          // Current state and recent state changes
CF_MONOLITHS      // Recent monolith blocks
CF_INDICES        // Query optimization indices

// Cold storage (historical data)
CF_HISTORICAL_BLOCKS    // All historical blocks (compressed)
CF_HISTORICAL_TXS       // All historical transactions (compressed)
CF_TIME_SERIES          // Time-series indices
CF_ADDRESS_INDEX        // Address-based indices
CF_CONTRACT_INDEX       // Contract-based indices
CF_EVENT_INDEX          // Event-based indices
```

### Data Compression Strategy

**Compression Algorithms**:
```rust
pub enum CompressionType {
    // Hot storage - speed optimized
    LZ4,      // Fast compression, good ratio
    
    // Cold storage - space optimized
    ZSTD,     // Best compression ratio, slower
    GZIP,     // Compatible, moderate ratio
}

impl CompressionStrategy {
    pub fn compress_data(&self, data: &[u8]) -> Result<CompressedData, CompressionError> {
        match self.algorithm {
            CompressionType::LZ4 => self.compress_lz4(data),
            CompressionType::ZSTD => self.compress_zstd(data),
            CompressionType::GZIP => self.compress_gzip(data),
        }
    }
    
    pub fn decompress_data(&self, compressed: &[u8]) -> Result<Vec<u8>, CompressionError> {
        match self.algorithm {
            CompressionType::LZ4 => self.decompress_lz4(compressed),
            CompressionType::ZSTD => self.decompress_zstd(compressed),
            CompressionType::GZIP => self.decompress_gzip(compressed),
        }
    }
}
```

## Query System

### Query Engine Architecture

```rust
pub struct ArchiveQueryEngine {
    // Query components
    pub parser: Arc<QueryParser>,              // SQL-like query parser
    pub planner: Arc<QueryPlanner>,            // Query execution planner
    pub executor: Arc<QueryExecutor>,          // Query execution engine
    
    // Data access
    pub hot_storage: Arc<HotStorage>,          // Recent data access
    pub cold_storage: Arc<ColdStorage>,        // Historical data access
    
    // Performance optimization
    pub query_cache: Arc<QueryCache>,          // Query result caching
    pub index_manager: Arc<IndexManager>,      // Index management
}
```

### Query Language

**Archive Query Language (AQL)**:
```sql
-- Block queries
SELECT * FROM blocks WHERE height >= 1000000 AND height <= 1000100;
SELECT block_hash, timestamp FROM blocks WHERE timestamp >= '2023-01-01';

-- Transaction queries
SELECT * FROM transactions WHERE from_address = '0x...' AND value > 1000;
SELECT * FROM transactions WHERE contract_address = '0x...' AND method = 'transfer';

-- Address queries
SELECT * FROM address_activity WHERE address = '0x...' ORDER BY timestamp DESC;
SELECT balance FROM address_balances WHERE address = '0x...' AND block_height = 1500000;

-- Contract queries
SELECT * FROM contract_events WHERE contract_address = '0x...' AND event_name = 'Transfer';
SELECT * FROM contract_state WHERE contract_address = '0x...' AND slot = 'balance';

-- Aggregation queries
SELECT COUNT(*) FROM transactions WHERE block_height >= 1000000;
SELECT SUM(value) FROM transactions WHERE from_address = '0x...';
SELECT AVG(gas_price) FROM blocks WHERE timestamp >= '2023-01-01';
```

### Index Management

**Index Types**:
```rust
pub enum IndexType {
    // Time-based indices
    BlockTimeIndex,      // Block timestamp -> block hash
    TransactionTimeIndex, // Transaction timestamp -> tx hash
    
    // Address-based indices
    AddressIndex,         // Address -> transaction hashes
    BalanceIndex,         // Address -> balance history
    
    // Contract-based indices
    ContractIndex,        // Contract address -> transaction hashes
    EventIndex,          // Contract address -> events
    StateIndex,           // Contract address -> state changes
    
    // Custom indices
    TokenTransferIndex,   // Token transfers
    NFTTransferIndex,     // NFT transfers
    GovernanceIndex,      // Governance actions
}
```

**Index Implementation**:
```rust
impl IndexManager {
    pub fn create_index(&mut self, index_type: IndexType) -> Result<IndexHandle, IndexError> {
        match index_type {
            IndexType::BlockTimeIndex => {
                let index = TimeSeriesIndex::new("block_time", "timestamp", "block_hash")?;
                self.indices.insert("block_time", index);
                Ok(IndexHandle::new("block_time"))
            },
            IndexType::AddressIndex => {
                let index = MultiValueIndex::new("address", "address", "tx_hash")?;
                self.indices.insert("address", index);
                Ok(IndexHandle::new("address"))
            },
            // ... other index types
        }
    }
    
    pub fn query_index(&self, index_name: &str, query: &IndexQuery) -> Result<Vec<IndexEntry>, IndexError> {
        let index = self.indices.get(index_name)
            .ok_or(IndexError::IndexNotFound)?;
        
        index.query(query)
    }
}
```

## Data Synchronization

### Sync Strategy

**Initial Sync**:
```rust
impl ArchiveNode {
    pub async fn initial_sync(&self) -> Result<SyncResult, SyncError> {
        // 1. Connect to multiple peers
        let peers = self.p2p.connect_to_peers(50).await?;
        
        // 2. Download block headers
        let headers = self.download_block_headers(0, self.current_height()).await?;
        
        // 3. Validate chain integrity
        self.validate_chain_integrity(&headers)?;
        
        // 4. Download blocks in parallel
        let block_batches = self.create_block_batches(&headers, 1000);
        let download_tasks = block_batches.into_iter()
            .map(|batch| self.download_block_batch(batch));
        
        let results = futures::future::join_all(download_tasks).await;
        
        // 5. Store blocks with compression
        for result in results {
            let blocks = result?;
            self.store_blocks_compressed(&blocks).await?;
        }
        
        // 6. Build indices
        self.build_historical_indices().await?;
        
        Ok(SyncResult {
            blocks_synced: headers.len(),
            indices_built: self.get_index_count(),
            sync_time: Instant::now() - sync_start,
        })
    }
}
```

**Incremental Sync**:
```rust
impl ArchiveNode {
    pub async fn incremental_sync(&self) -> Result<SyncResult, SyncError> {
        let current_height = self.get_current_height()?;
        let latest_height = self.get_latest_height_from_peers().await?;
        
        if latest_height > current_height {
            // Sync missing blocks
            let missing_blocks = self.download_blocks_range(current_height + 1, latest_height).await?;
            self.store_blocks_compressed(&missing_blocks).await?;
            
            // Update indices
            self.update_indices(&missing_blocks).await?;
            
            // Move old blocks to cold storage
            self.archive_old_blocks().await?;
        }
        
        Ok(SyncResult {
            blocks_synced: latest_height - current_height,
            indices_updated: self.get_updated_index_count(),
            sync_time: Instant::now() - sync_start,
        })
    }
}
```

### Data Archival

**Hot to Cold Migration**:
```rust
impl ArchiveNode {
    pub async fn archive_old_blocks(&self) -> Result<ArchivalResult, ArchivalError> {
        let cutoff_height = self.get_current_height() - 30 * 24 * 60; // 30 days ago
        
        // 1. Identify blocks to archive
        let blocks_to_archive = self.get_blocks_before_height(cutoff_height)?;
        
        // 2. Compress and move to cold storage
        for block in blocks_to_archive {
            let compressed = self.compress_block(&block)?;
            self.cold_storage.store_block(block.height, compressed).await?;
            self.hot_storage.remove_block(block.height).await?;
        }
        
        // 3. Update index references
        self.update_index_references(&blocks_to_archive).await?;
        
        // 4. Compact hot storage
        self.hot_storage.compact().await?;
        
        Ok(ArchivalResult {
            blocks_archived: blocks_to_archive.len(),
            space_saved: self.calculate_space_saved(),
            compression_ratio: self.get_compression_ratio(),
        })
    }
}
```

## API Services

### REST API

**Historical Data Endpoints**:
```rust
// Block endpoints
GET /api/v1/blocks/{height}
GET /api/v1/blocks/{hash}
GET /api/v1/blocks/range/{from}/{to}
GET /api/v1/blocks/timestamp/{timestamp}

// Transaction endpoints
GET /api/v1/transactions/{hash}
GET /api/v1/transactions/address/{address}
GET /api/v1/transactions/block/{height}
GET /api/v1/events/contract/{address}

// Address endpoints
GET /api/v1/addresses/{address}/balance
GET /api/v1/addresses/{address}/transactions
GET /api/v1/addresses/{address}/events
GET /api/v1/addresses/{address}/contracts

// Contract endpoints
GET /api/v1/contracts/{address}/state
GET /api/v1/contracts/{address}/events
GET /api/v1/contracts/{address}/transactions
GET /api/v1/contracts/{address}/storage

// Query endpoints
POST /api/v1/query/sql
GET /api/v1/query/{query_id}
DELETE /api/v1/query/{query_id}
```

### GraphQL API

**GraphQL Schema**:
```graphql
type Query {
  # Block queries
  block(height: Int!): Block
  blockByHash(hash: String!): Block
  blocks(fromHeight: Int!, toHeight: Int!): [Block!]!
  
  # Transaction queries
  transaction(hash: String!): Transaction
  transactionsByAddress(address: String!, from: Int, limit: Int): [Transaction!]!
  
  # Address queries
  address(address: String!): Address
  addressBalance(address: String!, height: Int): BigInt!
  addressTransactions(address: String!, from: Int, limit: Int): [Transaction!]!
  
  # Contract queries
  contract(address: String!): Contract
  contractEvents(address: String!, eventName: String, from: Int, limit: Int): [Event!]!
  
  # Analytics queries
  dailyStats(date: String!): DailyStats!
  hourlyStats(date: String!): [HourlyStats!]!
  topContracts(period: String!, limit: Int): [Contract!]!
}

type Subscription {
  # Real-time updates
  newBlocks: Block!
  newTransactions: Transaction!
  contractEvents(contract: String!): Event!
}
```

## Performance Optimization

### Query Optimization

**Query Planning**:
```rust
impl QueryPlanner {
    pub fn plan_query(&self, query: &ParsedQuery) -> Result<ExecutionPlan, PlanningError> {
        let mut plan = ExecutionPlan::new();
        
        // 1. Analyze query predicates
        let predicates = self.analyze_predicates(&query.where_clause)?;
        
        // 2. Select optimal indices
        let indices = self.select_indices(&predicates)?;
        
        // 3. Determine data sources
        let data_sources = self.select_data_sources(&query, &indices)?;
        
        // 4. Optimize join order
        let join_order = self.optimize_joins(&query, &data_sources)?;
        
        // 5. Add caching hints
        let cache_hints = self.generate_cache_hints(&query)?;
        
        plan.add_steps(indices);
        plan.add_steps(data_sources);
        plan.add_steps(join_order);
        plan.add_cache_hints(cache_hints);
        
        Ok(plan)
    }
}
```

### Caching Strategy

**Multi-Level Caching**:
```rust
pub struct ArchiveCache {
    // Level 1: In-memory cache (frequently accessed data)
    pub l1_cache: Arc<LruCache<CacheKey, CacheValue>>,
    
    // Level 2: SSD cache (medium frequency data)
    pub l2_cache: Arc<SSDCache>,
    
    // Level 3: Compressed cache (infrequent data)
    pub l3_cache: Arc<CompressedCache>,
    
    // Cache statistics
    pub stats: Arc<CacheStats>,
}

impl ArchiveCache {
    pub fn get(&self, key: &CacheKey) -> Option<CacheValue> {
        // Try L1 cache first
        if let Some(value) = self.l1_cache.get(key) {
            self.stats.record_hit(CacheLevel::L1);
            return Some(value);
        }
        
        // Try L2 cache
        if let Some(value) = self.l2_cache.get(key) {
            self.stats.record_hit(CacheLevel::L2);
            // Promote to L1
            self.l1_cache.put(key.clone(), value.clone());
            return Some(value);
        }
        
        // Try L3 cache
        if let Some(compressed) = self.l3_cache.get(key) {
            self.stats.record_hit(CacheLevel::L3);
            let value = self.decompress(&compressed)?;
            // Promote to L2 and L1
            self.l2_cache.put(key.clone(), value.clone());
            self.l1_cache.put(key.clone(), value);
            return Some(value);
        }
        
        self.stats.record_miss();
        None
    }
}
```

## Data Analytics

### Analytics Engine

**Time-Series Analytics**:
```rust
pub struct AnalyticsEngine {
    pub time_series: Arc<TimeSeriesDB>,
    pub aggregations: Arc<AggregationEngine>,
    pub reports: Arc<ReportGenerator>,
}

impl AnalyticsEngine {
    pub fn analyze_transaction_volume(&self, period: TimePeriod) -> Result<TransactionVolumeReport, AnalyticsError> {
        let query = TimeSeriesQuery {
            metric: "transaction_count",
            aggregation: "sum",
            period: period,
            filters: vec![],
        };
        
        let data = self.time_series.query(&query)?;
        
        Ok(TransactionVolumeReport {
            period,
            total_transactions: data.sum(),
            average_per_day: data.mean(),
            peak_volume: data.max(),
            trend: self.calculate_trend(&data),
        })
    }
    
    pub fn analyze_network_activity(&self, period: TimePeriod) -> Result<NetworkActivityReport, AnalyticsError> {
        let mut report = NetworkActivityReport::new(period);
        
        // Transaction volume
        report.transaction_volume = self.analyze_transaction_volume(period)?;
        
        // Active addresses
        report.active_addresses = self.analyze_active_addresses(period)?;
        
        // Gas usage
        report.gas_usage = self.analyze_gas_usage(period)?;
        
        // Block production
        report.block_production = self.analyze_block_production(period)?;
        
        // Network health
        report.network_health = self.analyze_network_health(period)?;
        
        Ok(report)
    }
}
```

### Custom Reports

**Report Templates**:
```rust
pub struct ReportTemplate {
    pub name: String,
    pub description: String,
    pub queries: Vec<ReportQuery>,
    pub visualizations: Vec<VisualizationConfig>,
}

impl ReportGenerator {
    pub fn generate_report(&self, template: &ReportTemplate, parameters: &ReportParameters) -> Result<Report, ReportError> {
        let mut report = Report::new(template.name.clone());
        
        // Execute queries
        for query in &template.queries {
            let result = self.execute_query(query, parameters)?;
            report.add_data(query.name.clone(), result);
        }
        
        // Generate visualizations
        for viz in &template.visualizations {
            let chart = self.generate_visualization(viz, &report.data)?;
            report.add_visualization(viz.name.clone(), chart);
        }
        
        // Add metadata
        report.metadata = ReportMetadata {
            generated_at: Utc::now(),
            parameters: parameters.clone(),
            template_version: template.version.clone(),
        };
        
        Ok(report)
    }
}
```

## Backup and Recovery

### Backup Strategy

**Incremental Backup**:
```rust
impl BackupManager {
    pub async fn create_incremental_backup(&self, since: Timestamp) -> Result<BackupResult, BackupError> {
        let mut backup = IncrementalBackup::new();
        
        // 1. Identify changed data
        let changed_blocks = self.get_changed_blocks(since).await?;
        let changed_transactions = self.get_changed_transactions(since).await?;
        let changed_state = self.get_changed_state(since).await?;
        
        // 2. Compress changed data
        backup.blocks = self.compress_data(&changed_blocks)?;
        backup.transactions = self.compress_data(&changed_transactions)?;
        backup.state = self.compress_data(&changed_state)?;
        
        // 3. Create backup manifest
        backup.manifest = BackupManifest {
            backup_type: BackupType::Incremental,
            since_timestamp: since,
            created_at: Utc::now(),
            data_checksums: self.calculate_checksums(&backup),
        };
        
        // 4. Store backup
        let backup_path = self.store_backup(&backup).await?;
        
        Ok(BackupResult {
            backup_path,
            size: backup.get_total_size(),
            compression_ratio: backup.get_compression_ratio(),
        })
    }
}
```

### Recovery Procedures

**Point-in-Time Recovery**:
```rust
impl RecoveryManager {
    pub async fn recover_to_point_in_time(&self, target_time: Timestamp) -> Result<RecoveryResult, RecoveryError> {
        // 1. Find appropriate backup
        let backup = self.find_backup_before_time(target_time)?;
        
        // 2. Restore from backup
        self.restore_from_backup(&backup).await?;
        
        // 3. Sync to target time
        let current_height = self.get_current_height()?;
        let target_height = self.get_height_at_time(target_time)?;
        
        if current_height > target_height {
            // Rollback to target height
            self.rollback_to_height(target_height).await?;
        } else {
            // Sync forward to target height
            self.sync_to_height(target_height).await?;
        }
        
        // 4. Verify data integrity
        self.verify_data_integrity().await?;
        
        Ok(RecoveryResult {
            recovered_height: target_height,
            recovered_time: target_time,
            verification_passed: true,
        })
    }
}
```

## Monitoring and Maintenance

### Archive-Specific Metrics

**Storage Metrics**:
```rust
pub struct ArchiveMetrics {
    // Storage utilization
    pub hot_storage_used: u64,
    pub cold_storage_used: u64,
    pub compression_ratio: f64,
    pub deduplication_ratio: f64,
    
    // Data access patterns
    pub hot_storage_hits: u64,
    pub cold_storage_hits: u64,
    pub cache_hit_rate: f64,
    pub query_latency: Duration,
    
    // Archive operations
    pub blocks_archived: u64,
    pub archive_operations: u64,
    pub index_updates: u64,
    pub backup_operations: u64,
}
```

### Maintenance Tasks

**Periodic Maintenance**:
```rust
impl ArchiveNode {
    pub async fn perform_maintenance(&self) -> Result<MaintenanceResult, MaintenanceError> {
        let mut result = MaintenanceResult::new();
        
        // 1. Archive old blocks
        let archive_result = self.archive_old_blocks().await?;
        result.blocks_archived = archive_result.blocks_archived;
        
        // 2. Update indices
        let index_result = self.update_indices().await?;
        result.indices_updated = index_result.count;
        
        // 3. Compact storage
        let compact_result = self.compact_storage().await?;
        result.space_freed = compact_result.space_freed;
        
        // 4. Create backup
        let backup_result = self.create_incremental_backup(self.last_backup_time).await?;
        result.backup_created = true;
        
        // 5. Update statistics
        self.update_metrics().await?;
        
        Ok(result)
    }
}
```

This archive node specification provides comprehensive guidance for deploying and operating Savitri Network archive nodes with optimal historical data management, query performance, and analytical capabilities.
