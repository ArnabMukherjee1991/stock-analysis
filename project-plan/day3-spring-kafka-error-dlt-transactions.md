# Day 3: Spring Kafka Deep Integration — Error Handling, DLT, Transactions

## Goal (2 hours)
Properly wire Spring Kafka in both producer and consumer — typed serializers, DefaultErrorHandler, Dead Letter Topics, retry with backoff, and transactional produce+consume.

If you enable SSL and Schema Registry, keep the same topic names and switch serializers first, then move consumers, then tighten compatibility checks.

---

## Part 1 — Typed JSON Serializer + Consumer Factory (30 min)

### Producer: Switch to typed `KafkaTemplate<String, StockData>`

The current producer uses raw `String` serialization via `Util.convertObjectToJson()` manually.  
Better approach: let Spring Kafka serialize using Jackson.

**Update `KafkaConfig.java` in producer:**
```java
// Add new dependency check in pom.xml first:
// spring-boot-starter-json is already transitive via spring-boot-starter-web

@Bean
public ProducerFactory<String, StockData> stockProducerFactory() {
    Map<String, Object> configProps = new HashMap<>();
    configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
    configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
    configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
    configProps.put(ProducerConfig.ACKS_CONFIG, "all");
    configProps.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
    configProps.put(ProducerConfig.RETRIES_CONFIG, 3);
    configProps.put(JsonSerializer.ADD_TYPE_INFO_HEADERS, false); // don't leak class names
    return new DefaultKafkaProducerFactory<>(configProps);
}

@Bean
public KafkaTemplate<String, StockData> stockKafkaTemplate() {
    return new KafkaTemplate<>(stockProducerFactory());
}
```

**Update `PublisherService.java`:**
```java
KafkaTemplate<String, StockData> kafkaTemplate;

// In produceData():
CompletableFuture<SendResult<String, StockData>> future =
    kafkaTemplate.send(topicName, stockData.getStock(), stockData);
```

### Consumer: Typed `@KafkaListener` with `JsonDeserializer`

**Update `ListenerConfig.java` in consumer:**
```java
@Configuration
public class ListenerConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Bean
    public ConsumerFactory<String, StockDataDTO> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "stock-consumer-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        props.put(JsonDeserializer.TRUSTED_PACKAGES, "com.myexchange.*");
        props.put(JsonDeserializer.VALUE_DEFAULT_TYPE, StockDataDTO.class.getName());
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        return new DefaultKafkaConsumerFactory<>(props,
            new StringDeserializer(),
            new JsonDeserializer<>(StockDataDTO.class, false));
    }

    @Bean
    public ConcurrentMessageListenerContainer<String, StockDataDTO>
    kafkaListenerContainerFactory() { ... }
}
```

**Update `StockConsumerService.java` — use typed record:**
```java
@KafkaListener(topics = "stock-data", groupId = "stock-consumer-group")
public void listenToStockData(ConsumerRecord<String, StockDataDTO> record) {
    log.info("Received: partition={} offset={} key={}", 
        record.partition(), record.offset(), record.key());
    stockDataRepository.save(record.value()).subscribe();
}
```

---

## Part 2 — DefaultErrorHandler + Dead Letter Topic (40 min)

### Add `spring-kafka-dlt` support — Update `ListenerConfig.java` in consumer

```java
@Bean
public DeadLetterPublishingRecoverer deadLetterPublishingRecoverer(
        KafkaTemplate<String, StockDataDTO> kafkaTemplate) {
    // DLT topic convention: stock-data.DLT
    return new DeadLetterPublishingRecoverer(kafkaTemplate,
        (record, ex) -> new TopicPartition(record.topic() + ".DLT", 0));
}

@Bean
public DefaultErrorHandler errorHandler(DeadLetterPublishingRecoverer recoverer) {
    // Retry 3 times with exponential backoff: 1s, 2s, 4s
    ExponentialBackOffWithMaxRetries backoff = new ExponentialBackOffWithMaxRetries(3);
    backoff.setInitialInterval(1000L);
    backoff.setMultiplier(2.0);
    backoff.setMaxInterval(10000L);

    DefaultErrorHandler handler = new DefaultErrorHandler(recoverer, backoff);
    // Don't retry on deserialization errors — send straight to DLT
    handler.addNotRetryableExceptions(JsonProcessingException.class);
    return handler;
}

@Bean
public ConcurrentKafkaListenerContainerFactory<String, StockDataDTO>
kafkaListenerContainerFactory(ConsumerFactory<String, StockDataDTO> consumerFactory,
                              DefaultErrorHandler errorHandler) {
    ConcurrentKafkaListenerContainerFactory<String, StockDataDTO> factory =
        new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory);
    factory.setCommonErrorHandler(errorHandler);
    factory.setConcurrency(3); // one thread per partition
    return factory;
}
```

**Create DLT topic via AdminController (producer app):**
```
POST /kafka/topics/add
{ "name": "stock-data.DLT", "partitions": 1, "replicationFactor": 1 }
```

### Add a DLT consumer to observe dead letters:
```java
// In StockConsumerService.java — add:
@KafkaListener(topics = "stock-data.DLT", groupId = "stock-dlt-monitor")
public void listenToDLT(ConsumerRecord<String, String> record) {
    log.error("MESSAGE IN DLT: topic={} offset={} cause={}",
        record.topic(), record.offset(),
        record.headers().lastHeader("kafka_exception-message"));
}
```

### Test it — simulate a poison pill:
Temporarily add a throw in `listenToStockData()`:
```java
if (record.key().equals("POISON")) throw new RuntimeException("Simulated failure");
```
Send a message with stock="POISON" from producer. Observe 3 retries in logs, then message lands in `stock-data.DLT`.

---

## Part 3 — Transactional Producer (30 min)

### Add `KafkaTransactionManager` to producer

**Update `KafkaConfig.java` in producer:**
```java
@Bean
public ProducerFactory<String, StockData> transactionalProducerFactory() {
    DefaultKafkaProducerFactory<String, StockData> factory = 
        new DefaultKafkaProducerFactory<>(configProps());
    factory.setTransactionIdPrefix("stock-tx-");  // enables transactions
    return factory;
}

@Bean
public KafkaTransactionManager<String, StockData> kafkaTransactionManager() {
    return new KafkaTransactionManager<>(transactionalProducerFactory());
}
```

**Use `@Transactional` in PublisherService:**
```java
@Transactional("kafkaTransactionManager")
public String produceDataTransactional(String topicName, StockData stockData) {
    // Both sends are atomic — either both commit or both abort
    kafkaTemplate.send(topicName, stockData.getStock(), stockData);
    kafkaTemplate.send(topicName + "-audit", stockData.getStock(), stockData);
    return "Sent transactionally";
}
```

> **Interview Insight:** Transactional producer sends to multiple topics atomically. Consumer must set `isolation.level=read_committed` to see only committed messages. This is the foundation of exactly-once semantics.

---

## Theory Checkpoint (20 min)

1. What happens when `DefaultErrorHandler` exhausts all retries? Where does the message go?
2. Why should `JsonProcessingException` NOT be retried (added to non-retryable list)?
3. What is "zombie fencing" in Kafka transactions? How does `transactional.id` help?
4. If the producer crashes after sending but before commit — will the consumer (with `read_committed`) see that message?

---

## Day 3 Deliverables
- [ ] Producer uses typed `KafkaTemplate<String, StockData>` with `JsonSerializer`
- [ ] Consumer uses `JsonDeserializer<StockDataDTO>` with `TRUSTED_PACKAGES`
- [ ] `DefaultErrorHandler` with exponential backoff configured
- [ ] Dead Letter Topic `stock-data.DLT` receives poison pill messages
- [ ] DLT consumer logs dead letter messages with cause header
- [ ] Transactional producer configured with `stock-tx-` prefix
- [ ] Observed 3 retries then DLT routing in consumer logs
