# Day 2: Multi-Broker Reliability + Producer acks + Idempotence

## Goal (2 hours)
Operate the current 3-broker Confluent KRaft cluster in `stock-analysis-ops`, configure `acks=all`, `min.insync.replicas`, and idempotent producer. Observe what happens in failure scenarios and keep the topic plan ready for later scale-up.

---

## Part 1 — Multi-Broker Docker Compose (40 min)

### Update `stock-analysis-ops/docker-compose.yaml`

The workspace already uses `kafka-1`, `kafka-2`, and `kafka-3` in KRaft mode. Keep Kafka UI pointed at all three brokers:

```yaml
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka-1:9092,kafka-2:9092,kafka-3:9092
```

Use external ports `19092`, `19093`, and `19094` for host access.

**Restart:**
```bash
docker compose -f stock-analysis-ops/docker-compose.yaml up -d
```

### Create a topic with replication factor 2
```bash
docker exec -it kafka-1 kafka-topics --bootstrap-server localhost:9092 \
  --create --topic stock-data-reliable \
  --partitions 3 --replication-factor 3
```

**Describe topic — observe leader vs replicas:**
```bash
docker exec -it kafka-1 kafka-topics --bootstrap-server localhost:9092 \
  --describe --topic stock-data-reliable
```

---

## Part 2 — Producer Reliability Config (40 min)

### Update `KafkaConfig.java` in producer

Add `acks=all`, `min.insync.replicas=2`, retries, and idempotence:

```java
// In producerFactory() method, add to configProps:
configProps.put(ProducerConfig.ACKS_CONFIG, "all");
configProps.put(ProducerConfig.RETRIES_CONFIG, 3);
configProps.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
configProps.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 1);
// Batching
configProps.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);
configProps.put(ProducerConfig.LINGER_MS_CONFIG, 5);
```

> **Interview Insight:** `ENABLE_IDEMPOTENCE_CONFIG=true` requires `acks=all` and `retries > 0`. Spring Kafka will throw if these conflict. Setting `MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION=1` ensures ordering even with retries.

### Update `PublisherService.java` to use typed KafkaTemplate and log metadata

Change `KafkaTemplate kafkaTemplate` (raw type) to `KafkaTemplate<String, String> kafkaTemplate`, and add callback logging:

```java
KafkaTemplate<String, String> kafkaTemplate;  // typed

// In produceData():
CompletableFuture<SendResult<String, String>> future =
    kafkaTemplate.send(topicName, stockData.getStock(), Util.convertObjectToJson(stockData));

future.whenComplete((result, ex) -> {
    if (ex == null) {
        log.info("Sent to topic={} partition={} offset={}",
            result.getRecordMetadata().topic(),
            result.getRecordMetadata().partition(),
            result.getRecordMetadata().offset());
    } else {
        log.error("Failed to send: {}", ex.getMessage());
    }
});
```

---

## Part 3 — Simulate Broker Failure (20 min)

### Scenario 1: Stop one broker, observe ISR shrink
```bash
docker stop kafka-3
```
Then call the producer API. With `acks=all`, replication factor 3, and `min.insync.replicas=2`, writes should continue as long as two brokers stay in ISR.

**Check under-replicated partitions:**
```bash
docker exec -it kafka-1 kafka-topics --bootstrap-server localhost:9092 --describe --topic stock-data-reliable
```

Observe the `Isr:` column shrink by one broker. If you stop two brokers, the next producer write should fail because the ISR is smaller than `min.insync.replicas=2`.

### Restore kafka2 and observe leader re-election:
```bash
docker start kafka-3
# Wait 10-15 seconds, then:
docker exec -it kafka-1 kafka-topics --bootstrap-server localhost:9092 --describe --topic stock-data-reliable
```
Observe ISR recovering.

---

## Part 4 — Theory Checkpoint (20 min)

Answer these:
1. With RF=3, if 2 brokers go down — can consumers still READ? Can producers still WRITE with `acks=all`?
2. What does `unclean.leader.election.enable=true` mean? When would you enable it despite data loss risk?
3. Why does `ENABLE_IDEMPOTENCE_CONFIG=true` prevent duplicates across producer retries?
4. Difference between `acks=1` (leader ack) vs `acks=all` (ISR ack) for a fintech stock trading system?

**Mental Model:**
- `acks=0`: Fire and forget — max throughput, possible loss
- `acks=1`: Leader confirms — common default, leader failure = loss
- `acks=all` + `min.insync.replicas=2`: Majority confirms — zero loss guarantee, slight latency

---

## Day 2 Deliverables
- [ ] 3-broker Kafka cluster running and visible in Kafka UI
- [ ] `stock-data-reliable` topic with RF=3 and 3 partitions
- [ ] Producer configured with `acks=all`, idempotence, retries
- [ ] Observed ISR shrink when one broker stopped
- [ ] Observed `NotEnoughReplicasException` when ISR drops below 2
- [ ] Producer logs show partition + offset per message
