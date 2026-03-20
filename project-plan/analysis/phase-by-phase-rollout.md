# Phase-by-Phase Kafka Rollout

## Phase 0: Inventory
- Confirm the current services: producer, consumer, Postgres, Kafka UI.
- Confirm ports and networks.
- Confirm whether the repo is a git monorepo.
- Confirm topic names and key strategy.

## Phase 1: Local Kafka Foundation
- Run the 3-broker KRaft cluster.
- Keep Kafka UI connected to all brokers.
- Keep Postgres isolated on the same shared network.
- Use `confluentinc/cp-kafka:8.2.0` for the brokers.
- Use external ports `19092`, `19093`, and `19094`.

## Phase 2: Topic Creation
- Create topics with the final partition count before application traffic.
- Use replication factor 3 for business topics.
- Set `min.insync.replicas=2`.
- Validate the topic leader, ISR, and replicas.
- Do not start with a single partition if you know you will scale consumers later.

## Phase 3: Producer Hardening
- Enable `acks=all`.
- Enable idempotence.
- Add retries and bounded in-flight requests.
- Log partition and offset on every successful send.
- Keep the message key stable for ordering.

## Phase 4: Consumer Hardening
- Use typed deserialization.
- Remove shared mutable state.
- Persist records directly to the repository.
- Add dead-letter handling for poison messages.
- Tune the consumer concurrency to match partitions.

## Phase 5: SSL
- Turn on TLS for client and broker connections.
- Keep local plaintext as a profile, not as the production target.
- Verify truststore and keystore paths before rollout.

## Phase 6: Schema Registry
- Pick one schema family first.
- Start with JSON Schema if you want the smallest migration step.
- Move to Avro if you want the strongest compatibility story in a Java-first codebase.
- Validate every schema change before release.

## Phase 7: Scaling
- Increase partitions before adding more consumer instances.
- Add brokers only when storage, availability, or throughput needs justify it.
- Rebalance after each scale event and verify lag.
- Keep key distribution even so one partition does not become a hotspot.

## Phase 8: Testing
- Run failure drills for broker loss.
- Run producer retry tests.
- Run consumer poison-pill tests.
- Run schema compatibility checks.
- Run EmbeddedKafka tests in CI.
