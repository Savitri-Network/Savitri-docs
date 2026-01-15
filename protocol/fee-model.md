# Fee Model

## Overview

This document describes the fee model, gas pricing, and economic incentives in Savitri Network.

## Gas Model

### Gas Concept

Gas is a unit measuring computational cost. Every operation consumes gas, and transactions include a gas limit and gas price.

### Gas Costs

#### Base Costs
- **Transaction**: 21,000 gas
- **Contract Creation**: 32,000 gas
- **Zero-byte Data**: 4 gas per byte
- **Non-zero-byte Data**: 16 gas per byte

#### Operation Costs
- **ADD/SUB**: 3 gas
- **MUL**: 5 gas
- **DIV**: 5 gas
- **SLOAD**: 200 gas
- **SSTORE**: 20,000 gas (new), 5,000 gas (update)

### Gas Calculation

```
total_gas = base_gas + data_gas + execution_gas
fee = total_gas × gas_price
```

## Fee Structure

### Transaction Fee

```
transaction_fee = gas_used × gas_price
```

### Fee Distribution

#### Proposer Reward
```
proposer_reward = transaction_fee × 0.4
```

#### Voter Rewards
```
voter_reward = (transaction_fee × 0.6) / num_voters
```

## Adaptive Fee Model

### Dynamic Gas Price

Gas price adjusts based on network demand:

```
base_gas_price = base_price
demand_factor = recent_blocks_gas_used / target_gas_used
adjusted_gas_price = base_gas_price × demand_factor
```

### Priority Scoring

Transactions prioritized by:

```
priority_score = (gas_price × gas_limit) / transaction_age
```

## Economic Incentives

### Validator Incentives
- Transaction fees
- Block rewards
- Performance bonuses

### User Incentives
- Predictable fees
- Fast confirmation
- Reliable service

---

*The fee model balances network sustainability with user accessibility.*


