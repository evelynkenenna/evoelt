# EvoELT

**Replayable event-processing architecture for reproducible evaluation, retrospective testing, and evolving decision systems.**

EvoELT is a Kotlin-based event-processing service that preserves raw events alongside processed outputs so transformation logic can change without losing the ability to revisit historical inputs.

The core idea is simple: **store what happened separately from what you concluded about it.**

That pattern is useful anywhere business rules, scoring logic, recommendation systems, or AI-assisted decisions evolve over time. Historical events can be retrieved as ordered sequences, processed again with newer transformation logic, and stored as new outputs without overwriting the original inputs.

EvoELT is demonstrated through [ELTsim](https://github.com/evelynkenenna/eltsim), a Sims-inspired simulation that generates behavioral events and uses the pipeline to analyze how activities affect character outcomes.

## Why This Architecture Exists

Many systems persist only the latest derived result.

That works until the logic producing the result changes.

If a scoring algorithm, recommendation rule, model version, or business policy changes, teams may want to answer questions such as:

* What would the new logic have concluded about historical events?
* How do outputs differ between transformation versions?
* Can a decision be reproduced from the original inputs?
* Can historical data be reprocessed without mutating the source events?

EvoELT separates **immutable-style source events** from **derived processed events** so those questions remain answerable.

The repository is a proof of concept for that architectural pattern rather than a production-ready AI or ML platform.

## Architecture

At a high level:

```text
Event Producer
      |
      v
 AWS SQS FIFO
      |
      v
    EvoELT
      |
      +------> PostgreSQL
      |          |
      |          +--> Raw events / sequences
      |          +--> Processed events / sequences
      |
      v
Processing / Transformation Application
      |
      +------ REST lookup of historical sequence
      |
      +------ transformed result
      |
      v
    EvoELT
```

### Processing Flow

1. A producer submits a raw event through an AWS SQS FIFO queue.
2. EvoELT stores the raw event and associates it with optional sequence labels.
3. EvoELT notifies a processing application that a new event is available.
4. The processor can retrieve the current event and its historical predecessors through the REST API.
5. Transformation logic generates one or more processed events.
6. Processed events are returned to EvoELT and stored separately from their raw source data.
7. Different transformation versions can produce additional processed outputs from the same historical raw sequence.

This allows processing logic to evolve while retaining the original event history.

## Core Concepts

### Raw Event

The original input to the system.

Examples include an action, transaction, observation, state change, or other source event.

### Raw Sequence

A set of related raw events grouped by labels and ordered for historical retrieval.

### Processed Event

An output derived from one or more raw events.

A single raw event may be associated with multiple processed events.

### Processed Sequence

Related processed outputs grouped by labels.

Transformation or model version identifiers can be included in labels so multiple generations of derived results can coexist.

## Replay and Retrospective Evaluation

The architectural pattern centers on retaining source data independently from derived conclusions.

For example, assume a sequence contains:

```text
Raw events:
[A, AB, ABC, ABCD]
```

A transformation may produce:

```text
Processed outputs:
[A, BA, CBA, DCBA]
```

Later, the transformation can change.

Rather than rewriting the original inputs, another processor can retrieve the same historical raw sequence and generate a new set of processed outputs.

This creates a foundation for patterns such as:

* retrospective rule evaluation
* replayable recommendation logic
* versioned scoring systems
* historical model comparison
* reproducible decision pipelines
* event-driven analytics
* simulation and behavioral analysis

EvoELT itself does not train models or prescribe a particular transformation strategy. The processing application owns that logic.

## Components

| Component              | Responsibility                                                        |
| ---------------------- | --------------------------------------------------------------------- |
| EvoELT service         | Event ingestion, sequence management, persistence, and retrieval      |
| AWS SQS FIFO           | Event delivery between producers, EvoELT, and processing applications |
| PostgreSQL             | Storage for raw and processed events and their sequences              |
| Processing application | Domain-specific transformation, scoring, analysis, or decision logic  |
| REST API               | Historical event-sequence retrieval                                   |
| ELTsim                 | Demonstration application generating and consuming simulation events  |

## Data Model

The data model separates raw and processed information so derived results do not replace original events.

Sequence labels provide a flexible mechanism for grouping related events. Examples might include a customer identifier, entity identifier, activity identifier, experiment identifier, or transformation version.

## Example Event Flow

### 1. Producer → SQS → EvoELT

```json
{
  "labels": [
    "character_id-abc",
    "household_id-xyz",
    "activity_id-123"
  ],
  "data": "{\"activity\":\"crafting\",\"duration_minutes\":120,\"energy_cost\":15,\"happiness_gain\":30}"
}
```

Labels are optional. When provided, they associate the event with one or more logical sequences.

### 2. EvoELT → Processor

```json
{
  "raw_event_id": "c87880c6-0506-49d1-a570-f50198f867fd",
  "raw_sequence_labels": [
    "character_id-abc",
    "household_id-xyz",
    "activity_id-123"
  ]
}
```

### 3. Processor → EvoELT REST API

The processor can retrieve the raw event and its predecessors:

```text
GET /lookup/raw/sequence?raw_event_id=c87880c6-0506-49d1-a570-f50198f867fd
```

Example response:

```json
{
  "labels": [
    "character_id-abc",
    "household_id-xyz",
    "activity_id-123"
  ],
  "events": [
    {
      "id": "c87880c6-0506-49d1-a570-f50198f867fd",
      "data": "{\"activity\":\"crafting\",\"duration_minutes\":120,\"energy_cost\":15,\"happiness_gain\":30}",
      "order_id": 2,
      "created_dt": "2025-04-15 14:30:00.000000 +00:00"
    },
    {
      "id": "45f1af40-5d75-4375-b9f5-fe6d40b0a01a",
      "data": "{\"activity\":\"crafting\",\"duration_minutes\":30,\"energy_cost\":20,\"happiness_gain\":10}",
      "order_id": 1,
      "created_dt": "2025-04-15 13:00:00.000000 +00:00"
    }
  ],
  "total_events": 2,
  "total_pages": 1,
  "pageable": {
    "page_offset": 0,
    "page_number": 0,
    "page_size": 100,
    "sort": "rawSequenceOrderId: DESC"
  }
}
```

Pagination parameters:

| Parameter | JSON key      | Default |
| --------- | ------------- | ------: |
| `page`    | `page_number` |     `0` |
| `size`    | `page_size`   |   `100` |

### 4. Processor → EvoELT

The processor returns a derived event:

```json
{
  "labels": [
    "character_id-abc",
    "household_id-xyz",
    "activity_id-123",
    "transformation_version-1.0.0"
  ],
  "raw_event_id": "c87880c6-0506-49d1-a570-f50198f867fd",
  "data": "{\"average_energy_cost\":17.5,\"total_happiness_gain\":40}"
}
```

The processed event is stored separately from the original raw event.

A later transformation can use another version label and produce a new result from the same historical source sequence.

## Demo: ELTsim

[EvoELT is demonstrated by ELTsim](https://github.com/evelynkenenna/eltsim), a small Sims-inspired event simulation.

ELTsim creates activity schedules and character-state events representing values such as energy, happiness, and relationships. Those events provide a concrete domain for demonstrating:

* asynchronous event ingestion
* ordered historical sequences
* external transformation logic
* processed outputs
* retrospective reprocessing
* recommendation-oriented workflows

The simulation is intentionally secondary to EvoELT's architectural purpose: it provides understandable data for exercising the event pipeline.

## Technology

EvoELT currently uses:

* Kotlin
* Gradle
* PostgreSQL
* AWS SQS FIFO queues
* LocalStack for local AWS-compatible queue infrastructure
* Docker/Podman
* REST APIs

## Running Locally

### Prerequisites

You will need:

* Docker or Podman
* PostgreSQL 16
* LocalStack
* an AWS SQS FIFO-compatible queue configuration


Start LocalStack:

```bash
podman run \
  -p 4566:4566 \
  -d \
  --env DEBUG=1 \
  --env EAGER_SERVICE_LOADING=1 \
  --name localstack \
  localstack/localstack:3.6.0
```

Start PostgreSQL:

```bash
podman run \
  -p 5432:5432 \
  -d \
  --env POSTGRES_PASSWORD=password \
  --name postgres \
  postgres:16
```

Populate the `evoelt` schema in the `postgres` database using:

```text
app/db/init.sql
```

Create the queues in LocalStack:

Set AWS Region (config or session)
```bash
aws configure set region us-east-1

export AWS_REGION=us-east-1
```

```bash
aws --endpoint-url=http://localhost:4566 sqs create-queue \
  --queue-name EVOELT_CONSUMER.fifo \
  --attributes FifoQueue=true

aws --endpoint-url=http://localhost:4566 sqs create-queue \
  --queue-name EVOELT_PRODUCER.fifo \
  --attributes FifoQueue=true

aws --endpoint-url=http://localhost:4566 sqs create-queue \
  --queue-name dead-letter-queue
```

Verify the queue urls
```bash
aws --endpoint-url=http://localhost:4566 sqs list-queues
```

Configure the dead-letter queue:

```bash
aws --endpoint-url=http://localhost:4566 sqs set-queue-attributes \
  --queue-url http://sqs.us-east-1.localhost.localstack.cloud:4566/000000000000/EVOELT_CONSUMER.fifo \
  --attributes '{"RedrivePolicy":"{\"deadLetterTargetArn\":\"arn:aws:sqs:us-east-1:000000000000:dead-letter-queue\",\"maxReceiveCount\":\"1\"}"}'
```

## Docker/Podman

Build the service image:

```bash
podman build . -t org.kenenna/evoelt:latest
```

Run EvoELT:

```bash
podman run -d \
  -p 8080:8080 \
  --env QUEUE_ENDPOINT=http://localstack:4566 \
  --env DB_URL=jdbc:postgresql://postgres:5432/postgres \
  --env AWS_REGION=us-east-1 \
  --env AWS_ACCESS_KEY_ID=localstack \
  --env AWS_SECRET_ACCESS_KEY=localstack \
  evoelt
  
```

## Environment Variables

| Variable                | Description                          | Default                                     | Required |
| ----------------------- | ------------------------------------ | ------------------------------------------- | -------- |
| `DB_URL`                | PostgreSQL JDBC URL                  | `jdbc:postgresql://localhost:5432/postgres` | Yes      |
| `DB_SCHEMA`             | Database schema                      | `evoelt`                                    | Yes      |
| `DB_USERNAME`           | Database username                    | `postgres`                                  | Yes      |
| `DB_PASSWORD`           | Database password                    | `password`                                  | Yes      |
| `QUEUE_ENDPOINT`        | Message queue endpoint               | `http://localhost:4566`                     | Yes      |
| `CONSUMER_QUEUE_URL`    | Queue consumed by EvoELT             | `EVOELT_CONSUMER.fifo`                      | Yes      |
| `PRODUCER_QUEUE_URL`    | Queue receiving EvoELT notifications | `EVOELT_PRODUCER.fifo`                      | Yes      |
| `AWS_ACCESS_KEY_ID`     | AWS/local credential                 | —                                           | Yes      |
| `AWS_SECRET_ACCESS_KEY` | AWS/local credential                 | —                                           | Yes      |
| `AWS_REGION`            | AWS region                           | —                                           | Yes      |

## Design Considerations

EvoELT is a proof of concept and intentionally leaves several production concerns outside its current scope.

Sequence ordering is represented numerically. A production implementation would need an explicit retention, archival, rollover, or compaction policy for extremely long-lived sequences rather than relying on unbounded sequence growth.

Other production concerns would include authentication and authorization, schema evolution, idempotency guarantees, retry policies, observability, deployment topology, data retention, and workload-specific scaling.

Those concerns are deliberately separated from the central architectural experiment: **can source events remain replayable while derived logic evolves independently?**

## Related Project

**ELTsim:** https://github.com/evelynkenenna/eltsim

A simulation environment used to demonstrate EvoELT with generated behavioral events and recommendation-oriented transformations.

## Related Writing

EvoELT accompanies my writing on suggestion engines, replayable systems, and architectures for evolving decision workflows:

https://medium.com/@evelynkenenna/building-suggestion-engines-with-microservices-in-house-or-out-of-house-b68ee28fee3f

## License

MIT
