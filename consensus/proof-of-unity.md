# Proof of Unity (PoU) Consensus Algorithm

## Algorithm Overview

Proof of Unity (PoU) is a custom consensus scoring algorithm that combines multiple performance and behavioral metrics to determine validator selection and block proposal priority. Unlike traditional Proof of Stake, PoU emphasizes useful network participation and long-term contribution over pure token holdings.

## Mathematical Foundation

### Scoring Formula

The PoU score is computed as a weighted sum of five components:

$$Score = w_U \cdot U + w_L \cdot L + w_I \cdot I + w_R \cdot R + w_P \cdot P$$

Where:
- $U$ = Availability score (0-1)
- $L$ = Latency score (0-1)  
- $I$ = Integrity score (0-1)
- $R$ = Reputation score (0-1)
- $P$ = Participation score (0-1)
- $w_U, w_L, w_I, w_R, w_P$ = Weight coefficients (sum to 1.0)

### Default Weight Configuration
```rust
pub struct PoUWeights {
    pub availability: f64,      // w_U = 0.25
    pub latency: f64,          // w_L = 0.20
    pub integrity: f64,         // w_I = 0.25
    pub reputation: f64,       // w_R = 0.20
    pub participation: f64,    // w_P = 0.10
}
```

**Weight Rationale:**
- **Availability (25%)**: Critical for network stability
- **Integrity (25%)**: Ensures correct protocol behavior
- **Latency (20%)**: Important for performance
- **Reputation (20%)**: Long-term reliability indicator
- **Participation (10%)**: Encourages active engagement

## Component Scoring

### 1. Availability Score (U)

Measures node uptime and connectivity reliability.

$$U = \frac{uptime_{observed}}{uptime_{expected}} \cdot e^{-\lambda \cdot downtime_{incidents}}$$

**Implementation:**
```rust
pub fn calculate_availability_score(
    uptime_percentage: f64,
    downtime_incidents: u32,
    lambda: f64, // Decay factor = 0.1
) -> f64 {
    let base_score = uptime_percentage / 100.0;
    let penalty = f64::exp(-lambda * downtime_incidents as f64);
    (base_score * penalty).min(1.0)
}
```

**Metrics Tracked:**
- Uptime percentage over rolling 24-hour window
- Number of downtime incidents
- Average incident duration
- Network connectivity stability

### 2. Latency Score (L)

Measures response time and network performance.

$$L = \frac{1}{1 + \alpha \cdot \overline{latency} + \beta \cdot \sigma_{latency}}$$

**Implementation:**
```rust
pub fn calculate_latency_score(
    avg_latency_ms: f64,
    latency_stddev_ms: f64,
    alpha: f64, // 0.01
    beta: f64,  // 0.005
) -> f64 {
    let penalty = alpha * avg_latency_ms + beta * latency_stddev_ms;
    (1.0 / (1.0 + penalty)).max(0.0)
}
```

**Metrics Tracked:**
- Average response time for P2P messages
- Latency standard deviation
- 95th percentile latency
- Network round-trip times

### 3. Integrity Score (I)

Measures correctness of behavior and protocol compliance.

$$I = \frac{correct_{actions} - \gamma \cdot incorrect_{actions}}{total_{actions}}$$

**Implementation:**
```rust
pub fn calculate_integrity_score(
    correct_actions: u64,
    incorrect_actions: u64,
    gamma: f64, // Penalty factor = 2.0
) -> f64 {
    let total_actions = correct_actions + incorrect_actions;
    if total_actions == 0 {
        return 0.0;
    }
    
    let weighted_score = correct_actions as f64 - gamma * incorrect_actions as f64;
    (weighted_score / total_actions as f64).max(0.0).min(1.0)
}
```

**Actions Tracked:**
- Valid block proposals
- Correct vote submissions
- Protocol compliance
- Evidence submission accuracy

### 4. Reputation Score (R)

Long-term historical performance indicator.

$$R = \sum_{i=1}^{n} \delta^{i-1} \cdot Score_{historical_i}$$

**Implementation:**
```rust
pub fn calculate_reputation_score(
    historical_scores: &[f64],
    delta: f64, // Decay factor = 0.95
) -> f64 {
    historical_scores
        .iter()
        .enumerate()
        .map(|(i, &score)| delta.powi(i as i32) * score)
        .sum()
}
```

**Historical Data:**
- Daily PoU scores for past 90 days
- Major contribution events
- Consensus participation history
- Network service quality

### 5. Participation Score (P)

Measures active engagement in network activities.

$$P = \frac{blocks_{proposed} + \epsilon \cdot votes_{cast} + \zeta \cdot services_{provided}}{max_{possible}}$$

**Implementation:**
```rust
pub fn calculate_participation_score(
    blocks_proposed: u64,
    votes_cast: u64,
    services_provided: u64,
    epsilon: f64, // Vote weight = 0.1
    zeta: f64,   // Service weight = 0.05
    max_possible: u64,
) -> f64 {
    let weighted_participation = blocks_proposed as f64 
        + epsilon * votes_cast as f64 
        + zeta * services_provided as f64;
    
    (weighted_participation / max_possible as f64).min(1.0)
}
```

**Services Tracked:**
- Block proposals (when selected as leader)
- Vote casting (in consensus rounds)
- P2P message relay
- State proof provision
- Light node services

## Leader Selection Algorithm

### Slot Assignment
```rust
pub struct SlotScheduler {
    pub epoch_length: u64,                      // Slots per epoch
    pub committee_size: usize,                  // Number of validators (4-256)
    pub pou_scorer: PoUScorer,                  // PoU scoring engine
    pub local_id: [u8; 32],                    // Local node identifier
    pub is_validator: bool,                     // Validator status
}

impl SlotScheduler {
    pub fn get_slot_role(&self, slot: u64, committee: &[Validator]) -> SlotRole {
        let leader = self.select_leader(slot, committee);
        
        match (&leader, self.is_validator) {
            (Some(l), true) if *l == self.local_id => SlotRole::Leader,
            (Some(_), true) => SlotRole::Follower,
            _ => SlotRole::Observer,  // Light nodes are always observers
        }
    }
    
    pub fn select_leader(&self, slot: u64, committee: &[Validator]) -> Option<[u8; 32]> {
        let epoch = slot / self.epoch_length;
        
        // Compute PoU scores for all committee members
        let mut scored_validators: Vec<_> = committee
            .iter()
            .map(|validator| {
                let score = self.pou_scorer.compute_score(&validator.id, epoch);
                (validator.id, score)
            })
            .collect();
        
        // Sort by score (descending)
        scored_validators.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());
        
        // Select leader using weighted random selection based on PoU scores
        let leader_index = self.weighted_random_select(&scored_validators, slot);
        scored_validators.get(leader_index).map(|(id, _)| *id)
    }
    
    fn weighted_random_select(&self, validators: &[( [u8; 32], f64)], seed: u64) -> usize {
        let total_weight: f64 = validators.iter().map(|(_, score)| score).sum();
        let mut rng = StdRng::seed_from_u64(seed);
        let random_value: f64 = rng.gen_range(0.0..total_weight);
        
        let mut cumulative_weight = 0.0;
        for (i, (_, score)) in validators.iter().enumerate() {
            cumulative_weight += score;
            if random_value <= cumulative_weight {
                return i;
            }
        }
        
        validators.len() - 1 // Fallback to last validator
    }
}
```

### Committee Selection
```rust
pub struct CommitteeSelector {
    pub min_committee_size: usize,               // Minimum committee size (4)
    pub max_committee_size: usize,               // Maximum committee size (256)
    pub bond_threshold: u128,                    // Minimum bond requirement (1M tokens)
}

impl CommitteeSelector {
    pub fn select_committee(&self, candidates: &[Validator]) -> Vec<Validator> {
        // Filter by bond requirement
        let qualified: Vec<_> = candidates
            .iter()
            .filter(|v| v.bond_amount >= self.bond_threshold)
            .filter(|v| self.is_validator_eligible(&v.id))
            .cloned()
            .collect();
        
        if qualified.len() < self.min_committee_size {
            return Vec::new(); // Insufficient qualified validators
        }
        
        // Sort by PoU score and bond amount
        qualified.sort_by(|a, b| {
            let score_a = a.pou_score * (a.bond_amount as f64).ln();
            let score_b = b.pou_score * (b.bond_amount as f64).ln();
            score_b.partial_cmp(&score_a).unwrap()
        });
        
        // Select top validators up to max size
        qualified.into_iter()
            .take(self.max_committee_size.min(qualified.len()))
            .collect()
    }
    
    fn is_validator_eligible(&self, validator_id: &[u8; 32]) -> bool {
        // Check if validator has active bond and no slashing history
        self.bond_manager.has_active_bond(validator_id).unwrap_or(false)
    }
}
```

## Score Computation Engine

### Real-time Score Updates
```rust
pub struct PoUScorer {
    pub weights: PoUWeights,                    // Component weights
    pub metrics_collector: MetricsCollector,    // Metrics collection
    pub score_cache: LruCache<[u8; 32], f64>,   // Score caching
    pub smoothing_factor: f64,                  // EMA smoothing (0.9)
}

impl PoUScorer {
    pub fn compute_score(&self, validator_id: &[u8; 32], epoch: u64) -> f64 {
        // Check cache first
        if let Some(cached_score) = self.score_cache.get(validator_id) {
            return *cached_score;
        }
        
        // Collect metrics for the epoch
        let metrics = self.metrics_collector.collect_metrics(validator_id, epoch);
        
        // Compute individual component scores
        let availability = calculate_availability_score(
            metrics.uptime_percentage,
            metrics.downtime_incidents,
            0.1 // lambda
        );
        
        let latency = calculate_latency_score(
            metrics.avg_latency_ms,
            metrics.latency_stddev_ms,
            0.01, // alpha
            0.005 // beta
        );
        
        let integrity = calculate_integrity_score(
            metrics.correct_actions,
            metrics.incorrect_actions,
            2.0 // gamma
        );
        
        let reputation = calculate_reputation_score(
            &metrics.historical_scores,
            0.95 // delta
        );
        
        let participation = calculate_participation_score(
            metrics.blocks_proposed,
            metrics.votes_cast,
            metrics.services_provided,
            0.1, // epsilon
            0.05, // zeta
            metrics.max_possible_participation
        );
        
        // Compute weighted final score
        let final_score = self.weights.availability * availability
            + self.weights.latency * latency
            + self.weights.integrity * integrity
            + self.weights.reputation * reputation
            + self.weights.participation * participation;
        
        // Cache the result
        self.score_cache.put(*validator_id, final_score);
        
        final_score
    }
    
    pub fn update_scores_batch(&mut self, validator_ids: &[[u8; 32]], epoch: u64) {
        for &validator_id in validator_ids {
            self.compute_score(&validator_id, epoch);
        }
    }
}
```

### Metrics Collection
```rust
pub struct MetricsCollector {
    pub storage: Arc<Storage>,                   // Storage backend
    pub time_window: Duration,                   // Metrics time window
}

impl MetricsCollector {
    pub fn collect_metrics(&self, validator_address: &[u8; 32]) -> ValidatorMetrics {
        let now = current_timestamp();
        let window_start = now - self.time_window.as_secs();
        
        ValidatorMetrics {
            uptime_percentage: self.calculate_uptime(validator_address, window_start, now),
            downtime_incidents: self.count_downtime_incidents(validator_address, window_start, now),
            avg_latency_ms: self.calculate_avg_latency(validator_address, window_start, now),
            latency_stddev_ms: self.calculate_latency_stddev(validator_address, window_start, now),
            correct_actions: self.count_correct_actions(validator_address, window_start, now),
            incorrect_actions: self.count_incorrect_actions(validator_address, window_start, now),
            historical_scores: self.get_historical_scores(validator_address),
            blocks_proposed: self.count_blocks_proposed(validator_address, window_start, now),
            votes_cast: self.count_votes_cast(validator_address, window_start, now),
            services_provided: self.count_services_provided(validator_address, window_start, now),
        }
    }
}
```

## Performance Optimization

### SIMD Score Computation
```rust
#[cfg(target_arch = "x86_64")]
#[target_feature(enable = "avx2")]
unsafe fn compute_scores_simd_batch(
    validators: &[Validator],
    weights: &PoUWeights,
) -> Vec<f64> {
    let mut scores = vec![0.0; validators.len()];
    
    // Process validators in batches of 4 (AVX2 width)
    for (chunk_idx, validator_chunk) in validators.chunks_exact(4).enumerate() {
        let base_idx = chunk_idx * 4;
        
        // Load validator metrics into SIMD registers
        let availability_vec = _mm256_loadu_pd(
            [validator_chunk[0].metrics.availability,
             validator_chunk[1].metrics.availability,
             validator_chunk[2].metrics.availability,
             validator_chunk[3].metrics.availability].as_ptr()
        );
        
        let latency_vec = _mm256_loadu_pd(
            [validator_chunk[0].metrics.latency,
             validator_chunk[1].metrics.latency,
             validator_chunk[2].metrics.latency,
             validator_chunk[3].metrics.latency].as_ptr()
        );
        
        // Compute weighted sum
        let weights_vec = _mm256_set_pd(weights.availability, weights.latency, weights.integrity, weights.reputation);
        let scores_vec = _mm256_mul_pd(availability_vec, weights_vec);
        
        // Store results
        _mm256_storeu_pd(scores.as_mut_ptr().add(base_idx), scores_vec);
    }
    
    // Handle remaining validators with scalar computation
    let remaining_start = (validators.len() / 4) * 4;
    for i in remaining_start..validators.len() {
        scores[i] = compute_score_scalar(&validators[i], weights);
    }
    
    scores
}
```

### Caching Strategy
```rust
pub struct ScoreCache {
    pub cache: LruCache<[u8; 32], CachedScore>,   // Score cache
    pub ttl: Duration,                           // Cache TTL
    pub max_size: usize,                          // Maximum cache size
}

pub struct CachedScore {
    pub score: f64,                               // Cached score
    pub timestamp: Instant,                       // Cache timestamp
    pub epoch: u64,                               // Epoch when computed
    pub metrics_hash: [u8; 32],                   // Metrics hash for invalidation
}
```

## Security Considerations

### Score Manipulation Prevention
```rust
pub struct ScoreValidator {
    pub max_score_change: f64,                    // Maximum score change per epoch
    pub anomaly_threshold: f64,                    // Anomaly detection threshold
    pub grace_period: Duration,                   // Grace period for new validators
}

impl ScoreValidator {
    pub fn validate_score_change(&self, old_score: f64, new_score: f64) -> bool {
        let change = (new_score - old_score).abs();
        change <= self.max_score_change
    }
    
    pub fn detect_anomaly(&self, score: f64, historical_avg: f64, stddev: f64) -> bool {
        let z_score = (score - historical_avg) / stddev;
        z_score.abs() > self.anomaly_threshold
    }
}
```

### Sybil Resistance
- Bond requirements prevent cheap Sybil attacks
- Reputation decay reduces long-term manipulation impact
- Multi-factor scoring makes single-vector attacks ineffective
- Network-wide metrics provide cross-validation

## Configuration Parameters

### Default Configuration
```rust
pub struct PoUConfig {
    pub weights: PoUWeights,                     // Component weights
    pub update_interval: Duration,               // Score update interval
    pub cache_ttl: Duration,                      // Cache TTL
    pub max_committee_size: usize,                // Maximum committee size
    pub min_bond_amount: u128,                   // Minimum bond requirement
    pub score_decay_factor: f64,                  // Historical score decay
    pub anomaly_threshold: f64,                  // Anomaly detection threshold
}

impl Default for PoUConfig {
    fn default() -> Self {
        Self {
            weights: PoUWeights {
                availability: 0.25,
                latency: 0.20,
                integrity: 0.25,
                reputation: 0.20,
                participation: 0.10,
            },
            update_interval: Duration::from_secs(300), // 5 minutes
            cache_ttl: Duration::from_secs(3600),      // 1 hour
            max_committee_size: 256,
            min_bond_amount: 1_000_000_000_000_000,   // 1M tokens
            score_decay_factor: 0.95,
            anomaly_threshold: 3.0, // 3 sigma
        }
    }
}
```

## Performance Metrics

### Target Performance
- **Score Computation**: < 1ms per validator
- **Batch Processing**: >1000 validators/second
- **Cache Hit Rate**: >90%
- **Memory Usage**: <100MB for 10K validators
- **Update Latency**: <5 seconds for score propagation

### Monitoring Metrics
```rust
pub struct PoUMetrics {
    pub validators_scored: u64,                  // Validators scored
    pub avg_score_computation_time: Duration,    // Average computation time
    pub cache_hit_rate: f64,                     // Cache hit rate
    pub leader_changes: u64,                     // Leader changes
    pub committee_changes: u64,                  // Committee changes
    pub anomaly_detections: u64,                  // Anomaly detections
}
```

Proof of Unity provides a robust, multi-dimensional consensus mechanism that rewards meaningful network participation while maintaining security and performance requirements for enterprise-grade blockchain infrastructure.
