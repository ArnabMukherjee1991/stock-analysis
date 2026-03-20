# Stock Analysis

Monorepo for a Kafka-based stock analysis proof of concept.

## Projects

- `exchange-data-producer`: Spring Boot producer and stock CRUD service.
- `exchange-data-consumer`: Spring Boot consumer, reactive processing, and real-time delivery.
- `stock-analysis-ops`: Infrastructure and local environment assets that the pipeline runs through.
- `project-plan`: Execution notes, gap analysis, and rollout planning.

## Current Plan

Start with the planning docs in `project-plan/`, then build the ops layer first, then move into the producer and consumer workstreams.
