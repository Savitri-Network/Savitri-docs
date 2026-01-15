# Validator Node Setup Guide

## Overview

This comprehensive guide covers the complete process of setting up and operating a Savitri Network validator node. Validator nodes are critical for network security and consensus, requiring careful planning, proper hardware, and ongoing maintenance.

## Technology Choice Rationale

### Why Validator Setup Guide

**Problem Statement**: Setting up a validator node involves complex requirements including bond management, key security, consensus participation, and slashing protection, requiring detailed guidance for operators.

**Chosen Solution**: Comprehensive validator setup guide covering economics, hardware, security, operations, and troubleshooting.

**Rationale**:
- **Security**: Proper key management and slashing protection
- **Economics**: Clear understanding of bond requirements and rewards
- **Reliability**: High availability and uptime requirements
- **Compliance**: Adherence to network rules and regulations
- **Performance**: Optimal hardware and network configuration

**Expected Results**:
- Secure and reliable validator operations
- Proper understanding of economic incentives and risks
- High availability and performance
- Compliance with network requirements
- Effective risk management and mitigation

## Validator Economics

### Bond Requirements

**Minimum Bond**: 1,000,000 SAV tokens (~$1M+ USD value)
```rust
// Bond amount constants
pub const DEFAULT_BOND_AMOUNT: u128 = 1_000_000_000_000_000; // 1M tokens
pub const MIN_BOND_AMOUNT: u128 = 500_000_000_000_000;    // 500K tokens
pub const MAX_BOND_AMOUNT: u128 = 10_000_000_000_000_000; // 10M tokens
```

**Bond Economics**:
- **Lock Period**: 30 days minimum
- **Unbonding Period**: 21 days withdrawal delay
- **Slashing Penalties**: Up to 50% of bond amount
- **Reward Rate**: Variable based on network participation

### Reward Structure

**Reward Calculation**:
```rust
impl RewardCalculator {
    pub fn calculate_validator_reward(&self, validator: &ValidatorInfo, block_height: u64) -> u128 {
        let base_reward = (self.base_reward_rate * 1e18) as u128;
        
        // Participation bonus (blocks participated / total blocks)
        let participation_bonus = (validator.participation_rate * self.participation_multiplier) as u128;
        
        // Uptime bonus (time online / total time)
        let uptime_bonus = (validator.uptime_rate * self.uptime_multiplier) as u128;
        
        // Performance bonus (blocks produced / expected blocks)
        let performance_bonus = (validator.performance_rate * self.performance_multiplier) as u128;
        
        let total_multiplier = 1.0 + participation_bonus + uptime_bonus + performance_bonus;
        
        (base_reward * total_multiplier as u128) / 1000
    }
}
```

**Reward Distribution**:
- **Block Rewards**: Distributed to leader and active validators
- **Transaction Fees**: Shared among participating validators
- **Network Fees**: Distributed based on participation and performance
- **Slash Penalties**: Distributed to honest validators when misbehavior detected

### Risk Analysis

**Slashing Conditions**:
```rust
pub enum SlashingCondition {
    // Equivocation (double signing)
    DoubleSign {
        block_height: u64,
        first_signature: Signature,
        second_signature: Signature,
    },
    
    // Unavailability (missing blocks)
    Unavailable {
        missed_slots: u64,
        total_slots: u64,
        period: Duration,
    },
    
    // Invalid block proposals
    InvalidBlock {
        block_hash: BlockHash,
        reason: InvalidBlockReason,
    },
    
    // Vote manipulation
    VoteManipulation {
        conflicting_votes: Vec<Vote>,
    },
}
```

**Slashing Penalties**:
- **Double Sign**: 50% of bond amount
- **Unavailability**: 10-30% based on downtime
- **Invalid Blocks**: 20-40% based on severity
- **Vote Manipulation**: 30-50% based on impact

## Hardware Requirements

### Minimum Specifications

**CPU**: 16 cores, 3.0GHz+ (Intel/AMD x64)
- **Rationale**: Consensus processing, block production, parallel validation
- **Expected Performance**: 2000+ TPS validation capacity

**Memory**: 32GB RAM
- **Rationale**: State caching, consensus state, transaction pool
- **Expected Usage**: 16GB for state, 8GB for consensus, 8GB for system

**Storage**: 1TB NVMe SSD
- **Rationale**: Blockchain storage, consensus state, fast I/O for block production
- **Expected Growth**: ~5GB per month with current network activity

**Network**: 1Gbps+ symmetric connection
- **Rationale**: Consensus messaging, block propagation, low latency requirements
- **Expected Bandwidth**: 100GB+ per month for consensus and P2P

### Recommended Specifications

**CPU**: 32 cores, 3.5GHz+ (Intel/AMD x64)
- **Benefits**: Better consensus performance, higher TPS, parallel processing
- **Recommended**: Intel Xeon or AMD EPYC series

**Memory**: 64GB RAM
- **Benefits**: Larger state cache, better consensus performance, higher mempool
- **Recommended**: ECC RAM for error correction

**Storage**: 2TB NVMe SSD + 10TB HDD
- **Benefits**: Faster hot data access, extended historical retention
- **Configuration**: Hot storage for recent data, cold storage for archives

**Network**: 10Gbps+ symmetric connection
- **Benefits**: Faster consensus messaging, better block propagation
- **Recommended**: Multiple network interfaces for redundancy

### Hardware Benchmarking

**CPU Performance Test**:
```bash
# CPU benchmark
sysbench cpu --cpu-max-prime=20000 --threads=32 run

# Cryptographic performance
openssl speed -multi 4 aes-256-cbc
openssl speed -multi 4 sha256
```

**Memory Performance Test**:
```bash
# Memory bandwidth
sysbench memory --memory-block-size=1K --memory-total-size=32G run

# Memory latency
mlc --loaded_latency -D0 -R
```

**Storage Performance Test**:
```bash
# Disk I/O benchmark
fio --name=randwrite --ioengine=libaio --iodepth=16 --rw=randwrite --bs=4k --direct=1 --size=32G --numjobs=1 --runtime=60 --group_reporting

# Disk latency
ioping -c 10 /var/lib/savitri
```

## Network Setup

### Network Architecture

**Network Topology**:
```
Internet
    |
    |-- Router/Firewall
        |
        |-- Validator Node (Primary)
        |   |-- 10Gbps primary connection
        |   |-- 1Gbps backup connection
        |
        |-- Monitoring Node
        |   |-- 1Gbps connection
        |
        |-- Backup Storage
            |-- 1Gbps connection
```

### Network Configuration

**Primary Network Interface**:
```bash
# Configure 10Gbps interface
sudo ip link set dev eth0 up
sudo ip addr add 192.168.1.10/24 dev eth0
sudo ip route add default via 192.168.1.1 dev eth0

# Optimize network settings
echo 'net.core.rmem_max = 134217728' | sudo tee -a /etc/sysctl.conf
echo 'net.core.wmem_max = 134217728' | sudo tee -a /etc/sysctl.conf
echo 'net.ipv4.tcp_rmem = 4096 87380 134217728' | sudo tee -a /etc/sysctl.conf
echo 'net.ipv4.tcp_wmem = 4096 65536 134217728' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

**Backup Network Interface**:
```bash
# Configure backup interface
sudo ip link set dev eth1 up
sudo ip addr add 192.168.2.10/24 dev eth1

# Add backup route
sudo ip route add 192.168.2.0/24 via 192.168.2.1 dev eth1
```

### Network Security

**Firewall Configuration**:
```bash
# Configure UFW firewall
sudo ufw --force reset
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH (restricted)
sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp

# Allow P2P port
sudo ufw allow 30333/tcp

# Allow RPC (restricted)
sudo ufw allow from 192.168.1.0/24 to any port 8545 proto tcp

# Enable firewall
sudo ufw enable
```

**DDoS Protection**:
```bash
# Install fail2ban
sudo apt install fail2ban

# Configure fail2ban for Savitri
sudo tee /etc/fail2ban/jail.local << 'EOF'
[savitri-ddos]
enabled = true
port = 30333
filter = savitri-ddos
logpath = /var/log/savitri/node.log
maxretry = 100
bantime = 3600
findtime = 60
EOF

sudo systemctl restart fail2ban
```

## Key Management

### Key Generation

**Generate Validator Key**:
```bash
# Create secure key directory
sudo mkdir -p /etc/savitri/keys
sudo chmod 700 /etc/savitri/keys

# Generate new validator key
savitri-node key generate \
  --scheme Ed25519 \
  --output-file /etc/savitri/keys/validator-key.json \
  --password-file /etc/savitri/keys/validator-password.txt

# Set secure permissions
sudo chmod 600 /etc/savitri/keys/validator-key.json
sudo chmod 600 /etc/savitri/keys/validator-password.txt
```

**Key Backup**:
```bash
# Create backup directory
sudo mkdir -p /backup/savitri/keys

# Backup keys
sudo cp /etc/savitri/keys/validator-key.json /backup/savitri/keys/
sudo cp /etc/savitri/keys/validator-password.txt /backup/savitri/keys/

# Encrypt backup
sudo gpg --symmetric --cipher-algo AES256 --output /backup/savitri/keys/validator-key.json.gpg /etc/savitri/keys/validator-key.json
sudo gpg --symmetric --cipher-algo AES256 --output /backup/savitri/keys/validator-password.txt.gpg /etc/savitri/keys/validator-password.txt

# Remove unencrypted backup
sudo rm /backup/savitri/keys/validator-key.json
sudo rm /backup/savitri/keys/validator-password.txt
```

### Hardware Security Module (HSM)

**HSM Configuration**:
```bash
# Install HSM software
sudo apt install opensc-pkcs11

# Configure HSM
sudo tee /etc/savitri/hsm.conf << 'EOF'
[hsm]
library = /usr/lib/x86_64-linux-gnu/opensc-pkcs11.so
slot = 0
pin = 123456
EOF

# Test HSM connection
pkcs11-tool --module /usr/lib/x86_64-linux-gnu/opensc-pkcs11.so --list-tokens
```

**HSM Key Storage**:
```bash
# Generate key in HSM
savitri-node key generate \
  --scheme Ed25519 \
  --key-store hsm \
  --hsm-slot 0 \
  --hsm-pin 123456 \
  --output-file /etc/savitri/keys/hsm-key-reference.json
```

## Software Installation

### System Preparation

**Update System**:
```bash
# Update package lists
sudo apt update

# Upgrade packages
sudo apt upgrade -y

# Install required packages
sudo apt install -y build-essential pkg-config libssl-dev git curl wget
```

**Create User Account**:
```bash
# Create savitri user
sudo useradd -m -s /bin/bash savitri

# Add to required groups
sudo usermod -aG sudo,docker savitri

# Switch to savitri user
sudo su - savitri
```

### Rust Installation

**Install Rust**:
```bash
# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"

# Verify installation
rustc --version
cargo --version
```

**Install Nightly Toolchain**:
```bash
# Install nightly for latest features
rustup toolchain install nightly
rustup default nightly

# Install required components
rustup component add rustfmt clippy
```

### Build Savitri Node

**Clone Repository**:
```bash
# Clone repository
git clone https://github.com/savitri-network/savitri-node.git
cd savitri-node

# Checkout specific version
git checkout v1.0.0
```

**Build from Source**:
```bash
# Build release version
cargo build --release --features validator

# Run tests
cargo test --release --features validator

# Install binary
sudo cp target/release/savitri-node /usr/local/bin/
sudo chmod +x /usr/local/bin/savitri-node
```

## Configuration

### Validator Configuration

**Create Configuration File**:
```toml
# /etc/savitri/validator.toml

[node]
# Node identity
name = "savitri-validator-01"
data_dir = "/var/lib/savitri"
log_level = "info"

# Network configuration
network = "mainnet"
bootnodes = [
  "/ip4/104.131.131.82/tcp/30333/p2p/12D3KooWJvyP3UJ6ApXPArq9CWKijKp7N2QzMKK4bL2X9G1XQz1",
  "/ip4/104.131.131.83/tcp/30333/p2p/12D3KooWJvyP3UJ6ApXPArq9CWKijKp7N2QzMKK4bL2X9G1XQz2",
]

# Validator configuration
[validator]
enable = true
key_file = "/etc/savitri/keys/validator-key.json"
password_file = "/etc/savitri/keys/validator-password.txt"
bond_amount = "1000000000000000"  # 1M tokens

# Consensus configuration
[consensus]
slot_duration = 6000  # 6 seconds
epoch_duration = 432000  # 72 slots (12 minutes)
leader_rotation = "round-robin"

# Slashing protection
[slashing]
enable = true
history_length = 10000
evidence_storage = "database"

# RPC configuration
[rpc]
enable = true
external = false  # Keep RPC internal for security
port = 8545
hosts = ["127.0.0.1"]

# WebSocket configuration
[websocket]
enable = true
external = false
port = 8546
hosts = ["127.0.0.1"]

# Database configuration
[database]
cache_size = "8GB"
max_open_files = 1000
write_buffer_size = "256MB"
compression = "zstd"

# Monitoring configuration
[monitoring]
prometheus_enable = true
prometheus_port = 9615
metrics_update_interval = 5000
```

### Systemd Service

**Create Service File**:
```ini
# /etc/systemd/system/savitri-validator.service
[Unit]
Description=Savitri Network Validator Node
After=network.target
Wants=network-online.target

[Service]
Type=simple
User=savitri
Group=savitri
ExecStart=/usr/local/bin/savitri-node \
  --config /etc/savitri/validator.toml \
  --validator \
  --base-path /var/lib/savitri
Restart=always
RestartSec=10
StartLimitInterval=60
StartLimitBurst=3

# Security settings
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/savitri /var/log/savitri

# Resource limits
LimitNOFILE=65536
LimitNPROC=4096

# Environment variables
Environment=RUST_LOG=info
Environment=RUST_BACKTRACE=1

[Install]
WantedBy=multi-user.target
```

**Enable and Start Service**:
```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable service
sudo systemctl enable savitri-validator

# Start service
sudo systemctl start savitri-validator

# Check status
sudo systemctl status savitri-validator
```

## Bond Management

### Initial Bond Setup

**Create Bond Transaction**:
```bash
# Create bond transaction
savitri-node \
  --config /etc/savitri/validator.toml \
  validator bond \
  --amount 1000000000000000 \
  --key-file /etc/savitri/keys/validator-key.json \
  --password-file /etc/savitri/keys/validator-password.txt \
  --output bond-transaction.json
```

**Sign and Submit Bond**:
```bash
# Sign transaction
savitri-node \
  transaction sign \
  --input bond-transaction.json \
  --key-file /etc/savitri/keys/validator-key.json \
  --password-file /etc/savitri/keys/validator-password.txt \
  --output signed-bond-transaction.json

# Submit transaction
savitri-node \
  transaction submit \
  --input signed-bond-transaction.json
```

### Bond Management Operations

**Increase Bond**:
```bash
# Increase bond amount
savitri-node \
  --config /etc/savitri/validator.toml \
  validator bond-increase \
  --additional-amount 500000000000000 \
  --key-file /etc/savitri/keys/validator-key.json \
  --password-file /etc/savitri/keys/validator-password.txt
```

**Unbond (Withdraw)**:
```bash
# Initiate unbonding
savitri-node \
  --config /etc/savitri/validator.toml \
  validator unbond \
  --amount 500000000000000 \
  --key-file /etc/savitri/keys/validator-key.json \
  --password-file /etc/savitri/keys/validator-password.txt

# Withdraw after unbonding period
savitri-node \
  --config /etc/savitri/validator.toml \
  validator withdraw \
  --amount 500000000000000 \
  --destination 0x1234567890123456789012345678901234567890 \
  --key-file /etc/savitri/keys/validator-key.json \
  --password-file /etc/savitri/keys/validator-password.txt
```

## Monitoring and Alerting

### Prometheus Monitoring

**Prometheus Configuration**:
```yaml
# /etc/prometheus/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'savitri-validator'
    static_configs:
      - targets: ['localhost:9615']
    scrape_interval: 5s
    metrics_path: /metrics
```

**Key Metrics**:
```bash
# Block production rate
savitri_block_production_total

# Consensus participation
savitri_consensus_participation_rate

# Validator uptime
savitri_validator_uptime_seconds

# Bond amount
savitri_validator_bond_amount

# Slashing events
savitri_validator_slashing_events_total
```

### Grafana Dashboard

**Dashboard Configuration**:
```json
{
  "dashboard": {
    "title": "Savitri Validator Monitoring",
    "panels": [
      {
        "title": "Block Production Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(savitri_block_production_total[5m])",
            "legendFormat": "Blocks/sec"
          }
        ],
        "yAxes": [
          {
            "label": "Blocks per Second"
          }
        ]
      },
      {
        "title": "Consensus Participation",
        "type": "stat",
        "targets": [
          {
            "expr": "savitri_consensus_participation_rate",
            "legendFormat": "Participation Rate"
          }
        ]
      },
      {
        "title": "Validator Uptime",
        "type": "stat",
        "targets": [
          {
            "expr": "savitri_validator_uptime_seconds",
            "legendFormat": "Uptime"
          }
        ]
      },
      {
        "title": "Bond Amount",
        "type": "stat",
        "targets": [
          {
            "expr": "savitri_validator_bond_amount",
            "legendFormat": "Bond Amount"
          }
        ]
      }
    ]
  }
}
```

### Alerting Rules

**Prometheus Alert Rules**:
```yaml
# /etc/prometheus/validator-alerts.yml
groups:
  - name: validator-alerts
    rules:
      - alert: ValidatorDown
        expr: up{job="savitri-validator"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Validator node is down"
          description: "Validator node {{ $labels.instance }} has been down for more than 1 minute"

      - alert: LowParticipation
        expr: savitri_consensus_participation_rate < 0.95
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Low consensus participation"
          description: "Validator participation rate is {{ $value }}%"

      - alert: SlashingEvent
        expr: increase(savitri_validator_slashing_events_total[5m]) > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "Slashing event detected"
          description: "Validator has been slashed"
```

## Security Best Practices

### Key Security

**Multi-Signature Protection**:
```bash
# Generate multiple keys
savitri-node key generate --scheme Ed25519 --output-file key1.json
savitri-node key generate --scheme Ed25519 --output-file key2.json
savitri-node key generate --scheme Ed25519 --output-file key3.json

# Create multi-sig configuration
savitri-node multi-sig create \
  --keys key1.json,key2.json,key3.json \
  --threshold 2 \
  --output-file multi-sig-config.json
```

**Hardware Security Module**:
```bash
# Use HSM for key storage
savitri-node \
  --key-store hsm \
  --hsm-slot 0 \
  --hsm-pin-file /etc/savitri/hsm-pin.txt
```

### Network Security

**VPN Configuration**:
```bash
# Install WireGuard
sudo apt install wireguard

# Generate keys
wg genkey | sudo tee /etc/wireguard/private.key
sudo chmod 600 /etc/wireguard/private.key
sudo cat /etc/wireguard/private.key | wg pubkey | sudo tee /etc/wireguard/public.key

# Configure WireGuard
sudo tee /etc/wireguard/wg0.conf << 'EOF'
[Interface]
PrivateKey = $(cat /etc/wireguard/private.key)
Address = 10.0.0.1/24
DNS = 8.8.8.8

[Peer]
PublicKey = PEER_PUBLIC_KEY
Endpoint = peer.example.com:51820
AllowedIPs = 10.0.0.2/32
EOF

# Enable WireGuard
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```

### System Security

**AppArmor Profile**:
```bash
# Create AppArmor profile
sudo tee /etc/apparmor.d/usr.local.bin.savitri-node << 'EOF'
#include <tunables/global>

/usr/local/bin/savitri-node {
  #include <abstractions/base>
  #include <abstractions/nameservice>
  
  /var/lib/savitri/** rw,
  /etc/savitri/** r,
  /var/log/savitri/** w,
  
  network inet stream,
  network inet dgram,
  
  deny /proc/sys/** w,
  deny /sys/** w,
}
EOF

# Load profile
sudo apparmor_parser -r /etc/apparmor.d/usr.local.bin.savitri-node
sudo apparmor_parser -r /etc/apparmor.d/usr.local.bin.savitri-node
```

## Troubleshooting

### Common Issues

**Validator Not Producing Blocks**:
```bash
# Check validator status
curl -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"validator_status","params":[],"id":1}'

# Check bond status
curl -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"validator_bondStatus","params":["0x..."],"id":1}'

# Check consensus participation
curl -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"validator_participation","params":[],"id":1}'
```

**High Memory Usage**:
```bash
# Check memory usage
ps aux | grep savitri-node
free -h

# Check RocksDB cache
curl -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"database_stats","params":[],"id":1}'

# Reduce cache size if needed
# Edit /etc/savitri/validator.toml
# [database]
# cache_size = "4GB"
```

**Network Connectivity Issues**:
```bash
# Check peer connections
curl -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"system_peers","params":[],"id":1}'

# Check network latency
ping -c 5 104.131.131.82

# Check bandwidth usage
iftop -i eth0
```

### Performance Optimization

**Database Optimization**:
```bash
# RocksDB tuning
savitri-node \
  --db-cache-size 16GB \
  --db-write-buffer-size 512MB \
  --db-max-background-compactions 4 \
  --db-max-background-flushes 2
```

**Consensus Optimization**:
```bash
# Consensus tuning
savitri-node \
  --consensus-cache-size 1GB \
  --consensus-threads 8 \
  --consensus-batch-size 100
```

## Maintenance

### Regular Maintenance Tasks

**Daily Tasks**:
```bash
#!/bin/bash
# daily-maintenance.sh

# Check validator status
systemctl status savitri-validator

# Check bond status
curl -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"validator_bondStatus","params":["0x..."],"id":1}'

# Check logs for errors
grep -i error /var/log/savitri/validator.log | tail -10

# Check system resources
free -h
df -h /var/lib/savitri
```

**Weekly Tasks**:
```bash
#!/bin/bash
# weekly-maintenance.sh

# Update software
git pull origin main
cargo build --release
sudo systemctl restart savitri-validator

# Backup configuration
sudo cp -r /etc/savitri /backup/savitri-config-$(date +%Y%m%d)

# Check performance metrics
curl -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"validator_performance","params":[],"id":1}'
```

**Monthly Tasks**:
```bash
#!/bin/bash
# monthly-maintenance.sh

# Full backup
sudo tar -czf /backup/savitri-full-$(date +%Y%m%d).tar.gz /var/lib/savitri /etc/savitri

# Security audit
sudo apparmor_status
sudo ufw status verbose

# Performance review
iostat -x 1 5
sar -u 1 5
```

### Backup and Recovery

**Automated Backup**:
```bash
#!/bin/bash
# backup-validator.sh

BACKUP_DIR="/backup/savitri"
DATE=$(date +%Y%m%d_%H%M%S)

# Create backup directory
mkdir -p "$BACKUP_DIR"

# Backup node data
sudo tar -czf "$BACKUP_DIR/validator-data-$DATE.tar.gz" /var/lib/savitri

# Backup configuration
sudo cp -r /etc/savitri "$BACKUP_DIR/config-$DATE"

# Backup keys (encrypted)
sudo gpg --symmetric --cipher-algo AES256 \
  --output "$BACKUP_DIR/keys-$DATE.gpg" \
  /etc/savitri/keys/validator-key.json

# Cleanup old backups (keep 7 days)
find "$BACKUP_DIR" -name "*.tar.gz" -mtime +7 -delete
find "$BACKUP_DIR" -name "*.gpg" -mtime +7 -delete

echo "Backup completed: $BACKUP_DIR/validator-data-$DATE.tar.gz"
```

**Recovery Procedure**:
```bash
#!/bin/bash
# recover-validator.sh

BACKUP_FILE=$1

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: $0 <backup-file>"
    exit 1
fi

# Stop validator
sudo systemctl stop savitri-validator

# Backup current data
sudo mv /var/lib/savitri /var/lib/savitri.bak

# Restore from backup
sudo tar -xzf "$BACKUP_FILE" -C /

# Restore configuration
sudo cp -r /backup/savitri/config-* /etc/savitri/

# Set permissions
sudo chown -R savitri:savitri /var/lib/savitri
sudo chmod 600 /etc/savitri/keys/*

# Start validator
sudo systemctl start savitri-validator

# Verify recovery
systemctl status savitri-validator
```

This comprehensive validator setup guide provides everything needed to successfully operate a Savitri Network validator node with proper security, monitoring, and maintenance procedures.
