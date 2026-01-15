# State Transitions

## Overview

This document describes state transition rules, account updates, and state management in Savitri Network.

## State Transition Function

### Transition Process

```
State_i → Transaction → State_i+1
```

### Transition Rules

#### Account Updates
- Balance updates
- Nonce increments
- Code updates (contracts)
- Storage updates (contracts)

#### State Root Computation
- Update state trie
- Compute new state root
- Store in block header

## Account State

### Account Fields
- Address
- Balance
- Nonce
- Code Hash
- Storage Root

### Account Updates

#### Balance Update
```
new_balance = old_balance + value_received - value_sent - gas_cost
```

#### Nonce Update
```
new_nonce = old_nonce + 1
```

## State Consistency

### Consistency Rules
- State root must match computed state
- Account balances must be non-negative
- Nonces must increment sequentially
- Storage must be consistent

---

*State transitions ensure deterministic and consistent network state.*


