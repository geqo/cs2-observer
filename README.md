# Geqo Observer: A Real-Time Game Telemetry & Analytics Platform

## I. Project Overview

**Geqo Observer** is a high-performance, distributed platform designed to collect, process, and analyze vast streams of telemetry data from Counter-Strike game servers. The system captures every in-game event in real-time—from a player's footstep to a grenade detonation—and transforms this chaotic data stream into structured information ready for advanced analytics and visualization.

This project serves as a case study in building a comprehensive **data pipeline**, engineered with a focus on **scalability, resilience, and low-latency processing**.

---

## II. System Architecture

Gqo Observer is built on an **Event-Driven Architecture** centered around the **Apache Kafka** message broker. This approach decouples system components, enabling them to operate, scale, and fail independently.

### Architectural Diagram

~~~mermaid
flowchart TD
    subgraph Game Server
        A[CS2 Server + GeqoObserver Plugin]
    end

    subgraph Ingestion Layer
        B[Ingestion Service]
    end

    subgraph Processing Pipeline
        C[Kafka Broker]
        D[Worker Service]
        E[PostgreSQL + TimescaleDB]
    end
    
    subgraph Real-time Layer
        F[Broadcaster Service]
        G[Web Dashboard / API Clients]
    end

    A -- WebSocket (Compressed MessagePack) --> B
    B -- Produce --> C(raw-game-events)
    C -- Consume --> D
    D -- Persist --> E
    D -- Produce --> C(processed-game-events)
    C -- Consume --> F
    F -- WebSocket --> G
    D -- Produce on Error --> C(raw-game-events-dlq)
~~~

### Core Architectural Principles

* **Asynchronicity & Decoupling (Kafka):** Instead of direct service-to-service communication, Kafka acts as a durable, asynchronous backbone. The `Ingestion Service` is completely unaware of how data will be processed; it simply and rapidly produces messages to a raw topic. This design allows the `Worker` service to be restarted, scaled, or taken offline for maintenance without interrupting data collection.

* **Resilience & Fault Tolerance (Dead Letter Queue):** The `Worker` implements the **DLQ pattern**. If a message cannot be processed due to corruption or an unexpected error, it is not lost or endlessly retried. Instead, it is rerouted to a dedicated `raw-game-events-dlq` topic for later analysis, ensuring the main data pipeline remains unblocked.

* **Efficiency & Performance (WebSocket + MessagePack):** WebSockets are used for low-latency, persistent communication with game servers. Instead of text-based JSON, **MessagePack** is employed for its superior performance. It provides fast binary serialization and **effective LZ4 compression**, significantly reducing payload size and network overhead—a critical factor for high-frequency event streams.

* **Scalability (Stateless Worker + Docker):** The core processing component, the `Worker Service`, is designed to be **stateless**. The state of each match is managed in-memory with fallbacks to the database. This architecture allows for seamless horizontal scaling; multiple instances of the `Worker` can be deployed to process Kafka partitions in parallel. The entire platform is containerized with Docker and orchestrated via Docker Compose, ensuring consistent deployments and simplifying operations.

---

## III. Data Flow

1.  **Collection (`GqoObserver.cs`):** A C# plugin for the game server hooks into native game events. It utilizes object pooling to minimize GC overhead, then **serializes and compresses** event data into the MessagePack format before streaming it over a WebSocket connection.

2.  **Ingestion (`Ingestion.cs`):** A lightweight service authenticates game servers and accepts the binary data stream. Its sole responsibility is to act as a high-throughput producer, immediately publishing the raw data into the `raw-game-events` Kafka topic. No business logic is performed at this stage to maximize ingestion speed.

3.  **Processing & Enrichment (`Worker.cs`):** This service is the brain of the platform. It consumes raw events from Kafka and performs several critical tasks:
    * **Match State Machine:** Implements a sophisticated state machine to accurately track a match's lifecycle (`Warmup`, `Live`, `RoundEndPhase`, `GameOver`, etc.). This solves complex scenarios like mid-game restarts, custom knife rounds, and pauses.
    * **Data Enrichment:** Adds valuable context to events. For example, it flags kills that occur after a round's official end with `is_post_round: true`.
    * **Persistence:** Persists the enriched, structured data into a **PostgreSQL** database supercharged with the **TimescaleDB** extension for efficient time-series data management.

4.  **Broadcasting (`Broadcaster`):** After processing, the `Worker` produces a clean, validated event stream to the `processed-game-events` topic. The `Broadcaster` service consumes from this topic and pushes real-time updates via WebSockets to connected clients, such as a live analytics dashboard.

---

## IV. Schema Overview

The database is designed to separate structured metadata from the high-volume event stream:

* **`matches`, `rounds`, `players`:** Standard relational tables that provide context and structure for the game.
* **`game_events`:** The core table, configured as a **TimescaleDB hypertable**. It is partitioned and indexed by `timestamp` to handle millions of event records efficiently. This table stores the enriched data, including contextual flags like `is_post_round` and `match_tag`.

---

## V. Local Deployment

The entire platform is containerized, enabling a one-command setup for the complete infrastructure.

1.  Ensure Docker and Docker Compose are installed.
2.  Clone the repository: `git clone <URL>`
3.  Navigate to the project's root directory.
4.  Execute: `docker-compose up -d --build`

The system will launch, and the `Ingestion Service` will be available to accept connections on port `13880`.

---

## VI. Roadmap & Future Work

* **Analytics Aggregation Service:** A dedicated post-match service to pre-calculate and store aggregate statistics (e.g., K/D/A, ADR), enabling lightning-fast API responses for the frontend.
* **Web Dashboard & Live 2D Replay:** A user-facing web application built with React/Vue to visualize match data and render live, tactical 2D replays using data from the `Broadcaster`.
* **CI/CD & Cloud Deployment:** Implementing CI/CD pipelines for automated testing and deployment to a cloud environment like Kubernetes (AWS EKS, GKE, or AKS).
* **Monitoring & Observability:** Integrating a monitoring stack like Prometheus and Grafana to track system health, Kafka lag, message throughput, and error rates.
