# Light Node Security Model

## Overview

The Savitri Network light node security model is designed to provide cryptographic verification and trust without storing the full blockchain state. Light nodes achieve security through selective verification, trust assumptions, and cryptographic proofs while maintaining minimal resource requirements.

## Technology Choice Rationale

### Why Light Node Security Model

**Problem Statement**: Light nodes need to verify blockchain data and transactions without access to the full blockchain state, while maintaining security guarantees against malicious actors and network attacks.

**Chosen Solution**: Cryptographic verification architecture with trust assumptions, state proofs, and selective data verification.

**Rationale**:
- **Cryptographic Verification**: All data verified through cryptographic proofs
- **Trust Minimization**: Minimal trust assumptions with peer verification
- **State Proofs**: Merkle proofs for state verification without full state
- **Attack Resistance**: Protection against common blockchain attacks
- **Resource Efficiency**: Security without full node requirements

**Expected Results**:
- Cryptographic verification of all received data
- Protection against invalid blocks and transactions
- Detection of malicious peers and network attacks
- Secure transaction verification without full state
- Trust establishment through cryptographic proofs

## Trust Model

### Trust Assumptions

**Trust Boundaries**:
```rust
pub struct TrustModel {
    pub trusted_peers: Vec<PeerId>,           // Trusted peer list
    pub minimum_trust_score: f64,             // Minimum trust score
    pub trust_decay_rate: f64,                 // Trust decay rate
    pub attack_threshold: f64,                 // Attack detection threshold
}

impl TrustModel {
    pub fn evaluate_peer_trust(&self, peer: &PeerId, behavior: &PeerBehavior) -> TrustScore {
        let mut trust_score = self.get_initial_trust_score(peer);
        
        // 1. Update based on behavior
        trust_score += behavior.correct_data_provided * 0.3;
        trust_score += behavior.timely_responses * 0.2;
        trust_score += behavior.consensus_participation * 0.2;
        trust_score -= behavior.incorrect_data_provided * 0.4;
        trust_score -= behavior.late_responses * 0.2;
        trust_score -= behavior.malicious_activity * 0.8;
        
        // 2. Apply trust decay
        let time_since_last_interaction = behavior.last_interaction.elapsed();
        let decay_factor = (time_since_last_interaction.as_secs() as f64 / 86400.0) * self.trust_decay_rate;
        trust_score *= (1.0 - decay_factor).max(0.0);
        
        // 3. Clamp to valid range
        trust_score = trust_score.clamp(0.0, 1.0);
        
        TrustScore {
            score: trust_score,
            confidence: self.calculate_trust_confidence(peer, behavior),
            last_updated: Instant::now(),
        }
    }
    
    pub fn is_peer_trusted(&self, peer: &PeerId, trust_score: &TrustScore) -> bool {
        // 1. Check if peer is in trusted list
        if self.trusted_peers.contains(peer) {
            return true;
        }
        
        // 2. Check trust score threshold
        if trust_score.score < self.minimum_trust_score {
            return false;
        }
        
        // 3. Check confidence level
        if trust_score.confidence < 0.7 {
            return false;
        }
        
        true
    }
}
```

### Trust Establishment

**Peer Trust Establishment**:
```rust
pub struct TrustEstablishment {
    pub initial_trust_score: f64,            // Initial trust score
    pub trust_building_period: Duration,      // Trust building period
    pub verification_requirements: VerificationRequirements, // Verification requirements
}

impl TrustEstablishment {
    pub async fn establish_trust(&self, peer: &PeerId) -> Result<TrustResult, TrustError> {
        let mut trust_score = self.initial_trust_score;
        let mut verification_results = Vec::new();
        
        // 1. Verify peer identity
        let identity_verification = self.verify_peer_identity(peer).await?;
        verification_results.push(identity_verification);
        
        if identity_verification.is_valid {
            trust_score += 0.2;
        }
        
        // 2. Verify peer capabilities
        let capability_verification = self.verify_peer_capabilities(peer).await?;
        verification_results.push(capability_verification);
        
        if capability_verification.is_valid {
            trust_score += 0.1;
        }
        
        // 3. Test peer responses
        let response_verification = self.test_peer_responses(peer).await?;
        verification_results.push(response_verification);
        
        if response_verification.is_valid {
            trust_score += 0.3;
        }
        
        // 4. Verify consensus participation
        let consensus_verification = self.verify_consensus_participation(peer).await?;
        verification_results.push(consensus_verification);
        
        if consensus_verification.is_valid {
            trust_score += 0.4;
        }
        
        // 5. Establish trust if threshold met
        if trust_score >= 0.7 {
            Ok(TrustResult::Established {
                trust_score,
                verification_results,
                establishment_time: Instant::now(),
            })
        } else {
            Ok(TrustResult::Insufficient {
                trust_score,
                verification_results,
                reason: "Trust score below threshold".to_string(),
            })
        }
    }
}
```

## Cryptographic Security

### Signature Verification

**Signature Verification Pipeline**:
```rust
pub struct SignatureVerifier {
    pub validator_set: Arc<ValidatorSet>,      // Current validator set
    pub cryptographic_engine: Arc<CryptoEngine>, // Cryptographic engine
    pub signature_cache: LruCache<SignatureHash, VerificationResult>, // Signature cache
}

impl SignatureVerifier {
    pub fn verify_block_signature(&self, header: &BlockHeader) -> Result<SignatureVerification, SignatureError> {
        // 1. Check cache first
        let signature_hash = self.calculate_signature_hash(header);
        if let Some(cached_result) = self.signature_cache.get(&signature_hash) {
            return Ok(cached_result.clone());
        }
        
        // 2. Get validator public key
        let validator_public_key = self.validator_set.get_validator_public_key(&header.proposer)?;
        
        // 3. Verify signature
        let signature_valid = self.cryptographic_engine.verify_signature(
            &header.proposer_signature,
            &header.hash,
            &validator_public_key,
        )?;
        
        // 4. Verify signature format
        let format_valid = self.verify_signature_format(&header.proposer_signature)?;
        
        // 5. Verify signature timestamp
        let timestamp_valid = self.verify_signature_timestamp(header)?;
        
        let verification_result = SignatureVerification {
            is_valid: signature_valid && format_valid && timestamp_valid,
            validator: header.proposer,
            verification_time: Instant::now(),
            details: vec![
                format!("Signature valid: {}", signature_valid),
                format!("Format valid: {}", format_valid),
                format!("Timestamp valid: {}", timestamp_valid),
            ],
        };
        
        // 6. Cache result
        self.signature_cache.put(signature_hash, verification_result.clone());
        
        Ok(verification_result)
    }
}
```

### Hash Verification

**Hash Verification Architecture**:
```rust
pub struct HashVerifier {
    pub hash_algorithms: Vec<HashAlgorithm>, // Supported hash algorithms
    pub hash_cache: LruCache<HashInput, HashOutput>, // Hash cache
}

impl HashVerifier {
    pub fn verify_merkle_root(&self, transactions: &[Transaction], expected_root: &MerkleRoot) -> Result<HashVerification, HashError> {
        // 1. Calculate transaction hashes
        let mut tx_hashes = Vec::new();
        for tx in transactions {
            let tx_hash = self.calculate_transaction_hash(tx)?;
            tx_hashes.push(tx_hash);
        }
        
        // 2. Build Merkle tree
        let merkle_tree = self.build_merkle_tree(&tx_hashes)?;
        
        // 3. Calculate root hash
        let calculated_root = merkle_tree.root_hash();
        
        // 4. Verify against expected root
        let root_valid = calculated_root == *expected_root;
        
        Ok(HashVerification {
            is_valid: root_valid,
            calculated_root,
            expected_root: *expected_root,
            tree_depth: merkle_tree.depth(),
            verification_time: Instant::now(),
        })
    }
    
    pub fn verify_state_root(&self, state_changes: &[StateChange], expected_root: &StateRoot) -> Result<HashVerification, HashError> {
        // 1. Calculate state change hashes
        let mut state_hashes = Vec::new();
        for change in state_changes {
            let state_hash = self.calculate_state_hash(change)?;
            state_hashes.push(state_hash);
        }
        
        // 2. Build state trie
        let state_trie = self.build_state_trie(&state_hashes)?;
        
        // 3. Calculate root hash
        let calculated_root = state_trie.root_hash();
        
        // 4. Verify against expected root
        let root_valid = calculated_root == *expected_root;
        
        Ok(HashVerification {
            is_valid: root_valid,
            calculated_root,
            expected_root: *expected_root,
            trie_depth: state_trie.depth(),
            verification_time: Instant::now(),
        })
    }
}
```

## Attack Detection

### Attack Detection System

**Attack Detection Architecture**:
```rust
pub struct AttackDetector {
    pub attack_patterns: Vec<AttackPattern>, // Attack patterns
    pub detection_threshold: f64,               // Detection threshold
    pub alert_manager: Arc<AlertManager>,     // Alert manager
    pub attack_history: VecDeque<AttackEvent>,  // Attack history
}

impl AttackDetector {
    pub fn detect_attacks(&self, network_activity: &NetworkActivity) -> Result<Vec<AttackDetection>, AttackError> {
        let mut detected_attacks = Vec::new();
        
        // 1. Check for eclipse attacks
        if let Some(eclipse_attack) = self.detect_eclipse_attack(network_activity)? {
            detected_attacks.push(eclipse_attack);
        }
        
        // 2. Check for sybil attacks
        if let Some(sybil_attack) = self.detect_sybil_attack(network_activity)? {
            detected_attacks.push(sybil_attack);
        }
        
        // 3. Check for replay attacks
        if let Some(replay_attack) = self.detect_replay_attack(network_activity)? {
            detected_attacks.push(replay_attack);
        }
        
        // 4. Check for data manipulation attacks
        if let Some(manipulation_attack) = self.detect_data_manipulation(network_activity)? {
            detected_attacks.push(manipulation_attack);
        }
        
        // 5. Check for timing attacks
        if let Some(timing_attack) = self.detect_timing_attack(network_activity)? {
            detected_attacks.push(timing_attack);
        }
        
        // 6. Update attack history
        for attack in &detected_attacks {
            self.update_attack_history(attack.clone());
        }
        
        Ok(detected_attacks)
    }
    
    fn detect_eclipse_attack(&self, activity: &NetworkActivity) -> Result<Option<AttackDetection>, AttackError> {
        // 1. Check peer concentration
        let peer_concentration = self.calculate_peer_concentration(activity)?;
        
        // 2. Check message source diversity
        let source_diversity = self.calculate_source_diversity(activity)?;
        
        // 3. Check network partition indicators
        let partition_indicators = self.detect_network_partition(activity)?;
        
        // 4. Calculate eclipse attack probability
        let eclipse_probability = (peer_concentration * 0.4) + 
                               (1.0 - source_diversity) * 0.3 + 
                               partition_indicators * 0.3;
        
        if eclipse_probability > self.detection_threshold {
            Ok(Some(AttackDetection {
                attack_type: AttackType::Eclipse,
                confidence: eclipse_probability,
                evidence: vec![
                    format!("Peer concentration: {:.2}", peer_concentration),
                    format!("Source diversity: {:.2}", source_diversity),
                    format!("Partition indicators: {:.2}", partition_indicators),
                ],
                detection_time: Instant::now(),
            }))
        } else {
            Ok(None)
        }
    }
}
```

### Specific Attack Types

**Eclipse Attack Detection**:
```rust
pub struct EclipseAttackDetector {
    pub max_peer_concentration: f64,            // Maximum peer concentration
    pub min_source_diversity: f64,             // Minimum source diversity
    pub partition_tolerance: f64,               // Partition tolerance
}

impl EclipseAttackDetector {
    pub fn analyze_peer_connections(&self, connections: &[PeerConnection]) -> EclipseAnalysis {
        let mut analysis = EclipseAnalysis::new();
        
        // 1. Analyze peer distribution
        let peer_distribution = self.analyze_peer_distribution(connections);
        analysis.peer_concentration = peer_distribution.concentration_score;
        
        // 2. Analyze message sources
        let message_sources = self.analyze_message_sources(connections);
        analysis.source_diversity = message_sources.diversity_score;
        
        // 3. Analyze network topology
        let network_topology = self.analyze_network_topology(connections);
        analysis.topology_anomaly = network_topology.anomaly_score;
        
        // 4. Calculate eclipse attack probability
        analysis.eclipse_probability = self.calculate_eclipse_probability(&analysis);
        
        // 5. Determine if eclipse attack is likely
        analysis.is_eclipse_attack = analysis.eclipse_probability > 0.7;
        
        analysis
    }
}
```

**Replay Attack Detection**:
```rust
pub struct ReplayAttackDetector {
    pub transaction_cache: LruCache<TxHash, Transaction>, // Transaction cache
    pub block_cache: LruCache<BlockHash, Block>,     // Block cache
    pub replay_window: Duration,                    // Replay window
}

impl ReplayAttackDetector {
    pub fn detect_replay_attack(&self, transaction: &SignedTransaction) -> Result<Option<ReplayAttack>, ReplayError> {
        // 1. Check if transaction was recently processed
        if let Some(cached_tx) = self.transaction_cache.get(&transaction.hash) {
            let time_diff = cached_tx.processed_at.elapsed();
            if time_diff < self.replay_window {
                return Ok(Some(ReplayAttack {
                    original_transaction: cached_tx,
                    replay_transaction: transaction.clone(),
                    replay_time: Instant::now(),
                    confidence: 0.9,
                }));
            }
        }
        
        // 2. Check nonce sequence
        if let Some(last_nonce) = self.get_last_nonce_for_address(&transaction.signer) {
            if transaction.nonce <= last_nonce {
                return Ok(Some(ReplayAttack {
                    original_transaction: None,
                    replay_transaction: transaction.clone(),
                    replay_time: Instant::now(),
                    confidence: 0.8,
                }));
            }
        }
        
        // 3. Check signature reuse
        if self.is_signature_reused(transaction)? {
            return Ok(Some(ReplayAttack {
                original_transaction: None,
                replay_transaction: transaction.clone(),
                replay_time: Instant::now(),
                confidence: 0.7,
            }));
        }
        
        Ok(None)
    }
}
```

## State Proof Security

### State Proof Verification

**State Proof Architecture**:
```rust
pub struct StateProofSecurity {
    pub proof_verifier: Arc<ProofVerifier>,   // Proof verifier
    pub state_tracker: Arc<StateTracker),     // State tracker
    pub proof_cache: LruCache<ProofHash, ProofVerification>, // Proof cache
}

impl StateProofSecurity {
    pub fn verify_state_proof(&self, proof: &StateProof, state_root: &StateRoot) -> Result<StateProofVerification, StateProofError> {
        // 1. Verify proof structure
        self.verify_proof_structure(proof)?;
        
        // 2. Verify Merkle path
        let merkle_verification = self.proof_verifier.verify_merkle_path(
            &proof.key,
            &proof.value,
            &proof.proof,
            state_root,
        )?;
        
        if !merkle_verification.is_valid {
            return Err(StateProofError::InvalidMerklePath);
        }
        
        // 3. Verify state transition
        let transition_verification = self.verify_state_transition(proof)?;
        
        if !transition_verification.is_valid {
            return Err(StateProofError::InvalidStateTransition);
        }
        
        // 4. Verify timestamp
        let timestamp_verification = self.verify_timestamp(proof)?;
        
        if !timestamp_verification.is_valid {
            return Err(StateProofError::InvalidTimestamp);
        }
        
        // 5. Cache verification result
        let verification_result = StateProofVerification {
            is_valid: true,
            merkle_verification,
            transition_verification,
            timestamp_verification,
            verification_time: Instant::now(),
        };
        
        self.proof_cache.put(proof.hash, verification_result.clone());
        
        Ok(verification_result)
    }
    
    fn verify_state_transition(&self, proof: &StateProof) -> Result<StateTransitionVerification, StateProofError> {
        // 1. Get previous state
        let previous_state = self.state_tracker.get_state(&proof.key);
        
        // 2. Verify transition rules
        let transition_valid = match (&previous_state, &proof.value) {
            (None, _) => true, // New state
            (Some(prev), Some(curr)) => self.verify_transition_rules(prev, curr)?,
            (Some(_), None) => false, // State deletion not allowed
            (None, None) => false, // No state change
        };
        
        Ok(StateTransitionVerification {
            is_valid: transition_valid,
            previous_state,
            new_state: proof.value.clone(),
            transition_rules: self.get_applicable_rules(&proof.key),
        })
    }
}
```

### Proof Generation Security

**Secure Proof Generation**:
```rust
pub struct ProofGenerator {
    pub cryptographic_engine: Arc<CryptoEngine>, // Cryptographic engine
    pub state_tracker: Arc<StateTracker>,     // State tracker
    pub proof_cache: LruCache<ProofInput, Proof>, // Proof cache
}

impl ProofGenerator {
    pub fn generate_state_proof(&self, key: &StateKey, state_root: &StateRoot) -> Result<StateProof, ProofError> {
        // 1. Get current state
        let current_state = self.state_tracker.get_state(key);
        
        // 2. Generate Merkle proof
        let merkle_proof = self.generate_merkle_proof(key, &current_state, state_root)?;
        
        // 3. Generate transition proof
        let transition_proof = self.generate_transition_proof(key, &current_state)?;
        
        // 4. Generate timestamp proof
        let timestamp_proof = self.generate_timestamp_proof()?;
        
        // 5. Assemble complete proof
        let proof = StateProof {
            key: key.clone(),
            value: current_state,
            proof: merkle_proof,
            transition_proof,
            timestamp_proof,
            hash: self.calculate_proof_hash(key, &current_state),
        };
        
        Ok(proof)
    }
    
    fn generate_merkle_proof(&self, key: &StateKey, value: &StateValue, state_root: &StateRoot) -> Result<MerkleProof, ProofError> {
        // 1. Create leaf node
        let leaf_hash = self.calculate_leaf_hash(key, value)?;
        
        // 2. Build Merkle path to root
        let merkle_path = self.build_merkle_path(&leaf_hash, state_root)?;
        
        // 3. Verify path integrity
        self.verify_merkle_path_integrity(&leaf_hash, &merkle_path, state_root)?;
        
        Ok(MerkleProof {
            leaf_hash,
            path: merkle_path,
            root_hash: *state_root,
        })
    }
}
```

## Network Security

### Secure Communication

**Secure Communication Architecture**:
```rust
pub struct SecureCommunication {
    pub encryption_engine: Arc<EncryptionEngine>, // Encryption engine
    pub authentication_manager: Arc<AuthenticationManager>, // Authentication manager
    pub session_manager: Arc<SessionManager>,   // Session manager
}

impl SecureCommunication {
    pub fn establish_secure_channel(&self, peer: &PeerId) -> Result<SecureChannel, CommunicationError> {
        // 1. Perform mutual authentication
        let authentication_result = self.authentication_manager.mutual_authenticate(peer)?;
        
        if !authentication_result.is_successful {
            return Err(CommunicationError::AuthenticationFailed);
        }
        
        // 2. Generate session keys
        let session_keys = self.session_manager.generate_session_keys(peer)?;
        
        // 3. Establish encrypted channel
        let encrypted_channel = self.encryption_engine.establish_encrypted_channel(
            peer,
            &session_keys,
        )?;
        
        Ok(SecureChannel {
            peer: *peer,
            session_keys,
            encrypted_channel,
            established_at: Instant::now(),
        })
    }
    
    pub fn send_secure_message(&self, channel: &SecureChannel, message: &NetworkMessage) -> Result<(), CommunicationError> {
        // 1. Encrypt message
        let encrypted_message = self.encryption_engine.encrypt_message(message, &channel.session_keys)?;
        
        // 2. Add authentication tag
        let authenticated_message = self.add_authentication_tag(&encrypted_message, &channel.session_keys)?;
        
        // 3. Send through secure channel
        channel.encrypted_channel.send(authenticated_message).await?;
        
        Ok(())
    }
}
```

### Message Authentication

**Message Authentication**:
```rust
pub struct MessageAuthenticator {
    pub hmac_engine: Arc<HmacEngine>,         // HMAC engine
    pub nonce_manager: Arc<NonceManager>,     // Nonce manager
    pub replay_protection: Arc<ReplayProtection>, // Replay protection
}

impl MessageAuthenticator {
    pub fn authenticate_message(&self, message: &NetworkMessage, signature: &MessageSignature) -> Result<AuthenticationResult, AuthenticationError> {
        // 1. Verify nonce freshness
        if !self.nonce_manager.is_nonce_fresh(&signature.nonce)? {
            return Err(AuthenticationError::ReplayAttack);
        }
        
        // 2. Calculate HMAC
        let message_hash = self.calculate_message_hash(message);
        let expected_hmac = self.hmac_engine.calculate_hmac(&message_hash, &signature.nonce);
        
        // 3. Verify HMAC signature
        let hmac_valid = self.hmac_engine.verify_hmac(&expected_hmac, &signature.hmac_signature)?;
        
        // 4. Verify timestamp
        let timestamp_valid = self.verify_timestamp(&signature.timestamp)?;
        
        // 5. Update nonce manager
        self.nonce_manager.mark_nonce_used(&signature.nonce)?;
        
        Ok(AuthenticationResult {
            is_authenticated: hmac_valid && timestamp_valid,
            verification_time: Instant::now(),
            details: vec![
                format!("HMAC valid: {}", hmac_valid),
                format!("Timestamp valid: {}", timestamp_valid),
            ],
        })
    }
}
```

This light node security model provides comprehensive cryptographic security, attack detection, and trust management while maintaining the resource constraints required for mobile and embedded devices.
