# Cryptography Specification

## Cryptographic Primitives

### Hash Functions

**Blake3**
- Purpose: General hashing, transaction hashes, state roots
- Output: 256 bits (32 bytes)
- Implementation: `blake3` crate v1.3
- Properties: High performance, parallelizable, secure

**SHA512**
- Purpose: Block hashing with domain separation
- Output: 512 bits (64 bytes)
- Implementation: `sha2` crate v0.10
- Properties: Widely analyzed, collision resistant

**Keccak256**
- Purpose: Function selector computation, contract hashing
- Output: 256 bits (32 bytes)
- Implementation: `sha3` crate v0.10
- Properties: Ethereum compatibility

### Digital Signatures

**Ed25519**
- Purpose: Transaction signing, validator signatures
- Key Size: 32 bytes private, 32 bytes public
- Signature Size: 64 bytes
- Implementation: `ed25519-dalek` crate v2.0
- Features: Batch verification, deterministic signatures

**Curve25519**
- Purpose: Diffie-Hellman key exchange
- Implementation: `curve25519-dalek` crate v4.0
- Features: Montgomery curve, constant-time operations

## Hash Computation

### Block Hash
```rust
pub fn compute_block_hash(header: &BlockHeader) -> Result<Hash64> {
    let bytes = canonical_serialize(header)?;
    let mut hasher = Sha512::new();
    hasher.update(b"BLK");  // Domain separator
    hasher.update(&bytes);
    Ok(Hash64::new(hasher.finalize().into()))
}
```

**Formula:** $H_{block} = SHA512("BLK" || serialized_{header})$

**Domain Separation:** Prevents hash collision across contexts

### Transaction Hash
```rust
pub fn compute_tx_hash(tx: &SignedTx) -> Hash32 {
    let bytes = canonical_serialize(tx);
    let hash = blake3::hash(&bytes);
    Hash32::new(hash.as_bytes().try_into().unwrap())
}
```

**Formula:** $H_{tx} = Blake3(serialized_{transaction})$

### State Root
```rust
pub fn compute_state_root(accounts: &BTreeMap<[u8; 32], Account>) -> Hash64 {
    let mut hasher = Blake3::new();
    for (address, account) in accounts {
        hasher.update(address);
        hasher.update(&canonical_serialize(account));
    }
    Hash64::new(hasher.finalize().into())
}
```

**Formula:** $H_{state} = Blake3(\sum_{accounts} address || account)$

## Signature Operations

### Transaction Signing
```rust
pub fn sign_transaction(
    tx: &UnsignedTx,
    private_key: &Ed25519PrivateKey,
) -> Result<[u8; 64]> {
    let message = canonical_serialize(tx);
    let signature = private_key.sign(&message);
    Ok(signature.to_bytes())
}
```

**Message:** Serialized unsigned transaction
**Signature:** Ed25519 signature (64 bytes)

### Signature Verification
```rust
pub fn verify_signature(
    signature: &[u8; 64],
    public_key: &[u8; 32],
    message: &[u8],
) -> bool {
    let pubkey = Ed25519PublicKey::from_bytes(public_key);
    let sig = Ed25519Signature::from_bytes(signature);
    pubkey.verify(message, &sig).is_ok()
}
```

**Batch Verification:**
```rust
pub fn batch_verify(
    messages: &[&[u8]],
    signatures: &[[u8; 64]],
    public_keys: &[[u8; 32]],
) -> bool {
    let batch: Vec<_> = messages.iter()
        .zip(signatures.iter())
        .zip(public_keys.iter())
        .map(|((m, s), pk)| (m, s, pk))
        .collect();
    
    Ed25519PublicKey::batch_verify(&batch).is_ok()
}
```

**Performance:** ~10x faster than individual verification

## Zero-Knowledge Proofs

### PLONK Proof System
- Purpose: Monolith verification, state consistency proofs
- Implementation: `zkp-plonk` crate
- Trusted Setup: Required for universal SRS

### Monolith Proof Generation
```rust
pub fn generate_monolith_proof(
    headers: &[BlockHeader],
    state_root: [u8; 64],
    epoch_id: u64,
) -> Result<Vec<u8>> {
    let statement = Statement::new(
        compress_root_64_to_32(&parent_root),
        compress_root_64_to_32(&headers_commit),
        compress_root_64_to_32(&state_commit),
        exec_height,
        epoch_id,
    );
    
    let witness = create_witness(headers, state_root)?;
    let proof = prove(&statement, &witness)?;
    Ok(proof)
}
```

### Monolith Proof Verification
```rust
pub fn verify_monolith_proof<V: ZkVerifier>(
    header: &MonolithHeader,
    prev_state_root: Option<[u8; 64]>,
    prev_epoch_id: Option<u64>,
    verifier: &V,
) -> anyhow::Result<bool> {
    // Epoch regression protection
    if let Some(prev_epoch) = prev_epoch_id {
        anyhow::ensure!(
            header.epoch_id >= prev_epoch,
            "monolith epoch regression detected (replay attempt)"
        );
    }
    
    let statement = Statement::new(
        compress_root_64_to_32(&parent_root),
        compress_root_64_to_32(&header.headers_commit),
        compress_root_64_to_32(&header.state_commit),
        header.exec_height,
        header.epoch_id,
    );
    
    let ok = verifier.verify(&statement, &header.proof)?;
    anyhow::ensure!(ok, "monolith zkp proof invalid");
    Ok(true)
}
```

**Verification Complexity:** O(1) regardless of monolith size

## Key Derivation

### Address Derivation
```rust
pub fn derive_address(public_key: &[u8; 32]) -> [u8; 32] {
    let hash = blake3::hash(public_key);
    let mut address = [0u8; 32];
    address.copy_from_slice(hash.as_bytes());
    address
}
```

**Formula:** $address = Blake3(public_{key})$

### Contract Address
```rust
pub fn derive_contract_address(
    deployer: &[u8; 32],
    nonce: u64,
    salt: &[u8; 32],
) -> [u8; 32] {
    let mut hasher = Blake3::new();
    hasher.update(b"CONTRACT");
    hasher.update(deployer);
    hasher.update(&nonce.to_le_bytes());
    hasher.update(salt);
    let hash = hasher.finalize();
    let mut address = [0u8; 32];
    address.copy_from_slice(hash.as_bytes());
    address
}
```

**Formula:** $address_{contract} = Blake3("CONTRACT" || deployer || nonce || salt)$

## Cryptographic Security Parameters

### Security Levels
**Ed25519:** 128-bit security level
**Blake3:** 256-bit security level  
**SHA512:** 256-bit security level
**PLONK:** 128-bit security level (depending on curve)

### Key Sizes
- Private Key (Ed25519): 32 bytes
- Public Key (Ed25519): 32 bytes
- Signature (Ed25519): 64 bytes
- Address: 32 bytes
- Hash (Blake3): 32 bytes
- Hash (SHA512): 64 bytes

### Random Number Generation
```rust
use rand::rngs::OsRng;
use rand::RngCore;

pub fn generate_private_key() -> [u8; 32] {
    let mut rng = OsRng;
    let mut key = [0u8; 32];
    rng.fill_bytes(&mut key);
    key
}
```

**Source:** Operating system entropy pool
**Quality:** Cryptographically secure

## Performance Optimizations

### SIMD Cryptographic Operations
```rust
#[cfg(target_arch = "x86_64")]
pub fn simd_hash_batch(data: &[&[u8]]) -> Vec<[u8; 32]> {
    if is_x86_feature_detected!("avx2") {
        // AVX2 optimized batch hashing
        avx2_hash_batch(data)
    } else {
        // Fallback to scalar
        scalar_hash_batch(data)
    }
}
```

**Performance:** 2-4x speedup for batch operations

### Parallel Verification
```rust
pub fn parallel_verify_signatures(
    messages: &[&[u8]],
    signatures: &[[u8; 64]],
    public_keys: &[[u8; 32]],
) -> Vec<bool> {
    use rayon::prelude::*;
    
    messages.par_iter()
        .zip(signatures.par_iter())
        .zip(public_keys.par_iter())
        .map(|((msg, sig), pk)| verify_signature(sig, pk, msg))
        .collect()
}
```

**Scaling:** Linear with CPU core count

## Cryptographic Protocols

### Key Exchange (ECDH)
```rust
pub fn ecdh_key_exchange(
    private_key: &Curve25519PrivateKey,
    public_key: &Curve25519PublicKey,
) -> [u8; 32] {
    let shared_secret = private_key.diffie_hellman(public_key);
    let mut secret = [0u8; 32];
    secret.copy_from_slice(shared_secret.as_bytes());
    secret
}
```

**Security:** Elliptic Curve Diffie-Hellman on Curve25519

### Message Authentication
```rust
pub fn compute_mac(key: &[u8; 32], message: &[u8]) -> [u8; 32] {
    let mut mac = HmacSha256::new(key);
    mac.update(message);
    let result = mac.finalize();
    let mut output = [0u8; 32];
    output.copy_from_slice(result.into_bytes().as_slice());
    output
}
```

**Algorithm:** HMAC-SHA256
**Purpose:** Message integrity and authentication

## Security Considerations

### Side-Channel Protection
- Constant-time operations for secret data
- No secret-dependent branching
- Memory sanitization for sensitive data

### Key Management
```rust
pub struct SecureKey {
    key: [u8; 32],
}

impl Drop for SecureKey {
    fn drop(&mut self) {
        // Zeroize memory on drop
        self.key.iter_mut().for_each(|b| *b = 0);
    }
}
```

### Randomness Validation
```rust
pub fn validate_entropy(data: &[u8]) -> bool {
    // Basic entropy validation
    if data.len() < 32 {
        return false;
    }
    
    // Count unique bytes
    let unique_bytes = data.iter().collect::<std::collections::HashSet<_>>().len();
    unique_bytes > data.len() / 2
}
```

## Cryptographic Constants

### Domain Separators
- "BLK": Block hash computation
- "TX": Transaction hash computation
- "CONTRACT": Contract address derivation
- "MONOLITH": Monolith proof computation

### Curve Parameters
**Curve25519:**
- Prime: $2^{255} - 19$
- Base Point: Standard generator
- Order: $2^{252} + 27742317777372353535851937790883648493$

**Ed25519:**
- Curve: Twisted Edwards form of Curve25519
- Cofactor: 8
- Hash to Curve: Elligator2

## Implementation Requirements

### Deterministic Behavior
All cryptographic operations must be deterministic across platforms:
- Same inputs → Same outputs
- No platform-dependent behavior
- Fixed precision for floating-point operations

### Performance Targets
- Hash computation: < 1μs per 1KB
- Signature verification: < 10μs
- Batch verification: < 1μs per signature
- ZKP verification: < 100ms

### Memory Safety
- No buffer overflows
- Proper bounds checking
- Secure memory allocation for secrets
- Constant-time operations where required

This cryptography specification provides the foundation for all security-critical operations in Savitri Network.
