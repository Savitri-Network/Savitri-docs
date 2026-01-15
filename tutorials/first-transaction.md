# First Transaction Tutorial

## Overview

This tutorial will guide you through creating and executing your first transaction on Savitri Network. By the end, you'll understand how to set up a wallet, generate keys, create a transaction, and broadcast it to the network.

## Prerequisites

### System Requirements
- **Operating System**: Windows 10+, macOS 10.15+, or Linux (Ubuntu 18.04+)
- **RAM**: Minimum 4GB, recommended 8GB
- **Storage**: 10GB free disk space
- **Network**: Stable internet connection

### Software Required
- **Rust**: 1.70.0 or later
- **Git**: For cloning repositories
- **Savitri CLI**: Command-line interface tool

### Network Access
- Access to Savitri Network testnet or mainnet
- Basic understanding of command-line operations
- Familiarity with cryptocurrency concepts (helpful but not required)

## Step 1: Setting Up Your Environment

### Install Rust

```bash
# Install Rust using rustup
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"

# Verify installation
rustc --version
cargo --version
```

### Install Savitri CLI

```bash
# Clone the Savitri repository
git clone https://github.com/savitri-network/savitri-cli.git
cd savitri-cli

# Build and install
cargo install --path .

# Verify installation
savitri --version
```

### Configure Network Access

```bash
# Connect to testnet (recommended for first transaction)
savitri config set network testnet

# Verify connection
savitri network status
```

## Step 2: Creating Your First Wallet

### Generate a New Wallet

```bash
# Create a new wallet
savitri wallet create my-first-wallet

# Output example:
# Wallet created successfully!
# Wallet ID: my-first-wallet
# Public Address: 0x1234567890abcdef1234567890abcdef12345678
# Mnemonic: abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about
# IMPORTANT: Write down your mnemonic phrase and store it safely!
```

### Secure Your Wallet

**Critical Security Step**: Your mnemonic phrase is the master key to your wallet. Anyone with access to it can control your funds.

```bash
# Write down your mnemonic phrase
# Store it in multiple secure locations:
# - Physical safe deposit box
# - Encrypted digital storage
# - Trusted family member
# - Secure cloud storage with encryption
```

### Verify Wallet Creation

```bash
# List your wallets
savitri wallet list

# Show wallet details
savitri wallet show my-first-wallet

# Check balance (should be 0 initially)
savitri wallet balance my-first-wallet
```

## Step 3: Understanding Account Structure

### Account Keys

Savitri uses public/private key cryptography for account security:

```rust
// Account structure (for understanding)
pub struct Account {
    pub address: [u8; 32],        // Public address (derived from private key)
    pub public_key: [u8; 32],     // Public key for verification
    pub private_key: [u8; 32],    // Private key (keep secret!)
    pub nonce: u64,               // Transaction counter
    pub balance: u128,            // Account balance
}
```

### Address Generation

Your public address is derived from your private key:

```bash
# Show your address
savitri wallet address my-first-wallet

# Verify address derivation
savitri wallet verify-address my-first-wallet 0x1234567890abcdef1234567890abcdef12345678
```

## Step 4: Getting Test Tokens

### Request Test Tokens from Faucet

```bash
# Request test tokens (testnet only)
savitri faucet request my-first-wallet

# Alternative: Use web faucet
# Visit: https://faucet.savitri.network
# Enter your address: 0x1234567890abcdef1234567890abcdef12345678
# Click "Request Tokens"
```

### Verify Token Receipt

```bash
# Check your balance
savitri wallet balance my-first-wallet

# Should show something like:
# Wallet: my-first-wallet
# Address: 0x1234567890abcdef1234567890abcdef12345678
# Balance: 1000 SAV
# Nonce: 0
```

### Wait for Confirmation

```bash
# Monitor transaction status
savitri faucet status <transaction-id>

# Wait for confirmation (usually 1-2 minutes on testnet)
```

## Step 5: Understanding Transaction Structure

### Transaction Components

A Savitri transaction contains:

```rust
pub struct Transaction {
    pub version: u8,              // Transaction version
    pub sender: [u8; 32],         // Sender address
    pub recipient: [u8; 32],       // Recipient address
    pub amount: u128,             // Amount to transfer
    pub fee: u64,                 // Transaction fee
    pub nonce: u64,               // Sender's nonce
    pub signature: [u8; 64],      // Digital signature
    pub memo: Option<String>,     // Optional memo
    pub timestamp: u64,           // Transaction timestamp
}
```

### Transaction Fees

Fees compensate validators for processing transactions:

```bash
# Check current fee rates
savitri network fees

# Estimate transaction fee
savitri wallet estimate-fee my-first-wallet --amount 100 --recipient 0xabcdef1234567890abcdef1234567890abcdef12

# Output example:
# Estimated fee: 0.001 SAV
# Gas limit: 21000
# Gas price: 0.000000047 SAV/gas
```

## Step 6: Creating Your First Transaction

### Prepare Transaction Details

```bash
# Set transaction parameters
RECIPIENT="0xabcdef1234567890abcdef1234567890abcdef12"
AMOUNT="100"  # 100 SAV tokens
MEMO="My first transaction!"

# Verify recipient address
savitri wallet verify-address $RECIPIENT
```

### Create the Transaction

```bash
# Create transaction (interactive mode)
savitri wallet send my-first-wallet

# Follow the prompts:
# 1. Enter recipient address: 0xabcdef1234567890abcdef1234567890abcdef12
# 2. Enter amount: 100
# 3. Enter fee (or press Enter for default): 0.001
# 4. Enter memo (optional): My first transaction!
# 5. Confirm transaction? (y/n): y
```

### Create Transaction (Non-Interactive)

```bash
# Create transaction with parameters
savitri wallet send my-first-wallet \
  --recipient 0xabcdef1234567890abcdef1234567890abcdef12 \
  --amount 100 \
  --fee 0.001 \
  --memo "My first transaction!" \
  --confirm
```

### Transaction Output

```bash
# Expected output:
# Transaction created successfully!
# Transaction ID: 0x9876543210fedcba9876543210fedcba9876543210fedcba9876543210fedcba
# From: 0x1234567890abcdef1234567890abcdef12345678
# To: 0xabcdef1234567890abcdef1234567890abcdef12
# Amount: 100 SAV
# Fee: 0.001 SAV
# Nonce: 0
# Status: Pending
```

## Step 7: Broadcasting the Transaction

### Automatic Broadcasting

The CLI automatically broadcasts transactions when created:

```bash
# Transaction is already broadcast from previous step
# Check status
savitri wallet transaction-status 0x9876543210fedcba9876543210fedcba9876543210fedcba9876543210fedcba
```

### Manual Broadcasting (if needed)

```bash
# Broadcast transaction manually
savitri wallet broadcast my-first-wallet \
  --transaction-id 0x9876543210fedcba9876543210fedcba9876543210fedcba9876543210fedcba
```

### Monitor Transaction Progress

```bash
# Watch transaction confirmation
savitri wallet watch 0x9876543210fedcba9876543210fedcba9876543210fedcba9876543210fedcba

# Output example:
# Watching transaction: 0x9876543210fedcba9876543210fedcba9876543210fedcba9876543210fedcba
# Status: Pending → Confirmed (Block #12345)
# Confirmations: 1/6
# Finality: Pending
```

## Step 8: Verifying Transaction Success

### Check Updated Balances

```bash
# Check sender balance
savitri wallet balance my-first-wallet

# Should show:
# Wallet: my-first-wallet
# Address: 0x1234567890abcdef1234567890abcdef12345678
# Balance: 899.999 SAV (1000 - 100 - 0.001 - fees)
# Nonce: 1

# Check recipient balance (if you control it)
savitri wallet balance recipient-wallet
```

### Verify on Blockchain Explorer

```bash
# Open blockchain explorer
# Testnet: https://testnet.explorer.savitri.network
# Mainnet: https://explorer.savitri.network

# Search for your transaction ID
# Transaction ID: 0x9876543210fedcba9876543210fedcba9876543210fedcba9876543210fedcba
```

### Transaction Details on Explorer

You should see:
- **Transaction Hash**: Your transaction ID
- **Status**: Confirmed
- **Block**: Block number where included
- **Timestamp**: When transaction was processed
- **From/To**: Sender and recipient addresses
- **Amount**: 100 SAV
- **Fee**: 0.001 SAV
- **Gas Used**: Actual gas consumed

## Step 9: Understanding Transaction Finality

### Finality Process

Savitri uses BFT consensus for transaction finality:

```bash
# Check finality status
savitri wallet finality 0x9876543210fedcba9876543210fedcba9876543210fedcba9876543210fedcba

# Output example:
# Transaction: 0x9876543210fedcba9876543210fedcba9876543210fedcba9876543210fedcba
# Block: #12345
# Confirmations: 6/6
# Finality: Final (irreversible)
# Finality Time: 15.2 seconds
```

### Finality Guarantees

Once final, your transaction is:
- **Irreversible**: Cannot be reversed or double-spent
- **Immutable**: Permanently recorded on blockchain
- **Verifiable**: Anyone can verify the transaction
- **Secure**: Protected by cryptographic consensus

## Step 10: Advanced Transaction Features

### Batch Transactions

```bash
# Create multiple transactions in batch
savitri wallet batch-create my-first-wallet

# Add transactions to batch
savitri wallet batch-add \
  --recipient 0x1111111111111111111111111111111111111111 \
  --amount 50

savitri wallet batch-add \
  --recipient 0x2222222222222222222222222222222222222222 \
  --amount 25

# Send batch
savitri wallet batch-send my-first-wallet --confirm
```

### Scheduled Transactions

```bash
# Schedule transaction for future execution
savitri wallet schedule my-first-wallet \
  --recipient 0xabcdef1234567890abcdef1234567890abcdef12 \
  --amount 100 \
  --execute-at $(date -d "+1 hour" +%s)
```

### Conditional Transactions

```bash
# Create transaction with conditions
savitri wallet conditional my-first-wallet \
  --recipient 0xabcdef1234567890abcdef1234567890abcdef12 \
  --amount 100 \
  --condition "block_height > 15000"
```

## Step 11: Troubleshooting Common Issues

### Transaction Not Confirming

```bash
# Check transaction status
savitri wallet transaction-status <tx-id>

# Common issues and solutions:
# 1. Low fee - increase fee and retry
savitri wallet bump-fee <tx-id> --new-fee 0.002

# 2. Network congestion - wait and retry
savitri wallet retry <tx-id>

# 3. Invalid nonce - reset and retry
savitri wallet reset-nonce my-first-wallet
```

### Insufficient Balance

```bash
# Check balance and required fees
savitri wallet balance my-first-wallet
savitri network fees

# Calculate total required
# Required = Amount + Fee + Buffer

# Request more tokens if needed
savitri faucet request my-first-wallet
```

### Network Connection Issues

```bash
# Check network status
savitri network status

# Test connectivity
savitri network ping

# Switch to different node
savitri config set node https://backup-node.savitri.network
```

### Wallet Access Issues

```bash
# Recover wallet with mnemonic
savitri wallet recover my-first-wallet \
  --mnemonic "abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon about"

# Verify wallet recovery
savitri wallet show my-first-wallet
```

## Step 12: Security Best Practices

### Private Key Security

```bash
# NEVER share your private key or mnemonic
# NEVER store mnemonic in plain text
# ALWAYS use hardware wallets for large amounts
# ALWAYS enable additional security features
```

### Transaction Security

```bash
# ALWAYS verify recipient address before sending
# ALWAYS double-check amount and fee
# ALWAYS start with small test transactions
# ALWAYS use 2FA when available
```

### Wallet Backup

```bash
# Export wallet backup (encrypted)
savitri wallet backup my-first-wallet \
  --output wallet-backup.json \
  --encrypt

# Store backup securely
# - Multiple physical locations
# - Encrypted cloud storage
# - Trusted family members
```

## Step 13: Next Steps

### Explore Advanced Features

```bash
# Learn about smart contracts
savitri docs smart-contracts

# Explore staking and validation
savitri docs staking

# Join community discussions
savitri community join
```

### Build Applications

```bash
# Get SDK for your preferred language
savitri sdk install rust
savitri sdk install javascript
savitri sdk install python

# Build your first dApp
savitri init my-first-dapp
cd my-first-dapp
savitri dev
```

### Contribute to Network

```bash
# Run a node
savitri node run --mode validator

# Participate in governance
savitri governance list-proposals
savitri governance vote <proposal-id> yes

# Join community
savitri community events
savitri community contribute
```

## Conclusion

Congratulations! You've successfully created and executed your first transaction on Savitri Network. You've learned:

- ✅ How to set up a secure wallet
- ✅ How to generate and manage cryptographic keys
- ✅ How to create and broadcast transactions
- ✅ How to verify transaction finality
- ✅ Security best practices for blockchain interactions

### What You've Achieved

- **Technical Understanding**: You now understand how blockchain transactions work
- **Practical Skills**: You can interact with Savitri Network confidently
- **Security Awareness**: You know how to protect your digital assets
- **Foundation for Growth**: You're ready to explore more advanced features

### Continue Your Journey

1. **Explore Smart Contracts**: Learn to build and interact with smart contracts
2. **Run a Node**: Contribute to network security by running a validator node
3. **Build Applications**: Create decentralized applications on Savitri
4. **Join Community**: Participate in governance and community discussions

### Resources for Learning

- **Documentation**: https://docs.savitri.network
- **Community Forum**: https://community.savitri.network
- **Developer Portal**: https://developers.savitri.network
- **GitHub Repository**: https://github.com/savitri-network

---

*Welcome to the decentralized future. Your first transaction is just the beginning of your journey with Savitri Network.*
