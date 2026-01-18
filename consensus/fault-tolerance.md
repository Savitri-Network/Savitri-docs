# Fault Tolerance and Recovery

## Fault Tolerance Overview

Savitri Network implements Byzantine Fault Tolerance (BFT) with $f = \lfloor(n-1)/3\rfloor$ fault tolerance, where $n$ is the number of validators. The system can continue operating correctly with up to $f$ Byzantine (malicious) validators.

## Byzantine Fault Model

### Fault Types
```rust
pub enum FaultType {
    /// Validator stops responding (crash fault)
    Crash,
    /// Validator sends conflicting messages (Byzantine fault)
    Byzantine,
    /// Validator delays messages (timing fault)
    Timing,
    /// Validator sends incorrect signatures (cryptographic fault)
    Cryptographic,
    /// Validator attempts double-spending (economic fault)
    DoubleSpend,
    /// Validator colludes with others (coordinated fault)
    Coordinated,
}
```

### Fault Detection
```rust
pub struct FaultDetector {
    pub timeout_detector: TimeoutDetector,       // Timeout-based detection
    pub inconsistency_detector: InconsistencyDetector, // Message inconsistency
    pub signature_detector: SignatureDetector,   // Signature verification
    pub behavior_detector: BehaviorDetector,     // Behavioral analysis
}

impl FaultDetector {
    pub fn detect_fault(&self, validator: &Validator, behavior: &ValidatorBehavior) -> Option<FaultType> {
        // 1. Check for timeout faults
        if self.timeout_detector.is_timeout(behavior.last_message_time) {
            return Some(FaultType::Crash);
        }
        
        // 2. Check for Byzantine behavior
        if let Some(fault) = self.inconsistency_detector.check_byzantine_behavior(validator, behavior) {
            return Some(fault);
        }
        
        // 3. Check for cryptographic faults
        if self.signature_detector.has_invalid_signatures(validator, behavior) {
            return Some(FaultType::Cryptographic);
        }
        
        // 4. Check for behavioral anomalies
        self.behavior_detector.analyze_behavior(validator, behavior)
    }
}
```

## BFT Consensus Safety

### Safety Properties
```rust
pub struct BFTSafety {
    pub fault_tolerance: usize,                  // Maximum Byzantine faults
    pub honest_threshold: usize,                  // Minimum honest validators
    pub quorum_size: usize,                      // Quorum for decisions
    pub safety_margin: f64,                      // Safety margin (0.0-1.0)
}

impl BFTSafety {
    pub fn new(total_validators: usize) -> Self {
        let fault_tolerance = (total_validators - 1) / 3;
        let honest_threshold = total_validators - fault_tolerance;
        let quorum_size = 2 * fault_tolerance + 1;
        let safety_margin = (honest_threshold as f64) / (total_validators as f64);
        
        Self {
            fault_tolerance,
            honest_threshold,
            quorum_size,
            safety_margin,
        }
    }
    
    pub fn is_safe(&self, honest_validators: usize) -> bool {
        honest_validators >= self.honest_threshold
    }
    
    pub fn can_reach_quorum(&self, participating_validators: usize) -> bool {
        participating_validators >= self.quorum_size
    }
}
```

### Safety Proof
```rust
impl BFTSafety {
    pub fn prove_safety(&self, certificate: &ConsensusCertificate) -> SafetyProof {
        let total_signatures = certificate.signatures.len();
        
        SafetyProof {
            block_hash: certificate.block_hash,
            height: certificate.height,
            total_validators: self.honest_threshold + self.fault_tolerance,
            byzantine_tolerance: self.fault_tolerance,
            signatures_received: total_signatures,
            honest_validators_min: self.honest_threshold,
            safety_guarantee: self.compute_safety_guarantee(total_signatures),
            mathematical_proof: self.generate_mathematical_proof(),
        }
    }
    
    fn compute_safety_guarantee(&self, signatures: usize) -> SafetyGuarantee {
        // With 2f+1 signatures, at most f validators can be Byzantine
        // Therefore at least f+1 validators are honest
        // Honest validators cannot sign conflicting blocks
        
        let byzantine_max = self.fault_tolerance;
        let honest_min = signatures - byzantine_max;
        
        SafetyGuarantee {
            cannot_fork: honest_min > byzantine_max,
            honest_majority: honest_min > signatures / 2,
            economic_security: honest_min > 0,
            finality_assured: signatures >= self.quorum_size,
        }
    }
}
```

## Network Partition Handling

### Partition Detection
```rust
pub struct PartitionDetector {
    pub connectivity_matrix: ConnectivityMatrix, // Network connectivity
    pub partition_threshold: f64,                // Partition detection threshold
    pub recovery_timeout: Duration,               // Recovery timeout
}

impl PartitionDetector {
    pub fn detect_partition(&self) -> Option<NetworkPartition> {
        let connectivity = self.connectivity_matrix.compute_connectivity();
        
        if connectivity < self.partition_threshold {
            Some(NetworkPartition {
                detected_at: Instant::now(),
                connectivity_score: connectivity,
                affected_validators: self.identify_affected_validators(),
                partition_type: self.classify_partition_type(),
            })
        } else {
            None
        }
    }
    
    pub fn can_recover(&self, partition: &NetworkPartition) -> bool {
        let recovery_time = partition.detected_at.elapsed();
        let connectivity_improved = self.connectivity_matrix.compute_connectivity() > self.partition_threshold;
        
        recovery_time < self.recovery_timeout && connectivity_improved
    }
}
```

### Partition Recovery
```rust
pub struct PartitionRecovery {
    pub state_sync: StateSynchronizer,          // State synchronization
    pub message_buffer: MessageBuffer,          // Buffered messages
    pub recovery_coordinator: RecoveryCoordinator, // Recovery coordination
}

impl PartitionRecovery {
    pub fn initiate_recovery(&mut self, partition: &NetworkPartition) -> Result<RecoveryPlan, RecoveryError> {
        // 1. Assess partition damage
        let damage_assessment = self.assess_partition_damage(partition)?;
        
        // 2. Create recovery plan
        let recovery_plan = RecoveryPlan {
            sync_strategy: self.determine_sync_strategy(&damage_assessment),
            message_replay_order: self.compute_message_replay_order(partition),
            validator_reintegration: self.plan_validator_reintegration(partition),
            estimated_recovery_time: self.estimate_recovery_time(&damage_assessment),
        };
        
        // 3. Execute recovery
        self.execute_recovery_plan(&recovery_plan)?;
        
        Ok(recovery_plan)
    }
    
    fn execute_recovery_plan(&mut self, plan: &RecoveryPlan) -> Result<(), RecoveryError> {
        // 1. Synchronize state
        self.state_sync.synchronize_to_latest_state(&plan.sync_strategy)?;
        
        // 2. Replay buffered messages
        self.replay_buffered_messages(&plan.message_replay_order)?;
        
        // 3. Reintegrate validators
        self.reintegrate_validators(&plan.validator_reintegration)?;
        
        // 4. Resume normal operation
        self.resume_normal_operation()?;
        
        Ok(())
    }
}
```

## Validator Recovery

### Crash Recovery
```rust
pub struct CrashRecovery {
    pub state_checkpoint: StateCheckpoint,      // State checkpointing
    pub message_log: MessageLog,                // Message logging
    pub recovery_coordinator: RecoveryCoordinator,
}

impl CrashRecovery {
    pub fn recover_from_crash(&mut self, validator_address: &[u8; 32]) -> Result<RecoveryResult, RecoveryError> {
        // 1. Find latest checkpoint
        let checkpoint = self.state_checkpoint.find_latest_checkpoint(validator_address)?;
        
        // 2. Restore state from checkpoint
        let restored_state = self.restore_state_from_checkpoint(&checkpoint)?;
        
        // 3. Replay messages since checkpoint
        let messages_to_replay = self.message_log.get_messages_since_checkpoint(&checkpoint)?;
        let replay_result = self.replay_messages(&messages_to_replay, &restored_state)?;
        
        // 4. Verify state consistency
        self.verify_state_consistency(&replay_result.final_state)?;
        
        // 5. Rejoin network
        self.rejoin_network(validator_address, &replay_result.final_state)?;
        
        Ok(RecoveryResult {
            recovered_height: replay_result.final_height,
            messages_replayed: messages_to_replay.len(),
            recovery_time: replay_result.recovery_time,
            state_hash: replay_result.final_state_hash,
        })
    }
}
```

### Byzantine Recovery
```rust
pub struct ByzantineRecovery {
    pub evidence_collector: EvidenceCollector,  // Evidence collection
    pub slashing_manager: SlashingManager,     // Slashing management
    pub reputation_system: ReputationSystem,     // Reputation system
}

impl ByzantineRecovery {
    pub fn handle_byzantine_behavior(&mut self, validator: &Validator, evidence: &Evidence) -> Result<ByzantineResponse, ByzantineError> {
        // 1. Validate evidence
        self.validate_evidence(evidence)?;
        
        // 2. Collect additional evidence
        let additional_evidence = self.evidence_collector.collect_evidence(validator)?;
        
        // 3. Determine fault severity
        let fault_severity = self.assess_fault_severity(evidence, &additional_evidence)?;
        
        // 4. Apply appropriate response
        let response = match fault_severity {
            FaultSeverity::Minor => self.apply_warning(validator),
            FaultSeverity::Moderate => self.apply_temporary_suspension(validator),
            FaultSeverity::Severe => self.apply_slashing(validator),
            FaultSeverity::Critical => self.apply_permanent_ban(validator),
        };
        
        // 5. Update reputation
        self.reputation_system.update_reputation(validator, &response)?;
        
        // 6. Broadcast evidence to network
        self.broadcast_evidence(evidence, &additional_evidence)?;
        
        Ok(response)
    }
    
    fn apply_slashing(&mut self, validator: &Validator) -> Result<ByzantineResponse, ByzantineError> {
        // 1. Calculate slash amount
        let slash_amount = self.calculate_slash_amount(validator)?;
        
        // 2. Execute slashing
        self.slashing_manager.slash_validator(validator, slash_amount)?;
        
        // 3. Remove from validator set
        self.remove_validator_from_set(validator)?;
        
        // 4. Update committee
        self.update_committee_membership()?;
        
        Ok(ByzantineResponse {
            action: ByzantineAction::Slashed,
            penalty_amount: slash_amount,
            removal_duration: Duration::from_secs(86400 * 30), // 30 days
            reputation_impact: -100,
        })
    }
}
```

## Timing Fault Tolerance

### Timeout Management
```rust
pub struct TimeoutManager {
    pub base_timeout: Duration,                   // Base timeout
    pub timeout_multiplier: f64,                 // Timeout multiplier
    pub adaptive_threshold: f64,                  // Adaptive threshold
    pub timeout_history: TimeoutHistory,         // Timeout history
}

impl TimeoutManager {
    pub fn calculate_timeout(&self, operation: Operation, network_conditions: &NetworkConditions) -> Duration {
        let base_time = match operation {
            Operation::Proposal => self.base_timeout,
            Operation::Vote => self.base_timeout / 2,
            Operation::Certificate => self.base_timeout * 2,
        };
        
        let network_multiplier = self.calculate_network_multiplier(network_conditions);
        let adaptive_multiplier = self.calculate_adaptive_multiplier();
        
        let final_timeout = Duration::from_millis(
            (base_time.as_millis() as f64 * network_multiplier * adaptive_multiplier) as u64
        );
        
        final_timeout
    }
    
    pub fn handle_timeout(&mut self, operation: Operation, validator: &Validator) -> TimeoutAction {
        // Record timeout
        self.timeout_history.record_timeout(validator, operation);
        
        // Determine timeout pattern
        let pattern = self.timeout_history.analyze_timeout_pattern(validator);
        
        match pattern {
            TimeoutPattern::Isolated => TimeoutAction::Retry,
            TimeoutPattern::Recurring => TimeoutAction::IncreaseTimeout,
            TimeoutPattern::Persistent => TimeoutAction::MarkSuspect,
            TimeoutPattern::Malicious => TimeoutAction::ReportByzantine,
        }
    }
}
```

### Adaptive Timing
```rust
pub struct AdaptiveTiming {
    pub network_monitor: NetworkMonitor,         // Network monitoring
    pub performance_tracker: PerformanceTracker, // Performance tracking
    pub timing_adjuster: TimingAdjuster,         // Timing adjustment
}

impl AdaptiveTiming {
    pub fn adjust_timing_parameters(&mut self, metrics: &NetworkMetrics) -> TimingAdjustment {
        // 1. Analyze network conditions
        let network_health = self.network_monitor.assess_network_health(metrics);
        
        // 2. Analyze performance trends
        let performance_trend = self.performance_tracker.analyze_trend(metrics);
        
        // 3. Calculate adjustments
        let adjustment = TimingAdjustment {
            timeout_multiplier: self.calculate_timeout_multiplier(&network_health),
            retry_strategy: self.determine_retry_strategy(&performance_trend),
            batch_size_adjustment: self.calculate_batch_adjustment(&network_health),
            parallelism_adjustment: self.calculate_parallelism_adjustment(&performance_trend),
        };
        
        // 4. Apply adjustments
        self.timing_adjuster.apply_adjustments(&adjustment)?;
        
        adjustment
    }
}
```

## State Recovery

### State Checkpointing
```rust
pub struct StateCheckpointManager {
    pub checkpoint_interval: Duration,            // Checkpoint interval
    pub checkpoint_retention: usize,              // Checkpoint retention count
    pub compression_enabled: bool,                // Compression enabled
    pub checkpoint_storage: CheckpointStorage,    // Checkpoint storage
}

impl StateCheckpointManager {
    pub fn create_checkpoint(&self, state: &StateDB) -> Result<Checkpoint, CheckpointError> {
        let checkpoint = Checkpoint {
            id: self.generate_checkpoint_id(),
            height: state.get_latest_height(),
            state_root: state.get_state_root(),
            timestamp: current_timestamp(),
            validator_set: state.get_validator_set(),
            compressed_data: if self.compression_enabled {
                self.compress_state_data(state)?
            } else {
                self.serialize_state_data(state)?
            },
        };
        
        // Store checkpoint
        self.checkpoint_storage.store_checkpoint(&checkpoint)?;
        
        // Cleanup old checkpoints
        self.cleanup_old_checkpoints()?;
        
        Ok(checkpoint)
    }
    
    pub fn restore_from_checkpoint(&self, checkpoint_id: &str) -> Result<StateDB, CheckpointError> {
        let checkpoint = self.checkpoint_storage.load_checkpoint(checkpoint_id)?;
        
        let state_data = if self.compression_enabled {
            self.decompress_state_data(&checkpoint.compressed_data)?
        } else {
            checkpoint.compressed_data.clone()
        };
        
        self.deserialize_state_data(&state_data, &checkpoint)
    }
}
```

### State Synchronization
```rust
pub struct StateSynchronizer {
    pub sync_strategy: SyncStrategy,             // Synchronization strategy
    pub peer_selector: PeerSelector,             // Peer selection
    pub state_validator: StateValidator,         // State validation
    pub sync_progress: SyncProgress,             // Sync progress tracking
}

impl StateSynchronizer {
    pub fn synchronize_state(&mut self, target_height: u64) -> Result<SyncResult, SyncError> {
        // 1. Select sync peers
        let sync_peers = self.peer_selector.select_sync_peers()?;
        
        // 2. Determine sync strategy
        let strategy = self.determine_sync_strategy(target_height)?;
        
        // 3. Execute synchronization
        let result = match strategy {
            SyncStrategy::FullSync => self.execute_full_sync(target_height, &sync_peers)?,
            SyncStrategy::IncrementalSync => self.execute_incremental_sync(target_height, &sync_peers)?,
            SyncStrategy::FastSync => self.execute_fast_sync(target_height, &sync_peers)?,
        };
        
        // 4. Validate synchronized state
        self.state_validator.validate_state(&result.final_state)?;
        
        // 5. Update sync progress
        self.sync_progress.update_progress(&result);
        
        Ok(result)
    }
    
    fn execute_full_sync(&mut self, target_height: u64, peers: &[PeerId]) -> Result<SyncResult, SyncError> {
        let mut current_height = self.get_current_height()?;
        let mut synced_blocks = Vec::new();
        
        while current_height < target_height {
            // Request block batch
            let batch_size = std::cmp::min(1000, target_height - current_height);
            let block_batch = self.request_block_batch(current_height, batch_size, peers)?;
            
            // Validate and store blocks
            for block in block_batch {
                self.validate_block(&block)?;
                self.store_block(&block)?;
                synced_blocks.push(block);
            }
            
            current_height += batch_size as u64;
            
            // Update progress
            self.sync_progress.update_height(current_height, target_height);
        }
        
        Ok(SyncResult {
            synced_height: current_height,
            blocks_synced: synced_blocks.len(),
            sync_time: self.get_sync_duration(),
            final_state: self.compute_final_state()?,
        })
    }
}
```

## Recovery Monitoring

### Recovery Metrics
```rust
pub struct RecoveryMetrics {
    pub recovery_events: u64,                    // Total recovery events
    pub avg_recovery_time: Duration,              // Average recovery time
    pub recovery_success_rate: f64,              // Recovery success rate
    pub fault_types_detected: BTreeMap<FaultType, u64>, // Fault types detected
    pub partitions_detected: u64,                 // Partitions detected
    pub validators_recovered: u64,               // Validators recovered
}

impl RecoveryMetrics {
    pub fn calculate_recovery_health(&self) -> RecoveryHealth {
        let success_rate = self.recovery_success_rate;
        let avg_time = self.avg_recovery_time;
        let event_frequency = self.recovery_events as f64 / self.get_uptime().as_secs_f64();
        
        let health_score = (success_rate * 0.5) 
            + (self.calculate_time_score(avg_time) * 0.3) 
            + (self.calculate_frequency_score(event_frequency) * 0.2);
        
        RecoveryHealth {
            overall_score: health_score,
            success_rate,
            avg_recovery_time: avg_time,
            event_frequency,
            recommendations: self.generate_recommendations(),
        }
    }
}
```

### Health Assessment
```rust
pub struct RecoveryHealthAssessment {
    pub fault_tolerance: FaultToleranceAssessment, // Fault tolerance assessment
    pub network_resilience: NetworkResilienceAssessment, // Network resilience
    pub state_consistency: StateConsistencyAssessment, // State consistency
    pub validator_behavior: ValidatorBehaviorAssessment, // Validator behavior
}

impl RecoveryHealthAssessment {
    pub fn assess_overall_health(&self) -> OverallHealth {
        let fault_score = self.fault_tolerance.assess();
        let network_score = self.network_resilience.assess();
        let state_score = self.state_consistency.assess();
        let validator_score = self.validator_behavior.assess();
        
        let overall_score = (fault_score + network_score + state_score + validator_score) / 4.0;
        
        OverallHealth {
            score: overall_score,
            status: self.determine_health_status(overall_score),
            critical_issues: self.identify_critical_issues(),
            recommendations: self.generate_health_recommendations(),
        }
    }
}
```

## Configuration

### Fault Tolerance Configuration
```rust
pub struct FaultToleranceConfig {
    pub bft_parameters: BFTParameters,           // BFT parameters
    pub recovery_config: RecoveryConfig,         // Recovery configuration
    pub monitoring_config: MonitoringConfig,     // Monitoring configuration
    pub timeout_config: TimeoutConfig,           // Timeout configuration
}

impl Default for FaultToleranceConfig {
    fn default() -> Self {
        Self {
            bft_parameters: BFTParameters::new(256),
            recovery_config: RecoveryConfig::default(),
            monitoring_config: MonitoringConfig::default(),
            timeout_config: TimeoutConfig::default(),
        }
    }
}
```

## Replay Protection

### Replay Guard Implementation
```rust
pub struct ReplayGuard {
    pub proposals: HashMap<(u64, u64, [u8; 32]), u64>, // (epoch, height, proposer) -> seq
    pub votes: HashMap<(u64, u64, [u8; 32]), u64>,      // (epoch, height, voter) -> seq
}

impl ReplayGuard {
    pub fn record_proposal(&mut self, proposal: &ConsensusProposal) -> ConsensusResult<()> {
        let key = (proposal.epoch_id, proposal.height, proposal.proposer);
        let current_seq = self.proposals.get(&key).copied().unwrap_or(0);
        
        if proposal.proposal_seq <= current_seq {
            return Err(ConsensusError::ReplayDetected);
        }
        
        self.proposals.insert(key, proposal.proposal_seq);
        Ok(())
    }
    
    pub fn record_vote(&mut self, vote: &ConsensusVote) -> ConsensusResult<()> {
        let key = (vote.epoch_id, vote.height, vote.voter);
        let current_seq = self.votes.get(&key).copied().unwrap_or(0);
        
        if vote.vote_seq <= current_seq {
            return Err(ConsensusError::ReplayDetected);
        }
        
        self.votes.insert(key, vote.vote_seq);
        Ok(())
    }
}
```

### ZKP Certificate Integration
```rust
pub struct ZKPCertificate {
    pub block_hash: Hash64,
    pub height: u64,
    pub round: u32,
    pub proposer: [u8; 32],
    pub signatures: Vec<Signature>,
    pub zkp_proof: ZKProof,                    // Zero-knowledge proof
    pub proposer_pou_score: u32,               // PoU score at time of proposal
}

impl ZKPCertificate {
    pub fn verify_zkp_proof<V: ZkVerifier>(&self, verifier: &V) -> bool {
        let statement = Statement::new(
            self.block_hash,
            self.height,
            self.round,
            self.proposer,
        );
        verifier.verify(&statement, &self.zkp_proof).unwrap_or(false)
    }
}
```

This fault tolerance system provides comprehensive protection against various failure modes while maintaining system availability and data integrity under adverse conditions.
