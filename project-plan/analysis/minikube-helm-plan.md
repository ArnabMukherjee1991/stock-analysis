# Minikube + Helm Plan

## Goal
Move the local Kafka stack from Docker Compose to Minikube with Helm so deploys become repeatable and environment configuration becomes chart-driven.

## What This Phase Owns
- Local Kubernetes cluster via Minikube
- Helm charts for Kafka, Kafka UI, and the Spring Boot services
- Kubernetes secrets and config maps
- Storage classes and persistent volumes
- Deployment values for local and secure modes

## Step-by-Step Process
1. Stand up Minikube with enough CPU and memory for Kafka.
2. Install Helm and create a namespace for the stack.
3. Create or vendor Helm charts for Kafka and Kafka UI.
4. Add Helm charts for the producer and consumer apps.
5. Move local configuration into `values.yaml` files.
6. Map SSL and Schema Registry settings to Kubernetes secrets.
7. Deploy Kafka first, then Kafka UI, then Postgres, then the apps.
8. Run the same smoke tests you used in Docker Compose.
9. Scale partitions and replicas after the base install is stable.

## Chart Structure
- `infra/helm/charts/kafka/`
- `infra/helm/charts/kafka-ui/`
- `infra/helm/charts/exchange-data-producer/`
- `infra/helm/charts/exchange-data-consumer/`
- `infra/helm/charts/shared/` if you want common templates

## Values Strategy
- `values-local.yaml` for plaintext local use.
- `values-ssl.yaml` for broker and client TLS.
- `values-schema.yaml` for Schema Registry enabled deploys.
- `values-test.yaml` for CI and smoke test settings.

## Kubernetes Checklist
- Namespace exists before install.
- Secrets are stored outside the chart templates.
- Persistent volumes are mapped for Kafka state.
- Readiness and liveness probes are defined.
- Resource requests and limits are set.
- Kafka UI points to the in-cluster bootstrap service.

## Exit Criteria
- Helm install and upgrade work repeatably.
- Pods recover cleanly after restart.
- The same app configs work in Compose and Minikube with only values changes.
- You can scale the deployment without editing manifests manually.
