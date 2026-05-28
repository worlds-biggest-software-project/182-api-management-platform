# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: API Management Platform · Created: 2026-05-20

## Philosophy

This model combines relational tables for operational CRUD (APIs, consumers, subscriptions, billing) with a property graph layer for relationship-heavy queries. The graph layer models the complex web of connections in an API management platform: APIs depend on other APIs, consumers subscribe to products that bundle APIs, policies inherit through organizational hierarchies, gateway clusters route traffic between services, and MCP agents compose tools from multiple servers.

In practice, API management is a graph problem masquerading as a CRUD problem. Consider the queries that matter most to platform operators: "What is the blast radius if this API goes down?" (dependency graph), "Which consumers are affected by this breaking change?" (subscription graph + dependency graph), "Does this AI agent have transitive access to PII through tool composition?" (MCP tool graph + data classification), "Which policies apply to this request after evaluating all inheritance rules?" (policy inheritance graph). These queries are expensive or impossible with relational JOINs but trivial with graph traversal.

The model implements the graph layer using PostgreSQL `ltree` for hierarchical paths, adjacency tables with recursive CTEs for dependency graphs, and optionally a dedicated graph database (Neo4j, Apache AGE) for complex traversal queries. Relational tables handle the transactional CRUD that the gateway, portal, and billing systems need.

**Best for:** Large-scale API platforms managing hundreds or thousands of APIs with complex interdependencies, policy inheritance hierarchies, multi-team governance, and AI agent tool composition analysis.

**Trade-offs:**
- Pro: Dependency and impact analysis queries are fast and natural
- Pro: Policy inheritance through organizational hierarchies is elegant
- Pro: AI agent tool composition and access analysis is straightforward
- Pro: "Blast radius" and breaking change impact queries are first-class capabilities
- Con: Graph layer adds operational complexity (either ltree/recursive CTEs or a separate graph database)
- Con: Keeping relational and graph representations in sync requires careful consistency management
- Con: Development team needs graph modeling and query expertise (Cypher or recursive SQL)
- Con: Overkill for small platforms with fewer than 50 APIs and simple flat policies
- Con: Graph databases add infrastructure cost if using a dedicated engine like Neo4j

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenAPI 3.2 | API specifications stored relationally; cross-API references ($ref) modeled as graph edges |
| AsyncAPI 3.1 | Event channels and bindings as nodes; producer/consumer relationships as edges |
| GraphQL September 2025 | Federated subgraph composition modeled as graph relationships between GraphQL services |
| gRPC / Protocol Buffers | Service-to-service dependencies modeled as edges in the dependency graph |
| OAuth 2.0 / RFC 6749 | OAuth scope inheritance modeled as hierarchical paths in the policy graph |
| MCP Specification 2025-11-25 | Agent-to-tool-to-server composition chains modeled as graph paths for access analysis |
| OWASP API Security Top 10 | BOLA and excessive data exposure detection via graph traversal of access paths |
| NIST SP 800-228 | API inventory with ownership, dependency mapping, and security classification as graph properties |
| OpenTelemetry | Distributed trace topology visualization maps directly to the service dependency graph |
| ISO/IEC 27001:2022 | Access control and data flow analysis via graph traversal |

---

## Relational Layer (Operational CRUD)

```sql
-- Core entities use standard relational design
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    parent_org_id   UUID REFERENCES organizations(id),  -- organizational hierarchy
    org_path        LTREE,                               -- materialized path: "root.division.team"
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_org_path ON organizations USING GIST (org_path);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organization_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    UNIQUE (organization_id, user_id)
);

CREATE TABLE apis (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(200) NOT NULL,
    api_type        VARCHAR(30) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    visibility      VARCHAR(20) NOT NULL DEFAULT 'private',
    description     TEXT,
    data_classification VARCHAR(30) DEFAULT 'internal', -- public, internal, confidential, restricted
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, slug)
);

CREATE INDEX idx_apis_org ON apis(organization_id);

CREATE TABLE api_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_id          UUID NOT NULL REFERENCES apis(id) ON DELETE CASCADE,
    version_string  VARCHAR(50) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    spec_format     VARCHAR(30),
    spec_content    TEXT,
    protocol_config JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (api_id, version_string)
);

CREATE TABLE consumers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    consumer_type   VARCHAR(30) NOT NULL DEFAULT 'developer',
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    consumer_id     UUID NOT NULL REFERENCES consumers(id) ON DELETE CASCADE,
    api_product_id  UUID NOT NULL,
    plan_id         UUID NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    starts_at       TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE gateways (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    region          VARCHAR(50),
    deployment_type VARCHAR(30) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Graph Layer (Relationship Modeling)

The graph layer uses a generic node/edge pattern that can represent any relationship type. This is the core differentiator of this model.

```sql
-- Generic graph nodes: every entity that participates in relationships
-- has a corresponding graph node
CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type     VARCHAR(50) NOT NULL,     -- api, api_version, consumer, organization, team,
                                              -- gateway, policy, mcp_server, mcp_tool, api_product
    entity_id       UUID NOT NULL,            -- FK to the relational table
    label           VARCHAR(255) NOT NULL,    -- human-readable label for visualization
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example for an API node:
    -- {
    --   "api_type": "rest",
    --   "status": "published",
    --   "data_classification": "confidential",
    --   "endpoint_count": 12,
    --   "avg_latency_ms": 45
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (entity_type, entity_id)
);

CREATE INDEX idx_graph_nodes_type ON graph_nodes(entity_type);
CREATE INDEX idx_graph_nodes_entity ON graph_nodes(entity_id);
CREATE INDEX idx_graph_nodes_props ON graph_nodes USING GIN (properties);

-- Generic graph edges: typed, directed relationships between nodes
CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type       VARCHAR(50) NOT NULL,     -- see edge type catalogue below
    properties      JSONB NOT NULL DEFAULT '{}',
    weight          DECIMAL(10,4) DEFAULT 1.0, -- for weighted graph algorithms
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_node_id, target_node_id, edge_type)
);

CREATE INDEX idx_graph_edges_source ON graph_edges(source_node_id);
CREATE INDEX idx_graph_edges_target ON graph_edges(target_node_id);
CREATE INDEX idx_graph_edges_type ON graph_edges(edge_type);
CREATE INDEX idx_graph_edges_props ON graph_edges USING GIN (properties);
```

### Edge Type Catalogue

```
Edge Type                  | Source           | Target           | Meaning
---------------------------|------------------|------------------|------------------------------
DEPENDS_ON                 | api              | api              | API A calls API B at runtime
PUBLISHES_TO               | api              | api (async)      | API A publishes events consumed by API B
SUBSCRIBES_TO              | consumer         | api_product      | Consumer has active subscription
BUNDLES                    | api_product      | api_version      | Product includes this API version
ROUTES_THROUGH             | api_version      | gateway          | Traffic flows through this gateway
APPLIES_POLICY             | policy           | api/api_version  | Policy is enforced on target
INHERITS_POLICY            | organization     | organization     | Child org inherits parent policies
OWNS                       | organization     | api              | Organization owns this API
MEMBER_OF                  | user             | organization     | User belongs to organization
COMPOSES_TOOL              | mcp_server       | mcp_tool         | Server exposes this tool
AGENT_USES                 | consumer(agent)  | mcp_tool         | AI agent invokes this tool
PROVIDES_DATA              | api              | mcp_tool         | API backs an MCP tool's data source
FEDERATION_MEMBER          | api(graphql)     | api(graphql)     | Subgraph participates in federated graph
BREAKING_CHANGE            | api_version      | api_version      | New version has breaking changes from old
SHADOW_CANDIDATE           | discovered       | api              | Shadow endpoint may belong to this API
```

## Policy Inheritance Graph

```sql
-- Policies with hierarchical inheritance via graph edges
CREATE TABLE policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    policy_type     VARCHAR(50) NOT NULL,
    rule_definition JSONB NOT NULL,
    priority        INT NOT NULL DEFAULT 100,
    is_inheritable  BOOLEAN NOT NULL DEFAULT true,  -- can child orgs inherit this?
    override_mode   VARCHAR(20) NOT NULL DEFAULT 'merge', -- merge, replace, deny_override
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Policy inheritance is modeled through graph_edges (INHERITS_POLICY, APPLIES_POLICY)
-- The effective policy for an API is computed by traversing:
-- 1. Direct APPLIES_POLICY edges to the API
-- 2. Walk up INHERITS_POLICY edges through the org hierarchy
-- 3. Merge/override based on override_mode and priority
```

### Effective Policy Resolution Query

```sql
-- Compute effective policies for an API by traversing the org hierarchy
WITH RECURSIVE org_hierarchy AS (
    -- Start from the API's owning organization
    SELECT o.id, o.parent_org_id, o.org_path, 0 AS depth
    FROM apis a
    JOIN organizations o ON o.id = a.organization_id
    WHERE a.id = 'target-api-uuid'
    
    UNION ALL
    
    -- Walk up the org tree
    SELECT o.id, o.parent_org_id, o.org_path, oh.depth + 1
    FROM org_hierarchy oh
    JOIN organizations o ON o.id = oh.parent_org_id
),
applicable_policies AS (
    -- Collect all policies from the org hierarchy that are inheritable
    SELECT p.*, oh.depth
    FROM org_hierarchy oh
    JOIN policies p ON p.organization_id = oh.id
    WHERE p.status = 'active'
      AND (oh.depth = 0 OR p.is_inheritable = true)
    
    UNION ALL
    
    -- Plus any policies directly attached to the API via graph edges
    SELECT p.*, -1 AS depth
    FROM graph_nodes gn
    JOIN graph_edges ge ON ge.target_node_id = gn.id AND ge.edge_type = 'APPLIES_POLICY'
    JOIN graph_nodes source_gn ON source_gn.id = ge.source_node_id AND source_gn.entity_type = 'policy'
    JOIN policies p ON p.id = source_gn.entity_id
    WHERE gn.entity_type = 'api' AND gn.entity_id = 'target-api-uuid'
      AND p.status = 'active'
)
-- Most specific policy wins (lowest depth, then highest priority)
SELECT DISTINCT ON (policy_type) *
FROM applicable_policies
ORDER BY policy_type, depth ASC, priority DESC;
```

## Dependency & Impact Analysis

```sql
-- "What is the blast radius if Payment API v2 goes down?"
-- Find all APIs and consumers transitively affected

WITH RECURSIVE impact_graph AS (
    -- Start from the failing API
    SELECT gn.id AS node_id, gn.entity_type, gn.entity_id, gn.label,
           0 AS depth, ARRAY[gn.id] AS path
    FROM graph_nodes gn
    WHERE gn.entity_type = 'api' AND gn.entity_id = 'payment-api-uuid'
    
    UNION ALL
    
    -- Follow DEPENDS_ON edges (reverse direction: who depends on us?)
    SELECT gn2.id, gn2.entity_type, gn2.entity_id, gn2.label,
           ig.depth + 1, ig.path || gn2.id
    FROM impact_graph ig
    JOIN graph_edges ge ON ge.target_node_id = ig.node_id
        AND ge.edge_type IN ('DEPENDS_ON', 'PUBLISHES_TO', 'PROVIDES_DATA')
    JOIN graph_nodes gn2 ON gn2.id = ge.source_node_id
    WHERE NOT gn2.id = ANY(ig.path)  -- prevent cycles
      AND ig.depth < 10              -- max traversal depth
)
SELECT entity_type, entity_id, label, depth
FROM impact_graph
WHERE depth > 0
ORDER BY depth, entity_type;
```

## MCP Agent Access Analysis

```sql
-- "Does AI agent 'research-bot' have transitive access to PII data?"
-- Trace: agent → mcp_tools → apis → data_classification

WITH RECURSIVE agent_access AS (
    -- Start from the agent's consumer node
    SELECT gn.id AS node_id, gn.entity_type, gn.entity_id, gn.label,
           gn.properties, 0 AS depth, ARRAY[gn.id] AS path,
           ARRAY[]::text[] AS edge_types
    FROM graph_nodes gn
    WHERE gn.entity_type = 'consumer' AND gn.entity_id = 'research-bot-uuid'
    
    UNION ALL
    
    -- Follow AGENT_USES → COMPOSES_TOOL → PROVIDES_DATA edges
    SELECT gn2.id, gn2.entity_type, gn2.entity_id, gn2.label,
           gn2.properties, aa.depth + 1, aa.path || gn2.id,
           aa.edge_types || ge.edge_type
    FROM agent_access aa
    JOIN graph_edges ge ON ge.source_node_id = aa.node_id
        AND ge.edge_type IN ('AGENT_USES', 'COMPOSES_TOOL', 'PROVIDES_DATA')
    JOIN graph_nodes gn2 ON gn2.id = ge.target_node_id
    WHERE NOT gn2.id = ANY(aa.path)
      AND aa.depth < 5
)
SELECT entity_type, label, properties->>'data_classification' AS data_classification,
       depth, edge_types
FROM agent_access
WHERE entity_type = 'api'
  AND properties->>'data_classification' IN ('confidential', 'restricted');
```

## GraphQL Federation Graph

```sql
-- Find all subgraphs in a federated GraphQL composition
SELECT source_gn.label AS subgraph,
       target_gn.label AS federated_graph,
       ge.properties->>'subgraph_name' AS subgraph_name,
       ge.properties->>'contributed_types' AS types
FROM graph_edges ge
JOIN graph_nodes source_gn ON source_gn.id = ge.source_node_id
JOIN graph_nodes target_gn ON target_gn.id = ge.target_node_id
WHERE ge.edge_type = 'FEDERATION_MEMBER';
```

## Analytics & Observability

```sql
CREATE TABLE request_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_id          UUID NOT NULL,
    consumer_id     UUID,
    gateway_id      UUID,
    request_method  VARCHAR(10),
    request_path    VARCHAR(1000),
    response_status INT,
    latency_ms      INT,
    token_count     INT,
    client_ip       INET,
    correlation_id  VARCHAR(100),
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (requested_at);

CREATE INDEX idx_request_logs_api ON request_logs(api_id);
CREATE INDEX idx_request_logs_consumer ON request_logs(consumer_id);
CREATE INDEX idx_request_logs_time ON request_logs(requested_at);

-- Service topology derived from request correlation IDs
-- Updated periodically by analyzing distributed traces
CREATE TABLE service_topology (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_api_id   UUID NOT NULL REFERENCES apis(id),
    target_api_id   UUID NOT NULL REFERENCES apis(id),
    avg_calls_per_request DECIMAL(10,2),
    avg_latency_ms  DECIMAL(10,2),
    error_rate      DECIMAL(5,4),
    last_observed   TIMESTAMPTZ NOT NULL,
    sample_count    BIGINT NOT NULL,
    UNIQUE (source_api_id, target_api_id)
);
-- service_topology feeds DEPENDS_ON edges in the graph layer
```

## Audit Trail

```sql
CREATE TABLE audit_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    actor_id        UUID,
    actor_type      VARCHAR(30) NOT NULL,
    action          VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID,
    changes         JSONB,
    graph_impact    JSONB,
    -- graph_impact example:
    -- {
    --   "edges_added": [{"type": "DEPENDS_ON", "source": "api-A", "target": "api-B"}],
    --   "edges_removed": [],
    --   "affected_consumers": ["consumer-1", "consumer-2"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_org ON audit_logs(organization_id);
CREATE INDEX idx_audit_resource ON audit_logs(resource_type, resource_id);
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
    confidence      DECIMAL(5,4),
    status          VARCHAR(30) NOT NULL DEFAULT 'unreviewed',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- When a shadow endpoint is matched to an API, a SHADOW_CANDIDATE graph edge is created
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Identity | 3 | organizations (with ltree), users, organization_members |
| API Registry | 2 | apis, api_versions |
| Consumer & Subscriptions | 2 | consumers, subscriptions |
| Gateway | 1 | gateways |
| Graph Layer | 2 | graph_nodes, graph_edges (the power layer) |
| Policies | 1 | policies (inheritance via graph edges) |
| Analytics | 2 | request_logs, service_topology |
| Audit | 1 | audit_logs |
| Shadow Discovery | 1 | discovered_endpoints |
| **Total** | **15** | Plus graph edges encoding ~15 relationship types |

---

## Key Design Decisions

1. **Generic graph_nodes/graph_edges pattern** — rather than creating a junction table for every relationship type (api_dependencies, consumer_subscriptions, policy_assignments), a single pair of graph tables models all relationships. New relationship types are added by defining a new `edge_type` string, not by creating new tables.

2. **Ltree for organizational hierarchy** — PostgreSQL's `ltree` extension enables fast subtree queries ("find all APIs in any sub-team of the Engineering division") using `@>` and `<@` operators. Combined with GIST indexes, this outperforms recursive CTEs for pure hierarchical queries.

3. **Graph layer is eventually consistent with relational layer** — when an API is created in the `apis` table, a corresponding `graph_nodes` entry is created in the same transaction. Edge creation may happen asynchronously (e.g., dependency detection from traffic analysis).

4. **Weight column on graph_edges** — enables weighted graph algorithms: shortest path (fastest route through gateway clusters), minimum spanning tree (critical dependencies), PageRank (most important APIs in the platform).

5. **Properties JSONB on both nodes and edges** — each graph entity carries typed properties that can be used in traversal predicates. For example, "find all paths from Agent X to APIs where data_classification = 'restricted'" uses node properties during traversal.

6. **Service topology feeds the dependency graph** — the `service_topology` table is populated by analyzing distributed traces (OpenTelemetry correlation IDs). Its rows are periodically synced to `DEPENDS_ON` graph edges, creating an automatically maintained dependency map.

7. **Policy inheritance through graph traversal** — instead of a separate policy inheritance mechanism, the org hierarchy (ltree) combined with `INHERITS_POLICY` and `APPLIES_POLICY` edges computes effective policies. The `override_mode` on policies controls merge behavior.

8. **MCP agent access analysis as graph traversal** — the chain `consumer(agent) → AGENT_USES → mcp_tool → PROVIDES_DATA → api` enables transitive access analysis. This is essential for AI governance: ensuring agents cannot reach restricted data through tool composition.

9. **Audit logs capture graph impact** — when changes affect the graph topology (e.g., adding a dependency, changing a policy assignment), the `graph_impact` JSONB field records which edges were added/removed and which consumers are affected.

10. **Fewest tables of all models (15)** — the graph layer subsumes many relationship tables. The trade-off is that relationship queries use graph traversal rather than simple JOINs, requiring developers to be comfortable with recursive CTEs or Cypher queries.
