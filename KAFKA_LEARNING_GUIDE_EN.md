# Apache Kafka Expert Learning Guide
## Educational Agent with Latest Ecosystem News (August 2026)

---

## 📰 Latest Kafka Ecosystem News

### 🚀 Apache Kafka 4.4.0 - Upcoming Release (September 2026)

**Current status**: In stabilization period
- **Code Freeze**: August 12, 2026
- **Expected release**: No earlier than September 9, 2026
- **Included KIPs**: 
  - KIP-1071: Streams Rebalance Protocol
  - KIP-1356: IQv2 for headers-aware state stores
  - Multiple improvements to KRaft and Tiered Storage

### ✅ Apache Kafka 4.3.1 (June 2026) - Current Stable Version

**Critical bugfix release**:
- **Main fix**: Memory leak in Kafka Streams with RocksDB causing OutOfMemory errors (KAFKA-20616)
- **Recommendation**: Upgrade immediately if using Kafka Streams
- 23 contributors involved in this release

### 🎯 Apache Kafka 4.3.0 (May 2026) - Latest Major Release

**Highlighted features**:
- 25 KIPs implemented
- Over 600 commits since 4.2.0
- **Important deprecation**: Kafka Streams Scala will be removed in version 5.0
- Significant improvements across all areas: Core, Streams, Connect

### 📊 Kafka 3.9.0 - Last Version of 3.x Line

**IMPORTANT**: This is the last version supporting ZooKeeper
- **Tiered Storage** is now production-ready
- ZooKeeper to KRaft migration fully mature
- Starting with Kafka 4.0, ZooKeeper is NO longer available

---

## 🏗️ Architecture and Core Concepts

### 1. KRaft: Kafka's New Heart

**What is KRaft?**
- KRaft (Kafka Raft) is Kafka's native consensus protocol
- **Completely replaces ZooKeeper** since Kafka 4.0
- Based on the Raft algorithm for metadata management

**KRaft Advantages**:
- ✅ Simplified architecture (fewer components)
- ✅ Better scalability (millions of partitions)
- ✅ Faster failure recovery
- ✅ Simpler deployments (no ZooKeeper cluster)
- ✅ Dynamic quorum (add/remove controllers without downtime)

**How to run Kafka in KRaft mode**:
```bash
# See config/kraft/README.md for detailed instructions
./bin/kafka-storage.sh format -t <cluster-id> -c config/kraft/server.properties
./bin/kafka-server-start.sh config/kraft/server.properties
```

**⚠️ Critical Migration Path**:
If you're on pre-3.7 versions with ZooKeeper:
1. Update to Kafka 3.9.x
2. Perform ZooKeeper to KRaft migration
3. Update to Kafka 4.x (no ZooKeeper support)

**No direct migration in 4.0+** - you must migrate in 3.9.x

---

### 2. Tiered Storage: Scalable Storage

**Status**: Production-ready since Kafka 3.9.0

**What is Tiered Storage?**
- Hierarchical log storage
- Historical data automatically moved to object stores (S3, Azure Blob, GCS)
- Reduces local storage costs
- Increases retention capacity without additional hardware

**New features in 2026**:

#### KIP-956: Tiered Storage Quotas
```properties
# Limit upload rate to remote storage
remote.log.manager.upload.quota.bytes.per.second=10485760  # 10MB/s

# Limit download rate from remote storage
remote.log.manager.fetch.quota.bytes.per.second=20971520  # 20MB/s
```

**Purpose**: Avoid producer/consumer performance degradation during massive transfers

#### KIP-950: Dynamic Tiered Storage Disablement
```bash
# Make remote logs read-only (no more copies)
kafka-configs.sh --alter --entity-type topics --entity-name my-topic \
  --add-config remote.storage.enable=true,remote.log.copy.disable=true

# Completely disable and delete remote logs
kafka-configs.sh --alter --entity-type topics --entity-name my-topic \
  --add-config remote.storage.enable=false,remote.log.delete.on.disable=true
```

**Basic configuration**:

```properties
# Enable on broker
remote.log.storage.system.enable=true
remote.log.storage.manager.class.name=org.apache.kafka.server.log.remote.storage.RemoteLogStorageManager
remote.log.metadata.manager.class.name=org.apache.kafka.server.log.remote.metadata.storage.TopicBasedRemoteLogMetadataManager

# Topic-level configuration
remote.storage.enable=true
local.retention.ms=86400000  # 1 day local
local.retention.bytes=1073741824  # 1GB local
retention.ms=604800000  # 7 days total
```

**⚠️ Important limitations**:
- ❌ Does NOT support compacted topics
- ❌ Topics created before Kafka 2.8.0 require manual steps
- ℹ️ Clients < 3.0 cannot perform Tiered Storage administrative actions
- ℹ️ Works with all clients for production/consumption

---

### 3. Kafka Streams: Stream Processing

**KIP-1071: New Rebalance Protocol**
- Progressive implementation since Kafka 4.2
- Significant improvement in rebalance times
- Support for warmup tasks
- Support for static group membership

**KIP-1049: Log Summary Control**
```java
Properties props = new Properties();
props.put(StreamsConfig.APPLICATION_ID_CONFIG, "my-app");
props.put("log.summary.interval.ms", "60000"); // Log summary every minute
```

**KIP-1033: Enhanced Exception Handler**
```java
streamsConfig.put(
    StreamsConfig.DEFAULT_PROCESSING_EXCEPTION_HANDLER_CLASS_CONFIG,
    CustomExceptionHandler.class
);
```

**⚠️ Important Deprecation**:
- **kafka-streams-scala** will be removed in Kafka 5.0
- Migrate to Java Kafka Streams DSL

---

### 4. Kafka Connect: Data Integration

**KIP-1017: Health Check Endpoint**
```bash
# New REST endpoint to verify worker health
curl http://localhost:8083/health
```

**Response**:
```json
{
  "status": "healthy",
  "workers": [
    {
      "id": "worker-1",
      "status": "healthy"
    }
  ]
}
```

**KIP-1031: Offset Translation Control in MirrorSourceConnector**
```properties
# Disable offset synchronization
emit.offset-syncs.enabled=false
```

**KIP-1040: Improved Nullable Value Handling**
- Additional configurations for InsertField, ExtractField transformations
- Better control over null fields in transformations

---

## 🎓 Learning Paths by Role

### 📘 For Application Developers

#### Level 1: Fundamentals
1. **Basic concepts**:
   - Producers and Consumers
   - Topics and Partitions
   - Offsets and Consumer Groups
   - Serialization/Deserialization

2. **First producer**:
```java
Properties props = new Properties();
props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, 
          "org.apache.kafka.common.serialization.StringSerializer");
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, 
          "org.apache.kafka.common.serialization.StringSerializer");

// Important performance configurations
props.put(ProducerConfig.ACKS_CONFIG, "all");
props.put(ProducerConfig.RETRIES_CONFIG, 3);
props.put(ProducerConfig.LINGER_MS_CONFIG, 1);
props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");

KafkaProducer<String, String> producer = new KafkaProducer<>(props);

ProducerRecord<String, String> record = 
    new ProducerRecord<>("my-topic", "key", "value");

producer.send(record, (metadata, exception) -> {
    if (exception != null) {
        exception.printStackTrace();
    } else {
        System.out.printf("Sent to partition %d, offset %d%n",
            metadata.partition(), metadata.offset());
    }
});

producer.close();
```

3. **First consumer**:
```java
Properties props = new Properties();
props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(ConsumerConfig.GROUP_ID_CONFIG, "my-consumer-group");
props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, 
          "org.apache.kafka.common.serialization.StringDeserializer");
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, 
          "org.apache.kafka.common.serialization.StringDeserializer");

// Important configurations
props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, "500");

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Collections.singletonList("my-topic"));

try {
    while (true) {
        ConsumerRecords<String, String> records = 
            consumer.poll(Duration.ofMillis(100));
        
        for (ConsumerRecord<String, String> record : records) {
            System.out.printf("offset = %d, key = %s, value = %s%n",
                record.offset(), record.key(), record.value());
        }
        
        consumer.commitSync();
    }
} finally {
    consumer.close();
}
```

#### Level 2: Advanced Patterns
1. **Exactly-Once Semantics (EOS)**
2. **Transactions**
3. **Idempotence**
4. **Custom Serializers/Deserializers**
5. **Error handling and retries**

#### Level 3: Kafka Streams
1. **Processing topologies**
2. **Stateful vs Stateless operations**
3. **Windowing**
4. **Joins**
5. **Interactive Queries**

---

### 🛠️ For DevOps/SRE Engineers

#### Level 1: Deployment and Configuration

**Quick installation with Docker Compose**:
```yaml
version: '3'
services:
  kafka:
    image: apache/kafka:latest
    container_name: kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: 'broker,controller'
      KAFKA_CONTROLLER_QUORUM_VOTERS: '1@kafka:9093'
      KAFKA_LISTENERS: 'PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093'
      KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://localhost:9092'
      KAFKA_CONTROLLER_LISTENER_NAMES: 'CONTROLLER'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: 'CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT'
      KAFKA_LOG_DIRS: '/tmp/kraft-combined-logs'
      CLUSTER_ID: 'MkU3OEVBNTcwNTJENDM2Qk'
```

**Critical production configurations**:

```properties
# Replication
default.replication.factor=3
min.insync.replicas=2

# Retention
log.retention.hours=168  # 7 days
log.retention.bytes=1073741824  # 1GB per partition

# Performance
num.network.threads=8
num.io.threads=16
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600

# Segments
log.segment.bytes=1073741824
log.segment.ms=604800000

# Compression
compression.type=producer
```

#### Level 2: Monitoring and Observability

**Critical metrics to monitor**:

1. **Broker health**:
   - `UnderReplicatedPartitions` (should be 0)
   - `OfflinePartitionsCount` (should be 0)
   - `ActiveControllerCount` (should be 1)
   - `RequestHandlerAvgIdlePercent` (> 20%)

2. **Producer metrics**:
   - `record-send-rate`
   - `request-latency-avg`
   - `record-error-rate`
   - `buffer-available-bytes`

3. **Consumer metrics**:
   - `records-lag-max`
   - `records-consumed-rate`
   - `fetch-latency-avg`
   - `commit-latency-avg`

**Export metrics with JMX Exporter for Prometheus**:
```bash
# Download JMX Exporter
wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.20.0/jmx_prometheus_javaagent-0.20.0.jar

# Start Kafka with JMX Exporter
export KAFKA_OPTS="-javaagent:./jmx_prometheus_javaagent-0.20.0.jar=7071:kafka-broker.yml"
./bin/kafka-server-start.sh config/server.properties
```

#### Level 3: Advanced Operations

**1. ZooKeeper to KRaft Migration (Kafka 3.9.x)**:
```bash
# 1. Prepare KRaft configuration
./bin/kafka-storage.sh random-uuid

# 2. Start migration process
./bin/kafka-metadata-quorum.sh --bootstrap-server localhost:9092 describe --status

# 3. Migrate metadata
# See official documentation: https://kafka.apache.org/documentation/#kraft_zk_migration
```

**2. Dynamic KRaft Quorum Management (KIP-853)**:
```bash
# Add new controller without downtime
./bin/kafka-metadata-quorum.sh --bootstrap-controller localhost:9093 \
  --command-config config/kraft/controller.properties \
  add-controller --controller-id 4 --controller-host kafka4 --controller-port 9093

# Remove controller
./bin/kafka-metadata-quorum.sh --bootstrap-controller localhost:9093 \
  --command-config config/kraft/controller.properties \
  remove-controller --controller-id 4
```

**3. Handling Offline Partitions**:
```bash
# Identify offline partitions
./bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --under-replicated-partitions

# Reassign replicas
cat reassignment.json
{
  "version": 1,
  "partitions": [
    {"topic": "my-topic", "partition": 0, "replicas": [1, 2, 3]}
  ]
}

./bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment.json --execute

# Monitor progress
./bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment.json --verify
```

---

### 🏛️ For Data Architects

#### Design Patterns

**1. Event Sourcing**:
- Store state as sequence of events
- Topics as immutable event log
- State reconstruction via replay

**2. CQRS (Command Query Responsibility Segregation)**:
- Separate writes (commands) from reads (queries)
- Kafka as backbone between write and read models
- Kafka Streams to materialize views

**3. Saga Pattern**:
- Distributed transactions via events
- Compensation on failure
- Coordination via event topics

#### Topic Design

**Partitioning strategies**:

```java
// Custom Partitioner
public class UserRegionPartitioner implements Partitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                        Object value, byte[] valueBytes, Cluster cluster) {
        int numPartitions = cluster.partitionCountForTopic(topic);
        
        // Partition by user region
        String userId = (String) key;
        String region = extractRegion(userId);
        
        return Math.abs(region.hashCode()) % numPartitions;
    }
}

props.put(ProducerConfig.PARTITIONER_CLASS_CONFIG, 
          UserRegionPartitioner.class.getName());
```

**Optimal number of partitions**:
```
Partitions = max(T/P, T/C)

Where:
T = Target throughput
P = Throughput per partition (producer)
C = Throughput per partition (consumer)
```

**Retention strategies**:
1. **Time-based**: `retention.ms`
2. **Size-based**: `retention.bytes`
3. **Compaction**: `cleanup.policy=compact` for state stores

---

## 🔥 Best Practices 2026

### 1. Producer Configuration

```properties
# Durability
acks=all
enable.idempotence=true
max.in.flight.requests.per.connection=5

# Performance
compression.type=snappy
batch.size=32768
linger.ms=10
buffer.memory=33554432

# Retries
retries=2147483647
delivery.timeout.ms=120000
request.timeout.ms=30000

# Transactions (if applicable)
transactional.id=my-transactional-id
transaction.timeout.ms=60000
```

### 2. Consumer Configuration

```properties
# Groups
group.id=my-consumer-group
group.instance.id=consumer-1  # Static membership

# Offsets
enable.auto.commit=false
auto.offset.reset=earliest
isolation.level=read_committed  # To read only completed transactions

# Performance
max.poll.records=500
max.poll.interval.ms=300000
session.timeout.ms=45000
heartbeat.interval.ms=3000

# Fetch
fetch.min.bytes=1
fetch.max.wait.ms=500
max.partition.fetch.bytes=1048576
```

### 3. Security

**SSL/TLS**:
```properties
security.protocol=SSL
ssl.truststore.location=/var/private/ssl/kafka.client.truststore.jks
ssl.truststore.password=test1234
ssl.keystore.location=/var/private/ssl/kafka.client.keystore.jks
ssl.keystore.password=test1234
ssl.key.password=test1234
```

**SASL/SCRAM**:
```properties
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
  username="admin" \
  password="admin-secret";
```

**ACLs**:
```bash
# Create user
./bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --alter --add-config 'SCRAM-SHA-512=[password=secret]' \
  --entity-type users --entity-name alice

# Add ACL
./bin/kafka-acls.sh --bootstrap-server localhost:9092 \
  --add --allow-principal User:alice \
  --operation Read --operation Write \
  --topic my-topic
```

### 4. Common Troubleshooting

#### Issue: High Consumer Lag

**Diagnosis**:
```bash
./bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group my-consumer-group
```

**Solutions**:
1. Increase number of consumers in the group
2. Increase `max.poll.records` if processing is fast
3. Optimize processing logic
4. Consider Kafka Streams for parallel processing

#### Issue: Under-Replicated Partitions

**Diagnosis**:
```bash
./bin/kafka-topics.sh --bootstrap-server localhost:9092 \
  --describe --under-replicated-partitions
```

**Solutions**:
1. Verify broker health
2. Review logs of problematic brokers
3. Check network between brokers
4. Consider increasing `replica.lag.time.max.ms`

#### Issue: Memory Leaks (especially Kafka Streams)

**⚠️ Upgrade to Kafka 4.3.1 immediately** if using Kafka Streams (fixes KAFKA-20616)

**Prevention**:
- Monitor heap usage constantly
- Configure JVM correctly:
```bash
export KAFKA_HEAP_OPTS="-Xms6g -Xmx6g -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35"
```

---

## 🌍 Ecosystem and Tools

### Schema Registry (Confluent)
- Centralized schema management
- Schema compatibility
- Avro, Protobuf, JSON Schema

### Kafka Connect
- 100+ certified connectors
- Source connectors: PostgreSQL, MySQL, MongoDB, etc.
- Sink connectors: Elasticsearch, S3, BigQuery, etc.

### ksqlDB
- SQL streaming over Kafka
- Creation of streams and tables
- Joins, aggregations, windowing

### Kafka UI Tools
- **Conduktor**: Complete management interface
- **Kafka UI (Provectus)**: Open source, lightweight
- **Kafdrop**: Simple, web-based

### Monitoring
- **Prometheus + Grafana**: Metrics and dashboards
- **Kafka Exporter**: Official metrics exporter
- **Burrow**: Specialized consumer lag monitoring

---

## 📚 Learning Resources

### Official Documentation
- **Official website**: https://kafka.apache.org
- **Documentation**: https://kafka.apache.org/documentation/
- **KIPs**: https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Improvement+Proposals

### Recommended Courses
1. **Confluent Developer**: Apache Kafka Fundamentals
2. **LinkedIn Learning**: Learning Apache Kafka
3. **Udemy**: Apache Kafka Series

### Books
1. **"Kafka: The Definitive Guide"** (O'Reilly) - 2nd Edition
2. **"Kafka Streams in Action"** (Manning)
3. **"Designing Event-Driven Systems"** (O'Reilly)

### Community
- **Mailing lists**: https://kafka.apache.org/contact
- **Slack**: Confluent Community Slack
- **Stack Overflow**: Tag `apache-kafka`
- **GitHub**: https://github.com/apache/kafka

### Blogs and News
- **Official Kafka Blog**: https://kafka.apache.org/blog/
- **Confluent Blog**: https://www.confluent.io/blog/
- **Kafka Monthly Digest**: Red Hat Developer

---

## 🎯 Suggested Learning Roadmap

### Week 1-2: Fundamentals
- [ ] Understand basic architecture
- [ ] Install Kafka locally (KRaft mode)
- [ ] Create first producer and consumer
- [ ] Experiment with CLI commands

### Week 3-4: Intermediate Development
- [ ] Implement producers with transactions
- [ ] Implement consumers with batch processing
- [ ] Understand consumer groups and rebalancing
- [ ] Implement custom serializers

### Week 5-6: Kafka Streams
- [ ] Create first Streams topology
- [ ] Implement stateful operations
- [ ] Work with windowing
- [ ] Perform joins between streams

### Week 7-8: Operations
- [ ] Set up multi-broker cluster
- [ ] Configure security (SSL/SASL)
- [ ] Implement monitoring
- [ ] Practice troubleshooting

### Week 9-10: Advanced
- [ ] Implement Tiered Storage
- [ ] Configure Kafka Connect
- [ ] Architecture patterns (Event Sourcing, CQRS)
- [ ] Performance tuning

### Week 11-12: Production
- [ ] Migrate from ZooKeeper to KRaft (in test environment)
- [ ] Implement disaster recovery
- [ ] Capacity planning
- [ ] Prepare for certification (optional)

---

## ⚠️ Critical Aspects to Remember

### Migrations and Upgrades

1. **No way back from KRaft to ZooKeeper**
2. **ZK→KRaft migration must be done in 3.9.x before upgrading to 4.x**
3. **Java 8 no longer supported in Kafka 4.x** (minimum Java 11)
4. **Scala 2.12 no longer supported in Kafka 4.x** (minimum Scala 2.13)

### Production Configuration

```bash
# ALWAYS configure temporary folders for compression
export KAFKA_OPTS="-Dorg.xerial.snappy.tempdir=/opt/kafka/tmp \
                    -Dio.airlift.compress.zstd.ZstdTempFolder=/opt/kafka/tmp"
```

**Note**: If `/tmp` has `noexec`, Kafka WILL NOT start without this configuration.

### Continuous Monitoring

Mandatory alerts:
1. `UnderReplicatedPartitions > 0`
2. `OfflinePartitionsCount > 0`
3. `ActiveControllerCount != 1`
4. Consumer lag > defined threshold
5. Disk usage > 80%

---

## 🔮 Upcoming Features (Post 4.4.0)

### KIP-848: Consumer Rebalance Protocol (Next Gen)
- More efficient incremental rebalancing
- Modern event-loop in Group Coordinator
- Significant reduction in pause time

### KIP-1352: Ranges and Range Aggregations
- Event-based aggregations (not just time)
- Windows before/after an anchor
- Greater flexibility in Streams DSL

### Containerization Improvements
- Better Kubernetes support
- Native health checks
- Metrics improvements

---

## 💡 Final Learning Tips

1. **Practice with the official repository**: You're already in it - experiment with the code
2. **Read the KIPs**: Understand the "why" behind decisions
3. **Contribute**: Kafka is open source - report bugs, improve documentation
4. **Follow the official blog**: Stay up to date with releases
5. **Join the community**: Mailing lists are very active
6. **Experiment with new versions**: Use feature branches to test
7. **Don't fear the source code**: It's readable and well-documented Java/Scala

---

## 📞 Contact and Contribution

- **Website**: https://kafka.apache.org
- **GitHub**: https://github.com/apache/kafka
- **Jira**: https://issues.apache.org/jira/browse/KAFKA
- **Mailing Lists**: https://kafka.apache.org/contact

### How to Contribute
See `CONTRIBUTING.md` in this repository and:
https://kafka.apache.org/contributing.html

---

**Last updated**: August 2026  
**Document version**: 1.0  
**Based on**: Apache Kafka 4.3.1 (stable) and 4.4.0 (in development)

---

Welcome to the Kafka ecosystem! 🚀
