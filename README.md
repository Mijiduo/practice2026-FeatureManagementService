# Feature Management Service

An enterprise-grade, high-throughput, and low-latency feature flagging platform designed to manage thousands of flags across more than 100 applications—spanning web portals, backend APIs, and mobile clients.

---

## 1. Core Business Functions

The platform enforces a strict separation of concerns between the **Management/Distribution Tier (Server)** and the **Execution Tier (Client/SDKs)** to guarantee sub-millisecond evaluation times without bloating infrastructure costs.

### Server & Infrastructure (Control Plane & Relay)
* **Configuration Management:** Provides an administrative interface and backend APIs to create, update, and archive feature flags, target segments, and percentage rollouts.
* **Real-time Configuration Streaming:** The **Relay Service** maintains persistent Server-Sent Events (SSE) connections with backend services to broadcast incremental rule updates (diffs) instantly when configurations change.
* **Edge Resolution:** The **Edge API** handles evaluations for thin clients (Web & Mobile) by receiving user contexts, executing rules against a fast Redis cache, and returning sanitized, pre-evaluated flag states.

### Client & Execution (SDKs)
* **Server-Side SDKs (Java, Go, Node.js):** Embedded directly inside backend microservices. They maintain a full copy of the environment rules in local memory and evaluate flags **locally** using a deterministic hashing engine. This completely eliminates network hops during business transactions.
* **Client-Side SDKs (JS, iOS, Android):** Tailored for untrusted frontend environments. Instead of fetching raw business rules, they request a map of pre-evaluated flag values from the Edge API and cache them locally.

---

## 2. Full API Design

The system's APIs are decoupled into three specialized domains to handle management operations, real-time delivery, and high-volume data ingestion independently.

### Key API Pillars
* **Management API (REST / HTTPS):** Secure CRUD operations for flags, environmental management, and access controls.
* **Sync & Edge API (SSE / gRPC / HTTPS):** High-availability endpoints for server SDK streaming and frontend flag resolution.
* **Telemetry API (HTTPS Batch Ingestion):** High-throughput endpoints dedicated to collecting asynchronous evaluation logs from SDKs.

> 📁 For exact endpoint specifications, payloads, and response schemas, please refer to the full [API Design Specification Document](./API_DESIGN.md).

---

## 3. Caching Strategy

To support massive traffic spikes while keeping cloud resource usage minimal, the architecture utilizes a multi-tiered caching paradigm:

* **Write-Through Global Cache:** The Control Plane immediately flushes any flag mutations to a **Redis Cluster**. This ensures the Relay and Edge layers always read rules from memory rather than hitting the main database.
* **In-Memory SDK Caching:** Server-side SDKs load all rules into thread-safe local memory structures (e.g., Caffeine Cache). Flag evaluations require zero network I/O, scaling linearly with the host application's CPU/memory capabilities.
* **Incremental Delta Pushes:** Instead of refetching the entire configuration catalog, the Relay Service pushes only the modified rule payload (the diff) over SSE to the SDKs, dramatically reducing network bandwidth.
* **Client Storage Caching:** Frontend and mobile clients store evaluated flag states inside secure local storage (e.g., SQLite or LocalStorage) to ensure immediate availability upon application cold starts.

---

## 4. Observability Strategy

The system provides complete visibility across infrastructure health and application-level flag performance through three telemetry streams:

* **Metrics (Prometheus & Grafana):** Tracks system-level health including Relay active connection counts, rule synchronization latency, Edge API QPS, and evaluation counts partitioned by flag variations.
* **Audit Logging:** Every administrative action—such as targeting rule modifications or kill-switch triggers—is logged as an immutable audit trail entry detailing the operator, timestamp, and a JSON diff of the change.
* **Distributed Tracing (OpenTelemetry):** SDKs seamlessly inject evaluated flag names and their active values as attributes into the current OpenTelemetry span. Engineers can instantly see which feature flag paths were executed inside any Jaeger or Datadog distributed trace.

---

## 5. Explainability Model

To reliably troubleshoot system behavior and answer questions like *"Why did a specific user in a specific region see this variant?"*, the system relies on an **Evaluation Lineage Model**.

### Asynchronous Telemetry Pipeline
When an SDK evaluates a flag, it produces an **Impression Event** describing the exact logic path taken. This event is handled completely out-of-band via an internal SDK ring-buffer and flushed in compressed batches to a **Kafka Event Stream**, preventing any business logic degradation.

### The Point-in-Time Data Lake
A specialized consumer ingest team batches these events into a **ClickHouse Columnar Data Lake**. Each record preserves the complete state of the evaluation:

```json
{
  "timestamp": "2026-06-09T07:15:00.123Z",
  "flagKey": "new-checkout-flow",
  "evaluatedValue": true,
  "matchReason": "RULE_MATCH",
  "matchedRuleId": "rule_us_east_premium",
  "ruleConfigVersion": 1042,
  "contextSnapshot": {
    "userId": "usr_94812",
    "region": "US-East",
    "tier": "premium"
  }
}
