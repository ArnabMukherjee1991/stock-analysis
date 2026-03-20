# Day 1: Fix Critical Bugs + Kafka Internals by Doing

## Goal (2 hours)
Fix the broken wiring in producer/consumer, understand Kafka internals by observing real behavior.

---

## Part 1 — Fix Critical Bugs (45 min)

### Fix 1: `PublisherConfig.java` — Add `@Configuration`
**File:** `exchange-data-producer/src/main/java/com/myexchange/producer/exchange/data/producer/config/PublisherConfig.java`

Add `@Configuration` annotation so Spring registers the beans:
```java
@Configuration  // ADD THIS
public class PublisherConfig {
```
Keep `KafkaAdmin` in `KafkaConfig.java` and `KafkaTemplate` in `PublisherConfig.java` for now so the bootstrap and producer concerns stay separated:

```java
@Configuration
public class KafkaConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Bean
    public KafkaAdmin admin() {
        Map<String, Object> configs = new HashMap<>();
        configs.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        return new KafkaAdmin(configs);
    }

    @Bean
    public ProducerFactory<String, String> producerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        // Day 2 — add acks, idempotence here
        return new DefaultKafkaProducerFactory<>(configProps);
    }

    @Bean
    public KafkaTemplate<String, String> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}
```
After this phase, keep the producer config explicit and add reliability settings there.

### Fix 2: `ListenerConfig.java` (consumer) — Add `@Configuration` + Fix Import
**File:** `exchange-data-consumer/src/main/java/com/myexchange/consumer/config/ListenerConfig.java`

```java
@Configuration  // ADD THIS
public class ListenerConfig {
```
Fix the wrong import:
```java
// REMOVE: import com.fasterxml.jackson.databind.deser.std.StringDeserializer;
// ADD:
import org.apache.kafka.common.serialization.StringDeserializer;
```

### Fix 3: `PublisherController.java` — Wrong annotation
**File:** `exchange-data-producer/.../controller/PublisherController.java`

```java
// CHANGE:
@PutExchange("/publishall")
// TO:
@PutMapping("/publishall")
```

### Fix 4: `StockConsumerService.java` — Remove shared mutable state
**File:** `exchange-data-consumer/.../service/StockConsumerService.java`

The `private final List<StockDataDTO> stockDataList` is a race condition. Fix:
```java
@KafkaListener(topics = "stock-data", groupId = "stock-consumer-group")
public void listenToStockData(String message) {
    log.info("Received message on partition {} offset {}", /* add these */);
    StockDataDTO stockDataDTO = Util.convertJsontoObject(message, StockDataDTO.class);
    stockDataRepository.save(stockDataDTO).subscribe();
}
```

---

## Part 2 — Kafka Internals by Observer (45 min)

### Step 1 — Start Kafka
```bash
cd d:/work/stock-analysis/stock-analysis-ops
docker-compose up -d kafka kafka-ui db
```

### Step 2 — Create a topic with 3 partitions via AdminController
Start producer app, then call:
```
POST http://localhost:8081/kafka/topics/add
{
  "name": "stock-data",
  "partitions": 3,
  "replicationFactor": 1
}
```

### Step 3 — Inspect the topic via CLI inside Kafka container
```bash
docker exec -it kafka /opt/bitnami/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe --topic stock-data
```
**Observe:** Leader, Replicas, ISR columns — understand what they mean.

### Step 4 — Send a few messages and observe partition assignment
Call `POST /api/v1/publisher/publish` with different stock symbols (AAPL, MSFT, GOOG).
Then check offsets:
```bash
docker exec -it kafka /opt/bitnami/kafka/bin/kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe --group stock-consumer-group
```
**Observe:** CURRENT-OFFSET, LOG-END-OFFSET, LAG columns per partition.

### Step 5 — Add offset + partition logging to consumer
In `StockConsumerService.java`:
```java
@KafkaListener(topics = "stock-data", groupId = "stock-consumer-group")
public void listenToStockData(
    ConsumerRecord<String, String> record) {
    log.info("topic={} partition={} offset={} key={}", 
        record.topic(), record.partition(), record.offset(), record.key());
    // ... rest of processing
}
```

---

## Part 3 — Theory Checkpoint (30 min)

Answer these by reasoning (no googling):

1. You have 3 partitions and 1 consumer — which consumer gets messages from all 3 partitions?
2. Start a second consumer in the same group — what happens to partition assignment?
3. Why does changing the stock symbol as the key matter for ordering?
4. What is "Log end offset" vs "committed offset" vs "current offset"?

**Key concept to internalize today:**
- The _key_ you pass to `kafkaTemplate.send(topic, key, value)` determines which partition the message lands on (via hash mod partitions).
- Since `PublisherService` uses `stockData.getStock()` as the key — ALL messages for the same stock symbol always go to the same partition → ordering per symbol is guaranteed.

---

## Day 1 Deliverables
- [ ] All 4 critical bugs fixed and app starts without errors
- [ ] Kafka running, `stock-data` topic with 3 partitions visible in Kafka UI (http://localhost:8085)
- [ ] Producer sends a message, consumer receives it, partition + offset logged
- [ ] Can describe topic and consumer group from CLI
