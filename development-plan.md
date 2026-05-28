# API Management Platform -- Phased Development Plan

> Project: 182-api-management-platform -- Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | TypeScript (Node.js 22 LTS) | API-heavy platform with web UI and OpenAPI spec generation; strong ecosystem for HTTP/WebSocket/gRPC proxying; shared types between gateway, management API, and developer portal frontend |
| API framework | Fastify 5 | High-performance HTTP framework with built-in schema validation (Ajv), OpenAPI 3.1 generation via @fastify/swagger, and plugin architecture mirroring the gateway's own plugin model |
| Gateway runtime | TypeScript + native proxy (http-proxy / undici) | Unified language with management plane; undici provides HTTP/1.1 and HTTP/2 client with connection pooling; custom middleware pipeline for policy enforcement |
| Database (primary) | PostgreSQL 17 | Supports JSONB for protocol-specific metadata (data model suggestion 3), ltree for org hierarchy (data model suggestion 4), time-partitioned tables for request logs, GIN indexes for flexible queries; required by all four data model suggestions |
| Database (cache/rate-limiting) | Redis 7 (Valkey compatible) | Sub-millisecond rate-limit counter operations via sliding window; shared state across gateway cluster nodes; pub/sub for configuration change propagation |
| Database (analytics) | TimescaleDB (PostgreSQL extension) | Time-series hypertable for request_logs with automatic partitioning, continuous aggregates for hourly/daily rollups, and retention policies; avoids a separate analytics database |
| Task queue | BullMQ (Redis-backed) | Async job processing for webhook delivery, spec validation, shadow API analysis, invoice generation, and ML pipeline triggers; supports retries, rate limiting, and priority queues |
| Frontend | Next.js 15 (App Router) | React-based developer portal and admin dashboard; server components for SEO-friendly API docs; shared TypeScript types with backend; Tailwind CSS for styling |
| ORM / query builder | Drizzle ORM | TypeScript-first schema definitions that generate SQL migrations; lightweight runtime with no hidden queries; PostgreSQL JSONB and ltree support via raw SQL escape hatches |
| Spec validation | @readme/openapi-parser + @asyncapi/parser | Validate imported OpenAPI 3.x and AsyncAPI 3.x specs; extract endpoints, schemas, and metadata for indexing |
| Authentication | Lucia Auth + oslo/jwt | Session-based auth for portal UI; JWT validation middleware for API gateway; OIDC client for external IdP integration |
| Containerisation | Docker + docker-compose | Multi-service deployment (gateway, management API, portal, PostgreSQL, Redis); Kubernetes Helm chart for production |
| Testing | Vitest + Supertest + Playwright | Vitest for unit/integration, Supertest for HTTP endpoint testing, Playwright for portal E2E tests |
| Code quality | Biome (lint + format) + tsc --noEmit | Single tool for linting and formatting; TypeScript strict mode for type checking |
| Package manager | pnpm 9 | Workspace support for monorepo (gateway, api, portal, shared packages); strict dependency resolution |
| Observability | OpenTelemetry SDK for Node.js | Traces, metrics, and logs exported via OTLP; aligns with OpenTelemetry standard from standards.md |
| CI/CD | GitHub Actions | Build, test, lint, Docker image push; matrix testing for Node 22 |

### Data Model Selection

This plan adopts **Data Model Suggestion 3 (Hybrid Relational + JSONB)** as the primary schema, enhanced with select elements from Suggestion 4 (graph_nodes/graph_edges for dependency analysis) and Suggestion 1 (explicit rate_limit_policies table for gateway hot-path performance). Rationale:

- 18 core tables vs 39 in full normalization -- manageable schema for MVP velocity
- JSONB protocol_config handles REST/GraphQL/gRPC/Kafka/MCP without per-protocol table sprawl
- GIN indexes on JSONB provide adequate query performance for the management plane
- Graph layer (2 additional tables) enables dependency and blast-radius analysis without full graph database overhead
- Relational columns for high-cardinality dimensions (api_type, status, consumer_type) preserve query performance

### Project Structure

```
api-management-platform/
├── pnpm-workspace.yaml
├── package.json
├── tsconfig.base.json
├── docker-compose.yml
├── Dockerfile.gateway
├── Dockerfile.api
├── Dockerfile.portal
├── .env.example
├── packages/
│   └── shared/                          # Shared types, constants, utilities
│       ├── src/
│       │   ├── types/
│       │   │   ├── api.ts               # API, ApiVersion, ApiEndpoint types
│       │   │   ├── consumer.ts          # Consumer, Subscription types
│       │   │   ├── policy.ts            # Policy, RateLimit types
│       │   │   ├── gateway.ts           # Gateway, Route types
│       │   │   ├── auth.ts              # OAuth, JWT, OIDC types
│       │   │   ├── analytics.ts         # RequestLog, Anomaly types
│       │   │   ├── mcp.ts               # McpServer, McpTool types
│       │   │   └── graph.ts             # GraphNode, GraphEdge types
│       │   ├── constants/
│       │   │   ├── api-types.ts         # API type enums
│       │   │   ├── status.ts            # Status enums
│       │   │   └── owasp.ts             # OWASP API Top 10 categories
│       │   ├── schemas/
│       │   │   ├── openapi-meta.ts      # JSON Schema for OpenAPI metadata
│       │   │   ├── asyncapi-meta.ts     # JSON Schema for AsyncAPI metadata
│       │   │   ├── graphql-meta.ts      # JSON Schema for GraphQL metadata
│       │   │   ├── grpc-meta.ts         # JSON Schema for gRPC metadata
│       │   │   └── mcp-meta.ts          # JSON Schema for MCP metadata
│       │   └── utils/
│       │       ├── slug.ts              # Slug generation and validation
│       │       ├── semver.ts            # Semver parsing
│       │       └── checksum.ts          # SHA-256 checksum utility
│       └── package.json
├── apps/
│   ├── gateway/                         # API Gateway (data plane)
│   │   ├── src/
│   │   │   ├── server.ts               # Gateway HTTP/HTTPS server
│   │   │   ├── proxy/
│   │   │   │   ├── router.ts           # Route matching engine
│   │   │   │   ├── upstream.ts         # Upstream connection manager
│   │   │   │   └── protocols/
│   │   │   │       ├── http.ts         # HTTP/HTTPS proxy
│   │   │   │       ├── websocket.ts    # WebSocket proxy
│   │   │   │       └── grpc.ts         # gRPC proxy
│   │   │   ├── middleware/
│   │   │   │   ├── pipeline.ts         # Middleware execution pipeline
│   │   │   │   ├── auth/
│   │   │   │   │   ├── jwt.ts          # JWT validation
│   │   │   │   │   ├── api-key.ts      # API key validation
│   │   │   │   │   └── oauth2.ts       # OAuth 2.0 token introspection
│   │   │   │   ├── rate-limit.ts       # Rate limiting (Redis-backed)
│   │   │   │   ├── quota.ts            # Quota enforcement
│   │   │   │   ├── cors.ts             # CORS policy
│   │   │   │   ├── request-transform.ts
│   │   │   │   ├── response-transform.ts
│   │   │   │   ├── logging.ts          # Request/response logging
│   │   │   │   └── token-metering.ts   # AI token counting
│   │   │   ├── config/
│   │   │   │   ├── loader.ts           # Config sync from management API
│   │   │   │   └── cache.ts            # Local route/policy cache
│   │   │   └── telemetry/
│   │   │       └── otel.ts             # OpenTelemetry integration
│   │   └── package.json
│   ├── api/                             # Management API (control plane)
│   │   ├── src/
│   │   │   ├── server.ts               # Fastify server entry
│   │   │   ├── db/
│   │   │   │   ├── schema.ts           # Drizzle schema definitions
│   │   │   │   ├── migrations/         # SQL migration files
│   │   │   │   └── seed.ts             # Development seed data
│   │   │   ├── routes/
│   │   │   │   ├── apis.ts             # /apis CRUD
│   │   │   │   ├── api-versions.ts     # /apis/:id/versions
│   │   │   │   ├── consumers.ts        # /consumers CRUD
│   │   │   │   ├── subscriptions.ts    # /subscriptions
│   │   │   │   ├── policies.ts         # /policies CRUD
│   │   │   │   ├── gateways.ts         # /gateways
│   │   │   │   ├── products.ts         # /products (API products)
│   │   │   │   ├── analytics.ts        # /analytics
│   │   │   │   ├── governance.ts       # /governance
│   │   │   │   ├── auth.ts             # /auth (login, register, OIDC)
│   │   │   │   └── admin.ts            # /admin (org management)
│   │   │   ├── services/
│   │   │   │   ├── api-lifecycle.ts     # API lifecycle state machine
│   │   │   │   ├── spec-validator.ts    # OpenAPI/AsyncAPI validation
│   │   │   │   ├── policy-engine.ts     # Policy evaluation
│   │   │   │   ├── subscription-mgr.ts  # Subscription workflows
│   │   │   │   ├── billing.ts           # Usage metering and invoicing
│   │   │   │   ├── governance.ts        # Governance rule evaluation
│   │   │   │   └── graph.ts             # Dependency graph operations
│   │   │   ├── jobs/
│   │   │   │   ├── worker.ts           # BullMQ worker entry
│   │   │   │   ├── spec-validation.ts  # Async spec validation job
│   │   │   │   ├── usage-rollup.ts     # Periodic usage aggregation
│   │   │   │   ├── invoice-gen.ts      # Invoice generation job
│   │   │   │   └── shadow-discovery.ts # Shadow API detection job
│   │   │   └── middleware/
│   │   │       ├── auth.ts             # Session/JWT auth middleware
│   │   │       ├── org-context.ts      # Multi-tenant org context
│   │   │       └── audit.ts            # Audit trail middleware
│   │   └── package.json
│   └── portal/                          # Developer Portal (Next.js)
│       ├── src/
│       │   └── app/
│       │       ├── layout.tsx
│       │       ├── page.tsx             # Landing / API catalog
│       │       ├── apis/
│       │       │   ├── page.tsx         # API listing
│       │       │   └── [slug]/
│       │       │       ├── page.tsx     # API detail + docs
│       │       │       └── try-it/
│       │       │           └── page.tsx # Interactive API testing
│       │       ├── dashboard/
│       │       │   ├── page.tsx         # Consumer dashboard
│       │       │   ├── subscriptions/
│       │       │   ├── keys/
│       │       │   └── usage/
│       │       ├── admin/
│       │       │   ├── apis/
│       │       │   ├── consumers/
│       │       │   ├── policies/
│       │       │   ├── gateways/
│       │       │   ├── governance/
│       │       │   └── analytics/
│       │       └── auth/
│       │           ├── login/
│       │           └── register/
│       └── package.json
└── tests/
    ├── fixtures/
    │   ├── openapi-petstore.yaml        # Valid OpenAPI 3.2 spec
    │   ├── openapi-invalid.yaml         # Invalid spec for error testing
    │   ├── asyncapi-kafka.yaml          # Valid AsyncAPI 3.1 spec
    │   ├── graphql-schema.graphql       # GraphQL SDL fixture
    │   └── grpc-service.proto           # Protobuf fixture
    ├── integration/
    │   ├── api/                         # Management API integration tests
    │   ├── gateway/                     # Gateway proxy tests
    │   └── db/                          # Database migration tests
    └── e2e/
        └── portal/                      # Playwright portal tests
```

---

## Phase 1: Foundation and Core Data Model

### Purpose

Establish the monorepo structure, shared type system, database schema with migrations, and the management API skeleton. After this phase, the database is running with the full schema, the API server starts and responds to health checks, and the shared type package is consumable by all apps.

### Tasks

#### 1.1 -- Monorepo Scaffolding and Toolchain

**What**: Initialize the pnpm workspace with shared, gateway, api, and portal packages, plus CI configuration.

**Design**:

```yaml
# pnpm-workspace.yaml
packages:
  - "packages/*"
  - "apps/*"
```

```jsonc
// tsconfig.base.json
{
  "compilerOptions": {
    "target": "ES2024",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "paths": {
      "@apim/shared": ["../../packages/shared/src"],
      "@apim/shared/*": ["../../packages/shared/src/*"]
    }
  }
}
```

```yaml
# docker-compose.yml
services:
  postgres:
    image: timescale/timescaledb:latest-pg17
    environment:
      POSTGRES_DB: apim
      POSTGRES_USER: apim
      POSTGRES_PASSWORD: apim_dev
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  pgdata:
```

Environment configuration:

```typescript
// apps/api/src/config.ts
export interface AppConfig {
  port: number;                    // default: 3001
  host: string;                    // default: "0.0.0.0"
  databaseUrl: string;             // default: "postgres://apim:apim_dev@localhost:5432/apim"
  redisUrl: string;                // default: "redis://localhost:6379"
  jwtSecret: string;               // required, no default
  logLevel: "debug" | "info" | "warn" | "error"; // default: "info"
  corsOrigins: string[];           // default: ["http://localhost:3000"]
}
```

**Testing**:
- `Unit: pnpm install completes without errors across all workspaces`
- `Unit: tsc --noEmit passes for all packages`
- `Unit: biome check passes for all packages`
- `Integration: docker-compose up starts PostgreSQL and Redis, both accept connections`
- `Integration: management API starts and responds to GET /health with { status: "ok" }`

#### 1.2 -- Shared Type Definitions

**What**: Define TypeScript types for all core domain entities in the shared package, aligned with data model suggestion 3.

**Design**:

```typescript
// packages/shared/src/types/api.ts
export type ApiType = "rest" | "graphql" | "grpc" | "websocket" | "kafka" | "mqtt" | "mcp";
export type ApiStatus = "draft" | "published" | "deprecated" | "retired";
export type Visibility = "private" | "internal" | "public";

export interface Api {
  id: string;                      // UUID
  organizationId: string;
  name: string;
  slug: string;
  apiType: ApiType;
  status: ApiStatus;
  visibility: Visibility;
  ownerTeam: string | null;
  tags: string[];
  labels: Record<string, string | string[]>;
  observabilityConfig: ObservabilityConfig;
  createdAt: Date;
  updatedAt: Date;
}

export interface ObservabilityConfig {
  tracingEnabled: boolean;
  samplingRate: number;            // 0.0 - 1.0
  logLevel: "debug" | "info" | "warn" | "error";
  otelExporter: "otlp" | "jaeger" | "zipkin";
  exportEndpoint?: string;
}

export interface ApiVersion {
  id: string;
  apiId: string;
  versionString: string;          // e.g., "v2.1.0"
  status: ApiStatus;
  specDocument: string | null;    // raw OpenAPI/AsyncAPI/SDL/proto text
  specFormat: SpecFormat | null;
  specChecksum: string | null;    // SHA-256
  protocolConfig: Record<string, unknown>; // protocol-specific JSONB
  changelog: string | null;
  deprecatedAt: Date | null;
  createdAt: Date;
  updatedAt: Date;
}

export type SpecFormat = "openapi_3_2" | "asyncapi_3_1" | "graphql_sdl" | "protobuf" | "mcp_manifest";
```

```typescript
// packages/shared/src/types/consumer.ts
export type ConsumerType = "developer" | "application" | "ai_agent";
export type SubscriptionStatus = "pending" | "active" | "suspended" | "cancelled";

export interface Consumer {
  id: string;
  organizationId: string;
  name: string;
  email: string | null;
  consumerType: ConsumerType;
  status: "active" | "suspended" | "deleted";
  metadata: Record<string, unknown>;
  credentials: ConsumerCredentials;
  createdAt: Date;
  updatedAt: Date;
}

export interface ConsumerCredentials {
  apiKeys?: ApiKeyCredential[];
  oauthClientId?: string;
  mtlsFingerprint?: string;
}

export interface ApiKeyCredential {
  keyPrefix: string;
  keyHash: string;
  name: string;
  status: "active" | "revoked" | "expired";
  expiresAt?: Date;
  createdAt: Date;
}

export interface Subscription {
  id: string;
  consumerId: string;
  apiProductId: string;
  planId: string;
  status: SubscriptionStatus;
  billingInfo: BillingInfo;
  approvedBy: string | null;
  startsAt: Date | null;
  expiresAt: Date | null;
  createdAt: Date;
  updatedAt: Date;
}

export interface BillingInfo {
  currentPeriodStart: Date;
  currentPeriodEnd: Date;
  usage: { requests: number; tokens: number; bytes: number };
  computedCostCents: number;
  paymentMethod?: string;
}
```

```typescript
// packages/shared/src/types/policy.ts
export type PolicyCategory = "security" | "rate_limit" | "transformation" | "caching" | "cors" | "logging";
export type PolicyType =
  | "jwt_validation" | "oauth2_enforcement" | "api_key_validation" | "ip_whitelist" | "mtls"
  | "request_rate" | "token_rate" | "spike_arrest"
  | "request_transform" | "response_transform"
  | "response_cache" | "cors_policy" | "request_logging";

export interface Policy {
  id: string;
  organizationId: string;
  name: string;
  policyCategory: PolicyCategory;
  policyType: PolicyType;
  priority: number;
  isGlobal: boolean;
  owaspCategory: string | null;    // OWASP API Security Top 10 2023 category
  ruleDefinition: Record<string, unknown>;
  status: "active" | "inactive";
  createdAt: Date;
  updatedAt: Date;
}

export interface PolicyBinding {
  id: string;
  policyId: string;
  targetType: "api" | "api_version" | "consumer" | "api_product" | "gateway";
  targetId: string;
  overrideConfig: Record<string, unknown> | null;
  isActive: boolean;
  createdAt: Date;
}

export interface RateLimitRule {
  limitType: "requests_per_second" | "requests_per_minute" | "tokens_per_minute" | "bandwidth_bytes";
  limitValue: number;
  windowSeconds: number;
  burstLimit?: number;
  scope: "global" | "per_consumer" | "per_api_key" | "per_ip";
  actionOnLimit: "reject" | "queue" | "throttle";
}
```

```typescript
// packages/shared/src/types/gateway.ts
export interface Gateway {
  id: string;
  organizationId: string;
  name: string;
  deploymentType: "cloud" | "self_hosted" | "edge";
  region: string | null;
  status: "active" | "inactive" | "draining";
  config: GatewayConfig;
  createdAt: Date;
  updatedAt: Date;
}

export interface GatewayConfig {
  gatewayUrl: string;
  adminUrl?: string;
  tls?: { certPath: string; minVersion: "1.2" | "1.3" };
  proxyBufferSize?: string;
  accessLogFormat?: "json" | "combined";
  healthCheck?: { intervalSeconds: number; timeoutSeconds: number };
}

export interface Route {
  id: string;
  apiVersionId: string;
  gatewayId: string;
  isActive: boolean;
  routingConfig: RoutingConfig;
  createdAt: Date;
  updatedAt: Date;
}

export interface RoutingConfig {
  upstreamUrl: string;
  pathPrefix: string;
  stripPath: boolean;
  hosts: string[];
  protocols: ("http" | "https" | "ws" | "wss" | "grpc" | "grpcs")[];
  timeoutMs: number;
  retries: number;
  loadBalancing: "round_robin" | "least_conn" | "ip_hash";
  circuitBreaker?: { threshold: number; timeoutSeconds: number };
}
```

```typescript
// packages/shared/src/types/graph.ts
export type GraphEntityType =
  | "api" | "api_version" | "consumer" | "organization" | "team"
  | "gateway" | "policy" | "mcp_server" | "mcp_tool" | "api_product";

export type GraphEdgeType =
  | "DEPENDS_ON" | "PUBLISHES_TO" | "SUBSCRIBES_TO" | "BUNDLES"
  | "ROUTES_THROUGH" | "APPLIES_POLICY" | "INHERITS_POLICY" | "OWNS"
  | "MEMBER_OF" | "COMPOSES_TOOL" | "AGENT_USES" | "PROVIDES_DATA"
  | "FEDERATION_MEMBER" | "BREAKING_CHANGE" | "SHADOW_CANDIDATE";

export interface GraphNode {
  id: string;
  entityType: GraphEntityType;
  entityId: string;
  label: string;
  properties: Record<string, unknown>;
  createdAt: Date;
  updatedAt: Date;
}

export interface GraphEdge {
  id: string;
  sourceNodeId: string;
  targetNodeId: string;
  edgeType: GraphEdgeType;
  properties: Record<string, unknown>;
  weight: number;
  createdAt: Date;
}
```

**Testing**:
- `Unit: all type definitions compile without errors under strict TypeScript`
- `Unit: type guard functions (isApiType, isApiStatus) correctly narrow types for valid/invalid inputs`
- `Unit: shared package exports all types from the package entry point`

#### 1.3 -- Database Schema and Migrations

**What**: Define the Drizzle ORM schema matching data model suggestion 3 (hybrid relational + JSONB) plus graph tables from suggestion 4, generate and run initial migration.

**Design**:

```typescript
// apps/api/src/db/schema.ts
import { pgTable, uuid, varchar, text, boolean, integer, bigint,
         timestamp, jsonb, inet, index, uniqueIndex, pgEnum } from "drizzle-orm/pg-core";

export const organizations = pgTable("organizations", {
  id: uuid("id").primaryKey().defaultRandom(),
  name: varchar("name", { length: 255 }).notNull(),
  slug: varchar("slug", { length: 100 }).notNull().unique(),
  planTier: varchar("plan_tier", { length: 50 }).notNull().default("free"),
  status: varchar("status", { length: 20 }).notNull().default("active"),
  settings: jsonb("settings").notNull().default({}),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

export const users = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  email: varchar("email", { length: 255 }).notNull().unique(),
  displayName: varchar("display_name", { length: 255 }),
  passwordHash: varchar("password_hash", { length: 255 }),
  status: varchar("status", { length: 20 }).notNull().default("active"),
  profile: jsonb("profile").notNull().default({}),
  lastLoginAt: timestamp("last_login_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

export const organizationMemberships = pgTable("organization_memberships", {
  id: uuid("id").primaryKey().defaultRandom(),
  organizationId: uuid("organization_id").notNull().references(() => organizations.id, { onDelete: "cascade" }),
  userId: uuid("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  role: varchar("role", { length: 50 }).notNull().default("member"),
  permissions: jsonb("permissions").notNull().default([]),
  joinedAt: timestamp("joined_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex("uq_org_membership").on(table.organizationId, table.userId),
]);

export const apis = pgTable("apis", {
  id: uuid("id").primaryKey().defaultRandom(),
  organizationId: uuid("organization_id").notNull().references(() => organizations.id, { onDelete: "cascade" }),
  name: varchar("name", { length: 255 }).notNull(),
  slug: varchar("slug", { length: 200 }).notNull(),
  apiType: varchar("api_type", { length: 30 }).notNull(),
  status: varchar("status", { length: 30 }).notNull().default("draft"),
  visibility: varchar("visibility", { length: 20 }).notNull().default("private"),
  ownerTeam: varchar("owner_team", { length: 255 }),
  tags: text("tags").array().default([]),
  labels: jsonb("labels").notNull().default({}),
  observabilityConfig: jsonb("observability_config").notNull().default({}),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex("uq_api_org_slug").on(table.organizationId, table.slug),
  index("idx_apis_org").on(table.organizationId),
  index("idx_apis_type").on(table.apiType),
  index("idx_apis_status").on(table.status),
]);

export const apiVersions = pgTable("api_versions", {
  id: uuid("id").primaryKey().defaultRandom(),
  apiId: uuid("api_id").notNull().references(() => apis.id, { onDelete: "cascade" }),
  versionString: varchar("version_string", { length: 50 }).notNull(),
  status: varchar("status", { length: 30 }).notNull().default("draft"),
  specDocument: text("spec_document"),
  specFormat: varchar("spec_format", { length: 30 }),
  specChecksum: varchar("spec_checksum", { length: 64 }),
  protocolConfig: jsonb("protocol_config").notNull().default({}),
  changelog: text("changelog"),
  deprecatedAt: timestamp("deprecated_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex("uq_api_version").on(table.apiId, table.versionString),
  index("idx_api_versions_api").on(table.apiId),
]);

// Remaining tables: gateways, routes, auth_providers, policies, policy_bindings,
// consumers, api_products, subscriptions, request_logs, anomaly_events,
// governance_rulesets, audit_trail, discovered_endpoints, graph_nodes, graph_edges
// All follow the same pattern from data model suggestions 3 and 4.

export const gateways = pgTable("gateways", {
  id: uuid("id").primaryKey().defaultRandom(),
  organizationId: uuid("organization_id").notNull().references(() => organizations.id, { onDelete: "cascade" }),
  name: varchar("name", { length: 255 }).notNull(),
  deploymentType: varchar("deployment_type", { length: 30 }).notNull(),
  region: varchar("region", { length: 50 }),
  status: varchar("status", { length: 20 }).notNull().default("active"),
  config: jsonb("config").notNull().default({}),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

export const routes = pgTable("routes", {
  id: uuid("id").primaryKey().defaultRandom(),
  apiVersionId: uuid("api_version_id").notNull().references(() => apiVersions.id, { onDelete: "cascade" }),
  gatewayId: uuid("gateway_id").notNull().references(() => gateways.id, { onDelete: "cascade" }),
  isActive: boolean("is_active").notNull().default(true),
  routingConfig: jsonb("routing_config").notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index("idx_routes_api_version").on(table.apiVersionId),
  index("idx_routes_gateway").on(table.gatewayId),
]);

export const consumers = pgTable("consumers", {
  id: uuid("id").primaryKey().defaultRandom(),
  organizationId: uuid("organization_id").notNull().references(() => organizations.id, { onDelete: "cascade" }),
  name: varchar("name", { length: 255 }).notNull(),
  email: varchar("email", { length: 255 }),
  consumerType: varchar("consumer_type", { length: 30 }).notNull().default("developer"),
  status: varchar("status", { length: 20 }).notNull().default("active"),
  metadata: jsonb("metadata").notNull().default({}),
  credentials: jsonb("credentials").notNull().default({}),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index("idx_consumers_org").on(table.organizationId),
  index("idx_consumers_type").on(table.consumerType),
]);

export const policies = pgTable("policies", {
  id: uuid("id").primaryKey().defaultRandom(),
  organizationId: uuid("organization_id").notNull().references(() => organizations.id, { onDelete: "cascade" }),
  name: varchar("name", { length: 255 }).notNull(),
  policyCategory: varchar("policy_category", { length: 30 }).notNull(),
  policyType: varchar("policy_type", { length: 50 }).notNull(),
  priority: integer("priority").notNull().default(100),
  isGlobal: boolean("is_global").notNull().default(false),
  owaspCategory: varchar("owasp_category", { length: 20 }),
  ruleDefinition: jsonb("rule_definition").notNull(),
  status: varchar("status", { length: 20 }).notNull().default("active"),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index("idx_policies_org").on(table.organizationId),
  index("idx_policies_category").on(table.policyCategory),
]);

export const policyBindings = pgTable("policy_bindings", {
  id: uuid("id").primaryKey().defaultRandom(),
  policyId: uuid("policy_id").notNull().references(() => policies.id, { onDelete: "cascade" }),
  targetType: varchar("target_type", { length: 30 }).notNull(),
  targetId: uuid("target_id").notNull(),
  overrideConfig: jsonb("override_config"),
  isActive: boolean("is_active").notNull().default(true),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex("uq_policy_binding").on(table.policyId, table.targetType, table.targetId),
  index("idx_policy_bindings_target").on(table.targetType, table.targetId),
]);

export const graphNodes = pgTable("graph_nodes", {
  id: uuid("id").primaryKey().defaultRandom(),
  entityType: varchar("entity_type", { length: 50 }).notNull(),
  entityId: uuid("entity_id").notNull(),
  label: varchar("label", { length: 255 }).notNull(),
  properties: jsonb("properties").notNull().default({}),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex("uq_graph_node_entity").on(table.entityType, table.entityId),
  index("idx_graph_nodes_type").on(table.entityType),
]);

export const graphEdges = pgTable("graph_edges", {
  id: uuid("id").primaryKey().defaultRandom(),
  sourceNodeId: uuid("source_node_id").notNull().references(() => graphNodes.id, { onDelete: "cascade" }),
  targetNodeId: uuid("target_node_id").notNull().references(() => graphNodes.id, { onDelete: "cascade" }),
  edgeType: varchar("edge_type", { length: 50 }).notNull(),
  properties: jsonb("properties").notNull().default({}),
  weight: integer("weight").notNull().default(1),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex("uq_graph_edge").on(table.sourceNodeId, table.targetNodeId, table.edgeType),
  index("idx_graph_edges_source").on(table.sourceNodeId),
  index("idx_graph_edges_target").on(table.targetNodeId),
  index("idx_graph_edges_type").on(table.edgeType),
]);

export const auditTrail = pgTable("audit_trail", {
  id: uuid("id").primaryKey().defaultRandom(),
  organizationId: uuid("organization_id").notNull(),
  actorId: uuid("actor_id"),
  actorType: varchar("actor_type", { length: 30 }).notNull(),
  action: varchar("action", { length: 100 }).notNull(),
  resourceType: varchar("resource_type", { length: 50 }).notNull(),
  resourceId: uuid("resource_id"),
  context: jsonb("context").notNull().default({}),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index("idx_audit_org").on(table.organizationId),
  index("idx_audit_resource").on(table.resourceType, table.resourceId),
  index("idx_audit_time").on(table.createdAt),
]);
```

**Testing**:
- `Unit: Drizzle schema compiles and generates valid SQL DDL`
- `Integration: drizzle-kit generate produces migration SQL without errors`
- `Integration: drizzle-kit migrate applies migration to running PostgreSQL`
- `Integration: all tables exist with correct columns, indexes, and constraints after migration`
- `Integration: JSONB columns accept and return valid JSON objects`
- `Integration: unique constraints reject duplicate entries (org slug, api org+slug, graph node entity)`

#### 1.4 -- Management API Skeleton with Health and Org CRUD

**What**: Fastify server with health endpoint, organization CRUD routes, and OpenAPI documentation generation.

**Design**:

```typescript
// apps/api/src/server.ts
import Fastify from "fastify";
import swagger from "@fastify/swagger";
import swaggerUi from "@fastify/swagger-ui";
import cors from "@fastify/cors";

export async function buildServer(config: AppConfig) {
  const app = Fastify({ logger: { level: config.logLevel } });

  await app.register(cors, { origin: config.corsOrigins });
  await app.register(swagger, {
    openapi: {
      info: {
        title: "API Management Platform",
        version: "0.1.0",
      },
      components: {
        securitySchemes: {
          bearerAuth: { type: "http", scheme: "bearer", bearerFormat: "JWT" },
        },
      },
    },
  });
  await app.register(swaggerUi, { routePrefix: "/docs" });

  // Health endpoint
  app.get("/health", {
    schema: {
      response: {
        200: {
          type: "object",
          properties: {
            status: { type: "string" },
            version: { type: "string" },
            uptime: { type: "number" },
          },
        },
      },
    },
  }, async () => ({
    status: "ok",
    version: process.env.npm_package_version ?? "0.1.0",
    uptime: process.uptime(),
  }));

  // Register route modules
  await app.register(import("./routes/admin"), { prefix: "/api/v1" });

  return app;
}
```

```typescript
// apps/api/src/routes/admin.ts -- Organization CRUD
import { FastifyPluginAsync } from "fastify";

const adminRoutes: FastifyPluginAsync = async (app) => {
  // POST /api/v1/organizations
  app.post("/organizations", {
    schema: {
      body: {
        type: "object",
        required: ["name", "slug"],
        properties: {
          name: { type: "string", maxLength: 255 },
          slug: { type: "string", maxLength: 100, pattern: "^[a-z0-9-]+$" },
          planTier: { type: "string", enum: ["free", "starter", "enterprise"] },
          settings: { type: "object" },
        },
      },
      response: { 201: { $ref: "Organization" } },
    },
  }, async (request, reply) => {
    // Insert into organizations table
    // Return created org
  });

  // GET /api/v1/organizations
  // GET /api/v1/organizations/:id
  // PUT /api/v1/organizations/:id
  // DELETE /api/v1/organizations/:id
};
```

**Testing**:
- `Unit: buildServer returns a Fastify instance that starts without errors`
- `Integration: GET /health returns 200 with status "ok"`
- `Integration: GET /docs returns Swagger UI HTML`
- `Integration: POST /api/v1/organizations with valid body returns 201 and persists to database`
- `Integration: POST /api/v1/organizations with duplicate slug returns 409`
- `Integration: GET /api/v1/organizations returns paginated list`
- `Integration: GET /api/v1/organizations/:id returns 404 for non-existent ID`
- `Integration: PUT /api/v1/organizations/:id updates name and settings`
- `Integration: DELETE /api/v1/organizations/:id sets status to "deleted"`

---

## Phase 2: API Registry and Lifecycle Management

### Purpose

Implement the core API registry: creating APIs, uploading versions with specifications, validating OpenAPI/AsyncAPI specs, and managing lifecycle transitions (draft -> published -> deprecated -> retired). After this phase, users can register APIs, upload specs, and manage their lifecycle.

### Tasks

#### 2.1 -- API CRUD Routes

**What**: Full CRUD for the `apis` table with filtering by type, status, visibility, and tags.

**Design**:

```typescript
// apps/api/src/routes/apis.ts
// POST   /api/v1/apis                 -- Create API
// GET    /api/v1/apis                 -- List APIs (filters: apiType, status, visibility, tags, search)
// GET    /api/v1/apis/:id             -- Get API by ID
// PUT    /api/v1/apis/:id             -- Update API
// DELETE /api/v1/apis/:id             -- Soft delete (status -> retired)

interface CreateApiRequest {
  name: string;
  slug: string;
  apiType: ApiType;
  visibility?: Visibility;           // default: "private"
  ownerTeam?: string;
  tags?: string[];
  labels?: Record<string, string | string[]>;
  description?: string;
}

interface ListApisQuery {
  apiType?: ApiType;
  status?: ApiStatus;
  visibility?: Visibility;
  tags?: string[];                   // filter by any matching tag
  search?: string;                   // full-text search on name and description
  page?: number;                     // default: 1
  pageSize?: number;                 // default: 20, max: 100
  sortBy?: "name" | "createdAt" | "updatedAt";
  sortOrder?: "asc" | "desc";
}

interface PaginatedResponse<T> {
  data: T[];
  pagination: {
    page: number;
    pageSize: number;
    total: number;
    totalPages: number;
  };
}
```

**Testing**:
- `Unit: slug validation rejects slugs with uppercase, spaces, or special characters`
- `Unit: apiType validation rejects unknown types`
- `Integration: POST /apis with valid REST API returns 201 with generated UUID`
- `Integration: POST /apis with duplicate org+slug returns 409`
- `Integration: GET /apis?apiType=graphql returns only GraphQL APIs`
- `Integration: GET /apis?tags=payments returns APIs tagged with "payments"`
- `Integration: GET /apis?search=payment returns APIs with "payment" in name or description`
- `Integration: GET /apis?page=2&pageSize=5 returns correct pagination metadata`
- `Integration: PUT /apis/:id transitions status from draft to published`
- `Integration: DELETE /apis/:id sets status to retired and soft-deletes`

#### 2.2 -- API Version Management

**What**: CRUD for API versions, including spec upload, semver parsing, and lifecycle transitions.

**Design**:

```typescript
// apps/api/src/routes/api-versions.ts
// POST   /api/v1/apis/:apiId/versions           -- Create version
// GET    /api/v1/apis/:apiId/versions            -- List versions
// GET    /api/v1/apis/:apiId/versions/:versionId -- Get version
// PUT    /api/v1/apis/:apiId/versions/:versionId -- Update version
// POST   /api/v1/apis/:apiId/versions/:versionId/spec -- Upload spec document
// GET    /api/v1/apis/:apiId/versions/:versionId/spec -- Download spec document

interface CreateVersionRequest {
  versionString: string;             // must be valid semver (v1.0.0) or free-form
  changelog?: string;
  protocolConfig?: Record<string, unknown>;
}

// API version lifecycle state machine
// draft -> published -> deprecated -> retired
// draft -> retired (skip published if never released)
// published -> draft (revert to draft for corrections - only if no active subscriptions)

interface VersionLifecycleTransition {
  from: ApiStatus;
  to: ApiStatus;
  guard?: (version: ApiVersion) => Promise<{ allowed: boolean; reason?: string }>;
}

const LIFECYCLE_TRANSITIONS: VersionLifecycleTransition[] = [
  { from: "draft", to: "published" },
  { from: "draft", to: "retired" },
  { from: "published", to: "deprecated", guard: async (v) => {
    // Must have a newer published version before deprecating
    return { allowed: true };
  }},
  { from: "deprecated", to: "retired" },
  { from: "published", to: "draft", guard: async (v) => {
    // Only allowed if no active subscriptions reference this version
    return { allowed: true };
  }},
];
```

```typescript
// apps/api/src/services/api-lifecycle.ts
export class ApiLifecycleService {
  async transitionVersion(
    versionId: string,
    targetStatus: ApiStatus,
    actorId: string,
  ): Promise<{ success: boolean; error?: string }>;

  async getAvailableTransitions(versionId: string): Promise<ApiStatus[]>;

  async validateSpecBeforePublish(versionId: string): Promise<ValidationResult>;
}

interface ValidationResult {
  valid: boolean;
  errors: ValidationError[];
  warnings: ValidationWarning[];
}
```

**Testing**:
- `Unit: semver parser extracts major/minor/patch from "v2.1.0", "2.1.0", "v0.1.0-beta"`
- `Unit: lifecycle transitions allow draft->published and reject retired->draft`
- `Unit: lifecycle guard prevents deprecation when no newer published version exists`
- `Integration: POST /apis/:id/versions creates version with draft status`
- `Integration: POST /apis/:id/versions with duplicate versionString returns 409`
- `Integration: POST /apis/:id/versions/:vid/spec uploads and stores spec with SHA-256 checksum`
- `Integration: GET /apis/:id/versions/:vid/spec returns raw spec document with correct content-type`
- `Integration: PUT /apis/:id/versions/:vid with status transition draft->published succeeds`
- `Integration: PUT /apis/:id/versions/:vid with invalid transition returns 422 with reason`
- `Fixture: upload openapi-petstore.yaml fixture, verify parsed endpoint count matches expected`

#### 2.3 -- Spec Validation Service

**What**: Validate OpenAPI 3.x and AsyncAPI 3.x specifications on upload, extracting endpoint metadata into protocolConfig.

**Design**:

```typescript
// apps/api/src/services/spec-validator.ts
import { validate as validateOpenAPI } from "@readme/openapi-parser";
import { Parser as AsyncAPIParser } from "@asyncapi/parser";

export class SpecValidatorService {
  async validateSpec(
    content: string,
    format: SpecFormat,
  ): Promise<SpecValidationResult>;

  async extractEndpoints(
    content: string,
    format: SpecFormat,
  ): Promise<ExtractedEndpoint[]>;

  async detectBreakingChanges(
    oldSpec: string,
    newSpec: string,
    format: SpecFormat,
  ): Promise<BreakingChange[]>;
}

interface SpecValidationResult {
  valid: boolean;
  specVersion: string;               // e.g., "3.2.0" for OpenAPI
  errors: Array<{
    path: string;                    // JSONPath to the error location
    message: string;
    severity: "error" | "warning";
  }>;
  metadata: {
    title: string;
    description?: string;
    endpointCount: number;
    schemaCount: number;
  };
}

interface ExtractedEndpoint {
  method: string;                    // GET, POST, PUT, DELETE, PATCH
  path: string;                      // /users/{id}
  operationId?: string;
  summary?: string;
  tags?: string[];
  deprecated?: boolean;
  requestBodySchema?: object;        // JSON Schema
  responseSchemas?: Record<string, object>; // status code -> JSON Schema
}

interface BreakingChange {
  type: "endpoint_removed" | "parameter_required" | "response_schema_changed" | "type_changed";
  path: string;
  description: string;
  severity: "breaking" | "non_breaking";
}
```

**Testing**:
- `Fixture: validate openapi-petstore.yaml returns valid: true with correct endpoint count`
- `Fixture: validate openapi-invalid.yaml returns valid: false with specific error paths`
- `Fixture: validate asyncapi-kafka.yaml returns valid: true with channel metadata`
- `Unit: extractEndpoints from Petstore spec returns GET /pets, POST /pets, GET /pets/{petId}`
- `Unit: detectBreakingChanges between two specs identifies removed endpoint as breaking`
- `Unit: detectBreakingChanges between two specs identifies added optional parameter as non-breaking`
- `Integration: spec upload with invalid YAML returns 400 with parse error`
- `Integration: spec upload triggers async validation job and stores results`

#### 2.4 -- Audit Trail Middleware

**What**: Automatic audit logging for all write operations on core entities.

**Design**:

```typescript
// apps/api/src/middleware/audit.ts
import { FastifyPluginAsync } from "fastify";

export const auditPlugin: FastifyPluginAsync = async (app) => {
  app.addHook("onResponse", async (request, reply) => {
    if (["POST", "PUT", "PATCH", "DELETE"].includes(request.method)) {
      await recordAuditEvent({
        organizationId: request.orgContext?.organizationId,
        actorId: request.user?.id,
        actorType: request.user ? "user" : "system",
        action: deriveAction(request.method, request.routeOptions.url),
        resourceType: deriveResourceType(request.routeOptions.url),
        resourceId: request.params?.id,
        context: {
          changes: request.auditChanges,      // set by route handlers
          ipAddress: request.ip,
          userAgent: request.headers["user-agent"],
          statusCode: reply.statusCode,
        },
      });
    }
  });
};

function deriveAction(method: string, path: string): string {
  // POST /api/v1/apis -> "api.created"
  // PUT /api/v1/apis/:id -> "api.updated"
  // DELETE /api/v1/apis/:id -> "api.deleted"
}
```

**Testing**:
- `Integration: POST /apis creates an audit_trail record with action "api.created"`
- `Integration: PUT /apis/:id creates an audit_trail record with before/after changes in context`
- `Integration: DELETE /apis/:id creates an audit_trail record with action "api.deleted"`
- `Integration: GET requests do not create audit records`
- `Integration: audit records include IP address, user agent, and actor ID`

---

## Phase 3: Security, Authentication, and Policy Engine

### Purpose

Implement the authentication system (user registration, login, JWT issuance), the policy engine (CRUD for policies, policy bindings), and API key management. After this phase, the management API is secured, consumers can get API keys, and policies can be attached to APIs.

### Tasks

#### 3.1 -- User Authentication (Registration, Login, JWT)

**What**: User registration, login, session management, and JWT token issuance for API access.

**Design**:

```typescript
// apps/api/src/routes/auth.ts
// POST /api/v1/auth/register         -- Register new user
// POST /api/v1/auth/login            -- Login and get JWT
// POST /api/v1/auth/refresh           -- Refresh JWT
// POST /api/v1/auth/logout            -- Invalidate session
// GET  /api/v1/auth/me                -- Get current user

interface RegisterRequest {
  email: string;
  password: string;                  // min 12 chars, per RFC 9700 security guidance
  displayName: string;
  organizationName?: string;         // creates org if provided
  organizationSlug?: string;
}

interface LoginRequest {
  email: string;
  password: string;
}

interface AuthResponse {
  accessToken: string;               // JWT, 15 min lifetime
  refreshToken: string;              // opaque token, 7 day lifetime
  expiresIn: number;                 // seconds until access token expiry
  user: { id: string; email: string; displayName: string };
}

// JWT payload structure
interface JwtPayload {
  sub: string;                       // user ID
  email: string;
  orgId: string;                     // current organization context
  role: string;                      // role in current org
  iat: number;
  exp: number;
}
```

```typescript
// apps/api/src/middleware/auth.ts
export const requireAuth: FastifyPluginAsync = async (app) => {
  app.addHook("preHandler", async (request, reply) => {
    const token = request.headers.authorization?.replace("Bearer ", "");
    if (!token) {
      reply.code(401).send({ error: "Missing authorization header" });
      return;
    }
    const payload = await verifyJwt(token, config.jwtSecret);
    request.user = payload;
  });
};
```

**Testing**:
- `Unit: password hashing uses Argon2id with correct parameters`
- `Unit: JWT generation includes all required claims (sub, email, orgId, role, iat, exp)`
- `Unit: JWT verification rejects expired tokens`
- `Unit: JWT verification rejects tokens with invalid signature`
- `Integration: POST /auth/register with valid email creates user and org`
- `Integration: POST /auth/register with existing email returns 409`
- `Integration: POST /auth/register with weak password returns 400`
- `Integration: POST /auth/login with valid credentials returns JWT`
- `Integration: POST /auth/login with invalid password returns 401`
- `Integration: protected routes return 401 without auth header`
- `Integration: protected routes return 401 with expired token`
- `Integration: POST /auth/refresh with valid refresh token returns new access token`

#### 3.2 -- Multi-Tenant Organization Context

**What**: Middleware that resolves the current organization context from JWT claims and enforces tenant isolation on all queries.

**Design**:

```typescript
// apps/api/src/middleware/org-context.ts
export interface OrgContext {
  organizationId: string;
  userId: string;
  role: "owner" | "admin" | "member" | "viewer";
  permissions: string[];
}

export const orgContextPlugin: FastifyPluginAsync = async (app) => {
  app.addHook("preHandler", async (request) => {
    const orgId = request.headers["x-organization-id"] as string || request.user.orgId;
    // Verify user is a member of the requested org
    const membership = await db.query.organizationMemberships.findFirst({
      where: and(
        eq(organizationMemberships.organizationId, orgId),
        eq(organizationMemberships.userId, request.user.sub),
      ),
    });
    if (!membership) {
      throw new ForbiddenError("Not a member of this organization");
    }
    request.orgContext = {
      organizationId: orgId,
      userId: request.user.sub,
      role: membership.role,
      permissions: membership.permissions,
    };
  });
};

// All queries must be scoped to the current org
// Example: list APIs for current org
const apis = await db.query.apis.findMany({
  where: eq(apisTable.organizationId, request.orgContext.organizationId),
});
```

**Testing**:
- `Integration: requests without X-Organization-Id use orgId from JWT`
- `Integration: requests with X-Organization-Id for a different org the user belongs to succeed`
- `Integration: requests with X-Organization-Id for an org the user does not belong to return 403`
- `Integration: API queries only return data belonging to the current org`
- `Integration: creating an API in org A is not visible to users in org B`

#### 3.3 -- Policy Engine CRUD and Bindings

**What**: CRUD for policies (security, rate limiting, transformation, caching) and policy bindings that attach policies to targets.

**Design**:

```typescript
// apps/api/src/routes/policies.ts
// POST   /api/v1/policies                -- Create policy
// GET    /api/v1/policies                -- List policies (filters: category, type, status)
// GET    /api/v1/policies/:id            -- Get policy
// PUT    /api/v1/policies/:id            -- Update policy
// DELETE /api/v1/policies/:id            -- Delete policy
// POST   /api/v1/policies/:id/bindings   -- Bind policy to target
// DELETE /api/v1/policies/:id/bindings/:bindingId -- Remove binding
// GET    /api/v1/apis/:id/effective-policies      -- Get all effective policies for an API

interface CreatePolicyRequest {
  name: string;
  policyCategory: PolicyCategory;
  policyType: PolicyType;
  priority?: number;                 // default: 100
  isGlobal?: boolean;                // default: false
  owaspCategory?: string;
  ruleDefinition: Record<string, unknown>;
}

interface CreateBindingRequest {
  targetType: "api" | "api_version" | "consumer" | "api_product" | "gateway";
  targetId: string;
  overrideConfig?: Record<string, unknown>;
}
```

```typescript
// apps/api/src/services/policy-engine.ts
export class PolicyEngine {
  // Compute effective policies for a target by evaluating:
  // 1. Global policies for the org
  // 2. Policies bound directly to the target
  // 3. Policies inherited through bindings on parent entities
  // 4. Override resolution (higher priority wins; target-level overrides org-level)
  async getEffectivePolicies(
    targetType: string,
    targetId: string,
    organizationId: string,
  ): Promise<EffectivePolicy[]>;
}

interface EffectivePolicy {
  policyId: string;
  name: string;
  policyCategory: PolicyCategory;
  policyType: PolicyType;
  priority: number;
  ruleDefinition: Record<string, unknown>;
  source: "global" | "direct" | "inherited";
  bindingId?: string;
}
```

**Testing**:
- `Unit: policy priority resolution selects highest priority when multiple policies of same type exist`
- `Unit: global policies apply to all APIs in the org`
- `Unit: direct binding overrides global policy of same type`
- `Integration: POST /policies creates a rate_limit policy with valid ruleDefinition`
- `Integration: POST /policies/:id/bindings attaches policy to an API`
- `Integration: GET /apis/:id/effective-policies returns merged set of global + direct policies`
- `Integration: deleting a binding removes policy from effective set`
- `Integration: OWASP category is indexed and filterable`

#### 3.4 -- Consumer and API Key Management

**What**: Consumer registration, API key generation/revocation, and credential management.

**Design**:

```typescript
// apps/api/src/routes/consumers.ts
// POST   /api/v1/consumers                     -- Create consumer
// GET    /api/v1/consumers                     -- List consumers
// GET    /api/v1/consumers/:id                 -- Get consumer
// PUT    /api/v1/consumers/:id                 -- Update consumer
// POST   /api/v1/consumers/:id/api-keys        -- Generate API key
// GET    /api/v1/consumers/:id/api-keys        -- List API keys (prefixes only)
// DELETE /api/v1/consumers/:id/api-keys/:keyId -- Revoke API key

interface GenerateApiKeyRequest {
  name: string;
  expiresAt?: string;                // ISO 8601 datetime
}

interface GenerateApiKeyResponse {
  keyId: string;
  key: string;                       // "apim_live_a1b2c3..." -- shown ONCE
  keyPrefix: string;                 // "apim_live_" -- for identification
  name: string;
  expiresAt: string | null;
  createdAt: string;
}

// Key format: apim_{env}_{32 random alphanumeric chars}
// env: "live" or "test"
// Example: apim_live_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6
// Stored as SHA-256 hash; the raw key is returned exactly once on creation.
```

**Testing**:
- `Unit: API key generation produces keys matching format apim_{env}_{32chars}`
- `Unit: API key hash uses SHA-256 and matches when verified`
- `Unit: expired keys are rejected during validation`
- `Integration: POST /consumers/:id/api-keys returns the full key exactly once`
- `Integration: GET /consumers/:id/api-keys returns only prefixes, not full keys`
- `Integration: DELETE /consumers/:id/api-keys/:keyId sets status to "revoked"`
- `Integration: revoked keys fail validation`

---

## Phase 4: API Gateway (Data Plane)

### Purpose

Build the API gateway that proxies requests to upstream services, enforcing authentication, rate limiting, and policies. After this phase, the gateway reads its configuration from the management API, authenticates requests via API keys and JWT, enforces rate limits via Redis, and logs requests for analytics.

### Tasks

#### 4.1 -- Gateway Server and Route Matching

**What**: HTTP/HTTPS server that matches incoming requests to configured routes and proxies them to upstream services.

**Design**:

```typescript
// apps/gateway/src/proxy/router.ts
export class RouteRouter {
  private routes: Map<string, RouteEntry>;  // key: "host:pathPrefix"

  async loadRoutes(gatewayId: string): Promise<void>;
  match(host: string, path: string, method: string): RouteMatch | null;
  reload(): Promise<void>;           // triggered by Redis pub/sub on config change
}

interface RouteMatch {
  route: Route;
  upstreamUrl: string;
  strippedPath: string;              // path with prefix removed if stripPath=true
  params: Record<string, string>;    // path parameters
}

interface RouteEntry {
  routeId: string;
  apiId: string;
  apiVersionId: string;
  pathPrefix: string;
  upstreamUrl: string;
  stripPath: boolean;
  hosts: string[];
  protocols: string[];
  timeoutMs: number;
  retries: number;
  loadBalancing: string;
}
```

```typescript
// apps/gateway/src/proxy/upstream.ts
import { Client } from "undici";

export class UpstreamManager {
  private pools: Map<string, Client>;

  getClient(upstreamUrl: string): Client;
  async proxy(
    request: IncomingRequest,
    match: RouteMatch,
  ): Promise<ProxyResponse>;
}

interface ProxyResponse {
  statusCode: number;
  headers: Record<string, string>;
  body: ReadableStream;
  latencyMs: number;
}
```

**Testing**:
- `Unit: router matches /v2/payments/charges to route with prefix /v2/payments`
- `Unit: router strips path prefix when stripPath=true, preserving /charges`
- `Unit: router returns null for unmatched paths`
- `Unit: router matches correct route when multiple routes share a prefix (longest match wins)`
- `Unit: router filters by host header when hosts are configured`
- `Integration: gateway proxies GET /v2/payments/charges to upstream and returns response`
- `Integration: gateway forwards request headers to upstream`
- `Integration: gateway returns 502 when upstream is unreachable`
- `Integration: gateway respects timeout and returns 504 on timeout`
- `Integration: config reload via Redis pub/sub picks up new routes within 1 second`

#### 4.2 -- Gateway Middleware Pipeline

**What**: Composable middleware pipeline that executes authentication, rate limiting, CORS, logging, and transformation in order.

**Design**:

```typescript
// apps/gateway/src/middleware/pipeline.ts
export type MiddlewareFunction = (
  ctx: RequestContext,
  next: () => Promise<void>,
) => Promise<void>;

export class MiddlewarePipeline {
  private middlewares: MiddlewareFunction[] = [];

  use(middleware: MiddlewareFunction): void;
  async execute(ctx: RequestContext): Promise<void>;
}

interface RequestContext {
  request: IncomingRequest;
  response: OutgoingResponse;
  route: RouteMatch;
  consumer?: Consumer;               // set by auth middleware
  policies: EffectivePolicy[];        // loaded from cache
  metadata: {
    startTime: number;
    correlationId: string;
    gatewayId: string;
  };
}

// Middleware execution order (configurable per route):
// 1. CORS check
// 2. Authentication (JWT / API Key / OAuth2)
// 3. Rate limiting
// 4. Request transformation
// 5. Proxy to upstream
// 6. Response transformation
// 7. Logging (always runs, even on error)
```

**Testing**:
- `Unit: pipeline executes middlewares in order`
- `Unit: pipeline stops execution when a middleware responds (does not call next)`
- `Unit: pipeline executes logging middleware even when upstream errors`
- `Unit: correlation ID is generated and passed through the pipeline`
- `Integration: pipeline applies CORS headers when cors policy is bound`
- `Integration: pipeline rejects unauthenticated request when auth policy is bound`
- `Integration: pipeline rate-limits after threshold, returns 429`

#### 4.3 -- Authentication Middleware (JWT and API Key)

**What**: Gateway middleware that validates API keys and JWT tokens, resolving the consumer identity.

**Design**:

```typescript
// apps/gateway/src/middleware/auth/api-key.ts
export const apiKeyAuth: MiddlewareFunction = async (ctx, next) => {
  const key = ctx.request.headers["x-api-key"]
    || ctx.request.headers["authorization"]?.replace("Bearer ", "");

  if (!key) {
    ctx.response.status(401).json({ error: "Missing API key" });
    return;
  }

  const keyPrefix = key.substring(0, key.indexOf("_", 5) + 1); // "apim_live_"
  const keyHash = sha256(key);

  // Lookup in Redis cache first, then fall back to database
  const consumer = await resolveConsumerByKeyHash(keyHash);
  if (!consumer || consumer.status !== "active") {
    ctx.response.status(401).json({ error: "Invalid or revoked API key" });
    return;
  }

  ctx.consumer = consumer;
  await next();
};

// apps/gateway/src/middleware/auth/jwt.ts
export const jwtAuth: MiddlewareFunction = async (ctx, next) => {
  const token = ctx.request.headers["authorization"]?.replace("Bearer ", "");
  if (!token) {
    ctx.response.status(401).json({ error: "Missing JWT" });
    return;
  }

  // Validate JWT signature, expiry, required claims per RFC 7519
  // Verify issuer matches configured OIDC providers
  // Extract consumer identity from claims
  const payload = await verifyJwt(token);
  ctx.consumer = await resolveConsumerByJwtSub(payload.sub);
  await next();
};
```

**Testing**:
- `Unit: API key auth extracts prefix and hashes key correctly`
- `Unit: API key auth rejects empty key`
- `Unit: API key auth rejects revoked key`
- `Unit: JWT auth validates signature and expiry`
- `Unit: JWT auth rejects expired token with 401`
- `Integration: request with valid API key resolves consumer and proxies`
- `Integration: request with invalid API key returns 401`
- `Integration: request with valid JWT from configured OIDC provider succeeds`
- `Integration: consumer identity is available in request logging`

#### 4.4 -- Rate Limiting Middleware (Redis-Backed Sliding Window)

**What**: Rate limiting using Redis sliding window counters, supporting per-consumer, per-IP, per-API-key, and global scopes, with token-aware limits for AI workloads.

**Design**:

```typescript
// apps/gateway/src/middleware/rate-limit.ts
import Redis from "ioredis";

export class RateLimiter {
  constructor(private redis: Redis) {}

  async checkLimit(
    key: string,                     // e.g., "rl:consumer:{id}:api:{id}"
    limit: number,
    windowSeconds: number,
    cost: number = 1,                // 1 for request-based; token count for token-based
  ): Promise<RateLimitResult>;
}

interface RateLimitResult {
  allowed: boolean;
  remaining: number;
  limit: number;
  resetAt: number;                   // Unix timestamp
  retryAfter?: number;               // seconds until a request would be allowed
}

// Redis key scheme:
// rl:{scope}:{scopeId}:{windowStart}
// Uses sorted set with timestamps for sliding window algorithm
// Lua script for atomic check-and-increment

export const rateLimitMiddleware: MiddlewareFunction = async (ctx, next) => {
  const ratePolicies = ctx.policies.filter(p => p.policyCategory === "rate_limit");
  for (const policy of ratePolicies) {
    const rule = policy.ruleDefinition as RateLimitRule;
    const key = buildRateLimitKey(rule.scope, ctx);
    const cost = rule.limitType === "tokens_per_minute"
      ? estimateTokenCount(ctx.request)  // pre-estimate for request; adjust on response
      : 1;
    const result = await rateLimiter.checkLimit(key, rule.limitValue, rule.windowSeconds, cost);

    // Set rate limit headers per draft-ietf-httpapi-ratelimit-headers
    ctx.response.setHeader("X-RateLimit-Limit", result.limit);
    ctx.response.setHeader("X-RateLimit-Remaining", result.remaining);
    ctx.response.setHeader("X-RateLimit-Reset", result.resetAt);

    if (!result.allowed) {
      ctx.response.setHeader("Retry-After", result.retryAfter);
      ctx.response.status(429).json({
        error: "Rate limit exceeded",
        retryAfter: result.retryAfter,
      });
      return;
    }
  }
  await next();
};
```

**Testing**:
- `Unit: sliding window counter increments correctly in Redis`
- `Unit: sliding window counter expires old entries outside the window`
- `Unit: rate limit with per_consumer scope uses consumer ID in key`
- `Unit: rate limit with per_ip scope uses client IP in key`
- `Unit: token-aware rate limit uses estimated token count as cost`
- `Integration: 10 requests within a 10 req/s limit all succeed`
- `Integration: 11th request within a 10 req/s limit returns 429`
- `Integration: rate limit headers are present on every response`
- `Integration: rate limit resets after window expires and requests succeed again`
- `Integration: multiple rate limit policies are evaluated (most restrictive wins)`

#### 4.5 -- Request Logging and Telemetry

**What**: Log every proxied request to PostgreSQL (request_logs table) and emit OpenTelemetry spans.

**Design**:

```typescript
// apps/gateway/src/middleware/logging.ts
export const requestLoggingMiddleware: MiddlewareFunction = async (ctx, next) => {
  const startTime = performance.now();
  try {
    await next();
  } finally {
    const latencyMs = Math.round(performance.now() - startTime);
    // Async write to avoid blocking the response
    enqueueLogEntry({
      apiId: ctx.route.apiId,
      apiVersionId: ctx.route.apiVersionId,
      consumerId: ctx.consumer?.id,
      gatewayId: ctx.metadata.gatewayId,
      requestMethod: ctx.request.method,
      requestPath: ctx.request.url,
      responseStatus: ctx.response.statusCode,
      latencyMs,
      tokenCount: ctx.metadata.tokenCount,
      clientIp: ctx.request.ip,
      correlationId: ctx.metadata.correlationId,
      requestMeta: {
        userAgent: ctx.request.headers["user-agent"],
        contentType: ctx.request.headers["content-type"],
        requestSizeBytes: ctx.request.contentLength,
        responseSizeBytes: ctx.response.contentLength,
        cacheHit: ctx.metadata.cacheHit ?? false,
        authMethod: ctx.consumer ? "api_key" : "none",
      },
      requestedAt: new Date(),
    });
  }
};

// Batch insert to PostgreSQL via BullMQ to avoid per-request DB writes
// Flushes every 1000 entries or every 5 seconds, whichever comes first
```

```typescript
// apps/gateway/src/telemetry/otel.ts
import { NodeSDK } from "@opentelemetry/sdk-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";

export function initTelemetry(config: TelemetryConfig): void;
// Creates spans for each request with attributes:
// http.method, http.url, http.status_code, http.latency_ms
// apim.api_id, apim.consumer_id, apim.gateway_id
```

**Testing**:
- `Integration: proxied request creates a request_logs entry with correct fields`
- `Integration: batch insert flushes 1000 entries in a single insert`
- `Integration: request_logs entry includes latency, status code, consumer ID`
- `Integration: OTel spans are emitted with correct attributes`
- `Unit: token count is extracted from AI API response headers (e.g., x-usage-tokens)`
- `Unit: log entries include correlation ID for distributed tracing`

---

## Phase 5: Developer Portal and API Products

### Purpose

Build the self-service developer portal (Next.js) where consumers discover APIs, read documentation, test endpoints interactively, manage subscriptions, and view their API keys. Also implement API products (bundles of API versions with pricing plans). After this phase, the platform has a complete developer experience.

### Tasks

#### 5.1 -- API Product and Subscription Plan Management

**What**: CRUD for API products (bundles of API versions with pricing plans) and subscription plans.

**Design**:

```typescript
// apps/api/src/routes/products.ts
// POST   /api/v1/products                       -- Create API product
// GET    /api/v1/products                       -- List products
// GET    /api/v1/products/:id                   -- Get product
// PUT    /api/v1/products/:id                   -- Update product
// POST   /api/v1/products/:id/subscribe         -- Subscribe consumer to product

interface CreateProductRequest {
  name: string;
  description?: string;
  visibility: "public" | "internal" | "private";
  approvalType: "auto" | "manual";
  includedApiVersionIds: string[];
  plans: PricingPlan[];
}

interface PricingPlan {
  planId?: string;                   // auto-generated if not provided
  name: string;
  pricingModel: "free" | "flat_rate" | "usage_based" | "tiered" | "freemium";
  basePrice?: { amountCents: number; currency: string; period: "monthly" | "annual" };
  limits: {
    requestsPerMonth?: number;
    tokensPerMonth?: number;
    bytesPerMonth?: number;
  };
  overage?: {
    per1000RequestsCents?: number;
    per1000TokensCents?: number;
  };
}

interface SubscribeRequest {
  consumerId: string;
  planId: string;
}
```

**Testing**:
- `Integration: POST /products creates product with two API versions and two plans`
- `Integration: POST /products/:id/subscribe with auto-approval activates subscription immediately`
- `Integration: POST /products/:id/subscribe with manual-approval sets status to "pending"`
- `Integration: subscribing to a product grants the consumer access to all bundled API versions`
- `Integration: listing products respects visibility (public products visible to all, internal to org, private to admins)`

#### 5.2 -- Developer Portal: API Catalog and Documentation

**What**: Next.js pages for browsing the API catalog, viewing API documentation rendered from OpenAPI specs, and filtering by type/tags.

**Design**:

```typescript
// apps/portal/src/app/apis/page.tsx
// Server component that fetches API catalog from management API
// Displays filterable grid of API cards with:
// - API name, type badge (REST/GraphQL/gRPC/etc.), version
// - Description snippet
// - Status badge (published/deprecated)
// - Tag chips

// apps/portal/src/app/apis/[slug]/page.tsx
// API detail page with:
// - Overview (description, owner team, tags, labels)
// - Version selector dropdown
// - Rendered API documentation (from OpenAPI/AsyncAPI spec)
// - Endpoint listing with method badges and paths
// - Schema viewer for request/response bodies
// - Authentication requirements
// - Rate limit information
// - Code examples (curl, JavaScript, Python)

interface ApiCatalogPageProps {
  searchParams: {
    type?: ApiType;
    tags?: string;
    search?: string;
    page?: string;
  };
}

// Use @stoplight/elements or redocly for OpenAPI spec rendering
// Use @asyncapi/react-component for AsyncAPI spec rendering
```

**Testing**:
- `E2E (Playwright): API catalog page loads and displays published APIs`
- `E2E: filtering by type "graphql" shows only GraphQL APIs`
- `E2E: searching for "payment" shows matching APIs`
- `E2E: clicking an API card navigates to the detail page`
- `E2E: API detail page renders OpenAPI documentation with endpoints`
- `E2E: version selector switches displayed documentation`
- `E2E: deprecated APIs show deprecation notice`

#### 5.3 -- Developer Portal: Interactive API Testing ("Try It")

**What**: Interactive API testing console where developers can send requests to published APIs directly from the portal.

**Design**:

```typescript
// apps/portal/src/app/apis/[slug]/try-it/page.tsx
// Client component with:
// - Endpoint selector (dropdown of available operations)
// - Request builder:
//   - Path parameters (auto-populated from spec)
//   - Query parameters (with type hints from spec)
//   - Headers (with auth header pre-filled from user's API key)
//   - Request body (JSON editor with schema validation)
// - Send button
// - Response viewer:
//   - Status code with color coding
//   - Response headers
//   - Response body (formatted JSON)
//   - Latency display
//   - Rate limit header display

interface TryItRequest {
  endpoint: string;
  method: string;
  pathParams: Record<string, string>;
  queryParams: Record<string, string>;
  headers: Record<string, string>;
  body?: string;
}

// Requests are proxied through the management API to avoid CORS issues:
// POST /api/v1/proxy/try-it
// The management API forwards to the gateway with the user's API key
```

**Testing**:
- `E2E: Try It page loads with endpoint dropdown populated from spec`
- `E2E: selecting GET /pets shows path parameter inputs`
- `E2E: sending a request displays response with status code and body`
- `E2E: request body editor validates JSON against the schema`
- `E2E: auth header is pre-filled with the user's API key`

#### 5.4 -- Developer Portal: Consumer Dashboard

**What**: Consumer dashboard showing subscriptions, API keys, and usage metrics.

**Design**:

```typescript
// apps/portal/src/app/dashboard/page.tsx
// - Active subscriptions list with product name, plan, status, usage bar
// - API keys list with prefix, name, status, last used date
// - Quick stats: total requests today, errors, avg latency

// apps/portal/src/app/dashboard/subscriptions/page.tsx
// - Detailed subscription list
// - Usage breakdown per subscription
// - Upgrade/downgrade plan

// apps/portal/src/app/dashboard/keys/page.tsx
// - API key management (create, revoke)
// - Copy-to-clipboard for new keys

// apps/portal/src/app/dashboard/usage/page.tsx
// - Usage graphs (requests over time, error rate, latency percentiles)
// - Per-API breakdown
// - Token usage (for AI APIs)
```

**Testing**:
- `E2E: dashboard shows active subscription count`
- `E2E: creating a new API key shows the full key with copy button`
- `E2E: revoking an API key updates the status to "revoked"`
- `E2E: usage page displays request count chart for the last 7 days`

---

## Phase 6: Analytics, Usage Metering, and Billing

### Purpose

Implement analytics aggregation from request logs, usage metering for billing, and invoice generation. After this phase, API providers can see usage analytics and consumers receive accurate usage-based invoices.

### Tasks

#### 6.1 -- Analytics Aggregation Pipeline

**What**: Background jobs that aggregate request_logs into hourly and daily metrics using TimescaleDB continuous aggregates.

**Design**:

```sql
-- TimescaleDB continuous aggregate for hourly API metrics
CREATE MATERIALIZED VIEW api_metrics_hourly
WITH (timescaledb.continuous) AS
SELECT
  api_id,
  api_version_id,
  time_bucket('1 hour', requested_at) AS hour,
  count(*) AS request_count,
  count(*) FILTER (WHERE response_status >= 400) AS error_count,
  avg(latency_ms) AS avg_latency_ms,
  percentile_cont(0.50) WITHIN GROUP (ORDER BY latency_ms) AS p50_latency_ms,
  percentile_cont(0.95) WITHIN GROUP (ORDER BY latency_ms) AS p95_latency_ms,
  percentile_cont(0.99) WITHIN GROUP (ORDER BY latency_ms) AS p99_latency_ms,
  sum(token_count) AS total_tokens,
  count(DISTINCT consumer_id) AS unique_consumers
FROM request_logs
GROUP BY api_id, api_version_id, time_bucket('1 hour', requested_at);

-- Consumer usage daily for billing
CREATE MATERIALIZED VIEW consumer_usage_daily
WITH (timescaledb.continuous) AS
SELECT
  consumer_id,
  api_id,
  time_bucket('1 day', requested_at) AS day,
  count(*) AS request_count,
  sum(token_count) AS token_count,
  count(*) FILTER (WHERE response_status >= 400) AS error_count
FROM request_logs
GROUP BY consumer_id, api_id, time_bucket('1 day', requested_at);
```

```typescript
// apps/api/src/routes/analytics.ts
// GET /api/v1/analytics/apis/:id/metrics
//   ?period=24h|7d|30d|90d
//   &granularity=hour|day
// GET /api/v1/analytics/apis/:id/consumers
//   ?period=7d&sortBy=requests|errors|latency
// GET /api/v1/analytics/consumers/:id/usage
//   ?period=current_billing_period

interface ApiMetricsResponse {
  apiId: string;
  period: { start: Date; end: Date };
  granularity: "hour" | "day";
  dataPoints: MetricDataPoint[];
  summary: {
    totalRequests: number;
    totalErrors: number;
    errorRate: number;
    avgLatencyMs: number;
    p95LatencyMs: number;
    p99LatencyMs: number;
    uniqueConsumers: number;
    totalTokens: number;
  };
}

interface MetricDataPoint {
  timestamp: Date;
  requestCount: number;
  errorCount: number;
  avgLatencyMs: number;
  p95LatencyMs: number;
  totalTokens: number;
}
```

**Testing**:
- `Integration: continuous aggregate materializes hourly data from request_logs`
- `Integration: GET /analytics/apis/:id/metrics?period=24h returns 24 hourly data points`
- `Integration: summary includes correct totals computed from data points`
- `Integration: consumer usage daily correctly aggregates per-consumer per-API`
- `Unit: error rate calculation handles zero-request periods without division by zero`

#### 6.2 -- Usage Metering and Billing

**What**: Compute usage against subscription plan limits, generate invoices for usage-based plans.

**Design**:

```typescript
// apps/api/src/services/billing.ts
export class BillingService {
  // Compute current usage for a subscription against its plan limits
  async getSubscriptionUsage(subscriptionId: string): Promise<UsageSummary>;

  // Generate invoice for a billing period
  async generateInvoice(
    consumerId: string,
    periodStart: Date,
    periodEnd: Date,
  ): Promise<Invoice>;

  // Check if consumer is approaching/exceeding quota
  async checkQuotaStatus(consumerId: string): Promise<QuotaStatus[]>;
}

interface UsageSummary {
  subscriptionId: string;
  planName: string;
  pricingModel: string;
  period: { start: Date; end: Date };
  usage: {
    requests: { used: number; limit: number | null; percentage: number };
    tokens: { used: number; limit: number | null; percentage: number };
    bytes: { used: number; limit: number | null; percentage: number };
  };
  estimatedCostCents: number;
}

interface Invoice {
  id: string;
  consumerId: string;
  periodStart: Date;
  periodEnd: Date;
  lineItems: InvoiceLineItem[];
  subtotalCents: number;
  taxCents: number;
  totalCents: number;
  currency: string;
  status: "draft" | "issued" | "paid" | "overdue" | "void";
}

interface InvoiceLineItem {
  description: string;
  quantity: number;
  unitPriceCents: number;
  totalCents: number;
}
```

**Testing**:
- `Unit: usage calculation sums requests across all APIs in a product for a billing period`
- `Unit: overage billing computes correct cost for usage exceeding plan limits`
- `Unit: free plan shows zero cost regardless of usage`
- `Unit: tiered pricing applies correct tier rates`
- `Integration: POST /billing/invoices/generate creates invoice with correct line items`
- `Integration: quota status returns "warning" at 80% usage, "exceeded" at 100%`

#### 6.3 -- Admin Analytics Dashboard

**What**: Admin dashboard pages showing platform-wide analytics, per-API metrics, and consumer usage.

**Design**:

```typescript
// apps/portal/src/app/admin/analytics/page.tsx
// Platform overview:
// - Total requests (24h/7d/30d)
// - Error rate trend
// - Top 10 APIs by request volume
// - Top 10 consumers by usage
// - Latency percentile trends

// apps/portal/src/app/admin/analytics/apis/[id]/page.tsx
// Per-API analytics:
// - Request volume over time (line chart)
// - Error rate over time
// - Latency distribution (histogram)
// - Top consumers
// - Endpoint breakdown
// - Token usage (for AI APIs)

// Chart library: recharts (React-native, composable)
```

**Testing**:
- `E2E: admin analytics page loads with request count charts`
- `E2E: date range selector updates chart data`
- `E2E: top APIs table sorts by request volume`
- `E2E: per-API detail page shows endpoint breakdown`

---

## Phase 7: Governance and Specification Compliance

### Purpose

Implement governance rulesets (Spectral-compatible) that validate API specifications against organizational standards, report violations, and optionally block publishing non-compliant APIs. After this phase, organizations can enforce naming conventions, security requirements, and documentation standards across all their APIs.

### Tasks

#### 7.1 -- Governance Ruleset Management

**What**: CRUD for governance rulesets and rule evaluation against API specifications.

**Design**:

```typescript
// apps/api/src/routes/governance.ts
// POST   /api/v1/governance/rulesets           -- Create ruleset
// GET    /api/v1/governance/rulesets           -- List rulesets
// PUT    /api/v1/governance/rulesets/:id       -- Update ruleset
// POST   /api/v1/governance/evaluate/:apiVersionId -- Evaluate spec against active rulesets
// GET    /api/v1/governance/violations/:apiVersionId -- Get violations for a version

interface CreateRulesetRequest {
  name: string;
  description?: string;
  rules: SpectralRuleset;            // Spectral-compatible rule definitions
  isActive?: boolean;
}

// Spectral-compatible rule format
interface SpectralRuleset {
  rules: Record<string, SpectralRule>;
}

interface SpectralRule {
  severity: "info" | "warning" | "error";
  message: string;
  given: string;                     // JSONPath expression
  then: {
    field?: string;
    function: "truthy" | "falsy" | "pattern" | "enumeration" | "length" | "schema";
    functionOptions?: Record<string, unknown>;
  };
}
```

```typescript
// apps/api/src/services/governance.ts
export class GovernanceService {
  async evaluateSpec(
    apiVersionId: string,
    rulesetIds?: string[],           // evaluate specific rulesets; all active if omitted
  ): Promise<GovernanceReport>;

  async getComplianceStatus(
    organizationId: string,
  ): Promise<OrgComplianceStatus>;
}

interface GovernanceReport {
  apiVersionId: string;
  evaluatedAt: Date;
  rulesets: RulesetResult[];
  summary: {
    totalViolations: number;
    errors: number;
    warnings: number;
    infos: number;
    passed: boolean;                 // true if no errors
  };
}

interface RulesetResult {
  rulesetId: string;
  rulesetName: string;
  violations: Violation[];
}

interface Violation {
  ruleName: string;
  severity: "info" | "warning" | "error";
  message: string;
  path: string;                      // JSONPath to the violation
  line?: number;                     // line number in spec document
}
```

**Testing**:
- `Unit: Spectral rule "operation-operationId" flags missing operationId as error`
- `Unit: Spectral rule "path-keys-no-trailing-slash" flags trailing slashes as warning`
- `Unit: custom rule "auth-required" flags endpoints without security schemes`
- `Fixture: evaluate openapi-petstore.yaml against default rulesets returns clean report`
- `Fixture: evaluate openapi-invalid.yaml against default rulesets returns violations`
- `Integration: POST /governance/evaluate/:apiVersionId returns report with violations`
- `Integration: publishing an API version with governance errors is blocked when enforcement is enabled`
- `Integration: compliance status aggregates violations across all APIs in the org`

#### 7.2 -- Breaking Change Detection

**What**: Compare two API versions and detect breaking changes, integrated into the publishing workflow.

**Design**:

```typescript
// apps/api/src/services/breaking-change.ts
export class BreakingChangeDetector {
  async compare(
    oldVersionId: string,
    newVersionId: string,
  ): Promise<BreakingChangeReport>;
}

interface BreakingChangeReport {
  oldVersion: string;
  newVersion: string;
  breakingChanges: BreakingChange[];
  nonBreakingChanges: Change[];
  summary: {
    breaking: number;
    nonBreaking: number;
    riskLevel: "none" | "low" | "medium" | "high";
  };
}

interface BreakingChange {
  type: "endpoint_removed" | "parameter_type_changed" | "required_parameter_added"
      | "response_field_removed" | "response_type_changed" | "enum_value_removed";
  path: string;
  description: string;
  affectedConsumers: number;         // count of consumers with active subscriptions
}
```

**Testing**:
- `Unit: removing an endpoint is classified as breaking`
- `Unit: adding an optional query parameter is classified as non-breaking`
- `Unit: changing a field type from string to number is classified as breaking`
- `Unit: adding a new endpoint is classified as non-breaking`
- `Integration: comparing two versions returns correct breaking change count`
- `Integration: publishing a version with breaking changes triggers a warning with affected consumer count`

---

## Phase 8: Gateway Enhancements -- Multi-Protocol and Advanced Routing

### Purpose

Extend the gateway to support WebSocket proxying, gRPC proxying, GraphQL query routing, and MCP server integration. After this phase, the platform handles all major API protocols, not just REST.

### Tasks

#### 8.1 -- WebSocket Proxy Support

**What**: Upgrade the gateway to proxy WebSocket connections through the middleware pipeline.

**Design**:

```typescript
// apps/gateway/src/proxy/protocols/websocket.ts
import { WebSocketServer } from "ws";

export class WebSocketProxy {
  constructor(private router: RouteRouter, private pipeline: MiddlewarePipeline) {}

  handleUpgrade(request: IncomingMessage, socket: Socket, head: Buffer): void {
    // 1. Match route (same router as HTTP)
    // 2. Run auth middleware (validate on upgrade request)
    // 3. Establish upstream WebSocket connection
    // 4. Bidirectionally pipe frames
    // 5. Log connection open/close events
  }
}
```

**Testing**:
- `Integration: WebSocket upgrade request with valid API key succeeds`
- `Integration: WebSocket upgrade without authentication returns 401`
- `Integration: messages are bidirectionally proxied between client and upstream`
- `Integration: WebSocket connection close is logged`
- `Integration: rate limiting applies to WebSocket message count`

#### 8.2 -- gRPC Proxy Support

**What**: Proxy gRPC requests through the gateway with authentication and rate limiting.

**Design**:

```typescript
// apps/gateway/src/proxy/protocols/grpc.ts
import { Server, ServerCredentials } from "@grpc/grpc-js";

export class GrpcProxy {
  // gRPC proxy uses HTTP/2 transparent proxying
  // Intercepts metadata (headers) for authentication
  // Routes based on service name + method
  // Rate limits based on call count or estimated tokens

  async proxy(call: ServerUnaryCall, callback: sendUnaryData): Promise<void>;
}
```

**Testing**:
- `Integration: gRPC unary call is proxied to upstream`
- `Integration: gRPC metadata includes API key for authentication`
- `Integration: gRPC server streaming is proxied correctly`
- `Integration: rate limiting applies to gRPC call count`

#### 8.3 -- MCP Server Gateway Integration

**What**: Register MCP servers as API products and proxy tool invocations through the gateway with governance policies.

**Design**:

```typescript
// MCP servers are registered as APIs with api_type = "mcp"
// Their protocol_config contains the MCP manifest:
// {
//   "server_name": "payment-tools",
//   "transport": "streamable_http",
//   "capabilities": ["tools", "resources"],
//   "tools": [
//     { "name": "create_charge", "input_schema": {...}, "is_destructive": true }
//   ]
// }

// The gateway proxies MCP tool invocations:
// POST /mcp/{serverSlug}/tools/{toolName}
// with request body matching the tool's input_schema

// Governance policies can restrict:
// - Which consumers (AI agents) can invoke which tools
// - Token rate limits per agent per tool
// - Destructive tool invocations require explicit approval
```

**Testing**:
- `Integration: MCP server registration stores tools in protocol_config`
- `Integration: tool invocation is proxied to the MCP server with correct transport`
- `Integration: consumer without tool permission receives 403`
- `Integration: destructive tool invocations are blocked without approval policy`
- `Integration: token metering tracks MCP tool invocation costs`

---

## Phase 9: Dependency Graph and Impact Analysis

### Purpose

Implement the graph layer from data model suggestion 4 for API dependency tracking, blast-radius analysis, and policy inheritance visualization. After this phase, platform operators can understand API interdependencies and assess the impact of changes or outages.

### Tasks

#### 9.1 -- Graph Node and Edge Management

**What**: Automatically create and maintain graph nodes/edges when APIs, consumers, and subscriptions are created or modified.

**Design**:

```typescript
// apps/api/src/services/graph.ts
export class GraphService {
  // Called by hooks on entity creation/update/deletion
  async syncEntityToGraph(
    entityType: GraphEntityType,
    entityId: string,
    label: string,
    properties: Record<string, unknown>,
  ): Promise<void>;

  async addEdge(
    sourceType: GraphEntityType, sourceId: string,
    targetType: GraphEntityType, targetId: string,
    edgeType: GraphEdgeType,
    properties?: Record<string, unknown>,
  ): Promise<void>;

  async removeEdge(edgeId: string): Promise<void>;

  // Dependency analysis
  async getBlastRadius(apiId: string, maxDepth?: number): Promise<ImpactGraph>;
  async getDependencies(apiId: string, direction: "upstream" | "downstream"): Promise<GraphNode[]>;

  // Policy inheritance
  async getEffectivePoliciesViaGraph(apiId: string): Promise<EffectivePolicy[]>;
}

interface ImpactGraph {
  rootNode: GraphNode;
  affectedApis: Array<{ api: GraphNode; depth: number; path: GraphNode[] }>;
  affectedConsumers: Array<{ consumer: GraphNode; throughApi: GraphNode }>;
  totalAffectedApis: number;
  totalAffectedConsumers: number;
}
```

**Testing**:
- `Integration: creating an API automatically creates a graph node`
- `Integration: creating a subscription creates a SUBSCRIBES_TO edge`
- `Integration: deleting an API removes its graph node and all connected edges`
- `Integration: blast radius for API A (depended on by B and C) returns both B and C`
- `Integration: circular dependency detection prevents infinite traversal`
- `Integration: graph edges include weight for weighted analysis`

#### 9.2 -- Dependency Visualization API

**What**: API endpoints that return graph data for visualization in the admin dashboard.

**Design**:

```typescript
// apps/api/src/routes/graph.ts
// GET /api/v1/graph/apis/:id/dependencies       -- upstream/downstream dependencies
// GET /api/v1/graph/apis/:id/blast-radius       -- impact analysis
// GET /api/v1/graph/topology                    -- full service topology
// GET /api/v1/graph/consumers/:id/access-paths  -- MCP agent access analysis

interface DependencyGraphResponse {
  nodes: Array<{ id: string; type: string; label: string; properties: Record<string, unknown> }>;
  edges: Array<{ source: string; target: string; type: string; weight: number }>;
}
```

**Testing**:
- `Integration: GET /graph/apis/:id/dependencies returns correct upstream/downstream nodes`
- `Integration: GET /graph/apis/:id/blast-radius returns affected consumers count`
- `Integration: GET /graph/topology returns all APIs and their connections`
- `Integration: graph response format is compatible with D3.js force-directed layout`

---

## Phase 10: Shadow API Discovery and AI Features

### Purpose

Implement ML-powered shadow API discovery (analyzing gateway traffic to find undocumented endpoints) and AI-assisted API design validation. These are the differentiating AI-native features identified in the research.

### Tasks

#### 10.1 -- Shadow API Discovery Pipeline

**What**: Background job that analyzes request_logs to discover undocumented endpoints and generates draft OpenAPI spec fragments.

**Design**:

```typescript
// apps/api/src/jobs/shadow-discovery.ts
export class ShadowApiDiscoveryJob {
  // Runs periodically (every 6 hours) or on-demand
  async execute(gatewayId: string): Promise<DiscoveryResult>;
}

interface DiscoveryResult {
  gatewayId: string;
  analyzedRequests: number;
  discoveredEndpoints: DiscoveredEndpoint[];
  newShadows: number;
  updatedShadows: number;
}

interface DiscoveredEndpoint {
  observedMethod: string;
  observedPath: string;              // parameterized: /users/{id} not /users/123
  requestCount: number;
  firstSeenAt: Date;
  lastSeenAt: Date;
  matchedApiId: string | null;       // null if shadow
  isShadow: boolean;
  confidence: number;                // 0.0 - 1.0
  analysis: {
    pathParameterization: string;    // pattern used to parameterize IDs
    responseCodeDistribution: Record<number, number>;
    avgResponseSizeBytes: number;
    generatedOpenApiFragment: object;
  };
}

// Algorithm:
// 1. Query request_logs for paths not matching any registered API endpoints
// 2. Cluster similar paths (e.g., /users/123, /users/456 -> /users/{id})
// 3. For each cluster, generate a draft OpenAPI path item
// 4. Score confidence based on request volume, consistency, and response patterns
// 5. Insert/update discovered_endpoints table
```

**Testing**:
- `Unit: path parameterization converts /users/123 to /users/{id}`
- `Unit: path parameterization converts /users/abc-def-123 to /users/{id} (UUID pattern)`
- `Unit: clustering groups /orders/1/items/2 and /orders/3/items/4 together`
- `Unit: confidence scoring weights request volume and consistency`
- `Integration: discovery job finds undocumented endpoint from request_logs`
- `Integration: discovered endpoint generates valid OpenAPI fragment`
- `Integration: re-running discovery updates existing shadows rather than creating duplicates`

#### 10.2 -- AI-Assisted Spec Validation and Suggestions

**What**: LLM-powered assistant that reviews API specifications and suggests improvements to field names, error codes, and documentation.

**Design**:

```typescript
// apps/api/src/services/ai-assistant.ts
export class AiSpecAssistant {
  async reviewSpec(
    specContent: string,
    specFormat: SpecFormat,
  ): Promise<AiReviewResult>;
}

interface AiReviewResult {
  suggestions: AiSuggestion[];
  overallScore: number;              // 0-100
  categories: {
    naming: number;                  // naming consistency score
    documentation: number;           // documentation completeness
    security: number;                // security best practices
    consistency: number;             // internal consistency
  };
}

interface AiSuggestion {
  category: "naming" | "documentation" | "security" | "consistency" | "best_practice";
  severity: "info" | "warning" | "error";
  path: string;                      // JSONPath to the relevant spec element
  message: string;
  suggestedFix?: string;
  standard?: string;                 // which standard this relates to (e.g., "RFC 9110", "OWASP A01")
}

// LLM prompt template:
// System: You are an API design expert reviewing an OpenAPI specification.
//   Evaluate against: RFC 9110 HTTP semantics, OWASP API Security Top 10 2023,
//   JSON:API naming conventions, and OpenAPI 3.2 best practices.
// User: Review the following OpenAPI specification and provide suggestions...
```

**Testing**:
- `Integration (mocked LLM): spec review returns naming suggestions for inconsistent casing`
- `Integration (mocked LLM): spec review flags missing authentication security scheme`
- `Integration (mocked LLM): spec review suggests improving sparse descriptions`
- `Unit: suggestion paths are valid JSONPath expressions`

#### 10.3 -- Anomaly Detection for API Abuse

**What**: ML-powered detection of suspicious API usage patterns (credential stuffing, data scraping, injection attempts).

**Design**:

```typescript
// apps/api/src/services/anomaly-detection.ts
export class AnomalyDetectionService {
  // Runs periodically, analyzing recent request_logs
  async detectAnomalies(
    lookbackMinutes: number,
  ): Promise<AnomalyEvent[]>;
}

interface AnomalyEvent {
  apiId: string;
  consumerId: string | null;
  anomalyType: "credential_stuffing" | "data_scraping" | "injection" | "rate_spike" | "latency_spike";
  severity: "low" | "medium" | "high" | "critical";
  confidence: number;
  description: string;
  evidence: {
    sampleRequests: number;
    timeWindowMinutes: number;
    sourceIps: string[];
    pattern: string;
  };
  recommendedAction: "monitor" | "rate_limit" | "block_consumer" | "block_ip";
}
```

**Testing**:
- `Unit: sequential ID enumeration pattern (GET /users/1, /users/2, ...) triggers BOLA detection`
- `Unit: high error rate from a single consumer triggers credential_stuffing detection`
- `Unit: abnormal request volume spike triggers rate_spike detection`
- `Integration: detected anomalies are inserted into anomaly_events table`
- `Integration: critical anomalies trigger webhook notification`

---

## Phase 11: Event-Driven API Support

### Purpose

Extend the platform to manage event-driven and asynchronous APIs: Kafka topic management, AsyncAPI spec support, WebSocket channel documentation, and protocol mediation (expose Kafka as REST). This aligns with Gravitee's differentiating feature set.

### Tasks

#### 11.1 -- AsyncAPI Spec Management

**What**: Full lifecycle support for AsyncAPI 3.1 specifications: import, validation, channel extraction, and documentation rendering.

**Design**:

```typescript
// Protocol config for Kafka/AsyncAPI:
// {
//   "channels": {
//     "payment.events": {
//       "protocol": "kafka",
//       "binding": { "groupId": "...", "partitions": 12 },
//       "publish": { "message_schema": {...} },
//       "subscribe": { "message_schema": {...} }
//     }
//   }
// }

// apps/api/src/services/asyncapi-handler.ts
export class AsyncApiHandler {
  async parseAndStore(specContent: string, apiVersionId: string): Promise<void>;
  async extractChannels(specContent: string): Promise<AsyncChannel[]>;
  async validateBindings(channels: AsyncChannel[]): Promise<ValidationResult>;
}

interface AsyncChannel {
  name: string;
  protocol: "kafka" | "mqtt" | "websocket" | "amqp" | "sse";
  binding: Record<string, unknown>;
  publishSchema?: object;
  subscribeSchema?: object;
}
```

**Testing**:
- `Fixture: asyncapi-kafka.yaml is parsed and channels are extracted`
- `Unit: Kafka binding validation checks required fields (groupId, topic)`
- `Integration: creating an API version with AsyncAPI spec stores channels in protocol_config`
- `Integration: developer portal renders AsyncAPI documentation`

#### 11.2 -- Protocol Mediation (Kafka-to-REST Bridge)

**What**: Expose Kafka topics as REST API endpoints, allowing consumers to publish/subscribe via HTTP.

**Design**:

```typescript
// apps/gateway/src/proxy/protocols/kafka-bridge.ts
// POST /bridge/kafka/{topic}         -- Publish message to Kafka topic
// GET  /bridge/kafka/{topic}/subscribe -- SSE stream of messages from topic

export class KafkaBridge {
  async publishMessage(topic: string, message: unknown): Promise<void>;
  createSubscriptionStream(topic: string, consumerGroup: string): ReadableStream;
}
```

**Testing**:
- `Integration: POST /bridge/kafka/test-topic publishes message to Kafka`
- `Integration: GET /bridge/kafka/test-topic/subscribe returns SSE stream with published messages`
- `Integration: authentication and rate limiting apply to bridge endpoints`

---

## Phase 12: Production Hardening and Deployment

### Purpose

Prepare the platform for production deployment: Kubernetes Helm charts, health checks, graceful shutdown, configuration management, monitoring dashboards, and comprehensive documentation.

### Tasks

#### 12.1 -- Kubernetes Helm Chart

**What**: Helm chart for deploying all platform components to Kubernetes.

**Design**:

```yaml
# helm/api-management-platform/Chart.yaml
apiVersion: v2
name: api-management-platform
version: 0.1.0
dependencies:
  - name: postgresql
    version: "17.x"
    repository: https://charts.bitnami.com/bitnami
  - name: redis
    version: "7.x"
    repository: https://charts.bitnami.com/bitnami

# Values.yaml includes:
# - gateway.replicas (default: 2)
# - api.replicas (default: 2)
# - portal.replicas (default: 2)
# - postgresql.primary.resources
# - redis.master.resources
# - ingress.enabled, ingress.hosts
# - gateway.autoscaling.enabled
```

**Testing**:
- `Unit: helm template renders valid Kubernetes YAML`
- `Unit: helm lint passes without warnings`
- `Integration: helm install to kind cluster starts all pods`
- `Integration: all services pass readiness probes`
- `Integration: gateway handles traffic after deployment`

#### 12.2 -- Health Checks, Graceful Shutdown, and Resilience

**What**: Readiness/liveness probes, graceful shutdown with connection draining, and circuit breaker patterns.

**Design**:

```typescript
// Readiness probe: GET /health/ready
// Returns 200 when:
// - PostgreSQL connection is alive
// - Redis connection is alive
// - Route cache is populated
// Returns 503 otherwise

// Liveness probe: GET /health/live
// Returns 200 if the process is running (lightweight check)

// Graceful shutdown:
// 1. Stop accepting new connections
// 2. Wait for in-flight requests to complete (30s timeout)
// 3. Flush request log buffer to database
// 4. Close database and Redis connections
// 5. Exit

process.on("SIGTERM", async () => {
  await server.close({ timeout: 30000 });
  await flushLogBuffer();
  await db.end();
  await redis.quit();
  process.exit(0);
});
```

**Testing**:
- `Integration: /health/ready returns 503 when database is unreachable`
- `Integration: /health/ready returns 200 when all dependencies are healthy`
- `Integration: SIGTERM triggers graceful shutdown, completing in-flight requests`
- `Integration: log buffer is flushed before shutdown`

#### 12.3 -- OIDC Provider Integration

**What**: Configure external OIDC providers (Azure AD, Google, Okta) for SSO login to the developer portal.

**Design**:

```typescript
// apps/api/src/routes/auth.ts
// GET  /api/v1/auth/oidc/providers              -- List configured providers
// GET  /api/v1/auth/oidc/:providerId/authorize  -- Redirect to OIDC authorize URL
// GET  /api/v1/auth/oidc/:providerId/callback   -- Handle OIDC callback

// OIDC flow:
// 1. User clicks "Sign in with Azure AD"
// 2. Redirect to provider's authorize endpoint with PKCE (per RFC 9700)
// 3. Provider redirects back with authorization code
// 4. Exchange code for tokens
// 5. Validate ID token, extract user info
// 6. Create or update user record
// 7. Issue session JWT
```

**Testing**:
- `Integration (mocked IdP): OIDC authorize redirect includes PKCE challenge`
- `Integration (mocked IdP): callback with valid code creates user and issues JWT`
- `Integration (mocked IdP): callback with invalid code returns 401`
- `Integration (mocked IdP): existing user login updates last_login_at`

---

## Phase Summary and Dependencies

```
Phase 1: Foundation                    ---- required by everything
    |
Phase 2: API Registry & Lifecycle      ---- requires Phase 1
    |
Phase 3: Security & Policy Engine      ---- requires Phase 1, Phase 2
    |
Phase 4: API Gateway                   ---- requires Phase 2, Phase 3
    |
    +---- Phase 5: Developer Portal    ---- requires Phase 2, Phase 3 (can parallel with Phase 6, 7)
    |
    +---- Phase 6: Analytics & Billing ---- requires Phase 4 (can parallel with Phase 5, 7)
    |
    +---- Phase 7: Governance          ---- requires Phase 2 (can parallel with Phase 5, 6)
    |
Phase 8: Multi-Protocol Gateway        ---- requires Phase 4
    |
Phase 9: Dependency Graph              ---- requires Phase 1 (can parallel with Phase 8)
    |
Phase 10: AI Features                  ---- requires Phase 4, Phase 6
    |
Phase 11: Event-Driven APIs            ---- requires Phase 2, Phase 8
    |
Phase 12: Production Hardening         ---- requires all previous phases
```

### Parallelism Opportunities

- **Phases 5, 6, 7** can be developed concurrently after Phase 4 is complete (developer portal, analytics, governance are independent)
- **Phases 8 and 9** can be developed concurrently (multi-protocol gateway and dependency graph are independent)
- **Phases 10 and 11** can be developed concurrently after Phase 8 (AI features and event-driven APIs are independent)

---

## Definition of Done (per phase)

1. All tasks in the phase are implemented
2. All unit tests pass (`pnpm test`)
3. All integration tests pass (`pnpm test:integration`)
4. Biome lint and format checks pass (`pnpm lint`)
5. TypeScript type checking passes (`pnpm typecheck`)
6. Docker build succeeds for affected services (`docker build .`)
7. Feature works end-to-end in docker-compose environment
8. New API endpoints appear in auto-generated OpenAPI spec at `/docs`
9. Database migrations are created and tested (up and down)
10. New environment variables are documented in `.env.example`
11. Audit trail captures all write operations for new entities
12. Request logging covers new gateway features
