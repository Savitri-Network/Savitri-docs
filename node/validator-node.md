# Validator Node Specification

## Overview

A Savitri Network validator node is a bonded network participant that actively participates in the Proof of Unity (PoU) consensus mechanism, validates blocks, produces blocks when selected as leader, and earns rewards for network security contributions. Validator nodes are critical for network security, finality, and decentralization.

## Technology Choice Rationale

### Why Validator Node Architecture

**Problem Statement**: Blockchain networks require secure, decentralized consensus mechanisms that can achieve finality while maintaining high performance and providing economic incentives for honest participation.

**Chosen Solution**: BFT-based validator system with PoU consensus, bond requirements, slashing mechanisms, and reward distribution.

**Rationale**:
- **Security**: Bond-based economic security with slashing for misbehavior
- **Performance**: BFT consensus provides fast finality and high throughput
- **Decentralization**: Distributed validator set prevents centralization
- **Incentives**: Reward system encourages honest participation
- **Accountability**: Slashing mechanisms ensure validator responsibility

**Expected Results**:
- Secure and fast consensus finality
- Economic incentives for honest behavior
- High network performance and throughput
- Decentralized validator participation
- Accountability through slashing mechanisms

## Validator Architecture

### Core Components

```rust
pub struct ValidatorNode {
    // Consensus components
    pub consensus_engine: ConsensusEngine,       // Consensus engine
    pub slot_scheduler: SlotScheduler,         // Slot scheduling
    pub evidence_collector: EvidenceCollector, // Evidence collection
    pub certificate_generator: CertGenerator,   // Certificate generation
    
    // Bond management
    pub bond_manager: BondManager,            // Bond management
    pub reward_distributor: RewardDistributor, // Reward distribution
    
    // P2P components
    pub p2p_service: P2PService,                // P2P service
    pub peer_manager: PeerManager,              // Peer management
    pub message_handler: MessageHandler,        // Message handling
    
    // Validation components
    pub block_validator: BlockValidator,       // Block validation
    pub transaction_validator: TransactionValidator, // Transaction validation
    pub signature_verifier: SignatureVerifier, // Signature verification
    
    // Monitoring components
    pub performance_monitor: PerformanceMonitor, // Performance monitoring
    pub health_checker: HealthChecker,          // Health checking
    pub metrics_collector: MetricsCollector,   // Metrics collection
}
```

### Validator Configuration

```rust
pub struct ValidatorConfig {
    // Consensus configuration
    pub committee_size: usize,                  // Committee size
    pub block_time: Duration,                    // Block time
    pub finality_threshold: u32,               // Finality threshold
    pub timeout: Duration,                      // Consensus timeout
    
    // Bond configuration
    pub bond_amount: u128,                     // Bond amount
    pub unbonding_period: Duration,             // Unbonding period
    pub slash_conditions: SlashingConditions, // Slashing conditions
    
    // Network configuration
    pub listen_port: u16,                      // P2P listen port
    pub max_peers: usize,                       // Maximum peers
    pub bootstrap_nodes: Vec<String>,          // Bootstrap nodes
    
    // Performance configuration
    pub worker_threads: usize,                  // Worker threads
    pub batch_size: usize,                      // Batch size
    pub cache_size: usize,                      // Cache size
    
    // Security configuration
    pub enable_slashing: bool,                   // Enable slashing
    pub enable_rewards: bool,                    // Enable rewards
    pub enable_monitoring: bool,                // Enable monitoring
}
```
        ## Economic Requirements

### Bond Requirements

**Minimum Bond**: 1,000,000 SAV tokens (~$1M+ USD value)
```rust
pub const DEFAULT_BOND_AMOUNT: u128 = 1_000_000_000_000_000; // 1M tokens

pub struct BondManager {
    pub active_bonds: HashMap<[u8; 32], BondInfo>,
    pub bond_history: Vec<BondEvent>,
    pub slashing_conditions: SlashingConditions,
}

impl BondManager {
    pub fn create_bond(&mut self, validator: [u8; 32], amount: u128) -> Result<BondId, BondError> {
        // 1. Validate minimum bond amount
        if amount < DEFAULT_BOND_AMOUNT {
            return Err(BondError::InsufficientBond);
        }
        
        // 2. Check validator eligibility
        if !self.is_validator_eligible(&validator)? {
            return Err(BondError::IneligibleValidator);
        }
        
        // 3. Create bond record
        let bond_id = self.generate_bond_id();
        let bond_info = BondInfo {
            validator,
            amount,
            created_at: current_timestamp(),
            status: BondStatus::Active,
            slashing_count: 0,
            last_slash_height: 0,
        };
        
        // 4. Store bond
        self.active_bonds.insert(validator, bond_info);
        
        // 5. Record bond event
        self.bond_history.push(BondEvent {
            bond_id,
            event_type: BondEventType::Created,
            validator,
            amount,
            timestamp: current_timestamp(),
        });
        
        Ok(bond_id)
    }
}
```

### Slashing Mechanism

```rust
pub struct SlashingConditions {
    pub slash_pct_equivocation: u16,              // 50% slash for equivocation
    pub slash_pct_double_vote: u16,               // 25% slash for double voting
    pub slash_pct_invalid_attestation: u16,        // 10% slash for invalid attestation
    pub slash_pct_unavailability: u16,           // 5% slash for unavailability
    pub slash_pct_invalid_signature: u16,         // 5% slash for invalid signature
    pub min_slash_amount: u128,                 // Minimum slash amount
    pub max_slash_amount: u128,                 // Maximum slash amount
}

impl SlashingConditions {
    pub fn calculate_slash_amount(&self, offense: SlashOffense, bond_amount: u128) -> u128 {
        let slash_percentage = match offense {
            SlashOffense::Equivocation => self.slash_pct_equivocation,
            SlashOffense::DoubleVote => self.slash_pct_double_vote,
            SlashOffense::InvalidAttestation => self.slash_pct_invalid_attestation,
            SlashOffense::Unavailability => self.slash_pct_unavailability,
            SlashOffense::InvalidSignature => self.slash_pct_invalid_signature,
        };
        
        let slash_amount = (bond_amount as f64 * slash_percentage as f64 / 10000.0) as u128;
        
        // Apply bounds
        slash_amount.max(self.min_slash_amount).min(self.max_slash_amount)
    }
}
```

## Consensus Participation

### Slot Scheduling

```rust
pub struct SlotScheduler {
    pub committee: Vec<ValidatorInfo>,          // Current committee
    pub slot_duration: Duration,                // Slot duration
    pub current_slot: u64,                     // Current slot
    pub epoch_length: u64,                     // Epoch length
    pub current_epoch: u64,                    // Current epoch
}

impl SlotScheduler {
    pub fn get_slot_role(&self, validator_id: &[u8; 32]) -> SlotRole {
        // 1. Check if validator is in committee
        if !self.is_in_committee(validator_id) {
            return SlotRole::Observer;
        }
        
        // 2. Check if validator is leader
        if self.is_leader(validator_id, self.current_slot) {
            return SlotRole::Leader;
        }
        
        // 3. Check if validator is follower
        return SlotRole::Follower;
    }
    
    pub fn is_leader(&self, validator_id: &[u8; 32], slot: u64) -> bool {
        let leader = self.get_leader_for_slot(slot);
        leader == validator_id
    }
    
    pub fn get_leader_for_slot(&self, slot: u64) -> [u8; 32] {
        let committee_index = (slot % self.committee.len()) as usize;
        self.committee[committee_index].validator_id
    }
}
```

### Block Production

```rust
pub struct BlockProducer {
    pub mempool: Arc<Mempool>,                  // Transaction pool
    pub executor: Arc<Executor>,                // Transaction executor
    pub block_builder: BlockBuilder,           // Block builder
    pub reward_calculator: RewardCalculator,   // Reward calculator
}

impl BlockProducer {
    pub fn produce_block(&mut self, slot: u64) -> Result<Block, ProductionError> {
        // 1. Get transactions from mempool
        let transactions = self.mempool.get_transactions_for_slot(slot)?;
        
        // 2. Execute transactions
        let execution_result = self.executor.execute_transactions(&transactions)?;
        
        // 3. Build block
        let block = self.block_builder.build_block(slot, transactions, execution_result)?;
        
        // 4. Calculate rewards
        let rewards = self.reward_calculator.calculate_rewards(&block)?;
        
        // 5. Add rewards to block
        let final_block = self.block_builder.add_rewards(block, rewards)?;
        
        Ok(final_block)
    }
}
```
            status: BondStatus::Active,
            slashing_count: 0,
        };
        
        // 4. Store bond
        self.active_bonds.insert(validator, bond_info);
        
        // 5. Update validator set
        self.update_validator_set()?;
        
        Ok(bond_id)
    }
}
```

**Bond Economics**:
- **Bond Lock Period**: 30 days minimum
- **Unbonding Period**: 21 days withdrawal delay
- **Slashing Penalties**: Up to 50% of bond amount
- **Reward Rate**: Variable based on network participation

### Reward Structure

**Reward Calculation**:
```rust
pub struct RewardCalculator {
    pub base_reward_rate: f64,              // Base reward per block
    pub participation_multiplier: f64,     // Participation bonus
    pub uptime_multiplier: f64,             // Uptime bonus
    pub performance_multiplier: f64,       // Performance bonus
}

impl RewardCalculator {
    pub fn calculate_validator_reward(&self, validator: &ValidatorInfo, block_height: u64) -> u128 {
        let base_reward = (self.base_reward_rate * 1e18) as u128;
        
        // Participation bonus (blocks participated / total blocks)
        let participation_bonus = (validator.participation_rate * self.participation_multiplier) as u128;
        
        // Uptime bonus (time online / total time)
        let uptime_bonus = (validator.uptime_rate * self.uptime_multiplier) as u128;
        
        // Performance bonus (blocks produced / expected blocks)
        let performance_bonus = (validator.performance_rate * self.performance_multiplier) as u128;
        
        let total_multiplier = 1.0 + participation_bonus + uptime_bonus + performance_bonus;
        
        (base_reward * total_multiplier as u128) / 1000
    }
}
```

**Reward Distribution**:
- **Block Rewards**: Distributed to leader and active validators
- **Transaction Fees**: Shared among participating validators
- **Network Fees**: Distributed based on participation and performance
- **Slash Penalties**: Distributed to honest validators when misbehavior detected

## Hardware Requirements

### Minimum Specifications

**CPU**: 16 cores, 3.0GHz+ (Intel/AMD x64)
- **Rationale**: Consensus processing, block production, parallel validation
- **Expected Performance**: 2000+ TPS validation capacity

**Memory**: 32GB RAM
- **Rationale**: State caching, consensus state, transaction pool
- **Expected Usage**: 16GB for state, 8GB for consensus, 8GB for system

**Storage**: 1TB NVMe SSD
- **Rationale**: Blockchain storage, consensus state, fast I/O for block production
- **Expected Growth**: ~5GB per month with current network activity

**Network**: 1Gbps+ symmetric connection
- **Rationale**: Consensus messaging, block propagation, low latency requirements
- **Expected Bandwidth**: 100GB+ per month for consensus and P2P

### Recommended Specifications

**CPU**: 32 cores, 3.5GHz+ (Intel/AMD x64)
- **Benefits**: Better consensus performance, higher TPS, parallel processing

**Memory**: 64GB RAM
- **Benefits**: Larger state cache, better consensus performance, higher mempool

**Storage**: 2TB NVMe SSD
- **Benefits**: Extended historical data, better I/O performance

**Network**: 10Gbps+ symmetric connection
- **Benefits**: Faster consensus messaging, better block propagation

## Consensus Participation

### Proof of Unity (PoU) Overview

**Consensus Algorithm**:
```rust
pub struct ProofOfUnity {
    pub validator_set: ValidatorSet,
    pub current_epoch: u64,
    pub slot_scheduler: SlotScheduler,
    pub vote_aggregator: VoteAggregator,
    pub finality_tracker: FinalityTracker,
}

impl ProofOfUnity {
    pub fn participate_in_consensus(&mut self, validator_key: &ValidatorKey) -> Result<ConsensusResult, ConsensusError> {
        // 1. Get current slot
        let current_slot = self.slot_scheduler.get_current_slot()?;
        
        // 2. Determine validator role
        let role = self.determine_validator_role(validator_key, &current_slot)?;
        
        match role {
            SlotRole::Leader => {
                // 3. Produce block as leader
                let block = self.produce_block(validator_key, &current_slot)?;
                
                // 4. Broadcast block
                self.broadcast_block(&block)?;
                
                // 5. Collect votes
                let votes = self.collect_votes(&block, current_slot.timeout)?;
                
                // 6. Achieve finality
                let finality = self.achieve_finality(&block, votes)?;
                
                Ok(ConsensusResult::BlockProduced { block, finality })
            },
            SlotRole::Follower => {
                // 3. Validate received block
                let block = self.wait_for_block(&current_slot)?;
                self.validate_block(&block)?;
                
                // 4. Vote on block
                let vote = self.create_vote(&block, validator_key)?;
                self.broadcast_vote(&vote)?;
                
                // 5. Wait for finality
                let finality = self.wait_for_finality(&block)?;
                
                Ok(ConsensusResult::BlockValidated { block, finality })
            },
            SlotRole::Observer => {
                // 3. Observe consensus process
                let finality = self.observe_consensus(&current_slot)?;
                Ok(ConsensusResult::Observed { finality })
            },
        }
    }
}
```

### Slot Scheduling

**Slot Assignment**:
```rust
pub struct SlotScheduler {
    pub validator_set: ValidatorSet,
    pub epoch_duration: Duration,
    pub slot_duration: Duration,
    pub current_epoch: u64,
}

impl SlotScheduler {
    pub fn get_slot_leader(&self, slot: u64) -> Result<ValidatorAddress, SchedulingError> {
        let epoch = slot / self.slots_per_epoch();
        let slot_in_epoch = slot % self.slots_per_epoch();
        
        // Use VRF to determine leader
        let leader_index = self.vrf_leader_selection(epoch, slot_in_epoch)?;
        
        if leader_index >= self.validator_set.len() {
            return Err(SchedulingError::InvalidLeaderIndex);
        }
        
        Ok(self.validator_set[leader_index].address)
    }
    
    pub fn get_validator_role(&self, validator: &ValidatorAddress, slot: u64) -> SlotRole {
        let leader = match self.get_slot_leader(slot) {
            Ok(leader) => leader,
            Err(_) => return SlotRole::Observer,
        };
        
        if leader == *validator {
            SlotRole::Leader
        } else if self.validator_set.contains(validator) {
            SlotRole::Follower
        } else {
            SlotRole::Observer
        }
    }
}
```

### Block Production

**Block Construction**:
```rust
impl BlockProducer {
    pub fn produce_block(&self, validator_key: &ValidatorKey, slot: &SlotInfo) -> Result<Block, ProductionError> {
        // 1. Get parent block
        let parent_block = self.get_parent_block()?;
        
        // 2. Select transactions from mempool
        let selected_txs = self.select_transactions()?;
        
        // 3. Execute transactions
        let execution_result = self.execute_transactions(&selected_txs)?;
        
        // 4. Create block header
        let header = BlockHeader {
            version: 1,
            exec_height: parent_block.exec_height + 1,
            timestamp: current_timestamp(),
            parent_exec_hash: parent_block.hash,
            parent_ref_hash: None,
            state_root: execution_result.state_root,
            tx_root: execution_result.tx_root,
            proposer: validator_key.address,
        };
        
        // 5. Create block
        let block = Block {
            header,
            transactions: selected_txs,
            certificate: None, // To be filled by consensus
        };
        
        // 6. Sign block
        let signature = validator_key.sign_block(&block)?;
        
        Ok(Block {
            certificate: Some(Certificate {
                proposer_signature: signature,
                slot_number: slot.number,
                epoch_number: slot.epoch,
            }),
            ..block
        })
    }
}
```

### Vote Aggregation

**Vote Collection**:
```rust
pub struct VoteAggregator {
    pub validator_set: ValidatorSet,
    pub pending_votes: HashMap<BlockHash, Vec<Vote>>,
    pub vote_threshold: f64,  // 2/3 + 1 for BFT
}

impl VoteAggregator {
    pub fn collect_votes(&mut self, block: &Block, timeout: Duration) -> Result<Vec<Vote>, VoteError> {
        let mut votes = Vec::new();
        let start_time = Instant::now();
        
        // 1. Broadcast vote request
        self.broadcast_vote_request(&block)?;
        
        // 2. Collect votes with timeout
        while start_time.elapsed() < timeout {
            if let Some(new_votes) = self.receive_votes()? {
                votes.extend(new_votes);
                
                // 3. Check if we have enough votes
                if self.has_quorum(&votes)? {
                    break;
                }
            }
            
            tokio::time::sleep(Duration::from_millis(100)).await;
        }
        
        // 4. Validate votes
        let valid_votes = self.validate_votes(&votes)?;
        
        // 5. Check quorum
        if !self.has_quorum(&valid_votes)? {
            return Err(VoteError::InsufficientVotes);
        }
        
        Ok(valid_votes)
    }
    
    fn has_quorum(&self, votes: &[Vote]) -> Result<bool, VoteError> {
        let total_voting_power = self.validator_set.total_voting_power();
        let collected_power: u64 = votes.iter()
            .map(|v| self.validator_set.get_voting_power(&v.validator))
            .sum();
        
        let threshold = (total_voting_power as f64 * self.vote_threshold) as u64;
        Ok(collected_power >= threshold)
    }
}
```

## Security Model

### Slashing Conditions

**Slashing Types**:
```rust
pub enum SlashingCondition {
    // Equivocation
    DoubleSign {
        block_height: u64,
        first_signature: Signature,
        second_signature: Signature,
    },
    
    // Unavailable
    Unavailable {
        missed_slots: u64,
        total_slots: u64,
        period: Duration,
    },
    
    // Invalid Block
    InvalidBlock {
        block_hash: BlockHash,
        reason: InvalidBlockReason,
    },
    
    // Vote Manipulation
    VoteManipulation {
        conflicting_votes: Vec<Vote>,
    },
}

impl SlashingManager {
    pub fn check_slashing_conditions(&self, validator: &ValidatorAddress) -> Result<SlashingResult, SlashingError> {
        // 1. Check for equivocation
        if let Some(equivocation) = self.detect_equivocation(validator)? {
            return Ok(SlashingResult {
                condition: SlashingCondition::DoubleSign(equivocation),
                penalty: self.calculate_slashing_penalty(&SlashingCondition::DoubleSign(equivocation))?,
            });
        }
        
        // 2. Check availability
        if let Some(unavailability) = self.check_unavailability(validator)? {
            return Ok(SlashingResult {
                condition: SlashingCondition::Unavailable(unavailability),
                penalty: self.calculate_slashing_penalty(&SlashingCondition::Unavailable(unavailability))?,
            });
        }
        
        // 3. Check for invalid blocks
        if let Some(invalid_block) = self.detect_invalid_blocks(validator)? {
            return Ok(SlashingResult {
                condition: SlashingCondition::InvalidBlock(invalid_block),
                penalty: self.calculate_slashing_penalty(&SlashingCondition::InvalidBlock(invalid_block))?,
            });
        }
        
        Ok(SlashingResult::NoSlashing)
    }
    
    fn calculate_slashing_penalty(&self, condition: &SlashingCondition) -> Result<u128, SlashingError> {
        let bond_amount = self.get_validator_bond(&condition.validator())?;
        
        let penalty_percentage = match condition {
            SlashingCondition::DoubleSign(_) => 0.5,      // 50% slash
            SlashingCondition::Unavailable { .. } => {
                let availability_rate = condition.availability_rate();
                if availability_rate < 0.8 { 0.1 } else { 0.0 }  // 10% for <80% availability
            },
            SlashingCondition::InvalidBlock(_) => 0.3,      // 30% slash
            SlashingCondition::VoteManipulation(_) => 0.4,   // 40% slash
        };
        
        Ok((bond_amount * penalty_percentage) as u128)
    }
}
```

### Key Management

**Validator Key Security**:
```rust
pub struct ValidatorKeyManager {
    pub validator_key: ValidatorKey,
    pub encryption_key: ChaCha20Poly1305Key,
    pub hardware_wallet: Option<HardwareWallet>,
}

impl ValidatorKeyManager {
    pub fn sign_block(&self, block: &Block) -> Result<Signature, KeyError> {
        // 1. Use hardware wallet if available
        if let Some(hw_wallet) = &self.hardware_wallet {
            return hw_wallet.sign_block(block);
        }
        
        // 2. Use software key with HSM protection
        let message = self.prepare_block_message(block)?;
        let signature = self.validator_key.sign(&message)?;
        
        // 3. Verify signature
        if !self.verify_signature(&signature, &message)? {
            return Err(KeyError::SignatureVerificationFailed);
        }
        
        Ok(signature)
    }
    
    pub fn rotate_keys(&mut self) -> Result<KeyRotationResult, KeyError> {
        // 1. Generate new key
        let new_key = ValidatorKey::generate()?;
        
        // 2. Create key rotation transaction
        let rotation_tx = self.create_key_rotation_tx(&new_key)?;
        
        // 3. Submit to network
        let tx_hash = self.submit_transaction(rotation_tx)?;
        
        // 4. Wait for confirmation
        self.wait_for_confirmation(tx_hash).await?;
        
        // 5. Update local key
        self.validator_key = new_key;
        
        Ok(KeyRotationResult {
            new_key_address: new_key.address(),
            rotation_tx_hash: tx_hash,
        })
    }
}
```

## Performance Optimization

### Consensus Optimization

**Parallel Processing**:
```rust
pub struct OptimizedConsensusEngine {
    pub parallel_validator: ParallelValidator,
    pub batch_processor: BatchProcessor,
    pub vote_cache: VoteCache,
}

impl OptimizedConsensusEngine {
    pub async fn process_consensus_batch(&mut self, proposals: Vec<BlockProposal>) -> Result<Vec<ConsensusResult>, ConsensusError> {
        // 1. Validate proposals in parallel
        let validation_tasks = proposals.iter()
            .map(|proposal| self.parallel_validator.validate_proposal(proposal));
        
        let validation_results = futures::future::join_all(validation_tasks).await;
        
        // 2. Filter valid proposals
        let valid_proposals: Vec<_> = proposals.into_iter()
            .zip(validation_results)
            .filter_map(|(proposal, result)| {
                result.ok().map(|_| proposal)
            })
            .collect();
        
        // 3. Process valid proposals
        let processing_tasks = valid_proposals.into_iter()
            .map(|proposal| self.process_single_proposal(proposal));
        
        let processing_results = futures::future::join_all(processing_tasks).await;
        
        Ok(processing_results)
    }
}
```

### Memory Optimization

**State Caching**:
```rust
pub struct StateCache {
    pub account_cache: LruCache<Address, Account>,
    pub contract_cache: LruCache<Address, ContractState>,
    pub storage_cache: LruCache<(Address, [u8; 32]), Vec<u8>>,
    pub cache_stats: CacheStats,
}

impl StateCache {
    pub fn get_account(&mut self, address: &Address) -> Option<Account> {
        if let Some(account) = self.account_cache.get(address) {
            self.cache_stats.record_hit(CacheType::Account);
            Some(account)
        } else {
            self.cache_stats.record_miss(CacheType::Account);
            None
        }
    }
    
    pub fn cache_account(&mut self, address: Address, account: Account) {
        self.account_cache.put(address, account);
    }
    
    pub fn optimize_cache_size(&mut self, memory_limit: usize) {
        let current_usage = self.calculate_memory_usage();
        
        if current_usage > memory_limit {
            let reduction = current_usage - memory_limit;
            self.reduce_cache_size(reduction);
        }
    }
}
```

## Monitoring and Maintenance

### Validator Metrics

**Key Performance Indicators**:
```rust
pub struct ValidatorMetrics {
    // Consensus metrics
    pub blocks_produced: u64,
    pub blocks_validated: u64,
    pub consensus_participation: f64,
    pub vote_success_rate: f64,
    
    // Performance metrics
    pub block_production_time: Duration,
    pub vote_latency: Duration,
    pub finality_time: Duration,
    pub tps_processed: f64,
    
    // Economic metrics
    pub rewards_earned: u128,
    pub slashing_penalties: u128,
    pub net_profit: u128,
    pub roi: f64,
    
    // Availability metrics
    pub uptime_percentage: f64,
    pub missed_slots: u64,
    pub network_connectivity: f64,
}
```

### Health Monitoring

**Health Checks**:
```rust
impl ValidatorHealthChecker {
    pub async fn check_health(&self) -> HealthStatus {
        let mut status = HealthStatus::Healthy;
        
        // Check consensus participation
        if self.consensus_participation < 0.95 {
            status = HealthStatus::Degraded("Low consensus participation");
        }
        
        // Check network connectivity
        if self.peer_count() < MIN_PEERS {
            status = HealthStatus::Degraded("Low peer connectivity");
        }
        
        // Check performance
        if self.block_production_time > MAX_PRODUCTION_TIME {
            status = HealthStatus::Critical("Slow block production");
        }
        
        // Check availability
        if self.uptime_percentage < 0.99 {
            status = HealthStatus::Critical("Low availability");
        }
        
        // Check economic health
        if self.net_profit < 0 {
            status = HealthStatus::Warning("Negative profitability");
        }
        
        status
    }
}
```

## Troubleshooting

### Common Issues

**1. Consensus Timeout**
```bash
# Check network connectivity
curl -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"net_peerCount","params":[],"id":1}'

# Check consensus status
curl -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"consensus_status","params":[],"id":1}'

# Restart consensus engine
curl -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"admin_restartConsensus","params":[],"id":1}'
```

**2. Slashing Detection**
```bash
# Check slashing status
curl -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"validator_slashingStatus","params":[],"id":1}'

# Check validator performance
curl -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"validator_performance","params":[],"id":1}'
```

**3. Bond Issues**
```bash
# Check bond status
curl -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"validator_bondStatus","params":["0x..."],"id":1}'

# Check reward distribution
curl -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"validator_rewards","params":["0x..."],"id":1}'
```

This validator node specification provides comprehensive guidance for operating secure and performant Savitri Network validators with proper economic incentives, security measures, and monitoring capabilities.
