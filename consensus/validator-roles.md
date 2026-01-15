# Validator Roles and Responsibilities

## Role Hierarchy

```
┌─────────────────────────────────────────────────────────────────┐
│                        Consensus Layer                          │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ │
│  │   Leader    │ │  Follower   │ │   Observer  │ │   Backup    │ │
│  │  (Primary)  │ │ (Secondary) │ │ (Tertiary)  │ │ (Reserve)   │ │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ │
│  │ Masternode  │ │ Full Node   │ │ Light Node  │ │ Archive     │ │
│  │ (Validator) │ │ (Non-Val)   │ │ (Mobile)    │ │ (Historical)│ │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

## Consensus Roles

### 1. Leader (Primary Validator)

**Responsibilities:**
- Block proposal for current slot
- Transaction ordering and selection
- Block header construction
- Signature of proposed block

**Selection Criteria:**
```rust
pub struct LeaderSelection {
    pub pou_score: f64,                         // PoU score weight: 60%
    pub bond_amount: u128,                      // Bond amount weight: 30%
    pub random_factor: f64,                     // Randomness weight: 10%
}

impl LeaderSelection {
    pub fn compute_leader_weight(&self, validator: &Validator) -> f64 {
        let pou_weight = validator.pou_score * 0.6;
        let bond_weight = (validator.bond_amount as f64).ln() / 1_000_000.0 * 0.3;
        let random_weight = validator.random_factor * 0.1;
        
        pou_weight + bond_weight + random_weight
    }
}
```

**Leader Algorithm:**
```rust
impl SlotScheduler {
    pub fn select_leader(&self, slot: u64, committee: &[Validator]) -> Option<&Validator> {
        let epoch = slot / EPOCH_LENGTH;
        let slot_in_epoch = slot % EPOCH_LENGTH;
        
        // Compute deterministic seed
        let seed = compute_seed(epoch, slot_in_epoch, committee);
        let mut rng = StdRng::seed_from_u64(seed);
        
        // Weighted random selection based on PoU scores
        let total_weight: f64 = committee.iter()
            .map(|v| self.compute_leader_weight(v))
            .sum();
        
        let random_value: f64 = rng.gen_range(0.0..total_weight);
        let mut cumulative_weight = 0.0;
        
        for validator in committee {
            cumulative_weight += self.compute_leader_weight(validator);
            if random_value <= cumulative_weight {
                return Some(validator);
            }
        }
        
        committee.last() // Fallback
    }
}
```

**Leader Duties:**
1. **Block Construction**
   ```rust
   pub fn propose_block(&self, mempool_txs: Vec<MempoolTx>) -> Result<Block, ProposeError> {
       // 1. Select transactions using adaptive weights
       let selected_txs = self.execution_dispatcher.schedule_transactions(mempool_txs)?;
       
       // 2. Execute transactions
       let execution_results = self.execute_transactions(&selected_txs)?;
       
       // 3. Build block header
       let header = BlockHeader {
           version: CURRENT_VERSION,
           exec_height: self.current_height + 1,
           timestamp: current_timestamp(),
           parent_exec_hash: self.latest_block_hash,
           parent_ref_hash: None,
           state_root: self.compute_state_root(),
           tx_root: self.compute_tx_root(&selected_txs),
           proposer: self.validator_address,
           hash: [0; 64], // Will be computed
       };
       
       // 4. Sign block
       let signature = self.sign_block_header(&header)?;
       
       Ok(Block {
           header,
           transactions: selected_txs,
           signature,
       })
   }
   ```

2. **Proposal Broadcasting**
   ```rust
   pub fn broadcast_proposal(&self, proposal: ConsensusProposal) -> Result<(), NetworkError> {
       let message = GossipsubMessage::Consensus(ConsensusGossip {
           message_type: PROPOSAL,
           round: self.current_round,
           height: self.current_height,
           block_hash: proposal.block_hash,
           validator: self.validator_address,
           timestamp: current_timestamp(),
       });
       
       self.p2p_service.publish(TOPIC_CONSENSUS, message)?;
       Ok(())
   }
   ```

### 2. Follower (Secondary Validator)

**Responsibilities:**
- Vote on block proposals
- Validate leader proposals
- Participate in consensus rounds
- Maintain consensus state

**Voting Algorithm:**
```rust
impl FollowerValidator {
    pub fn vote_on_proposal(&self, proposal: &ConsensusProposal) -> Result<ConsensusVote, VoteError> {
       // 1. Validate proposal structure
       self.validate_proposal_structure(proposal)?;
       
       // 2. Verify block validity
       let block = self.get_block(&proposal.block_hash)?;
       self.validate_block(&block)?;
       
       // 3. Check leader eligibility
       if !self.is_valid_leader(&proposal.proposer, proposal.round) {
           return Err(VoteError::InvalidLeader);
       }
       
       // 4. Verify leader signature
       if !self.verify_signature(&proposal.signature, &proposal.proposer, &proposal.block_hash) {
           return Err(VoteError::InvalidSignature);
       }
       
       // 5. Create vote
       let vote = ConsensusVote {
           block_hash: proposal.block_hash,
           round: proposal.round,
           height: proposal.height,
           voter: self.validator_address,
           timestamp: current_timestamp(),
           signature: [0; 64], // Will be computed
       };
       
       // 6. Sign vote
       let signature = self.sign_vote(&vote)?;
       vote.signature = signature;
       
       Ok(vote)
   }
}
```

**Vote Broadcasting:**
```rust
pub fn broadcast_vote(&self, vote: ConsensusVote) -> Result<(), NetworkError> {
    let message = GossipsubMessage::Consensus(ConsensusGossip {
        message_type: VOTE,
        round: vote.round,
        height: vote.height,
        block_hash: vote.block_hash,
        validator: vote.voter,
        timestamp: vote.timestamp,
    });
    
    self.p2p_service.publish(TOPIC_CONSENSUS, message)?;
    Ok(())
}
```

### 3. Observer (Tertiary Validator)

**Responsibilities:**
- Monitor consensus progress
- Validate finality certificates
- Maintain network connectivity
- Provide fallback capacity

**Observer Implementation:**
```rust
pub struct ObserverValidator {
    pub consensus_state: ConsensusState,        // Consensus state tracking
    pub certificate_validator: CertValidator,    // Certificate validation
    pub network_monitor: NetworkMonitor,        // Network health monitoring
}

impl ObserverValidator {
    pub fn monitor_consensus(&mut self) -> Result<(), ObserverError> {
        // 1. Track consensus rounds
        let latest_messages = self.p2p_service.get_latest_consensus_messages()?;
        
        for message in latest_messages {
            match message.message_type {
                PROPOSAL => self.handle_proposal(&message)?,
                VOTE => self.handle_vote(&message)?,
                CERTIFICATE => self.handle_certificate(&message)?,
            }
        }
        
        // 2. Validate network health
        self.network_monitor.check_connectivity()?;
        
        // 3. Update consensus state
        self.consensus_state.update_state()?;
        
        Ok(())
    }
    
    pub fn validate_certificate(&self, cert: &ConsensusCertificate) -> Result<bool, CertError> {
        // 1. Verify certificate structure
        self.validate_certificate_structure(cert)?;
        
        // 2. Check signature count (2f+1 threshold)
        let required_signatures = 2 * self.consensus_state.fault_tolerance + 1;
        if cert.signatures.len() < required_signatures {
            return Ok(false);
        }
        
        // 3. Verify all signatures
        for signature in &cert.signatures {
            if !self.verify_certificate_signature(signature, cert)? {
                return Ok(false);
            }
        }
        
        // 4. Check certificate freshness
        let cert_age = current_timestamp() - cert.timestamp;
        if cert_age > CERTIFICATE_MAX_AGE {
            return Ok(false);
        }
        
        Ok(true)
    }
}
```

### 4. Backup (Reserve Validator)

**Responsibilities:**
- Maintain readiness state
- Monitor primary validator health
- Provide rapid failover capacity
- Participate in emergency consensus

**Backup Implementation:**
```rust
pub struct BackupValidator {
    pub readiness_state: ReadinessState,         // Readiness status
    pub health_monitor: HealthMonitor,           // Primary health monitoring
    pub failover_trigger: FailoverTrigger,       // Failover conditions
}

impl BackupValidator {
    pub fn monitor_primary(&mut self) -> Result<(), BackupError> {
        let primary_health = self.health_monitor.check_primary_health()?;
        
        match primary_health {
            HealthStatus::Healthy => {
                self.readiness_state = ReadinessState::Standby;
            }
            HealthStatus::Degraded => {
                self.readiness_state = ReadinessState::Ready;
            }
            HealthStatus::Failed => {
                if self.failover_trigger.should_failover()? {
                    self.initiate_failover()?;
                }
            }
        }
        
        Ok(())
    }
    
    pub fn initiate_failover(&mut self) -> Result<(), BackupError> {
        // 1. Announce failover intention
        self.broadcast_failover_intent()?;
        
        // 2. Wait for backup consensus
        let backup_consensus = self.wait_for_backup_consensus()?;
        
        if backup_consensus {
            // 3. Assume primary role
            self.transition_to_primary()?;
        }
        
        Ok(())
    }
}
```

## Node Types

### 1. Masternode (Validator Node)

**Requirements:**
- **Hardware**: 32GB RAM, 8+ CPU cores, 2TB NVMe SSD
- **Network**: 1Gbps symmetric connection, <10ms latency to peers
- **Bond**: 1,000,000 tokens minimum
- **Uptime**: 99.9% availability requirement

**Configuration:**
```rust
pub struct MasternodeConfig {
    pub validator_role: ValidatorRole,           // Consensus role
    pub bond_amount: u128,                      // Bonded amount
    pub max_peers: usize,                       // Maximum peer connections
    pub consensus_config: ConsensusConfig,       // Consensus parameters
    pub storage_config: StorageConfig,          // Storage configuration
}
```

**Services:**
```rust
pub struct MasternodeServices {
    pub consensus_engine: ConsensusEngine,       // Consensus participation
    pub execution_engine: ExecutionEngine,       // Transaction execution
    pub p2p_service: P2PService,                // Network participation
    pub storage_service: StorageService,        // Data persistence
    pub rpc_service: RPCService,                // API endpoints
    pub monitoring_service: MonitoringService,  // Health monitoring
}
```

### 2. Full Node (Non-Validator)

**Requirements:**
- **Hardware**: 16GB RAM, 4+ CPU cores, 1TB SSD
- **Network**: 100Mbps connection, <50ms latency
- **Storage**: Full blockchain state
- **Uptime**: 99% availability requirement

**Services:**
```rust
pub struct FullNodeServices {
    pub p2p_service: P2PService,                // Network participation
    pub storage_service: StorageService,        // Full state storage
    pub rpc_service: RPCService,                // Public API
    pub sync_service: SyncService,              // Blockchain synchronization
    pub mempool_service: MempoolService,        // Transaction pool
}
```

### 3. Light Node (Mobile)

**Requirements:**
- **Hardware**: 512MB RAM, mobile CPU class
- **Network**: Mobile data connection
- **Storage**: Block headers + state proofs only
- **Power**: Battery-optimized operation

**Services:**
```rust
pub struct LightNodeServices {
    pub p2p_service: P2PService,                // Limited network participation
    pub header_service: HeaderService,          // Block header verification
    pub proof_service: ProofService,            // State proof verification
    pub wallet_service: WalletService,          // Wallet functionality
    pub sync_service: LightSyncService,        // Light synchronization
}
```

**Limitations:**
- No consensus participation
- No block production
- No transaction validation beyond headers
- Limited peer connections (max 10)

### 4. Archive Node (Historical)

**Requirements:**
- **Hardware**: 64GB RAM, 8+ CPU cores, 4TB NVMe SSD
- **Network**: 1Gbps connection
- **Storage**: Complete blockchain history
- **Indexing**: Full transaction and state indexing

**Services:**
```rust
pub struct ArchiveNodeServices {
    pub full_node_services: FullNodeServices,   // All full node services
    pub archive_service: ArchiveService,        // Historical data access
    pub indexing_service: IndexingService,    // Data indexing
    pub query_service: QueryService,            // Historical queries
    pub export_service: ExportService,          // Data export
}
```

## Role Transitions

### Promotion Path
```
Observer → Follower → Leader → Masternode
    ↓         ↓         ↓         ↓
  Monitor   Vote    Propose   Govern
```

**Transition Conditions:**
```rust
pub struct RoleTransition {
    pub min_pou_score: f64,                     // Minimum PoU score
    pub min_bond_amount: u128,                  // Minimum bond requirement
    pub min_uptime: Duration,                   // Minimum uptime
    pub performance_threshold: f64,             // Performance threshold
}

impl RoleTransition {
    pub fn can_promote_to_follower(&self, validator: &ObserverValidator) -> bool {
        validator.pou_score >= self.min_pou_score
            && validator.uptime >= self.min_uptime
            && validator.performance >= self.performance_threshold
    }
    
    pub fn can_promote_to_leader(&self, validator: &FollowerValidator) -> bool {
        validator.pou_score >= self.min_pou_score * 1.2
            && validator.bond_amount >= self.min_bond_amount
            && validator.vote_participation >= 0.95
    }
    
    pub fn can_promote_to_masternode(&self, validator: &LeaderValidator) -> bool {
        validator.pou_score >= self.min_pou_score * 1.5
            && validator.bond_amount >= self.min_bond_amount * 2
            && validator.leadership_performance >= 0.98
    }
}
```

### Demotion Conditions
- **Performance Degradation**: PoU score below threshold
- **Misbehavior**: Protocol violations or double-signing
- **Insufficient Bond**: Bond amount below requirement
- **Low Availability**: Uptime below minimum

## Performance Metrics

### Role-Specific Metrics

**Leader Metrics:**
```rust
pub struct LeaderMetrics {
    pub blocks_proposed: u64,                   // Blocks proposed
    pub avg_proposal_time: Duration,            // Average proposal time
    pub proposal_success_rate: f64,             // Proposal success rate
    pub transaction_throughput: f64,            // Transaction throughput
}
```

**Follower Metrics:**
```rust
pub struct FollowerMetrics {
    pub votes_cast: u64,                        // Votes cast
    pub vote_success_rate: f64,                // Vote success rate
    pub avg_vote_time: Duration,                // Average vote time
    pub consensus_participation: f64,           // Consensus participation rate
}
```

**Observer Metrics:**
```rust
pub struct ObserverMetrics {
    pub certificates_validated: u64,            // Certificates validated
    pub network_connectivity: f64,              // Network connectivity score
    pub sync_status: SyncStatus,                // Synchronization status
    pub health_check_rate: f64,                // Health check success rate
}
```

### Performance Targets
- **Leader Block Time**: < 500ms
- **Follower Vote Time**: < 100ms
- **Observer Validation**: < 50ms
- **Role Transition**: < 5 seconds
- **Failover Time**: < 30 seconds

## Security Considerations

### Role-Based Security
```rust
pub struct RoleSecurity {
    pub leader_permissions: Vec<Permission>,     // Leader permissions
    pub follower_permissions: Vec<Permission>,   // Follower permissions
    pub observer_permissions: Vec<Permission>,   // Observer permissions
    pub backup_permissions: Vec<Permission>,     // Backup permissions
}

#[derive(Debug, Clone)]
pub enum Permission {
    ProposeBlocks,                              // Leader only
    VoteOnProposals,                            // Follower and above
    ValidateCertificates,                       // Observer and above
    InitiateFailover,                           // Backup only
    AccessFullState,                            // Masternode only
}
```

### Misbehavior Detection
```rust
pub struct MisbehaviorDetector {
    pub double_signing_detector: DoubleSigningDetector,
    pub invalid_proposal_detector: InvalidProposalDetector,
    pub vote_inconsistency_detector: VoteInconsistencyDetector,
    pub timing_anomaly_detector: TimingAnomalyDetector,
}

impl MisbehaviorDetector {
    pub fn detect_misbehavior(&self, validator: &Validator, action: &ValidatorAction) -> Option<Misbehavior> {
        match action {
            ValidatorAction::ProposeBlock(block) => {
                self.invalid_proposal_detector.check_proposal(validator, block)
            }
            ValidatorAction::Vote(vote) => {
                self.vote_inconsistency_detector.check_vote(validator, vote)
            }
            ValidatorAction::Sign(message) => {
                self.double_signing_detector.check_signature(validator, message)
            }
        }
    }
}
```

This role hierarchy provides a flexible, secure, and performant validator ecosystem for Savitri Network consensus operations.
