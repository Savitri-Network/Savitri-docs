# Running a Savitri Network Node

## Overview

This guide provides comprehensive instructions for setting up and running various types of nodes in the Savitri Network. Whether you're running a full node, validator node, archive node, or RPC node, this guide covers the complete process from hardware requirements to ongoing maintenance.

## Technology Choice Rationale

### Why Comprehensive Node Guide

**Problem Statement**: Users need clear, step-by-step instructions for deploying and managing Savitri Network nodes across different use cases and environments.

**Chosen Solution**: Comprehensive operational guide with hardware specifications, software installation, configuration management, and troubleshooting procedures.

**Rationale**:
- **Accessibility**: Lower barrier to entry for node operators
- **Standardization**: Consistent deployment procedures across environments
- **Reliability**: Proven configurations and best practices
- **Scalability**: Support for different node types and use cases
- **Maintainability**: Clear procedures for ongoing operations

**Expected Results**:
- Successful node deployment for all node types
- Consistent operational procedures across the network
- Reduced deployment time and configuration errors
- Better network decentralization through easier node setup
- Improved node operator experience

## Prerequisites

### System Requirements

**Minimum System Requirements**:
- **OS**: Linux (Ubuntu 20.04+), macOS 10.15+, Windows 10+
- **CPU**: 4 cores, 2.5GHz+ (full nodes), 8 cores+ (validators)
- **Memory**: 8GB RAM (full nodes), 16GB+ (validators)
- **Storage**: 500GB SSD (full nodes), 1TB+ (archive nodes)
- **Network**: 100Mbps+ symmetric connection

**Recommended System Requirements**:
- **OS**: Ubuntu 22.04 LTS (recommended)
- **CPU**: 8 cores, 3.0GHz+ (full nodes), 16 cores+ (validators)
- **Memory**: 16GB RAM (full nodes), 32GB+ (validators)
- **Storage**: 1TB NVMe SSD (full nodes), 2TB+ (archive nodes)
- **Network**: 1Gbps+ symmetric connection

### Software Dependencies

**Required Software**:
```bash
# Rust toolchain
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"

# System dependencies (Ubuntu/Debian)
sudo apt update
sudo apt install -y build-essential pkg-config libssl-dev

# System dependencies (macOS)
brew install openssl pkg-config

# System dependencies (Windows)
# Install Visual Studio Build Tools and OpenSSL
```

**Optional Software**:
```bash
# Docker (for containerized deployment)
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Node.js (for monitoring dashboards)
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# PostgreSQL (for analytics nodes)
sudo apt install postgresql postgresql-contrib
```

## Node Types Selection

### Full Node
**Use Case**: General network participation, transaction verification, block validation
**Requirements**: Moderate hardware, standard network connection
**Benefits**: Complete blockchain state, network services, voting rights

### Validator Node
**Use Case**: Block production, consensus participation, earning rewards
**Requirements**: High-performance hardware, stable network, bond requirements
**Benefits**: Block rewards, voting power, network influence

### Archive Node
**Use Case**: Historical data access, analytics, blockchain exploration
**Requirements**: Large storage, high-performance storage systems
**Benefits**: Complete historical data, query services, analytics capabilities

### RPC Node
**Use Case**: Application backend, API services, developer tools
**Requirements**: Moderate hardware, high network bandwidth
**Benefits**: API access, developer services, application support

## Installation

### Source Installation

**Step 1: Clone Repository**
```bash
git clone https://github.com/savitri-network/savitri-node.git
cd savitri-node
```

**Step 2: Build from Source**
```bash
# Install Rust dependencies
cargo build --release

# Run tests to verify installation
cargo test --release

# Install binary (optional)
sudo cp target/release/savitri-node /usr/local/bin/
```

**Step 3: Verify Installation**
```bash
savitri-node --version
savitri-node --help
```

### Binary Installation

**Step 1: Download Binary**
```bash
# Download latest release
wget https://github.com/savitri-network/savitri-node/releases/latest/download/savitri-node-linux-x86_64.tar.gz

# Extract archive
tar -xzf savitri-node-linux-x86_64.tar.gz
cd savitri-node-linux-x86_64
```

**Step 2: Install Binary**
```bash
# Copy to system path
sudo cp savitri-node /usr/local/bin/
sudo chmod +x /usr/local/bin/savitri-node

# Verify installation
savitri-node --version
```

### Docker Installation

**Step 1: Pull Docker Image**
```bash
docker pull savitri-network/savitri-node:latest
```

**Step 2: Create Docker Network**
```bash
docker network create savitri-network
```

**Step 3: Run Node Container**
```bash
docker run -d \
  --name savitri-node \
  --network savitri-network \
  -p 8545:8545 \
  -p 30333:30333 \
  -v /path/to/data:/data \
  savitri-network/savitri-node:latest \
  --base-path /data \
  --rpc-external \
  --ws-external
```

## Configuration

### Basic Configuration

**Create Configuration File**:
```toml
# savitri.toml

[node]
# Node identity
name = "my-savitri-node"
data_dir = "/var/lib/savitri"

# Network configuration
network = "mainnet"
bootnodes = [
  "/ip4/104.131.131.82/tcp/30333/p2p/12D3KooWJvyP3UJ6ApXPArq9CWKijKp7N2QzMKK4bL2X9G1XQz1",
  "/ip4/104.131.131.83/tcp/30333/p2p/12D3KooWJvyP3UJ6ApXPArq9CWKijKp7N2QzMKK4bL2X9G1XQz2",
]

# RPC configuration
[rpc]
enable = true
external = true
port = 8545
hosts = ["127.0.0.1", "::1"]

# WebSocket configuration
[websocket]
enable = true
external = true
port = 8546
hosts = ["127.0.0.1", "::1"]

# Logging configuration
[logging]
level = "info"
file = "/var/log/savitri/node.log"
```

### Validator Configuration

**Validator-Specific Settings**:
```toml
# savitri-validator.toml

[node]
# ... basic node configuration ...

# Validator configuration
[validator]
enable = true
key_file = "/etc/savitri/validator-key.json"
bond_amount = "1000000000000000"  # 1M tokens

# Consensus configuration
[consensus]
slot_duration = 6000  # 6 seconds
epoch_duration = 432000  # 72 slots (12 minutes)

# Slashing protection
[slashing]
enable = true
history_length = 10000
```

### Archive Node Configuration

**Archive-Specific Settings**:
```toml
# savitri-archive.toml

[node]
# ... basic node configuration ...

# Archive configuration
[archive]
enable = true
prune = false
retention_blocks = "all"

# Storage configuration
[storage]
rocksdb_cache_size = "8GB"
rocksdb_max_open_files = 1000

# Query configuration
[query]
enable = true
max_concurrent_queries = 100
query_timeout = 30000  # 30 seconds
```

## Key Management

### Generate New Keys

**Step 1: Generate Validator Key**
```bash
# Generate new validator key
savitri-node key generate \
  --scheme Ed25519 \
  --output-file validator-key.json \
  --password-file validator-password.txt
```

**Step 2: Secure Key Storage**
```bash
# Set appropriate permissions
chmod 600 validator-key.json
chmod 600 validator-password.txt

# Move to secure location
sudo mv validator-key.json /etc/savitri/
sudo mv validator-password.txt /etc/savitri/
```

### Import Existing Keys

**Step 1: Import from Mnemonic**
```bash
# Import from mnemonic phrase
savitri-node key import \
  --mnemonic \
  --scheme Ed25519 \
  --output-file validator-key.json \
  --password-file validator-password.txt
```

**Step 2: Import from Private Key**
```bash
# Import from private key file
savitri-node key import \
  --private-key-file private-key.txt \
  --scheme Ed25519 \
  --output-file validator-key.json \
  --password-file validator-password.txt
```

## Network Setup

### Firewall Configuration

**Ubuntu/Debian**:
```bash
# Configure UFW firewall
sudo ufw allow 22/tcp      # SSH
sudo ufw allow 8545/tcp    # JSON-RPC
sudo ufw allow 8546/tcp    # WebSocket
sudo ufw allow 30333/tcp   # P2P
sudo ufw enable
```

**CentOS/RHEL**:
```bash
# Configure firewalld
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-port=8545/tcp
sudo firewall-cmd --permanent --add-port=8546/tcp
sudo firewall-cmd --permanent --add-port=30333/tcp
sudo firewall-cmd --reload
```

### Network Optimization

**TCP Tuning**:
```bash
# Add to /etc/sysctl.conf
echo "net.core.rmem_max = 134217728" | sudo tee -a /etc/sysctl.conf
echo "net.core.wmem_max = 134217728" | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.tcp_rmem = 4096 87380 134217728" | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.tcp_wmem = 4096 65536 134217728" | sudo tee -a /etc/sysctl.conf

# Apply changes
sudo sysctl -p
```

**File Descriptor Limits**:
```bash
# Add to /etc/security/limits.conf
echo "* soft nofile 65536" | sudo tee -a /etc/security/limits.conf
echo "* hard nofile 65536" | sudo tee -a /etc/security/limits.conf

# Apply changes (requires relogin)
ulimit -n 65536
```

## Running the Node

### Start Full Node

**Command Line**:
```bash
savitri-node \
  --base-path /var/lib/savitri \
  --chain mainnet \
  --rpc-external \
  --ws-external \
  --bootnodes "/ip4/104.131.131.82/tcp/30333/p2p/12D3KooWJvyP3UJ6ApXPArq9CWKijKp7N2QzMKK4bL2X9G1XQz1" \
  --name "my-full-node"
```

**Systemd Service**:
```ini
# /etc/systemd/system/savitri-node.service
[Unit]
Description=Savitri Network Node
After=network.target

[Service]
Type=simple
User=savitri
Group=savitri
ExecStart=/usr/local/bin/savitri-node \
  --base-path /var/lib/savitri \
  --chain mainnet \
  --rpc-external \
  --ws-external
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

**Enable and Start Service**:
```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable service
sudo systemctl enable savitri-node

# Start service
sudo systemctl start savitri-node

# Check status
sudo systemctl status savitri-node
```

### Start Validator Node

**Command Line**:
```bash
savitri-node \
  --base-path /var/lib/savitri \
  --chain mainnet \
  --validator \
  --key-file /etc/savitri/validator-key.json \
  --password-file /etc/savitri/validator-password.txt \
  --bootnodes "/ip4/104.131.131.82/tcp/30333/p2p/12D3KooWJvyP3UJ6ApXPArq9CWKijKp7N2QzMKK4bL2X9G1XQz1" \
  --name "my-validator-node"
```

### Start Archive Node

**Command Line**:
```bash
savitri-node \
  --base-path /var/lib/savitri \
  --chain mainnet \
  --archive \
  --prune=none \
  --rpc-external \
  --ws-external \
  --bootnodes "/ip4/104.131.131.82/tcp/30333/p2p/12D3KooWJvyP3UJ6ApXPArq9CWKijKp7N2QzMKK4bL2X9G1XQz1" \
  --name "my-archive-node"
```

## Monitoring

### Basic Monitoring

**Node Status**:
```bash
# Check node status
curl -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"system_health","params":[],"id":1}'

# Check sync status
curl -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"system_syncState","params":[],"id":1}'

# Check peer count
curl -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"system_peers","params":[],"id":1}'
```

**Log Monitoring**:
```bash
# Follow node logs
tail -f /var/log/savitri/node.log

# Filter for errors
grep -i error /var/log/savitri/node.log

# Filter for consensus messages
grep -i consensus /var/log/savitri/node.log
```

### Advanced Monitoring

**Prometheus Metrics**:
```bash
# Enable Prometheus metrics
savitri-node \
  --prometheus-external \
  --prometheus-port 9615 \
  ... # other flags
```

**Grafana Dashboard**:
```json
{
  "dashboard": {
    "title": "Savitri Node Monitoring",
    "panels": [
      {
        "title": "Block Production",
        "type": "graph",
        "targets": [
          {
            "expr": "savitri_block_production_total",
            "legendFormat": "Blocks Produced"
          }
        ]
      },
      {
        "title": "Peer Connections",
        "type": "graph",
        "targets": [
          {
            "expr": "savitri_peer_count",
            "legendFormat": "Peer Count"
          }
        ]
      }
    ]
  }
}
```

### Alerting

**Systemd Alerts**:
```bash
# Create alert script
cat > /usr/local/bin/savitri-alert.sh << 'EOF'
#!/bin/bash
NODE_STATUS=$(systemctl is-active savitri-node)
if [ "$NODE_STATUS" != "active" ]; then
    echo "Savitri node is not running: $NODE_STATUS" | mail -s "Savitri Node Alert" admin@example.com
fi
EOF

chmod +x /usr/local/bin/savitri-alert.sh

# Add to cron
echo "*/5 * * * * /usr/local/bin/savitri-alert.sh" | crontab -
```

## Maintenance

### Regular Maintenance Tasks

**Daily Tasks**:
```bash
# Check node status
systemctl status savitri-node

# Check disk space
df -h /var/lib/savitri

# Check memory usage
free -h

# Review logs for errors
grep -i error /var/log/savitri/node.log | tail -10
```

**Weekly Tasks**:
```bash
# Update node software
git pull origin main
cargo build --release
sudo systemctl restart savitri-node

# Backup configuration
sudo cp /etc/savitri/savitri.toml /backup/savitri-$(date +%Y%m%d).toml

# Check network connectivity
ping -c 3 8.8.8.8
```

**Monthly Tasks**:
```bash
# Full system backup
sudo tar -czf /backup/savitri-full-$(date +%Y%m%d).tar.gz /var/lib/savitri /etc/savitri

# Security updates
sudo apt update && sudo apt upgrade -y

# Performance tuning review
iostat -x 1 5
```

### Backup and Recovery

**Data Backup**:
```bash
# Create backup script
cat > /usr/local/bin/savitri-backup.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/backup/savitri"
DATE=$(date +%Y%m%d_%H%M%S)

# Create backup directory
mkdir -p "$BACKUP_DIR"

# Backup node data
sudo tar -czf "$BACKUP_DIR/savitri-data-$DATE.tar.gz" /var/lib/savitri

# Backup configuration
sudo cp -r /etc/savitri "$BACKUP_DIR/config-$DATE"

# Cleanup old backups (keep 7 days)
find "$BACKUP_DIR" -name "*.tar.gz" -mtime +7 -delete

echo "Backup completed: $BACKUP_DIR/savitri-data-$DATE.tar.gz"
EOF

chmod +x /usr/local/bin/savitri-backup.sh

# Add to cron (daily at 2 AM)
echo "0 2 * * * /usr/local/bin/savitri-backup.sh" | crontab -
```

**Data Recovery**:
```bash
# Stop node
sudo systemctl stop savitri-node

# Backup current data
sudo mv /var/lib/savitri /var/lib/savitri.bak

# Restore from backup
sudo tar -xzf /backup/savitri/savitri-data-20231201_020000.tar.gz -C /

# Start node
sudo systemctl start savitri-node

# Verify recovery
systemctl status savitri-node
```

## Troubleshooting

### Common Issues

**Node Won't Start**:
```bash
# Check configuration file
savitri-node --config /etc/savitri/savitri.toml --validate

# Check permissions
ls -la /var/lib/savitri
ls -la /etc/savitri

# Check system resources
free -h
df -h
```

**Sync Issues**:
```bash
# Check sync status
curl -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"system_syncState","params":[],"id":1}'

# Check peer connections
curl -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"system_peers","params":[],"id":1}'

# Reset sync state (if needed)
sudo systemctl stop savitri-node
sudo rm -rf /var/lib/savitri/chains/mainnet/db
sudo systemctl start savitri-node
```

**Performance Issues**:
```bash
# Check CPU usage
top -p $(pgrep savitri-node)

# Check memory usage
ps aux | grep savitri-node

# Check disk I/O
iotop -p $(pgrep savitri-node)

# Check network usage
iftop -i eth0
```

### Debug Mode

**Enable Debug Logging**:
```bash
savitri-node \
  --log-level debug \
  --log-file /var/log/savitri/debug.log \
  ... # other flags
```

**Enable Tracing**:
```bash
savitri-node \
  --tracing-targets savitri::consensus \
  --tracing-profiling \
  ... # other flags
```

### Performance Tuning

**Database Optimization**:
```bash
# RocksDB tuning
savitri-node \
  --db-cache-size 8GB \
  --db-write-buffer-size 256MB \
  --db-max-open-files 1000 \
  ... # other flags
```

**Network Optimization**:
```bash
# Network tuning
savitri-node \
  --max-peers 50 \
  --reserved-peers 25 \
  --reserved-only-peers 5 \
  ... # other flags
```

## Security

### Basic Security

**SSH Security**:
```bash
# Disable password authentication
sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config

# Use key-based authentication only
sudo sed -i 's/#PubkeyAuthentication yes/PubkeyAuthentication yes/' /etc/ssh/sshd_config

# Restart SSH service
sudo systemctl restart ssh
```

**Firewall Security**:
```bash
# Only allow necessary ports
sudo ufw deny incoming
sudo ufw allow ssh
sudo ufw allow 8545/tcp  # RPC (if needed)
sudo ufw allow 30333/tcp # P2P
sudo ufw enable
```

### Advanced Security

**Fail2Ban Configuration**:
```bash
# Install fail2ban
sudo apt install fail2ban

# Create Savitri jail
sudo tee /etc/fail2ban/jail.local << 'EOF'
[savitri]
enabled = true
port = 8545,8546,30333
filter = savitri
logpath = /var/log/savitri/node.log
maxretry = 5
bantime = 3600
EOF

# Restart fail2ban
sudo systemctl restart fail2ban
```

**Key Security**:
```bash
# Use hardware security module (HSM)
savitri-node \
  --key-store hsm \
  --hsm-library /usr/lib/libpkcs11.so \
  ... # other flags
```

## Best Practices

### Operational Best Practices

1. **Regular Backups**: Daily automated backups of critical data
2. **Monitoring**: Comprehensive monitoring with alerting
3. **Security Updates**: Regular system and software updates
4. **Performance Tuning**: Optimize for specific use cases
5. **Documentation**: Maintain configuration and operational documentation

### Network Best Practices

1. **Peer Diversity**: Connect to diverse set of peers
2. **Bandwidth Management**: Monitor and limit bandwidth usage
3. **Connection Limits**: Set appropriate connection limits
4. **Network Security**: Use VPNs for sensitive operations
5. **Redundancy**: Multiple network connections for critical nodes

### Security Best Practices

1. **Key Management**: Secure storage and backup of keys
2. **Access Control**: Limit access to node operations
3. **Audit Logging**: Comprehensive logging of all operations
4. **Regular Audits**: Periodic security audits
5. **Incident Response**: Clear incident response procedures

This comprehensive guide provides everything needed to successfully run and maintain Savitri Network nodes across different use cases and environments.
