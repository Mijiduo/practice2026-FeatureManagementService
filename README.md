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
#### Typical SDK Usage (Pseudocode)

To ensure a seamless developer experience, all SDKs follow a consistent 3-step initialization and evaluation pattern. 

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
```
* **Client-Side SDKs (Web, iOS, Android):** Designed for untrusted frontend environments. Instead of downloading raw rules, they utilize **Edge Resolution**—fetching pre-computed flag values from the Edge API and serving them from a local device cache. **Result: High security and low bandwidth.**

#### Typical SDK Usage (Pseudocode)

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
```
---

## 2. APIs

The system's APIs are decoupled into three specialized domains to handle management operations, real-time delivery, and high-volume data ingestion independently.

### Key API Pillars
* **Management API (REST / HTTPS):** Secure CRUD operations for flags, environmental management, and access controls.
* **Sync & Edge API (SSE / gRPC / HTTPS):** High-availability endpoints for server SDK streaming and frontend flag resolution.
* **Telemetry API (HTTPS Batch Ingestion):** High-throughput endpoints dedicated to collecting asynchronous evaluation logs from SDKs.

### Specification
**Version:** 1.0.0
**Base URLs:**
* Management API: `https://api.feature-admin.internal/v1`
* Distribution (Relay/Edge): `https://edge.feature-relay.internal/v1`
* Telemetry: `https://telemetry.feature-data.internal/v1`

#### 1. Authentication & Authorization

All APIs require an API Key passed in the `Authorization` header. Keys are scoped by environment and role:
* **Admin Key (`api-key-...`)**: Full CRUD access. Used by the Management API.
* **Server SDK Key (`srv-key-...`)**: Read-only access to raw rules and stream endpoints.
* **Client Edge Key (`cli-key-...`)**: Read-only access to pre-evaluated Edge endpoints only.

`Authorization: Bearer <API_KEY>`


#### 2. Management API (Control Plane)
Used by the Admin UI and CI/CD pipelines to manage configurations.

##### 2.1 Create a Feature Flag
`POST /api/flags`

**Request Body:**
```json
{
  "key": "new-checkout-flow",
  "name": "New Checkout Experience",
  "description": "Enables the redesigned checkout flow for Q3.",
  "returnType": "boolean",
  "defaultValue": false
}
```
##### 2.2 Update Flag Rules
`PUT /api/flags/{flagKey}/rules`
Updates the targeting segments, conditional rules, and rollout percentages.

**Request Body:**
```json
{
  "rules": [
    {
      "id": "rule_beta_testers",
      "type": "TARGET_MATCH",
      "users": ["user-123", "user-456"],
      "value": true
    },
    {
      "id": "rule_us_premium",
      "type": "CONDITION_MATCH",
      "conditions": [
        { "attribute": "region", "operator": "IN", "values": ["US-East", "US-West"] },
        { "attribute": "tier", "operator": "EQUALS", "values": ["premium"] }
      ],
      "value": true
    },
    {
      "id": "rule_global_rollout",
      "type": "ROLLOUT_MATCH",
      "percentage": 25,
      "value": true
    }
  ],
  "fallbackValue": false
}
```
#### 3. Distribution API (Edge & Relay)
Used by SDKs to fetch or evaluate rules.

##### 3.1 Initial Full Sync (Server SDKs)
`GET /client/flags`
Fetches the complete rule payload for the environment. Cached heavily in Redis.
```json
{
  "environment": "production",
  "version": 1042,
  "flags": {
    "new-checkout-flow": {
      "rules": [ /* Array of rule objects */ ],
      "fallback": false
    }
  }
}
```
##### 3.2 Real-time SSE Stream (Server SDKs)
`GET /client/stream`
Establishes a persistent Server-Sent Events (SSE) connection.
Stream Payload (Event: PATCH):
```json
{
  "event": "patch"
  "data": {
    "version": 1043,
    "flagKey": "new-checkout-flow",
    "action": "UPDATE_ROLLOUT",
    "changes": { "percentage": 50 }
  }
}
```
##### 3.3 Edge Resolution (Client SDKs)
`POST /client/evaluate`
Evaluates flags on the Edge server to hide raw business rules from public frontend clients.

Request Body:
```json
{
  "context": {
    "userId": "user-94812",
    "region": "US-East",
    "tier": "premium",
    "appVersion": "2.1.0"
  }
}
```
Response:
```json
{
  "evaluatedFlags": {
    "new-checkout-flow": true,
    "summer-sale-banner": false,
    "max-items-in-cart": 15
  }
}
```


## 3. Caching Strategy

To support massive traffic spikes while maintaining sub-millisecond evaluation latency, our architecture employs a 3-tier caching hierarchy. This design ensures extreme performance and high availability, even if downstream data stores become temporarily unreachable.

---

### 1. The 3-Tier Cache Hierarchy

#### Tier 1: Global Write-Through Cache (Redis Cluster)
* **Location:** Deployed between the Control Plane (Management) and the Distribution Layer (Relay/Edge).
* **Role:** Acts as the primary source of truth for the Distribution Layer, shielding the PostgreSQL database from high-volume read requests.
* **Update Policy:** Management API follows a **Write-Through** pattern: any flag mutation is persisted to the database and simultaneously updated in Redis.

#### Tier 2: In-Memory Server Cache (SDK Level)
* **Location:** Embedded within backend microservices (via Server SDKs).
* **Role:** Enables **zero-network I/O** flag evaluation by keeping the full rule set in the local process memory.
* **Update Policy:** Uses **Incremental Delta Pushes**. SDKs maintain a persistent SSE (Server-Sent Events) connection to the Relay Service to receive small JSON diffs, allowing for real-time memory updates without full re-fetches.

#### Tier 3: Client-Side Device Cache (Client SDK Level)
* **Location:** Secure local storage on user devices (SQLite, LocalStorage, etc.).
* **Role:** Eliminates "UI flicker" and enables offline flag availability.
* **Update Policy:** Uses **Stale-While-Revalidate**. The SDK instantly renders the UI using the last known cache, then asynchronously fetches the latest state from the Edge API to update the local store in the background.


### 2. Data Flow Architecture

#### A. The Mutation & Sync Flow (Configuration Updates)
When a flag configuration is modified:
1. **Control Plane:** Management API updates the database and **Redis (Tier 1)**.
2. **Event Broadcast:** Redis Pub/Sub notifies the Relay Service of the configuration change.
3. **Delta Calculation:** Relay Service compares the new state with the previous version to generate a minimal JSON patch.
4. **SSE Push:** Relay broadcasts the patch to all connected Server SDKs.
5. **Local Update:** Server SDKs apply the change to their **In-Memory Cache (Tier 2)** instantly.

#### B. The Evaluation Flow (Execution)

##### Server-Side Evaluation (Backend)
1. Application calls `getBoolVariation(key, context)`.
2. SDK hits the **Tier 2 Local Memory Cache** directly.
3. Deterministic hashing (MurmurHash3) is performed locally.
4. **Result:** < 1ms latency, 0 network hops.

##### Client-Side Evaluation (Frontend/Mobile)
1. Application calls `getBoolean(key)`.
2. SDK reads the **Tier 3 Device Cache** and returns the value immediately.
3. SDK triggers an asynchronous request to the **Edge API**.
4. Edge API executes the rule against the **Tier 1 Redis Cache**.
5. SDK updates **Tier 3 Device Cache** with the fresh result for subsequent usage.

### 3. Fault Tolerance & Resiliency

Our architecture treats caching not just as a performance optimization, but as a critical availability layer:

* **Database Outage:** If PostgreSQL fails, the system remains fully operational as all reads are served from **Redis (Tier 1)** and **In-Memory Cache (Tier 2)**.
* **Relay/Cache Outage:** If the distribution layer fails, Server SDKs continue to evaluate using the last known state preserved in their **Local Memory (Tier 2)**, effectively "freezing" the configuration until connectivity is restored.
* **Network Instability:** Client SDKs rely on **Local Storage (Tier 3)**, ensuring that mobile users or users with poor network connections do not experience broken UI or failed application logic.

---

## 4. Observability Strategy

The system provides complete visibility across infrastructure health and application-level flag performance through three telemetry streams:

* **Metrics (Prometheus & Grafana):** Tracks system-level health including Relay active connection counts, rule synchronization latency, Edge API QPS, and evaluation counts partitioned by flag variations.
* **Audit Logging:** Every administrative action—such as targeting rule modifications or kill-switch triggers—is logged as an immutable audit trail entry detailing the operator, timestamp, and a JSON diff of the change.
* **Distributed Tracing (OpenTelemetry):** SDKs seamlessly inject evaluated flag names and their active values as attributes into the current OpenTelemetry span. Engineers can instantly see which feature flag paths were executed inside any Jaeger or Datadog distributed trace.

---

## 5. Explainability Model

To ensure complete auditability and "decision-transparency" in a high-scale environment, the Explainability Model captures the provenance of every flag decision. This system allows developers to reconstruct the exact logic state at any point in history, answering the question: *"Why did a specific user receive this specific variant?"*


### 1. Layered Architecture

The explainability system follows a "Collect-Transport-Analyze" architecture designed to ensure it never bottlenecks primary business logic.

* **Generation Layer (SDKs):** The site of evaluation. Every decision triggers an `Impression Event`. This layer handles data sanitization (stripping PII) and local buffering.
* **Transport Layer (Kafka):** A high-throughput, decoupled stream acting as the buffer between high-velocity SDK events and analytical storage.
* **Ingestion Layer (Processors):** Specialized microservices that perform batch aggregation, enrichment, and schema validation.
* **Storage & Analytics Layer (ClickHouse):** A columnar data lake optimized for time-series analytics, allowing sub-second querying of billions of evaluation events.

---

### 2. Core Operation Mechanism

The core of this model is the **Evaluation Snapshot**. Every decision record is immutable and complete.



* **Context Capture:** Every decision stores a snapshot of the `context` (userId, region, tier, appVersion) at the exact moment of evaluation.
* **Lineage Linkage:** The event carries the `ruleConfigVersion`—a global counter that uniquely identifies the configuration state of the system at that millisecond.
* **Reasoning Code:** Every decision is tagged with a `reason` (e.g., `TARGET_MATCH`, `ROLLOUT_MATCH`), allowing developers to distinguish between forced overrides and percentage-based rollouts.

---

### 3. Full Lifecycle Flow by Scenario

#### Scenario A: Backend Logic (Server-Side)
1. **Evaluation:** The Server SDK evaluates a flag locally.
2. **Buffering:** The event is pushed to an in-memory Ring Buffer within the application process (prevents I/O blocking).
3. **Flushing:** A background thread flushes the buffer to the **Kafka Stream** in compressed batches.
4. **Enrichment & Storage:** The Ingestion Service attaches infrastructure metadata (pod ID, region) and commits the event to **ClickHouse**.

#### Scenario B: Public Edge Evaluation (Client-Side)
1. **Request:** The Client SDK sends an `identify` request to the **Edge API**.
2. **Edge Logic:** The Edge API performs the evaluation and generates the `Impression Event` server-side.
3. **Direct Ingestion:** Because the Edge API is an infrastructure component, it streams events directly to the telemetry pipeline, bypassing the SDK flush cycle.

#### Scenario C: Troubleshooting & Audit ("The Playback")
When a discrepancy is reported, developers perform a "Playback":
1. **Lookup:** Query ClickHouse by `userId` and `timestamp`.
2. **Reconstruction:** Fetch the `ruleConfigVersion` from the event record.
3. **Playback:** Feed this `ruleConfigVersion` back into the **Shared Evaluation Engine**.
4. **Verification:** The system mimics the exact historical environment, allowing developers to "replay" the logic to confirm the decision.


### 4. Reason Code Significance

| Reason Code | Interpretation | Audit Significance |
| :--- | :--- | :--- |
| `TARGET_MATCH` | Explicit Whitelist/Blacklist hit | **High:** Indicates an administrative override. |
| `RULE_MATCH` | Condition-based rule (e.g., Tier/Region) | **Medium:** Represents standard business logic. |
| `ROLLOUT_MATCH` | Hash-based percentage bucket hit | **Low:** Expected statistical distribution. |
| `FALLBACK` | System error or no rule matched | **Critical:** Indicates potential configuration gaps. |

By separating the **Evaluation Path** from the **Business Logic Path**, the system provides 100% decision transparency without introducing technical debt or performance degradation.
