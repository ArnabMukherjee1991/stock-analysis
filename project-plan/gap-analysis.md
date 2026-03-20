# Gap Analysis: Current State vs Kafka Mastery Plan

## Current Phase Targets
- Data format decision: JSON now, schema-aware format next.
- Transport security: plaintext local today, SSL for the secured phase.
- Schema governance: Schema Registry and compatibility checks before wider scale-out.
- Scaling policy: create topics first, then raise partitions, then add consumers, then add brokers.

## Current State Summary (as of March 2026)

### exchange-data-producer (Spring Boot 3.4.3, Java 21)
| Component | Status | Notes |
|-----------|--------|-------|
| `KafkaConfig.java` | Partial | Only has `KafkaAdmin` bean — no `ProducerFactory`, no `KafkaTemplate` bean here |
| `PublisherConfig.java` | Partial | Has `ProducerFactory` + `KafkaTemplate` — but **missing `@Configuration`** annotation, so beans are NOT registered |
| `ListenerConfig.java` | Empty | Class body is empty, stubs/imports only |
| `PublisherService.java` | Working | Uses `KafkaTemplate` (raw type), sends `stock:json` keyed messages, async with `CompletableFuture` |
| `StockConsumerService.java` | Commented out | Consumer listener commented out — no consumer active yet in producer app |
| `AdminService.java` | Working | List + create topics via `KafkaAdmin` |
| `AdminController.java` | Working | REST endpoints for topic management |
| `PublisherController.java` | Working | `/publish` and `/publishall` — `publishall` has wrong annotation `@PutExchange` instead of `@PutMapping` |
| `StockDataController.java` | Partial | `@DeleteMapping` method empty, `@PostMapping("/insert")` missing `@RequestBody` |
| Serialization | Basic | Raw String JSON — no typed serializer, no Avro/JSON schema |
| Error Handling | None | No `DefaultErrorHandler`, no DLT, no retry topics |
| Idempotent Producer | Missing | No `ENABLE_IDEMPOTENCE_CONFIG` set |
| Transactions | Missing | No `transactional.id`, no `KafkaTransactionManager` |
| acks / min.insync | Missing | Using defaults — not configured for reliability |
| Batch publishing | Stub | `publishAll()` reads CSV but does not batch-publish properly |
| Tests | Empty | No Kafka integration tests |

### exchange-data-consumer (Spring Boot 3.4.2, Java 21, WebFlux + R2DBC)
| Component | Status | Notes |
|-----------|--------|-------|
| `ListenerConfig.java` | Broken | Not annotated with `@Configuration`, uses wrong import (`fasterxml StringDeserializer` instead of Kafka's) |
| `StockConsumerService.java` | Partial | `@KafkaListener` exists, saves via reactive repo — but `stockDataList` is a shared mutable field (race condition!) |
| `StockDataRepository.java` | Working | `ReactiveCrudRepository` extending ok |
| `PGDBConfig.java` | Working | R2DBC connection factory wired correctly |
| `ListenerController.java` | Partial | `consumeAllStockData()` returns in-memory list only, not DB data |
| Reactive Kafka | Missing | Dependency NOT in pom.xml — consumer uses blocking `@KafkaListener` inside WebFlux app |
| Error Handling | None | No `DefaultErrorHandler`, no DLT, no retry |
| Manual Offset Commit | Missing | Using auto-commit defaults |
| WebSocket | Missing | No WebSocket or STOMP dependencies or config |
| Dead Letter Topic | Missing | Not implemented |
| Consumer Group Awareness | Missing | Group ID hardcoded as string literal |
| Tests | Empty | No consumer integration tests |

### stock-analysis-ops (Docker Compose)
| Component | Status | Notes |
|-----------|--------|-------|
| Kafka | Single broker | No multi-broker setup |
| Kafka UI | Configured | Port 8085 |
| Postgres | Configured | `exchangedb`, init SQL wired |
| Redis | Missing | Referenced in Dockerfile at root but not in ops docker-compose |
| Flink | Data folder exists | No Flink service in docker-compose |
| MongoDB | Data folder exists | No Mongo service in docker-compose |
| Schema Registry | Commented out | Available but disabled |
| Kafka Connect | Commented out | Available but disabled |

---

## Critical Bugs to Fix Before Day 1

1. **`PublisherConfig.java`** — missing `@Configuration` → KafkaTemplate bean NOT created → producer will fail
2. **`ListenerConfig.java` (consumer)** — missing `@Configuration` + wrong `StringDeserializer` import → consumer will NOT work
3. **`PublisherController.java`** — `@PutExchange` is a declarative HTTP client annotation, not a REST endpoint — should be `@PutMapping`
4. **`StockConsumerService.java`** — shared mutable `stockDataList` field is a race condition in concurrent listener
5. **`StockDataController.java`** — `@PostMapping("/insert")` missing `@RequestBody`

---

## Gap Map: study.md Topics vs Workspace

| Study Plan Topic | Gap Level | What's Missing |
|-----------------|-----------|----------------|
| Topics/Partitions/Offsets | HIGH | No partition key strategy, no offset logging |
| ISR / Replication / acks | HIGH | acks defaults, no min.insync.replicas config |
| Idempotent Producer | HIGH | Not configured |
| Transactional Producer+Consumer | HIGH | Not started |
| Spring KafkaTemplate (typed) | MEDIUM | Raw type used, no JSON serializer config |
| @KafkaListener + Error Handler | HIGH | No DefaultErrorHandler, no DLT |
| Dead Letter Topic | HIGH | Not implemented |
| Retry Topics with Backoff | HIGH | Not implemented |
| Reactive Kafka (Reactor Kafka) | HIGH | Dependency missing in consumer pom.xml |
| WebFlux integration | MEDIUM | Consumer already is WebFlux, but Kafka not reactive |
| WebSocket + Kafka fan-out | HIGH | No WebSocket at all |
| STOMP over WebSocket | HIGH | Not started |
| Multi-broker Kafka | HIGH | Single broker in docker-compose |
| Performance Tuning configs | HIGH | No batch size, linger, compression configs |
| Consumer lag monitoring | HIGH | No metrics/monitoring |
| Redis caching | HIGH | Redis not wired in any project |
| Flink stream processing | HIGH | No Flink integration |
| Integration Tests | HIGH | No @EmbeddedKafka tests |
| Jackson JSON Serializer (typed) | MEDIUM | Using raw String serializer |
| Manual offset commit | HIGH | Auto-commit only |
