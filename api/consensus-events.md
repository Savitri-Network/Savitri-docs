# Consensus Events API Reference

## Overview

The consensus engine emits events to notify the system about important state changes and actions. These events are used for monitoring, logging, and triggering downstream processes.

## EngineEvent Enum

```rust
#[derive(Debug, Clone)]
pub enum EngineEvent {
    /// A new round has started
    RoundStarted {
        height: u64,
        round: u32,
        is_leader: bool,
    },
    /// We should propose a block (we are leader)
    ShouldPropose { height: u64, round: u32 },
    /// A proposal was received
    ProposalReceived {
        height: u64,
        round: u32,
        block_hash: Hash64,
    },
    /// A certificate was created (block finalized)
    CertificateCreated {
        height: u64,
        round: u32,
        certificate: ConsensusCertificate,
    },
    /// Evidence of misbehavior was detected
    EvidenceDetected { evidence: ConsensusEvidence },
    /// Round timed out
    RoundTimeout { height: u64, round: u32 },
    /// Engine state changed
    StateChanged { old: EngineState, new: EngineState },
    
    // ⭐ PoU Integration Events
    /// Block finalized with proposer information
    BlockFinalized {
        height: u64,
        round: u32,
        proposer_id: [u8; 32],
        proposer_pou_score: u32,
        certificate: ConsensusCertificate,
    },
    /// Proposer score update needed
    ProposerScoreUpdate {
        proposer_id: [u8; 32],
        old_score: u32,
        new_score: u32,
        reason: ScoreUpdateReason,
    },
    /// Reward distribution triggered
    RewardDistributionTriggered {
        epoch_id: u64,
        height: u64,
        total_fees: u128,
    },
}
```

## Event Details

### RoundStarted
Emitted when a new consensus round begins.

**Fields:**
- `height: u64` - Block height for this round
- `round: u32` - Round number within the height
- `is_leader: bool` - Whether this node is the leader

**Use Cases:**
- Initialize round state
- Start timeout timers
- Update metrics

### ShouldPropose
Emitted when the node is selected as leader and should propose a block.

**Fields:**
- `height: u64` - Block height for proposal
- `round: u32` - Round number for proposal

**Use Cases:**
- Trigger block proposal logic
- Collect transactions from mempool
- Generate proposal

### ProposalReceived
Emitted when a block proposal is received from the network.

**Fields:**
- `height: u64` - Block height of proposal
- `round: u32` - Round number of proposal
- `block_hash: Hash64` - Hash of the proposed block

**Use Cases:**
- Validate proposal
- Start voting process
- Update network metrics

### CertificateCreated
Emitted when a consensus certificate is generated (block finalized).

**Fields:**
- `height: u64` - Finalized block height
- `round: u32` - Round number of finalization
- `certificate: ConsensusCertificate` - The consensus certificate

**Use Cases:**
- Commit block to storage
- Update state trie
- Notify other components

### EvidenceDetected
Emitted when misbehavior evidence is collected.

**Fields:**
- `evidence: ConsensusEvidence` - Evidence of misbehavior

**Use Cases:**
- Trigger slashing process
- Update reputation scores
- Broadcast to network

### RoundTimeout
Emitted when a round times out without reaching consensus.

**Fields:**
- `height: u64` - Block height that timed out
- `round: u32` - Round number that timed out

**Use Cases:**
- Start new round
- Update timeout parameters
- Log performance metrics

### StateChanged
Emitted when the engine state changes.

**Fields:**
- `old: EngineState` - Previous engine state
- `new: EngineState` - New engine state

**Use Cases:**
- Update monitoring dashboards
- Log state transitions
- Trigger cleanup tasks

## PoU Integration Events

### BlockFinalized
Emitted when a block is finalized with proposer PoU information.

**Fields:**
- `height: u64` - Finalized block height
- `round: u32` - Round number of finalization
- `proposer_id: [u8; 32]` - ID of the block proposer
- `proposer_pou_score: u32` - PoU score of proposer at finalization
- `certificate: ConsensusCertificate` - The consensus certificate

**Use Cases:**
- Update proposer reputation
- Calculate rewards
- Track proposer performance

### ProposerScoreUpdate
Emitted when a proposer's PoU score needs updating.

**Fields:**
- `proposer_id: [u8; 32]` - ID of the proposer
- `old_score: u32` - Previous PoU score
- `new_score: u32` - New PoU score
- `reason: ScoreUpdateReason` - Reason for score change

**Use Cases:**
- Update score cache
- Trigger reputation changes
- Log score adjustments

### RewardDistributionTriggered
Emitted when fee rewards should be distributed.

**Fields:**
- `epoch_id: u64` - Epoch for reward distribution
- `height: u64` - Block height triggering distribution
- `total_fees: u128` - Total fees to distribute

**Use Cases:**
- Calculate reward shares
- Update vote token balances
- Trigger treasury operations

## ScoreUpdateReason Enum

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum ScoreUpdateReason {
    /// Successfully proposed a block
    SuccessfulProposal,
    /// Failed to propose (timeout, invalid)
    FailedProposal,
    /// Participated in consensus (voted correctly)
    ConsensusParticipation,
    /// Misbehavior detected (equivocation, etc.)
    MisbehaviorDetected,
    /// Periodic reputation adjustment
    ReputationAdjustment,
}
```

## Event Handling

### Event Receiver Setup

```rust
use tokio::sync::mpsc;

let (event_tx, mut event_rx) = mpsc::channel(1000);

// Subscribe to consensus events
let mut engine = ConsensusEngine::new(config, committee, leader_schedule, event_tx);

// Process events
while let Some(event) = event_rx.recv().await {
    match event {
        EngineEvent::BlockFinalized { height, proposer_id, proposer_pou_score, .. } => {
            // Handle PoU-based finalization
            update_proposer_metrics(proposer_id, proposer_pou_score);
            log_block_finalization(height);
        }
        EngineEvent::ProposerScoreUpdate { proposer_id, old_score, new_score, reason } => {
            // Handle score updates
            update_score_cache(proposer_id, new_score);
            log_score_change(proposer_id, old_score, new_score, reason);
        }
        EngineEvent::RewardDistributionTriggered { epoch_id, height, total_fees } => {
            // Handle reward distribution
            distribute_rewards(epoch_id, height, total_fees);
        }
        _ => {
            // Handle other events
        }
    }
}
```

### Event Filtering

```rust
// Filter for PoU-related events only
let pou_events = event_rx
    .filter(|event| {
        matches!(event, 
            EngineEvent::BlockFinalized { .. } |
            EngineEvent::ProposerScoreUpdate { .. } |
            EngineEvent::RewardDistributionTriggered { .. }
        )
    });
```

## Performance Considerations

### Event Rate
- **RoundStarted**: ~1 per second (per height)
- **ProposalReceived**: ~1 per round (when not leader)
- **CertificateCreated**: ~1 per round (when consensus reached)
- **BlockFinalized**: ~1 per round (PoU integration)
- **ProposerScoreUpdate**: Variable (based on proposer behavior)
- **RewardDistributionTriggered**: ~1 per epoch

### Memory Usage
- Events are lightweight (few bytes each)
- Channel buffer prevents backpressure
- Clone semantics for efficient sharing

### Threading
- Events are sent from consensus thread
- Received on application thread
- No blocking operations in event handlers

## Monitoring Integration

### Metrics Collection

```rust
use prometheus::{IntCounter, Histogram};

let events_total = IntCounter::new("consensus_events_total", "Total consensus events")?;
let round_duration = Histogram::with_name("consensus_round_duration_seconds")?;

// In event handler
match event {
    EngineEvent::RoundStarted { height, round, .. } => {
        events_total.inc();
        round_start_time.set(current_timestamp());
    }
    EngineEvent::CertificateCreated { height, round, .. } => {
        let duration = current_timestamp() - round_start_time.get();
        round_duration.observe(duration);
    }
    _ => {}
}
```

### Alerting

```rust
// Alert on high timeout rate
let timeout_rate = timeouts_total as f64 / rounds_total as f64;
if timeout_rate > 0.1 {
    alert_manager.send_alert(Alert {
        level: AlertLevel::Warning,
        message: "High consensus timeout rate detected".to_string(),
        metrics: vec![("timeout_rate".to_string(), timeout_rate)],
    });
}
```

## Error Handling

### Event Channel Errors
- Channel disconnection indicates component shutdown
- Implement graceful degradation
- Log errors for debugging

### Event Processing Errors
- Isolate event handler errors
- Prevent consensus engine blocking
- Implement retry logic where appropriate

## Security Considerations

### Event Validation
- Validate event fields before processing
- Check for malformed data
- Implement rate limiting

### Privacy
- Avoid sensitive data in events
- Use anonymized IDs where appropriate
- Implement access controls for event streams

This API provides comprehensive visibility into consensus operations while maintaining performance and security for production deployments.
