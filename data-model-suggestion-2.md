# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: API Management Platform · Created: 2026-05-20

## Philosophy

This model treats every state change in the API management platform as an immutable event stored in an append-only event store. The current state of any entity (an API, a subscription, a rate limit policy) is derived by replaying events rather than reading a mutable row. Read-optimized materialized views (projections) serve the queries that the developer portal, gateway, and analytics dashboards need, following the CQRS (Command Query Responsibility Segregation) pattern.

Event sourcing is used by financial trading platforms (event logs of every order), healthcare systems (immutable medical records), and audit-heavy regulatory systems. In the API management domain, this approach is particularly powerful because API lifecycle management is inherently temporal: APIs move through draft, published, deprecated, and retired states; subscriptions are approved, suspended, and renewed; rate limits are adjusted over time. An event-sourced model can answer questions like "what policies were active on API X at 3:00 AM when the incident occurred?" without complex audit log reconstruction.

The architecture separates the write side (command handlers that validate and emit events) from the read side (projections that maintain denormalized views). This separation allows the analytics pipeline to consume the same event stream for real-time dashboards, anomaly detection, and ML-driven insights without impacting the operational database.

**Best for:** Organizations requiring complete audit trails, temporal querying ("what was true on date X?"), regulatory compliance (NIST SP 800-228, ISO 27001), and AI-powered analytics on change patterns.

**Trade-offs:**
- Pro: Complete, immutable audit trail — every change is recorded with full context
- Pro: Temporal queries are trivial — replay events up to any point in time
- Pro: Event stream feeds analytics, ML, and anomaly detection directly
- Pro: Natural fit for distributed systems — events can be replicated across regions
- Pro: Schema evolution via new event types without migrating existing data
- Con: Higher storage requirements — events accumulate indefinitely (snapshots help)
- Con: Eventual consistency between write and read sides requires careful UX design
- Con: More complex to implement than direct CRUD — developers need event sourcing experience
- Con: Debugging requires understanding event replay, not just inspecting current state
- Con: Read model rebuilds can be slow for large event stores without snapshots

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenAPI 3.2 | API specification content stored as `ApiSpecificationUploaded` events; current spec derived from latest event |
| AsyncAPI 3.1 | Async API definitions stored as events; protocol bindings captured in event payloads |
| OAuth 2.0 / RFC 6749 | OAuth client lifecycle tracked as events: `OAuthClientCreated`, `OAuthClientSecretRotated`, `OAuthClientRevoked` |
| RFC 9700 | Security policy changes tracked as events enabling temporal security audit ("when was PKCE enforcement enabled?") |
| OWASP API Security Top 10 | Security violations recorded as domain events feeding the anomaly detection projection |
| NIST SP 800-228 | Full lifecycle audit trail satisfies NIST requirement for API inventory with ownership and runtime metrics |
| ISO/IEC 27001:2022 | Immutable event store inherently satisfies audit logging and change tracking controls |
| MCP Specification 2025-11-25 | MCP server and tool registrations tracked as events; agent interactions logged for governance |
| OpenTelemetry | Events exported to OTel-compatible collectors for distributed tracing correlation |
| OCSF | Security events structured following Open Cybersecurity Schema Framework patterns |

---

## Event Store (Write Side)

```sql
-- The single source of truth: an append-only event store
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,           -- aggregate root ID (e.g., API ID, subscription ID)
    stream_type     VARCHAR(50) NOT NULL,     -- Api, Subscription, Consumer, Policy, Gateway, McpServer
    event_type      VARCHAR(100) NOT NULL,    -- e.g., ApiCreated, ApiVersionPublished, SubscriptionApproved
    event_version   INT NOT NULL,             -- schema version of this event type
    sequence_number BIGINT NOT NULL,          -- per-stream ordering
    payload         JSONB NOT NULL,           -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}', -- actor, correlation_id, ip, user_agent
    organization_id UUID NOT NULL,            -- partition key for multi-tenancy
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, sequence_number)
) PARTITION BY RANGE (occurred_at);

CREATE INDEX idx_events_stream ON events(stream_id, sequence_number);
CREATE INDEX idx_events_type ON events(event_type);
CREATE INDEX idx_events_org ON events(organization_id);
CREATE INDEX idx_events_occurred ON events(occurred_at);

-- Snapshots to avoid replaying entire event history
CREATE TABLE snapshots (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,
    snapshot_version BIGINT NOT NULL,        -- event sequence_number at snapshot time
    state           JSONB NOT NULL,          -- serialized aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);
```

### Event Type Catalogue

```sql
-- Example event payloads (documented as reference, not stored here)

-- ApiCreated
-- {
--   "name": "Payment Processing API",
--   "slug": "payment-processing",
--   "api_type": "rest",
--   "visibility": "internal",
--   "team_id": "uuid",
--   "description": "Handles payment transactions"
-- }

-- ApiVersionPublished
-- {
--   "version_string": "v2.1.0",
--   "semver": {"major": 2, "minor": 1, "patch": 0},
--   "spec_format": "openapi_3_2",
--   "spec_checksum": "sha256:abc123...",
--   "changelog": "Added /refunds endpoint"
-- }

-- SubscriptionApproved
-- {
--   "consumer_id": "uuid",
--   "api_product_id": "uuid",
--   "plan_id": "uuid",
--   "approved_by": "uuid",
--   "starts_at": "2026-05-20T00:00:00Z",
--   "rate_limit": {"requests_per_second": 100}
-- }

-- RateLimitAdjusted (ML-driven recommendation applied)
-- {
--   "previous_limit": 100,
--   "new_limit": 250,
--   "adjustment_reason": "ml_recommendation",
--   "confidence": 0.92,
--   "traffic_pattern": "gradual_increase"
-- }

-- SecurityPolicyViolationDetected
-- {
--   "owasp_category": "BOLA",
--   "severity": "high",
--   "endpoint_path": "/users/{id}/orders",
--   "evidence": {"pattern": "sequential_id_enumeration", "sample_size": 150}
-- }

-- McpToolRegistered
-- {
--   "mcp_server_id": "uuid",
--   "tool_name": "search_documents",
--   "input_schema": {...},
--   "is_destructive": false
-- }

-- ShadowApiDiscovered
-- {
--   "observed_path": "/internal/metrics",
--   "observed_method": "GET",
--   "request_count": 4521,
--   "confidence": 0.87,
--   "generated_spec_fragment": {...}
-- }
```

## Read Models (Projections)

Projections are materialized views rebuilt from events. Each projection is optimized for a specific query pattern.

### API Registry Projection

```sql
-- Materialized by consuming Api* events
CREATE TABLE rm_apis (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    team_id         UUID,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(200) NOT NULL,
    api_type        VARCHAR(30) NOT NULL,
    status          VARCHAR(30) NOT NULL,
    visibility      VARCHAR(20) NOT NULL,
    current_version VARCHAR(50),
    endpoint_count  INT NOT NULL DEFAULT 0,
    subscriber_count INT NOT NULL DEFAULT 0,
    last_event_seq  BIGINT NOT NULL,        -- last processed event sequence
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_apis_org ON rm_apis(organization_id);
CREATE INDEX idx_rm_apis_status ON rm_apis(status);

-- Materialized by consuming ApiVersion* events
CREATE TABLE rm_api_versions (
    id              UUID PRIMARY KEY,
    api_id          UUID NOT NULL REFERENCES rm_apis(id),
    version_string  VARCHAR(50) NOT NULL,
    status          VARCHAR(30) NOT NULL,
    spec_format     VARCHAR(30),
    spec_content    TEXT,
    endpoint_count  INT NOT NULL DEFAULT 0,
    published_at    TIMESTAMPTZ,
    deprecated_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
```

### Gateway Configuration Projection

```sql
-- Hot read model consumed by gateway nodes for routing decisions
-- Optimized for low-latency lookups by host + path
CREATE TABLE rm_gateway_routes (
    id              UUID PRIMARY KEY,
    gateway_cluster_id UUID NOT NULL,
    api_id          UUID NOT NULL,
    api_version_id  UUID NOT NULL,
    host            VARCHAR(255),
    path_prefix     VARCHAR(500) NOT NULL,
    upstream_url    VARCHAR(1000) NOT NULL,
    protocols       VARCHAR(50)[] NOT NULL,
    timeout_ms      INT NOT NULL,
    strip_path      BOOLEAN NOT NULL,
    is_active       BOOLEAN NOT NULL,
    last_event_seq  BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_routes_lookup ON rm_gateway_routes(gateway_cluster_id, host, path_prefix) WHERE is_active = true;

-- Security policies flattened for gateway enforcement
CREATE TABLE rm_active_policies (
    id              UUID PRIMARY KEY,
    api_version_id  UUID NOT NULL,
    policy_type     VARCHAR(50) NOT NULL,
    configuration   JSONB NOT NULL,
    priority        INT NOT NULL,
    last_event_seq  BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_policies_api ON rm_active_policies(api_version_id);

-- Rate limits flattened for gateway enforcement
CREATE TABLE rm_active_rate_limits (
    id              UUID PRIMARY KEY,
    target_type     VARCHAR(30) NOT NULL,
    target_id       UUID NOT NULL,
    limit_type      VARCHAR(30) NOT NULL,
    limit_value     BIGINT NOT NULL,
    window_seconds  INT NOT NULL,
    burst_limit     INT,
    last_event_seq  BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_rate_limits_target ON rm_active_rate_limits(target_type, target_id);
```

### Consumer & Subscription Projection

```sql
CREATE TABLE rm_consumers (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    email           VARCHAR(255),
    consumer_type   VARCHAR(30) NOT NULL,
    status          VARCHAR(20) NOT NULL,
    active_subscription_count INT NOT NULL DEFAULT 0,
    total_api_calls BIGINT NOT NULL DEFAULT 0,
    last_event_seq  BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE TABLE rm_subscriptions (
    id              UUID PRIMARY KEY,
    consumer_id     UUID NOT NULL,
    api_product_id  UUID NOT NULL,
    plan_name       VARCHAR(255),
    pricing_model   VARCHAR(30),
    status          VARCHAR(30) NOT NULL,
    starts_at       TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    last_event_seq  BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_subs_consumer ON rm_subscriptions(consumer_id);
```

### Analytics Projection

```sql
-- Hourly aggregation built from ApiRequestLogged events
CREATE TABLE rm_api_metrics_hourly (
    api_id          UUID NOT NULL,
    api_version_id  UUID,
    hour            TIMESTAMPTZ NOT NULL,
    request_count   BIGINT NOT NULL DEFAULT 0,
    error_count     BIGINT NOT NULL DEFAULT 0,
    avg_latency_ms  DECIMAL(10,2),
    p50_latency_ms  INT,
    p95_latency_ms  INT,
    p99_latency_ms  INT,
    total_tokens    BIGINT DEFAULT 0,
    unique_consumers INT DEFAULT 0,
    bytes_in        BIGINT DEFAULT 0,
    bytes_out       BIGINT DEFAULT 0,
    PRIMARY KEY (api_id, hour)
);

-- Consumer-level daily aggregation for billing
CREATE TABLE rm_consumer_usage_daily (
    consumer_id     UUID NOT NULL,
    subscription_id UUID NOT NULL,
    day             DATE NOT NULL,
    request_count   BIGINT NOT NULL DEFAULT 0,
    token_count     BIGINT DEFAULT 0,
    bytes_transferred BIGINT DEFAULT 0,
    error_count     BIGINT NOT NULL DEFAULT 0,
    computed_cost_cents BIGINT,
    PRIMARY KEY (consumer_id, subscription_id, day)
);

-- Anomaly detection results
CREATE TABLE rm_anomalies (
    id              UUID PRIMARY KEY,
    api_id          UUID NOT NULL,
    consumer_id     UUID,
    anomaly_type    VARCHAR(50) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    confidence      DECIMAL(5,4),
    description     TEXT,
    evidence        JSONB,
    status          VARCHAR(20) NOT NULL,
    detected_at     TIMESTAMPTZ NOT NULL,
    resolved_at     TIMESTAMPTZ
);

CREATE INDEX idx_rm_anomalies_api ON rm_anomalies(api_id);
CREATE INDEX idx_rm_anomalies_status ON rm_anomalies(status) WHERE status = 'open';
```

### Governance Projection

```sql
-- Current governance compliance state per API
CREATE TABLE rm_governance_status (
    api_version_id  UUID NOT NULL,
    ruleset_id      UUID NOT NULL,
    ruleset_name    VARCHAR(255) NOT NULL,
    total_violations INT NOT NULL DEFAULT 0,
    error_count     INT NOT NULL DEFAULT 0,
    warning_count   INT NOT NULL DEFAULT 0,
    last_checked_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (api_version_id, ruleset_id)
);

-- Shadow API inventory
CREATE TABLE rm_shadow_apis (
    id              UUID PRIMARY KEY,
    gateway_cluster_id UUID NOT NULL,
    observed_method VARCHAR(10) NOT NULL,
    observed_path   VARCHAR(1000) NOT NULL,
    first_seen_at   TIMESTAMPTZ NOT NULL,
    last_seen_at    TIMESTAMPTZ NOT NULL,
    request_count   BIGINT NOT NULL,
    is_documented   BOOLEAN NOT NULL DEFAULT false,
    matched_api_id  UUID,
    confidence      DECIMAL(5,4),
    status          VARCHAR(30) NOT NULL
);
```

## Subscription & Outbox Pattern

```sql
-- Outbox table for reliable event publishing to external consumers (Kafka, webhooks)
CREATE TABLE event_outbox (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL REFERENCES events(event_id),
    destination     VARCHAR(100) NOT NULL,   -- kafka_topic, webhook_url, analytics_pipeline
    status          VARCHAR(20) NOT NULL DEFAULT 'pending', -- pending, published, failed
    retry_count     INT NOT NULL DEFAULT 0,
    last_error      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at    TIMESTAMPTZ
);

CREATE INDEX idx_outbox_pending ON event_outbox(status) WHERE status = 'pending';

-- Projection checkpoints for idempotent rebuilds
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_sequence   BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Example Queries

### Temporal query: "What rate limits were active on API X at 3:00 AM?"

```sql
-- Replay RateLimitCreated/RateLimitAdjusted/RateLimitRemoved events
-- for the API stream up to the target timestamp
SELECT payload
FROM events
WHERE stream_type = 'RateLimit'
  AND payload->>'api_id' = 'target-api-uuid'
  AND occurred_at <= '2026-05-20 03:00:00+00'
ORDER BY sequence_number ASC;
```

### Audit query: "Who changed the security policy on the Payment API last month?"

```sql
SELECT
    event_type,
    payload,
    metadata->>'actor_id' AS actor,
    metadata->>'ip_address' AS ip,
    occurred_at
FROM events
WHERE stream_id = 'payment-api-uuid'
  AND event_type LIKE 'SecurityPolicy%'
  AND occurred_at >= '2026-04-20'
  AND occurred_at < '2026-05-20'
ORDER BY occurred_at ASC;
```

### Analytics: Hourly error rate trend

```sql
SELECT
    hour,
    request_count,
    error_count,
    ROUND(100.0 * error_count / NULLIF(request_count, 0), 2) AS error_rate_pct,
    avg_latency_ms
FROM rm_api_metrics_hourly
WHERE api_id = 'target-api-uuid'
  AND hour >= now() - INTERVAL '7 days'
ORDER BY hour;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store (Write Side) | 2 | events, snapshots |
| API Registry Projection | 2 | rm_apis, rm_api_versions |
| Gateway Configuration Projection | 3 | rm_gateway_routes, rm_active_policies, rm_active_rate_limits |
| Consumer & Subscription Projection | 2 | rm_consumers, rm_subscriptions |
| Analytics Projection | 3 | rm_api_metrics_hourly, rm_consumer_usage_daily, rm_anomalies |
| Governance Projection | 2 | rm_governance_status, rm_shadow_apis |
| Infrastructure | 2 | event_outbox, projection_checkpoints |
| **Total** | **16** | Plus the event store as the single source of truth |

---

## Key Design Decisions

1. **Single event store as source of truth** — all domain state derives from the `events` table. This eliminates the need for separate audit logging; the audit trail IS the data model. Satisfies NIST SP 800-228 and ISO 27001 audit requirements inherently.

2. **Stream-per-aggregate pattern** — each API, subscription, consumer, and policy is an aggregate root with its own event stream (identified by `stream_id`). This enables independent lifecycle management and targeted snapshots.

3. **JSONB payloads with versioned schemas** — `event_version` allows the same `event_type` to evolve its payload structure over time. Event upcasters transform old versions to current versions during replay, avoiding destructive schema migrations.

4. **Separate projections per query pattern** — the gateway needs low-latency route lookups (`rm_gateway_routes`); the portal needs consumer-facing API lists (`rm_apis`); analytics needs time-series aggregations (`rm_api_metrics_hourly`). Each is independently rebuildable from events.

5. **Outbox pattern for reliable event distribution** — rather than direct Kafka/webhook publishing from command handlers (which risks message loss), events are written to `event_outbox` in the same transaction as `events`, then published asynchronously.

6. **Time-partitioned event store** — `PARTITION BY RANGE (occurred_at)` enables efficient retention policies and time-range queries. Old partitions can be archived to cold storage without affecting operational queries.

7. **Snapshot-based optimization** — for aggregates with long event histories (e.g., an API with thousands of policy changes), snapshots store the computed state at a point in time. Replay only needs events after the latest snapshot.

8. **Projection checkpoints for idempotent rebuilds** — `projection_checkpoints` tracks the last processed event per projection, enabling exactly-once processing and full projection rebuilds without data duplication.

9. **Event-driven ML pipeline** — the same event stream that powers projections feeds the anomaly detection and ML recommendation engines. `RateLimitAdjusted` events with `adjustment_reason: "ml_recommendation"` close the feedback loop.

10. **Fewer tables overall (16 vs 39)** — the event-sourced model has dramatically fewer tables because the event store replaces dozens of mutable entity tables. Complexity shifts from schema design to event handler logic.
