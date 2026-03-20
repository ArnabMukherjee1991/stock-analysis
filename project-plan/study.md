# 1-Week Kafka Mastery Plan for Senior Java Architect

## Your Current State Assessment
With 13+ years Java experience, you don't need to learn Java - you need to **translate your existing distributed systems knowledge into Kafka concepts** and identify the gaps between what you know and what interviewers expect at senior level.

---

## Week Overview

| Day | Focus Area | Time Required |
|-----|------------|---------------|
| Day 1 | Kafka Architecture & Core Concepts | 3-4 hours |
| Day 2 | Multi-Broker Clusters & Reliability | 3-4 hours |
| Day 3 | Kafka with Spring Boot (The Right Way) | 4-5 hours |
| Day 4 | WebFlux/Reactive Kafka Integration | 3-4 hours |
| Day 5 | WebSocket + Kafka Real-time Patterns | 3-4 hours |
| Day 6 | Performance Tuning & Troubleshooting | 3-4 hours |
| Day 7 | Mock Interviews & Revision | 4-5 hours |

## Rollout Track

Use [analysis/README.md](analysis/README.md) as the operating guide for the workspace:
- Decide the payload format and schema strategy first.
- Secure Kafka with SSL next.
- Enable Schema Registry and validation after transport security is stable.
- Scale partitions and consumers before adding more brokers.

---

## Day 1: Kafka Architecture & Core Concepts

### Morning Session (2 hours) - Theory Brush-up

**Core Concepts to Master:**
- Topics, Partitions, Offsets - but go deeper:
  - How partition assignment affects parallelism
  - Offset commit strategies and exactly-once semantics
  - Log compaction - when and why

**Broker Internals:**
- Controller responsibilities in cluster coordination
- ISR (In-Sync Replicas) - how it's maintained, what breaks it
- Leader election process - what triggers it, how long it takes

**Senior-Level Questions to Answer:**
- "If you have 10 partitions and 3 consumers in a group, what happens?"
- "How would you design for message ordering across retries?"
- "What's the trade-off between more partitions vs more brokers?"

### Afternoon Session (1.5 hours) - Deep Dive Topics

**Key Papers/Blogs to Read:**
- LinkedIn Engineering's Kafka original paper (summary)
- Confluent blog: "Exactly-Once Semantics in Kafka"
- "Kafka Replication Protocol" deep dive

**Tricky Areas to Internalize:**
- Idempotent producers - how they prevent duplicates
- Transactional producers and consumers
- Zombie fencing with transactional IDs

### Evening Review (30 mins)
- Create a mental map: how data flows from producer to consumer
- Identify gaps in your understanding of replication

---

## Day 2: Multi-Broker Clusters & Reliability Patterns

### Morning Session (2 hours) - Cluster Architecture

**Multi-Broker Deep Dive:**
- Rack awareness - why it matters for disaster recovery
- Replica distribution strategy
- Preferred replica election
- Unclean leader election - when to enable, risks

**Reliability Patterns:**
- **acks=all** vs **acks=1** vs **acks=0** - real tradeoffs
- **min.insync.replicas** - how it prevents false acks
- Replication factor decisions based on use case

**Senior-Level Scenarios to Reason Through:**
- "2 brokers down out of 5 with replication factor 3 - what happens to producers? consumers?"
- "Network partition separates controller - how does cluster behave?"
- "You need zero data loss but can't afford latency - impossible? Compromise?"

### Afternoon Session (1.5 hours) - Operations & Monitoring

**What Senior Architects Must Know:**
- Kafka metrics that matter:
  - Under-replicated partitions
  - ISR shrinks per second
  - Request handler idle ratio
- Rebalancing triggers and how to minimize impact
- Partition reassignment strategies

**Tricky Concepts:**
- Static membership vs dynamic membership
- Cooperative rebalancing vs eager rebalancing
- Sticky partition assignor - why it exists

### Evening Review (30 mins)
- Draw a 5-broker cluster with replication factor 3
- Simulate broker failures in your mind, trace impact

---

## Day 3: Spring Boot + Kafka Integration

### Morning Session (2 hours) - Spring Kafka Fundamentals

**Key Components to Understand:**
- `KafkaTemplate` - threading model, blocking behavior
- `@KafkaListener` - how it works under the hood
- Container factories - concurrent vs single-threaded
- `ConsumerFactory` and `ProducerFactory` configuration

**Configuration Mastery:**
- Thread pooling in listeners
- Error handling strategies:
  - `SeekToCurrentErrorHandler` (old) vs `DefaultErrorHandler` (new)
  - Dead Letter Topic patterns
  - Retry topics with exponential backoff

**Senior-Level Nuances:**
- "How does Spring manage consumer group coordination?"
- "What happens when a @KafkaListener throws exception?"
- "How to handle poison pills without blocking the entire consumer?"

### Afternoon Session (1.5 hours) - Advanced Spring Integration

**Transaction Patterns:**
- `@Transactional` with Kafka - database and Kafka in same transaction
- ChainedKafkaTransactionManager - when needed
- Idempotent receivers - preventing duplicate processing

**Container Customization:**
- ConcurrentMessageListenerContainer vs KafkaMessageListenerContainer
- Pause/resume capabilities for backpressure
- Record filtering strategies

**Tricky Scenarios:**
- Manual offset commits with complex processing
- Rebalancing listeners - how to handle before/after rebalance
- Dynamic topic subscription at runtime

### Evening Review (30 mins)
- Map Spring concepts to Kafka native concepts
- Identify what Spring abstracts and where leaks happen

---

## Day 4: WebFlux + Reactive Kafka

### Morning Session (2 hours) - Reactive Foundations

**Reactive Programming Refresh:**
- Project Reactor core concepts (but focused on integration)
- Backpressure - how it works end-to-end
- Non-blocking vs async - critical distinction

**Reactor Kafka:**
- `ReactiveKafkaProducerTemplate` vs regular `KafkaTemplate`
- `KafkaReceiver` and `KafkaSender` APIs
- How backpressure propagates from consumer to broker

**Senior-Level Questions:**
- "Does reactive Kafka give better throughput? Always? When not?"
- "Threading model in Reactor Kafka - how many threads, who manages them?"
- "How to handle backpressure when downstream is slow?"

### Afternoon Session (1.5 hours) - Integration Patterns

**Reactive Flow Patterns:**
- Streaming from Kafka to WebFlux endpoints
- Windowing and buffering strategies
- Combining multiple Kafka streams
- Error handling in reactive flows - retry, fallback, circuit breaker

**Performance Implications:**
- When reactive Kafka makes sense (and when it doesn't)
- Memory pressure with large messages
- Scheduler strategies - parallel vs boundedElastic

**Tricky Concepts:**
- Transaction support in reactive Kafka (spoiler: limited)
- Exactly-once in reactive world
- Testing reactive Kafka flows

### Evening Review (30 mins)
- Contrast imperative Spring Kafka vs reactive approach
- Define 3 scenarios where you'd choose each

---

## Day 5: WebSocket + Kafka Real-time Patterns

### Morning Session (2 hours) - WebSocket Fundamentals

**WebSocket Refresh:**
- STOMP protocol over WebSocket
- Spring's WebSocket support - `@MessageMapping`, `@SendTo`
- Session management and user destinations

**Real-time Patterns:**
- Fan-out: one Kafka message to many WebSocket clients
- User-specific messages (private queues)
- Topic subscriptions via WebSocket

**Senior-Level Architecture:**
- "1M concurrent WebSocket connections - can Kafka handle the fan-out?"
- "How to scale WebSocket servers with Kafka as the backbone?"
- "What's the latency budget from Kafka produce to WebSocket delivery?"

### Afternoon Session (1.5 hours) - Integration Deep Dive

**Integration Patterns:**
- `@KafkaListener` + `SimpMessagingTemplate` - bridging layers
- Inbound: WebSocket message → Kafka producer
- Outbound: Kafka consumer → WebSocket broadcast
- Handling disconnects during message processing

**Scalability Considerations:**
- Session affinity vs stateless WebSocket servers
- Using Kafka partitions as a scaling primitive
- Consumer groups per WebSocket server instance

**Tricky Scenarios:**
- "WebSocket client disconnects while Kafka message is processing"
- "Broadcast storm - same message to 100K clients at once"
- "Ordering guarantees across reconnections"

### Evening Review (30 mins)
- Design a chat system with Kafka + WebSocket
- Identify failure modes in your design

---

## Day 6: Performance Tuning & Troubleshooting

### Morning Session (2 hours) - Performance Deep Dive

**Producer Tuning:**
- Batch size, linger.ms, compression - interactions
- max.in.flight.requests.per.connection - ordering vs throughput
- buffer.memory and block.on.buffer.full

**Consumer Tuning:**
- fetch.min.bytes, fetch.max.wait.ms, max.partition.fetch.bytes
- max.poll.records - finding the sweet spot
- session.timeout.ms vs heartbeat.interval.ms

**Broker Tuning:**
- num.network.threads vs num.io.threads
- log.segment.bytes and log.retention policies
- Background threads and their impact

**Senior-Level Tradeoffs:**
- "You need sub-10ms latency but high throughput - impossible?"
- "How to size partitions for your workload"
- "When does compression hurt more than help?"

### Afternoon Session (1.5 hours) - Troubleshooting Scenarios

**Common Problems & Solutions:**
- Consumer lag growing - diagnosis steps
- Rebalancing too frequent - how to debug
- Under-replicated partitions - investigation
- OutOfMemory in Kafka clients - common causes

**Diagnostic Tools:**
- kafka-consumer-groups CLI
- kafka-reassign-partitions
- JMX metrics that matter
- Kafka Lag Exporter

**Real War Stories (Prepare Your Own):**
- The time a misconfigured consumer took down production
- The poison pill that crashed everything
- The network partition that taught you about min.insync.replicas

### Evening Review (30 mins)
- Create a checklist for investigating high consumer lag
- Define your "first 5 minutes" when Kafka cluster degrades

---

## Day 7: Mock Interviews & Final Revision

### Morning Session (3 hours) - Mock Interviews

**Self-Mock Format:**
- 30 mins: Whiteboard a Kafka design (ride-sharing, gaming leaderboard, financial system)
- 30 mins: Deep dive on your design choices
- 30 mins: Troubleshooting scenario
- 30 mins: Code review (conceptual) of Spring Kafka configuration

**Question Categories to Practice:**
1. **Architecture Design:**
   - "Design a real-time fraud detection system"
   - "Design a notification service with Kafka"
   - "How would you migrate from RabbitMQ to Kafka?"

2. **Deep Dive Questions:**
   - "Explain Kafka's replication protocol in detail"
   - "What happens when a broker dies and comes back?"
   - "How does Kafka achieve exactly-once semantics?"

3. **Troubleshooting:**
   - "Consumers stopped consuming but no errors"
   - "Messages are being duplicated despite idempotent producer"
   - "Producer performance degraded after adding 3rd broker"

4. **Spring-Specific:**
   - "@KafkaListener method takes 30 seconds - problems?"
   - "How to implement circuit breaker for Kafka producer"
   - "Multiple consumers in same group - how partitions assigned"

### Afternoon Session (1.5 hours) - Key Concepts Quick Revision

**Flashcard-Worthy Concepts:**
- Exactly-once semantics prerequisites
- ISR condition for leader election
- Transactional producer flow
- Cooperative rebalancing vs eager
- Sticky partition assignor
- Idempotent producer sequence numbers
- Zombie fencing mechanism
- Log compaction vs retention

**Key Numbers to Memorize:**
- Default replication factor
- Default partition count
- Typical ISR timeout
- Consumer poll timeout defaults
- Heartbeat intervals

### Evening Session (1 hour) - Mental Preparation

**Confidence Builders:**
- Review your 13 years of distributed systems experience
- Map each Kafka concept to something you've done:
  - Partitioning → Your sharding strategy in database
  - Replication → Your database replication knowledge
  - Consumer groups → Your load balancing experience

**Your Unique Value Proposition:**
"You've built systems that handled millions of users. Kafka is just another distributed system. You understand CAP theorem, you've debugged production at 3 AM, you've made tradeoffs under pressure. Kafka is just vocabulary on top of that foundation."

**Final Mindset:**
- The interview is a conversation between peers
- "I don't know" followed by "but here's how I'd reason about it" is acceptable
- Your experience in other distributed systems transfers directly

---

## Quick Reference: Key Concepts to Revise Night Before

| Category | Must-Know Concepts |
|----------|---------------------|
| **Core** | Topics, Partitions, Offsets, Brokers, Zookeeper/KRaft, Producers, Consumers, Consumer Groups |
| **Reliability** | Replication, ISR, acks, min.insync.replicas, unclean leader election, exactly-once semantics |
| **Performance** | Batching, compression, threading model, page cache, zero-copy |
| **Operations** | Rebalancing, partition assignment, reassignment, preferred leader, metrics |
| **Spring** | @KafkaListener, KafkaTemplate, containers, error handlers, transactions |
| **Reactive** | Backpressure, Reactor Kafka, non-blocking vs async, schedulers |
| **WebSocket** | STOMP, SimpMessagingTemplate, session management, fan-out patterns |

---

## Your Success Path

**Days 1-2**: Build mental model of Kafka internals
**Days 3-5**: Map that model to Spring/Reactive/WebSocket
**Day 6**: Learn how to fix it when it breaks
**Day 7**: Prove you can architect with it

Remember: With 13 years experience, you're not learning Kafka from scratch. You're learning Kafka's specific implementation of distributed systems concepts you already understand. The patterns are the same - just different terminology.

**You've got this. The fear is just excitement in disguise.**