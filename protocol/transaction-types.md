# Transaction Types Specification

## Transaction Overview

Savitri Network implements a sophisticated transaction system with multiple transaction types, each designed for specific use cases while maintaining security, efficiency, and composability. The transaction system supports everything from simple transfers to complex smart contract interactions and governance operations.

## Technology Choice Rationale

### Why Multi-Type Transaction System

**Problem Statement**: Single transaction type systems force all operations through a generic interface, leading to inefficient encoding, unnecessary complexity, and poor user experience for common operations.

**Chosen Solution**: Multi-type transaction system with optimized encoding, specialized validation, and type-specific execution paths.

**Rationale**:
- **Efficiency**: Optimized encoding for each transaction type reduces gas costs
- **Clarity**: Clear transaction types improve user experience and tooling
- **Validation**: Type-specific validation enhances security
- **Composability**: Standardized interfaces enable complex operations

**Expected Results**:
- 20-40% reduction in gas costs through optimized encoding
- Improved developer and user experience
- Enhanced security through type-specific validation
- Better tooling support and ecosystem development

### Why Unified Transaction Interface

**Problem Statement**: Completely separate transaction types would lead to code duplication, inconsistent behavior, and maintenance challenges.

**Chosen Solution**: Unified transaction interface with type-specific implementations while maintaining common behaviors.

**Rationale**:
- **Consistency**: Common behaviors across all transaction types
- **Maintainability**: Shared validation and execution logic
- **Extensibility**: Easy addition of new transaction types
- **Interoperability**: Consistent handling across the ecosystem

**Expected Results**:
- Reduced code duplication and maintenance overhead
- Consistent behavior across transaction types
- Easy addition of new transaction types
- Better ecosystem interoperability

## Transaction Type System

### Base Transaction Structure
```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SignedTx {
    pub from: Vec<u8>,                          // Sender address
    pub to: Vec<u8>,                            // Recipient address
    pub amount: u128,                           // Transfer amount
    pub nonce: u64,                             // Sender nonce
    pub fee: Option<u128>,                      // Transaction fee
    pub pubkey: [u8; 32],                       // Public key
    #[serde(with = "serde_big_array::BigArray")]
    pub sig: [u8; 64],                         // Ed25519 signature
    pub pre_verified: bool,                     // Pre-verification flag
}
```

**Key Design Decisions:**
- **Simplified Structure**: Minimal fields for efficiency
- **Ed25519 Signatures**: Fast verification and security
- **Optional Fee**: Flexible fee structure
- **Pre-verification**: Optimized validation path
- **Nonce-based Ordering**: Prevents replay attacks

### Transaction Types

#### 1. Native Transfer Transaction
```rust
// Simplified transfer using base SignedTx structure
pub fn create_transfer_transaction(
    from: Vec<u8>,
    to: Vec<u8>,
    amount: u128,
    nonce: u64,
    fee: u128,
    keypair: &Keypair,
) -> SignedTx {
    let message = encode_transfer_message(&from, &to, amount, nonce);
    let signature = keypair.sign(&message);
    
    SignedTx {
        from,
        to,
        amount,
        nonce,
        fee: Some(fee),
        pubkey: keypair.public.to_bytes(),
        sig: signature.to_bytes(),
        pre_verified: false,
    }
}

fn encode_transfer_message(from: &[u8], to: &[u8], amount: u128, nonce: u64) -> Vec<u8> {
    let mut message = Vec::new();
    message.extend_from_slice(b"savitri-tx-v1"); // Domain separator
    message.extend_from_slice(from);
    message.extend_from_slice(to);
    message.extend_from_slice(&amount.to_be_bytes());
    message.extend_from_slice(&nonce.to_be_bytes());
    message
}
```

#### 2. Contract Call Transaction
```rust
pub fn create_contract_call(
    from: Vec<u8>,
    contract: Vec<u8>,
    data: Vec<u8>,
    value: u128,
    nonce: u64,
    fee: u128,
    keypair: &Keypair,
) -> SignedTx {
    let message = encode_contract_call_message(&from, &contract, &data, value, nonce);
    let signature = keypair.sign(&message);
    
    SignedTx {
        from,
        to: contract,
        amount: value,
        nonce,
        fee: Some(fee),
        pubkey: keypair.public.to_bytes(),
        sig: signature.to_bytes(),
        pre_verified: false,
    }
}
```

#### 3. Governance Transaction
```rust
pub fn create_governance_transaction(
    from: Vec<u8>,
    proposal_id: u64,
    vote: bool,
    nonce: u64,
    fee: u128,
    keypair: &Keypair,
) -> SignedTx {
    let message = encode_governance_message(&from, proposal_id, vote, nonce);
    let signature = keypair.sign(&message);
    
    SignedTx {
        from,
        to: vec![0u8; 32], // Governance contract address
        amount: 0,
        nonce,
        fee: Some(fee),
        pubkey: keypair.public.to_bytes(),
        sig: signature.to_bytes(),
        pre_verified: false,
    }
}
```

## Transaction Validation

### Validation Pipeline
```rust
pub struct TransactionValidator {
    pub storage: Arc<Storage>,
    pub max_tx_size: usize,                    // 1MB max size
    pub min_fee: u128,                        // Minimum fee
    pub max_fee: u128,                        // Maximum fee
}

impl TransactionValidator {
    pub fn validate_transaction(&self, tx: &SignedTx) -> Result<ValidationResult, ValidationError> {
        // 1. Basic validation
        self.validate_basic_structure(tx)?;
        
        // 2. Signature verification
        self.verify_signature(tx)?;
        
        // 3. Account validation
        self.validate_accounts(tx)?;
        
        // 4. Nonce validation
        self.validate_nonce(tx)?;
        
        // 5. Balance validation
        self.validate_balance(tx)?;
        
        Ok(ValidationResult::Valid)
    }
    
    fn validate_basic_structure(&self, tx: &SignedTx) -> Result<(), ValidationError> {
        // Check transaction size
        let serialized = bincode::serialize(tx).map_err(|_| ValidationError::SerializationError)?;
        if serialized.len() > self.max_tx_size {
            return Err(ValidationError::TooLarge);
        }
        
        // Check addresses
        if tx.from.len() != 32 || tx.to.len() != 32 {
            return Err(ValidationError::InvalidAddress);
        }
        
        // Check amount
        if tx.amount == 0 {
            return Err(ValidationError::ZeroAmount);
        }
        
        Ok(())
    }
    
    fn verify_signature(&self, tx: &SignedTx) -> Result<(), ValidationError> {
        let message = self.encode_message_for_signing(tx);
        let public_key = PublicKey::from_bytes(&tx.pubkey)
            .map_err(|_| ValidationError::InvalidPublicKey)?;
        
        let signature = Signature::from_bytes(&tx.sig)
            .map_err(|_| ValidationError::InvalidSignature)?;
        
        public_key.verify(&message, &signature)
            .map_err(|_| ValidationError::SignatureVerificationFailed)
    }
    
    fn validate_nonce(&self, tx: &SignedTx) -> Result<(), ValidationError> {
        let account = self.storage.get_account(&tx.from)
            .map_err(|_| ValidationError::AccountNotFound)?;
        
        if tx.nonce != account.nonce {
            return Err(ValidationError::InvalidNonce);
        }
        
        Ok(())
    }
    
    fn validate_balance(&self, tx: &SignedTx) -> Result<(), ValidationError> {
        let account = self.storage.get_account(&tx.from)
            .map_err(|_| ValidationError::AccountNotFound)?;
        
        let total_cost = tx.amount + tx.fee.unwrap_or(0);
        if account.balance < total_cost {
            return Err(ValidationError::InsufficientBalance);
        }
        
        Ok(())
    }
}
```

## Transaction Classification

### Transaction Classes
```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum TxClass {
    /// Financial transactions (transfers, payments)
    Financial,
    /// IoT data submissions (sensor readings, telemetry)
    IoTData,
    /// Federated learning updates (model gradients, aggregations)
    FederatedUpdate,
    /// System transactions (governance, upgrades, maintenance)
    System,
}

impl TxClass {
    /// Classify transaction based on content/pattern
    pub fn from_tx_bytes(_bytes: &[u8]) -> Self {
        // TODO: Implement actual classification logic
        // For now, default to Financial
        TxClass::Financial
    }
}
```

### Class-Aware Processing
```rust
pub struct TransactionClassifier {
    pub patterns: HashMap<Vec<u8>, TxClass>,      // Known patterns
    pub ml_model: Option<Box<dyn MLClassifier>>,  // ML-based classification
}

impl TransactionClassifier {
    pub fn classify_transaction(&self, tx: &SignedTx) -> TxClass {
        // 1. Check known patterns first
        if let Some(tx_class) = self.patterns.get(&tx.to) {
            return *tx_class;
        }
        
        // 2. Use ML model if available
        if let Some(model) = &self.ml_model {
            let features = self.extract_features(tx);
            return model.classify(features);
        }
        
        // 3. Default classification
        self.default_classification(tx)
    }
    
    fn default_classification(&self, tx: &SignedTx) -> TxClass {
        // Simple heuristic classification
        if tx.amount > 0 && tx.to != [0u8; 32] {
            TxClass::Financial
        } else if tx.to == [0u8; 32] {
            TxClass::System
        } else {
            TxClass::Financial // Default
        }
    }
}
```

## Performance Optimization

### SIMD Transaction Processing
```rust
#[cfg(target_arch = "x86_64")]
#[target_feature(enable = "avx2")]
unsafe fn process_transactions_simd_batch(
    transactions: &[SignedTx],
    weights: &[f64],
) -> Vec<f64> {
    let mut scores = vec![0.0; transactions.len()];
    
    // Process transactions in batches of 4 (AVX2 width)
    for (chunk_idx, tx_chunk) in transactions.chunks_exact(4).enumerate() {
        let base_idx = chunk_idx * 4;
        
        // Load transaction amounts into SIMD register
        let amounts_vec = _mm256_loadu_pd([
            tx_chunk[0].amount as f64,
            tx_chunk[1].amount as f64,
            tx_chunk[2].amount as f64,
            tx_chunk[3].amount as f64,
        ].as_ptr());
        
        // Load weights into SIMD register
        let weights_vec = _mm256_loadu_pd([
            weights[base_idx],
            weights[base_idx + 1],
            weights[base_idx + 2],
            weights[base_idx + 3],
        ].as_ptr());
        
        // Compute scores
        let scores_vec = _mm256_mul_pd(amounts_vec, weights_vec);
        
        // Store results
        _mm256_storeu_pd(scores.as_mut_ptr().add(base_idx), scores_vec);
    }
    
    // Handle remaining transactions with scalar computation
    let remaining_start = (transactions.len() / 4) * 4;
    for i in remaining_start..transactions.len() {
        scores[i] = transactions[i].amount as f64 * weights[i];
    }
    
    scores
}
```

### Memory-Efficient Transaction Storage
```rust
pub struct CompactTransactionPool {
    pub transactions: Vec<CompactTx>,            // Compact storage
    pub full_tx_cache: LRUCache<TxHash, SignedTx>, // Full transaction cache
    pub index_by_sender: HashMap<[u8; 32], Vec<usize>>, // Sender index
    pub index_by_nonce: HashMap<( [u8; 32], u64), usize>, // Nonce index
}

#[derive(Debug, Clone)]
pub struct CompactTx {
    pub tx_hash: TxHash,                        // Transaction hash
    pub sender_id: u32,                          // Compact sender ID
    pub nonce: u64,                              // Transaction nonce
    pub fee: u64,                                // Transaction fee
    pub class: TxClass,                          // Transaction class
    pub priority: f64,                           // Priority score
}

impl CompactTransactionPool {
    pub fn add_transaction(&mut self, tx: SignedTx) -> Result<(), PoolError> {
        let compact_tx = self.to_compact(&tx);
        let index = self.transactions.len();
        
        // Update indexes
        self.index_by_sender
            .entry(tx.from.try_into().unwrap())
            .or_insert_with(Vec::new)
            .push(index);
            
        self.index_by_nonce
            .insert((tx.from.try_into().unwrap(), tx.nonce), index);
        
        // Store compact transaction
        self.transactions.push(compact_tx);
        
        // Cache full transaction
        self.full_tx_cache.put(tx.tx_hash(), tx);
        
        Ok(())
    }
    
    pub fn get_transaction(&self, tx_hash: &TxHash) -> Option<&SignedTx> {
        self.full_tx_cache.get(tx_hash)
    }
}
```

## Security Considerations

### Replay Attack Prevention
```rust
pub struct ReplayProtection {
    pub executed_txs: LRUCache<TxHash, u64>,     // tx_hash -> block height
    pub account_nonces: HashMap<[u8; 32], u64>,  // address -> last nonce
    pub window_size: u64,                        // Protection window
}

impl ReplayProtection {
    pub fn is_replay(&self, tx: &SignedTx, current_height: u64) -> bool {
        let tx_hash = tx.tx_hash();
        
        // Check if transaction was already executed
        if let Some(executed_height) = self.executed_txs.get(&tx_hash) {
            if current_height - executed_height < self.window_size {
                return true;
            }
        }
        
        // Check nonce sequence
        if let Some(last_nonce) = self.account_nonces.get(&tx.from.try_into().unwrap()) {
            if tx.nonce <= *last_nonce {
                return true;
            }
        }
        
        false
    }
    
    pub fn mark_executed(&mut self, tx: &SignedTx, block_height: u64) {
        let tx_hash = tx.tx_hash();
        let sender = tx.from.try_into().unwrap();
        
        // Mark as executed
        self.executed_txs.put(tx_hash, block_height);
        
        // Update account nonce
        let last_nonce = self.account_nonces.entry(sender).or_insert(0);
        if tx.nonce > *last_nonce {
            *last_nonce = tx.nonce;
        }
    }
}
```

### Double Spend Protection
```rust
pub struct DoubleSpendProtection {
    pub pending_spends: HashMap<[u8; 32], u128>, // address -> pending amount
    pub max_pending_per_address: u128,           // Maximum pending amount
}

impl DoubleSpendProtection {
    pub fn check_double_spend(&self, tx: &SignedTx) -> Result<(), ValidationError> {
        let sender = tx.from.try_into().unwrap();
        let pending_amount = self.pending_spends.get(&sender).unwrap_or(&0);
        
        if *pending_amount + tx.amount > self.max_pending_per_address {
            return Err(ValidationError::ExcessivePendingSpending);
        }
        
        Ok(())
    }
    
    pub fn add_pending(&mut self, tx: &SignedTx) {
        let sender = tx.from.try_into().unwrap();
        let pending = self.pending_spends.entry(sender).or_insert(0);
        *pending += tx.amount;
    }
    
    pub fn remove_pending(&mut self, tx: &SignedTx) {
        let sender = tx.from.try_into().unwrap();
        if let Some(pending) = self.pending_spends.get_mut(&sender) {
            *pending = pending.saturating_sub(tx.amount);
        }
    }
}
```

## Configuration Parameters

### Transaction Limits
```rust
pub struct TransactionLimits {
    pub max_size: usize,                        // Maximum transaction size (1MB)
    pub max_gas_limit: u64,                      // Maximum gas limit
    pub min_fee: u128,                           // Minimum fee
    pub max_fee: u128,                           // Maximum fee
    pub max_pending_per_address: u128,           // Max pending per address
    pub replay_protection_window: u64,           // Replay protection window
}

impl Default for TransactionLimits {
    fn default() -> Self {
        Self {
            max_size: 1024 * 1024,               // 1MB
            max_gas_limit: 10_000_000,           // 10M gas
            min_fee: 100_000_000_000_000,       // 0.0001 token
            max_fee: 1_000_000_000_000_000_000, // 1.0 token
            max_pending_per_address: 1_000_000_000_000_000_000, // 1M tokens
            replay_protection_window: 1000,      // 1000 blocks
        }
    }
}
```

## Performance Metrics

### Target Performance
- **Validation**: < 1ms per transaction
- **Classification**: < 0.1ms per transaction
- **SIMD Processing**: 4x speedup for batches > 32
- **Memory Usage**: < 100MB for 100K transactions
- **Cache Hit Rate**: > 95% for recent transactions

### Monitoring Metrics
```rust
pub struct TransactionMetrics {
    pub transactions_validated: u64,             // Total validated
    pub transactions_rejected: u64,              // Total rejected
    pub avg_validation_time: Duration,          // Average validation time
    pub cache_hit_rate: f64,                     // Cache hit rate
    pub simd_utilization: f64,                   // SIMD utilization rate
    pub replay_attempts_blocked: u64,            // Replay attempts blocked
    pub double_spends_prevented: u64,            // Double spends prevented
    pub total_gas_used: u64,                     // Total gas used
    pub avg_gas_price: u128,                     // Average gas price
}
```

This transaction system provides a robust, efficient, and secure foundation for blockchain operations while maintaining compatibility with existing infrastructure and enabling future extensibility.
            (StateAccessType::Read, StateAccessType::Read) => false,
        }
    }
    
    fn validate_conditional_dependencies(&self) -> Result<(), TransactionError> {
        // Validate conditional transaction dependencies
        // This is a simplified implementation
        Ok(())
    }
}
```

## Transaction Processing

### Transaction Validation Pipeline
```rust
pub struct TransactionValidator {
    pub syntactic_validator: SyntacticValidator,   // Syntactic validation
    pub semantic_validator: SemanticValidator,     // Semantic validation
    pub economic_validator: EconomicValidator,      // Economic validation
    pub security_validator: SecurityValidator,     // Security validation
    pub performance_validator: PerformanceValidator, // Performance validation
}

impl TransactionValidator {
    pub fn validate_transaction(&self, tx: &SignedTransaction, state: &StateDB) -> Result<ValidationResult, ValidationError> {
        // 1. Syntactic validation
        let syntactic_result = self.syntactic_validator.validate(tx)?;
        
        // 2. Semantic validation
        let semantic_result = self.semantic_validator.validate(tx, state)?;
        
        // 3. Economic validation
        let economic_result = self.economic_validator.validate(tx, state)?;
        
        // 4. Security validation
        let security_result = self.security_validator.validate(tx, state)?;
        
        // 5. Combine results
        let combined_result = ValidationResult {
            is_valid: syntactic_result.is_valid && 
                       semantic_result.is_valid && 
                       economic_result.is_valid && 
                       security_result.is_valid,
            syntactic_issues: syntactic_result.issues,
            semantic_issues: semantic_result.issues,
            economic_issues: economic_result.issues,
            security_issues: security_result.issues,
            gas_estimate: syntactic_result.gas_estimate,
            priority_score: self.calculate_priority_score(&syntactic_result, &semantic_result, &economic_result),
        };
        
        Ok(combined_result)
    }
    
    fn calculate_priority_score(&self, syntactic: &SyntacticResult, semantic: &SemanticResult, economic: &EconomicResult) -> f64 {
        let syntactic_score = syntactic.complexity_score;
        let semantic_score = semantic.value_score;
        let economic_score = economic.economic_score;
        
        (syntactic_score * 0.2) + (semantic_score * 0.3) + (economic_score * 0.5)
    }
}
```

### Transaction Execution Engine
```rust
pub struct TransactionExecutor {
    pub runtime: ContractRuntime,              // Contract runtime
    pub state_manager: StateManager,            // State management
    pub gas_meter: GasMeter,                    // Gas metering
    pub event_emitter: EventEmitter,            // Event emission
}

impl TransactionExecutor {
    pub fn execute_transaction(&mut self, tx: &SignedTransaction, state: &mut StateDB) -> Result<ExecutionResult, ExecutionError> {
        // 1. Parse transaction type
        let tx_type = self.parse_transaction_type(&tx.transaction)?;
        
        // 2. Execute based on type
        let result = match tx_type {
            TransactionType::Transfer => self.execute_transfer(tx, state)?,
            TransactionType::ContractCall => self.execute_contract_call(tx, state)?,
            TransactionType::ContractDeployment => self.execute_contract_deployment(tx, state)?,
            TransactionType::Governance => self.execute_governance(tx, state)?,
            TransactionType::Batch => self.execute_batch(tx, state)?,
        };
        
        // 3. Update state
        self.state_manager.apply_state_changes(result.state_changes.clone())?;
        
        // 4. Emit events
        for event in result.events {
            self.event_emitter.emit_event(event)?;
        }
        
        Ok(result)
    }
    
    fn parse_transaction_type(&self, tx: &BaseTransaction) -> Result<TransactionType, ParseError> {
        if tx.data.is_empty() {
            return Ok(TransactionType::Transfer);
        }
        
        let selector = u32::from_le_bytes([
            tx.data[0], tx.data[1], tx.data[2], tx.data[3]
        ]);
        
        match selector {
            0x00000001..=0x0000FFFF => Ok(TransactionType::Transfer),
            0x10000001..=0x1000FFFF => Ok(TransactionType::ContractCall),
            0x20000001..=0x2000FFFF => Ok(TransactionType::ContractDeployment),
            0x30000001..=0x3000FFFF => Ok(TransactionType::Governance),
            0x40000001..=0x4000FFFF => Ok(TransactionType::Batch),
            _ => Err(ParseError::UnknownTransactionType),
        }
    }
}
```

This transaction types specification provides a comprehensive foundation for blockchain operations with optimized encoding, type-specific validation, and efficient execution paths.
