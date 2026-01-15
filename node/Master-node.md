# Master Node Specification

## Overview

A Savitri Network master node is a bonded network participant that serves as the validator and certifier in the Proof of Unity (PoU) consensus mechanism. Master nodes receive block proposals from proposers, validate their correctness, and issue cryptographic certificates that guarantee block finality. Master nodes are critical for network security, finality, and decentralization.

## Technology Choice Rationale

### Why Master Node Architecture

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

### Block Validation and Certification

**Master Node Role**: The master node does NOT produce blocks. Instead, it receives block proposals from proposers, validates them, and issues certificates.

**Block Validation Process**:
```rust
impl MasterNode {
    pub async fn validate_and_certify_block(
        &self,
        proposed_block: Block,
        proposer_address: Address,
    ) -> Result<CertifiedBlock, ValidationError> {
        // 1. Receive block from proposer
        let block = proposed_block;
        
        // 2. Validate block structure
        self.validate_block_structure(&block)?;
        
        // 3. Validate parent block exists and is valid
        let parent_block = self.get_block(&block.header.parent_hash)?;
        self.validate_parent_block(&block, &parent_block)?;
        
        // 4. Validate all transactions
        for tx in &block.transactions {
            self.validate_transaction(tx)?;
        }
        
        // 5. Execute transactions and verify state root
        let execution_result = self.execute_transactions(&block.transactions)?;
        if execution_result.state_root != block.header.state_root {
            return Err(ValidationError::InvalidStateRoot);
        }
        
        // 6. Verify transaction root
        let calculated_tx_root = self.calculate_transaction_root(&block.transactions)?;
        if calculated_tx_root != block.header.tx_root {
            return Err(ValidationError::InvalidTransactionRoot);
        }
        
        // 7. Verify proposer signature
        self.verify_proposer_signature(&block, proposer_address)?;
        
        // 8. Create certificate
        let certificate = self.create_certificate(&block)?;
        
        // 9. Return certified block to proposer
        Ok(CertifiedBlock {
            block,
            certificate,
            master_node_signature: self.sign_certificate(&certificate)?,
        })
    }
    
    fn create_certificate(&self, block: &Block) -> Result<Certificate, CertificateError> {
        Ok(Certificate {
            block_hash: block.hash(),
            block_number: block.header.number,
            master_node: self.address,
            timestamp: current_timestamp(),
            signature: None, // Will be signed
        })
    }
    
    fn sign_certificate(&self, certificate: &Certificate) -> Result<[u8; 96], SignatureError> {
        let message = self.serialize_certificate(certificate)?;
        self.master_key.sign(&message)
    }
}
```

### Master Node Communication Flow

**Block Proposal Reception**:
```rust
impl MasterNode {
    pub async fn handle_block_proposal(
        &mut self,
        proposal: BlockProposal,
    ) -> Result<CertifiedBlock, ValidationError> {
        // 1. Receive block proposal from proposer
        let proposer_address = proposal.proposer;
        let block = proposal.block;
        
        // 2. Validate and certify
        let certified_block = self.validate_and_certify_block(block, proposer_address).await?;
        
        // 3. Send certified block back to proposer
        self.send_certified_block_to_proposer(proposer_address, &certified_block).await?;
        
        Ok(certified_block)
    }
    
    async fn send_certified_block_to_proposer(
        &self,
        proposer: Address,
        certified_block: &CertifiedBlock,
    ) -> Result<(), NetworkError> {
        let message = MasterNodeMessage::CertifiedBlock {
            block: certified_block.clone(),
            master_node: self.address,
            timestamp: current_timestamp(),
        };
        
        self.network.send_to_proposer(proposer, message).await?;
        Ok(())
    }
}
```
    
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

This master node specification provides comprehensive guidance for operating secure and performant Savitri Network validators with proper economic incentives, security measures, and monitoring capabilities.

