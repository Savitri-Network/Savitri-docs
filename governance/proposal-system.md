# Proposal System

## Overview

The Savitri Network proposal system enables community members to submit, review, and vote on governance proposals. The system is designed to prevent spam proposals while allowing meaningful protocol improvements and parameter adjustments.

## Proposal Lifecycle

### Phase 1: Proposal Creation

```rust
pub struct ProposalCreation {
    pub creator: Vec<u8>,                      // Creator address
    pub deposit: u128,                        // Required deposit
    pub description: String,                   // Proposal description
    pub action: ProposalAction,              // Governance action
    pub metadata: ProposalMetadata,           // Additional metadata
}
```

### Phase 2: Review Period (24 hours)

```rust
pub struct ReviewPeriod {
    pub start_time: u64,                      // Review start
    pub end_time: u64,                        // Review end
    pub early_termination: bool,               // Can be terminated early
    pub community_feedback: Vec<Feedback>,    // Community feedback
}
```

### Phase 3: Voting Period (7 days)

```rust
pub struct VotingPeriod {
    pub start_time: u64,                      // Voting start
    pub end_time: u64,                        // Voting end
    pub quorum_required: u128,                 // Minimum voting power
    pub approval_threshold: f64,               // Approval threshold
}
```

### Phase 4: Execution

```rust
pub struct ProposalExecution {
    pub execution_height: u64,                // Execution block height
    pub execution_result: ExecutionResult,   // Execution result
    pub refund_amount: u128,                  // Deposit refund
}
```

## Proposal Types

### 1. Fee Parameter Proposals

```rust
pub struct FeeProposal {
    pub fee_treasury_bps: u16,                  // Treasury fee basis points
    pub max_fee_change_pct: u16,               // Maximum fee change percentage
    pub fee_adjustment_period: u64,           // Adjustment period
    pub justification: String,                  // Justification
}

impl FeeProposal {
    pub fn validate(&self) -> Result<(), ProposalError> {
        // 1. Validate fee bounds
        if self.fee_treasury_bps > 5000 {      // Max 50%
            return Err(ProposalError::InvalidFeeRate);
        }
        
        // 2. Validate change limits
        if self.max_fee_change_pct > 100 {     // Max 100% change
            return Err(ProposalError::ExcessiveFeeChange);
        }
        
        // 3. Validate adjustment period
        if self.fee_adjustment_period < 1000 {  // Minimum 1000 blocks
            return Err(ProposalError::InvalidAdjustmentPeriod);
        }
        
        Ok(())
    }
}
```

### 2. Slashing Policy Proposals

```rust
pub struct SlashingProposal {
    pub slash_pct_equivocation: u16,           // Equivocation slash percentage
    pub slash_pct_double_vote: u16,            // Double vote slash percentage
    pub slash_pct_invalid_attestation: u16,   // Invalid attestation slash
    pub min_bond_amount: u128,                // Minimum bond amount
    pub evidence_requirements: EvidenceRequirements, // Evidence requirements
}

pub struct EvidenceRequirements {
    pub min_evidence_age: u64,                 // Minimum evidence age
    pub max_evidence_age: u64,                 // Maximum evidence age
    pub required_signatures: u32,              // Required signatures
    pub evidence_format: EvidenceFormat,      // Evidence format
}
```

### 3. Committee Size Proposals

```rust
pub struct CommitteeProposal {
    pub min_committee_size: usize,             // Minimum committee size
    pub max_committee_size: usize,             // Maximum committee size
    pub transition_period: u64,                // Transition period
    pub security_analysis: SecurityAnalysis,   // Security analysis
}

pub struct SecurityAnalysis {
    pub byzantine_tolerance: u32,             // Byzantine fault tolerance
    pub decentralization_metrics: f64,        // Decentralization score
    pub performance_impact: PerformanceImpact, // Performance impact
    pub risk_assessment: RiskLevel,           // Risk level
}
```

### 4. Federated Learning Proposals

```rust
pub struct FLProposal {
    pub fee_treasury_bps: u16,                  // FL fee to treasury
    pub max_models: u32,                       // Maximum active models
    pub whitelist_aggregators: Vec<Vec<u8>>,   // Approved aggregators
    pub min_reputation: f64,                   // Minimum reputation
    pub model_approval_required: bool,        // Require approval
    pub privacy_requirements: PrivacyRequirements, // Privacy requirements
}

pub struct PrivacyRequirements {
    pub min_participants: u32,                 // Minimum participants
    pub data_encryption: bool,                 // Require encryption
    pub differential_privacy: bool,            // Differential privacy
    pub audit_trail: bool,                     // Audit trail required
}
```

### 5. System Upgrade Proposals

```rust
pub struct UpgradeProposal {
    pub target_version: String,                // Target version
    pub activation_height: u64,                // Activation block height
    pub rollback_enabled: bool,                // Enable rollback
    pub upgrade_path: Vec<UpgradeStep>,       // Upgrade steps
    pub compatibility_matrix: CompatibilityMatrix, // Compatibility
}

pub struct UpgradeStep {
    pub step_number: u32,                      // Step number
    pub action: UpgradeAction,                 // Upgrade action
    pub verification_required: bool,           // Verification required
    pub rollback_available: bool,              // Rollback available
}

pub enum UpgradeAction {
    ParameterUpdate { parameter: String, value: Vec<u8> },
    ContractUpgrade { contract: Vec<u8>, bytecode: Vec<u8> },
    ProtocolChange { change_type: ProtocolChange, data: Vec<u8> },
}
```

## Proposal Creation Process

### Step 1: Validation

```rust
pub struct ProposalValidator {
    pub deposit_manager: DepositManager,       // Deposit management
    pub action_validator: ActionValidator,     // Action validation
    pub metadata_validator: MetadataValidator, // Metadata validation
}

impl ProposalValidator {
    pub fn validate_proposal(&self, proposal: &ProposalCreation) -> Result<ValidationResult, ValidationError> {
        // 1. Validate deposit
        let deposit_valid = self.deposit_manager.validate_deposit(
            &proposal.creator,
            proposal.deposit,
            ProposalType::from(&proposal.action)
        )?;
        
        // 2. Validate action
        let action_valid = self.action_validator.validate_action(&proposal.action)?;
        
        // 3. Validate metadata
        let metadata_valid = self.metadata_validator.validate_metadata(&proposal.metadata)?;
        
        // 4. Check proposal limits
        self.check_proposal_limits(&proposal.creator)?;
        
        Ok(ValidationResult {
            deposit_valid,
            action_valid,
            metadata_valid,
            warnings: vec![],
        })
    }
}
```

### Step 2: Deposit Locking

```rust
pub struct DepositManager {
    pub min_deposit: u128,                    // Minimum deposit
    pub deposit_multipliers: HashMap<ProposalType, f64>, // Multipliers
    pub slashing_conditions: SlashingConditions, // Slashing rules
}

impl DepositManager {
    pub fn lock_deposit(&self, creator: &[u8], amount: u128, proposal_type: ProposalType) -> Result<DepositLock, DepositError> {
        // 1. Calculate required deposit
        let required_deposit = self.calculate_required_deposit(proposal_type);
        
        // 2. Check sufficient balance
        if amount < required_deposit {
            return Err(DepositError::InsufficientDeposit);
        }
        
        // 3. Lock deposit
        let lock = DepositLock {
            creator: creator.to_vec(),
            amount,
            proposal_type,
            locked_at: current_timestamp(),
            expires_at: current_timestamp() + PROPOSAL_LIFETIME,
        };
        
        // 4. Store lock
        self.store_deposit_lock(&lock)?;
        
        Ok(lock)
    }
    
    pub fn calculate_required_deposit(&self, proposal_type: ProposalType) -> u128 {
        let base_deposit = self.min_deposit;
        let multiplier = self.deposit_multipliers.get(&proposal_type).unwrap_or(&1.0);
        
        (base_deposit as f64 * multiplier) as u128
    }
}
```

### Step 3: Proposal Storage

```rust
impl Storage {
    pub fn store_proposal(&self, proposal: &Proposal) -> Result<u64, StorageError> {
        // 1. Generate proposal ID
        let proposal_id = self.generate_proposal_id()?;
        
        // 2. Encode proposal
        let key = self.encode_proposal_key(proposal_id);
        let value = bincode::serialize(proposal)?;
        
        // 3. Store with version
        self.put_with_version(CF_GOVERNANCE, &key, &value, PROPOSAL_VERSION)?;
        
        // 4. Update indexes
        self.update_proposal_indexes(proposal_id, proposal)?;
        
        // 5. Emit event
        self.emit_proposal_created_event(proposal_id, proposal)?;
        
        Ok(proposal_id)
    }
    
    fn update_proposal_indexes(&self, proposal_id: u64, proposal: &Proposal) -> Result<(), StorageError> {
        // Creator index
        let creator_key = self.encode_creator_index_key(&proposal.creator, proposal_id);
        self.put(CF_GOVERNANCE, &creator_key, &proposal_id.to_le_bytes())?;
        
        // Status index
        let status_key = self.encode_status_index_key(proposal.status, proposal_id);
        self.put(CF_GOVERNANCE, &status_key, &proposal_id.to_le_bytes())?;
        
        // Type index
        let type_key = self.encode_type_index_key(proposal.action.type_id(), proposal_id);
        self.put(CF_GOVERNANCE, &type_key, &proposal_id.to_le_bytes())?;
        
        Ok(())
    }
}
```

## Proposal Review Process

### Community Feedback

```rust
pub struct CommunityFeedback {
    pub feedback_id: u64,                     // Feedback ID
    pub proposal_id: u64,                     // Proposal ID
    pub reviewer: Vec<u8>,                     // Reviewer address
    pub rating: FeedbackRating,                // Rating (1-5)
    pub comments: String,                       // Comments
    pub suggestions: Vec<String>,              // Suggestions
    pub timestamp: u64,                        // Feedback timestamp
}

pub enum FeedbackRating {
    VeryPoor,    // 1 star
    Poor,        // 2 stars
    Average,     // 3 stars
    Good,        // 4 stars
    Excellent,   // 5 stars
}
```

### Early Termination

```rust
pub struct EarlyTermination {
    pub proposal_id: u64,                     // Proposal ID
    pub termination_reason: TerminationReason, // Termination reason
    pub initiator: Vec<u8>,                    // Initiator address
    pub support_votes: u128,                  // Support for termination
    pub timestamp: u64,                        // Termination timestamp
}

pub enum TerminationReason {
    SpamProposal,                              // Spam proposal
    MaliciousIntent,                           // Malicious intent
    TechnicalFlaws,                           // Technical flaws
    CommunityOpposition,                      // Community opposition
    SecurityConcerns,                         // Security concerns
}
```

## Proposal Execution

### Execution Engine

```rust
pub struct ExecutionEngine {
    pub parameter_manager: ParameterManager,  // Parameter management
    pub contract_upgrader: ContractUpgrader,  // Contract upgrades
    pub system_upgrader: SystemUpgrader,      // System upgrades
    pub fl_manager: FLManager,                 // FL management
}

impl ExecutionEngine {
    pub fn execute_proposal(&mut self, proposal: &Proposal, current_height: u64) -> Result<ExecutionResult, ExecutionError> {
        match &proposal.action {
            ProposalAction::FeeVariation { .. } => {
                self.execute_fee_change(proposal, current_height)
            },
            ProposalAction::SlashingPolicy { .. } => {
                self.execute_slashing_policy_change(proposal, current_height)
            },
            ProposalAction::CommitteeSize { .. } => {
                self.execute_committee_size_change(proposal, current_height)
            },
            ProposalAction::FederatedLearning { .. } => {
                self.execute_fl_policy_change(proposal, current_height)
            },
            ProposalAction::ApproveFLModel { .. } => {
                self.execute_fl_model_approval(proposal, current_height)
            },
            ProposalAction::SystemUpgrade { .. } => {
                self.execute_system_upgrade(proposal, current_height)
            },
        }
    }
}
```

### Parameter Updates

```rust
impl ExecutionEngine {
    fn execute_fee_change(&mut self, proposal: &Proposal, current_height: u64) -> Result<ExecutionResult, ExecutionError> {
        if let ProposalAction::FeeVariation { fee_treasury_bps, max_fee_change_pct } = &proposal.action {
            // 1. Validate parameters
            self.validate_fee_parameters(*fee_treasury_bps, *max_fee_change_pct)?;
            
            // 2. Update parameters
            let old_params = self.parameter_manager.get_fee_parameters()?;
            let new_params = FeeParameters {
                treasury_bps: *fee_treasury_bps,
                max_change_pct: *max_fee_change_pct,
                ..old_params
            };
            
            // 3. Store new parameters
            self.parameter_manager.update_fee_parameters(new_params, current_height)?;
            
            // 4. Emit event
            self.emit_parameter_updated_event("fee_parameters", current_height)?;
            
            Ok(ExecutionResult::Success {
                execution_height: current_height,
                changes_made: 2,
                gas_used: 10000,
            })
        } else {
            Err(ExecutionError::InvalidActionType)
        }
    }
}
```

### Contract Upgrades

```rust
impl ExecutionEngine {
    fn execute_contract_upgrade(&mut self, proposal: &Proposal, current_height: u64) -> Result<ExecutionResult, ExecutionError> {
        if let ProposalAction::SystemUpgrade { version, activation_height, .. } = &proposal.action {
            // 1. Validate upgrade
            self.validate_system_upgrade(version, activation_height)?;
            
            // 2. Prepare upgrade
            let upgrade_plan = self.prepare_upgrade_plan(version)?;
            
            // 3. Execute upgrade steps
            let mut changes_made = 0;
            for step in upgrade_plan.steps {
                self.execute_upgrade_step(&step, current_height)?;
                changes_made += 1;
            }
            
            // 4. Update system version
            self.update_system_version(version, current_height)?;
            
            // 5. Emit event
            self.emit_system_upgraded_event(version, current_height)?;
            
            Ok(ExecutionResult::Success {
                execution_height: current_height,
                changes_made,
                gas_used: 50000,
            })
        } else {
            Err(ExecutionError::InvalidActionType)
        }
    }
}
```

## Security Features

### Anti-Spam Measures

```rust
pub struct AntiSpamMeasures {
    pub rate_limiter: RateLimiter,             // Rate limiting
    pub content_filter: ContentFilter,         // Content filtering
    pub reputation_filter: ReputationFilter,   // Reputation filtering
    pub duplicate_detector: DuplicateDetector, // Duplicate detection
}

impl AntiSpamMeasures {
    pub fn check_proposal_spam(&self, creator: &[u8], content: &str) -> Result<SpamCheckResult, SpamError> {
        // 1. Rate limiting
        self.rate_limiter.check_rate_limit(creator)?;
        
        // 2. Content filtering
        self.content_filter.check_content(content)?;
        
        // 3. Reputation filtering
        self.reputation_filter.check_reputation(creator)?;
        
        // 4. Duplicate detection
        self.duplicate_detector.check_duplicate(content)?;
        
        Ok(SpamCheckResult::Clean)
    }
}
```

### Malicious Proposal Detection

```rust
pub struct MaliciousProposalDetector {
    pub pattern_analyzer: PatternAnalyzer,     // Pattern analysis
    pub intent_classifier: IntentClassifier,   // Intent classification
    pub risk_assessor: RiskAssessor,          // Risk assessment
}

impl MaliciousProposalDetector {
    pub fn analyze_proposal(&self, proposal: &Proposal) -> Result<RiskAssessment, AnalysisError> {
        // 1. Pattern analysis
        let patterns = self.pattern_analyzer.analyze_patterns(proposal)?;
        
        // 2. Intent classification
        let intent = self.intent_classifier.classify_intent(proposal)?;
        
        // 3. Risk assessment
        let risk = self.risk_assessor.assess_risk(patterns, intent)?;
        
        // 4. Flag if high risk
        if risk.risk_level >= RiskLevel::High {
            return Err(AnalysisError::HighRiskProposal(risk));
        }
        
        Ok(risk)
    }
}
```

## Performance Metrics

### Proposal Processing Metrics

```rust
pub struct ProposalMetrics {
    pub proposals_created: u64,               // Total proposals
    pub proposals_reviewed: u64,              // Proposals reviewed
    pub proposals_voted: u64,                 // Proposals voted on
    pub proposals_executed: u64,              // Proposals executed
    pub avg_review_time: Duration,             // Average review time
    pub avg_voting_time: Duration,             // Average voting time
    pub avg_execution_time: Duration,          // Average execution time
    pub success_rate: f64,                     // Success rate
}
```

### System Impact Metrics

```rust
pub struct SystemImpactMetrics {
    pub storage_usage: u64,                    // Storage used
    pub gas_consumption: u64,                  // Gas consumed
    pub network_bandwidth: u64,                 // Network bandwidth
    pub computational_load: f64,               // Computational load
    pub user_participation: f64,               // User participation rate
    pub decentralization_score: f64,           // Decentralization score
}
```

This proposal system provides a comprehensive framework for decentralized governance while preventing spam and malicious proposals through deposit requirements, review periods, and security measures.
