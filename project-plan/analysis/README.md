# Analysis

This folder turns the Kafka rollout into an executable plan for the current workspace.

The operational path is `stock-analysis-ops` first, then producer and consumer on top of that foundation.

## Track Overview
- [Local Docker Compose Plan](local-docker-compose-plan.md)
- [Minikube + Helm Plan](minikube-helm-plan.md)
- [AWS Terraform Plan](aws-terraform-plan.md)
- [Security and Schema Plan](security-and-schema-plan.md)
- [Phase-by-Phase Rollout](phase-by-phase-rollout.md)

## What Kafka Can Carry
Kafka only stores bytes. The payload can be:
- `String` for simple text or JSON text
- `byte[]` for raw binary
- JSON objects serialized as text or via schema-aware serializers
- Avro for compact typed records with schema evolution
- Protobuf for strongly typed cross-language payloads
- CSV or XML when you are migrating legacy feeds, although these are usually transitional formats

## Recommended Format for This Repo
- Keep the current apps working with JSON for the first local phase.
- Move to schema-aware payloads once Schema Registry is enabled.
- Prefer Avro or JSON Schema for validation and compatibility control.
- Use Protobuf only if you need stricter contracts across multiple languages.

## Phased Rollout
1. Build the local Docker Compose foundation in `stock-analysis-ops` with scripts for certificates and smoke tests.
2. Enable SSL locally and prove end-to-end connectivity.
3. Add Schema Registry and schema validation locally.
4. Move the same topology to Minikube with Helm charts.
5. Keep local and Kubernetes values files aligned.
6. Promote the same app and chart structure to AWS with Terraform.
7. Scale partitions and consumers before adding more brokers.
8. Add automated tests and failure drills at each stage.

## Security and Validation Path
- For SSL, configure broker certificates, truststores, and client truststores.
- For mutual TLS, add keystores for both clients and brokers.
- For Schema Registry, set compatibility mode before production traffic.
- Validate schemas in CI before code reaches the cluster.

## Spring Boot Service Changes
- Producer should publish typed events and preserve key-based ordering.
- Consumer should deserialize to typed DTOs and avoid shared mutable buffers.
- Both services should read Kafka, SSL, and Schema Registry settings from environment variables.
- Use separate config profiles for local plaintext and secured environments.

## Files To Read Next
- [project-plan/README.md](../README.md)
- [project-plan/gap-analysis.md](../gap-analysis.md)
- [project-plan/day2-multi-broker-reliability.md](../day2-multi-broker-reliability.md)
- [project-plan/day3-spring-kafka-error-dlt-transactions.md](../day3-spring-kafka-error-dlt-transactions.md)
