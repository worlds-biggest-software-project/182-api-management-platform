# Standards & API Reference

> Project: API Management Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

No domain-specific ISO standards govern API management directly. The following ISO standards are relevant to adjacent concerns:

- **ISO/IEC 27001:2022 — Information Security Management Systems**
  URL: https://www.iso.org/standard/27001
  Relevant because API management platforms handle sensitive authentication tokens, API keys, and consumer data. ISO 27001 certification is increasingly expected for enterprise-grade SaaS platforms. Governs security controls across credential storage, access logging, and incident response.

- **ISO/IEC 29101:2018 — Privacy Architecture Framework**
  URL: https://www.iso.org/standard/45124.html
  Relevant for developer portals and API consumer management where personally identifiable information (PII) is collected. Establishes architectural principles for privacy-by-design in system components handling user registration and authentication.

### W3C & IETF Standards

- **RFC 9110 — HTTP Semantics (replaces RFC 7231)**
  URL: https://httpwg.org/specs/rfc9110.html
  The foundational standard governing HTTP request methods, status codes, headers, and content negotiation. Essential reading for any API gateway implementing HTTP/1.1, HTTP/2, or HTTP/3 proxying and routing. Supersedes RFC 7231 (2014) with a cleaner separation of semantics from transport framing.

- **RFC 6749 — The OAuth 2.0 Authorization Framework**
  URL: https://www.rfc-editor.org/rfc/rfc6749
  Defines the core OAuth 2.0 authorization flows (Authorization Code, Client Credentials, Implicit, ROPC). All API management platforms implement OAuth 2.0 as the primary mechanism for securing access tokens and enforcing scoped authorization policies at the gateway layer.

- **RFC 9700 — Best Current Practice for OAuth 2.0 Security (2025)**
  URL: https://datatracker.ietf.org/doc/rfc9700/
  Published January 2025. Updates the threat model and security advice in RFCs 6749, 6750, and 6819. Mandates PKCE for all browser and native clients, eliminates Implicit Flow and ROPC as recommended patterns, and introduces PAR (Pushed Authorization Requests) and JAR (JWT Authorization Requests) for hardened authorization endpoints. Any API management platform generating or validating OAuth tokens must align with RFC 9700.

- **OpenID Connect Core 1.0 (incorporating errata set 2)**
  URL: https://openid.net/specs/openid-connect-core-1_0.html
  Identity layer built on OAuth 2.0 that defines the ID token format, UserInfo endpoint, and authentication flows. Required for API management platforms offering SSO integration and developer portal authentication through external identity providers.

- **RFC 8288 — Web Linking**
  URL: https://www.rfc-editor.org/rfc/rfc8288
  Defines the `Link` header field and link relation types used extensively in REST API hypermedia patterns (HATEOAS). Referenced by API design guides for pagination, versioning, and resource navigation in developer portals.

- **RFC 8259 — The JavaScript Object Notation (JSON) Data Interchange Format**
  URL: https://www.rfc-editor.org/rfc/rfc8259
  Current IETF Internet Standard (STD 90) for JSON encoding. Every API management platform uses JSON as the primary payload format for management APIs, configuration, and event data. All schema validation and policy engines must conform to RFC 8259 parsing rules.

- **RFC 7519 — JSON Web Token (JWT)**
  URL: https://www.rfc-editor.org/rfc/rfc7519
  Defines the JWT format used for bearer tokens, ID tokens, and signed API authorization claims. API gateways universally implement JWT validation as a core authentication policy, including signature verification, claim inspection, and expiry enforcement.

- **RFC 7517 — JSON Web Key (JWK)**
  URL: https://www.rfc-editor.org/rfc/rfc7517
  Specifies the JSON structure for representing cryptographic keys used to sign and verify JWTs. API management platforms expose JWKS (JSON Web Key Sets) endpoints to allow downstream consumers and gateways to verify token signatures dynamically.

### Data Model & API Specifications

- **OpenAPI Specification (OAS) 3.2.0**
  URL: https://spec.openapis.org/oas/v3.2.0.html
  The de-facto industry standard for describing RESTful APIs in a machine-readable format (YAML or JSON). Version 3.2.0 (September 2025) added structured tag nesting, streaming media type support (SSE, JSON Lines, JSON Sequences), native QUERY HTTP method support, and OAuth 2.0 Device Authorization Flow. All API management platforms use OAS for API import/export, documentation generation, and contract testing. The OpenAPI Arazzo Specification companion defines multi-step API workflow sequences.

- **AsyncAPI Specification 3.1.0**
  URL: https://www.asyncapi.com/docs/reference/specification/v3.1.0
  The counterpart to OpenAPI for event-driven and message-based APIs. Supports Kafka, MQTT, WebSocket, AMQP, STOMP, and other async protocols via "bindings." Increasingly required in API management platforms as event-driven architectures gain parity with REST. AsyncAPI 3.0 introduced improved OpenAPI alignment and stronger support for complex event schemas.

- **GraphQL Specification — September 2025 Edition**
  URL: https://spec.graphql.org/September2025/
  The first major GraphQL specification update since October 2021. Key additions in the 2025 edition: schema coordinates (standardized references to schema elements), expanded deprecation support, descriptions on documents, and full Unicode support. API management platforms must handle GraphQL proxying, schema stitching, and federation as enterprise adoption of GraphQL has reached approximately 61% in production.

- **gRPC Protocol over HTTP/2 (CNCF)**
  URL: https://grpc.io/docs/what-is-grpc/introduction/
  Google-originated RPC framework now governed by the CNCF. Uses HTTP/2 for transport and Protocol Buffers as the interface definition language. Binary encoding produces significantly smaller messages and faster parsing than JSON/REST. API gateways increasingly need to proxy, document, and apply policies to gRPC services in microservices architectures.

- **JSON Schema (Draft 2020-12)**
  URL: https://json-schema.org/specification
  Specifies a JSON-based format to define the structure of JSON data for validation and documentation. Used extensively within OpenAPI 3.1+ for request/response body validation and within API gateway policy engines for payload inspection and schema enforcement.

- **JSON:API v1.1**
  URL: https://jsonapi.org/format/
  A specification for building APIs in JSON that standardises document structure, resource relationships, error formats, and pagination conventions. Widely adopted in developer portal APIs and management plane REST APIs.

### Security & Authentication Standards

- **OWASP API Security Top 10 (2023 Edition)**
  URL: https://owasp.org/API-Security/editions/2023/en/0x11-t10/
  The canonical reference for API-specific vulnerabilities, updated June 2023. The top risks include Broken Object Level Authorization (BOLA, ~40% of API attacks), Broken Authentication, Broken Object Property Level Authorization, Unrestricted Resource Consumption, and Security Misconfiguration. API management platforms are evaluated against this list for built-in threat protection, rate limiting, and policy enforcement capabilities. Supersedes the 2019 edition with additional risks: Unsafe API Consumption, Unrestricted Access to Sensitive Business Flows, and SSRF.

- **NIST SP 800-228 — Guidelines for API Protection for Cloud-Native Systems (June 2025)**
  URL: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-228.pdf
  Published June 2025. The first NIST special publication dedicated to API security. Defines how to secure APIs across the full lifecycle (design, development, deployment, runtime, retirement) using zero-trust principles. Mandates five controls per API call: encryption in transit, service authentication, service authorization, end-user authentication, and end-user authorization. Requires centralised API inventory with ownership, security classification, and runtime metrics. Recommends OpenAPI, gRPC, or GraphQL as Interface Definition Languages for all APIs.

- **OAuth 2.0 + PKCE (RFC 7636)**
  URL: https://www.rfc-editor.org/rfc/rfc7636
  Proof Key for Code Exchange (PKCE) extension, now mandatory for public clients per RFC 9700. API management platforms must enforce PKCE when acting as authorization servers or validating authorization code flows from developer portal client applications.

- **OpenTelemetry (OTel) — CNCF Standard**
  URL: https://opentelemetry.io/
  Vendor-neutral, open-source observability framework for collecting traces, metrics, and logs. The CNCF project has become the industry standard for API gateway observability, enabling end-to-end distributed tracing from client request through API gateway to backend services. Provides language SDKs for Java, Python, Go, Node.js, .NET, and more, plus a Collector component for telemetry export to any monitoring backend.

### MCP Server Specifications

- **Model Context Protocol (MCP) Specification — Version 2025-11-25**
  URL: https://modelcontextprotocol.io/specification/2025-11-25
  Open protocol introduced by Anthropic in November 2024 and donated to the Linux Foundation's Agentic AI Foundation (AAIF) in December 2025. Standardises how AI systems (LLMs and agents) integrate with external data sources and tools. The June 2025 spec release added structured tool outputs, OAuth-based authorization, elicitation for server-initiated user interactions, and improved security best practices. The 2026 roadmap prioritises scalable HTTP transport, enterprise readiness (audit trails, SSO-integrated auth, gateway behaviour), task lifecycle semantics, and protocol extensions for specific industries. Several API management platforms now support MCP servers as consumable API products (Azure APIM September 2025, WSO2 API Manager 2025), representing the convergence of API management with AI agent governance.

---

## Similar Products — Developer Documentation & APIs

### Kong Gateway / Kong Konnect

- **Description:** High-performance open-source API gateway (Apache 2.0) with a 100+ plugin ecosystem and Konnect cloud management plane. Widely used in cloud-native and Kubernetes environments.
- **API Documentation:** https://developer.konghq.com/api/
- **Main Documentation:** https://developer.konghq.com/
- **SDKs/Libraries:** Official plugins available in Lua; community SDKs in Python, Go, Node.js for the Admin API. Kong Terraform provider available.
- **Developer Guide:** https://developer.konghq.com/gateway/
- **Standards:** REST/JSON, OpenAPI 3.x specification import, gRPC, WebSocket proxying
- **Authentication:** OAuth 2.0, JWT, API Key, HMAC, mTLS, LDAP, OIDC (via plugin)

---

### Apigee (Google Cloud)

- **Description:** Enterprise-grade API lifecycle management platform covering gateway, analytics, developer portal, and monetization, operated as a Google Cloud managed service. Recognized as a Gartner Magic Quadrant Leader since 2015.
- **API Documentation:** https://docs.cloud.google.com/apigee/docs/reference/apis/apigee/rest
- **Main Documentation:** https://cloud.google.com/apigee/docs
- **SDKs/Libraries:** Managed via Google Cloud SDK (gcloud CLI); client libraries for Java, Python, Node.js, Go, Ruby, PHP via Google API client libraries.
- **Developer Guide:** https://docs.apigee.com/api-platform/fundamentals/api-development-lifecycle
- **Standards:** REST/JSON, OpenAPI 3.x import/export, GraphQL, gRPC proxying, SOAP/XML
- **Authentication:** OAuth 2.0, SAML 2.0, JWT, API Key, Client Certificate (mTLS)

---

### MuleSoft Anypoint Platform

- **Description:** Unified iPaaS and API management platform owned by Salesforce. Provides API Designer, Flex Gateway, governance rulesets, and Anypoint Exchange for reusable API components.
- **API Documentation:** https://anypoint.mulesoft.com/exchange/portals/anypoint-platform/ (Anypoint Platform API)
- **Main Documentation:** https://docs.mulesoft.com/general/
- **SDKs/Libraries:** Anypoint Studio IDE, Maven plugins, Anypoint CLI; REST APIs for Anypoint Platform management (Platform API, Flex Gateway API).
- **Developer Guide:** https://docs.mulesoft.com/general/api-led-develop
- **Standards:** REST/JSON, OpenAPI 3.x, RAML 1.0, AsyncAPI (Kafka, JMS), GraphQL, gRPC via Flex Gateway
- **Authentication:** OAuth 2.0, SAML 2.0, JWT, Client ID/Secret, LDAP, OIDC via external IdPs

---

### AWS API Gateway

- **Description:** Serverless, fully managed API gateway tightly integrated with AWS Lambda, IAM, and Cognito. Supports REST, HTTP, and WebSocket APIs with pay-per-request pricing.
- **API Documentation:** https://docs.aws.amazon.com/apigateway/latest/developerguide/api-ref.html
- **Main Documentation:** https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html
- **SDKs/Libraries:** AWS SDK for JavaScript, Python (Boto3), Java, Go, .NET, Ruby, PHP. SDK generation from API Gateway console for client-side SDKs (JavaScript, Java, Android, iOS Objective-C, iOS Swift).
- **Developer Guide:** https://docs.aws.amazon.com/apigateway/latest/developerguide/rest-api-develop.html
- **Standards:** REST/JSON, HTTP, WebSocket, OpenAPI 3.x (import/export), CloudFormation/Terraform IaC
- **Authentication:** AWS IAM SigV4, Amazon Cognito User Pools, Lambda Authorizers, API Keys, JWT (HTTP APIs)

---

### Azure API Management

- **Description:** Full API lifecycle management platform from Microsoft Azure with a developer portal, policy engine, self-hosted gateway, and MCP server support. Offers hybrid and multi-cloud deployment capability.
- **API Documentation:** https://learn.microsoft.com/en-us/rest/api/apimanagement/
- **Main Documentation:** https://learn.microsoft.com/en-us/azure/api-management/
- **SDKs/Libraries:** Azure SDK for .NET, Python, Java, JavaScript, Go, Ruby; Azure CLI; Terraform provider.
- **Developer Guide:** https://learn.microsoft.com/en-us/azure/api-management/
- **Standards:** REST/JSON, OpenAPI 3.x, AsyncAPI, GraphQL, gRPC, SOAP/XML, WebSocket, MCP (2025)
- **Authentication:** OAuth 2.0, OpenID Connect, Azure AD/Entra ID, JWT, Client Certificate (mTLS), Subscription Key

---

### Tyk API Gateway

- **Description:** Open-source (Apache 2.0) API gateway written in Go with a no-feature-lockout philosophy. Self-Managed edition includes gateway, dashboard, developer portal, and Tyk Cloud managed offering.
- **API Documentation:** https://tyk.io/docs/ (Tyk Gateway API reference included)
- **Main Documentation:** https://tyk.io/docs/
- **SDKs/Libraries:** Tyk Go SDK; custom middleware plugins in Go, Python, or gRPC-based plugins; Postman collection for Tyk Gateway API v5.0.
- **Developer Guide:** https://tyk.io/docs/getting-started/create-api/
- **Standards:** REST/JSON, GraphQL, GraphQL federation (Apollo Federation v1), gRPC, TCP, OpenAPI 3.x import
- **Authentication:** OAuth 2.0, JWT, OIDC, Bearer Tokens, Basic Auth, HMAC, Client Certificates, Mutual TLS

---

### WSO2 API Manager

- **Description:** 100% open-source (Apache 2.0) full-lifecycle API management platform. Version 4.5.0 (2025) integrates Moesif analytics and adds native support for MCP servers and AI agent management.
- **API Documentation:** https://apim.docs.wso2.com/en/4.5.0/reference/product-apis/overview/
- **Main Documentation:** https://apim.docs.wso2.com/en/latest/
- **SDKs/Libraries:** Management APIs include Publisher API v4, Developer Portal API v3, Admin Portal API v4, Gateway API v2, Service Catalog API v1, DevOps API v0. Java SDK for Mule and custom extension points.
- **Developer Guide:** https://apim.docs.wso2.com/en/4.5.0/get-started/api-manager-quick-start-guide/
- **Standards:** REST/JSON, OpenAPI 3.x, AsyncAPI, GraphQL, Kafka, WebSocket, MCP (2025)
- **Authentication:** OAuth 2.0, SAML 2.0, OIDC, JWT, Basic Auth, API Key, Client Certificate

---

### Gravitee.io API Management

- **Description:** Open-source, event-native API management platform with a reactive execution engine. Supports multi-protocol management including Kafka, MQTT, Solace, WebSocket, gRPC, SSE, and GraphQL via protocol mediation.
- **API Documentation:** https://documentation.gravitee.io/apim/management-api-reference
- **Main Documentation:** https://documentation.gravitee.io/apim
- **SDKs/Libraries:** REST Management API with Personal Access Token (PAT) authentication; interactive API viewer with downloadable OpenAPI specification; Helm charts for Kubernetes deployment.
- **Developer Guide:** https://documentation.gravitee.io/apim/how-to-guides/use-case-tutorials/create-and-publish-an-api-using-the-management-api
- **Standards:** REST/JSON, OpenAPI 3.x, AsyncAPI, GraphQL, gRPC, WebSocket, Kafka, MQTT, Solace, SSE
- **Authentication:** OAuth 2.0, JWT, API Key, Basic Auth, mTLS, Custom Auth via policy plugins

---

### Hasura GraphQL Engine / DDN

- **Description:** GraphQL-native API layer that auto-generates a unified GraphQL API from databases and REST/GraphQL sources. Supports data federation, event triggers, scheduled triggers, and the Hasura Data Delivery Network (DDN).
- **API Documentation:** https://hasura.io/docs/2.0/api-reference/overview/
- **Main Documentation:** https://hasura.io/docs/2.0/index/
- **SDKs/Libraries:** TypeScript, Python, and Go Connector SDKs for custom business logic; GraphQL client SDKs (Apollo Client, urql, etc.); REST API accessible via the Hasura v1/query and v1/metadata endpoints.
- **Developer Guide:** https://hasura.io/docs/2.0/getting-started/how-it-works/index/
- **Standards:** GraphQL (September 2025 spec), Apollo Federation v1, REST via remote joins, OpenAPI for remote schema stitching
- **Authentication:** JWT, Webhook-based auth, Role-based Access Control (RBAC), API Key for admin operations

---

## Notes

**Evolving standards areas to monitor:**

- **OpenAPI Arazzo Specification**: Companion to OAS that describes multi-step API call sequences (workflows). Expected to become a table-stakes requirement for developer portal documentation as AI agents increasingly need to orchestrate sequences of API calls.

- **MCP as an API management surface**: The Model Context Protocol is converging with traditional API management. Platforms governing MCP servers as first-class API products represent an emerging architecture. The 2026 MCP roadmap's focus on enterprise readiness (audit trails, SSO auth, gateway behaviour, `.well-known` capability advertising) directly overlaps with core API management concerns.

- **Token-aware quota standards**: No formal standard yet governs LLM token consumption quotas at the API gateway level. Azure APIM's token-aware rate limiting (2025) is vendor-specific. This gap is likely to produce either a formal extension to OpenAPI or a new IETF RFC as AI API traffic scales.

- **NIST SP 800-228 adoption timeline**: Published as an Initial Public Draft in June 2025 and withdrawn (finalized) by March 2026. Organizations in regulated industries (financial services, healthcare, government) are expected to begin requiring compliance from API management vendors through 2026–2027.
