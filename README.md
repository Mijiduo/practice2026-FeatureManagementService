# Feature Management Service

An enterprise-grade, high-throughput, and low-latency feature flagging platform designed to manage thousands of flags across more than 100 applications—spanning web portals, backend APIs, and mobile clients.

---

## 1. Core Architecture & Components

The system is built on a decoupled architecture comprising the Server Infrastructure, Client SDKs, and a foundational Shared Engine that guarantees identical behavior across all environments.

### A. Shared Component: Universal Evaluation Engine
The mathematical core embedded inside all evaluation layers (Backend SDKs and Edge Servers). 
* **Deterministic Rollouts:** Utilizes `MurmurHash3(flagKey + userKey)` to guarantee that a specific user receives the exact same flag variant, regardless of where the code executes.
* **Unified Rule Parser:** Processes targeting segments and multi-attribute conditions uniformly, completely eliminating "flag drift" (inconsistencies) across different programming languages.

### B. Server Infrastructure (Management & Distribution)
The centralized backend responsible for configuration, storage, and real-time delivery.
* **Management API (Control Plane):** The source of truth. Provides REST APIs and a UI for flag CRUD operations, targeting configurations, audit logging, and RBAC.
* **Relay Service:** Maintains persistent Server-Sent Events (SSE) connections to broadcast incremental rule diffs to internal microservices instantly.
* **Edge API:** An internet-facing gateway that performs on-the-fly evaluations for untrusted clients, returning sanitized state payloads to protect sensitive business logic.

### C. Client SDKs (Execution Plane)
The lightweight integration libraries running inside business applications.
* **Server-Side SDKs (Java, Go, Node.js):** Designed for trusted backend environments. They synchronize the full rule set into local memory and evaluate flags locally via the Shared Engine. **Result: Zero network latency.**
### Typical SDK Usage (Pseudocode)

To ensure a seamless developer experience, all SDKs follow a consistent 3-step initialization and evaluation pattern. 

#### Server-Side SDK (Backend / Microservices)
Designed for synchronous, zero-latency local evaluation.

```java
// Step 1: Initialize the SDK as a singleton on application startup
FeatureClient client = new FeatureClient("SERVER_SDK_KEY");

// Step 2: Build the context for the current request/user
Context userContext = new Context.Builder("user-94812")
    .setAttribute("region", "US-East")
    .setAttribute("tier", "premium")
    .build();

// Step 3: Evaluate the flag locally (Zero network I/O)
// Signature: getBoolVariation(flagKey, context, fallbackValue)
boolean enableNewCheckout = client.getBoolVariation("new-checkout-flow", userContext, false);

if (enableNewCheckout) {
    // Route to the new highly-optimized database checkout logic
    checkoutService.processV2(order);
} else {
    // Route to the legacy monolithic checkout logic
    checkoutService.processLegacy(order);
}

* **Client-Side SDKs (Web, iOS, Android):** Designed for untrusted frontend environments. Instead of downloading raw rules, they utilize **Edge Resolution**—fetching pre-computed flag values from the Edge API and serving them from a local device cache. **Result: High security and low bandwidth.**

### Typical SDK Usage (Pseudocode)

```JavaScript
// Step 1: Initialize the SDK with an environment key
const client = new FeatureClient("CLIENT_PUBLIC_KEY");

// Step 2: Identify the user asynchronously (e.g., after login)
// This calls the Edge API to fetch pre-evaluated flag states
await client.identify({
    userId: "user-94812",
    platform: "web",
    appVersion: "2.1.0"
});

// Step 3: Evaluate synchronously against the local device cache
const showDiscountBanner = client.getBoolean("summer-sale-banner", false);

if (showDiscountBanner) {
    render(<DiscountBanner />);
}

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
