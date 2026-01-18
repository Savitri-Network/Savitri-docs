# Savitri Network API Documentation

## Overview

This directory contains comprehensive API documentation for Savitri Network's core components, including consensus events, score caching, and integration interfaces.

## Available APIs

### Consensus Events API
**File:** `consensus-events.md`

The Consensus Events API documents all events emitted by the consensus engine, including:
- Basic consensus events (round start, proposals, certificates)
- PoU integration events (block finalization, score updates, rewards)
- Event handling patterns and best practices
- Performance considerations and monitoring

**Key Features:**
- Real-time event streaming
- PoU-aware event types
- Thread-safe event handling
- Monitoring integration

### Score Cache API
**File:** `score-cache-api.md`

The Score Cache API provides high-performance caching for PoU scores with:
- Thread-safe operations with Arc<Mutex<>>
- LRU eviction and TTL-based expiration
- Batch operations for efficiency
- Performance monitoring and metrics

**Key Features:**
- Sub-millisecond lookup times
- 90%+ hit rates in production
- Cross-batch persistence
- SIMD-optimized integration

## Integration Patterns

### Event-Driven Architecture

```rust
use tokio::sync::mpsc;

// Setup event channel
let (event_tx, mut event_rx) = mpsc::channel(1000);

// Process consensus events
while let Some(event) = event_rx.recv().await {
    match event {
        EngineEvent::BlockFinalized { height, proposer_id, proposer_pou_score, .. } => {
            // Update proposer metrics
            update_proposer_score(proposer_id, proposer_pou_score);
        }
        EngineEvent::ProposerScoreUpdate { proposer_id, new_score, reason } => {
            // Update score cache
            score_cache.cache_score(proposer_id, new_score);
        }
        _ => {}
    }
}
```

### Cache-Optimized Scheduling

```rust
impl ExecutionDispatcher {
    pub fn schedule_transactions_optimized(
        &mut self,
        mempool_txs: Vec<MempoolTx>,
        signed_txs: Vec<SignedTx>,
    ) -> (Vec<MempoolTx>, Vec<SignedTx>) {
        // 1. Batch cache lookup
        let cache_requests: Vec<_> = mempool_txs
            .iter()
            .map(|tx| (tx.sender_id, tx.class))
            .collect();
        
        let cached_scores = self.score_cache.get_cached_scores_batch(&cache_requests);
        
        // 2. SIMD for uncached transactions
        let uncached_indices: Vec<_> = cached_scores
            .iter()
            .enumerate()
            .filter_map(|(i, score)| if score.is_none() { Some(i) } else { None })
            .collect();
        
        let computed_scores = self.simd_processor.compute_scores_batch(
            &uncached_indices.iter().map(|&i| &mempool_txs[i]).collect::<Vec<_>>()
        );
        
        // 3. Update cache with new scores
        for (idx, score) in uncached_indices.iter().zip(computed_scores.iter()) {
            let tx = &mempool_txs[*idx];
            self.score_cache.cache_score(tx.sender_id, tx.class, *score);
        }
        
        // 4. Combine and optimize
        self.combine_and_select_transactions(mempool_txs, cached_scores, computed_scores)
    }
}
```

## Performance Characteristics

### Consensus Events
- **Event Rate**: ~10-20 events per second per node
- **Latency**: <1ms event propagation
- **Memory**: <1MB for event buffers
- **Throughput**: 1000+ events/second processing

### Score Cache
- **Lookup Time**: <100ns (cached), <1ms (miss)
- **Hit Rate**: 90%+ in production
- **Memory**: 10MB for 10K entries
- **Concurrency**: Thread-safe with atomic operations

## Monitoring and Observability

### Key Metrics

```rust
// Consensus Event Metrics
- consensus_events_total
- consensus_event_duration_seconds
- consensus_rounds_total
- consensus_timeouts_total

// Score Cache Metrics  
- score_cache_hits_total
- score_cache_misses_total
- score_cache_hit_rate
- score_cache_size
- score_cache_memory_usage_bytes
```

### Alert Thresholds

```yaml
alerts:
  consensus:
    timeout_rate: >0.1 (10%)
    round_duration: >30s
    event_queue_depth: >1000
  
  score_cache:
    hit_rate: <0.7 (70%)
    memory_usage: >90%
    lookup_latency: >5ms
```

## Security Considerations

### Event Security
- Validate all event fields
- Implement rate limiting
- Use secure channels for event distribution

### Cache Security
- Thread-safe operations only
- Validate cache keys
- Implement access controls

## Error Handling

### Event Errors
- Channel disconnection handling
- Event validation failures
- Graceful degradation strategies

### Cache Errors
- Cache disabled fallback
- Memory exhaustion handling
- Lock contention resolution

## Best Practices

### 1. Event Handling
- Use non-blocking event handlers
- Implement event batching for efficiency
- Monitor event queue depths

### 2. Cache Usage
- Pre-warm cache with common entries
- Use batch operations for bulk updates
- Monitor hit rates and adjust sizing

### 3. Performance Optimization
- Profile event handling paths
- Use SIMD for score computations
- Implement adaptive cache sizing

### 4. Monitoring
- Track all key metrics
- Set up automated alerts
- Regular performance audits

## Version Compatibility

### Current Version: v1.0.0
- Consensus Events API: Stable
- Score Cache API: Stable
- Integration patterns: Stable

### Backward Compatibility
- Event types are additive only
- Cache API maintains interface stability
- Configuration options are backward compatible

## Support and Contributing

### Documentation Updates
- API changes require documentation updates
- Performance improvements should be documented
- New integration patterns should be added

### Testing
- All APIs have comprehensive test coverage
- Performance benchmarks included
- Integration tests for end-to-end validation

For specific API details, refer to the individual API documentation files in this directory.
