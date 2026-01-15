# Build on Savitri

## Overview

This tutorial provides comprehensive guidance for developers building applications on the Savitri Network. It covers smart contract development, decentralized application (DApp) architecture, integration patterns, and best practices for creating robust blockchain solutions.

## Prerequisites

- Rust programming knowledge
- Understanding of blockchain concepts
- Familiarity with smart contracts
- Development environment setup
- Savitri Network node access

## Smart Contract Development

### Contract Structure

Savitri smart contracts follow a structured architecture with base functionality, standards compliance, and execution environment integration.

```rust
// src/contracts/base.rs
pub trait BaseContract {
    fn owner(&self) -> Address;
    fn transfer_ownership(&mut self, new_owner: Address) -> ContractResult<()>;
    fn version(&self) -> String;
    fn reserved_storage(&self, slot: u32) -> Option<Vec<u8>>;
}
```

### Token Standards

#### SAVITRI-20 (Fungible Tokens)

```rust
// src/contracts/standards/savitri20.rs
pub trait Savitri20 {
    fn total_supply(&self) -> u128;
    fn balance_of(&self, owner: Address) -> u128;
    fn transfer(&mut self, recipient: Address, amount: u128) -> ContractResult<()>;
    fn approve(&mut self, spender: Address, amount: u128) -> ContractResult<()>;
    fn transfer_from(&mut self, sender: Address, recipient: Address, amount: u128) -> ContractResult<()>;
    fn allowance(&self, owner: Address, spender: Address) -> u128;
}
```

#### SAVITRI-721 (Non-Fungible Tokens)

```rust
// src/contracts/standards/savitri721.rs
pub trait Savitri721 {
    fn balance_of(&self, owner: Address) -> u128;
    fn owner_of(&self, token_id: u64) -> Option<Address>;
    fn transfer_from(&mut self, from: Address, to: Address, token_id: u64) -> ContractResult<()>;
    fn approve(&mut self, to: Address, token_id: u64) -> ContractResult<()>;
    fn safe_transfer_from(&mut self, from: Address, to: Address, token_id: u64, data: Option<Vec<u8>>) -> ContractResult<()>;
    fn token_uri(&self, token_id: u64) -> Option<String>;
}
```

### Contract Implementation Example

```rust
use savitri::contracts::{BaseContract, Savitri20};
use savitri::types::{Address, ContractResult};

pub struct MyToken {
    // Base contract fields
    owner: Address,
    version: String,
    
    // SAVITRI-20 fields
    total_supply: u128,
    balances: std::collections::HashMap<Address, u128>,
    allowances: std::collections::HashMap<(Address, Address), u128>,
    
    // Custom fields
    name: String,
    symbol: String,
    decimals: u8,
}

impl BaseContract for MyToken {
    fn owner(&self) -> Address {
        self.owner
    }
    
    fn transfer_ownership(&mut self, new_owner: Address) -> ContractResult<()> {
        if msg::sender() != self.owner {
            return Err("Only owner can transfer ownership");
        }
        self.owner = new_owner;
        Ok(())
    }
    
    fn version(&self) -> String {
        self.version.clone()
    }
    
    fn reserved_storage(&self, slot: u32) -> Option<Vec<u8>> {
        match slot {
            0 => Some(self.owner.as_bytes().to_vec()),
            1 => Some(self.total_supply.to_le_bytes().to_vec()),
            _ => None,
        }
    }
}

impl Savitri20 for MyToken {
    fn total_supply(&self) -> u128 {
        self.total_supply
    }
    
    fn balance_of(&self, owner: Address) -> u128 {
        self.balances.get(&owner).copied().unwrap_or(0)
    }
    
    fn transfer(&mut self, recipient: Address, amount: u128) -> ContractResult<()> {
        let sender = msg::sender();
        let sender_balance = self.balance_of(sender);
        
        if sender_balance < amount {
            return Err("Insufficient balance");
        }
        
        self.balances.insert(sender, sender_balance - amount);
        self.balances.insert(recipient, self.balance_of(recipient) + amount);
        
        // Emit transfer event
        event::emit("Transfer", &[
            sender.as_bytes(),
            recipient.as_bytes(),
            &amount.to_le_bytes()
        ]);
        
        Ok(())
    }
    
    fn approve(&mut self, spender: Address, amount: u128) -> ContractResult<()> {
        let owner = msg::sender();
        self.allowances.insert((owner, spender), amount);
        
        // Emit approval event
        event::emit("Approval", &[
            owner.as_bytes(),
            spender.as_bytes(),
            &amount.to_le_bytes()
        ]);
        
        Ok(())
    }
    
    fn transfer_from(&mut self, sender: Address, recipient: Address, amount: u128) -> ContractResult<()> {
        let spender = msg::sender();
        let allowance = self.allowances.get(&(sender, spender)).copied().unwrap_or(0);
        
        if allowance < amount {
            return Err("Insufficient allowance");
        }
        
        let sender_balance = self.balance_of(sender);
        if sender_balance < amount {
            return Err("Insufficient balance");
        }
        
        // Update balances
        self.balances.insert(sender, sender_balance - amount);
        self.balances.insert(recipient, self.balance_of(recipient) + amount);
        
        // Update allowance
        self.allowances.insert((sender, spender), allowance - amount);
        
        // Emit events
        event::emit("Transfer", &[
            sender.as_bytes(),
            recipient.as_bytes(),
            &amount.to_le_bytes()
        ]);
        
        event::emit("Approval", &[
            sender.as_bytes(),
            spender.as_bytes(),
            &allowance.to_le_bytes()
        ]);
        
        Ok(())
    }
    
    fn allowance(&self, owner: Address, spender: Address) -> u128 {
        self.allowances.get(&(owner, spender)).copied().unwrap_or(0)
    }
}

impl MyToken {
    pub fn new(name: String, symbol: String, initial_supply: u128, decimals: u8) -> Self {
        let owner = msg::sender();
        let mut balances = std::collections::HashMap::new();
        balances.insert(owner, initial_supply);
        
        Self {
            owner,
            version: "1.0.0".to_string(),
            total_supply: initial_supply,
            balances,
            allowances: std::collections::HashMap::new(),
            name,
            symbol,
            decimals,
        }
    }
    
    pub fn name(&self) -> &str {
        &self.name
    }
    
    pub fn symbol(&self) -> &str {
        &self.symbol
    }
    
    pub fn decimals(&self) -> u8 {
        self.decimals
    }
}
```

## DApp Architecture

### Frontend Integration

#### Web3 Connection

```javascript
// JavaScript frontend integration
class SavitriWeb3 {
    constructor() {
        this.provider = null;
        this.signer = null;
        this.contract = null;
    }
    
    async connect() {
        // Connect to Savitri network
        if (typeof window.savitri !== 'undefined') {
            this.provider = new window.savitri.providers.Web3Provider(
                window.savitri
            );
            await this.provider.send("eth_requestAccounts", []);
            this.signer = this.provider.getSigner();
        } else {
            throw new Error("Savitri wallet not detected");
        }
    }
    
    async loadContract(address, abi) {
        this.contract = new window.savitri.Contract(address, abi, this.signer);
        return this.contract;
    }
    
    async getBalance(address) {
        const balance = await this.provider.getBalance(address);
        return window.savitri.utils.formatEther(balance);
    }
    
    async sendTransaction(to, value, data = "0x") {
        const tx = {
            to: to,
            value: window.savitri.utils.parseEther(value.toString()),
            data: data
        };
        
        const receipt = await this.signer.sendTransaction(tx);
        return receipt;
    }
}

// Usage example
const web3 = new SavitriWeb3();

async function initializeDApp() {
    try {
        await web3.connect();
        const contractAddress = "0x1234..."; // Your contract address
        const contractABI = [/* Your contract ABI */];
        const contract = await web3.loadContract(contractAddress, contractABI);
        
        // Read contract data
        const totalSupply = await contract.totalSupply();
        console.log("Total Supply:", totalSupply.toString());
        
        // Write to contract
        const recipient = "0x5678...";
        const amount = "100";
        const tx = await contract.transfer(recipient, amount);
        console.log("Transaction hash:", tx.hash);
        
    } catch (error) {
        console.error("DApp initialization failed:", error);
    }
}
```

#### React Integration

```jsx
// React component example
import React, { useState, useEffect } from 'react';
import { SavitriWeb3 } from './savitri-web3';

function TokenBalance({ address }) {
    const [balance, setBalance] = useState('0');
    const [loading, setLoading] = useState(true);
    
    useEffect(() => {
        const fetchBalance = async () => {
            try {
                const web3 = new SavitriWeb3();
                await web3.connect();
                const balance = await web3.getBalance(address);
                setBalance(balance);
            } catch (error) {
                console.error("Failed to fetch balance:", error);
            } finally {
                setLoading(false);
            }
        };
        
        fetchBalance();
    }, [address]);
    
    if (loading) return <div>Loading balance...</div>;
    
    return <div>Balance: {balance} SAV</div>;
}

function TokenTransfer() {
    const [recipient, setRecipient] = useState('');
    const [amount, setAmount] = useState('');
    const [loading, setLoading] = useState(false);
    
    const handleTransfer = async (e) => {
        e.preventDefault();
        setLoading(true);
        
        try {
            const web3 = new SavitriWeb3();
            await web3.connect();
            
            const tx = await web3.sendTransaction(recipient, amount);
            console.log("Transfer successful:", tx.hash);
            
            // Reset form
            setRecipient('');
            setAmount('');
        } catch (error) {
            console.error("Transfer failed:", error);
        } finally {
            setLoading(false);
        }
    };
    
    return (
        <form onSubmit={handleTransfer}>
            <div>
                <label>Recipient:</label>
                <input
                    type="text"
                    value={recipient}
                    onChange={(e) => setRecipient(e.target.value)}
                    placeholder="0x..."
                    required
                />
            </div>
            <div>
                <label>Amount:</label>
                <input
                    type="number"
                    value={amount}
                    onChange={(e) => setAmount(e.target.value)}
                    placeholder="0.0"
                    step="0.000001"
                    required
                />
            </div>
            <button type="submit" disabled={loading}>
                {loading ? 'Transferring...' : 'Transfer'}
            </button>
        </form>
    );
}

export function TokenDApp() {
    const [account, setAccount] = useState(null);
    
    useEffect(() => {
        const connectWallet = async () => {
            try {
                const web3 = new SavitriWeb3();
                await web3.connect();
                const address = await web3.signer.getAddress();
                setAccount(address);
            } catch (error) {
                console.error("Failed to connect wallet:", error);
            }
        };
        
        connectWallet();
    }, []);
    
    if (!account) return <div>Please connect your wallet</div>;
    
    return (
        <div>
            <h1>Token DApp</h1>
            <TokenBalance address={account} />
            <TokenTransfer />
        </div>
    );
}
```

### Backend Integration

#### Node.js Backend

```javascript
// Node.js backend integration
const { SavitriProvider } = require('@savitri/providers');
const { Contract } = require('@savitri/contract');
const { Wallet } = require('@savitri/wallet');

class SavitriBackend {
    constructor(rpcUrl, privateKey) {
        this.provider = new SavitriProvider(rpcUrl);
        this.wallet = new Wallet(privateKey, this.provider);
        this.contracts = new Map();
    }
    
    async loadContract(address, abi) {
        const contract = new Contract(address, abi, this.wallet);
        this.contracts.set(address, contract);
        return contract;
    }
    
    async getBlockNumber() {
        return await this.provider.getBlockNumber();
    }
    
    async getTransactionReceipt(txHash) {
        return await this.provider.getTransactionReceipt(txHash);
    }
    
    async sendTransaction(to, data, value = 0) {
        const tx = {
            to: to,
            data: data,
            value: value,
            gasLimit: 1000000,
            gasPrice: await this.provider.getGasPrice()
        };
        
        const signedTx = await this.wallet.signTransaction(tx);
        const receipt = await this.provider.sendTransaction(signedTx);
        return receipt;
    }
    
    async callContract(address, method, params = []) {
        const contract = this.contracts.get(address);
        if (!contract) {
            throw new Error("Contract not loaded");
        }
        
        return await contract[method](...params);
    }
    
    async listenToEvents(address, eventName, callback) {
        const contract = this.contracts.get(address);
        if (!contract) {
            throw new Error("Contract not loaded");
        }
        
        contract.on(eventName, callback);
    }
}

// Usage example
const backend = new SavitriBackend(
    "https://rpc.savitri.network",
    "your-private-key"
);

async function initializeBackend() {
    try {
        // Load contract
        const contractAddress = "0x1234...";
        const contractABI = [/* Your contract ABI */];
        const contract = await backend.loadContract(contractAddress, contractABI);
        
        // Read contract data
        const totalSupply = await backend.callContract(contractAddress, "totalSupply");
        console.log("Total Supply:", totalSupply.toString());
        
        // Listen to events
        backend.listenToEvents(contractAddress, "Transfer", (from, to, amount, event) => {
            console.log("Transfer event:", { from, to, amount: amount.toString() });
            console.log("Transaction hash:", event.transactionHash);
        });
        
        // Send transaction
        const recipient = "0x5678...";
        const amount = 1000000000000000000; // 1 token in wei
        const data = contract.interface.encodeFunctionData("transfer", [recipient, amount]);
        
        const receipt = await backend.sendTransaction(contractAddress, data);
        console.log("Transaction receipt:", receipt);
        
    } catch (error) {
        console.error("Backend initialization failed:", error);
    }
}
```

#### Python Backend

```python
# Python backend integration
from web3 import Web3
from web3.middleware import geth_poa_middleware
import json

class SavitriPython:
    def __init__(self, rpc_url, private_key):
        self.w3 = Web3(Web3.HTTPProvider(rpc_url))
        self.w3.middleware_onion.inject(geth_poa_middleware, layer=0)
        self.account = self.w3.eth.account.from_key(private_key)
        self.contracts = {}
    
    def load_contract(self, address, abi_path):
        with open(abi_path, 'r') as f:
            abi = json.load(f)
        
        contract = self.w3.eth.contract(
            address=address,
            abi=abi
        )
        self.contracts[address] = contract
        return contract
    
    def get_block_number(self):
        return self.w3.eth.block_number
    
    def get_transaction_receipt(self, tx_hash):
        return self.w3.eth.get_transaction_receipt(tx_hash)
    
    def send_transaction(self, to, data, value=0):
        transaction = {
            'to': to,
            'data': data,
            'value': value,
            'gas': 1000000,
            'gasPrice': self.w3.eth.gas_price,
            'nonce': self.w3.eth.get_transaction_count(self.account.address)
        }
        
        signed_txn = self.w3.eth.account.sign_transaction(transaction, self.account.key)
        tx_hash = self.w3.eth.send_raw_transaction(signed_txn.rawTransaction)
        receipt = self.w3.eth.wait_for_transaction_receipt(tx_hash)
        return receipt
    
    def call_contract(self, address, method, *args):
        contract = self.contracts.get(address)
        if not contract:
            raise ValueError("Contract not loaded")
        
        return getattr(contract.functions, method)(*args).call()
    
    def listen_to_events(self, address, event_name, callback):
        contract = self.contracts.get(address)
        if not contract:
            raise ValueError("Contract not loaded")
        
        event = getattr(contract.events, event_name)
        event_filter = event.create_filter(fromBlock='latest')
        
        while True:
            for event_data in event_filter.get_new_entries():
                callback(event_data)

# Usage example
backend = SavitriPython(
    "https://rpc.savitri.network",
    "your-private-key"
)

def initialize_backend():
    try:
        # Load contract
        contract_address = "0x1234..."
        contract = backend.load_contract(contract_address, "contract_abi.json")
        
        # Read contract data
        total_supply = backend.call_contract(contract_address, "totalSupply")
        print(f"Total Supply: {total_supply}")
        
        # Send transaction
        recipient = "0x5678..."
        amount = 1000000000000000000  # 1 token in wei
        
        # Build transaction data
        contract = backend.contracts[contract_address]
        data = contract.functions.transfer(recipient, amount).build_transaction({
            'from': backend.account.address,
            'gas': 1000000,
            'gasPrice': backend.w3.eth.gas_price,
            'nonce': backend.w3.eth.get_transaction_count(backend.account.address)
        })['data']
        
        receipt = backend.send_transaction(contract_address, data)
        print(f"Transaction receipt: {receipt}")
        
    except Exception as error:
        print(f"Backend initialization failed: {error}")

if __name__ == "__main__":
    initialize_backend()
```

## Development Tools

### Contract Testing

```rust
// Contract testing framework
#[cfg(test)]
mod tests {
    use super::*;
    use savitri::testing::{MockContract, TestContext};
    
    #[test]
    fn test_token_transfer() {
        let mut context = TestContext::new();
        let owner = context.create_account("owner");
        let recipient = context.create_account("recipient");
        
        // Deploy contract
        let contract = MyToken::new(
            "Test Token".to_string(),
            "TEST".to_string(),
            1000000,
            18
        );
        
        // Test initial balance
        assert_eq!(contract.balance_of(owner), 1000000);
        assert_eq!(contract.balance_of(recipient), 0);
        
        // Test transfer
        context.set_sender(owner);
        contract.transfer(recipient, 100).unwrap();
        
        assert_eq!(contract.balance_of(owner), 999900);
        assert_eq!(contract.balance_of(recipient), 100);
    }
    
    #[test]
    fn test_approval_and_transfer_from() {
        let mut context = TestContext::new();
        let owner = context.create_account("owner");
        let spender = context.create_account("spender");
        let recipient = context.create_account("recipient");
        
        let contract = MyToken::new(
            "Test Token".to_string(),
            "TEST".to_string(),
            1000000,
            18
        );
        
        // Approve spender
        context.set_sender(owner);
        contract.approve(spender, 500).unwrap();
        
        assert_eq!(contract.allowance(owner, spender), 500);
        
        // Transfer from
        context.set_sender(spender);
        contract.transfer_from(owner, recipient, 300).unwrap();
        
        assert_eq!(contract.balance_of(owner), 999700);
        assert_eq!(contract.balance_of(recipient), 300);
        assert_eq!(contract.allowance(owner, spender), 200);
    }
}
```

### Deployment Scripts

```rust
// Contract deployment script
use savitri::deployment::{Deployer, ContractConfig};
use savitri::types::Address;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Initialize deployer
    let deployer = Deployer::new("https://rpc.savitri.network")
        .with_private_key("your-private-key")
        .await?;
    
    // Deploy token contract
    let token_config = ContractConfig {
        name: "MyToken".to_string(),
        constructor_args: vec![
            "My Token".to_string(),
            "MTK".to_string(),
            "1000000000000000000000000".to_string(), // 1M tokens
            "18".to_string(),
        ],
    };
    
    let token_address = deployer
        .deploy_contract("MyToken", token_config)
        .await?;
    
    println!("Token deployed at: {:?}", token_address);
    
    // Verify contract
    let verified = deployer
        .verify_contract(token_address, "MyToken")
        .await?;
    
    if verified {
        println!("Contract verified successfully");
    } else {
        println!("Contract verification failed");
    }
    
    Ok(())
}
```

## Best Practices

### Security Considerations

1. **Input Validation**: Always validate user inputs and contract parameters
2. **Access Control**: Implement proper ownership and permission checks
3. **Reentrancy Protection**: Use reentrancy guards for external calls
4. **Integer Overflow**: Use safe arithmetic operations
5. **Gas Optimization**: Minimize gas consumption for cost efficiency

### Performance Optimization

1. **Batch Operations**: Group multiple operations into single transactions
2. **Event Logging**: Use events for off-chain data storage
3. **Storage Optimization**: Minimize storage writes and use efficient data structures
4. **Caching**: Implement caching for frequently accessed data

### Development Workflow

1. **Local Testing**: Test contracts thoroughly on local testnet
2. **Testnet Deployment**: Deploy to testnet for integration testing
3. **Security Audit**: Conduct professional security audits
4. **Mainnet Deployment**: Deploy to mainnet with proper monitoring

## Common Patterns

### Factory Pattern

```rust
// Factory contract for deploying multiple instances
pub struct TokenFactory {
    owner: Address,
    deployed_tokens: Vec<Address>,
}

impl TokenFactory {
    pub fn new() -> Self {
        Self {
            owner: msg::sender(),
            deployed_tokens: Vec::new(),
        }
    }
    
    pub fn create_token(
        &mut self,
        name: String,
        symbol: String,
        initial_supply: u128,
        decimals: u8,
    ) -> Address {
        let token = MyToken::new(name, symbol, initial_supply, decimals);
        let token_address = token.deploy();
        self.deployed_tokens.push(token_address);
        token_address
    }
    
    pub fn get_deployed_tokens(&self) -> &Vec<Address> {
        &self.deployed_tokens
    }
}
```

### Upgradeable Contracts

```rust
// Proxy pattern for upgradeable contracts
pub struct ProxyContract {
    implementation: Address,
    admin: Address,
}

impl ProxyContract {
    pub fn new(implementation: Address) -> Self {
        Self {
            implementation,
            admin: msg::sender(),
        }
    }
    
    pub fn upgrade(&mut self, new_implementation: Address) -> ContractResult<()> {
        if msg::sender() != self.admin {
            return Err("Only admin can upgrade");
        }
        
        self.implementation = new_implementation;
        Ok(())
    }
    
    pub fn fallback(&mut self, data: &[u8]) -> ContractResult<Vec<u8>> {
        // Delegate call to implementation
        delegate_call(self.implementation, data)
    }
}
```

## Integration Examples

### DeFi Integration

```rust
// DeFi lending protocol integration
pub struct LendingProtocol {
    // Lending pool state
    total_deposits: u128,
    total_borrows: u128,
    deposit_rates: std::collections::HashMap<Address, u128>,
    borrow_rates: std::collections::HashMap<Address, u128>,
    
    // User positions
    user_deposits: std::collections::HashMap<Address, u128>,
    user_borrows: std::collections::HashMap<Address, u128>,
    
    // Token contracts
    collateral_token: Address,
    loan_token: Address,
}

impl LendingProtocol {
    pub fn deposit(&mut self, amount: u128) -> ContractResult<()> {
        let user = msg::sender();
        
        // Transfer collateral token
        self.transfer_from(self.collateral_token, user, address!(Self), amount)?;
        
        // Update user deposit
        let current_deposit = self.user_deposits.get(&user).copied().unwrap_or(0);
        self.user_deposits.insert(user, current_deposit + amount);
        self.total_deposits += amount;
        
        // Mint liquidity tokens
        self.mint_liquidity_tokens(user, amount)?;
        
        Ok(())
    }
    
    pub fn borrow(&mut self, amount: u128) -> ContractResult<()> {
        let user = msg::sender();
        
        // Check collateral ratio
        let collateral_value = self.get_collateral_value(user);
        let borrow_value = self.get_borrow_value(user) + amount;
        
        if borrow_value * 100 > collateral_value * 150 { // 150% collateral ratio
            return Err("Insufficient collateral");
        }
        
        // Transfer loan token
        self.transfer(self.loan_token, user, amount)?;
        
        // Update user borrow
        let current_borrow = self.user_borrows.get(&user).copied().unwrap_or(0);
        self.user_borrows.insert(user, current_borrow + amount);
        self.total_borrows += amount;
        
        Ok(())
    }
    
    pub fn repay(&mut self, amount: u128) -> ContractResult<()> {
        let user = msg::sender();
        
        // Transfer loan token back
        self.transfer_from(self.loan_token, user, address!(Self), amount)?;
        
        // Update user borrow
        let current_borrow = self.user_borrows.get(&user).copied().unwrap_or(0);
        let new_borrow = current_borrow.saturating_sub(amount);
        self.user_borrows.insert(user, new_borrow);
        self.total_borrows = self.total_borrows.saturating_sub(amount);
        
        // Burn liquidity tokens proportionally
        self.burn_liquidity_tokens(user, amount)?;
        
        Ok(())
    }
}
```

### NFT Marketplace

```rust
// NFT marketplace integration
pub struct NFTMarketplace {
    // Marketplace state
    listings: std::collections::HashMap<u64, Listing>,
    sales: Vec<Sale>,
    marketplace_fee: u128, // in basis points (100 = 1%)
    
    // NFT contract
    nft_contract: Address,
    
    // Payment token
    payment_token: Address,
}

pub struct Listing {
    seller: Address,
    token_id: u64,
    price: u128,
    expires: u64,
}

pub struct Sale {
    token_id: u64,
    seller: Address,
    buyer: Address,
    price: u128,
    timestamp: u64,
}

impl NFTMarketplace {
    pub fn list_token(&mut self, token_id: u64, price: u128, duration: u64) -> ContractResult<()> {
        let seller = msg::sender();
        
        // Check ownership
        let owner = self.owner_of(self.nft_contract, token_id);
        if owner != seller {
            return Err("Not token owner");
        }
        
        // Transfer NFT to marketplace
        self.transfer_from(self.nft_contract, seller, address!(Self), token_id)?;
        
        // Create listing
        let listing = Listing {
            seller,
            token_id,
            price,
            expires: block::timestamp() + duration,
        };
        
        self.listings.insert(token_id, listing);
        
        // Emit listing event
        event::emit("TokenListed", &[
            seller.as_bytes(),
            &token_id.to_le_bytes(),
            &price.to_le_bytes(),
        ]);
        
        Ok(())
    }
    
    pub fn buy_token(&mut self, token_id: u64) -> ContractResult<()> {
        let buyer = msg::sender();
        let listing = self.listings.get(&token_id)
            .ok_or("Token not listed")?;
        
        // Check if listing is still valid
        if block::timestamp() > listing.expires {
            return Err("Listing expired");
        }
        
        // Calculate fees
        let marketplace_fee = listing.price * self.marketplace_fee / 10000;
        let seller_amount = listing.price - marketplace_fee;
        
        // Transfer payment token
        self.transfer_from(self.payment_token, buyer, address!(Self), listing.price)?;
        
        // Transfer to seller
        self.transfer(self.payment_token, listing.seller, seller_amount)?;
        
        // Transfer NFT to buyer
        self.transfer(self.nft_contract, buyer, token_id)?;
        
        // Record sale
        let sale = Sale {
            token_id,
            seller: listing.seller,
            buyer,
            price: listing.price,
            timestamp: block::timestamp(),
        };
        self.sales.push(sale);
        
        // Remove listing
        self.listings.remove(&token_id);
        
        // Emit sale event
        event::emit("TokenSold", &[
            listing.seller.as_bytes(),
            buyer.as_bytes(),
            &token_id.to_le_bytes(),
            &listing.price.to_le_bytes(),
        ]);
        
        Ok(())
    }
    
    pub fn cancel_listing(&mut self, token_id: u64) -> ContractResult<()> {
        let seller = msg::sender();
        let listing = self.listings.get(&token_id)
            .ok_or("Token not listed")?;
        
        // Check ownership
        if listing.seller != seller {
            return Err("Not listing owner");
        }
        
        // Transfer NFT back to seller
        self.transfer(self.nft_contract, seller, token_id)?;
        
        // Remove listing
        self.listings.remove(&token_id);
        
        // Emit cancellation event
        event::emit("ListingCancelled", &[
            seller.as_bytes(),
            &token_id.to_le_bytes(),
        ]);
        
        Ok(())
    }
}
```

## Troubleshooting

### Common Issues

1. **Transaction Failures**: Check gas limits and contract state
2. **Connection Issues**: Verify RPC endpoint and network status
3. **Contract Deployment**: Ensure proper constructor arguments
4. **Event Listening**: Use correct event signatures and filters

### Debugging Tools

```rust
// Debugging utilities
#[cfg(debug_assertions)]
pub mod debug {
    use super::*;
    
    pub fn log_contract_state(contract: &dyn BaseContract) {
        println!("Contract owner: {:?}", contract.owner());
        println!("Contract version: {}", contract.version());
    }
    
    pub fn log_transaction(tx: &Transaction) {
        println!("Transaction hash: {:?}", tx.hash);
        println!("From: {:?}", tx.from);
        println!("To: {:?}", tx.to);
        println!("Value: {}", tx.value);
        println!("Gas: {}", tx.gas);
        println!("Gas price: {}", tx.gas_price);
    }
}
```

## Resources

### Documentation

- [Savitri Network Documentation](https://docs.savitri.network)
- [Smart Contract Development Guide](https://docs.savitri.network/contracts)
- [API Reference](https://docs.savitri.network/api)

### Tools

- [Savitri CLI](https://github.com/savitri/cli)
- [Contract Development Kit](https://github.com/savitri/cdk)
- [Testing Framework](https://github.com/savitri/testing)

### Community

- [Discord Server](https://discord.gg/savitri)
- [Telegram Group](https://t.me/savitri)
- [GitHub Discussions](https://github.com/savitri/discussions)

## Next Steps

1. **Set up development environment** with Savitri CLI and tools
2. **Create your first contract** using the provided templates
3. **Test thoroughly** on local testnet before deployment
4. **Deploy to testnet** for integration testing
5. **Launch on mainnet** with proper monitoring and security measures

This tutorial provides the foundation for building robust applications on the Savitri Network. For more advanced topics and specific use cases, refer to the detailed documentation and community resources.
