# RPC Node Specification

## Overview

A Savitri Network RPC node is a specialized full node optimized for providing Remote Procedure Call (RPC) services to applications, wallets, and external systems. RPC nodes handle high volumes of API requests, provide data access services, and act as gateways between the blockchain network and external applications.

## Technology Choice Rationale

### Why RPC Node Architecture

**Problem Statement**: Blockchain applications need reliable, high-performance access to blockchain data and functionality, but full nodes may not be optimized for serving external API requests at scale.

**Chosen Solution**: Dedicated RPC node architecture with optimized request handling, caching, rate limiting, and comprehensive API support.

**Rationale**:
- **Performance**: Optimized for high-volume API request handling
- **Reliability**: Dedicated resources ensure consistent service availability
- **Scalability**: Horizontal scaling capabilities for growing demand
- **Security**: Isolated from consensus operations with access controls
- **Monitoring**: Comprehensive API usage and performance metrics

**Expected Results**:
- High-performance API service for applications
- Reliable blockchain data access for external systems
- Scalable architecture for growing user demand
- Secure and controlled access to blockchain functionality
- Comprehensive monitoring and analytics capabilities

## Hardware Requirements

### Minimum Specifications

**CPU**: 12 cores, 3.0GHz+ (Intel/AMD x64)
- **Rationale**: API request processing, data serialization, and concurrent connections
- **Expected Performance**: 10,000+ RPC requests per second

**Memory**: 32GB RAM
- **Rationale**: Request caching, connection pooling, and data processing
- **Expected Usage**: 16GB for data cache, 8GB for connections, 8GB for system

**Storage**: 1TB NVMe SSD
- **Rationale**: Blockchain storage, request cache, and log storage
- **Expected Growth**: ~5GB per month with current network activity

**Network**: 10Gbps+ symmetric connection
- **Rationale**: High-volume API serving, peer synchronization, and data access
- **Expected Bandwidth**: 500GB+ per month for API serving and sync

### Recommended Specifications

**CPU**: 24 cores, 3.5GHz+ (Intel/AMD x64)
- **Benefits**: Higher request throughput, better concurrent processing
- **Expected Performance**: 20,000+ RPC requests per second

**Memory**: 64GB RAM
- **Benefits**: Larger request cache, more concurrent connections
- **Expected Usage**: 32GB for data cache, 16GB for connections, 16GB for system

**Storage**: 2TB NVMe SSD
- **Benefits**: Extended cache, better I/O performance
- **Expected Growth**: Support for higher request volumes

**Network**: 25Gbps+ symmetric connection
- **Benefits**: Higher API serving capacity, better user experience

## API Architecture

### RPC Server Architecture

```rust
pub struct RPCServer {
    // Core components
    pub http_server: Arc<HttpServer>,          // HTTP/HTTPS API
    pub websocket_server: Arc<WebSocketServer>,  // WebSocket API
    pub ipc_server: Arc<IpcServer>,            // IPC API
    
    // Request handling
    pub request_router: Arc<RequestRouter>,    // Request routing
    pub request_handler: Arc<RequestHandler>,  // Request processing
    pub response_serializer: Arc<Serializer>,   // Response serialization
    
    // Performance optimization
    pub request_cache: Arc<RequestCache>,     // Request caching
    pub connection_pool: Arc<ConnectionPool>,  // Connection management
    pub rate_limiter: Arc<RateLimiter>,       // Rate limiting
    
    // Security
    pub auth_manager: Arc<AuthManager>,        // Authentication
    pub access_control: Arc<AccessControl>,  // Authorization
    
    // Monitoring
    pub metrics: Arc<MetricsCollector>,       // Performance metrics
    pub logger: Arc<Logger>,                   // Request logging
}
```

### Request Processing Pipeline

**Request Flow**:
```rust
impl RequestHandler {
    pub async fn handle_request(&self, request: RPCRequest) -> Result<RPCResponse, RPCError> {
        // 1. Authentication
        let auth_result = self.auth_manager.authenticate(&request)?;
        if !auth_result.is_authenticated {
            return Err(RPCError::Unauthorized);
        }
        
        // 2. Authorization
        if !self.access_control.check_permission(&auth_result.user, &request.method) {
            return Err(RPCError::Forbidden);
        }
        
        // 3. Rate limiting
        if !self.rate_limiter.check(&auth_result.user, &request.method) {
            return Err(RPCError::RateLimited);
        }
        
        // 4. Request validation
        self.validate_request(&request)?;
        
        // 5. Check cache
        if let Some(cached_response) = self.request_cache.get(&request) {
            return Ok(cached_response);
        }
        
        // 6. Process request
        let response = self.process_request_internal(&request).await?;
        
        // 7. Cache response
        self.request_cache.put(request.clone(), response.clone());
        
        // 8. Log request
        self.logger.log_request(&request, &response, &auth_result);
        
        // 9. Update metrics
        self.metrics.record_request(&request, &response, auth_result.user);
        
        Ok(response)
    }
}
```

### API Endpoints

**Standard JSON-RPC Methods**:
```rust
// Blockchain methods
"eth_blockNumber" -> Get current block number
"eth_getBlockByNumber" -> Get block by number
"eth_getBlockByHash" -> Get block by hash
"eth_getTransactionByHash" -> Get transaction by hash
"eth_getTransactionReceipt" -> Get transaction receipt
"eth_call" -> Call contract method
"eth_estimateGas" -> Estimate gas usage
"eth_sendRawTransaction" -> Send raw transaction
"eth_getBalance" -> Get account balance
"eth_getTransactionCount" -> Get transaction count
"eth_getCode" -> Get contract code
"eth_getStorageAt" -> Get contract storage
"eth_getLogs" -> Get event logs

// Network methods
"net_version" -> Get network version
"net_listening" -> Get listening address
"net_peerCount" -> Get peer count
"net_peers" -> Get peer list

// Savitri-specific methods
"savitri_getValidatorSet" -> Get current validator set
"savitri_getConsensusStatus" -> Get consensus status
"savitri_getMonolithData" -> Get monolith data
"savitri_getProof" -> Get state proof
"savitri_getVoteTokenBalance" -> Get vote token balance
"savitri_submitProposal" -> Submit governance proposal
"savitri_getProposal" -> Get proposal details
```

### WebSocket API

**Real-time Subscriptions**:
```rust
pub struct WebSocketManager {
    pub subscriptions: Arc<RwLock<HashMap<SubscriptionId, Subscription>>>,
    pub event_emitter: Arc<EventEmitter>,
    pub connection_manager: Arc<ConnectionManager>,
}

impl WebSocketManager {
    pub async fn subscribe(&self, params: SubscribeParams) -> Result<SubscriptionId, SubscribeError> {
        let subscription_id = self.generate_subscription_id();
        
        match params.subscription_type {
            SubscriptionType::NewBlocks => {
                let subscription = BlockSubscription {
                    id: subscription_id,
                    filters: params.filters,
                    connection_id: params.connection_id,
                };
                
                self.subscriptions.write().await.insert(subscription_id, subscription);
                self.event_emitter.subscribe(BlockEvent::new(), subscription_id).await;
            },
            
            SubscriptionType::NewTransactions => {
                let subscription = TransactionSubscription {
                    id: subscription_id,
                    filters: params.filters,
                    connection_id: params.connection_id,
                };
                
                self.subscriptions.write().await.insert(subscription_id, subscription);
                self.event_emitter.subscribe(TransactionEvent::new(), subscription_id).await;
            },
            
            SubscriptionType::Logs => {
                let subscription = LogSubscription {
                    id: subscription_id,
                    filters: params.filters,
                    connection_id: params.connection_id,
                };
                
                self.subscriptions.write().await.insert(subscription_id, subscription);
                self.event_emitter.subscribe(LogEvent::new(), subscription_id).await;
            },
            
            SubscriptionType::ContractEvents => {
                let subscription = ContractEventSubscription {
                    id: subscription_id,
                    contract_address: params.contract_address,
                    event_names: params.event_names,
                    connection_id: params.connection_id,
                };
                
                self.subscriptions.write().await.insert(subscription_id, subscription);
                self.event_emitter.subscribe(ContractEvent::new(), subscription_id).await;
            },
        }
        
        Ok(subscription_id)
    }
    
    pub async fn unsubscribe(&self, subscription_id: SubscriptionId) -> Result<(), UnsubscribeError> {
        if let Some(subscription) = self.subscriptions.write().await.remove(&subscription_id) {
            self.event_emitter.unsubscribe(subscription_id).await;
            self.connection_manager.close_connection(subscription.connection_id).await?;
        }
        
        Ok(())
    }
}
```

## Performance Optimization

### Request Caching

**Multi-Level Caching**:
```rust
pub struct RequestCache {
    // Level 1: In-memory cache (hot requests)
    pub l1_cache: Arc<LruCache<CacheKey, CachedResponse>>,
    
    // Level 2: Redis cache (warm requests)
    pub l2_cache: Arc<RedisCache>,
    
    // Level 3: Database cache (cold requests)
    pub l3_cache: Arc<DatabaseCache>,
    
    // Cache management
    pub cache_stats: Arc<CacheStats>,
    pub cache_policy: CachePolicy,
}

impl RequestCache {
    pub async fn get(&self, request: &RPCRequest) -> Option<CachedResponse> {
        let cache_key = self.generate_cache_key(request);
        
        // Try L1 cache first
        if let Some(response) = self.l1_cache.get(&cache_key) {
            self.cache_stats.record_hit(CacheLevel::L1);
            return Some(response);
        }
        
        // Try L2 cache
        if let Some(response) = self.l2_cache.get(&cache_key).await {
            self.cache_stats.record_hit(CacheLevel::L2);
            // Promote to L1
            self.l1_cache.put(cache_key.clone(), response.clone());
            return Some(response);
        }
        
        // Try L3 cache
        if let Some(response) = self.l3_cache.get(&cache_key).await {
            self.cache_stats.record_hit(CacheLevel::L3);
            // Promote to L2 and L1
            self.l2_cache.put(cache_key.clone(), response.clone()).await;
            self.l1_cache.put(cache_key, response);
            return Some(response);
        }
        
        self.cache_stats.record_miss();
        None
    }
    
    pub async fn put(&self, request: &RPCRequest, response: &CachedResponse) {
        let cache_key = self.generate_cache_key(request);
        let ttl = self.cache_policy.calculate_ttl(request, response);
        
        // Store in all cache levels
        self.l1_cache.put(cache_key.clone(), response.clone());
        self.l2_cache.put(cache_key.clone(), response.clone(), ttl).await;
        self.l3_cache.put(cache_key, response, ttl).await;
    }
}
```

### Connection Pooling

**Database Connection Pool**:
```rust
pub struct DatabasePool {
    pub connections: Arc<Mutex<Vec<DatabaseConnection>>>,
    pub available_connections: Arc<Mutex<Vec<usize>>>,
    pub max_connections: usize,
    pub min_connections: usize,
}

impl DatabasePool {
    pub async fn get_connection(&self) -> Result<DatabaseConnection, PoolError> {
        let mut connections = self.connections.lock().await;
        let mut available = self.available_connections.lock().await;
        
        if let Some(index) = available.pop() {
            Ok(connections[index].clone())
        } else if connections.len() < self.max_connections {
            // Create new connection
            let new_connection = self.create_connection().await?;
            connections.push(new_connection.clone());
            Ok(new_connection)
        } else {
            Err(PoolError::NoAvailableConnections)
        }
    }
    
    pub async fn return_connection(&self, connection: DatabaseConnection) {
        let connections = self.connections.lock().await;
        let mut available = self.available_connections.lock().await;
        
        // Find connection index
        if let Some(index) = connections.iter().position(|c| c.id() == connection.id()) {
            available.push(index);
        }
    }
}
```

### Request Batching

**Batch Processing**:
```rust
pub struct BatchProcessor {
    pub batch_size: usize,
    pub batch_timeout: Duration,
    pub pending_requests: Arc<Mutex<Vec<BatchRequest>>>,
    pub processor: Arc<dyn BatchProcessor>,
}

impl BatchProcessor {
    pub async fn add_request(&self, request: RPCRequest) -> Result<BatchResult, BatchError> {
        let mut pending = self.pending_requests.lock().await;
        
        let batch_request = BatchRequest {
            request,
            added_at: Instant::now(),
            response_sender: None,
        };
        
        pending.push(batch_request);
        
        // Check if batch is ready
        if pending.len() >= self.batch_size {
            self.process_batch().await?;
        }
        
        Ok(BatchResult::Queued)
    }
    
    pub async fn process_batch(&self) -> Result<(), BatchError> {
        let mut pending = self.pending_requests.lock().await;
        
        if pending.is_empty() {
            return Ok(());
        }
        
        let batch_requests: Vec<_> = pending.drain(..).collect();
        drop(pending);
        
        // Process batch
        let results = self.processor.process_batch_requests(&batch_requests).await?;
        
        // Send responses
        for (request, result) in batch_requests.iter().zip(results) {
            if let Some(sender) = &request.response_sender {
                let _ = sender.send(result);
            }
        }
        
        Ok(())
    }
}
```

## Security and Access Control

### Authentication

**JWT Authentication**:
```rust
pub struct AuthManager {
    pub jwt_secret: Vec<u8>,
    pub token_store: Arc<TokenStore>,
    pub user_store: Arc<UserStore>,
}

impl AuthManager {
    pub fn authenticate(&self, request: &RPCRequest) -> Result<AuthResult, AuthError> {
        // 1. Extract token from request
        let token = self.extract_token(request)?;
        
        // 2. Validate token
        let claims = self.validate_jwt_token(&token)?;
        
        // 3. Check token revocation
        if self.token_store.is_revoked(&claims.jti)? {
            return Err(AuthError::TokenRevoked);
        }
        
        // 4. Load user information
        let user = self.user_store.get_user(&claims.sub)?;
        
        // 5. Check user status
        if user.status != UserStatus::Active {
            return Err(AuthError::UserInactive);
        }
        
        Ok(AuthResult {
            user,
            claims,
            permissions: user.permissions,
        })
    }
    
    pub fn validate_jwt_token(&self, token: &str) -> Result<JWTClaims, AuthError> {
        let validation = Validation::new(ValidationOptions {
            algorithms: vec![Algorithm::HS256],
            leeway: 60, // 1 minute leeway
            validate_exp: true,
            validate_nbf: true,
        });
        
        let token_data = jsonwebtoken::decode::<JWTClaims>(
            token,
            &DecodingKey::from_secret(&self.jwt_secret),
            &validation,
        )?;
        
        Ok(token_data.claims)
    }
}
```

### Authorization

**Role-Based Access Control**:
```rust
pub struct AccessControl {
    pub role_permissions: HashMap<Role, Vec<Permission>>,
    pub user_roles: HashMap<UserId, Vec<Role>>,
    pub resource_permissions: HashMap<Resource, Vec<Permission>>,
}

impl AccessControl {
    pub fn check_permission(&self, user: &User, method: &str) -> bool {
        // 1. Get user roles
        let user_roles = self.user_roles.get(&user.id)
            .unwrap_or(&vec![]);
        
        // 2. Get permissions for roles
        let mut permissions = HashSet::new();
        for role in user_roles {
            if let Some(role_perms) = self.role_permissions.get(role) {
                permissions.extend(role_perms);
            }
        }
        
        // 3. Check method permission
        let required_permission = Permission::from_method(method);
        permissions.contains(&required_permission)
    }
    
    pub fn check_resource_access(&self, user: &User, resource: &Resource, action: &str) -> bool {
        // 1. Get user roles
        let user_roles = self.user_roles.get(&user.id)
            .unwrap_or(&vec![]);
        
        // 2. Get permissions for roles
        let mut permissions = HashSet::new();
        for role in user_roles {
            if let Some(role_perms) = self.role_permissions.get(role) {
                permissions.extend(role_perms);
            }
        }
        
        // 3. Get resource permissions
        let resource_perms = self.resource_permissions.get(resource)
            .unwrap_or(&vec![]);
        
        // 4. Check action permission
        let required_permission = Permission::from_action(action);
        permissions.contains(&required_permission) || 
        resource_perms.contains(&required_permission)
    }
}
```

### Rate Limiting

**Multi-Dimensional Rate Limiting**:
```rust
pub struct RateLimiter {
    pub user_limits: HashMap<UserId, UserLimits>,
    pub method_limits: HashMap<String, MethodLimits>,
    pub global_limits: GlobalLimits,
    pub redis_client: Arc<RedisClient>,
}

impl RateLimiter {
    pub async fn check(&self, user: &UserId, method: &str) -> bool {
        // 1. Check global limits
        if !self.check_global_limits().await {
            return false;
        }
        
        // 2. Check user limits
        if let Some(user_limits) = self.user_limits.get(user) {
            if !self.check_user_limits(user, user_limits).await {
                return false;
            }
        }
        
        // 3. Check method limits
        if let Some(method_limits) = self.method_limits.get(method) {
            if !self.check_method_limits(user, method_limits).await {
                return false;
            }
        }
        
        true
    }
    
    async fn check_user_limits(&self, user: &UserId, limits: &UserLimits) -> bool {
        let key = format!("user_limit:{}:{}", user, limits.window);
        let current_count = self.redis_client.incr(&key).await.unwrap_or(0);
        
        if current_count == 1 {
            // Set expiration for new key
            self.redis_client.expire(&key, limits.window).await;
        }
        
        current_count <= limits.max_requests
    }
    
    async fn check_method_limits(&self, user: &UserId, limits: &MethodLimits) -> bool {
        let key = format!("method_limit:{}:{}:{}", user, limits.method, limits.window);
        let current_count = self.redis_client.incr(&key).await.unwrap_or(0);
        
        if current_count == 1 {
            self.redis_client.expire(&key, limits.window).await;
        }
        
        current_count <= limits.max_requests
    }
}
```

## Monitoring and Analytics

### API Metrics

**Performance Metrics**:
```rust
pub struct APIMetrics {
    // Request metrics
    pub total_requests: u64,
    pub successful_requests: u64,
    pub failed_requests: u64,
    pub average_response_time: Duration,
    
    // Method metrics
    pub method_metrics: HashMap<String, MethodMetrics>,
    
    // User metrics
    pub user_metrics: HashMap<UserId, UserMetrics>,
    
    // System metrics
    pub connection_metrics: ConnectionMetrics,
    pub cache_metrics: CacheMetrics,
    pub error_metrics: ErrorMetrics,
}

impl MetricsCollector {
    pub fn record_request(&self, request: &RPCRequest, response: &RPCResponse, user: &UserId) {
        // Update global metrics
        self.global_metrics.total_requests += 1;
        
        if response.is_success() {
            self.global_metrics.successful_requests += 1;
        } else {
            self.global_metrics.failed_requests += 1;
        }
        
        // Update method metrics
        let method_metrics = self.method_metrics.entry(request.method.clone())
            .or_insert_with(MethodMetrics::default);
        
        method_metrics.total_requests += 1;
        method_metrics.total_time += response.response_time;
        
        // Update user metrics
        let user_metrics = self.user_metrics.entry(user.clone())
            .or_insert_with(UserMetrics::default);
        
        user_metrics.total_requests += 1;
        user_metrics.total_time += response.response_time;
        
        // Update error metrics
        if let Some(error) = &response.error {
            let error_metrics = self.error_metrics.entry(error.code.clone())
                .or_insert_with(ErrorMetrics::default);
            
            error_metrics.count += 1;
        }
    }
}
```

### Analytics Dashboard

**Real-time Analytics**:
```rust
pub struct AnalyticsDashboard {
    pub metrics_collector: Arc<MetricsCollector>,
    pub alert_manager: Arc<AlertManager>,
    pub report_generator: Arc<ReportGenerator>,
}

impl AnalyticsDashboard {
    pub async fn get_real_time_stats(&self) -> Result<RealTimeStats, AnalyticsError> {
        let metrics = self.metrics_collector.get_current_metrics().await?;
        
        Ok(RealTimeStats {
            requests_per_second: metrics.requests_per_second,
            average_response_time: metrics.average_response_time,
            success_rate: metrics.success_rate,
            active_connections: metrics.active_connections,
            cache_hit_rate: metrics.cache_hit_rate,
            error_rate: metrics.error_rate,
        })
    }
    
    pub async fn generate_report(&self, period: ReportPeriod) -> Result<AnalyticsReport, ReportError> {
        let metrics = self.metrics_collector.get_metrics_for_period(period).await?;
        
        let mut report = AnalyticsReport::new(period);
        
        // Request analytics
        report.requests = self.generate_request_analytics(&metrics)?;
        
        // Performance analytics
        report.performance = self.generate_performance_analytics(&metrics)?;
        
        // User analytics
        report.users = self.generate_user_analytics(&metrics)?;
        
        // Error analytics
        report.errors = self.generate_error_analytics(&metrics)?;
        
        // Recommendations
        report.recommendations = self.generate_recommendations(&metrics)?;
        
        Ok(report)
    }
}
```

## Configuration

### Server Configuration

**RPC Server Config**:
```toml
[rpc]
# Server configuration
enabled = true
listen_addr = "0.0.0.0:8545"
websocket_addr = "0.0.0.0:8546"
ipc_path = "/var/lib/savitri/savitri.ipc"

# Performance configuration
max_connections = 10000
request_timeout = 30
batch_size = 100
batch_timeout = 50

# Cache configuration
cache_enabled = true
cache_size = "16GB"
cache_ttl = 300
redis_url = "redis://localhost:6379"

# Security configuration
auth_enabled = true
jwt_secret = "your-secret-key"
rate_limit_enabled = true
cors_enabled = true

# Logging configuration
log_requests = true
log_level = "info"
log_file = "/var/log/savitri/rpc.log"

# Metrics configuration
metrics_enabled = true
metrics_port = 9090
health_check_port = 8080
```

### Method Configuration

**Method-Specific Settings**:
```toml
[rpc.methods]
# Eth methods
eth_blockNumber = { enabled = true, cache_ttl = 10 }
eth_getBlockByNumber = { enabled = true, cache_ttl = 60 }
eth_getTransactionByHash = { enabled = true, cache_ttl = 300 }
eth_call = { enabled = true, cache_ttl = 60 }
eth_sendRawTransaction = { enabled = true, cache_ttl = 0 }

# Network methods
net_peerCount = { enabled = true, cache_ttl = 30 }
net_peers = { enabled = true, cache_ttl = 60 }

# Savitri methods
savitri_getValidatorSet = { enabled = true, cache_ttl = 300 }
savitri_getConsensusStatus = { enabled = true, cache_ttl = 10 }
savitri_getMonolithData = { enabled = true, cache_ttl = 600 }

# Subscription methods
eth_subscribe = { enabled = true, max_subscriptions = 1000 }
eth_unsubscribe = { enabled = true }
```

## Troubleshooting

### Common Issues

**1. High Response Times**
```bash
# Check request metrics
curl http://localhost:9090/metrics | grep rpc_response_time

# Check cache hit rate
curl http://localhost:9090/metrics | grep cache_hit_rate

# Check database connections
curl http://localhost:9090/metrics | grep db_connections

# Clear cache
curl -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"admin_clearCache","params":[],"id":1}'
```

**2. Rate Limiting Issues**
```bash
# Check rate limit status
curl -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"admin_rateLimitStatus","params":[],"id":1}'

# Reset rate limits
curl -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"admin_resetRateLimits","params":[],"id":1}'
```

**3. Authentication Issues**
```bash
# Check auth status
curl -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"admin_authStatus","params":[],"id":1}'

# Test authentication
curl -X POST http://localhost:8545 -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'
```

This RPC node specification provides comprehensive guidance for deploying and operating high-performance Savitri Network RPC nodes with optimized API services, security controls, and monitoring capabilities.
