# Governance System Overview

## Overview

The Savitri Network governance system enables decentralized decision-making through on-chain proposals, voting, and parameter adjustments. The system is designed to be secure, transparent, and adaptable while maintaining network stability and preventing malicious proposals.

## Technology Choice Rationale

### Why On-Chain Governance

**Problem Statement**: Blockchain networks need a secure, transparent mechanism for protocol upgrades and parameter adjustments that doesn't rely on centralized coordination.

**Chosen Solution**: On-chain governance with proposal lifecycle, voting periods, deposit requirements, and automatic execution.

**Rationale**:
- **Security**: Deposit requirements prevent spam proposals
- **Transparency**: All proposals and votes are on-chain
- **Accountability**: Voting power tied to stake/reputation
- **Flexibility**: Protocol can evolve without hard forks
- **Stability**: Time delays prevent rash decisions

**Expected Results**:
- Secure protocol evolution
- Community-driven development
- Protection against malicious proposals
- Transparent decision-making process
- Adaptive parameter tuning

## Governance Architecture

### Core Components

```rust
pub struct GovernanceSystem {
    pub proposal_engine: ProposalEngine,        // Proposal management
    pub voting_engine: VotingEngine,            // Vote processing
    pub execution_engine: ExecutionEngine,      // Proposal execution
    pub parameter_manager: ParameterManager,    // Parameter updates
    pub fl_policy_manager: FLPolicyManager,     // Federated Learning policy
}
```

### Proposal Lifecycle

```rust
pub enum ProposalStatus {
    Pending,      // 24h review period
    ActiveVoting, // 7 days voting period
    Approved,     // Proposal approved
    Rejected,     // Proposal rejected
    Executed,     // Proposal executed
    Expired,      // Voting period ended
}
```

## Proposal System

### Proposal Creation

```rust
pub struct Proposal {
    pub id: u64,                              // Unique identifier
    pub creator: Vec<u8>,                      // Creator address (32 bytes)
    pub deposit: u128,                        // Required deposit
    pub description: String,                   // Proposal description
    pub action: ProposalAction,              // Governance action
    pub status: ProposalStatus,               // Current status
    pub created_at: u64,                      // Creation timestamp
    pub review_end: u64,                       // Review period end
    pub voting_end: u64,                       // Voting period end
    pub yes_votes: u128,                       // Yes votes count
    pub no_votes: u128,                        // No votes count
    pub abstain_votes: u128,                   // Abstain votes count
}
```

### Proposal Types

```rust
pub enum ProposalAction {
    FeeVariation {
        fee_treasury_bps: u16,              // Treasury fee basis points
        max_fee_change_pct: u16,             // Max fee change percentage
    },
    SlashingPolicy {
        slash_pct_equivocation: u16,          // Equivocation slash percentage
        slash_pct_double_vote: u16,           // Double vote slash percentage
        min_bond_amount: u128,                // Minimum bond amount
    },
    CommitteeSize {
        min_committee_size: usize,           // Minimum committee size
        max_committee_size: usize,           // Maximum committee size
    },
    FederatedLearning {
        fee_treasury_bps: u16,              // FL fee to treasury
        max_models: u32,                       // Maximum models
        whitelist_aggregators: Vec<Vec<u8>>,   // Approved aggregators
    },
    ApproveFLModel {
        model_id: Vec<u8>,                    // Model identifier
    },
    SystemUpgrade {
        version: String,                      // Target version
        activation_height: u64,               // Activation block
    },
}
```

## Voting System

### Voting Power

```rust
pub struct VotingPower {
    pub vote_tokens: u128,                    // Vote token balance
    pub pou_score: f64,                       // PoU reputation score
    pub bond_amount: u128,                     // Bonded tokens
    pub time_multiplier: f64,                 // Time-based multiplier
}
```

### Vote Structure

```rust
pub struct Vote {
    pub proposal_id: u64,                     // Proposal ID
    pub voter: Vec<u8>,                         // Voter address
    pub vote_type: VoteType,                   // Vote type
    pub vote_amount: u128,                     // Voting power used
    pub timestamp: u64,                        // Vote timestamp
    pub reason: Option<String>,                // Vote reason
}
```

### Voting Types

```rust
pub enum VoteType {
    Yes,        // Vote for proposal
    No,         // Vote against proposal
    Abstain,    // Abstain from voting
}
```

## Parameter Management

### Adjustable Parameters

```rust
pub struct GovernanceParameters {
    // Fee parameters
    pub fee_treasury_bps: u16,              // Treasury fee percentage
    pub max_fee_change_pct: u16,             // Max fee change
    
    // Slashing parameters
    pub slash_pct_equivocation: u16,          // Equivocation slash
    pub slash_pct_double_vote: u16,           // Double vote slash
    pub min_bond_amount: u128,                // Minimum bond
    
    // Committee parameters
    pub min_committee_size: usize,           // Minimum committee size
    pub max_committee_size: usize,           // Maximum committee size
    
    // FL parameters
    pub fl_fee_treasury_bps: u16,            // FL fee to treasury
    pub fl_max_models: u32,                   // Maximum FL models
}
```

### Parameter Updates

```rust
pub struct ParameterUpdate {
    pub name: String,                         // Parameter name
    pub old_value: Vec<u8>,                   // Current value
    pub new_value: Vec<u8>,                   // New value
    pub reason: String,                        // Update reason
    pub effective_height: u64,                 // Effective block height
}
```

## Federated Learning Governance

### FL Policy Management

```rust
pub struct FLPolicy {
    pub fee_treasury_bps: u16,                  // Fee percentage to treasury
    pub max_models: u32,                       // Maximum active models
    pub whitelist_aggregators: Vec<Vec<u8>>,   // Approved aggregators
    pub min_reputation: f64,                   // Minimum reputation
    pub model_approval_required: bool,        // Require approval
}
```

### Model Approval

```rust
pub struct FLModelApproval {
    pub model_id: Vec<u8>,                    // Model identifier
    pub aggregator: Vec<u8>,                  // Aggregator address
    pub approval_height: u64,                 // Approval block height
    pub model_hash: Vec<u8>,                  // Model content hash
    pub metadata: ModelMetadata,              // Model metadata
}
```

## Security Features

### Deposit Requirements

```rust
pub struct DepositRequirements {
    pub min_deposit: u128,                    // Minimum deposit
    pub review_period: Duration,               // Review period
    pub voting_period: Duration,              // Voting period
    pub slashing_conditions: SlashingConditions, // Slashing rules
}
```

### Slashing Conditions

```rust
pub struct SlashingConditions {
    pub spam_proposals: bool,                   // Slash spam proposals
    pub malicious_votes: bool,                  // Slash malicious voting
    pub invalid_evidence: bool,                  // Slash invalid evidence
    pub double_voting: bool,                    // Slash double voting
}
```

## Implementation Details

### Storage Structure

```rust
// Column families
const CF_GOVERNANCE: &str = "governance";
const CF_VOTE_TOKENS: &str = "vote_tokens";
const CF_PROPOSALS: &str = "proposals";
const CF_VOTES: &str = "votes";
```

### Key Storage Keys

```rust
const NEXT_PROPOSAL_ID_KEY: &[u8] = b"next_id";
const PROPOSAL_PREFIX: u8 = 0x01;
const VOTE_PREFIX: u8 = 0x02;
const FL_POLICY_KEY: &[u8] = b"fl_policy";
```

### Proposal Storage

```rust
impl Storage {
    pub fn create_proposal(&self, proposal: &Proposal) -> Result<u64, StorageError> {
        // 1. Generate proposal ID
        let proposal_id = self.get_next_proposal_id()?;
        
        // 2. Store proposal
        let key = self.encode_proposal_key(proposal_id);
        let value = bincode::serialize(proposal)?;
        self.put(CF_GOVERNANCE, &key, &value)?;
        
        // 3. Lock deposit
        self.lock_deposit(&proposal.creator, proposal.deposit)?;
        
        Ok(proposal_id)
    }
    
    pub fn cast_vote(&self, vote: &Vote) -> Result<(), StorageError> {
        // 1. Validate voting power
        let voting_power = self.calculate_voting_power(&vote.voter)?;
        
        // 2. Check if already voted
        if self.has_voted(vote.proposal_id, &vote.voter)? {
            return Err(StorageError::AlreadyVoted);
        }
        
        // 3. Store vote
        let key = self.encode_vote_key(vote.proposal_id, &vote.voter);
        let value = bincode::serialize(vote)?;
        self.put(CF_VOTES, &key, &value)?;
        
        // 4. Update proposal vote counts
        self.update_proposal_votes(vote.proposal_id, vote.vote_type, voting_power)?;
        
        Ok(())
    }
}
```

## Performance Metrics

### Target Performance

- **Proposal Creation**: < 100ms
- **Vote Casting**: < 50ms
- **Proposal Execution**: < 200ms
- **Parameter Updates**: < 50ms
- **Storage Operations**: < 10ms

### Monitoring Metrics

```rust
pub struct GovernanceMetrics {
    pub proposals_created: u64,               // Total proposals
    pub proposals_approved: u64,              // Approved proposals
    pub proposals_rejected: u64,              // Rejected proposals
    pub total_votes_cast: u64,                 // Total votes
    pub active_proposals: u64,                // Active proposals
    pub total_deposits: u128,                  // Total deposits
    pub total_slashed: u128,                   // Total slashed amount
    pub fl_models_approved: u64,               // FL models approved
}
```

## Configuration

### Default Parameters

```rust
impl Default for GovernanceParameters {
    fn default() -> Self {
        Self {
            fee_treasury_bps: 1000,              // 10%
            max_fee_change_pct: 50,               // 50%
            slash_pct_equivocation: 5000,          // 50%
            slash_pct_double_vote: 2500,           // 25%
            min_bond_amount: 100_000_000_000_000, // 100K tokens
            min_committee_size: 4,                // Minimum 4 validators
            max_committee_size: 256,              // Maximum 256 validators
            fl_fee_treasury_bps: 500,              // 5%
            fl_max_models: 100,                    // 100 models
        }
    }
}
```

### Time Parameters

```rust
pub struct TimeParameters {
    pub review_period: Duration,               // 24 hours
    pub voting_period: Duration,               // 7 days
    pub execution_delay: Duration,              // 24 hours
    pub unbonding_period: Duration,             // 7 days
}
```

This governance system provides a comprehensive, secure, and adaptable framework for decentralized decision-making in the Savitri Network while maintaining network stability and preventing malicious proposals.
