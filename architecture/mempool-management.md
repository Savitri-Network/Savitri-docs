# Mempool Management Architecture

## Mempool Overview

Savitri Network implements a sophisticated mempool management system with adaptive scheduling, priority-based ordering, and intelligent transaction selection. The mempool serves as the critical interface between user transactions and block production, ensuring optimal throughput and fair resource allocation.

## Technology Choice Rationale

### Why Adaptive Mempool Scheduling

**Problem Statement**: Traditional mempools use simple fee-based sorting which can lead to transaction starvation, poor network utilization, and suboptimal block composition under varying load conditions.

**Chosen Solution**: Adaptive weight scheduling with multi-dimensional transaction classification and dynamic priority adjustment.

**Rationale**:
- **Fairness**: Prevents transaction starvation through priority rotation
- **Efficiency**: Maximizes block space utilization through intelligent selection
- **Responsiveness**: Adapts to network conditions in real-time
- **Economic Efficiency**: Balances fee revenue with network health

**Expected Results**:
- 30-50% improvement in transaction processing efficiency
- Reduced average confirmation times
- Better network utilization under high load
- More predictable fee market behavior

### Why Multi-Dimensional Transaction Classification

**Problem Statement**: Fee-only classification fails to capture the true value and urgency of transactions, leading to suboptimal scheduling decisions.

**Chosen Solution**: Multi-dimensional classification system with fee, urgency, system importance, and governance considerations.

**Rationale**:
- **Granularity**: More nuanced transaction prioritization
- **System Health**: Prioritizes critical system operations
- **Governance**: Ensures governance transactions are processed
- **Economic Signals**: Maintains fee market efficiency

**Expected Results**:
- Better block composition with higher economic value
- Improved system responsiveness and reliability
- More predictable confirmation times for different transaction types
- Enhanced network security through prioritized system transactions

## Mempool Architecture

### Core Components
```rust
pub struct MempoolManager {
    pub transaction_pool: TransactionPool,     // Transaction storage
    pub scheduler: AdaptiveScheduler,         // Adaptive scheduling
    pub classifier: TransactionClassifier,     // Transaction classification
    pub validator: TransactionValidator,      // Transaction validation
    pub metrics: MempoolMetrics,              // Performance metrics
    pub config: MempoolConfig,                // Configuration
}

pub struct TransactionPool {
    pub pending_transactions: BTreeMap<TxPriority, VecDeque<SignedTx>>, // Priority-ordered queues
    pub replacement_pool: HashMap<Hash32, SignedTx>, // Transaction replacement
    pub eviction_candidates: LruCache<Hash32, SignedTx>, // Eviction candidates
    pub total_size: usize,                    // Total mempool size
    pub max_size: usize,                      // Maximum mempool size
}
```

### Transaction Classification System
```rust
#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord)]
pub enum TxClass {
    Financial,                                // High-value financial transactions
    System,                                   // System-critical operations
    Governance,                               // Governance proposals and votes
    Standard,                                 // Regular user transactions
}

#[derive(Debug, Clone)]
pub struct TxPriority {
    pub class: TxClass,                       // Transaction class
    pub fee_rate: f64,                        // Fee rate (tokens/gas)
    pub urgency: UrgencyLevel,                // Urgency level
    pub age: Duration,                        // Time in mempool
    pub gas_limit: u64,                       // Gas limit
    pub sender_reputation: f64,               // Sender reputation score
}

#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord)]
pub enum UrgencyLevel {
    Critical,                                 // Must be included immediately
    High,                                     // High priority
    Normal,                                   // Normal priority
    Low,                                      // Low priority
}

impl TransactionClassifier {
    pub fn classify_transaction(&self, tx: &SignedTx, state: &StateDB) -> TxClass {
        // Analyze call data for transaction type
        let call_analysis = self.analyze_call_data(&tx.call_data);
        
        match call_analysis {
            CallAnalysis::TokenTransfer => TxClass::Financial,
            CallAnalysis::SystemOperation => TxClass::System,
            CallAnalysis::GovernanceAction => TxClass::Governance,
            CallAnalysis::ContractCall => {
                // Further classification based on contract type
                self.classify_contract_call(&tx.call_data)
            },
            CallAnalysis::StandardTransfer => TxClass::Standard,
        }
    }
    
    fn analyze_call_data(&self, call_data: &[u8]) -> CallAnalysis {
        if call_data.len() < 4 {
            return CallAnalysis::StandardTransfer;
        }
        
        let selector = u32::from_le_bytes([
            call_data[0], call_data[1], call_data[2], call_data[3]
        ]);
        
        match selector {
            // System operation selectors
            0x00000001..=0x0000FFFF => CallAnalysis::SystemOperation,
            
            // Governance selectors
            0x10000001..=0x1000FFFF => CallAnalysis::GovernanceAction,
            
            // Financial operation selectors
            0x20000001..=0x2000FFFF => CallAnalysis::TokenTransfer,
            
            // Contract call selectors
            0x30000001..=0x3000FFFF => CallAnalysis::ContractCall,
            
            _ => CallAnalysis::StandardTransfer,
        }
    }
}
```

### Adaptive Scheduling Algorithm
```rust
pub struct AdaptiveScheduler {
    pub weights: AdaptiveWeights,              // Adaptive weights
    pub performance_history: VecDeque<SchedulingResult>, // Performance history
    pub adjustment_factor: f64,               // Weight adjustment factor
    pub rebalance_interval: Duration,          // Rebalance interval
}

#[derive(Debug, Clone)]
pub struct AdaptiveWeights {
    pub transaction_fee: f64,                 // Fee weight (0.0-1.0)
    pub system_priority: f64,                  // System priority weight
    pub financial_priority: f64,               // Financial priority weight
    pub governance_priority: f64,               // Governance priority weight
    pub age_factor: f64,                      // Age factor weight
}

impl AdaptiveScheduler {
    pub fn calculate_transaction_score(&self, tx: &SignedTx, priority: &TxPriority) -> f64 {
        let fee_score = self.calculate_fee_score(priority.fee_rate);
        let class_score = self.calculate_class_score(priority.class);
        let urgency_score = self.calculate_urgency_score(priority.urgency);
        let age_score = self.calculate_age_score(priority.age);
        let reputation_score = priority.sender_reputation;
        
        // Apply adaptive weights
        fee_score * self.weights.transaction_fee +
        class_score * self.weights.get_class_weight(priority.class) +
        urgency_score * 0.1 +
        age_score * self.weights.age_factor +
        reputation_score * 0.05
    }
    
    pub fn adjust_weights(&mut self, recent_performance: &SchedulingMetrics) {
        // Analyze recent performance to adjust weights
        let fee_efficiency = recent_performance.fee_revenue_per_block;
        let system_health = recent_performance.system_transaction_processing_rate;
        let governance_activity = recent_performance.governance_transaction_rate;
        
        // Adjust fee weight based on revenue efficiency
        if fee_efficiency < 0.8 {
            self.weights.transaction_fee = (self.weights.transaction_fee * 1.1).min(0.8);
        } else if fee_efficiency > 1.2 {
            self.weights.transaction_fee = (self.weights.transaction_fee * 0.9).max(0.3);
        }
        
        // Adjust system priority based on system health
        if system_health < 0.9 {
            self.weights.system_priority = (self.weights.system_priority * 1.05).min(0.4);
        } else if system_health > 0.95 {
            self.weights.system_priority = (self.weights.system_priority * 0.95).max(0.1);
        }
        
        // Adjust governance priority based on activity
        if governance_activity < 0.5 {
            self.weights.governance_priority = (self.weights.governance_priority * 1.1).min(0.3);
        }
        
        // Normalize weights to sum to 1.0
        self.normalize_weights();
    }
    
    fn normalize_weights(&mut self) {
        let total = self.weights.transaction_fee + 
                   self.weights.system_priority + 
                   self.weights.financial_priority + 
                   self.weights.governance_priority + 
                   self.weights.age_factor;
        
        if total > 0.0 {
            self.weights.transaction_fee /= total;
            self.weights.system_priority /= total;
            self.weights.financial_priority /= total;
            self.weights.governance_priority /= total;
            self.weights.age_factor /= total;
        }
    }
}
```

## Transaction Management

### Transaction Addition
```rust
impl MempoolManager {
    pub fn add_transaction(&mut self, tx: SignedTx, state: &StateDB) -> Result<TxAddResult, MempoolError> {
        // 1. Validate transaction
        self.validator.validate_transaction(&tx, state)?;
        
        // 2. Check for replacement
        if let Some(existing_tx) = self.transaction_pool.get_transaction_by_sender(&tx.signer) {
            if self.should_replace_transaction(&existing_tx, &tx)? {
                self.transaction_pool.replace_transaction(existing_tx.hash(), tx.clone());
                return Ok(TxAddResult::Replaced);
            } else {
                return Err(MempoolError::ReplacementRejected);
            }
        }
        
        // 3. Classify transaction
        let tx_class = self.classifier.classify_transaction(&tx, state);
        let priority = self.calculate_priority(&tx, tx_class);
        
        // 4. Check mempool capacity
        if self.transaction_pool.total_size >= self.transaction_pool.max_size {
            self.evict_low_priority_transactions()?;
        }
        
        // 5. Add to appropriate queue
        self.transaction_pool.add_transaction(priority, tx);
        
        // 6. Update metrics
        self.metrics.record_transaction_added(&tx_class);
        
        Ok(TxAddResult::Added)
    }
    
    fn should_replace_transaction(&self, existing: &SignedTx, new: &SignedTx) -> Result<bool, MempoolError> {
        // Replacement rules:
        // 1. Same sender and nonce
        if existing.signer != new.signer || existing.nonce != new.nonce {
            return Ok(false);
        }
        
        // 2. New transaction must have higher fee
        if new.fee <= existing.fee * 1.1 { // 10% fee bump required
            return Ok(false);
        }
        
        // 3. New transaction must not be significantly larger
        let size_increase = new.call_data.len().saturating_sub(existing.call_data.len());
        if size_increase > MAX_REPLACEMENT_SIZE_INCREASE {
            return Ok(false);
        }
        
        Ok(true)
    }
}
```

### Transaction Selection
```rust
impl MempoolManager {
    pub fn select_transactions_for_block(&mut self, block_gas_limit: u64) -> Vec<SignedTx> {
        let mut selected_transactions = Vec::new();
        let mut remaining_gas = block_gas_limit;
        
        // Get transactions from all priority queues
        let mut all_candidates = self.collect_candidate_transactions();
        
        // Sort by adaptive score
        all_candidates.sort_by(|a, b| {
            self.scheduler.calculate_transaction_score(&a.tx, &a.priority)
                .partial_cmp(&self.scheduler.calculate_transaction_score(&b.tx, &b.priority))
                .unwrap_or(std::cmp::Ordering::Equal)
        });
        
        // Select transactions until gas limit reached
        for candidate in all_candidates {
            if candidate.tx.gas_limit <= remaining_gas {
                selected_transactions.push(candidate.tx.clone());
                remaining_gas -= candidate.tx.gas_limit;
                
                // Remove from mempool
                self.transaction_pool.remove_transaction(&candidate.tx.hash());
            }
        }
        
        // Update metrics
        self.metrics.record_block_selection(&selected_transactions);
        
        selected_transactions
    }
    
    fn collect_candidate_transactions(&self) -> Vec<TransactionCandidate> {
        let mut candidates = Vec::new();
        
        // Collect from all priority queues
        for (priority, queue) in &self.transaction_pool.pending_transactions {
            for tx in queue {
                candidates.push(TransactionCandidate {
                    tx: tx.clone(),
                    priority: priority.clone(),
                    original_position: candidates.len(),
                });
            }
        }
        
        candidates
    }
}
```

## Performance Optimization

### Memory Management
```rust
pub struct MempoolMemoryManager {
    pub eviction_policy: EvictionPolicy,       // Eviction policy
    pub memory_threshold: f64,                // Memory usage threshold
    pub cleanup_interval: Duration,            // Cleanup interval
    pub compression_enabled: bool,             // Compression enabled
}

impl MempoolMemoryManager {
    pub fn optimize_memory_usage(&mut self, mempool: &mut TransactionPool) -> Result<(), MemoryError> {
        let memory_usage = self.calculate_memory_usage(mempool);
        
        if memory_usage > self.memory_threshold {
            // Apply eviction policy
            self.apply_eviction_policy(mempool)?;
        }
        
        // Compress old transactions if enabled
        if self.compression_enabled {
            self.compress_old_transactions(mempool)?;
        }
        
        // Cleanup expired transactions
        self.cleanup_expired_transactions(mempool)?;
        
        Ok(())
    }
    
    fn apply_eviction_policy(&self, mempool: &mut TransactionPool) -> Result<(), MemoryError> {
        let mut evicted_count = 0;
        let target_eviction = (mempool.total_size as f64 * 0.2) as usize; // Evict 20%
        
        // Evict lowest priority transactions first
        for (priority, queue) in &mut mempool.pending_transactions {
            while evicted_count < target_eviction && !queue.is_empty() {
                if let Some(tx) = queue.pop_back() {
                    evicted_count += 1;
                    mempool.total_size -= self.calculate_transaction_size(&tx);
                }
            }
        }
        
        Ok(())
    }
}
```

### Cache Optimization
```rust
pub struct MempoolCache {
    pub transaction_cache: LruCache<Hash32, CachedTransaction>, // Transaction cache
    pub priority_cache: LruCache<Hash32, TxPriority>,           // Priority cache
    pub validation_cache: LruCache<Hash32, ValidationResult>,    // Validation cache
    pub cache_hit_rate: f64,                                    // Cache hit rate
}

impl MempoolCache {
    pub fn get_cached_transaction(&mut self, tx_hash: &Hash32) -> Option<CachedTransaction> {
        self.transaction_cache.get(tx_hash).cloned()
    }
    
    pub fn cache_transaction(&mut self, tx_hash: Hash32, tx: CachedTransaction) {
        self.transaction_cache.put(tx_hash, tx);
    }
    
    pub fn calculate_cache_efficiency(&self) -> f64 {
        let total_requests = self.transaction_cache.hits() + self.transaction_cache.misses();
        if total_requests > 0 {
            self.transaction_cache.hits() as f64 / total_requests as f64
        } else {
            0.0
        }
    }
}
```

## Monitoring and Metrics

### Mempool Performance Metrics
```rust
pub struct MempoolMetrics {
    pub total_transactions: u64,               // Total transactions processed
    pub average_confirmation_time: Duration,    // Average confirmation time
    pub fee_revenue_per_block: f64,            // Fee revenue per block
    pub system_transaction_rate: f64,          // System transaction rate
    pub governance_transaction_rate: f64,      // Governance transaction rate
    pub eviction_rate: f64,                    // Transaction eviction rate
    pub replacement_rate: f64,                 // Transaction replacement rate
    pub cache_hit_rate: f64,                   // Cache hit rate
    pub memory_utilization: f64,               // Memory utilization
}

impl MempoolMetrics {
    pub fn calculate_health_score(&self) -> f64 {
        let confirmation_weight = 0.3;
        let revenue_weight = 0.25;
        let system_weight = 0.2;
        let governance_weight = 0.15;
        let efficiency_weight = 0.1;
        
        let confirmation_score = self.calculate_confirmation_score();
        let revenue_score = self.calculate_revenue_score();
        let system_score = self.system_transaction_rate;
        let governance_score = self.governance_transaction_rate;
        let efficiency_score = self.cache_hit_rate;
        
        confirmation_weight * confirmation_score +
        revenue_weight * revenue_score +
        system_weight * system_score +
        governance_weight * governance_score +
        efficiency_weight * efficiency_score
    }
}
```

### Real-time Monitoring
```rust
pub struct MempoolMonitor {
    pub metrics_collector: MetricsCollector,   // Metrics collection
    pub alert_thresholds: AlertThresholds,     // Alert thresholds
    pub performance_analyzer: PerformanceAnalyzer, // Performance analysis
}

impl MempoolMonitor {
    pub fn monitor_mempool_health(&mut self) -> HealthReport {
        let current_metrics = self.metrics_collector.collect_metrics();
        let health_score = current_metrics.calculate_health_score();
        
        HealthReport {
            overall_health: health_score,
            critical_issues: self.identify_critical_issues(&current_metrics),
            performance_trends: self.performance_analyzer.analyze_trends(&current_metrics),
            recommendations: self.generate_recommendations(&current_metrics),
            alerts: self.check_alert_conditions(&current_metrics),
        }
    }
    
    fn identify_critical_issues(&self, metrics: &MempoolMetrics) -> Vec<CriticalIssue> {
        let mut issues = Vec::new();
        
        if metrics.average_confirmation_time > Duration::from_secs(300) {
            issues.push(CriticalIssue::SlowConfirmation);
        }
        
        if metrics.memory_utilization > 0.9 {
            issues.push(CriticalIssue::HighMemoryUsage);
        }
        
        if metrics.eviction_rate > 0.3 {
            issues.push(CriticalIssue::HighEvictionRate);
        }
        
        if metrics.system_transaction_rate < 0.8 {
            issues.push(CriticalIssue::LowSystemTransactionRate);
        }
        
        issues
    }
}
```

## Configuration

### Mempool Configuration
```rust
pub struct MempoolConfig {
    pub max_size: usize,                      // Maximum mempool size
    pub max_transaction_size: usize,           // Maximum transaction size
    pub min_fee_rate: f64,                     // Minimum fee rate
    pub eviction_policy: EvictionPolicy,      // Eviction policy
    pub replacement_policy: ReplacementPolicy, // Replacement policy
    pub cache_config: CacheConfig,             // Cache configuration
    pub adaptive_scheduling: AdaptiveSchedulingConfig, // Adaptive scheduling
}

impl Default for MempoolConfig {
    fn default() -> Self {
        Self {
            max_size: 100_000,                 // 100K transactions
            max_transaction_size: 1_048_576,   // 1MB per transaction
            min_fee_rate: 0.001,               // 0.001 tokens/gas
            eviction_policy: EvictionPolicy::LowestPriority,
            replacement_policy: ReplacementPolicy::FeeBump10Percent,
            cache_config: CacheConfig::default(),
            adaptive_scheduling: AdaptiveSchedulingConfig::default(),
        }
    }
}
```

This mempool management architecture provides intelligent transaction handling with adaptive scheduling, ensuring optimal performance and fair resource allocation under varying network conditions.
