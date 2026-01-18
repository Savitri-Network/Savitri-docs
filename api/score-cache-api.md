# Score Cache API Reference

## Overview

The Score Cache system provides high-performance caching for PoU (Proof of Unity) scores with thread-safe operations, LRU eviction, and TTL-based expiration. It's designed to optimize transaction scheduling by avoiding redundant score computations.

## Core Types

### ScoreCache

```rust
pub struct ScoreCache {
    cache: HashMap<(u64, TxClass), CacheEntry>,
    max_size: usize,
    ttl_blocks: u64,
    hits: AtomicU64,
    misses: AtomicU64,
    current_block: AtomicU64,
}

struct CacheEntry {
    score: f64,
    block_created: u64,
    last_accessed: u64,
}
```

### Configuration

```rust
pub struct ScoreCacheConfig {
    pub max_size: usize,        // Maximum cache entries (default: 10_000)
    pub ttl_blocks: u64,        // TTL in blocks (default: 100)
    pub enable_cache: bool,     // Enable/disable caching (default: true)
}
```

## API Methods

### Core Operations

#### get_cached_score
Retrieve a cached score if available and not expired.

```rust
pub fn get_cached_score(&self, sender_id: u64, class: TxClass) -> Option<f64>
```

**Parameters:**
- `sender_id: u64` - Sender identifier
- `class: TxClass` - Transaction class

**Returns:**
- `Option<f64>` - Cached score if available and valid

**Example:**
```rust
let cache = ScoreCache::new(config);

if let Some(score) = cache.get_cached_score(sender_id, TxClass::Payment) {
    println!("Cached score: {}", score);
} else {
    // Compute score and cache it
    let computed_score = compute_score(sender_id, TxClass::Payment);
    cache.cache_score(sender_id, TxClass::Payment, computed_score);
}
```

#### cache_score
Store a score in the cache with LRU eviction if needed.

```rust
pub fn cache_score(&mut self, sender_id: u64, class: TxClass, score: f64)
```

**Parameters:**
- `sender_id: u64` - Sender identifier
- `class: TxClass` - Transaction class
- `score: f64` - Score to cache

**Behavior:**
- Updates existing entry or creates new one
- Evicts oldest entries if cache is full
- Updates access time for LRU tracking

#### cleanup_cache
Remove expired entries based on TTL.

```rust
pub fn cleanup_cache(&mut self, current_block: u64)
```

**Parameters:**
- `current_block: u64` - Current block height for TTL calculation

**Behavior:**
- Removes entries older than TTL
- Updates internal block height
- Thread-safe operation

#### clear_cache
Remove all entries from the cache.

```rust
pub fn clear_cache(&mut self)
```

**Behavior:**
- Clears all cache entries
- Resets statistics
- Thread-safe operation

### Statistics

#### get_cache_stats
Get cache performance statistics.

```rust
pub fn get_cache_stats(&self) -> CacheStats
```

**Returns:**
```rust
pub struct CacheStats {
    pub hits: u64,              // Cache hits
    pub misses: u64,            // Cache misses
    pub hit_rate: f64,          // Hit rate (0.0-1.0)
    pub size: usize,            // Current cache size
    pub max_size: usize,        // Maximum cache size
    pub current_block: u64,     // Current block height
}
```

**Example:**
```rust
let stats = cache.get_cache_stats();
println!("Cache hit rate: {:.2}%", stats.hit_rate * 100.0);
println!("Cache size: {}/{}", stats.size, stats.max_size);
```

#### get_hit_rate
Calculate current hit rate.

```rust
pub fn get_hit_rate(&self) -> f64
```

**Returns:**
- `f64` - Hit rate between 0.0 and 1.0

## Thread-Safe Operations

### Arc<Mutex<ScoreCache>>

For concurrent access, wrap the cache in Arc<Mutex<>>:

```rust
use std::sync::{Arc, Mutex};
use tokio::sync::Mutex as AsyncMutex;

let cache = Arc::new(Mutex::new(ScoreCache::new(config)));

// Thread-safe get operation
let score = {
    let cache = cache.lock().unwrap();
    cache.get_cached_score(sender_id, class)
};

// Thread-safe cache update
{
    let mut cache = cache.lock().unwrap();
    cache.cache_score(sender_id, class, computed_score);
}
```

### Async Operations

For async contexts, use AsyncMutex:

```rust
use tokio::sync::Mutex;

let cache = Arc::new(AsyncMutex::new(ScoreCache::new(config)));

// Async get operation
let score = {
    let cache = cache.lock().await;
    cache.get_cached_score(sender_id, class)
};

// Async cache update
{
    let mut cache = cache.lock().await;
    cache.cache_score(sender_id, class, computed_score);
}
```

## Integration with ExecutionDispatcher

### Cache-Aware Scheduling

```rust
impl ExecutionDispatcher {
    pub fn schedule_transactions(
        &mut self,
        mempool_txs: Vec<MempoolTx>,
        signed_txs: Vec<SignedTx>,
    ) -> (Vec<MempoolTx>, Vec<SignedTx>) {
        // 1. Check cache for known transactions
        let mut cached_scores = Vec::new();
        let mut uncached_indices = Vec::new();
        
        for (i, tx) in mempool_txs.iter().enumerate() {
            if let Some(score) = self.score_cache.get_cached_score(tx.sender_id, tx.class) {
                cached_scores.push((i, score));
            } else {
                uncached_indices.push(i);
            }
        }
        
        // 2. Compute scores for uncached transactions
        let computed_scores = self.compute_scores_batch(&uncached_indices, &mempool_txs);
        
        // 3. Cache new scores
        for (idx, score) in uncached_indices.iter().zip(computed_scores.iter()) {
            let tx = &mempool_txs[*idx];
            self.score_cache.cache_score(tx.sender_id, tx.class, *score);
        }
        
        // 4. Combine and sort
        let mut all_scores = cached_scores;
        all_scores.extend(uncached_indices.iter().zip(computed_scores.iter()));
        all_scores.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());
        
        // 5. Select optimal transactions
        self.select_transactions_by_scores(&mempool_txs, &all_scores)
    }
}
```

### Configuration Integration

```rust
impl ExecutionDispatcher {
    pub fn configure_cache(&mut self, config: ScoreCacheConfig) {
        self.score_cache = ScoreCache::new(config);
    }
    
    pub fn get_cache_performance(&self) -> CachePerformance {
        let stats = self.score_cache.get_cache_stats();
        CachePerformance {
            hit_rate: stats.hit_rate,
            total_operations: stats.hits + stats.misses,
            cache_efficiency: self.calculate_efficiency(&stats),
        }
    }
}
```

## Performance Optimization

### Batch Operations

```rust
impl ScoreCache {
    pub fn get_cached_scores_batch(&self, requests: &[(u64, TxClass)]) -> Vec<Option<f64>> {
        requests
            .iter()
            .map(|(sender_id, class)| self.get_cached_score(*sender_id, *class))
            .collect()
    }
    
    pub fn cache_scores_batch(&mut self, entries: &[(u64, TxClass, f64)]) {
        for (sender_id, class, score) in entries {
            self.cache_score(*sender_id, *class, *score);
        }
    }
}
```

### Memory Management

```rust
impl ScoreCache {
    pub fn optimize_memory(&mut self) {
        // Remove expired entries
        self.cleanup_cache(self.current_block.load(Ordering::Relaxed));
        
        // If still over size limit, remove least recently used
        if self.cache.len() > self.max_size {
            self.evict_lru_entries(self.cache.len() - self.max_size);
        }
    }
    
    fn evict_lru_entries(&mut self, count: usize) {
        let mut entries_by_access: Vec<_> = self.cache
            .iter()
            .map(|(&(sender_id, class), entry)| {
                (sender_id, class, entry.last_accessed)
            })
            .collect();
        
        entries_by_access.sort_by_key(|(_, _, accessed)| *accessed);
        
        for (sender_id, class, _) in entries_by_access.into_iter().take(count) {
            self.cache.remove(&(sender_id, class));
        }
    }
}
```

## Monitoring and Metrics

### Prometheus Metrics

```rust
use prometheus::{IntCounter, Histogram, Gauge};

lazy_static! {
    static ref CACHE_HITS: IntCounter = IntCounter::new(
        "score_cache_hits_total", 
        "Total cache hits"
    ).unwrap();
    
    static ref CACHE_MISSES: IntCounter = IntCounter::new(
        "score_cache_misses_total", 
        "Total cache misses"
    ).unwrap();
    
    static ref CACHE_SIZE: Gauge = Gauge::new(
        "score_cache_size", 
        "Current cache size"
    ).unwrap();
    
    static ref CACHE_HIT_RATE: Gauge = Gauge::new(
        "score_cache_hit_rate", 
        "Cache hit rate"
    ).unwrap();
}

impl ScoreCache {
    pub fn update_metrics(&self) {
        let stats = self.get_cache_stats();
        
        CACHE_HITS.set(stats.hits);
        CACHE_MISSES.set(stats.misses);
        CACHE_SIZE.set(stats.size as i64);
        CACHE_HIT_RATE.set(stats.hit_rate);
    }
}
```

### Performance Alerts

```rust
impl ScoreCache {
    pub fn check_performance(&self) -> Vec<PerformanceAlert> {
        let stats = self.get_cache_stats();
        let mut alerts = Vec::new();
        
        // Low hit rate alert
        if stats.hit_rate < 0.7 {
            alerts.push(PerformanceAlert {
                level: AlertLevel::Warning,
                message: "Low cache hit rate detected".to_string(),
                value: stats.hit_rate,
                threshold: 0.7,
            });
        }
        
        // High memory usage alert
        let memory_usage = stats.size as f64 / stats.max_size as f64;
        if memory_usage > 0.9 {
            alerts.push(PerformanceAlert {
                level: AlertLevel::Critical,
                message: "Cache memory usage high".to_string(),
                value: memory_usage,
                threshold: 0.9,
            });
        }
        
        alerts
    }
}
```

## Error Handling

### Cache Errors

```rust
#[derive(Debug, thiserror::Error)]
pub enum CacheError {
    #[error("Cache is disabled")]
    Disabled,
    
    #[error("Cache entry expired")]
    Expired,
    
    #[error("Cache size limit exceeded")]
    SizeLimitExceeded,
    
    #[error("Invalid cache key")]
    InvalidKey,
    
    #[error("Cache operation failed: {0}")]
    OperationFailed(String),
}
```

### Graceful Degradation

```rust
impl ScoreCache {
    pub fn get_cached_score_safe(&self, sender_id: u64, class: TxClass) -> Result<Option<f64>, CacheError> {
        if !self.enable_cache {
            return Err(CacheError::Disabled);
        }
        
        Ok(self.get_cached_score(sender_id, class))
    }
    
    pub fn cache_score_safe(&mut self, sender_id: u64, class: TxClass, score: f64) -> Result<(), CacheError> {
        if !self.enable_cache {
            return Err(CacheError::Disabled);
        }
        
        self.cache_score(sender_id, class, score);
        Ok(())
    }
}
```

## Best Practices

### 1. Cache Sizing
- Use 10,000 entries for production workloads
- Adjust based on memory constraints
- Monitor hit rate to optimize size

### 2. TTL Configuration
- Use 100 blocks for typical workloads
- Shorter TTL for high volatility
- Longer TTL for stable patterns

### 3. Thread Safety
- Always use Arc<Mutex<>> for concurrent access
- Minimize lock duration
- Consider lock-free alternatives for high contention

### 4. Performance Monitoring
- Track hit rates continuously
- Set up alerts for performance degradation
- Regular memory usage audits

### 5. Error Handling
- Implement graceful degradation
- Log cache errors for debugging
- Provide fallback mechanisms

This API provides a robust, high-performance caching solution for PoU score optimization in production environments.
