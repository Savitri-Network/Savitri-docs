# IoT Connector Architecture

## IoT Connector Overview

Savitri Network implements a comprehensive IoT connector that enables seamless integration between IoT devices and blockchain infrastructure. The connector provides secure, scalable, and efficient communication channels for IoT data streams, device management, and real-time processing.

## Technology Choice Rationale

### Why Dedicated IoT Connector

**Problem Statement**: Traditional blockchain systems are not designed for high-frequency, low-latency IoT data streams, leading to performance bottlenecks, high costs, and poor user experience for IoT applications.

**Chosen Solution**: Dedicated IoT connector with optimized data ingestion, batch processing, and edge computing capabilities.

**Rationale**:
- **Performance**: Optimized for high-frequency IoT data streams
- **Cost Efficiency**: Batch processing reduces transaction costs
- **Scalability**: Edge computing and data aggregation
- **Security**: Device authentication and data integrity

**Expected Results**:
- 1000x improvement in IoT data processing throughput
- 90% reduction in transaction costs through batching
- Sub-second latency for critical IoT operations
- Enterprise-grade security for IoT ecosystems

### Why Edge Computing Integration

**Problem Statement**: Sending every IoT reading directly to blockchain is inefficient and expensive, creating unnecessary network congestion and costs.

**Chosen Solution**: Edge computing integration with local data processing, intelligent filtering, and selective blockchain submission.

**Rationale**:
- **Efficiency**: Local processing reduces blockchain load
- **Cost Reduction**: Only important data is submitted
- **Latency**: Critical operations handled locally
- **Reliability**: Offline operation capability

**Expected Results**:
- 95% reduction in blockchain transactions
- Millisecond response times for local operations
- Improved system reliability and uptime
- Lower operational costs

## IoT Connector Architecture

### Core Components
```rust
pub struct IoTConnector {
    pub device_manager: DeviceManager,        // Device management
    pub data_ingestion: DataIngestionEngine,   // Data ingestion
    pub edge_processor: EdgeProcessor,         // Edge computing
    pub batch_processor: BatchProcessor,       // Batch processing
    pub blockchain_interface: BlockchainInterface, // Blockchain interface
    pub security_manager: SecurityManager,     // Security management
    pub config: IoTConfig,                     // Configuration
}

pub struct DeviceManager {
    pub devices: HashMap<DeviceId, Device>,   // Registered devices
    pub device_groups: HashMap<GroupId, DeviceGroup>, // Device groups
    pub authentication: DeviceAuth,             // Device authentication
    pub provisioning: DeviceProvisioning,       // Device provisioning
    pub monitoring: DeviceMonitoring,         // Device monitoring
}

#[derive(Debug, Clone, Hash, PartialEq, Eq)]
pub struct DeviceId {
    pub id: String,                            // Device identifier
    pub type_: DeviceType,                     // Device type
    pub version: String,                       // Device version
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum DeviceType {
    Sensor,                                   // Sensor device
    Actuator,                                 // Actuator device
    Gateway,                                  // Gateway device
    EdgeNode,                                 // Edge computing node
    Mobile,                                   // Mobile device
    Industrial,                               // Industrial device
}

#[derive(Debug, Clone)]
pub struct Device {
    pub id: DeviceId,                         // Device ID
    pub name: String,                         // Device name
    pub owner: [u8; 32],                      // Device owner
    pub public_key: [u8; 32],                  // Device public key
    pub capabilities: Vec<DeviceCapability>,   // Device capabilities
    pub status: DeviceStatus,                  // Device status
    pub last_seen: u64,                        // Last seen timestamp
    pub metadata: DeviceMetadata,              // Device metadata
    pub configuration: DeviceConfig,           // Device configuration
}

#[derive(Debug, Clone)]
pub struct DeviceCapability {
    pub capability_type: CapabilityType,       // Capability type
    pub parameters: HashMap<String, CapabilityParameter>, // Parameters
    pub data_format: DataFormat,               // Data format
    pub frequency: Frequency,                   // Data frequency
    pub priority: Priority,                    // Priority level
}

#[derive(Debug, Clone)]
pub enum CapabilityType {
    Temperature,                               // Temperature sensing
    Humidity,                                 // Humidity sensing
    Pressure,                                 // Pressure sensing
    Motion,                                   // Motion detection
    Light,                                    // Light sensing
    Sound,                                    // Sound sensing
    Location,                                 // Location tracking
    Actuation,                                // Actuation control
    Custom(String),                          // Custom capability
}
```

### Data Ingestion Engine
```rust
pub struct DataIngestionEngine {
    pub protocols: HashMap<String, Box<dyn ProtocolHandler>>, // Protocol handlers
    pub data_validator: DataValidator,        // Data validation
    pub data_transformer: DataTransformer,    // Data transformation
    pub rate_limiter: RateLimiter,            // Rate limiting
    pub buffer_manager: BufferManager,        // Buffer management
}

pub trait ProtocolHandler {
    fn handle_connection(&mut self, connection: &mut dyn Connection) -> Result<(), ProtocolError>;
    fn parse_data(&self, raw_data: &[u8]) -> Result<IoTData, ParseError>;
    fn validate_device(&self, device_id: &DeviceId) -> Result<bool, AuthError>;
}

pub struct MQTTHandler {
    pub client: mqtt::Client,                 // MQTT client
    pub topics: HashMap<String, TopicConfig>,  // Topic configurations
    pub qos_levels: HashMap<String, QoS>,     // QoS levels
}

impl ProtocolHandler for MQTTHandler {
    fn handle_connection(&mut self, connection: &mut dyn Connection) -> Result<(), ProtocolError> {
        // Handle MQTT connection
        self.client.connect(connection)?;
        
        // Subscribe to topics
        for (topic, config) in &self.topics {
            self.client.subscribe(topic, config.qos)?;
        }
        
        Ok(())
    }
    
    fn parse_data(&self, raw_data: &[u8]) -> Result<IoTData, ParseError> {
        // Parse MQTT message
        let message: mqtt::Message = serde_json::from_slice(raw_data)?;
        
        // Extract IoT data
        let iot_data = IoTData {
            device_id: DeviceId::from_string(&message.device_id)?,
            timestamp: message.timestamp,
            data_type: DataType::from_string(&message.data_type)?,
            value: message.value,
            unit: message.unit,
            quality: message.quality.unwrap_or(1.0),
            metadata: message.metadata,
        };
        
        Ok(iot_data)
    }
    
    fn validate_device(&self, device_id: &DeviceId) -> Result<bool, AuthError> {
        // Validate device against registry
        // This would check device certificates, etc.
        Ok(true)
    }
}

pub struct CoAPHandler {
    pub server: coap::Server,                 // CoAP server
    pub resources: HashMap<String, Resource>,  // CoAP resources
}

impl ProtocolHandler for CoAPHandler {
    fn handle_connection(&mut self, connection: &mut dyn Connection) -> Result<(), ProtocolError> {
        // Handle CoAP connection
        self.server.handle_connection(connection)?;
        Ok(())
    }
    
    fn parse_data(&self, raw_data: &[u8]) -> Result<IoTData, ParseError> {
        // Parse CoAP message
        let message: coap::Message = coap::Message::from_bytes(raw_data)?;
        
        // Extract IoT data
        let iot_data = IoTData {
            device_id: DeviceId::from_coap_uri(&message.uri_path)?,
            timestamp: current_timestamp(),
            data_type: DataType::from_coap_content_format(&message.content_format)?,
            value: self.extract_coap_value(&message.payload)?,
            unit: self.extract_coap_unit(&message.options)?,
            quality: 1.0,
            metadata: HashMap::new(),
        };
        
        Ok(iot_data)
    }
    
    fn validate_device(&self, device_id: &DeviceId) -> Result<bool, AuthError> {
        // Validate CoAP device
        Ok(true)
    }
}

#[derive(Debug, Clone)]
pub struct IoTData {
    pub device_id: DeviceId,                   // Device ID
    pub timestamp: u64,                        // Timestamp
    pub data_type: DataType,                   // Data type
    pub value: DataValue,                      // Data value
    pub unit: Option<String>,                  // Unit
    pub quality: f64,                         // Data quality (0.0-1.0)
    pub metadata: HashMap<String, String>,     // Metadata
}

#[derive(Debug, Clone)]
pub enum DataType {
    Temperature,                               // Temperature data
    Humidity,                                 // Humidity data
    Pressure,                                 // Pressure data
    Motion,                                   // Motion data
    Light,                                    // Light data
    Sound,                                    // Sound data
    Location,                                 // Location data
    Binary,                                   // Binary data
    Text,                                     // Text data
    Custom(String),                          // Custom data type
}

#[derive(Debug, Clone)]
pub enum DataValue {
    Float(f64),                               // Floating point value
    Integer(i64),                             // Integer value
    Boolean(bool),                            // Boolean value
    String(String),                           // String value
    Binary(Vec<u8>),                          // Binary value
    Struct(HashMap<String, DataValue>),      // Structured data
    Array(Vec<DataValue>),                    // Array data
}
```

### Edge Processing Engine
```rust
pub struct EdgeProcessor {
    pub rules_engine: RulesEngine,            // Rules engine
    pub anomaly_detector: AnomalyDetector,    // Anomaly detection
    pub data_aggregator: DataAggregator,      // Data aggregation
    pub local_storage: LocalStorage,          // Local storage
    pub ml_models: HashMap<String, MLModel>,  // ML models
}

impl EdgeProcessor {
    pub fn process_data(&mut self, data: IoTData) -> Result<ProcessingResult, ProcessingError> {
        // 1. Apply rules
        let rule_result = self.rules_engine.apply_rules(&data)?;
        
        // 2. Detect anomalies
        let anomaly_result = self.anomaly_detector.detect_anomalies(&data)?;
        
        // 3. Aggregate data if needed
        let aggregated_data = self.data_aggregator.aggregate(data.clone())?;
        
        // 4. Apply ML models
        let ml_result = self.apply_ml_models(&data)?;
        
        // 5. Determine processing action
        let action = self.determine_processing_action(&rule_result, &anomaly_result, &aggregated_data, &ml_result)?;
        
        // 6. Store locally if needed
        if action.store_locally {
            self.local_storage.store_data(&data)?;
        }
        
        Ok(ProcessingResult {
            action,
            processed_data: aggregated_data,
            rule_matches: rule_result.matches,
            anomalies: anomaly_result.anomalies,
            ml_predictions: ml_result.predictions,
            processing_time: self.get_processing_time(),
        })
    }
    
    fn determine_processing_action(
        &self,
        rule_result: &RuleResult,
        anomaly_result: &AnomalyResult,
        aggregated_data: &Option<IoTData>,
        ml_result: &MLResult,
    ) -> Result<ProcessingAction, ProcessingError> {
        let mut action = ProcessingAction::default();
        
        // Check for critical alerts
        if rule_result.has_critical_alerts() || anomaly_result.has_critical_anomalies() {
            action.priority = Priority::Critical;
            action.submit_to_blockchain = true;
            action.trigger_alert = true;
            action.store_locally = true;
        }
        
        // Check for ML predictions
        if ml_result.has_predictions() {
            for prediction in &ml_result.predictions {
                match prediction.prediction_type {
                    PredictionType::Maintenance => {
                        action.schedule_maintenance = true;
                        action.submit_to_blockchain = true;
                    },
                    PredictionType::Failure => {
                        action.priority = Priority::High;
                        action.trigger_alert = true;
                        action.submit_to_blockchain = true;
                    },
                    PredictionType::Optimization => {
                        action.apply_optimization = true;
                        action.store_locally = true;
                    },
                }
            }
        }
        
        // Check for aggregated data
        if aggregated_data.is_some() {
            action.submit_to_blockchain = true;
            action.priority = Priority::Normal;
        }
        
        Ok(action)
    }
    
    fn apply_ml_models(&mut self, data: &IoTData) -> Result<MLResult, ProcessingError> {
        let mut predictions = Vec::new();
        
        for (model_name, model) in &self.ml_models {
            if model.is_applicable_for_data(data) {
                let prediction = model.predict(data)?;
                predictions.push(prediction);
            }
        }
        
        Ok(MLResult {
            predictions,
            confidence: self.calculate_overall_confidence(&predictions),
            processing_time: self.get_ml_processing_time(),
        })
    }
}

pub struct RulesEngine {
    pub rules: Vec<Rule>,                     // Processing rules
    pub rule_optimizer: RuleOptimizer,       // Rule optimization
    pub rule_cache: LruCache<String, RuleResult>, // Rule cache
}

#[derive(Debug, Clone)]
pub struct Rule {
    pub id: String,                           // Rule ID
    pub name: String,                         // Rule name
    pub conditions: Vec<Condition>,           // Rule conditions
    pub actions: Vec<Action>,                 // Rule actions
    pub priority: Priority,                    // Rule priority
    pub enabled: bool,                        // Rule enabled
    pub metadata: RuleMetadata,               // Rule metadata
}

#[derive(Debug, Clone)]
pub struct Condition {
    pub field: String,                         // Field name
    pub operator: ComparisonOperator,         // Comparison operator
    pub value: DataValue,                     // Comparison value
    pub logical_operator: Option<LogicalOperator>, // Logical operator
}

#[derive(Debug, Clone)]
pub enum ComparisonOperator {
    Equals,                                   // Equals
    NotEquals,                               // Not equals
    GreaterThan,                             // Greater than
    LessThan,                                // Less than
    GreaterThanOrEqual,                      // Greater than or equal
    LessThanOrEqual,                         // Less than or equal
    Contains,                                // Contains
    NotContains,                             // Not contains
    In,                                      // In set
    NotIn,                                   // Not in set
}

#[derive(Debug, Clone)]
pub enum Action {
    Alert {                                  // Alert action
        message: String,                     // Alert message
        severity: Severity,                   // Alert severity
        recipients: Vec<String>,              // Alert recipients
    },
    Submit {                                 // Submit action
        priority: Priority,                   // Submission priority
        metadata: HashMap<String, String>,   // Submission metadata
    },
    Store {                                  // Store action
        location: String,                     // Storage location
        retention: Duration,                   // Retention period
    },
    Trigger {                                // Trigger action
        target: String,                       // Trigger target
        parameters: HashMap<String, String>,  // Trigger parameters
    },
}

impl RulesEngine {
    pub fn apply_rules(&mut self, data: &IoTData) -> Result<RuleResult, RulesError> {
        let mut matches = Vec::new();
        let mut actions = Vec::new();
        
        for rule in &self.rules {
            if !rule.enabled {
                continue;
            }
            
            if self.evaluate_rule(rule, data)? {
                matches.push(rule.clone());
                actions.extend(rule.actions.clone());
            }
        }
        
        Ok(RuleResult {
            matches,
            actions,
            processing_time: self.get_rule_processing_time(),
        })
    }
    
    fn evaluate_rule(&self, rule: &Rule, data: &IoTData) -> Result<bool, RulesError> {
        let mut result = true;
        
        for (i, condition) in rule.conditions.iter().enumerate() {
            let condition_result = self.evaluate_condition(condition, data)?;
            
            if i == 0 {
                result = condition_result;
            } else if let Some(op) = condition.logical_operator {
                match op {
                    LogicalOperator::And => result &= condition_result,
                    LogicalOperator::Or => result |= condition_result,
                }
            }
        }
        
        Ok(result)
    }
    
    fn evaluate_condition(&self, condition: &Condition, data: &IoTData) -> Result<bool, RulesError> {
        let field_value = self.extract_field_value(&condition.field, data)?;
        
        let result = match condition.operator {
            ComparisonOperator::Equals => self.compare_equals(&field_value, &condition.value),
            ComparisonOperator::NotEquals => !self.compare_equals(&field_value, &condition.value),
            ComparisonOperator::GreaterThan => self.compare_greater_than(&field_value, &condition.value),
            ComparisonOperator::LessThan => self.compare_less_than(&field_value, &condition.value),
            ComparisonOperator::GreaterThanOrEqual => self.compare_greater_than_or_equal(&field_value, &condition.value),
            ComparisonOperator::LessThanOrEqual => self.compare_less_than_or_equal(&field_value, &condition.value),
            ComparisonOperator::Contains => self.compare_contains(&field_value, &condition.value),
            ComparisonOperator::NotContains => !self.compare_contains(&field_value, &condition.value),
            ComparisonOperator::In => self.compare_in(&field_value, &condition.value),
            ComparisonOperator::NotIn => !self.compare_in(&field_value, &condition.value),
        };
        
        Ok(result)
    }
    
    fn extract_field_value(&self, field: &str, data: &IoTData) -> Result<DataValue, RulesError> {
        match field {
            "device_id" => Ok(DataValue::String(data.device_id.id.clone())),
            "timestamp" => Ok(DataValue::Integer(data.timestamp as i64)),
            "data_type" => Ok(DataValue::String(format!("{:?}", data.data_type))),
            "value" => Ok(data.value.clone()),
            "quality" => Ok(DataValue::Float(data.quality)),
            _ => Err(RulesError::UnknownField(field.to_string())),
        }
    }
}
```

### Batch Processing Engine
```rust
pub struct BatchProcessor {
    pub batch_config: BatchConfig,            // Batch configuration
    pub aggregation_strategy: AggregationStrategy, // Aggregation strategy
    pub compression_engine: CompressionEngine, // Compression engine
    pub batch_scheduler: BatchScheduler,     // Batch scheduler
    pub blockchain_batcher: BlockchainBatcher, // Blockchain batcher
}

#[derive(Debug, Clone)]
pub struct BatchConfig {
    pub max_batch_size: usize,                // Maximum batch size
    pub max_batch_delay: Duration,             // Maximum batch delay
    pub min_batch_size: usize,                // Minimum batch size
    pub compression_enabled: bool,            // Compression enabled
    pub aggregation_enabled: bool,            // Aggregation enabled
}

#[derive(Debug, Clone)]
pub struct Batch {
    pub id: BatchId,                          // Batch ID
    pub data: Vec<IoTData>,                   // Batch data
    pub created_at: u64,                      // Creation timestamp
    pub size: usize,                          // Batch size
    pub compressed: bool,                      // Compressed flag
    pub aggregated: bool,                     // Aggregated flag
    pub priority: Priority,                    // Batch priority
}

impl BatchProcessor {
    pub fn process_batch(&mut self, data: IoTData) -> Result<Option<BatchId>, BatchError> {
        // 1. Add data to current batch
        let batch_id = self.add_to_batch(data)?;
        
        // 2. Check if batch should be submitted
        if self.should_submit_batch(&batch_id)? {
            let batch = self.get_batch(&batch_id)?;
            self.submit_batch(batch)?;
            self.remove_batch(&batch_id)?;
            return Ok(None);
        }
        
        Ok(Some(batch_id))
    }
    
    fn add_to_batch(&mut self, data: IoTData) -> Result<BatchId, BatchError> {
        let batch_id = self.get_or_create_batch()?;
        
        let batch = self.batches.get_mut(&batch_id)
            .ok_or(BatchError::BatchNotFound)?;
        
        batch.data.push(data);
        batch.size += 1;
        batch.created_at = current_timestamp();
        
        Ok(batch_id)
    }
    
    fn should_submit_batch(&self, batch_id: &BatchId) -> Result<bool, BatchError> {
        let batch = self.batches.get(batch_id)
            .ok_or(BatchError::BatchNotFound)?;
        
        let size_condition = batch.size >= self.batch_config.max_batch_size;
        let time_condition = batch.created_at + self.batch_config.max_batch_delay.as_secs() <= current_timestamp();
        let priority_condition = batch.priority == Priority::Critical;
        
        Ok(size_condition || time_condition || priority_condition)
    }
    
    fn submit_batch(&mut self, mut batch: Batch) -> Result<(), BatchError> {
        // 1. Aggregate data if enabled
        if self.batch_config.aggregation_enabled {
            batch = self.aggregate_batch(batch)?;
        }
        
        // 2. Compress data if enabled
        if self.batch_config.compression_enabled {
            batch = self.compress_batch(batch)?;
        }
        
        // 3. Create blockchain transaction
        let tx = self.blockchain_batcher.create_batch_transaction(batch)?;
        
        // 4. Submit to blockchain
        self.blockchain_batcher.submit_transaction(tx)?;
        
        Ok(())
    }
    
    fn aggregate_batch(&self, batch: Batch) -> Result<Batch, BatchError> {
        let mut aggregated_data = Vec::new();
        
        // Group data by device and type
        let mut grouped_data: HashMap<(DeviceId, DataType), Vec<IoTData>> = HashMap::new();
        
        for data in batch.data {
            let key = (data.device_id.clone(), data.data_type.clone());
            grouped_data.entry(key).or_insert_with(Vec::new).push(data);
        }
        
        // Apply aggregation strategy
        for ((device_id, data_type), data_group) in grouped_data {
            let aggregated = self.aggregation_strategy.aggregate(device_id, data_type, data_group)?;
            aggregated_data.push(aggregated);
        }
        
        Ok(Batch {
            data: aggregated_data,
            aggregated: true,
            ..batch
        })
    }
    
    fn compress_batch(&self, batch: Batch) -> Result<Batch, BatchError> {
        let mut compressed_data = Vec::new();
        
        for data in batch.data {
            let compressed = self.compression_engine.compress_data(&data)?;
            compressed_data.push(compressed);
        }
        
        Ok(Batch {
            data: compressed_data,
            compressed: true,
            ..batch
        })
    }
}

pub struct AggregationStrategy {
    pub strategies: HashMap<DataType, Box<dyn AggregationMethod>>, // Aggregation methods
}

pub trait AggregationMethod {
    fn aggregate(&self, data_group: Vec<IoTData>) -> Result<IoTData, AggregationError>;
}

pub struct NumericAggregator {
    pub method: NumericAggregationMethod,    // Aggregation method
}

#[derive(Debug, Clone)]
pub enum NumericAggregationMethod {
    Average,                                 // Average
    Sum,                                     // Sum
    Min,                                     // Minimum
    Max,                                     // Maximum
    Median,                                  // Median
    Percentile(f64),                         // Percentile
}

impl AggregationMethod for NumericAggregator {
    fn aggregate(&self, data_group: Vec<IoTData>) -> Result<IoTData, AggregationError> {
        if data_group.is_empty() {
            return Err(AggregationError::EmptyDataGroup);
        }
        
        let numeric_values: Vec<f64> = data_group.iter()
            .filter_map(|data| match &data.value {
                DataValue::Float(f) => Some(*f),
                DataValue::Integer(i) => Some(*i as f64),
                _ => None,
            })
            .collect();
        
        if numeric_values.is_empty() {
            return Err(AggregationError::NoNumericValues);
        }
        
        let aggregated_value = match self.method {
            NumericAggregationMethod::Average => {
                numeric_values.iter().sum::<f64>() / numeric_values.len() as f64
            },
            NumericAggregationMethod::Sum => numeric_values.iter().sum(),
            NumericAggregationMethod::Min => numeric_values.iter().fold(f64::INFINITY, |a, &b| a.min(b)),
            NumericAggregationMethod::Max => numeric_values.iter().fold(f64::NEG_INFINITY, |a, &b| a.max(b)),
            NumericAggregationMethod::Median => self.calculate_median(&numeric_values),
            NumericAggregationMethod::Percentile(p) => self.calculate_percentile(&numeric_values, p),
        };
        
        // Create aggregated data point
        let aggregated_data = IoTData {
            device_id: data_group[0].device_id.clone(),
            timestamp: current_timestamp(),
            data_type: data_group[0].data_type.clone(),
            value: DataValue::Float(aggregated_value),
            unit: data_group[0].unit.clone(),
            quality: self.calculate_aggregated_quality(&data_group),
            metadata: self.create_aggregated_metadata(&data_group),
        };
        
        Ok(aggregated_data)
    }
}
```

### Security Management
```rust
pub struct SecurityManager {
    pub device_auth: DeviceAuthenticator,     // Device authentication
    pub data_encryption: DataEncryption,     // Data encryption
    pub access_control: AccessControl,        // Access control
    pub audit_logger: AuditLogger,            // Audit logging
}

pub struct DeviceAuthenticator {
    pub certificate_store: CertificateStore, // Certificate store
    pub token_manager: TokenManager,          // Token management
    pub biometric_auth: BiometricAuth,        // Biometric authentication
}

impl DeviceAuthenticator {
    pub fn authenticate_device(&self, device_id: &DeviceId, credentials: &DeviceCredentials) -> Result<bool, AuthError> {
        match credentials {
            DeviceCredentials::Certificate(cert) => {
                self.verify_certificate(device_id, cert)
            },
            DeviceCredentials::Token(token) => {
                self.verify_token(device_id, token)
            },
            DeviceCredentials::Biometric(bio_data) => {
                self.verify_biometric(device_id, bio_data)
            },
        }
    }
    
    fn verify_certificate(&self, device_id: &DeviceId, certificate: &Certificate) -> Result<bool, AuthError> {
        // 1. Check certificate validity
        if !self.is_certificate_valid(certificate)? {
            return Ok(false);
        }
        
        // 2. Check certificate chain
        if !self.verify_certificate_chain(certificate)? {
            return Ok(false);
        }
        
        // 3. Check device certificate mapping
        if !self.certificate_store.is_device_certificate(device_id, certificate)? {
            return Ok(false);
        }
        
        // 4. Check revocation status
        if self.certificate_store.is_certificate_revoked(certificate)? {
            return Ok(false);
        }
        
        Ok(true)
    }
    
    fn verify_token(&self, device_id: &DeviceId, token: &AuthToken) -> Result<bool, AuthError> {
        // 1. Check token format
        if !self.is_valid_token_format(token)? {
            return Ok(false);
        }
        
        // 2. Check token signature
        if !self.verify_token_signature(token)? {
            return Ok(false);
        }
        
        // 3. Check token expiration
        if self.is_token_expired(token)? {
            return Ok(false);
        }
        
        // 4. Check token scope
        if !self.token_manager.has_required_scope(token, &["iot:read", "iot:write"])? {
            return Ok(false);
        }
        
        Ok(true)
    }
    
    fn verify_biometric(&self, device_id: &DeviceId, bio_data: &BiometricData) -> Result<bool, AuthError> {
        // 1. Check biometric data format
        if !self.is_valid_biometric_format(bio_data)? {
            return Ok(false);
        }
        
        // 2. Compare with stored biometric template
        let stored_template = self.biometric_auth.get_template(device_id)?;
        let similarity = self.calculate_biometric_similarity(bio_data, &stored_template)?;
        
        // 3. Check similarity threshold
        Ok(similarity >= BIOMETRIC_SIMILARITY_THRESHOLD)
    }
}

pub struct DataEncryption {
    pub encryption_key_manager: EncryptionKeyManager, // Key management
    pub encryption_algorithm: EncryptionAlgorithm, // Encryption algorithm
    pub data_integrity: DataIntegrity,        // Data integrity
}

impl DataEncryption {
    pub fn encrypt_data(&self, data: &IoTData, recipient: &[u8; 32]) -> Result<EncryptedData, EncryptionError> {
        // 1. Get encryption key
        let key = self.encryption_key_manager.get_encryption_key(recipient)?;
        
        // 2. Encrypt data
        let encrypted_payload = self.encryption_algorithm.encrypt(&self.serialize_data(data)?, &key)?;
        
        // 3. Create integrity hash
        let integrity_hash = self.data_integrity.create_hash(&encrypted_payload)?;
        
        // 4. Create encrypted data structure
        let encrypted_data = EncryptedData {
            version: 1,
            algorithm: self.encryption_algorithm.get_algorithm_id(),
            key_id: key.id,
            encrypted_payload,
            integrity_hash,
            metadata: HashMap::new(),
        };
        
        Ok(encrypted_data)
    }
    
    pub fn decrypt_data(&self, encrypted_data: &EncryptedData, recipient: &[u8; 32]) -> Result<IoTData, DecryptionError> {
        // 1. Verify integrity
        if !self.data_integrity.verify_hash(&encrypted_data.encrypted_payload, &encrypted_data.integrity_hash)? {
            return Err(DecryptionError::IntegrityCheckFailed);
        }
        
        // 2. Get decryption key
        let key = self.encryption_key_manager.get_decryption_key(&encrypted_data.key_id, recipient)?;
        
        // 3. Decrypt data
        let decrypted_payload = self.encryption_algorithm.decrypt(&encrypted_data.encrypted_payload, &key)?;
        
        // 4. Deserialize data
        let data = self.deserialize_data(&decrypted_payload)?;
        
        Ok(data)
    }
}
```

## Configuration and Deployment

### IoT Configuration
```rust
pub struct IoTConfig {
    pub connector_config: ConnectorConfig,   // Connector configuration
    pub device_config: DeviceConfig,         // Device configuration
    pub security_config: SecurityConfig,     // Security configuration
    pub performance_config: PerformanceConfig, // Performance configuration
}

impl Default for IoTConfig {
    fn default() -> Self {
        Self {
            connector_config: ConnectorConfig::default(),
            device_config: DeviceConfig::default(),
            security_config: SecurityConfig::default(),
            performance_config: PerformanceConfig::default(),
        }
    }
}
```

This IoT connector architecture provides a comprehensive foundation for blockchain-IoT integration with edge computing, batch processing, and enterprise-grade security features.
