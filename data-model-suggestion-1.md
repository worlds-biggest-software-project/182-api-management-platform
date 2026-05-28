# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: API Management Platform · Created: 2026-05-20

## Philosophy

This model follows classical relational database design with full normalization (3NF/BCNF). Every distinct concept — APIs, versions, endpoints, consumers, subscriptions, policies, rate limits, analytics events — gets its own table with strongly typed columns and explicit foreign key relationships. Junction tables handle many-to-many relationships (e.g., policy-to-API assignments, consumer-to-subscription mappings).

This approach mirrors how platforms like Kong and Tyk structure their internal configuration databases: each entity is independently queryable, relationships are enforced at the database level, and schema migrations are explicit. It aligns well with the OpenAPI 3.2 specification's hierarchical structure (API → Path → Operation → Parameter/Response) and with OAuth 2.0's entity model (Client → Grant → Token → Scope).

The normalized model excels when data integrity is paramount, cross-entity queries are frequent (e.g., "show all APIs with expiring OAuth clients in jurisdiction X"), and regulatory environments demand clear audit boundaries. It is the most straightforward model for teams familiar with traditional RDBMS design.

**Best for:** Teams prioritizing data integrity, complex cross-entity reporting, and regulatory compliance in a single-cloud or on-premises deployment.

**Trade-offs:**
- Pro: Strong referential integrity prevents orphaned records and inconsistent state
- Pro: Standard SQL queries work without specialized knowledge (no JSONB operators, no graph traversals)
- Pro: Well-understood migration patterns with tools like Flyway or Alembic
- Pro: Excellent tooling support (ORMs, query builders, schema visualization)
- Con: High table count (~55-65 tables) increases schema complexity
- Con: Adding jurisdiction-specific or protocol-specific fields requires schema migrations
- Con: Many JOIN operations for common read paths (e.g., loading an API with all its policies, endpoints, and consumers)
- Con: Less flexible for multi-protocol variations (GraphQL vs REST vs gRPC have different metadata needs)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenAPI 3.2 | API/version/path/operation/parameter hierarchy directly maps to table structure |
| AsyncAPI 3.1 | Separate `async_api_channels` and `async_api_bindings` tables for event-driven API metadata |
| OAuth 2.0 / RFC 6749 | `oauth_clients`, `oauth_grants`, `oauth_tokens`, `oauth_scopes` tables model the full OAuth entity graph |
| RFC 9700 (OAuth Security BCP) | PKCE enforcement flag on `oauth_clients`; `allowed_grant_types` excludes Implicit/ROPC |
| OWASP API Security Top 10 | `security_policies` table with `owasp_category` column for classification |
| OpenID Connect Core 1.0 | `oidc_providers` table for external IdP configuration |
| JSON Schema Draft 2020-12 | `schema_definitions` table stores JSON Schema documents for request/response validation |
| ISO/IEC 27001:2022 | Audit tables (`audit_logs`, `access_logs`) support information security controls |
| NIST SP 800-228 | `api_inventory` view combining ownership, classification, and runtime metrics |
| MCP Specification 2025-11-25 | `mcp_servers` and `mcp_tools` tables for AI agent governance |
| GraphQL September 2025 | `graphql_schemas` and `graphql_operations` tables for GraphQL-specific metadata |
| gRPC / Protocol Buffers | `grpc_services` and `grpc_methods` tables for gRPC service definitions |
| OpenTelemetry | `telemetry_config` table for per-API observability settings |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    billing_email   VARCHAR(255),
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',  -- free, starter, enterprise
    status          VARCHAR(20) NOT NULL DEFAULT 'active', -- active, suspended, deleted
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255),
    password_hash   VARCHAR(255),        -- NULL if SSO-only
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organization_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member', -- owner, admin, member, viewer
    invited_by      UUID REFERENCES users(id),
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, user_id)
);

CREATE INDEX idx_org_members_org ON organization_members(organization_id);
CREATE INDEX idx_org_members_user ON organization_members(user_id);

CREATE TABLE teams (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, name)
);

CREATE TABLE team_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    UNIQUE (team_id, user_id)
);
```

## API Registry & Lifecycle

```sql
CREATE TABLE apis (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    team_id         UUID REFERENCES teams(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(200) NOT NULL,
    description     TEXT,
    api_type        VARCHAR(30) NOT NULL, -- rest, graphql, grpc, websocket, async, mcp
    status          VARCHAR(30) NOT NULL DEFAULT 'draft', -- draft, published, deprecated, retired
    visibility      VARCHAR(20) NOT NULL DEFAULT 'private', -- private, internal, public
    contact_email   VARCHAR(255),
    license_name    VARCHAR(100),
    license_url     VARCHAR(500),
    terms_of_service_url VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);

CREATE INDEX idx_apis_org ON apis(organization_id);
CREATE INDEX idx_apis_type ON apis(api_type);
CREATE INDEX idx_apis_status ON apis(status);

CREATE TABLE api_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_id          UUID NOT NULL REFERENCES apis(id) ON DELETE CASCADE,
    version_string  VARCHAR(50) NOT NULL,   -- e.g., "v2.1.0"
    semver_major    INT,
    semver_minor    INT,
    semver_patch    INT,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft', -- draft, active, deprecated, retired
    deprecated_at   TIMESTAMPTZ,
    retired_at      TIMESTAMPTZ,
    changelog       TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (api_id, version_string)
);

CREATE INDEX idx_api_versions_api ON api_versions(api_id);

CREATE TABLE api_specifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_version_id  UUID NOT NULL REFERENCES api_versions(id) ON DELETE CASCADE,
    spec_format     VARCHAR(30) NOT NULL,  -- openapi_3_2, asyncapi_3_1, graphql_sdl, protobuf, mcp
    spec_content    TEXT NOT NULL,          -- raw spec document (YAML/JSON/SDL/proto)
    checksum        VARCHAR(64),           -- SHA-256 of spec_content
    validated       BOOLEAN NOT NULL DEFAULT false,
    validation_errors JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE api_endpoints (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_version_id  UUID NOT NULL REFERENCES api_versions(id) ON DELETE CASCADE,
    http_method     VARCHAR(10),           -- GET, POST, PUT, DELETE, PATCH, QUERY (RFC 9110)
    path            VARCHAR(1000) NOT NULL,
    operation_id    VARCHAR(255),
    summary         VARCHAR(500),
    description     TEXT,
    is_deprecated   BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_api_endpoints_version ON api_endpoints(api_version_id);

CREATE TABLE api_tags (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_id          UUID NOT NULL REFERENCES apis(id) ON DELETE CASCADE,
    tag_name        VARCHAR(100) NOT NULL,
    tag_value       VARCHAR(500),
    UNIQUE (api_id, tag_name)
);
```

## Gateway & Routing

```sql
CREATE TABLE gateway_clusters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    region          VARCHAR(50),           -- e.g., us-east-1, eu-west-1
    deployment_type VARCHAR(30) NOT NULL,  -- cloud, self_hosted, hybrid, edge
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    gateway_url     VARCHAR(500),
    admin_url       VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE routes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_version_id  UUID NOT NULL REFERENCES api_versions(id) ON DELETE CASCADE,
    gateway_cluster_id UUID NOT NULL REFERENCES gateway_clusters(id) ON DELETE CASCADE,
    upstream_url    VARCHAR(1000) NOT NULL,
    path_prefix     VARCHAR(500),
    strip_path      BOOLEAN NOT NULL DEFAULT true,
    protocols       VARCHAR(50)[] NOT NULL DEFAULT '{https}',
    hosts           VARCHAR(255)[],
    timeout_ms      INT NOT NULL DEFAULT 30000,
    retries         INT NOT NULL DEFAULT 3,
    load_balancing  VARCHAR(30) DEFAULT 'round_robin', -- round_robin, least_conn, ip_hash
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_routes_api_version ON routes(api_version_id);
CREATE INDEX idx_routes_gateway ON routes(gateway_cluster_id);
```

## Security & Authentication

```sql
CREATE TABLE oauth_clients (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    client_id       VARCHAR(255) NOT NULL UNIQUE,
    client_secret_hash VARCHAR(255),
    client_name     VARCHAR(255) NOT NULL,
    client_type     VARCHAR(20) NOT NULL,     -- confidential, public (per RFC 6749)
    redirect_uris   VARCHAR(1000)[],
    allowed_grant_types VARCHAR(50)[] NOT NULL, -- authorization_code, client_credentials, refresh_token (per RFC 9700: no implicit, no ROPC)
    allowed_scopes  VARCHAR(100)[],
    pkce_required   BOOLEAN NOT NULL DEFAULT true, -- mandatory per RFC 9700 for public clients
    token_lifetime_seconds INT NOT NULL DEFAULT 3600,
    refresh_token_lifetime_seconds INT DEFAULT 86400,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE oidc_providers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    issuer_url      VARCHAR(500) NOT NULL,
    client_id       VARCHAR(255) NOT NULL,
    client_secret_enc VARCHAR(500),        -- encrypted at rest
    discovery_url   VARCHAR(500),          -- .well-known/openid-configuration
    jwks_uri        VARCHAR(500),
    supported_scopes VARCHAR(100)[],
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    consumer_id     UUID NOT NULL REFERENCES api_consumers(id) ON DELETE CASCADE,
    key_hash        VARCHAR(255) NOT NULL UNIQUE,
    key_prefix      VARCHAR(10) NOT NULL,    -- first 8 chars for identification
    name            VARCHAR(255),
    status          VARCHAR(20) NOT NULL DEFAULT 'active', -- active, revoked, expired
    expires_at      TIMESTAMPTZ,
    last_used_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_api_keys_consumer ON api_keys(consumer_id);
CREATE INDEX idx_api_keys_prefix ON api_keys(key_prefix);

CREATE TABLE security_policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    policy_type     VARCHAR(50) NOT NULL,   -- jwt_validation, oauth2, api_key, mtls, ip_whitelist, cors, threat_protection
    owasp_category  VARCHAR(20),            -- BOLA, BrokenAuth, BOPLA, etc. (OWASP API Top 10 2023)
    configuration   JSONB NOT NULL,
    priority        INT NOT NULL DEFAULT 100,
    is_global       BOOLEAN NOT NULL DEFAULT false,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE api_security_policy_assignments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_version_id  UUID NOT NULL REFERENCES api_versions(id) ON DELETE CASCADE,
    security_policy_id UUID NOT NULL REFERENCES security_policies(id) ON DELETE CASCADE,
    apply_to        VARCHAR(30) NOT NULL DEFAULT 'all', -- all, specific_endpoints
    UNIQUE (api_version_id, security_policy_id)
);
```

## Consumer & Subscription Management

```sql
CREATE TABLE api_consumers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    email           VARCHAR(255),
    consumer_type   VARCHAR(30) NOT NULL DEFAULT 'developer', -- developer, application, ai_agent
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE api_products (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    visibility      VARCHAR(20) NOT NULL DEFAULT 'public',
    approval_type   VARCHAR(20) NOT NULL DEFAULT 'auto', -- auto, manual
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE api_product_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_product_id  UUID NOT NULL REFERENCES api_products(id) ON DELETE CASCADE,
    api_version_id  UUID NOT NULL REFERENCES api_versions(id) ON DELETE CASCADE,
    UNIQUE (api_product_id, api_version_id)
);

CREATE TABLE subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    consumer_id     UUID NOT NULL REFERENCES api_consumers(id) ON DELETE CASCADE,
    api_product_id  UUID NOT NULL REFERENCES api_products(id) ON DELETE CASCADE,
    plan_id         UUID NOT NULL REFERENCES subscription_plans(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'pending', -- pending, active, suspended, cancelled
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    starts_at       TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_subscriptions_consumer ON subscriptions(consumer_id);
CREATE INDEX idx_subscriptions_product ON subscriptions(api_product_id);
```

## Rate Limiting & Quotas

```sql
CREATE TABLE rate_limit_policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    limit_type      VARCHAR(30) NOT NULL,  -- requests_per_second, requests_per_minute, tokens_per_minute, bandwidth_bytes
    limit_value     BIGINT NOT NULL,
    window_seconds  INT NOT NULL,
    burst_limit     INT,                   -- spike arrest
    scope           VARCHAR(30) NOT NULL,  -- global, per_consumer, per_api_key, per_ip
    action_on_limit VARCHAR(30) NOT NULL DEFAULT 'reject', -- reject, queue, throttle
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE rate_limit_assignments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rate_limit_policy_id UUID NOT NULL REFERENCES rate_limit_policies(id) ON DELETE CASCADE,
    target_type     VARCHAR(30) NOT NULL,  -- api, api_product, consumer, subscription
    target_id       UUID NOT NULL,
    priority        INT NOT NULL DEFAULT 100,
    UNIQUE (rate_limit_policy_id, target_type, target_id)
);

CREATE INDEX idx_rate_limit_assign_target ON rate_limit_assignments(target_type, target_id);
```

## Monetization & Billing

```sql
CREATE TABLE subscription_plans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    pricing_model   VARCHAR(30) NOT NULL,  -- free, flat_rate, usage_based, tiered, freemium
    base_price_cents BIGINT DEFAULT 0,
    currency        VARCHAR(3) NOT NULL DEFAULT 'USD', -- ISO 4217
    billing_period  VARCHAR(20) NOT NULL DEFAULT 'monthly', -- monthly, annual, pay_as_you_go
    usage_unit      VARCHAR(30),           -- requests, tokens, bytes, calls
    included_units  BIGINT,
    overage_price_cents BIGINT,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE usage_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id UUID NOT NULL REFERENCES subscriptions(id),
    period_start    TIMESTAMPTZ NOT NULL,
    period_end      TIMESTAMPTZ NOT NULL,
    request_count   BIGINT NOT NULL DEFAULT 0,
    token_count     BIGINT DEFAULT 0,      -- for AI/LLM APIs (token-aware metering)
    bytes_transferred BIGINT DEFAULT 0,
    error_count     BIGINT NOT NULL DEFAULT 0,
    computed_cost_cents BIGINT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_usage_records_sub ON usage_records(subscription_id);
CREATE INDEX idx_usage_records_period ON usage_records(period_start, period_end);

CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    consumer_id     UUID NOT NULL REFERENCES api_consumers(id),
    period_start    TIMESTAMPTZ NOT NULL,
    period_end      TIMESTAMPTZ NOT NULL,
    subtotal_cents  BIGINT NOT NULL,
    tax_cents       BIGINT NOT NULL DEFAULT 0,
    total_cents     BIGINT NOT NULL,
    currency        VARCHAR(3) NOT NULL DEFAULT 'USD',
    status          VARCHAR(20) NOT NULL DEFAULT 'draft', -- draft, issued, paid, overdue, void
    issued_at       TIMESTAMPTZ,
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Analytics & Observability

```sql
CREATE TABLE api_request_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_id          UUID NOT NULL,
    api_version_id  UUID,
    endpoint_id     UUID,
    consumer_id     UUID,
    gateway_cluster_id UUID,
    request_method  VARCHAR(10),
    request_path    VARCHAR(1000),
    response_status INT,
    latency_ms      INT,
    request_size_bytes BIGINT,
    response_size_bytes BIGINT,
    token_count     INT,                    -- for AI API calls
    client_ip       INET,
    user_agent      VARCHAR(500),
    correlation_id  VARCHAR(100),
    error_code      VARCHAR(50),
    error_message   TEXT,
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (requested_at);

CREATE INDEX idx_request_logs_api ON api_request_logs(api_id);
CREATE INDEX idx_request_logs_consumer ON api_request_logs(consumer_id);
CREATE INDEX idx_request_logs_time ON api_request_logs(requested_at);

CREATE TABLE anomaly_detections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_id          UUID NOT NULL REFERENCES apis(id),
    consumer_id     UUID REFERENCES api_consumers(id),
    anomaly_type    VARCHAR(50) NOT NULL,   -- credential_stuffing, data_scraping, injection, rate_spike, latency_spike
    severity        VARCHAR(20) NOT NULL,   -- low, medium, high, critical
    confidence      DECIMAL(5,4),           -- ML model confidence score 0.0000-1.0000
    description     TEXT,
    evidence        JSONB,
    status          VARCHAR(20) NOT NULL DEFAULT 'open', -- open, investigating, resolved, false_positive
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);

CREATE TABLE telemetry_config (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_id          UUID NOT NULL REFERENCES apis(id) ON DELETE CASCADE,
    tracing_enabled BOOLEAN NOT NULL DEFAULT true,
    metrics_enabled BOOLEAN NOT NULL DEFAULT true,
    log_level       VARCHAR(10) NOT NULL DEFAULT 'info', -- debug, info, warn, error
    sampling_rate   DECIMAL(5,4) NOT NULL DEFAULT 1.0,   -- 0.0-1.0
    otel_exporter   VARCHAR(30) DEFAULT 'otlp',          -- otlp, jaeger, zipkin
    export_endpoint VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (api_id)
);
```

## Developer Portal

```sql
CREATE TABLE portal_applications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    consumer_id     UUID NOT NULL REFERENCES api_consumers(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    website_url     VARCHAR(500),
    callback_urls   VARCHAR(1000)[],
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE portal_pages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    slug            VARCHAR(200) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    content_html    TEXT,
    content_markdown TEXT,
    page_type       VARCHAR(30) NOT NULL, -- guide, tutorial, changelog, faq, custom
    sort_order      INT NOT NULL DEFAULT 0,
    is_published    BOOLEAN NOT NULL DEFAULT false,
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);
```

## MCP & AI Agent Governance

```sql
CREATE TABLE mcp_servers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_id          UUID NOT NULL REFERENCES apis(id) ON DELETE CASCADE,
    server_name     VARCHAR(255) NOT NULL,
    transport_type  VARCHAR(30) NOT NULL,   -- http_sse, stdio, streamable_http
    server_url      VARCHAR(500),
    capabilities    VARCHAR(50)[] NOT NULL DEFAULT '{}', -- tools, resources, prompts
    auth_method     VARCHAR(30),            -- oauth2, api_key, none
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE mcp_tools (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    mcp_server_id   UUID NOT NULL REFERENCES mcp_servers(id) ON DELETE CASCADE,
    tool_name       VARCHAR(255) NOT NULL,
    description     TEXT,
    input_schema    JSONB NOT NULL,         -- JSON Schema for tool parameters
    output_schema   JSONB,                  -- structured output schema (MCP 2025-11-25)
    is_destructive  BOOLEAN NOT NULL DEFAULT false,
    requires_confirmation BOOLEAN NOT NULL DEFAULT false,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (mcp_server_id, tool_name)
);

CREATE TABLE mcp_agent_registrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    consumer_id     UUID NOT NULL REFERENCES api_consumers(id) ON DELETE CASCADE,
    agent_name      VARCHAR(255) NOT NULL,
    agent_type      VARCHAR(50),            -- llm_agent, workflow_agent, autonomous
    allowed_tools   UUID[],                 -- references mcp_tools.id
    max_tokens_per_minute INT,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Governance & Compliance

```sql
CREATE TABLE governance_rulesets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    rule_type       VARCHAR(50) NOT NULL,   -- naming_convention, security_requirement, documentation, versioning
    rule_definition JSONB NOT NULL,         -- spectral-compatible ruleset definition
    severity        VARCHAR(20) NOT NULL DEFAULT 'warning', -- info, warning, error
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE governance_violations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ruleset_id      UUID NOT NULL REFERENCES governance_rulesets(id),
    api_version_id  UUID NOT NULL REFERENCES api_versions(id),
    violation_path  VARCHAR(1000),          -- JSONPath to the violation in the spec
    message         TEXT NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    resolved        BOOLEAN NOT NULL DEFAULT false,
    resolved_at     TIMESTAMPTZ,
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gov_violations_api ON governance_violations(api_version_id);

CREATE TABLE audit_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    actor_id        UUID,                   -- user or system
    actor_type      VARCHAR(30) NOT NULL,   -- user, system, api_key, agent
    action          VARCHAR(100) NOT NULL,  -- api.created, policy.updated, subscription.approved, etc.
    resource_type   VARCHAR(50) NOT NULL,   -- api, policy, consumer, subscription, etc.
    resource_id     UUID,
    changes         JSONB,                  -- before/after diff
    ip_address      INET,
    user_agent      VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_logs_org ON audit_logs(organization_id);
CREATE INDEX idx_audit_logs_resource ON audit_logs(resource_type, resource_id);
CREATE INDEX idx_audit_logs_time ON audit_logs(created_at);
```

## Shadow API Discovery

```sql
CREATE TABLE discovered_endpoints (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    gateway_cluster_id UUID NOT NULL REFERENCES gateway_clusters(id),
    observed_method VARCHAR(10) NOT NULL,
    observed_path   VARCHAR(1000) NOT NULL,
    first_seen_at   TIMESTAMPTZ NOT NULL,
    last_seen_at    TIMESTAMPTZ NOT NULL,
    request_count   BIGINT NOT NULL DEFAULT 1,
    matched_api_id  UUID REFERENCES apis(id),      -- NULL if shadow/undocumented
    is_shadow       BOOLEAN NOT NULL DEFAULT true,
    confidence      DECIMAL(5,4),                   -- ML classification confidence
    generated_spec  JSONB,                          -- draft OpenAPI spec fragment
    status          VARCHAR(30) NOT NULL DEFAULT 'unreviewed', -- unreviewed, confirmed, ignored, documented
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_discovered_shadow ON discovered_endpoints(is_shadow) WHERE is_shadow = true;
CREATE INDEX idx_discovered_gateway ON discovered_endpoints(gateway_cluster_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 5 | organizations, users, org_members, teams, team_members |
| API Registry & Lifecycle | 5 | apis, api_versions, api_specifications, api_endpoints, api_tags |
| Gateway & Routing | 2 | gateway_clusters, routes |
| Security & Authentication | 5 | oauth_clients, oidc_providers, api_keys, security_policies, policy_assignments |
| Consumer & Subscriptions | 4 | api_consumers, api_products, api_product_versions, subscriptions |
| Rate Limiting | 2 | rate_limit_policies, rate_limit_assignments |
| Monetization & Billing | 4 | subscription_plans, usage_records, invoices (+ line_items if needed) |
| Analytics & Observability | 3 | api_request_logs, anomaly_detections, telemetry_config |
| Developer Portal | 2 | portal_applications, portal_pages |
| MCP & AI Agent | 3 | mcp_servers, mcp_tools, mcp_agent_registrations |
| Governance & Compliance | 3 | governance_rulesets, governance_violations, audit_logs |
| Shadow API Discovery | 1 | discovered_endpoints |
| **Total** | **39** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — enables distributed ID generation across gateway clusters without coordination, and avoids sequential ID enumeration attacks on consumer-facing APIs.

2. **Explicit API type column rather than inheritance** — `apis.api_type` (rest/graphql/grpc/websocket/async/mcp) with type-specific child tables (mcp_servers, graphql_schemas) keeps the base registry unified while allowing protocol-specific metadata.

3. **Separate api_versions and api_specifications tables** — decouples the version lifecycle (draft→active→deprecated→retired) from the specification document itself, allowing multiple spec formats per version and spec re-uploads without version changes.

4. **Time-partitioned request logs and audit logs** — `PARTITION BY RANGE (requested_at)` enables efficient time-range queries and partition-level data retention policies (e.g., drop partitions older than 90 days).

5. **Rate limit assignments via polymorphic target** — `target_type` + `target_id` pattern allows rate limits to be attached to APIs, products, consumers, or subscriptions without separate junction tables for each.

6. **OAuth model aligned with RFC 9700** — PKCE required by default, Implicit and ROPC grant types excluded from allowed values, reflecting 2025 security best practices.

7. **Token-aware metering** — `usage_records.token_count` and `api_request_logs.token_count` columns support AI/LLM API billing alongside traditional request-count billing.

8. **MCP as a first-class entity** — dedicated tables for MCP servers, tools, and agent registrations reflect the convergence of API management with AI agent governance per MCP Specification 2025-11-25.

9. **Shadow API discovery as a separate domain** — `discovered_endpoints` lives outside the main API registry, allowing ML-driven discovery to populate candidates that are reviewed and promoted into the formal registry.

10. **JSONB used sparingly** — only for genuinely variable structures (policy configuration, evidence payloads, tool schemas) rather than as a catch-all, maintaining queryability of core fields via standard SQL.
