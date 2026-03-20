# Day 4: Reactive Kafka + WebFlux Integration in Consumer

## Goal (2 hours)
The consumer project is ALREADY WebFlux + R2DBC — but the Kafka listener is blocking (`@KafkaListener`).  
Today: wire Reactor Kafka (`ReactiveKafkaConsumerTemplate`) alongside the existing listener config, add a reactive endpoint for streaming data, and understand the threading model.

---

## Part 1 — Add Reactor Kafka Dependency (15 min)

### Update `exchange-data-consumer/pom.xml`

Add these dependencies:
```xml
<!-- Reactor Kafka -->
<dependency>
    <groupId>io.projectreactor.kafka</groupId>
    <artifactId>reactor-kafka</artifactId>
    <version>1.3.23</version>
</dependency>

<!-- Jackson for reactive JSON handling -->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
</dependency>

<!-- WebSocket support for Day 5 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

Also add to `exchange-data-producer/pom.xml`:
```xml
<!-- Reactor Kafka for reactive producing -->
<dependency>
    <groupId>io.projectreactor.kafka</groupId>
    <artifactId>reactor-kafka</artifactId>
    <version>1.3.23</version>
</dependency>
```

---

## Part 2 — Reactive Consumer Config (40 min)

### Create `ReactiveKafkaConfig.java` in consumer
**File:** `exchange-data-consumer/src/main/java/com/myexchange/consumer/config/ReactiveKafkaConfig.java`

```java
@Configuration
public class ReactiveKafkaConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Bean
    public ReactiveKafkaConsumerTemplate<String, String> reactiveKafkaConsumer() {
        ReceiverOptions<String, String> receiverOptions = ReceiverOptions
            .<String, String>create(Map.of(
                ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers,
                ConsumerConfig.GROUP_ID_CONFIG, "stock-reactive-group",
                ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class,
                ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class,
                ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "latest",
                // Manual offset commit — important for reactive flows
                ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false,
                ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 100
            ))
            .subscription(Collections.singleton("stock-data"))
            // Commit offset AFTER processing acknowledgment
            .commitStrategy(CommitStrategy.throttled(100, Duration.ofSeconds(5)));

        return new ReactiveKafkaConsumerTemplate<>(receiverOptions);
    }
}
```

### Create `ReactiveStockConsumerService.java`
**File:** `exchange-data-consumer/src/main/java/com/myexchange/consumer/service/ReactiveStockConsumerService.java`

```java
@Service
@Slf4j
@RequiredArgsConstructor
public class ReactiveStockConsumerService {

    private final ReactiveKafkaConsumerTemplate<String, String> reactiveKafkaConsumer;
    private final StockDataRepository stockDataRepository;
    private final ObjectMapper objectMapper;

    // Emit incoming Kafka messages as a hot Flux (for WebSocket fan-out on Day 5)
    private final Sinks.Many<StockDataDTO> stockSink = 
        Sinks.many().multicast().onBackpressureBuffer(1000);

    @PostConstruct
    public void startConsuming() {
        reactiveKafkaConsumer
            .receiveAutoAck()  // auto-acks after Flux item is consumed
            .flatMap(record -> {
                log.info("Reactive received: partition={} offset={} key={}",
                    record.partition(), record.offset(), record.key());
                return Mono.fromCallable(() ->
                        objectMapper.readValue(record.value(), StockDataDTO.class))
                    .flatMap(dto -> {
                        stockSink.tryEmitNext(dto); // push to sink for WebSocket
                        return stockDataRepository.save(dto);
                    })
                    .onErrorResume(ex -> {
                        log.error("Error processing reactive message at offset {}: {}",
                            record.offset(), ex.getMessage());
                        return Mono.empty(); // don't crash the pipeline
                    });
            })
            .subscribeOn(Schedulers.boundedElastic()) // Kafka poll on separate thread
            .subscribe(
                saved -> log.debug("Saved stock: {}", saved.getStock()),
                error -> log.error("Reactive consumer pipeline error", error)
            );
    }

    // Expose as Flux for WebSocket streaming (used on Day 5)
    public Flux<StockDataDTO> streamStockUpdates() {
        return stockSink.asFlux();
    }
}
```

---

## Part 3 — Reactive SSE Endpoint in Consumer (25 min)

### Add Streaming REST endpoint in `ListenerController.java`

The consumer already has WebFlux (`spring-boot-starter-webflux`). Add SSE endpoint:

```java
// Add to ListenerController.java:
private final ReactiveStockConsumerService reactiveStockConsumerService;

@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<StockDataDTO> streamStockData() {
    // Clients connect here and get real-time stock updates as SSE
    return reactiveStockConsumerService.streamStockUpdates()
        .doOnSubscribe(s -> log.info("New SSE subscriber connected"))
        .doOnCancel(() -> log.info("SSE subscriber disconnected"));
}
```

### Test SSE endpoint:
```bash
curl -N http://localhost:8001/api/v1/stocks/stream
```
Then send stock data from producer. Observe the events streaming in real time.

---

## Part 4 — Add Reactive Producer in Producer App (20 min)

### Create `ReactiveKafkaProducerConfig.java` in producer
```java
@Configuration
public class ReactiveKafkaProducerConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Bean
    public ReactiveKafkaProducerTemplate<String, String> reactiveKafkaProducer() {
        SenderOptions<String, String> senderOptions = SenderOptions.create(Map.of(
            ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers,
            ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class,
            ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class,
            ProducerConfig.ACKS_CONFIG, "all",
            ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true
        ));
        return new ReactiveKafkaProducerTemplate<>(senderOptions);
    }
}
```

### Add reactive publish method to `PublisherService.java`:
```java
private final ReactiveKafkaProducerTemplate<String, String> reactiveKafkaProducer;

public Mono<String> produceReactive(String topicName, StockData stockData) {
    SenderRecord<String, String, String> record = SenderRecord.create(
        new ProducerRecord<>(topicName, stockData.getStock(),
            Util.convertObjectToJson(stockData)),
        stockData.getStock() // correlation metadata
    );
    return reactiveKafkaProducer.send(Mono.just(record))
        .map(result -> "Sent to partition: " + result.recordMetadata().partition())
        .onErrorReturn("Failed to send reactively");
}
```

---

## Theory Checkpoint (20 min)

1. **Threading:** Reactor Kafka polls Kafka on which thread? Where does `flatMap` processing happen? What does `subscribeOn(Schedulers.boundedElastic())` change?
2. **Backpressure:** If the DB is slow and messages arrive faster than saved — what happens to the `Sinks.Many` buffer of 1000?
3. **acks in reactive:** Does using `ReactiveKafkaProducerTemplate` change how `acks` work vs blocking `KafkaTemplate`?
4. **Auto-commit vs manual:** Why did we use `ENABLE_AUTO_COMMIT_CONFIG=false` for the `ReactiveKafkaConsumerTemplate`?

**Key Insight:**
> Reactive Kafka does NOT automatically give better throughput. It gives non-blocking integration — so your WebFlux app doesn't block its event loop threads. On a single-app basis, throughput is similar. The win is **resource efficiency** under high concurrency.

---

## Day 4 Deliverables
- [ ] `reactor-kafka` dependency added to both pom.xml files
- [ ] `ReactiveKafkaConfig.java` created with `ReceiverOptions` and manual commit
- [ ] `ReactiveStockConsumerService.java` processes messages reactively with `Sinks.Many` sink
- [ ] SSE endpoint `/api/v1/stocks/stream` works — `curl` shows real-time events
- [ ] Reactive producer sends a message via `ReactiveKafkaProducerTemplate`
- [ ] Observed different threads in logs: Kafka poll thread vs processing thread
