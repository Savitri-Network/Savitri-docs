# Voting System

## Overview

The Savitri Network voting system enables stakeholders to participate in governance decisions through a secure, transparent, and efficient voting mechanism. The system supports multiple voting power sources and ensures fair representation while preventing malicious voting patterns.

## Voting Power Calculation

### Voting Power Sources

```rust
pub struct VotingPower {
    pub vote_tokens: u128,                    // Vote token balance
    pub pou_score: f64,                       // PoU reputation score
    pub bond_amount: u128,                     // Bonded tokens
    pub time_multiplier: f64,                 // Time-based multiplier
    pub participation_bonus: f64,              // Participation bonus
}

impl VotingPower {
    pub fn calculate_total_power(&self) -> u128 {
        let token_power = self.vote_tokens;
        let reputation_power = (self.pou_score * 1000.0) as u128; // Convert to token units
        let bond_power = self.bond_amount / 10; // 10% of bond weight
        let time_bonus = (self.time_multiplier * token_power as f64) as u128;
        let participation_bonus = (self.participation_bonus * token_power as f64) as u128;
        
        token_power + reputation_power + bond_power + time_bonus + participation_bonus
    }
}
```

### Voting Power Components

#### 1. Vote Token Power

```rust
pub struct VoteTokenPower {
    pub balance: u128,                        // Token balance
    pub vesting_schedule: VestingSchedule,    // Vesting schedule
    pub lockup_period: Option<u64>,           // Lockup period
    pub delegation_info: Option<DelegationInfo>, // Delegation info
}

impl VoteTokenPower {
    pub fn calculate_effective_power(&self, current_time: u64) -> u128 {
        let mut effective_power = self.balance;
        
        // Apply vesting
        effective_power = self.apply_vesting_multiplier(effective_power, current_time);
        
        // Apply lockup bonus
        if let Some(lockup) = self.lockup_period {
            if current_time < lockup {
                effective_power = (effective_power as f64 * 1.2) as u128; // 20% bonus
            }
        }
        
        effective_power
    }
}
```

#### 2. PoU Reputation Power

```rust
pub struct PoUPower {
    pub availability_score: f64,              // Availability score (U)
    pub latency_score: f64,                   // Latency score (L)
    pub integrity_score: f64,                  // Integrity score (I)
    pub reputation_score: f64,                 // Reputation score (R)
    pub participation_score: f64,              // Participation score (P)
    pub total_score: f64,                      // Total PoU score
}

impl PoUPower {
    pub fn calculate_voting_power(&self) -> u128 {
        // Convert PoU score to voting power (max 1000 tokens worth)
        let max_power = 1000_000_000_000_000; // 1000 tokens in wei
        let power = (self.total_score / 100.0) * max_power as f64;
        
        power as u128
    }
}
```

#### 3. Bond Power

```rust
pub struct BondPower {
    pub bond_amount: u128,                     // Bonded amount
    pub bond_duration: u64,                    // Bond duration
    pub bond_type: BondType,                   // Bond type
    pub slashing_history: SlashingHistory,    // Slashing history
}

pub enum BondType {
    Validator,                                 // Validator bond
    Delegator,                                // Delegator bond
    System,                                   // System bond
}

impl BondPower {
    pub fn calculate_voting_power(&self) -> u128 {
        let base_power = self.bond_amount / 10; // 10% of bond weight
        
        // Apply duration bonus
        let duration_bonus = if self.bond_duration > 1000 { // > 1000 blocks
            (base_power as f64 * 1.1) as u128 // 10% bonus
        } else {
            base_power
        };
        
        // Apply type multiplier
        let type_multiplier = match self.bond_type {
            BondType::Validator => 1.5,
            BondType::Delegator => 1.2,
            BondType::System => 1.0,
        };
        
        (duration_bonus as f64 * type_multiplier) as u128
    }
}
```

## Voting Process

### Step 1: Vote Validation

```rust
pub struct VoteValidator {
    pub power_calculator: VotingPowerCalculator, // Power calculation
    pub eligibility_checker: EligibilityChecker, // Eligibility checking
    pub duplicate_detector: DuplicateDetector,   // Duplicate detection
}

impl VoteValidator {
    pub fn validate_vote(&self, vote: &Vote, proposal: &Proposal) -> Result<ValidatedVote, VoteValidationError> {
        // 1. Check voting eligibility
        self.eligibility_checker.check_eligibility(&vote.voter, proposal)?;
        
        // 2. Calculate voting power
        let voting_power = self.power_calculator.calculate_power(&vote.voter)?;
        
        // 3. Check for duplicate voting
        self.duplicate_detector.check_duplicate(vote.proposal_id, &vote.voter)?;
        
        // 4. Validate vote amount
        if vote.vote_amount > voting_power {
            return Err(VoteValidationError::InsufficientPower);
        }
        
        // 5. Validate voting period
        if !self.is_voting_period_active(proposal) {
            return Err(VoteValidationError::VotingPeriodClosed);
        }
        
        Ok(ValidatedVote {
            vote: vote.clone(),
            voting_power,
            validation_timestamp: current_timestamp(),
        })
    }
}
```

### Step 2: Vote Casting

```rust
pub struct VoteCaster {
    pub storage: Arc<Storage>,                 // Storage
    pub vote_aggregator: VoteAggregator,       // Vote aggregation
    pub event_emitter: EventEmitter,           // Event emission
}

impl VoteCaster {
    pub fn cast_vote(&mut self, validated_vote: ValidatedVote) -> Result<VoteReceipt, VoteCastingError> {
        // 1. Store vote
        let vote_key = self.encode_vote_key(validated_vote.vote.proposal_id, &validated_vote.vote.voter);
        let vote_value = bincode::serialize(&validated_vote.vote)?;
        self.storage.put(CF_VOTES, &vote_key, &vote_value)?;
        
        // 2. Update proposal vote counts
        self.update_proposal_votes(&validated_vote)?;
        
        // 3. Update voting power usage
        self.update_voting_power_usage(&validated_vote)?;
        
        // 4. Emit vote event
        self.event_emitter.emit_vote_casted_event(&validated_vote)?;
        
        // 5. Generate receipt
        let receipt = VoteReceipt {
            vote_id: self.generate_vote_id(),
            proposal_id: validated_vote.vote.proposal_id,
            voter: validated_vote.vote.voter.clone(),
            vote_type: validated_vote.vote.vote_type,
            vote_amount: validated_vote.vote.vote_amount,
            voting_power: validated_vote.voting_power,
            timestamp: current_timestamp(),
            block_height: self.storage.get_current_block_height()?,
        };
        
        Ok(receipt)
    }
    
    fn update_proposal_votes(&mut self, vote: &ValidatedVote) -> Result<(), StorageError> {
        let mut proposal = self.storage.get_proposal(vote.vote.proposal_id)?;
        
        match vote.vote.vote_type {
            VoteType::Yes => proposal.yes_votes += vote.vote.vote_amount,
            VoteType::No => proposal.no_votes += vote.vote.vote_amount,
            VoteType::Abstain => proposal.abstain_votes += vote.vote.vote_amount,
        }
        
        self.storage.update_proposal(&proposal)?;
        Ok(())
    }
}
```

### Step 3: Vote Aggregation

```rust
pub struct VoteAggregator {
    pub vote_tally: VoteTally,                  // Vote tally
    pub quorum_calculator: QuorumCalculator,   // Quorum calculation
    pub result_determiner: ResultDeterminer,   // Result determination
}

impl VoteAggregator {
    pub fn aggregate_votes(&self, proposal_id: u64) -> Result<VoteResult, AggregationError> {
        // 1. Get all votes for proposal
        let votes = self.get_all_votes_for_proposal(proposal_id)?;
        
        // 2. Calculate vote totals
        let vote_totals = self.calculate_vote_totals(&votes);
        
        // 3. Check quorum
        let quorum_met = self.quorum_calculator.check_quorum(&vote_totals)?;
        
        // 4. Determine result
        let result = if quorum_met {
            self.result_determiner.determine_result(&vote_totals)?
        } else {
            VoteResult::QuorumNotMet
        };
        
        Ok(result)
    }
    
    fn calculate_vote_totals(&self, votes: &[Vote]) -> VoteTotals {
        let mut totals = VoteTotals::default();
        
        for vote in votes {
            match vote.vote_type {
                VoteType::Yes => totals.yes_votes += vote.vote_amount,
                VoteType::No => totals.no_votes += vote.vote_amount,
                VoteType::Abstain => totals.abstain_votes += vote.vote_amount,
            }
            totals.total_votes += vote.vote_amount;
        }
        
        totals
    }
}
```

## Voting Types

### 1. Standard Voting

```rust
pub struct StandardVote {
    pub proposal_id: u64,                     // Proposal ID
    pub voter: Vec<u8>,                         // Voter address
    pub vote_type: VoteType,                   // Vote type
    pub vote_amount: u128,                     // Voting power used
    pub reason: Option<String>,                // Vote reason
    pub timestamp: u64,                        // Vote timestamp
}
```

### 2. Delegated Voting

```rust
pub struct DelegatedVote {
    pub proposal_id: u64,                     // Proposal ID
    pub delegator: Vec<u8],                     // Delegator address
    pub delegate: Vec<u8],                      // Delegate address
    pub vote_type: VoteType,                   // Vote type
    pub voting_power: u128,                     // Voting power
    pub delegation_proof: DelegationProof,     // Delegation proof
}

pub struct DelegationProof {
    pub signature: [u8; 64],                   // Delegator signature
    pub nonce: u64,                            // Delegation nonce
    pub expiry: u64,                           // Delegation expiry
    pub scope: DelegationScope,                // Delegation scope
}

pub enum DelegationScope {
    All,                                       // All proposals
    Category(ProposalCategory),              // Specific category
    Proposal(u64),                            // Specific proposal
    TimeLimited(u64),                         // Time-limited
}
```

### 3. Conditional Voting

```rust
pub struct ConditionalVote {
    pub proposal_id: u64,                     // Proposal ID
    pub voter: Vec<u8],                         // Voter address
    pub conditions: Vec<VotingCondition>,    // Voting conditions
    pub vote_type: VoteType,                   // Vote type
    pub vote_amount: u128,                     // Voting power
    pub condition_proof: ConditionProof,     // Condition proof
}

pub struct VotingCondition {
    pub condition_type: ConditionType,        // Condition type
    pub threshold: u128,                      // Condition threshold
    pub oracle: Vec<u8>,                       // Oracle address
    pub deadline: u64,                         // Condition deadline
}

pub enum ConditionType {
    PriceAbove { price: u128 },               // Price above threshold
    PriceBelow { price: u128 },               // Price below threshold
    ParticipationAbove { count: u32 },         // Participation above threshold
    TimeElapsed { blocks: u64 },             // Time elapsed
}
```

## Security Features

### Vote Integrity

```rust
pub struct VoteIntegrity {
    pub signature_verifier: SignatureVerifier, // Signature verification
    pub timestamp_validator: TimestampValidator, // Timestamp validation
    pub replay_detector: ReplayDetector,       // Replay detection
}

impl VoteIntegrity {
    pub fn verify_vote_integrity(&self, vote: &Vote) -> Result<IntegrityResult, IntegrityError> {
        // 1. Verify signature
        let message = self.encode_vote_message(vote)?;
        self.signature_verifier.verify_signature(&message, &vote.signature, &vote.voter)?;
        
        // 2. Validate timestamp
        self.timestamp_validator.validate_timestamp(vote.timestamp)?;
        
        // 3. Check for replay
        self.replay_detector.check_replay(vote)?;
        
        Ok(IntegrityResult::Valid)
    }
}
```

### Anti-Collusion

```rust
pub struct AntiCollusion {
    pub collusion_detector: CollusionDetector, // Collusion detection
    pub voting_pattern_analyzer: PatternAnalyzer, // Pattern analysis
    pub network_analyzer: NetworkAnalyzer,     // Network analysis
}

impl AntiCollusion {
    pub fn detect_collusion(&self, votes: &[Vote]) -> Result<CollusionReport, CollusionError> {
        // 1. Analyze voting patterns
        let patterns = self.voting_pattern_analyzer.analyze_patterns(votes)?;
        
        // 2. Check network relationships
        let relationships = self.network_analyzer.analyze_relationships(votes)?;
        
        // 3. Detect collusion
        let collusion_detected = self.collusion_detector.detect_collusion(patterns, relationships)?;
        
        if collusion_detected.is_collusion {
            return Err(CollusionError::CollusionDetected(collusion_detected));
        }
        
        Ok(CollusionReport {
            is_clean: true,
            suspicious_patterns: vec![],
            network_anomalies: vec![],
        })
    }
}
```

### Rate Limiting

```rust
pub struct VotingRateLimiter {
    pub per_proposal_limit: u32,                // Per proposal limit
    pub per_voter_limit: u32,                   // Per voter limit
    pub time_window: Duration,                  // Time window
    pub penalty_system: PenaltySystem,          // Penalty system
}

impl VotingRateLimiter {
    pub fn check_rate_limit(&self, voter: &[u8], proposal_id: u64) -> Result<RateLimitResult, RateLimitError> {
        // 1. Check per-proposal limit
        let proposal_votes = self.count_votes_for_proposal(voter, proposal_id)?;
        if proposal_votes >= self.per_proposal_limit {
            return Err(RateLimitError::ProposalLimitExceeded);
        }
        
        // 2. Check per-voter limit
        let total_votes = self.count_total_votes(voter)?;
        if total_votes >= self.per_voter_limit {
            return Err(RateLimitError::VoterLimitExceeded);
        }
        
        // 3. Check time window
        if self.is_in_penalty_period(voter)? {
            return Err(RateLimitError::PenaltyPeriodActive);
        }
        
        Ok(RateLimitResult::Allowed)
    }
}
```

## Performance Optimization

### Batch Vote Processing

```rust
pub struct BatchVoteProcessor {
    pub batch_size: usize,                      // Batch size
    pub processing_threads: usize,              // Processing threads
    pub vote_cache: VoteCache,                  // Vote cache
}

impl BatchVoteProcessor {
    pub fn process_votes_batch(&mut self, votes: Vec<Vote>) -> Result<Vec<VoteReceipt>, BatchProcessingError> {
        let mut receipts = Vec::new();
        
        // 1. Group votes by proposal
        let mut proposal_votes: HashMap<u64, Vec<Vote>> = HashMap::new();
        for vote in votes {
            proposal_votes.entry(vote.proposal_id).or_insert_with(Vec::new).push(vote);
        }
        
        // 2. Process each proposal's votes
        for (proposal_id, proposal_votes) in proposal_votes {
            let batch_receipts = self.process_proposal_votes(proposal_id, proposal_votes)?;
            receipts.extend(batch_receipts);
        }
        
        Ok(receipts)
    }
    
    fn process_proposal_votes(&mut self, proposal_id: u64, votes: Vec<Vote>) -> Result<Vec<VoteReceipt>, BatchProcessingError> {
        let mut receipts = Vec::new();
        
        // 1. Validate all votes
        let validated_votes: Vec<ValidatedVote> = votes
            .into_iter()
            .map(|vote| self.validate_vote(vote))
            .collect::<Result<Vec<_>, _>>()?;
        
        // 2. Cast all votes
        for validated_vote in validated_votes {
            let receipt = self.cast_vote(validated_vote)?;
            receipts.push(receipt);
        }
        
        // 3. Update proposal vote counts
        self.update_proposal_vote_counts(proposal_id, &receipts)?;
        
        Ok(receipts)
    }
}
```

### Vote Caching

```rust
pub struct VoteCache {
    pub vote_cache: LRUCache<(u64, Vec<u8>), VoteReceipt>, // Vote cache
    pub power_cache: LRUCache<Vec<u8>, VotingPower>,   // Power cache
    pub proposal_cache: LRUCache<u64, Proposal>,     // Proposal cache
    pub cache_stats: CacheStats,                    // Cache statistics
}

impl VoteCache {
    pub fn get_cached_vote(&self, proposal_id: u64, voter: &[u8]) -> Option<&VoteReceipt> {
        self.vote_cache.get(&(proposal_id, voter.to_vec()))
    }
    
    pub fn cache_vote(&mut self, proposal_id: u64, voter: &[u8], receipt: VoteReceipt) {
        self.vote_cache.put((proposal_id, voter.to_vec()), receipt);
        self.cache_stats.vote_cache_hits += 1;
    }
    
    pub fn get_cached_power(&self, voter: &[u8]) -> Option<&VotingPower> {
        self.power_cache.get(voter)
    }
    
    pub fn cache_power(&mut self, voter: &[u8], power: VotingPower) {
        self.power_cache.put(voter.to_vec(), power);
        self.cache_stats.power_cache_hits += 1;
    }
}
```

## Monitoring Metrics

### Voting Metrics

```rust
pub struct VotingMetrics {
    pub total_votes_cast: u64,                  // Total votes cast
    pub votes_per_proposal: f64,                // Average votes per proposal
    pub voter_participation: f64,               // Voter participation rate
    pub voting_power_distribution: PowerDistribution, // Power distribution
    pub vote_processing_time: Duration,          // Average processing time
    pub cache_hit_rate: f64,                     // Cache hit rate
}

pub struct PowerDistribution {
    pub token_power_percentage: f64,           // Token power percentage
    pub reputation_power_percentage: f64,       // Reputation power percentage
    pub bond_power_percentage: f64,             // Bond power percentage
    pub gini_coefficient: f64,                 // Gini coefficient
    pub top_10_percent_share: f64,              // Top 10% share
}
```

### System Performance

```rust
pub struct VotingPerformance {
    pub throughput: f64,                       // Votes per second
    pub latency_p50: Duration,                  // 50th percentile latency
    pub latency_p95: Duration,                  // 95th percentile latency
    pub latency_p99: Duration,                  // 99th percentile latency
    pub error_rate: f64,                       // Error rate
    pub resource_utilization: ResourceUtilization, // Resource utilization
}

pub struct ResourceUtilization {
    pub cpu_usage: f64,                        // CPU usage percentage
    pub memory_usage: f64,                      // Memory usage percentage
    pub storage_io: f64,                        // Storage I/O operations
    pub network_io: f64,                        // Network I/O operations
}
```

This voting system provides a comprehensive, secure, and efficient framework for decentralized governance participation while ensuring fair representation and preventing malicious voting patterns.
