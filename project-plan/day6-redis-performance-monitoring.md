# Day 6: Redis Caching + Performance Tuning + Monitoring

## Goal (2 hours)
Add Redis for caching (latest stock price per symbol). Tune producer/consumer configs for performance. Add consumer lag monitoring. Wire Flink concept via discussion + basic ops docker-compose additions.

---

## Part 1 — Add Redis to docker-compose (20 min)

The root `Dockerfile` is already a Redis image. Wire it into `stock-analysis-ops/docker-compose.yaml`:

```yaml
  redis:
    image: redis:latest
    container_name: redis
    ports:
      - "6379:6379"
    networks:
      - app-tier
    volumes:
      - ./cache/redis:/data
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
```

Start:
```bash
docker-compose up -d redis
```

---

## Part 2 — Redis Cache in Consumer (35 min)

### Add Redis dependency to `exchange-data-consumer/pom.xml`:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
```

### Create `RedisConfig.java` in consumer
**File:** `exchange-data-consumer/src/main/java/com/myexchange/consumer/config/RedisConfig.java`

```java
@Configuration
public class RedisConfig {

    @Bean
    public ReactiveRedisConnectionFactory reactiveRedisConnectionFactory() {
        return new LettuceConnectionFactory("localhost", 6379);
    }

    @Bean
    public ReactiveRedisTemplate<String, StockDataDTO> reactiveRedisTemplate(
            ReactiveRedisConnectionFactory factory) {
        Jackson2JsonRedisSerializer<StockDataDTO> serializer =
            new Jackson2JsonRedisSerializer<>(StockDataDTO.class);
        RedisSerializationContext<String, StockDataDTO> context =
            RedisSerializationContext.<String, StockDataDTO>newSerializationContext()
                .key(RedisSerializer.string())
                .value(serializer)
                .hashKey(RedisSerializer.string())
                .hashValue(serializer)
                .build();
        return new ReactiveRedisTemplate<>(factory, context);
    }
}
```

### Update `ReactiveStockConsumerService.java` — cache latest price per symbol:

```java
private final ReactiveRedisTemplate<String, StockDataDTO> redisTemplate;

// In the reactive pipeline's flatMap, after saving to DB:
.flatMap(dto -> stockDataRepository.save(dto)
    .flatMap(saved -> {
        // Cache latest price: key = "stock:AAPL", TTL = 1 hour
        String cacheKey = "stock:" + saved.getStock();
        return redisTemplate.opsForValue()
            .set(cacheKey, saved, Duration.ofHours(1))
            .thenReturn(saved);
    })
)
```

### Add cache-read endpoint in consumer for low-latency lookup:
```java
// In ListenerController.java:
@GetMapping("/latest/{symbol}")
public Mono<StockDataDTO> getLatestStock(@PathVariable String symbol) {
    String cacheKey = "stock:" + symbol.toUpperCase();
    return redisTemplate.opsForValue().get(cacheKey)
        .switchIfEmpty(
            // Cache miss — fall back to DB
            stockDataRepository.findTopByStockOrderByDateDesc(symbol)
                .doOnNext(dto -> redisTemplate.opsForValue()
                    .set(cacheKey, dto, Duration.ofHours(1)).subscribe())
        );
}
```

**Test:**
```bash
# After sending a stock message via producer:
curl http://localhost:8001/api/v1/stocks/latest/AAPL
# Should return from Redis (< 1ms) not DB
```

---

## Part 3 — Performance Tuning (30 min)

### Producer tuning — Update `KafkaConfig.java` in producer:
```java
// Batching — accumulate messages before sending
configProps.put(ProducerConfig.BATCH_SIZE_CONFIG, 32768);        // 32KB batch
configProps.put(ProducerConfig.LINGER_MS_CONFIG, 20);            // wait 20ms to fill batch
configProps.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "lz4");  // compress batches
configProps.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 67108864L); // 64MB buffer

// For high throughput bulk publishing (publishAll):
configProps.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5); // more pipelining
```

> **Interview Tradeoff:** `linger.ms=20` adds 20ms latency per message in exchange for larger batches and better throughput. For a stock trading system with latency SLAs < 5ms — do NOT use this. For analytics/bulk load — use it.

### Consumer tuning — Update `ListenerConfig.java` in consumer:
```java
// For the blocking @KafkaListener path:
props.put(ConsumerConfig.FETCH_MIN_BYTES_CONFIG, 1024);         // wait for 1KB before fetch
props.put(ConsumerConfig.FETCH_MAX_WAIT_MS_CONFIG, 500);        // or wait max 500ms
props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 500);         // process 500 records per poll
props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 45000);     // 45s before rebalance
props.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, 3000);   // heartbeat every 3s

// Concurrency = number of partitions (3)
factory.setConcurrency(3);
```

### ConcurrentKafkaListenerContainerFactory concurrency:
```java
// With 3 partitions on stock-data topic and concurrency=3:
// Each thread handles 1 partition → parallel processing at partition level
factory.setConcurrency(3);
```

---

## Part 4 — Consumer Lag Monitoring (15 min)

### Add lag metrics endpoint to consumer app:

```java
// Add to ListenerController.java:
private final KafkaListenerEndpointRegistry kafkaListenerEndpointRegistry;

@GetMapping("/lag")
public Map<String, Object> getConsumerLag() {
    Map<String, Object> result = new HashMap<>();
    kafkaListenerEndpointRegistry.getListenerContainers().forEach(container -> {
        container.metrics().forEach((topicPartition, metric) -> {
            result.put(topicPartition.toString(), metric);
        });
    });
    return result;
}
```

### CLI-based lag check:
```bash
docker exec -it kafka /opt/bitnami/kafka/bin/kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe --group stock-consumer-group

# Output columns:
# GROUP  TOPIC  PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG  CONSUMER-ID
```

**What to look for:**
- LAG > 0 and growing → consumer is falling behind
- LAG = 0 → consumer is caught up
- No CONSUMER-ID assigned → consumer is not running or crashed

---

## Part 5 — Flink Overview + Docker Compose Addition (10 min)

### Add Flink to `stock-analysis-ops/docker-compose.yaml` for future Day 7:

```yaml
  flink-jobmanager:
    image: flink:1.19-scala_2.12
    container_name: flink-jobmanager
    ports:
      - "8081:8081"
    networks:
      - app-tier
    environment:
      - FLINK_PROPERTIES=jobmanager.rpc.address: flink-jobmanager
    command: jobmanager

  flink-taskmanager:
    image: flink:1.19-scala_2.12
    container_name: flink-taskmanager
    networks:
      - app-tier
    depends_on:
      - flink-jobmanager
    environment:
      - FLINK_PROPERTIES=jobmanager.rpc.address: flink-jobmanager
    command: taskmanager
```

**Start Flink:**
```bash
docker-compose up -d flink-jobmanager flink-taskmanager
# Flink UI: http://localhost:8081
```

---

## Theory Checkpoint (10 min)

1. `max.poll.records=500` + slow DB = what risk? (Answer: `max.poll.interval.ms` exceeded → rebalance)
2. Why does `linger.ms` hurt a stock TRADING system but help a stock ANALYTICS system?
3. Redis vs DB for latest-price lookup — what happens if Redis restarts without persistence?
4. Explain: "Kafka is NOT a database but Redis IS a cache" — when does this distinction matter in your system?

---

## Day 6 Deliverables
- [ ] Redis service added to docker-compose and running
- [ ] `ReactiveRedisTemplate<String, StockDataDTO>` caches latest price per symbol
- [ ] `/api/v1/stocks/latest/{symbol}` returns from Redis cache
- [ ] Producer batch size, linger, compression configured
- [ ] Consumer `max.poll.records=500`, concurrency=3 configured
- [ ] CLI lag check shows LAG=0 after consuming
- [ ] Flink services added to docker-compose (jobmanager + taskmanager running)
- [ ] Flink UI accessible at http://localhost:8081
