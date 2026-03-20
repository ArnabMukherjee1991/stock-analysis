# SSL, Schema Registry, and Validation Plan

## Delivery Sequence
1. Local Docker Compose with scripts.
2. Minikube with Helm.
3. AWS with Terraform and Helm.

## Data Formats You Can Use With Kafka
Kafka itself is format-agnostic. The value is a byte array on the wire. In practice, your choices are:

- `String`: simple and easy, usually JSON text in early stages.
- `byte[]`: raw binary when you already own the codec.
- JSON: human-readable, easy for Spring Boot, good for local development.
- Avro: best fit when you want Schema Registry enforcement and compact payloads.
- Protobuf: good for cross-language services with strict contracts.
- CSV/XML: useful for migration from legacy feeds, but not ideal for long-term Kafka contracts.

## Best Fit For This Workspace
- Current local development: JSON payloads.
- Next step for governance: Avro or JSON Schema with Schema Registry.
- Strict cross-language contracts: Protobuf.

## Local Compose First
- Use Docker Compose as the fastest way to verify broker settings.
- Add scripts for certificate generation so SSL is repeatable.
- Add smoke tests that prove end-to-end send, consume, and persistence.
- Keep topic creation scripted so the cluster can be recreated quickly.

## Minikube Next
- Port the same values into Helm charts.
- Keep the same topic names and service names.
- Replace shell scripts with chart values and Kubernetes secrets.
- Run the same smoke tests against the cluster IP or ingress path.

## AWS Last
- Use Terraform to provision the AWS foundation.
- Use Helm to install the workloads on EKS.
- Keep the chart values close to the Minikube setup.
- Add remote state, environment separation, and secret management.

## SSL Enablement Checklist
1. Create broker certificates for each Kafka node.
2. Configure brokers with SSL listeners.
3. Expose a separate internal listener for broker-to-broker traffic.
4. Give each Spring Boot service a truststore.
5. Add a keystore only if you want mutual TLS.
6. Keep plaintext enabled only in local development profiles.

## Schema Registry Enablement Checklist
1. Run Schema Registry on the same Docker network as Kafka.
2. Point producer and consumer services to `schema.registry.url`.
3. Choose a compatibility mode before traffic starts.
4. Use typed payloads and schema-aware serializers.
5. Reject incompatible schema changes in CI.

## Validation Rules
- New fields should be backward compatible.
- Removed fields should only happen after all consumers are migrated.
- Never change a field type without a compatibility plan.
- Use subject naming consistently across producer and consumer.

## Operational Guardrails
- Create topics before turning on application traffic.
- Start with replication factor 3 and `min.insync.replicas=2`.
- Scale partitions only after consumer lag and key distribution are understood.
- Add brokers later for availability, not as a substitute for partition planning.

## Spring Boot Service Updates
- Make Kafka bootstrap servers configurable from environment variables.
- Add optional SSL and Schema Registry properties to both services.
- Keep local plaintext defaults so the apps still run in this workspace.
- Replace shared mutable consumer buffers with repository-backed reads.

## Deployment Rule
- Do not jump to AWS before the local Compose and Minikube stages are stable.
- Keep the same payload model across all three phases to reduce drift.
