# Troubleshooting Guide

## Overview

This guide provides comprehensive troubleshooting procedures for common issues encountered when running and maintaining Savitri Network nodes. It covers diagnostic techniques, problem resolution, and preventive measures for all node types.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Diagnostic Tools](#diagnostic-tools)
- [Common Issues](#common-issues)
- [Node-Specific Troubleshooting](#node-specific-troubleshooting)
- [Network Issues](#network-issues)
- [Storage Issues](#storage-issues)
- [Performance Issues](#performance-issues)
- [Security Issues](#security-issues)
- [Recovery Procedures](#recovery-procedures)
- [Preventive Maintenance](#preventive-maintenance)

## Prerequisites

### Required Tools

```bash
# System monitoring tools
sudo apt install -y \
    htop \
    iotop \
    nethogs \
    strace \
    lsof \
    netstat \
    tcpdump \
    wireshark

# Debugging tools
sudo apt install -y \
    gdb \
    valgrind \
    perf \
    massif-visualizer

# Log analysis tools
sudo apt install -y \
    jq \
    awk \
    sed \
    grep \
    less
```

### Debug Configuration

**Enable Debug Logging:**
```bash
# Set RUST_LOG environment variable
export RUST_LOG=debug,savitri=trace

# Enable backtraces
export RUST_BACKTRACE=1

# Full backtraces for detailed debugging
export RUST_BACKTRACE=full
```

**Node Configuration for Debugging:**
```toml
# node.toml
[debug]
enable_profiling = true
enable_tracing = true
log_level = "trace"
metrics_interval_seconds = 10
panic_on_errors = false

[debug.profiling]
cpu_profiling = true
memory_profiling = true
block_profiling = true
```

## Diagnostic Tools

### System Diagnostics

**System Health Check:**
```bash
#!/bin/bash
# system-health-check.sh

echo "=== System Health Check ==="
echo "Time: $(date)"
echo "Hostname: $(hostname)"
echo "Uptime: $(uptime -p)"
echo ""

echo "=== CPU Information ==="
lscpu | grep -E "(Model name|CPU\(s\)|Thread|Core)"
echo ""

echo "=== Memory Information ==="
free -h
echo ""

echo "=== Disk Information ==="
df -h
echo ""

echo "=== Network Information ==="
ip addr show | grep -E "(inet|UP|DOWN)"
echo ""

echo "=== Process Information ==="
ps aux | grep savitri | head -10
echo ""

echo "=== Service Status ==="
systemctl status savitri-node --no-pager
echo ""
```

**Resource Usage Monitor:**
```bash
#!/bin/bash
# resource-monitor.sh

NODE_PID=$(pgrep savitri-node)

if [ -z "$NODE_PID" ]; then
    echo "Savitri node not running"
    exit 1
fi

echo "=== Resource Usage for PID $NODE_PID ==="
echo "CPU: $(ps -p $NODE_PID -o %cpu --no-headers)%"
echo "Memory: $(ps -p $NODE_PID -o %mem --no-headers)%"
echo "Threads: $(ps -p $NODE_PID -o nlwp --no-headers)"
echo "FD Count: $(lsof -p $NODE_PID | wc -l)"
echo ""

echo "=== I/O Statistics ==="
iotop -p $NODE_PID -b -n 1 2>/dev/null || echo "iotop not available"
echo ""

echo "=== Network Connections ==="
netstat -p | grep $NODE_PID | head -10
echo ""
```

### Blockchain Diagnostics

**Node Status Check:**
```rust
// diagnostic.rs
use std::process::Command;

pub struct NodeDiagnostics {
    pub node_id: String,
    pub block_height: u64,
    pub peer_count: u64,
    pub sync_status: SyncStatus,
    pub consensus_status: ConsensusStatus,
    pub mempool_size: u64,
}

impl NodeDiagnostics {
    pub async fn collect() -> Result<Self, DiagnosticError> {
        let node_id = Self::get_node_id().await?;
        let block_height = Self::get_block_height().await?;
        let peer_count = Self::get_peer_count().await?;
        let sync_status = Self::get_sync_status().await?;
        let consensus_status = Self::get_consensus_status().await?;
        let mempool_size = Self::get_mempool_size().await?;
        
        Ok(Self {
            node_id,
            block_height,
            peer_count,
            sync_status,
            consensus_status,
            mempool_size,
        })
    }
    
    async fn get_node_id() -> Result<String, DiagnosticError> {
        let output = Command::new("curl")
            .args(&["-s", "http://localhost:30333/node_id"])
            .output()
            .await?;
        
        Ok(String::from_utf8(output.stdout)?)
    }
    
    // Similar methods for other diagnostics...
}
```

**Consensus Health Check:**
```bash
#!/bin/bash
# consensus-health.sh

echo "=== Consensus Health Check ==="

# Check if node is in committee
COMMITTEE_STATUS=$(curl -s http://localhost:30333/consensus/status | jq -r '.in_committee')

if [ "$COMMITTEE_STATUS" = "true" ]; then
    echo "✓ Node is in consensus committee"
    
    # Get current slot
    CURRENT_SLOT=$(curl -s http://localhost:30333/consensus/slot | jq -r '.current_slot')
    echo "Current slot: $CURRENT_SLOT"
    
    # Check last block produced
    LAST_BLOCK=$(curl -s http://localhost:30333/consensus/last_block | jq -r '.height')
    echo "Last block produced: $LAST_BLOCK"
    
    # Check participation rate
    PARTICIPATION=$(curl -s http://localhost:30333/consensus/participation | jq -r '.participation_rate')
    echo "Participation rate: $PARTIPATION%"
    
    if (( $(echo "$PARTIPATION < 95" | bc -l) )); then
        echo "⚠️  Low participation rate detected"
    fi
else
    echo "ℹ️  Node is not in consensus committee"
fi

echo ""
```

## Common Issues

### Node Startup Issues

**Issue: Node fails to start**
```bash
# Check configuration file
./savitri-node --config node.toml --check-config

# Validate genesis file
./savitri-node --validate-genesis genesis.json

# Check permissions
ls -la /var/lib/savitri/
ls -la /etc/savitri/

# Check for port conflicts
netstat -tlnp | grep :30333
```

**Issue: Configuration errors**
```bash
# Validate TOML configuration
python3 -c "import tomllib; tomllib.load(open('node.toml'))"

# Check required fields
grep -E "^(network|storage|consensus)" node.toml

# Validate network settings
curl -s http://localhost:30333/config | jq .
```

### Sync Issues

**Issue: Node not syncing**
```rust
// sync-diagnostics.rs
pub struct SyncDiagnostics {
    pub current_height: u64,
    pub target_height: u64,
    pub sync_speed: f64,
    pub peer_count: u64,
    pub sync_status: SyncStatus,
}

impl SyncDiagnostics {
    pub async fn diagnose_sync_issues() -> Vec<SyncIssue> {
        let mut issues = Vec::new();
        
        let diagnostics = Self::collect().await?;
        
        // Check if stuck
        if diagnostics.sync_speed < 0.1 {
            issues.push(SyncIssue::SlowSync);
        }
        
        // Check peer connectivity
        if diagnostics.peer_count < 3 {
            issues.push(SyncIssue::InsufficientPeers);
        }
        
        // Check if far behind
        let height_gap = diagnostics.target_height - diagnostics.current_height;
        if height_gap > 1000 {
            issues.push(SyncIssue::FarBehind);
        }
        
        issues
    }
}
```

**Sync Recovery Commands:**
```bash
# Reset sync state
./savitri-node --reset-sync-state

# Force resync from trusted checkpoint
./savitri-node --force-resync --checkpoint-height 1000000

# Clear bad blocks
./savitri-node --clear-bad-blocks

# Rebuild index
./savitri-node --rebuild-index
```

### Memory Issues

**Issue: High memory usage**
```bash
# Check memory usage
ps aux | grep savitri-node | awk '{print $4, $11}'

# Monitor memory over time
watch -n 5 'ps aux | grep savitri-node | head -5'

# Check for memory leaks
valgrind --tool=massif ./savitri-node

# Analyze heap usage
heaptrack ./savitri-node
```

**Memory Optimization:**
```rust
// memory-config.rs
pub struct MemoryConfig {
    pub max_cache_size_mb: u64,
    pub gc_threshold_percent: f64,
    pub memory_limit_mb: u64,
    pub enable_memory_profiling: bool,
}

impl Default for MemoryConfig {
    fn default() -> Self {
        Self {
            max_cache_size_mb: 1024,
            gc_threshold_percent: 80.0,
            memory_limit_mb: 4096,
            enable_memory_profiling: false,
        }
    }
}
```

### Network Issues

**Issue: Peer connection problems**
```bash
# Check peer connectivity
curl -s http://localhost:30333/peers | jq '.peers | length'

# Test network connectivity
ping -c 3 bootstrap-node.savitri.network

# Check firewall rules
sudo ufw status
sudo iptables -L

# Monitor network traffic
tcpdump -i eth0 port 30333
```

**Network Recovery:**
```bash
# Reset peer connections
curl -X POST http://localhost:30333/peers/reset

# Add bootstrap nodes manually
curl -X POST http://localhost:30333/peers/add \
  -H "Content-Type: application/json" \
  -d '{"address": "/ip4/1.2.3.4/tcp/30333"}'

# Check DNS resolution
nslookup bootstrap-node.savitri.network
```

## Node-Specific Troubleshooting

### Full Node Issues

**Issue: Block validation failures**
```rust
// validation-diagnostics.rs
pub struct ValidationDiagnostics {
    pub failed_blocks: u64,
    pub validation_errors: Vec<ValidationError>,
    pub last_valid_block: u64,
    pub validation_speed: f64,
}

impl ValidationDiagnostics {
    pub async fn diagnose_validation_issues() -> Vec<ValidationIssue> {
        let mut issues = Vec::new();
        
        // Check for consistent validation failures
        if self.failed_blocks > 10 {
            issues.push(ValidationIssue::ConsistentFailures);
        }
        
        // Check validation speed
        if self.validation_speed < 1.0 {
            issues.push(ValidationIssue::SlowValidation);
        }
        
        // Analyze error patterns
        for error in &self.validation_errors {
            match error.error_type {
                ErrorType::Signature => issues.push(ValidationIssue::SignatureErrors),
                ErrorType::StateRoot => issues.push(ValidationIssue::StateRootMismatch),
                ErrorType::Timestamp => issues.push(ValidationIssue::TimestampInvalid),
            }
        }
        
        issues
    }
}
```

**Full Node Recovery:**
```bash
# Reset validation cache
./savitri-node --reset-validation-cache

# Rebuild state from blocks
./savitri-node --rebuild-state

# Clear corrupted blocks
./savitri-node --clear-corrupted-blocks

# Verify database integrity
./savitri-node --verify-database
```

### Validator Node Issues

**Issue: Slashing events**
```rust
// slashing-diagnostics.rs
pub struct SlashingDiagnostics {
    pub slashing_events: Vec<SlashingEvent>,
    pub bond_status: BondStatus,
    pub participation_rate: f64,
    pub last_block_produced: u64,
}

impl SlashingDiagnostics {
    pub async fn diagnose_slashing_risk() -> Vec<SlashingRisk> {
        let mut risks = Vec::new();
        
        // Check participation rate
        if self.participation_rate < 95.0 {
            risks.push(SlashingRisk::LowParticipation);
        }
        
        // Check if missing blocks
        let current_height = get_current_height().await?;
        if current_height - self.last_block_produced > 100 {
            risks.push(SlashingRisk::MissedBlocks);
        }
        
        // Analyze past slashing events
        for event in &self.slashing_events {
            match event.reason {
                SlashingReason::Equivocation => risks.push(SlashingRisk::EquivocationRisk),
                SlashingReason::Downtime => risks.push(SlashingRisk::DowntimeRisk),
            }
        }
        
        risks
    }
}
```

**Validator Recovery:**
```bash
# Check bond status
curl -s http://localhost:30333/validator/bond | jq .

# Report slashing evidence
curl -X POST http://localhost:30333/validator/report-evidence \
  -H "Content-Type: application/json" \
  -d '{"evidence": "..."}'

# Withdraw bond (if necessary)
./savitri-node --withdraw-bond --amount 1000000

# Re-register as validator
./savitri-node --register-validator --bond 1000000
```

### Archive Node Issues

**Issue: Query performance problems**
```rust
// query-diagnostics.rs
pub struct QueryDiagnostics {
    pub average_query_time_ms: f64,
    pub slow_queries: Vec<SlowQuery>,
    pub cache_hit_rate: f64,
    pub index_usage: HashMap<String, f64>,
}

impl QueryDiagnostics {
    pub async fn diagnose_query_issues() -> Vec<QueryIssue> {
        let mut issues = Vec::new();
        
        // Check query performance
        if self.average_query_time_ms > 1000.0 {
            issues.push(QueryIssue::SlowQueries);
        }
        
        // Check cache efficiency
        if self.cache_hit_rate < 80.0 {
            issues.push(QueryIssue::LowCacheHitRate);
        }
        
        // Analyze slow queries
        for query in &self.slow_queries {
            if query.execution_time_ms > 5000.0 {
                issues.push(QueryIssue::VerySlowQuery(query.clone()));
            }
        }
        
        // Check index usage
        for (index, usage) in &self.index_usage {
            if *usage < 50.0 {
                issues.push(QueryIssue::UnusedIndex(index.clone()));
            }
        }
        
        issues
    }
}
```

**Archive Node Recovery:**
```bash
# Rebuild query indexes
./savitri-node --rebuild-indexes

# Clear query cache
./savitri-node --clear-query-cache

# Optimize database
./savitri-node --optimize-database

# Compact storage
./savitri-node --compact-storage
```

### RPC Node Issues

**Issue: API rate limiting**
```rust
// rpc-diagnostics.rs
pub struct RpcDiagnostics {
    pub requests_per_second: f64,
    pub rate_limited_requests: u64,
    pub average_response_time_ms: f64,
    pub active_connections: u64,
}

impl RpcDiagnostics {
    pub async fn diagnose_rpc_issues() -> Vec<RpcIssue> {
        let mut issues = Vec::new();
        
        // Check request rate
        if self.requests_per_second > 1000.0 {
            issues.push(RpcIssue::HighRequestRate);
        }
        
        // Check rate limiting
        if self.rate_limited_requests > 100 {
            issues.push(RpcIssue::ExcessiveRateLimiting);
        }
        
        // Check response time
        if self.average_response_time_ms > 500.0 {
            issues.push(RpcIssue::SlowResponse);
        }
        
        // Check connection limits
        if self.active_connections > 1000 {
            issues.push(RpcIssue::ConnectionLimit);
        }
        
        issues
    }
}
```

**RPC Node Recovery:**
```bash
# Reset rate limiter
curl -X POST http://localhost:30333/rpc/reset-rate-limiter

# Clear connection cache
curl -X POST http://localhost:30333/rpc/clear-cache

# Adjust rate limits
curl -X POST http://localhost:30333/rpc/update-config \
  -H "Content-Type: application/json" \
  -d '{"rate_limit": 2000, "connection_limit": 2000}'
```

### Light Node Issues

**Issue: Synchronization problems**
```rust
// light-node-diagnostics.rs
pub struct LightNodeDiagnostics {
    pub header_sync_height: u64,
    pub trusted_peers: u64,
    pub sync_status: LightSyncStatus,
    pub battery_usage: f64,
    pub data_usage_mb: u64,
}

impl LightNodeDiagnostics {
    pub async fn diagnose_light_node_issues() -> Vec<LightNodeIssue> {
        let mut issues = Vec::new();
        
        // Check trusted peers
        if self.trusted_peers < 3 {
            issues.push(LightNodeIssue::InsufficientTrustedPeers);
        }
        
        // Check sync progress
        match self.sync_status {
            LightSyncStatus::Stalled => issues.push(LightNodeIssue::SyncStalled),
            LightSyncStatus::Slow => issues.push(LightNodeIssue::SlowSync),
            _ => {}
        }
        
        // Check battery usage (mobile)
        if self.battery_usage > 20.0 {
            issues.push(LightNodeIssue::HighBatteryUsage);
        }
        
        // Check data usage
        if self.data_usage_mb > 1000 {
            issues.push(LightNodeIssue::HighDataUsage);
        }
        
        issues
    }
}
```

**Light Node Recovery:**
```bash
# Reset trusted peers
./savitri-light-node --reset-trusted-peers

# Add trusted bootstrap nodes
./savitri-light-node --add-trusted-peer "/ip4/1.2.3.4/tcp/30333"

# Clear header cache
./savitri-light-node --clear-header-cache

# Optimize for mobile
./savitri-light-node --mobile-mode --low-power
```

## Network Issues

### Connectivity Problems

**Diagnose Network Issues:**
```bash
#!/bin/bash
# network-diagnosis.sh

echo "=== Network Diagnosis ==="

# Check basic connectivity
echo "Checking internet connectivity..."
ping -c 3 8.8.8.8 > /dev/null 2>&1
if [ $? -eq 0 ]; then
    echo "✓ Internet connectivity OK"
else
    echo "✗ Internet connectivity failed"
fi

# Check DNS resolution
echo "Checking DNS resolution..."
nslookup bootstrap-node.savitri.network > /dev/null 2>&1
if [ $? -eq 0 ]; then
    echo "✓ DNS resolution OK"
else
    echo "✗ DNS resolution failed"
fi

# Check port availability
echo "Checking port 30333..."
netstat -tlnp | grep :30333 > /dev/null 2>&1
if [ $? -eq 0 ]; then
    echo "✓ Port 30333 is listening"
else
    echo "✗ Port 30333 is not listening"
fi

# Check firewall
echo "Checking firewall rules..."
sudo ufw status | grep 30333 > /dev/null 2>&1
if [ $? -eq 0 ]; then
    echo "✓ Firewall allows port 30333"
else
    echo "⚠️  Firewall may block port 30333"
fi

echo ""
```

**Network Recovery:**
```bash
# Restart networking
sudo systemctl restart networking

# Flush DNS cache
sudo systemd-resolve --flush-caches

# Reset firewall rules
sudo ufw --force reset
sudo ufw allow 30333
sudo ufw enable

# Restart node
sudo systemctl restart savitri-node
```

### Peer Discovery Issues

**Diagnose Peer Discovery:**
```rust
// peer-discovery-diagnostics.rs
pub struct PeerDiscoveryDiagnostics {
    pub bootstrap_nodes: Vec<String>,
    pub discovered_peers: u64,
    pub connected_peers: u64,
    pub discovery_time_ms: f64,
}

impl PeerDiscoveryDiagnostics {
    pub async fn diagnose_peer_discovery() -> Vec<PeerDiscoveryIssue> {
        let mut issues = Vec::new();
        
        // Check bootstrap nodes
        if self.bootstrap_nodes.is_empty() {
            issues.push(PeerDiscoveryIssue::NoBootstrapNodes);
        }
        
        // Check discovery success
        if self.discovered_peers < 10 {
            issues.push(PeerDiscoveryIssue::LowDiscoveryRate);
        }
        
        // Check connection success
        let connection_rate = self.connected_peers as f64 / self.discovered_peers as f64;
        if connection_rate < 0.5 {
            issues.push(PeerDiscoveryIssue::LowConnectionRate);
        }
        
        // Check discovery speed
        if self.discovery_time_ms > 30000.0 {
            issues.push(PeerDiscoveryIssue::SlowDiscovery);
        }
        
        issues
    }
}
```

**Peer Discovery Recovery:**
```bash
# Reset peer discovery
curl -X POST http://localhost:30333/peers/discovery/reset

# Add bootstrap nodes
curl -X POST http://localhost:30333/peers/bootstrap/add \
  -H "Content-Type: application/json" \
  -d '{"nodes": ["/ip4/1.2.3.4/tcp/30333"]}'

# Force peer discovery
curl -X POST http://localhost:30333/peers/discovery/force

# Clear peer cache
curl -X POST http://localhost:30333/peers/cache/clear
```

## Storage Issues

### Database Corruption

**Diagnose Database Issues:**
```bash
#!/bin/bash
# database-diagnosis.sh

DB_PATH="/var/lib/savitri/db"

echo "=== Database Diagnosis ==="

# Check database file integrity
echo "Checking database integrity..."
if [ -f "$DB_PATH/CURRENT" ]; then
    echo "✓ Database files exist"
    
    # Check file permissions
    ls -la "$DB_PATH" | head -5
    
    # Check disk space
    DISK_USAGE=$(df "$DB_PATH" | tail -1 | awk '{print $5}' | sed 's/%//')
    if [ $DISK_USAGE -gt 90 ]; then
        echo "⚠️  High disk usage: $DISK_USAGE%"
    else
        echo "✓ Disk usage OK: $DISK_USAGE%"
    fi
    
    # Check for lock files
    if [ -f "$DB_PATH/LOCK" ]; then
        echo "⚠️  Database lock file exists"
    else
        echo "✓ No database lock"
    fi
else
    echo "✗ Database files missing"
fi

echo ""
```

**Database Recovery:**
```bash
# Create database backup
cp -r /var/lib/savitri/db /var/lib/savitri/db.backup.$(date +%s)

# Run database repair
./savitri-node --repair-database

# Rebuild from scratch (last resort)
./savitri-node --rebuild-database

# Verify database integrity
./savitri-node --verify-database
```

### Storage Performance Issues

**Diagnose Storage Performance:**
```rust
// storage-diagnostics.rs
pub struct StorageDiagnostics {
    pub read_latency_ms: f64,
    pub write_latency_ms: f64,
    pub read_throughput_mbps: f64,
    pub write_throughput_mbps: f64,
    pub iops: u64,
    pub disk_usage_percent: f64,
}

impl StorageDiagnostics {
    pub async fn diagnose_storage_issues() -> Vec<StorageIssue> {
        let mut issues = Vec::new();
        
        // Check read latency
        if self.read_latency_ms > 10.0 {
            issues.push(StorageIssue::HighReadLatency);
        }
        
        // Check write latency
        if self.write_latency_ms > 50.0 {
            issues.push(StorageIssue::HighWriteLatency);
        }
        
        // Check throughput
        if self.read_throughput_mbps < 100.0 {
            issues.push(StorageIssue::LowReadThroughput);
        }
        
        if self.write_throughput_mbps < 50.0 {
            issues.push(StorageIssue::LowWriteThroughput);
        }
        
        // Check disk space
        if self.disk_usage_percent > 90.0 {
            issues.push(StorageIssue::LowDiskSpace);
        }
        
        issues
    }
}
```

**Storage Optimization:**
```bash
# Optimize database
./savitri-node --optimize-database

# Compact storage
./savitri-node --compact-storage

# Clear old data
./savitri-node --prune-old-data --blocks-to-keep 100000

# Rebuild indexes
./savitri-node --rebuild-indexes

# Adjust cache size
./savitri-node --cache-size 2048
```

## Performance Issues

### CPU Performance

**Diagnose CPU Issues:**
```bash
#!/bin/bash
# cpu-diagnosis.sh

NODE_PID=$(pgrep savitri-node)

if [ -z "$NODE_PID" ]; then
    echo "Savitri node not running"
    exit 1
fi

echo "=== CPU Diagnosis ==="

# Check CPU usage
CPU_USAGE=$(ps -p $NODE_PID -o %cpu --no-headers)
echo "CPU Usage: $CPU_USAGE%"

# Check thread count
THREAD_COUNT=$(ps -p $NODE_PID -o nlwp --no-headers)
echo "Thread Count: $THREAD_COUNT"

# Check CPU affinity
taskset -pc $NODE_PID

# Profile CPU usage
perf top -p $NODE_PID

echo ""
```

**CPU Optimization:**
```rust
// cpu-config.rs
pub struct CpuConfig {
    pub worker_threads: usize,
    pub io_threads: usize,
    pub enable_cpu_profiling: bool,
    pub cpu_affinity: Option<Vec<usize>>,
}

impl Default for CpuConfig {
    fn default() -> Self {
        Self {
            worker_threads: num_cpus::get(),
            io_threads: num_cpus::get() / 2,
            enable_cpu_profiling: false,
            cpu_affinity: None,
        }
    }
}
```

### Memory Performance

**Diagnose Memory Issues:**
```bash
#!/bin/bash
# memory-diagnosis.sh

NODE_PID=$(pgrep savitri-node)

if [ -z "$NODE_PID" ]; then
    echo "Savitri node not running"
    exit 1
fi

echo "=== Memory Diagnosis ==="

# Check memory usage
MEMORY_USAGE=$(ps -p $NODE_PID -o %mem --no-headers)
echo "Memory Usage: $MEMORY_USAGE%"

# Check resident set size
RSS=$(ps -p $NODE_PID -o rss --no-headers)
echo "RSS: $RSS KB"

# Check virtual memory size
VSZ=$(ps -p $NODE_PID -o vsz --no-headers)
echo "VSZ: $VSZ KB"

# Check for memory leaks
valgrind --tool=massif --pages-as-heap=yes ./savitri-node

echo ""
```

**Memory Optimization:**
```rust
// memory-optimizer.rs
pub struct MemoryOptimizer {
    pub gc_interval_seconds: u64,
    pub max_heap_size_mb: u64,
    pub enable_memory_pooling: bool,
    pub pool_size_mb: u64,
}

impl MemoryOptimizer {
    pub fn optimize_memory_usage(&self) {
        // Enable memory pooling
        if self.enable_memory_pooling {
            self.enable_memory_pool();
        }
        
        // Set up periodic garbage collection
        self.setup_gc_scheduler();
        
        // Monitor memory usage
        self.start_memory_monitoring();
    }
}
```

## Security Issues

### Authentication Problems

**Diagnose Authentication Issues:**
```rust
// auth-diagnostics.rs
pub struct AuthDiagnostics {
    pub failed_logins: u64,
    pub active_sessions: u64,
    pub last_login_attempt: SystemTime,
    pub security_events: Vec<SecurityEvent>,
}

impl AuthDiagnostics {
    pub async fn diagnose_auth_issues() -> Vec<AuthIssue> {
        let mut issues = Vec::new();
        
        // Check for brute force attacks
        if self.failed_logins > 100 {
            issues.push(AuthIssue::BruteForceAttack);
        }
        
        // Check session management
        if self.active_sessions > 1000 {
            issues.push(AuthIssue::SessionOverflow);
        }
        
        // Analyze security events
        for event in &self.security_events {
            match event.event_type {
                SecurityEventType::SuspiciousLogin => issues.push(AuthIssue::SuspiciousActivity),
                SecurityEventType::PrivilegeEscalation => issues.push(AuthIssue::SecurityBreach),
            }
        }
        
        issues
    }
}
```

**Security Recovery:**
```bash
# Reset authentication
curl -X POST http://localhost:30333/auth/reset

# Block suspicious IPs
curl -X POST http://localhost:30333/auth/block-ip \
  -H "Content-Type: application/json" \
  -d '{"ip": "1.2.3.4", "duration": 3600}'

# Clear sessions
curl -X POST http://localhost:30333/auth/clear-sessions

# Enable additional security
curl -X POST http://localhost:30333/auth/enable-2fa
```

### Network Security

**Diagnose Security Issues:**
```bash
#!/bin/bash
# security-diagnosis.sh

echo "=== Security Diagnosis ==="

# Check for suspicious connections
echo "Checking active connections..."
netstat -an | grep :30333 | grep ESTABLISHED | wc -l

# Check for failed authentication attempts
echo "Checking auth logs..."
grep "authentication failed" /var/log/savitri/auth.log | tail -10

# Check firewall status
echo "Checking firewall..."
sudo ufw status

# Check for unusual processes
echo "Checking processes..."
ps aux | grep savitri | grep -v grep

echo ""
```

## Recovery Procedures

### Emergency Recovery

**Complete Node Recovery:**
```bash
#!/bin/bash
# emergency-recovery.sh

echo "=== Emergency Recovery Procedure ==="

# Stop the node
sudo systemctl stop savitri-node

# Create backup
BACKUP_DIR="/var/lib/savitri/backup/$(date +%s)"
mkdir -p "$BACKUP_DIR"
cp -r /var/lib/savitri/* "$BACKUP_DIR/"

# Clear corrupted data
rm -rf /var/lib/savitri/db/*
rm -rf /var/lib/savitri/cache/*

# Reset configuration
cp /etc/savitri/node.toml.default /etc/savitri/node.toml

# Restart with clean state
sudo systemctl start savitri-node

# Monitor recovery
watch -n 5 'curl -s http://localhost:30333/health | jq .'

echo ""
```

**Partial Recovery:**
```bash
# Reset only specific components
./savitri-node --reset-mempool
./savitri-node --reset-consensus-state
./savitri-node --reset-peer-table

# Recover from specific height
./savitri-node --recover-from-height 1000000

# Import trusted state
./savitri-node --import-state trusted-state.json
```

### Data Recovery

**Database Recovery:**
```bash
# Restore from backup
./savitri-node --restore-from-backup /path/to/backup

# Repair corrupted database
./savitri-node --repair-database --force

# Rebuild missing indexes
./savitri-node --rebuild-missing-indexes

# Verify data integrity
./savitri-node --verify-data-integrity
```

## Preventive Maintenance

### Regular Maintenance Tasks

**Maintenance Script:**
```bash
#!/bin/bash
# maintenance.sh

echo "=== Preventive Maintenance ==="

# Clean old logs
find /var/log/savitri -name "*.log" -mtime +7 -delete

# Compact database
./savitri-node --compact-storage

# Clear cache
./savitri-node --clear-cache

# Update peer list
curl -X POST http://localhost:30333/peers/refresh

# Check disk space
DISK_USAGE=$(df /var/lib/savitri | tail -1 | awk '{print $5}' | sed 's/%//')
if [ $DISK_USAGE -gt 80 ]; then
    echo "Warning: High disk usage ($DISK_USAGE%)"
    ./savitri-node --prune-old-data
fi

# Backup configuration
cp /etc/savitri/node.toml /etc/savitri/node.toml.backup.$(date +%s)

echo "Maintenance completed"
echo ""
```

### Health Monitoring

**Health Check Automation:**
```rust
// health-monitor.rs
pub struct HealthMonitor {
    pub check_interval_seconds: u64,
    pub alert_thresholds: AlertThresholds,
    pub notification_channels: Vec<NotificationChannel>,
}

impl HealthMonitor {
    pub async fn start_monitoring(&self) {
        let mut interval = tokio::time::interval(
            Duration::from_secs(self.check_interval_seconds)
        );
        
        loop {
            interval.tick().await;
            
            let health = self.check_node_health().await;
            
            if !health.is_healthy() {
                self.send_alert(&health).await;
            }
        }
    }
    
    async fn check_node_health(&self) -> NodeHealth {
        NodeHealth {
            cpu_usage: self.get_cpu_usage().await,
            memory_usage: self.get_memory_usage().await,
            disk_usage: self.get_disk_usage().await,
            network_status: self.get_network_status().await,
            blockchain_status: self.get_blockchain_status().await,
        }
    }
}
```

### Performance Tuning

**Performance Optimization:**
```rust
// performance-tuner.rs
pub struct PerformanceTuner {
    pub target_cpu_usage: f64,
    pub target_memory_usage: f64,
    pub target_response_time_ms: f64,
}

impl PerformanceTuner {
    pub async fn auto_tune(&mut self) {
        let current_metrics = self.collect_metrics().await;
        
        // Adjust worker threads based on CPU usage
        if current_metrics.cpu_usage > self.target_cpu_usage {
            self.reduce_worker_threads();
        } else if current_metrics.cpu_usage < self.target_cpu_usage * 0.5 {
            self.increase_worker_threads();
        }
        
        // Adjust cache size based on memory usage
        if current_metrics.memory_usage > self.target_memory_usage {
            self.reduce_cache_size();
        } else if current_metrics.memory_usage < self.target_memory_usage * 0.7 {
            self.increase_cache_size();
        }
        
        // Adjust network parameters
        if current_metrics.average_response_time > self.target_response_time_ms {
            self.optimize_network_settings();
        }
    }
}
```

This comprehensive troubleshooting guide provides systematic approaches to diagnosing and resolving issues across all Savitri Network node types, ensuring reliable operation and quick recovery from problems.
