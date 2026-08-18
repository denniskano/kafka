# Guía de Aprendizaje Experta en Apache Kafka
## Agente Educativo con las Últimas Noticias del Ecosistema (Agosto 2026)

---

## 📰 Últimas Noticias del Ecosistema Kafka

### 🚀 Apache Kafka 4.4.0 - Próximo Lanzamiento (Septiembre 2026)

**Estado actual**: En periodo de estabilización
- **Code Freeze**: 12 de agosto de 2026
- **Lanzamiento esperado**: No antes del 9 de septiembre de 2026
- **KIPs incluidos**: 
  - KIP-1071: Protocolo de Rebalanceo de Streams
  - KIP-1356: IQv2 para state stores conscientes de headers
  - Múltiples mejoras a KRaft y Tiered Storage

### ✅ Apache Kafka 4.3.1 (Junio 2026) - Versión Actual Estable

**Lanzamiento crítico de corrección de bugs**:
- **Corrección principal**: Memory leak en Kafka Streams con RocksDB que causaba OutOfMemory errors (KAFKA-20616)
- **Recomendación**: Actualizar inmediatamente si usas Kafka Streams
- 23 contribuidores involucrados en esta versión

### 🎯 Apache Kafka 4.3.0 (Mayo 2026) - Última Release Mayor

**Características destacadas**:
- 25 KIPs implementados
- Más de 600 commits desde 4.2.0
- **Deprecación importante**: Kafka Streams Scala será removido en versión 5.0
- Mejoras significativas en todas las áreas: Core, Streams, Connect

### 📊 Kafka 3.9.0 - Última Versión de la Línea 3.x

**IMPORTANTE**: Esta es la última versión que soporta ZooKeeper
- **Tiered Storage** ahora es production-ready
- Migración de ZooKeeper a KRaft completamente madura
- A partir de Kafka 4.0, ZooKeeper ya NO está disponible

---

## 🏗️ Arquitectura y Conceptos Fundamentales

### 1. KRaft: El Nuevo Corazón de Kafka

**¿Qué es KRaft?**
- KRaft (Kafka Raft) es el protocolo de consenso nativo de Kafka
- **Reemplaza completamente a ZooKeeper** desde Kafka 4.0
- Basado en el algoritmo Raft para gestión de metadatos

**Ventajas de KRaft**:
- ✅ Arquitectura simplificada (menos componentes)
- ✅ Mejor escalabilidad (millones de particiones)
- ✅ Recuperación más rápida de fallos
- ✅ Despliegues más simples (sin ZooKeeper cluster)
- ✅ Quorum dinámico (añadir/remover controladores sin downtime)

**Cómo ejecutar Kafka en modo KRaft**:
```bash
# Ver config/kraft/README.md para instrucciones detalladas
./bin/kafka-storage.sh format -t <cluster-id> -c config/kraft/server.properties
./bin/kafka-server-start.sh config/kraft/server.properties
```

**⚠️ Ruta de Migración Crítica**:
Si estás en versiones pre-3.7 con ZooKeeper:
1. Actualizar a Kafka 3.9.x
2. Realizar migración de ZooKeeper a KRaft
3. Actualizar a Kafka 4.x (sin soporte ZooKeeper)

**No hay migración directa en 4.0+** - debes migrar en 3.9.x

---

### 2. Tiered Storage: Almacenamiento Escalable

**Estado**: Production-ready desde Kafka 3.9.0

**¿Qué es Tiered Storage?**
- Almacenamiento jerárquico de logs
- Datos históricos se mueven automáticamente a object stores (S3, Azure Blob, GCS)
- Reduce costos de almacenamiento local
- Aumenta capacidad de retención sin hardware adicional

**Nuevas características en 2026**:

#### KIP-956: Quotas de Tiered Storage
```properties
# Limita la tasa de subida a remote storage
remote.log.manager.upload.quota.bytes.per.second=10485760  # 10MB/s

# Limita la tasa de descarga desde remote storage
remote.log.manager.fetch.quota.bytes.per.second=20971520  # 20MB/s
```

**Propósito**: Evitar degradación de rendimiento de productores/consumidores durante transferencias masivas

#### KIP-950: Deshabilitación Dinámica de Tiered Storage
```bash
# Hacer logs remotos de solo lectura (no más copias)
kafka-configs.sh --alter --entity-type topics --entity-name mi-topic \
  --add-config remote.storage.enable=true,remote.log.copy.disable=true

# Deshabilitar completamente y eliminar logs remotos
kafka-configs.sh --alter --entity-type topics --entity-name mi-topic \
  --add-config remote.storage.enable=false,remote.log.delete.on.disable=true
```

**Configuración básica**:

```properties
# Habilitar en el broker
remote.log.storage.system.enable=true
remote.log.storage.manager.class.name=org.apache.kafka.server.log.remote.storage.RemoteLogStorageManager
remote.log.metadata.manager.class.name=org.apache.kafka.server.log.remote.metadata.storage.TopicBasedRemoteLogMetadataManager

# Configuración a nivel de tópico
remote.storage.enable=true
local.retention.ms=86400000  # 1 día local
local.retention.bytes=1073741824  # 1GB local
retention.ms=604800000  # 7 días total
```

**⚠️ Limitaciones importantes**:
- ❌ NO soporta tópicos compactados
- ❌ Tópicos creados antes de Kafka 2.8.0 requieren pasos manuales
- ℹ️ Clientes < 3.0 no pueden realizar acciones administrativas de Tiered Storage
- ℹ️ Funciona con todos los clientes para producción/consumo

---

### 3. Kafka Streams: Procesamiento de Streams

**KIP-1071: Nuevo Protocolo de Rebalanceo**
- Implementación progresiva desde Kafka 4.2
- Mejora significativa en tiempos de rebalanceo
- Soporte para warmup tasks
- Soporte para static group membership

**KIP-1049: Control de Log Summary**
```java
Properties props = new Properties();
props.put(StreamsConfig.APPLICATION_ID_CONFIG, "mi-app");
props.put("log.summary.interval.ms", "60000"); // Log summary cada minuto
```

**KIP-1033: Mejorado Exception Handler**
```java
streamsConfig.put(
    StreamsConfig.DEFAULT_PROCESSING_EXCEPTION_HANDLER_CLASS_CONFIG,
    CustomExceptionHandler.class
);
```

**⚠️ Deprecación Importante**:
- **kafka-streams-scala** será removido en Kafka 5.0
- Migrar a Kafka Streams DSL de Java

---

### 4. Kafka Connect: Integración de Datos

**KIP-1017: Health Check Endpoint**
```bash
# Nuevo endpoint REST para verificar salud de workers
curl http://localhost:8083/health
```

**Respuesta**:
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

**KIP-1031: Control de Traducción de Offsets en MirrorSourceConnector**
```properties
# Deshabilitar sincronización de offsets
emit.offset-syncs.enabled=false
```

**KIP-1040: Manejo Mejorado de Valores Nullable**
- Configuraciones adicionales para transformaciones InsertField, ExtractField
- Mejor control sobre campos null en transformaciones

---

## 🎓 Rutas de Aprendizaje por Rol

### 📘 Para Desarrolladores de Aplicaciones

#### Nivel 1: Fundamentos
1. **Conceptos básicos**:
   - Productores y Consumidores
   - Tópicos y Particiones
   - Offsets y Consumer Groups
   - Serialización/Deserialización

2. **Primer productor**:
```java
Properties props = new Properties();
props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, 
          "org.apache.kafka.common.serialization.StringSerializer");
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, 
          "org.apache.kafka.common.serialization.StringSerializer");

// Configuraciones importantes de rendimiento
props.put(ProducerConfig.ACKS_CONFIG, "all");
props.put(ProducerConfig.RETRIES_CONFIG, 3);
props.put(ProducerConfig.LINGER_MS_CONFIG, 1);
props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");

KafkaProducer<String, String> producer = new KafkaProducer<>(props);

ProducerRecord<String, String> record = 
    new ProducerRecord<>("mi-topico", "clave", "valor");

producer.send(record, (metadata, exception) -> {
    if (exception != null) {
        exception.printStackTrace();
    } else {
        System.out.printf("Enviado a partition %d, offset %d%n",
            metadata.partition(), metadata.offset());
    }
});

producer.close();
```

3. **Primer consumidor**:
```java
Properties props = new Properties();
props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(ConsumerConfig.GROUP_ID_CONFIG, "mi-grupo-consumidor");
props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, 
          "org.apache.kafka.common.serialization.StringDeserializer");
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, 
          "org.apache.kafka.common.serialization.StringDeserializer");

// Configuraciones importantes
props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, "500");

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Collections.singletonList("mi-topico"));

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

#### Nivel 2: Patrones Avanzados
1. **Exactly-Once Semantics (EOS)**
2. **Transacciones**
3. **Idempotencia**
4. **Custom Serializers/Deserializers**
5. **Manejo de errores y reintentos**

#### Nivel 3: Kafka Streams
1. **Topologías de procesamiento**
2. **Stateful vs Stateless operations**
3. **Windowing**
4. **Joins**
5. **Interactive Queries**

---

### 🛠️ Para Ingenieros DevOps/SRE

#### Nivel 1: Despliegue y Configuración

**Instalación rápida con Docker Compose**:
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

**Configuraciones críticas de producción**:

```properties
# Replicación
default.replication.factor=3
min.insync.replicas=2

# Retención
log.retention.hours=168  # 7 días
log.retention.bytes=1073741824  # 1GB por partición

# Performance
num.network.threads=8
num.io.threads=16
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600

# Segmentos
log.segment.bytes=1073741824
log.segment.ms=604800000

# Compresión
compression.type=producer
```

#### Nivel 2: Monitoreo y Observabilidad

**Métricas críticas a monitorear**:

1. **Broker health**:
   - `UnderReplicatedPartitions` (debe ser 0)
   - `OfflinePartitionsCount` (debe ser 0)
   - `ActiveControllerCount` (debe ser 1)
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

**Exportar métricas con JMX Exporter para Prometheus**:
```bash
# Descargar JMX Exporter
wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.20.0/jmx_prometheus_javaagent-0.20.0.jar

# Iniciar Kafka con JMX Exporter
export KAFKA_OPTS="-javaagent:./jmx_prometheus_javaagent-0.20.0.jar=7071:kafka-broker.yml"
./bin/kafka-server-start.sh config/server.properties
```

#### Nivel 3: Operaciones Avanzadas

**1. Migración de ZooKeeper a KRaft (Kafka 3.9.x)**:
```bash
# 1. Preparar configuración KRaft
./bin/kafka-storage.sh random-uuid

# 2. Iniciar el proceso de migración
./bin/kafka-metadata-quorum.sh --bootstrap-server localhost:9092 describe --status

# 3. Migrar metadatos
# Ver documentación oficial: https://kafka.apache.org/documentation/#kraft_zk_migration
```

**2. Gestión dinámica de Quorum KRaft (KIP-853)**:
```bash
# Añadir nuevo controlador sin downtime
./bin/kafka-metadata-quorum.sh --bootstrap-controller localhost:9093 \
  --command-config config/kraft/controller.properties \
  add-controller --controller-id 4 --controller-host kafka4 --controller-port 9093

# Remover controlador
./bin/kafka-metadata-quorum.sh --bootstrap-controller localhost:9093 \
  --command-config config/kraft/controller.properties \
  remove-controller --controller-id 4
```

**3. Manejo de Partitions Offline**:
```bash
# Identificar particiones offline
./bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --under-replicated-partitions

# Reasignar réplicas
cat reassignment.json
{
  "version": 1,
  "partitions": [
    {"topic": "mi-topico", "partition": 0, "replicas": [1, 2, 3]}
  ]
}

./bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment.json --execute

# Monitorear progreso
./bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassignment.json --verify
```

---

### 🏛️ Para Arquitectos de Datos

#### Patrones de Diseño

**1. Event Sourcing**:
- Almacenar estado como secuencia de eventos
- Tópicos como event log inmutable
- Reconstrucción de estado mediante replay

**2. CQRS (Command Query Responsibility Segregation)**:
- Separar escrituras (commands) de lecturas (queries)
- Kafka como backbone entre write y read models
- Kafka Streams para materializar vistas

**3. Saga Pattern**:
- Transacciones distribuidas mediante eventos
- Compensación en caso de fallo
- Coordinación mediante tópicos de eventos

#### Diseño de Tópicos

**Estrategias de particionamiento**:

```java
// Custom Partitioner
public class UserRegionPartitioner implements Partitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                        Object value, byte[] valueBytes, Cluster cluster) {
        int numPartitions = cluster.partitionCountForTopic(topic);
        
        // Particionar por región del usuario
        String userId = (String) key;
        String region = extractRegion(userId);
        
        return Math.abs(region.hashCode()) % numPartitions;
    }
}

props.put(ProducerConfig.PARTITIONER_CLASS_CONFIG, 
          UserRegionPartitioner.class.getName());
```

**Número óptimo de particiones**:
```
Particiones = max(T/P, T/C)

Donde:
T = Throughput objetivo
P = Throughput por partición (productor)
C = Throughput por partición (consumidor)
```

**Estrategias de retención**:
1. **Time-based**: `retention.ms`
2. **Size-based**: `retention.bytes`
3. **Compaction**: `cleanup.policy=compact` para state stores

---

## 🔥 Best Practices 2026

### 1. Configuración de Productores

```properties
# Durabilidad
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

# Transacciones (si aplica)
transactional.id=mi-transactional-id
transaction.timeout.ms=60000
```

### 2. Configuración de Consumidores

```properties
# Grupos
group.id=mi-grupo-consumidor
group.instance.id=consumer-1  # Static membership

# Offsets
enable.auto.commit=false
auto.offset.reset=earliest
isolation.level=read_committed  # Para leer solo transacciones completadas

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

### 3. Seguridad

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
# Crear usuario
./bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --alter --add-config 'SCRAM-SHA-512=[password=secret]' \
  --entity-type users --entity-name alice

# Añadir ACL
./bin/kafka-acls.sh --bootstrap-server localhost:9092 \
  --add --allow-principal User:alice \
  --operation Read --operation Write \
  --topic mi-topico
```

### 4. Troubleshooting Común

#### Problema: Consumer Lag Alto

**Diagnóstico**:
```bash
./bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group mi-grupo-consumidor
```

**Soluciones**:
1. Aumentar número de consumidores en el grupo
2. Aumentar `max.poll.records` si procesamiento es rápido
3. Optimizar lógica de procesamiento
4. Considerar Kafka Streams para procesamiento paralelo

#### Problema: Under-Replicated Partitions

**Diagnóstico**:
```bash
./bin/kafka-topics.sh --bootstrap-server localhost:9092 \
  --describe --under-replicated-partitions
```

**Soluciones**:
1. Verificar salud de brokers
2. Revisar logs de brokers con problemas
3. Verificar red entre brokers
4. Considerar aumentar `replica.lag.time.max.ms`

#### Problema: Memory Leaks (especialmente Kafka Streams)

**⚠️ Actualizar a Kafka 4.3.1 inmediatamente** si usas Kafka Streams (corrige KAFKA-20616)

**Prevención**:
- Monitorear heap usage constantemente
- Configurar JVM correctamente:
```bash
export KAFKA_HEAP_OPTS="-Xms6g -Xmx6g -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35"
```

---

## 🌍 Ecosistema y Herramientas

### Schema Registry (Confluent)
- Gestión centralizada de schemas
- Compatibilidad de schemas
- Avro, Protobuf, JSON Schema

### Kafka Connect
- +100 conectores certificados
- Source connectors: PostgreSQL, MySQL, MongoDB, etc.
- Sink connectors: Elasticsearch, S3, BigQuery, etc.

### ksqlDB
- SQL streaming sobre Kafka
- Creación de streams y tablas
- Joins, agregaciones, windowing

### Kafka UI Tools
- **Conduktor**: Interfaz completa de gestión
- **Kafka UI (Provectus)**: Open source, ligero
- **Kafdrop**: Simple, basado en web

### Monitoring
- **Prometheus + Grafana**: Métricas y dashboards
- **Kafka Exporter**: Exportador oficial de métricas
- **Burrow**: Monitoreo especializado en consumer lag

---

## 📚 Recursos de Aprendizaje

### Documentación Oficial
- **Web oficial**: https://kafka.apache.org
- **Documentación**: https://kafka.apache.org/documentation/
- **KIPs**: https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Improvement+Proposals

### Cursos Recomendados
1. **Confluent Developer**: Apache Kafka Fundamentals
2. **LinkedIn Learning**: Learning Apache Kafka
3. **Udemy**: Apache Kafka Series

### Libros
1. **"Kafka: The Definitive Guide"** (O'Reilly) - 2nd Edition
2. **"Kafka Streams in Action"** (Manning)
3. **"Designing Event-Driven Systems"** (O'Reilly)

### Comunidad
- **Mailing lists**: https://kafka.apache.org/contact
- **Slack**: Confluent Community Slack
- **Stack Overflow**: Tag `apache-kafka`
- **GitHub**: https://github.com/apache/kafka

### Blogs y Noticias
- **Blog oficial de Kafka**: https://kafka.apache.org/blog/
- **Confluent Blog**: https://www.confluent.io/blog/
- **Kafka Monthly Digest**: Red Hat Developer

---

## 🎯 Roadmap de Aprendizaje Sugerido

### Semana 1-2: Fundamentos
- [ ] Entender arquitectura básica
- [ ] Instalar Kafka localmente (modo KRaft)
- [ ] Crear primer productor y consumidor
- [ ] Experimentar con comandos CLI

### Semana 3-4: Desarrollo Intermedio
- [ ] Implementar productores con transacciones
- [ ] Implementar consumidores con procesamiento en batch
- [ ] Entender consumer groups y rebalancing
- [ ] Implementar custom serializers

### Semana 5-6: Kafka Streams
- [ ] Crear primera topología de Streams
- [ ] Implementar operaciones stateful
- [ ] Trabajar con windowing
- [ ] Realizar joins entre streams

### Semana 7-8: Operaciones
- [ ] Montar cluster multi-broker
- [ ] Configurar seguridad (SSL/SASL)
- [ ] Implementar monitoreo
- [ ] Practicar troubleshooting

### Semana 9-10: Avanzado
- [ ] Implementar Tiered Storage
- [ ] Configurar Kafka Connect
- [ ] Patrones de arquitectura (Event Sourcing, CQRS)
- [ ] Performance tuning

### Semana 11-12: Producción
- [ ] Migrar de ZooKeeper a KRaft (en entorno de prueba)
- [ ] Implementar disaster recovery
- [ ] Capacity planning
- [ ] Preparar para certificación (opcional)

---

## ⚠️ Aspectos Críticos a Recordar

### Migraciones y Actualizaciones

1. **No hay camino de vuelta desde KRaft a ZooKeeper**
2. **Migración ZK→KRaft debe hacerse en 3.9.x antes de actualizar a 4.x**
3. **Java 8 ya no está soportado en Kafka 4.x** (mínimo Java 11)
4. **Scala 2.12 ya no está soportado en Kafka 4.x** (mínimo Scala 2.13)

### Configuración de Producción

```bash
# SIEMPRE configurar carpetas temporales para compresión
export KAFKA_OPTS="-Dorg.xerial.snappy.tempdir=/opt/kafka/tmp \
                    -Dio.airlift.compress.zstd.ZstdTempFolder=/opt/kafka/tmp"
```

**Nota**: Si `/tmp` tiene `noexec`, Kafka NO arrancará sin esta configuración.

### Monitoreo Continuo

Alertas obligatorias:
1. `UnderReplicatedPartitions > 0`
2. `OfflinePartitionsCount > 0`
3. `ActiveControllerCount != 1`
4. Consumer lag > umbral definido
5. Disk usage > 80%

---

## 🔮 Próximas Características (Post 4.4.0)

### KIP-848: Consumer Rebalance Protocol (Next Gen)
- Rebalanceo incremental más eficiente
- Event-loop moderno en Group Coordinator
- Reducción significativa de pause time

### KIP-1352: Ranges and Range Aggregations
- Agregaciones basadas en eventos (no solo tiempo)
- Ventanas antes/después de un anchor
- Mayor flexibilidad en Streams DSL

### Mejoras en Containerización
- Mejor soporte para Kubernetes
- Health checks nativos
- Mejoras en métricas

---

## 💡 Tips Finales para el Aprendizaje

1. **Practica con el repositorio oficial**: Ya estás en él - experimenta con el código
2. **Lee los KIPs**: Entender el "por qué" detrás de las decisiones
3. **Contribuye**: Kafka es open source - reporta bugs, mejora documentación
4. **Sigue el blog oficial**: Mantente al día con releases
5. **Únete a la comunidad**: Las mailing lists son muy activas
6. **Experimenta con versiones nuevas**: Usa ramas de feature para probar
7. **No temas al código fuente**: Es Java/Scala legible y bien documentado

---

## 📞 Contacto y Contribución

- **Website**: https://kafka.apache.org
- **GitHub**: https://github.com/apache/kafka
- **Jira**: https://issues.apache.org/jira/browse/KAFKA
- **Mailing Lists**: https://kafka.apache.org/contact

### Cómo Contribuir
Ver `CONTRIBUTING.md` en este repositorio y:
https://kafka.apache.org/contributing.html

---

**Última actualización**: Agosto 2026  
**Versión del documento**: 1.0  
**Basado en**: Apache Kafka 4.3.1 (estable) y 4.4.0 (en desarrollo)

---

¡Bienvenido al ecosistema Kafka! 🚀
