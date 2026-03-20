# 7-Day Practical Kafka POC — Master Index

## Projects You're Building On

| Project | Tech Stack | Port | Role |
|---------|-----------|------|------|
| `exchange-data-producer` | Spring Boot 3.4.3, Java 21, JPA, Kafka, OpenAPI | 8081 | Kafka producer + Stock DB CRUD |
| `exchange-data-consumer` | Spring Boot 3.4.2, Java 21, WebFlux, R2DBC, Kafka | 8001 | Kafka consumer + Reactive DB + WebSocket |
| `stock-analysis-ops` | Docker Compose | - | All infrastructure |

---

## Daily Progress Map

| Day | File | Key Deliverable |
|-----|------|-----------------|
| Day 1 | [day1-kafka-internals-fix-bugs.md](day1-kafka-internals-fix-bugs.md) | Fix 4 critical bugs. Topic + partition + offset visibility |
| Day 2 | [day2-multi-broker-reliability.md](day2-multi-broker-reliability.md) | 2-broker cluster, acks=all, idempotent producer, ISR failure |
| Day 3 | [day3-spring-kafka-error-dlt-transactions.md](day3-spring-kafka-error-dlt-transactions.md) | Typed serializers, DefaultErrorHandler, DLT, transactions |
| Day 4 | [day4-reactive-kafka-webflux.md](day4-reactive-kafka-webflux.md) | Reactor Kafka, reactive consumer, SSE streaming endpoint |
| Day 5 | [day5-websocket-kafka-fanout.md](day5-websocket-kafka-fanout.md) | STOMP WebSocket, Kafka→WS bridge, browser test page |
| Day 6 | [day6-redis-performance-monitoring.md](day6-redis-performance-monitoring.md) | Redis cache, producer/consumer tuning, lag monitoring, Flink setup |
| Day 7 | [day7-interview-scenarios-tests.md](day7-interview-scenarios-tests.md) | EmbeddedKafka tests, 6 interview scenarios, zero-loss checklist |

---

## Gap Analysis
See [gap-analysis.md](gap-analysis.md) for full list of critical bugs, missing configs, and feature gaps found in the current codebase.

## Analysis Folder
Use [analysis/README.md](analysis/README.md) as the execution guide for data formats, SSL, Schema Registry, and the phase-by-phase rollout.

---

## Infrastructure Evolution (End State after Day 7)

```
docker-compose (stock-analysis-ops):
  kafka-1        (broker/controller)     :19092 external
  kafka-2        (broker/controller)     :19093 external
  kafka-3        (broker/controller)     :19094 external
  kafka-ui                               :8085
  db (postgres)                          :5432
  redis                                  :6379             ← Optional later phase
  flink-jobmanager                       :8081             ← Optional later phase
  flink-taskmanager
```

---

## Code Evolution Summary

### `exchange-data-producer` changes by day:

| Day | File Changed | Change |
|-----|-------------|--------|
| 1 | `KafkaConfig.java` | Add `@Configuration`, merge `PublisherConfig` into it, add `ProducerFactory` + `KafkaTemplate` |
| 1 | `PublisherConfig.java` | Delete (merged into KafkaConfig) |
| 1 | `PublisherController.java` | Fix `@PutExchange` → `@PutMapping` |
| 2 | `KafkaConfig.java` | Add `acks=all`, `ENABLE_IDEMPOTENCE`, `retries`, batching configs |
| 2 | `PublisherService.java` | Change raw `KafkaTemplate` → typed `KafkaTemplate<String, StockData>`, add `whenComplete` logging |
| 3 | `KafkaConfig.java` | Add `JsonSerializer`, `KafkaTransactionManager` bean |
| 3 | `PublisherService.java` | Add transactional publish method |
| 4 | `pom.xml` | Add `reactor-kafka` dependency |
| 4 | `ReactiveKafkaProducerConfig.java` | New file — `ReactiveKafkaProducerTemplate` bean |
| 4 | `PublisherService.java` | Add `produceReactive()` method |

### `exchange-data-consumer` changes by day:

| Day | File Changed | Change |
|-----|-------------|--------|
| 1 | `ListenerConfig.java` | Add `@Configuration`, fix `StringDeserializer` import |
| 1 | `StockConsumerService.java` | Remove shared mutable list, log partition+offset |
| 3 | `ListenerConfig.java` | Switch to `JsonDeserializer<StockDataDTO>`, add `DefaultErrorHandler`, DLT, backoff, concurrency=3 |
| 3 | `StockConsumerService.java` | Typed `ConsumerRecord<String, StockDataDTO>`, add DLT listener |
| 4 | `pom.xml` | Add `reactor-kafka`, `spring-boot-starter-websocket` |
| 4 | `ReactiveKafkaConfig.java` | New file — `ReactiveKafkaConsumerTemplate` bean |
| 4 | `ReactiveStockConsumerService.java` | New file — reactive pipeline + `Sinks.Many` |
| 4 | `ListenerController.java` | Add `/stream` SSE endpoint |
| 5 | `WebSocketConfig.java` | New file — STOMP + SockJS config |
| 5 | `KafkaWebSocketBridgeService.java` | New file — bridges Kafka Flux → STOMP topics |
| 5 | `ws-test.html` | New test page in `static/` |
| 6 | `pom.xml` | Add `spring-boot-starter-data-redis-reactive` |
| 6 | `RedisConfig.java` | New file — `ReactiveRedisTemplate` bean |
| 6 | `ReactiveStockConsumerService.java` | Cache latest price in Redis after DB save |
| 6 | `ListenerController.java` | Add `/latest/{symbol}` Redis-backed endpoint, `/lag` endpoint |
| 7 | `ConsumerIntegrationTest.java` | New EmbeddedKafka integration test |

---

## Key Interview Concepts Mapped to Code

| Concept | Where in Code |
|---------|---------------|
| Partition key strategy | `PublisherService.kafkaTemplate.send(topic, stock.getStock(), ...)` |
| ISR + acks | `KafkaConfig.producerFactory()` — `acks=all`, `ENABLE_IDEMPOTENCE_CONFIG` |
| DLT + retry | `ListenerConfig.deadLetterPublishingRecoverer()` + `DefaultErrorHandler` |
| Exactly-once | `KafkaConfig.transactionalProducerFactory()` + `KafkaTransactionManager` |
| Reactive non-blocking | `ReactiveStockConsumerService` + `Sinks.Many` + `Schedulers.boundedElastic()` |
| WebSocket fan-out | `KafkaWebSocketBridgeService` + `SimpMessagingTemplate` |
| Redis cache-aside | `ReactiveStockConsumerService` → `redisTemplate.opsForValue().set()` |
| Consumer lag | CLI `kafka-consumer-groups.sh --describe` + `/lag` endpoint |
| Poison pill handling | `handler.addNotRetryableExceptions(JsonProcessingException.class)` → DLT |
| Rebalancing awareness | `ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG`, `HEARTBEAT_INTERVAL_MS_CONFIG` |
