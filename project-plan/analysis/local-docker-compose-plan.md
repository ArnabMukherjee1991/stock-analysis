# Local Docker Compose Plan

## Goal
Run the whole Kafka stack locally with Docker Compose, shell scripts for certificates, SSL-enabled broker/client paths, and end-to-end testing before moving to Kubernetes.

## What This Phase Owns
- Kafka brokers in local Compose
- Kafka UI
- Postgres and any shared local dependencies
- Certificate generation scripts
- SSL configuration for brokers and clients
- End-to-end smoke tests

## Step-by-Step Process
1. Keep the current 3-broker Confluent KRaft compose as the baseline.
2. Add a `scripts/certs/` folder for certificate generation.
3. Add a `scripts/ssl/` folder for truststore and keystore setup.
4. Add a `scripts/test/` folder for topic creation and smoke tests.
5. Start with plaintext local Compose, then enable SSL through a second compose profile.
6. Verify Kafka UI, topic creation, producer sends, and consumer reads.
7. Add failure drills for broker stop and recovery.
8. Lock in topic creation defaults before moving to Kubernetes.

## Scripts To Add
- `scripts/certs/generate-certs.sh`
- `scripts/ssl/create-truststores.sh`
- `scripts/ssl/create-keystores.sh`
- `scripts/test/create-topics.sh`
- `scripts/test/run-smoke-test.sh`
- `scripts/test/run-failure-drill.sh`

## SSL Checklist
- Each broker gets its own certificate.
- Local clients trust the broker CA.
- Broker listeners expose SSL on a separate port.
- Plaintext remains available only for local fallback if needed.
- Secrets are not hardcoded in compose files.

## End-to-End Test Checklist
- Producer can publish a JSON event.
- Consumer can read and persist the event.
- Kafka UI can inspect the cluster.
- SSL clients can connect successfully.
- Schema validation is ready for the next phase.
- Broker failure does not lose acknowledged messages.

## Exit Criteria
- Compose stack starts cleanly.
- SSL scripts are repeatable.
- Topic creation is scripted.
- Smoke tests pass on a clean machine.
