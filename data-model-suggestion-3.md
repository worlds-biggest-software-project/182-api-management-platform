# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: API Management Platform · Created: 2026-05-20

## Philosophy

This model uses traditional relational tables for core entities (APIs, consumers, subscriptions) but leans heavily on PostgreSQL JSONB columns for protocol-specific metadata, policy configurations, jurisdiction-specific fields, and extensible properties. The key insight is that an API management platform must handle wildly different API types (REST, GraphQL, gRPC, WebSocket, Kafka, MQTT, MCP) and each type has unique metadata that does not share a common column structure.

Rather than creating separate tables for REST-specific fields, GraphQL-specific fields, gRPC-specific fields, and so on (which leads to table sprawl and complex JOINs), this model stores the core identity and lifecycle in relational columns and puts the variable metadata in indexed JSONB columns. This mirrors how platforms like Gravitee.io handle multi-protocol support and how Kong stores plugin configurations as JSON documents.

The hybrid approach is ideal for rapid MVP development because new protocol support or new policy types can be added without schema migrations — just define a new JSON structure. It also suits multi-region deployments where different jurisdictions may require different metadata fields. GIN indexes on JSONB columns provide performant querying when needed.

**Best for:** Teams building a multi-protocol API management platform quickly, where protocol diversity (REST/GraphQL/gRPC/async) and rapid feature iteration matter more than rigid schema enforcement.

**Trade-offs:**
- Pro: Dramatically fewer tables (~25 vs ~40 in normalized model)
- Pro: New protocol types or policy types added without schema migrations
- Pro: JSONB containment queries (`@>`) are performant with GIN indexes
- Pro: Flexible per-organization or per-jurisdiction custom fields
- Pro: Natural fit for storing OpenAPI/AsyncAPI/GraphQL specs as structured data alongside metadata
- Con: JSONB fields lack database-level type enforcement — validation must happen in application code
- Con: Harder to write complex cross-field queries spanning JSONB and relational columns
- Con: JSONB columns can become "junk drawers" without disciplined documentation
- Con: Foreign keys cannot reference fields inside JSONB — referential integrity is partial
- Con: Schema evolution inside JSONB requires application-level migration logic

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenAPI 3.2 | Full spec stored in `api_versions.spec_document` JSONB; parsed fields extracted for indexing |
| AsyncAPI 3.1 | Async bindings (Kafka, MQTT, Solace) stored in `api_versions.protocol_config` JSONB |
| GraphQL September 2025 | GraphQL SDL and schema coordinates stored in `api_versions.protocol_config` |
| gRPC / Protocol Buffers | Service definitions and method metadata in `api_versions.protocol_config` |
| OAuth 2.0 / RFC 6749 | OAuth client configuration in `auth_providers.config` JSONB with type-specific fields |
| RFC 9700 | Security BCP enforcement flags within `auth_providers.config` structure |
| MCP Specification 2025-11-25 | MCP server capabilities and tool schemas in `api_versions.protocol_config` |
| OWASP API Security Top 10 | Security policy rules as JSONB documents in `policies.rule_definition` |
| NIST SP 800-228 | API inventory view combining relational fields with JSONB metadata |
| JSON Schema Draft 2020-12 | Used to validate JSONB structures at the application layer |
| OpenTelemetry | Per-API observability settings in `apis.observability_config` JSONB |
| ISO 4217 | Currency codes in monetization JSONB structures |

---

## Core Entities

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "billing_email": "billing@acme.com",
    --   "default_gateway_region": "us-east-1",
    --   "custom_domains": ["api.acme.com"],
    --   "branding": {"logo_url": "...", "primary_color": "#0066CC"},
    --   "jurisdiction_config": {"data_residency": "EU", "gdpr_dpa_signed": true}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255),
    password_hash   VARCHAR(255),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    profile         JSONB NOT NULL DEFAULT '{}',
    -- profile example:
    -- {
    --   "avatar_url": "...",
    --   "timezone": "America/New_York",
    --   "notification_preferences": {"email": true, "slack": false}
    -- }
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organization_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- permissions example:
    -- ["api:read", "api:write", "policy:read", "consumer:manage", "billing:view"]
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, user_id)
);

CREATE INDEX idx_org_memberships_org ON organization_memberships(organization_id);
CREATE INDEX idx_org_memberships_user ON organization_memberships(user_id);
```

## APIs & Versions (Multi-Protocol via JSONB)

```sql
CREATE TABLE apis (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(200) NOT NULL,
    api_type        VARCHAR(30) NOT NULL,    -- rest, graphql, grpc, websocket, kafka, mqtt, mcp
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    visibility      VARCHAR(20) NOT NULL DEFAULT 'private',
    owner_team      VARCHAR(255),
    tags            VARCHAR(100)[] DEFAULT '{}',
    labels          JSONB NOT NULL DEFAULT '{}',
    -- labels example:
    -- {"department": "payments", "cost_center": "CC-4500", "compliance": ["pci-dss", "sox"]}
    observability_config JSONB NOT NULL DEFAULT '{}',
    -- observability_config example:
    -- {
    --   "tracing_enabled": true,
    --   "sampling_rate": 0.1,
    --   "log_level": "info",
    --   "otel_exporter": "otlp",
    --   "export_endpoint": "https://otel.internal:4317"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);

CREATE INDEX idx_apis_org ON apis(organization_id);
CREATE INDEX idx_apis_type ON apis(api_type);
CREATE INDEX idx_apis_status ON apis(status);
CREATE INDEX idx_apis_tags ON apis USING GIN (tags);
CREATE INDEX idx_apis_labels ON apis USING GIN (labels);

CREATE TABLE api_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_id          UUID NOT NULL REFERENCES apis(id) ON DELETE CASCADE,
    version_string  VARCHAR(50) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    
    -- The raw specification document (OpenAPI YAML/JSON, GraphQL SDL, protobuf, etc.)
    spec_document   TEXT,
    spec_format     VARCHAR(30),             -- openapi_3_2, asyncapi_3_1, graphql_sdl, protobuf, mcp_manifest
    spec_checksum   VARCHAR(64),
    
    -- Protocol-specific configuration in JSONB — this is the key flexibility point
    protocol_config JSONB NOT NULL DEFAULT '{}',
    -- REST example:
    -- {
    --   "base_path": "/v2/payments",
    --   "servers": [{"url": "https://api.acme.com", "description": "Production"}],
    --   "endpoints": [
    --     {"method": "POST", "path": "/charges", "operation_id": "createCharge", "summary": "Create a charge"},
    --     {"method": "GET", "path": "/charges/{id}", "operation_id": "getCharge", "summary": "Get a charge"}
    --   ],
    --   "content_types": ["application/json"],
    --   "pagination_style": "cursor"
    -- }
    --
    -- GraphQL example:
    -- {
    --   "schema_sdl": "type Query { ... }",
    --   "operations": ["query", "mutation", "subscription"],
    --   "federation_version": "v2",
    --   "subgraph_name": "payments",
    --   "schema_coordinates": ["Query.charge", "Mutation.createCharge"]
    -- }
    --
    -- gRPC example:
    -- {
    --   "proto_package": "acme.payments.v2",
    --   "services": [
    --     {"name": "PaymentService", "methods": [
    --       {"name": "CreateCharge", "input": "CreateChargeRequest", "output": "Charge", "streaming": "none"},
    --       {"name": "StreamCharges", "input": "StreamRequest", "output": "Charge", "streaming": "server"}
    --     ]}
    --   ]
    -- }
    --
    -- Kafka/AsyncAPI example:
    -- {
    --   "channels": {
    --     "payment.events": {
    --       "protocol": "kafka",
    --       "binding": {"groupId": "payment-consumers", "partitions": 12},
    --       "publish": {"message_schema": {...}},
    --       "subscribe": {"message_schema": {...}}
    --     }
    --   }
    -- }
    --
    -- MCP example:
    -- {
    --   "server_name": "payment-tools",
    --   "transport": "streamable_http",
    --   "capabilities": ["tools", "resources"],
    --   "tools": [
    --     {"name": "create_charge", "description": "...", "input_schema": {...}, "is_destructive": true}
    --   ],
    --   "resources": [
    --     {"uri": "payment://charges/{id}", "name": "Charge details"}
    --   ]
    -- }
    
    changelog       TEXT,
    deprecated_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (api_id, version_string)
);

CREATE INDEX idx_api_versions_api ON api_versions(api_id);
CREATE INDEX idx_api_versions_protocol ON api_versions USING GIN (protocol_config);
```

## Gateway & Routing

```sql
CREATE TABLE gateways (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    deployment_type VARCHAR(30) NOT NULL,     -- cloud, self_hosted, edge
    region          VARCHAR(50),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "gateway_url": "https://gw.acme.com",
    --   "admin_url": "https://gw-admin.acme.com:8444",
    --   "tls": {"cert_path": "/certs/gw.pem", "min_version": "1.2"},
    --   "proxy_buffer_size": "16k",
    --   "access_log_format": "json",
    --   "health_check": {"interval_seconds": 10, "timeout_seconds": 5}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE routes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_version_id  UUID NOT NULL REFERENCES api_versions(id) ON DELETE CASCADE,
    gateway_id      UUID NOT NULL REFERENCES gateways(id) ON DELETE CASCADE,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    routing_config  JSONB NOT NULL,
    -- routing_config example:
    -- {
    --   "upstream_url": "https://payments-svc.internal:8080",
    --   "path_prefix": "/v2/payments",
    --   "strip_path": true,
    --   "hosts": ["api.acme.com"],
    --   "protocols": ["https"],
    --   "timeout_ms": 30000,
    --   "retries": 3,
    --   "load_balancing": "round_robin",
    --   "circuit_breaker": {"threshold": 5, "timeout_seconds": 30},
    --   "request_transforms": [
    --     {"type": "add_header", "name": "X-Request-ID", "value": "$request_id"}
    --   ],
    --   "response_transforms": [
    --     {"type": "remove_header", "name": "X-Internal-Debug"}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_routes_api_version ON routes(api_version_id);
CREATE INDEX idx_routes_gateway ON routes(gateway_id);
```

## Security & Policies (Unified Policy Engine)

```sql
CREATE TABLE auth_providers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    provider_type   VARCHAR(30) NOT NULL,    -- oauth2, oidc, saml, mtls, api_key
    name            VARCHAR(255) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    config          JSONB NOT NULL,
    -- oauth2 example:
    -- {
    --   "issuer_url": "https://auth.acme.com",
    --   "jwks_uri": "https://auth.acme.com/.well-known/jwks.json",
    --   "allowed_grant_types": ["authorization_code", "client_credentials"],
    --   "pkce_required": true,
    --   "token_lifetime_seconds": 3600,
    --   "scopes": ["read", "write", "admin"],
    --   "audience": "https://api.acme.com"
    -- }
    --
    -- oidc example:
    -- {
    --   "discovery_url": "https://login.microsoftonline.com/tenant/.well-known/openid-configuration",
    --   "client_id": "app-uuid",
    --   "client_secret_ref": "vault:secret/oidc-client-secret",
    --   "supported_scopes": ["openid", "profile", "email"]
    -- }
    --
    -- mtls example:
    -- {
    --   "ca_cert_path": "/certs/ca.pem",
    --   "required_cn_pattern": "*.acme.com",
    --   "crl_url": "https://pki.acme.com/crl"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Unified policy table: rate limiting, security, transformation, caching all in one
CREATE TABLE policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    policy_category VARCHAR(30) NOT NULL,    -- security, rate_limit, transformation, caching, cors, logging
    policy_type     VARCHAR(50) NOT NULL,    -- jwt_validation, oauth2_enforcement, ip_whitelist, request_rate, token_rate, spike_arrest, request_transform, response_cache
    priority        INT NOT NULL DEFAULT 100,
    is_global       BOOLEAN NOT NULL DEFAULT false,
    owasp_category  VARCHAR(20),
    rule_definition JSONB NOT NULL,
    -- rate_limit example:
    -- {
    --   "limit_type": "tokens_per_minute",
    --   "limit_value": 10000,
    --   "window_seconds": 60,
    --   "burst_limit": 500,
    --   "scope": "per_consumer",
    --   "action_on_limit": "reject",
    --   "response_headers": {"X-RateLimit-Limit": true, "X-RateLimit-Remaining": true}
    -- }
    --
    -- security example (IP whitelist):
    -- {
    --   "allowed_cidrs": ["10.0.0.0/8", "172.16.0.0/12"],
    --   "deny_action": "reject_403",
    --   "log_denied": true
    -- }
    --
    -- caching example:
    -- {
    --   "ttl_seconds": 300,
    --   "cache_key": ["path", "query_params", "accept_header"],
    --   "methods": ["GET", "HEAD"],
    --   "bypass_header": "X-Cache-Bypass"
    -- }
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_policies_org ON policies(organization_id);
CREATE INDEX idx_policies_category ON policies(policy_category);
CREATE INDEX idx_policies_rule ON policies USING GIN (rule_definition);

-- Flexible policy assignment: attach to APIs, versions, endpoints, consumers, products
CREATE TABLE policy_bindings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id       UUID NOT NULL REFERENCES policies(id) ON DELETE CASCADE,
    target_type     VARCHAR(30) NOT NULL,    -- api, api_version, consumer, api_product, gateway
    target_id       UUID NOT NULL,
    override_config JSONB,                   -- per-target overrides to the base policy
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (policy_id, target_type, target_id)
);

CREATE INDEX idx_policy_bindings_target ON policy_bindings(target_type, target_id);
```

## Consumers, Subscriptions & Monetization

```sql
CREATE TABLE consumers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    email           VARCHAR(255),
    consumer_type   VARCHAR(30) NOT NULL DEFAULT 'developer',  -- developer, application, ai_agent
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- ai_agent example:
    -- {
    --   "agent_framework": "langchain",
    --   "model": "claude-opus-4-20250514",
    --   "allowed_mcp_tools": ["search_docs", "create_ticket"],
    --   "max_tokens_per_minute": 50000,
    --   "auto_approval": false
    -- }
    credentials     JSONB NOT NULL DEFAULT '{}',
    -- credentials example:
    -- {
    --   "api_keys": [
    --     {"key_prefix": "sk_live_", "key_hash": "sha256:...", "name": "Production", "status": "active", "created_at": "..."}
    --   ],
    --   "oauth_client_id": "client-uuid",
    --   "mtls_fingerprint": "sha256:..."
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_consumers_org ON consumers(organization_id);
CREATE INDEX idx_consumers_type ON consumers(consumer_type);
CREATE INDEX idx_consumers_credentials ON consumers USING GIN (credentials);

CREATE TABLE api_products (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    visibility      VARCHAR(20) NOT NULL DEFAULT 'public',
    approval_type   VARCHAR(20) NOT NULL DEFAULT 'auto',
    included_api_versions UUID[] NOT NULL DEFAULT '{}',  -- references api_versions.id
    plans           JSONB NOT NULL DEFAULT '[]',
    -- plans example:
    -- [
    --   {
    --     "plan_id": "uuid",
    --     "name": "Free Tier",
    --     "pricing_model": "free",
    --     "limits": {"requests_per_month": 1000, "tokens_per_month": 10000},
    --     "price": null
    --   },
    --   {
    --     "plan_id": "uuid",
    --     "name": "Pro",
    --     "pricing_model": "usage_based",
    --     "base_price": {"amount_cents": 9900, "currency": "USD", "period": "monthly"},
    --     "limits": {"requests_per_month": 100000, "tokens_per_month": 1000000},
    --     "overage": {"per_1000_requests_cents": 50, "per_1000_tokens_cents": 10}
    --   }
    -- ]
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    consumer_id     UUID NOT NULL REFERENCES consumers(id) ON DELETE CASCADE,
    api_product_id  UUID NOT NULL REFERENCES api_products(id),
    plan_id         UUID NOT NULL,           -- references plans[].plan_id in api_products.plans JSONB
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    billing_info    JSONB NOT NULL DEFAULT '{}',
    -- billing_info example:
    -- {
    --   "current_period_start": "2026-05-01T00:00:00Z",
    --   "current_period_end": "2026-06-01T00:00:00Z",
    --   "usage": {"requests": 45230, "tokens": 125000, "bytes": 89000000},
    --   "computed_cost_cents": 4523,
    --   "payment_method": "stripe:pm_xxx"
    -- }
    approved_by     UUID REFERENCES users(id),
    starts_at       TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_subscriptions_consumer ON subscriptions(consumer_id);
CREATE INDEX idx_subscriptions_product ON subscriptions(api_product_id);
```

## Analytics & Request Logs

```sql
CREATE TABLE request_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_id          UUID NOT NULL,
    api_version_id  UUID,
    consumer_id     UUID,
    gateway_id      UUID,
    request_method  VARCHAR(10),
    request_path    VARCHAR(1000),
    response_status INT,
    latency_ms      INT,
    token_count     INT,
    client_ip       INET,
    correlation_id  VARCHAR(100),
    request_meta    JSONB NOT NULL DEFAULT '{}',
    -- request_meta example:
    -- {
    --   "user_agent": "python-requests/2.31",
    --   "content_type": "application/json",
    --   "request_size_bytes": 1024,
    --   "response_size_bytes": 4096,
    --   "cache_hit": false,
    --   "auth_method": "jwt",
    --   "error_code": "RATE_LIMITED",
    --   "error_message": "Quota exceeded",
    --   "upstream_latency_ms": 145,
    --   "gateway_latency_ms": 12
    -- }
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (requested_at);

CREATE INDEX idx_request_logs_api ON request_logs(api_id);
CREATE INDEX idx_request_logs_consumer ON request_logs(consumer_id);
CREATE INDEX idx_request_logs_time ON request_logs(requested_at);

CREATE TABLE anomaly_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_id          UUID NOT NULL REFERENCES apis(id),
    consumer_id     UUID REFERENCES consumers(id),
    anomaly_type    VARCHAR(50) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    confidence      DECIMAL(5,4),
    details         JSONB NOT NULL,
    -- details example:
    -- {
    --   "pattern": "sequential_id_enumeration",
    --   "owasp_category": "BOLA",
    --   "sample_requests": 150,
    --   "time_window_minutes": 5,
    --   "source_ips": ["10.0.1.5", "10.0.1.6"],
    --   "ml_model_version": "anomaly-v3.2",
    --   "recommended_action": "block_consumer"
    -- }
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);
```

## Governance & Audit

```sql
CREATE TABLE governance_rulesets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    rules           JSONB NOT NULL,
    -- rules example (spectral-compatible):
    -- {
    --   "rules": {
    --     "operation-operationId": {"severity": "error", "message": "All operations must have operationId"},
    --     "path-keys-no-trailing-slash": {"severity": "warning"},
    --     "info-contact": {"severity": "info"},
    --     "custom-auth-required": {
    --       "severity": "error",
    --       "message": "All endpoints must declare security scheme",
    --       "given": "$.paths.*.*",
    --       "then": {"field": "security", "function": "truthy"}
    --     }
    --   }
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_trail (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    actor_id        UUID,
    actor_type      VARCHAR(30) NOT NULL,
    action          VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID,
    context         JSONB NOT NULL DEFAULT '{}',
    -- context example:
    -- {
    --   "changes": {"status": {"from": "draft", "to": "published"}},
    --   "ip_address": "203.0.113.5",
    --   "user_agent": "Mozilla/5.0...",
    --   "reason": "Approved by security team after review"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_org ON audit_trail(organization_id);
CREATE INDEX idx_audit_resource ON audit_trail(resource_type, resource_id);
CREATE INDEX idx_audit_time ON audit_trail(created_at);
```

## Shadow API Discovery

```sql
CREATE TABLE discovered_endpoints (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    gateway_id      UUID NOT NULL REFERENCES gateways(id),
    observed_method VARCHAR(10) NOT NULL,
    observed_path   VARCHAR(1000) NOT NULL,
    first_seen_at   TIMESTAMPTZ NOT NULL,
    last_seen_at    TIMESTAMPTZ NOT NULL,
    request_count   BIGINT NOT NULL DEFAULT 1,
    matched_api_id  UUID REFERENCES apis(id),
    is_shadow       BOOLEAN NOT NULL DEFAULT true,
    analysis        JSONB NOT NULL DEFAULT '{}',
    -- analysis example:
    -- {
    --   "confidence": 0.87,
    --   "cluster_id": "cluster-42",
    --   "generated_openapi_fragment": {
    --     "path": "/internal/metrics",
    --     "get": {"summary": "Internal metrics endpoint", "responses": {"200": {"description": "Prometheus metrics"}}}
    --   },
    --   "request_patterns": {"avg_size_bytes": 0, "response_codes": {"200": 4500, "500": 21}},
    --   "ml_model_version": "shadow-v2.1"
    -- }
    status          VARCHAR(30) NOT NULL DEFAULT 'unreviewed',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_discovered_shadow ON discovered_endpoints(is_shadow) WHERE is_shadow = true;
```

## Example Queries

### Find all APIs with Kafka bindings

```sql
SELECT a.name, av.version_string, av.protocol_config->'channels' AS kafka_channels
FROM apis a
JOIN api_versions av ON av.api_id = a.id
WHERE a.api_type = 'kafka'
  AND av.status = 'active'
  AND av.protocol_config ? 'channels';
```

### Find consumers exceeding token quota

```sql
SELECT c.name, s.billing_info->'usage'->>'tokens' AS tokens_used,
       ap.plans->0->'limits'->>'tokens_per_month' AS token_limit
FROM subscriptions s
JOIN consumers c ON c.id = s.consumer_id
JOIN api_products ap ON ap.id = s.api_product_id
WHERE s.status = 'active'
  AND (s.billing_info->'usage'->>'tokens')::bigint > 
      (ap.plans->0->'limits'->>'tokens_per_month')::bigint;
```

### Find all MCP tools across the platform

```sql
SELECT a.name AS api_name, 
       av.version_string,
       tool->>'name' AS tool_name,
       tool->>'description' AS tool_description,
       tool->>'is_destructive' AS destructive
FROM apis a
JOIN api_versions av ON av.api_id = a.id,
     jsonb_array_elements(av.protocol_config->'tools') AS tool
WHERE a.api_type = 'mcp'
  AND av.status = 'active';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Identity | 3 | organizations, users, organization_memberships |
| API Registry & Versions | 2 | apis, api_versions (protocol_config JSONB handles all protocol types) |
| Gateway & Routing | 2 | gateways, routes |
| Security | 2 | auth_providers, policies + policy_bindings |
| Policy Bindings | 1 | policy_bindings (unified attachment mechanism) |
| Consumers & Subscriptions | 3 | consumers, api_products, subscriptions |
| Analytics | 2 | request_logs, anomaly_events |
| Governance & Audit | 2 | governance_rulesets, audit_trail |
| Shadow API Discovery | 1 | discovered_endpoints |
| **Total** | **18** | Protocol-specific tables eliminated via JSONB |

---

## Key Design Decisions

1. **Protocol-specific metadata in `protocol_config` JSONB** — instead of separate tables for REST endpoints, GraphQL schemas, gRPC services, Kafka channels, and MCP tools, a single JSONB column stores whatever structure the protocol requires. Adding support for a new protocol (e.g., NATS, RabbitMQ) requires zero schema migrations.

2. **Unified policy table with JSONB rule definitions** — rate limiting, security, transformation, caching, and CORS policies all live in one table. `policy_category` and `policy_type` classify them; `rule_definition` JSONB contains the type-specific configuration. This mirrors Kong's plugin architecture.

3. **Consumer credentials in JSONB** — API keys, OAuth client IDs, and mTLS fingerprints coexist in a single `credentials` JSONB column. GIN index enables lookups. This avoids three separate credential tables and handles consumers who authenticate via multiple methods.

4. **Subscription plans embedded in api_products** — pricing plans are JSONB arrays within `api_products` rather than a separate table. This keeps the product + pricing read path to a single row fetch. The trade-off is that plan changes require JSONB updates rather than row inserts.

5. **GIN indexes on all JSONB columns used in queries** — `apis.labels`, `apis.tags`, `consumers.credentials`, and `policies.rule_definition` all have GIN indexes for containment (`@>`) and existence (`?`) queries.

6. **Relational columns for high-cardinality query dimensions** — `api_type`, `status`, `visibility`, `consumer_type` remain as VARCHAR columns with B-tree indexes rather than JSONB fields, because these are the most common filter/group-by dimensions.

7. **Request metadata in JSONB** — `request_logs.request_meta` stores variable request details (user agent, cache hit, auth method, error details) while keeping the most-queried fields (api_id, consumer_id, response_status, latency_ms) as indexed columns.

8. **Only 18 tables total** — compared to ~39 in the normalized model. The reduction comes from collapsing protocol-specific tables, credential tables, and policy-type-specific tables into JSONB columns. The trade-off is weaker database-level enforcement in exchange for faster iteration.

9. **JSON Schema validation at application layer** — since JSONB columns do not enforce structure, the application layer validates all JSONB writes against JSON Schema definitions. This is documented but not enforced by the database.

10. **Audit trail with JSONB context** — the `audit_trail.context` column captures the before/after diff, IP address, user agent, and any justification text in a single JSONB document, avoiding a rigid columnar audit structure.
