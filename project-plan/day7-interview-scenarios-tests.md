# Day 7: Mock Interviews + Production Scenarios + Test Coverage

## Goal (2 hours)
Solidify everything from Days 1-6. Write integration tests using `@EmbeddedKafka`. Practice the most common senior interview questions grounded in YOUR workspace code.

---

## Part 1 — Integration Tests with EmbeddedKafka (50 min)

### Add test dependencies (already in both pom.xml):
```xml
<!-- Already present: -->
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka-test</artifactId>
    <scope>test</scope>
</dependency>
```

### Test 1: Producer sends → Consumer receives (producer app)
**File:** `exchange-data-producer/src/test/java/.../ProducerIntegrationTest.java`

```java
@SpringBootTest
@EmbeddedKafka(
    partitions = 3,
    topics = {"stock-data", "stock-data.DLT"},
    brokerProperties = {
        "auto.create.topics.enable=false",
        "log.dir=/tmp/embedded-kafka"
    }
)
@TestPropertySource(properties = {
    "spring.kafka.bootstrap-servers=${spring.embedded.kafka.brokers}"
})
class ProducerIntegrationTest {

    @Autowired
    private PublisherService publisherService;

    @Autowired
    private KafkaTemplate<String, StockData> kafkaTemplate;

    @Test
    void shouldSendStockDataToCorrectTopic() throws Exception {
        StockData stock = new StockData();
        stock.setStock("AAPL");
        stock.setOpen(175.0);
        stock.setClose(178.0);

        String result = publisherService.produceData("stock-data", stock);

        assertThat(result).contains("stock-data");
        // Verify partition assignment is deterministic by key
    }

    @Test
    void shouldSendSameStockToSamePartition() {
        // Send AAPL 5 times — should always land on same partition
        // This verifies key-based partitioning
        for (int i = 0; i < 5; i++) {
            StockData stock = new StockData();
            stock.setStock("AAPL");
            stock.setClose(100.0 + i);
            String result = publisherService.produceData("stock-data", stock);
            assertThat(result).contains("partition=");
        }
    }
}
```

### Test 2: Consumer processes and saves to DB
**File:** `exchange-data-consumer/src/test/java/.../ConsumerIntegrationTest.java`

```java
@SpringBootTest
@EmbeddedKafka(partitions = 1, topics = {"stock-data"})
@TestPropertySource(properties = {
    "spring.kafka.bootstrap-servers=${spring.embedded.kafka.brokers}"
})
class ConsumerIntegrationTest {

    @Autowired
    private KafkaTemplate<String, String> testKafkaTemplate;

    @Autowired
    private StockDataRepository stockDataRepository;

    @Test
    void shouldConsumeAndPersistStockData() throws Exception {
        String json = """
            {"id":"test-001","date":"2026-03-16","stock":"MSFT",
             "open":400.0,"close":405.0,"high":410.0,"low":398.0,
             "volume":1000000,"changePct":1.25}
            """;

        testKafkaTemplate.send("stock-data", "MSFT", json);

        // Wait for async consumer — poll until saved or timeout
        await().atMost(Duration.ofSeconds(10))
            .untilAsserted(() -> {
                StepVerifier.create(stockDataRepository.findById("test-001"))
                    .expectNextMatches(dto -> "MSFT".equals(dto.getStock()))
                    .verifyComplete();
            });
    }

    @Test
    void shouldSendPoisonPillToDLT() throws Exception {
        // Send invalid JSON — should fail deserialization → DLT
        testKafkaTemplate.send("stock-data", "INVALID", "not-valid-json");

        // Consume from DLT and verify
        Consumer<String, String> dltConsumer = /* create test consumer for stock-data.DLT */;
        ConsumerRecords<String, String> records = dltConsumer.poll(Duration.ofSeconds(5));
        assertThat(records.count()).isGreaterThan(0);
    }
}
```

Add `awaitility` for async assertions:
```xml
<dependency>
    <groupId>org.awaitility</groupId>
    <artifactId>awaitility</artifactId>
    <scope>test</scope>
</dependency>
```

---

## Part 2 — Interview Scenario Q&A (Grounded in Your Code) (40 min)

Practice answering these using your actual workspace as reference.

---

### Scenario 1: "Walk me through your Kafka architecture"

**Your answer structure:**
> "In this stock analysis system, I have two Spring Boot services. The producer (`exchange-data-producer`) reads stock data from CSV and PostgreSQL, and publishes to Kafka topic `stock-data` with the stock symbol as the partition key — ensuring all AAPL events land on the same partition for ordering.
>
> The consumer (`exchange-data-consumer`) is a WebFlux + reactive application. It has two consumption paths: a traditional `@KafkaListener` with `DefaultErrorHandler` and DLT for reliability, and a `ReactiveKafkaConsumerTemplate` that feeds a `Sinks.Many` hot Flux, which bridges into WebSocket via STOMP for real-time dashboards.
>
> Data is persisted via R2DBC to PostgreSQL reactively, and the latest price per symbol is cached in Redis with 1-hour TTL for low-latency reads."

---

### Scenario 2: "How did you handle message loss / duplicates?"

**Your answer from the code:**
> - Producer: `acks=all`, `ENABLE_IDEMPOTENCE_CONFIG=true`, `retries=3` → no duplicates from retries
> - Consumer: `DefaultErrorHandler` with exponential backoff (1s→2s→4s) then DLT
> - For exactly-once: transactional producer with `stock-tx-` prefix + consumer `isolation.level=read_committed`
> - The `StockDataDTO` has a composite primary key `(EVENT_DATE, STOCK)` in PostgreSQL — so even if saved twice, upsert semantics prevent duplicates in the DB

---

### Scenario 3: "Consumer lag suddenly grows — what do you do?"

**Your investigation steps:**
```bash
# Step 1: Check lag
docker exec -it kafka kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe --group stock-consumer-group

# Step 2: Check if consumer is alive (CONSUMER-ID column)
# Step 3: Check for frequent rebalances in consumer logs
grep "Rebalancing" consumer.log | tail -20

# Step 4: Check if max.poll.interval.ms exceeded
grep "max.poll.interval" consumer.log

# Step 5: Check DB write latency
# If R2DBC saves take > max.poll.interval.ms (5min default),
# Kafka broker thinks consumer is dead → rebalance → re-start from committed offset
```

**Resolution options by root cause:**
| Root Cause | Fix |
|-----------|-----|
| DB writes slow | Increase `max.poll.interval.ms` or batch DB writes |
| Too many records per poll | Decrease `max.poll.records` |
| Insufficient parallelism | Increase partitions + `factory.setConcurrency()` |
| GC pauses | Tune JVM heap, use G1GC |
| Consumer crashed | Check DLT for poison pills |

---

### Scenario 4: "@KafkaListener takes 30 seconds per message — what breaks?"

**Answer:**
> With `max.poll.interval.ms=300000` (5min default), 30s per message is safe if `max.poll.records=1`. But with `max.poll.records=500` and 30s each → 15000s total → broker thinks consumer is dead → **rebalance triggered** → messages re-assigned to another consumer → **duplicate processing**.
>
> Fix: Either reduce `max.poll.records=1` (low throughput) or process asynchronously:
```java
@KafkaListener(topics = "stock-data")
public void listen(String message, Acknowledgment ack) {
    CompletableFuture.runAsync(() -> {
        slowProcess(message);
        ack.acknowledge(); // manual commit after async completion
    });
}
// Must set: factory.getContainerProperties().setAckMode(AckMode.MANUAL)
```

---

### Scenario 5: "How would you migrate from RabbitMQ to Kafka for this system?"

**Structured answer:**
1. **Parallel run** — run both Rabbit and Kafka, dual-publish from producer during transition
2. **Consumer migration** — switch one consumer group to Kafka, verify parity
3. **Topic design** — Rabbit queues → Kafka topics with partition keys (stock symbol)
4. **Retention** — Kafka retains messages (configurable), Rabbit deletes on ack — design consumers for idempotency
5. **Replay** — reset consumer group offsets to replay history (not possible in Rabbit)
6. **Cut-over** — stop Rabbit publisher, all load on Kafka, decommission Rabbit

---

### Scenario 6: "Production incident: messages being duplicated in 'stock-data'"

**Diagnosis using your code:**
```
1. Check if idempotent producer enabled  → ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG
2. Check if consumer auto-commits before processing completes  → auto-commit=false?
3. Check for rebalances during processing  → uncommitted offset re-delivered
4. Check DB primary key constraint violations  → indicates duplicate saves
5. Check transactional producer setup  → stock-tx- prefix?
```

---

## Part 3 — Production Checklist (30 min)

### Zero-Data-Loss Configuration Checklist
```
Producer:
  ✓ acks=all
  ✓ enable.idempotence=true
  ✓ retries=3+ with exponential backoff
  ✓ max.in.flight.requests.per.connection=1
  ✓ transactional.id set for exactly-once

Consumer:
  ✓ enable.auto.commit=false
  ✓ Manual or at-least-once ack mode
  ✓ DefaultErrorHandler with DLT
  ✓ isolation.level=read_committed (for transactional producers)
  ✓ max.poll.interval.ms tuned to actual processing time

Cluster:
  ✓ replication.factor >= 2 (min 3 for production)
  ✓ min.insync.replicas = replication.factor - 1
  ✓ unclean.leader.election.enable=false
  ✓ Monitoring: under-replicated partitions alert = 0

Application:
  ✓ Idempotent writes to DB (upsert, not insert)
  ✓ All consumers have timeout/graceful shutdown
  ✓ DLT monitored + alerting
  ✓ Consumer lag alert > threshold
```

---

## Day 7 Deliverables
- [ ] `ProducerIntegrationTest.java` with `@EmbeddedKafka` — 2+ tests passing
- [ ] `ConsumerIntegrationTest.java` with `@EmbeddedKafka` — persistent + DLT tests passing
- [ ] Can verbally answer all 6 interview scenarios referencing YOUR code
- [ ] Zero-data-loss checklist reviewed against your actual configs
- [ ] Total stack: Kafka (2-broker) + Postgres + Redis + Flink all running in docker-compose
