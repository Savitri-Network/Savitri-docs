# Transaction Format Specification

## Overview

This document specifies the exact format of transactions in Savitri Network, including signed transactions, mempool format, and validation rules.

## Signed Transaction Format

### Complete Transaction Structure

```
SignedTx {
    from: [20 bytes]          // Sender address
    to: [20 bytes]            // Recipient address (0x0000... for contract creation)
    value: [32 bytes]          // Transfer amount (big-endian uint256)
    data: [variable]           // Transaction data (RLP encoded)
    nonce: [8 bytes]           // Account nonce (big-endian uint64)
    gas_limit: [8 bytes]       // Maximum gas (big-endian uint64)
    gas_price: [32 bytes]     // Gas price (big-endian uint256)
    chain_id: [8 bytes]       // Chain identifier (big-endian uint64)
    signature: [65 bytes]      // ECDSA signature (r, s, v)
}
```

### Field Descriptions

#### from
- **Type**: 20-byte address
- **Purpose**: Transaction sender
- **Derivation**: Derived from public key
- **Validation**: Must be valid address

#### to
- **Type**: 20-byte address
- **Purpose**: Transaction recipient
- **Special**: All zeros (0x0000...) for contract creation
- **Validation**: Must be valid address or zero

#### value
- **Type**: 32-byte unsigned integer (big-endian)
- **Purpose**: Amount to transfer
- **Range**: 0 to 2^256 - 1
- **Unit**: Smallest unit (wei equivalent)

#### data
- **Type**: Variable length bytes
- **Purpose**: Transaction data or contract code
- **Encoding**: RLP encoded
- **Max Size**: 128 KB

#### nonce
- **Type**: 8-byte unsigned integer (big-endian)
- **Purpose**: Prevents replay attacks
- **Range**: 0 to 2^64 - 1
- **Validation**: Must match account nonce

#### gas_limit
- **Type**: 8-byte unsigned integer (big-endian)
- **Purpose**: Maximum gas to consume
- **Range**: 21,000 to 2^64 - 1
- **Validation**: Must be >= minimum

#### gas_price
- **Type**: 32-byte unsigned integer (big-endian)
- **Purpose**: Price per unit of gas
- **Range**: 0 to 2^256 - 1
- **Unit**: Smallest unit per gas

#### chain_id
- **Type**: 8-byte unsigned integer (big-endian)
- **Purpose**: Prevents cross-chain replay
- **Value**: Network-specific identifier
- **Validation**: Must match network

#### signature
- **Type**: 65 bytes
- **Purpose**: ECDSA signature
- **Format**: r (32 bytes) || s (32 bytes) || v (1 byte)
- **Validation**: Must be valid signature

## Signature Format

### ECDSA Signature Structure

```
Signature {
    r: [32 bytes]             // ECDSA signature component r
    s: [32 bytes]             // ECDSA signature component s
    v: [1 byte]               // Recovery ID (27 or 28)
}
```

### Signature Calculation

```
1. Create unsigned transaction (without signature)
   ↓
2. RLP encode unsigned transaction
   ↓
3. Hash with Keccak-256: hash = keccak256(rlp_encode(unsigned_tx))
   ↓
4. Sign hash with ECDSA: signature = ecdsa_sign(hash, private_key)
   ↓
5. Encode signature: r || s || v
```

### Signature Verification

```
1. Extract r, s, v from signature
   ↓
2. Recover public key from signature
   ↓
3. Derive address from public key
   ↓
4. Compare with from address
   ↓
5. Verify signature validity
```

## Transaction Hash

### Hash Calculation

```
tx_hash = keccak256(rlp_encode(signed_tx))
```

### Hash Properties
- **Deterministic**: Same transaction = same hash
- **Unique**: Different transactions = different hashes
- **Purpose**: Transaction identification

## Mempool Transaction Format

### Mempool Entry

```
MempoolTx {
    signed_tx: SignedTx       // Signed transaction
    received_at: [8 bytes]     // Timestamp (Unix, nanoseconds)
    priority: [8 bytes]        // Priority score (big-endian uint64)
    source: [20 bytes]          // Source peer (optional)
}
```

### Priority Calculation

```
priority = (gas_price × gas_limit) / (current_time - received_at)
```

### Priority Factors
- **Gas Price**: Higher = higher priority
- **Gas Limit**: Higher = higher priority
- **Age**: Older = lower priority
- **Result**: Fee-based prioritization

## Transaction Types

### Transfer Transaction

```
TransferTx {
    from: [20 bytes]
    to: [20 bytes]
    value: [32 bytes]
    data: []                   // Empty
    nonce: [8 bytes]
    gas_limit: [8 bytes]        // Minimum: 21,000
    gas_price: [32 bytes]
    chain_id: [8 bytes]
    signature: [65 bytes]
}
```

### Contract Creation Transaction

```
ContractCreationTx {
    from: [20 bytes]
    to: [20 bytes]             // All zeros
    value: [32 bytes]           // Optional deployment payment
    data: [variable]           // Contract bytecode
    nonce: [8 bytes]
    gas_limit: [8 bytes]       // Higher than transfer
    gas_price: [32 bytes]
    chain_id: [8 bytes]
    signature: [65 bytes]
}
```

### Contract Call Transaction

```
ContractCallTx {
    from: [20 bytes]
    to: [20 bytes]             // Contract address
    value: [32 bytes]           // Optional payment
    data: [variable]           // Function call + parameters
    nonce: [8 bytes]
    gas_limit: [8 bytes]        // Estimated gas
    gas_price: [32 bytes]
    chain_id: [8 bytes]
    signature: [65 bytes]
}
```

## Transaction Validation

### Pre-Execution Validation

#### Format Validation
- **Size**: Transaction size reasonable
- **Fields**: All fields present and valid
- **Encoding**: Valid RLP encoding
- **Signature**: Valid signature format

#### Content Validation
- **From Address**: Valid address format
- **To Address**: Valid address or zero
- **Value**: Non-negative
- **Nonce**: Non-negative
- **Gas Limit**: >= minimum (21,000)
- **Gas Price**: Non-negative
- **Chain ID**: Matches network

#### Signature Validation
- **Signature Format**: Valid r, s, v
- **Signature Validity**: Valid ECDSA signature
- **Address Match**: Recovered address matches from
- **Replay Protection**: Chain ID prevents replay

### Account Validation

#### Balance Check
```
required_balance = value + (gas_limit × gas_price)
if account.balance < required_balance:
    reject transaction
```

#### Nonce Check
```
if transaction.nonce != account.nonce:
    reject transaction
```

#### Account Existence
- **From Account**: Must exist (or will be created)
- **To Account**: Created if doesn't exist (for transfers)

## Transaction Execution

### Execution Process

```
1. Validate transaction (pre-execution)
   ↓
2. Deduct gas_cost from sender balance
   ↓
3. Execute transaction logic
   ↓
4. Update account balances
   ↓
5. Update account nonces
   ↓
6. Emit events
   ↓
7. Generate receipt
```

### Gas Calculation

```
gas_used = base_gas + data_gas + execution_gas
gas_cost = gas_used × gas_price
```

#### Base Gas
- **Transfer**: 21,000 gas
- **Contract Creation**: 32,000 gas

#### Data Gas
- **Zero Bytes**: 4 gas per byte
- **Non-Zero Bytes**: 16 gas per byte

#### Execution Gas
- **Operation Costs**: Per-operation gas costs
- **Storage Costs**: SLOAD, SSTORE costs
- **Computation Costs**: ADD, MUL, etc.

## Transaction Receipt

### Receipt Format

```
Receipt {
    status: [1 byte]           // 0 = failed, 1 = success
    gas_used: [8 bytes]        // Gas consumed (big-endian uint64)
    logs: [Log[]]              // Event logs
    logs_bloom: [256 bytes]    // Bloom filter for logs
    contract_address: [20 bytes] // Created contract (if applicable)
}
```

### Receipt Storage
- **Key**: Block number || transaction index
- **Value**: Serialized receipt
- **Location**: RocksDB Receipts column family
- **Index**: Transaction hash → receipt

## Transaction Limits

### Size Limits
- **Maximum Data Size**: 128 KB
- **Maximum Gas Limit**: 2^64 - 1
- **Maximum Value**: 2^256 - 1

### Rate Limits
- **Per Account**: Limited by nonce
- **Per Block**: Limited by block gas limit
- **Per Mempool**: Limited by mempool size

## Transaction Serialization

### RLP Encoding
- **Format**: Recursive Length Prefix
- **Purpose**: Ethereum compatibility
- **Usage**: Transaction serialization
- **Benefits**: Deterministic, efficient

### Serialization Order
1. from
2. to
3. value
4. data
5. nonce
6. gas_limit
7. gas_price
8. chain_id
9. signature (r, s, v)

## Transaction Propagation

### Propagation Format
- **Broadcast**: Full signed transaction
- **Request**: Transaction hash
- **Response**: Full signed transaction
- **Deduplication**: Hash-based

### Propagation Protocol
```
1. User creates and signs transaction
   ↓
2. Transaction broadcast to peers
   ↓
3. Peers validate transaction
   ↓
4. Valid transactions added to mempool
   ↓
5. Transaction forwarded to other peers
   ↓
6. Proposer includes in block
```

---

*The transaction format is critical for network interoperability. All implementations must strictly adhere to this specification.*


