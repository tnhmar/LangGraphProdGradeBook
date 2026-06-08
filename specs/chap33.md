# Chapter 33 Spec — Integrating with Enterprise Systems

> **Volume 1 reference:** Part 7 — Tooling and Integrations, Chapter 33
> **Part:** 7 — Tooling and Integrations
> **Depends on:** chap32 (`TOOL-ABS-001–003`, `TOOL-CONTRACT-001–005`, `TOOL-EXEC-001–006`, `TOOL-SEC-001–005`)
> **Java + LangGraph implementation target:** Volume 2 — Chapter 9: Enterprise Tool Handlers; Chapter 10: Secrets Management; Chapter 20: Enterprise Governance
> **Bridges to:** chap34 (Advanced Tool Patterns)

> *"The hardest part of connecting to an enterprise system is not the API. It is the decade of assumptions baked into its data model."*
> — Attributed to enterprise integration practitioners

---

## Chapter Purpose

Chapter 32 built the abstraction layer that normalizes every tool call behind a uniform contract. This chapter fills that layer with concrete implementations for the enterprise systems that most production agents actually need to reach: REST and GraphQL APIs, relational databases, message queues, CRM and ERP platforms, identity providers, and object stores.

These systems share a critical characteristic: they were not designed with agents in mind. Their APIs were built for human developers writing stateless request-response clients — not for reasoning loops that compose dozens of calls per task, operate across long sessions, and may be compromised by prompt injection. Integrating them reliably requires adapting their interfaces to fit the tool contract model established in Chapter 32.

By the end of this chapter the reader can:
- Classify enterprise integration surfaces and select the correct pattern per surface type
- Implement production-grade authentication and secrets management for agent tool credentials
- Build safe, reliable handlers for REST/GraphQL APIs, relational databases, message queues, and major SaaS platforms
- Apply data governance controls (PII minimization, data residency, tenant isolation, audit logging) at the tool layer
- Design connection management and graceful degradation strategies for multi-backend agent systems

---

## Section 33.1 — The Enterprise Integration Landscape

### Six Integration Surface Types

Six surface types cover the majority of enterprise agent deployments. Classifying the target system before writing any integration code determines which concerns must be addressed:

| Surface Type | Examples | Primary Integration Concerns |
|---|---|---|
| **REST / GraphQL APIs** | Internal microservices, modern SaaS APIs | Versioning drift, pagination, rate limiting, scope minimization |
| **Relational databases** | PostgreSQL, Oracle, SQL Server, MySQL | Connection pooling, query safety, read/write isolation, least-privilege roles |
| **Message queues / event buses** | Kafka, RabbitMQ, Azure Service Bus, AWS SQS | Idempotency, at-least-once delivery, deduplication, backpressure |
| **SaaS platforms** | Salesforce, SAP, ServiceNow, Microsoft 365, Google Workspace | Platform-specific auth, governor limits, proprietary query languages, approval flows |
| **File / object stores** | S3, Azure Blob, GCS, SharePoint, network file systems | Large object handling, presigned URLs, access control, scan-before-use |
| **Identity / directory systems** | Active Directory, LDAP, Okta, Azure AD | Read-only access, group membership resolution, identity-aware tool scoping |

> ⚠️ **This chapter focuses on the integration layer — how to connect reliably and securely.** Schema design, data modeling, and the business logic of each system are out of scope. The goal is to produce well-behaved tool handlers that hide the operational complexity of each backend from the agent's reasoning loop.

### Design Rules

| Rule ID | Rule |
|---|---|
| `ENT-LAND-001` | Every enterprise integration MUST be classified by surface type before implementation — the surface type determines the canonical set of authentication, failure, lifecycle, and governance concerns to address. |
| `ENT-LAND-002` | Tool handlers MUST hide backend operational complexity from the agent reasoning loop — the agent calls `toolLayer.invoke(name, args, ctx)` and receives a result envelope; it MUST NOT be aware of connection pool state, token refresh cycles, or platform governor limits. |

---

## Section 33.2 — Authentication and Secrets Management

### The Credentials Problem at Scale

An agent integrating with five enterprise systems needs five sets of credentials. A multi-agent system with dozens of agents needs a disciplined approach to credential lifecycle: provisioning, storage, rotation, scoping, and revocation.

Ad hoc credential management — environment variables hardcoded per service, shared API keys passed between agents — is the single most common source of enterprise integration security incidents. **The correct architecture separates credential storage from credential use.** Credentials live in a secrets store; tool handlers retrieve them at call time via a secrets client. No credential appears in source code, configuration files, container images, or log output.

### Authentication Patterns by System Type

| System Type | Primary Pattern | Engineering Considerations |
|---|---|---|
| SaaS REST/GraphQL APIs | OAuth 2.0 client credentials | Token expiry/refresh; scope minimization; per-tenant credential isolation |
| Internal microservices | mTLS or service account JWT | Certificate rotation; service mesh integration; identity propagation across call chains |
| Relational databases | Username/password or IAM role | Connection pool credential refresh; least-privilege DB roles; no shared superuser accounts |
| AWS services | IAM role / instance or pod identity | Role scoping per agent; no static access keys; cross-account role assumption |
| Azure services | Managed Identity / Service Principal | Workload identity federation; RBAC scope minimization; credential-free where possible |
| Google Cloud | Workload Identity Federation | Service account key avoidance; per-service SA; IAM condition-based scope |
| Legacy enterprise | API key or basic auth | Key rotation schedule; IP allowlisting; single-use keys per integration where possible |

### OAuth 2.0 Client Credentials — Token Manager

For SaaS integrations, the OAuth 2.0 client credentials flow is the standard machine-to-machine authentication pattern. The tool handler exchanges a client ID and secret for a bearer token, uses it until expiry, and refreshes transparently.

```java
// Java OAuth2 client credentials token manager
@Component
public class OAuth2TokenManager {
    private final SecretsClient secrets;
    private final HttpClient httpClient;

    // Per-integration token cache — keyed by integration name
    private final ConcurrentHashMap<String, CachedToken> tokenCache =
        new ConcurrentHashMap<>();
    private final ReentrantLock refreshLock = new ReentrantLock();

    public String getToken(String integrationName) throws IOException {
        CachedToken cached = tokenCache.get(integrationName);

        // 60-second buffer before expiry to avoid using a near-expired token
        if (cached != null && cached.isValid(60)) {
            return cached.accessToken();
        }

        // ENT-AUTH-003: single-writer refresh — all waiters receive the same new token
        refreshLock.lock();
        try {
            // Re-check after acquiring lock (another thread may have refreshed)
            cached = tokenCache.get(integrationName);
            if (cached != null && cached.isValid(60)) return cached.accessToken();

            OAuth2Config config = secrets.getOAuth2Config(integrationName);
            TokenResponse response = httpClient.post(config.tokenUrl(),
                Map.of(
                    "grant_type", "client_credentials",
                    "client_id", config.clientId(),
                    "client_secret", config.clientSecret(),
                    "scope", String.join(" ", config.scopes())
                ));

            CachedToken newToken = CachedToken.of(
                response.accessToken(),
                Instant.now().plusSeconds(response.expiresIn())
            );
            tokenCache.put(integrationName, newToken);
            return newToken.accessToken();

        } finally {
            refreshLock.unlock();
        }
    }
}
```

> ⚠️ **Without a lock, multiple threads checking `isValid()` simultaneously during token expiry will each issue a refresh request — a thundering-herd failure that spikes load on the authorization server and may trigger rate limiting.** The lock ensures only one thread refreshes while all others wait and reuse the new token. Never pass the raw client secret from a config file; retrieve it from the secrets store at initialization time.

### Secrets Management Infrastructure

Production secrets management requires a dedicated secrets store. Three dominant options:

| Store | Best For | Key Capability |
|---|---|---|
| **HashiCorp Vault** | On-premise / multi-cloud | Dynamic secrets (on-demand, auto-expiry); AppRole / K8s auth; full audit log |
| **AWS Secrets Manager** | AWS-native deployments | Automatic RDS credential rotation; ECS/EKS pod identity integration |
| **Azure Key Vault** | Azure-native deployments | Managed Identity credential-free access; RBAC scope minimization |

The abstraction layer's secrets client MUST present a uniform interface regardless of the backend — swapping from Vault to AWS Secrets Manager is a configuration change, not a code change:

```java
public interface SecretsClient {
    String getSecret(String path);
    Map<String, String> getSecretMap(String path);
    OAuth2Config getOAuth2Config(String integrationName);
}
```

> 💡 **Use dynamic secrets for database credentials wherever the secrets store supports them.** Vault's database secrets engine generates a unique username and password per connection request, automatically revoked after a configurable TTL. An agent whose database credentials are compromised mid-session exposes only that session, not the shared credential used by all agents.

### Design Rules

| Rule ID | Rule |
|---|---|
| `ENT-AUTH-001` | No credential (API key, client secret, password, certificate private key) MAY appear in source code, configuration files committed to version control, container images, or log output — all credentials MUST be retrieved from a secrets store at runtime. |
| `ENT-AUTH-002` | Tool handlers MUST use the least-privilege credential scope for their declared `sideEffects` classification — read-only tools MUST NOT hold write credentials for the same backend. |
| `ENT-AUTH-003` | OAuth 2.0 token refresh MUST be protected by a single-writer lock — concurrent refresh on token expiry is a thundering-herd defect. |
| `ENT-AUTH-004` | The secrets client MUST present a uniform interface regardless of the secrets store backend — direct SDK calls to Vault, AWS Secrets Manager, or Azure Key Vault MUST NOT appear in tool handler code. |
| `ENT-AUTH-005` | Dynamic secrets (credentials generated on demand and auto-expiring) MUST be used for database integrations wherever the secrets store supports them — shared static credentials are not acceptable for production database tool handlers. |

---

## Section 33.3 — REST and GraphQL API Integration

### REST Adapter Pattern

A REST adapter is a tool handler that calls an HTTP-based backend API. Four concerns must be addressed explicitly in every REST adapter:

**1. Pagination caps**
REST APIs that return lists MUST have a maximum-page-size cap enforced by the tool handler. An agent that iterates until a list is exhausted will overflow its context window and exhaust its token budget on large datasets.

```java
@Component
public class RestApiAdapter {
    private final HttpClient httpClient;
    private final OAuth2TokenManager tokenManager;

    public List<Map<String, Object>> pagedGet(
            String integrationName, String endpoint,
            Map<String, String> params, int maxPages) throws IOException {

        List<Map<String, Object>> results = new ArrayList<>();
        String nextUrl = endpoint;
        int page = 0;

        while (nextUrl != null && page < maxPages) {    // ENT-REST-002: pagination cap
            String token = tokenManager.getToken(integrationName);
            HttpResponse response = httpClient.get(nextUrl, params,
                Map.of("Authorization", "Bearer " + token));

            if (response.statusCode() == 429) {          // ENT-REST-003: 429 handling
                long retryAfterMs = parseRetryAfter(response) * 1000L;
                Thread.sleep(retryAfterMs);
                continue;
            }

            response.assertSuccess();
            PagedResponse body = response.as(PagedResponse.class);
            results.addAll(body.items());
            nextUrl = body.nextPageUrl();
            page++;
        }
        return results;
    }
}
```

**2. Rate limit handling**
On HTTP 429, the adapter MUST honor the `Retry-After` response header rather than applying a fixed backoff interval. Fixed backoff ignores the server's actual recovery window and may retry too early or too late.

**3. API version pinning**
Every REST request MUST include an explicit version header or URL segment (e.g., `Accept: application/vnd.api+json;version=2`). Relying on implicit "latest" version behavior causes silent behavioral drift when the API provider upgrades.

**4. Response schema validation**
REST API responses MUST be validated against the tool's registered output schema (chap32 Stage 5) before being returned to the agent. Schema drift — where the deployed API behavior has diverged from the registered contract — surfaces as a structured `TOOL_SCHEMA_VIOLATION` rather than a silent data corruption.

### GraphQL Integration

GraphQL is the dominant API surface in modern SaaS platforms — GitHub, Shopify, Contentful, and increasingly Salesforce. Four additional engineering concerns beyond REST:

| Concern | Requirement |
|---|---|
| **Schema introspection** | Call the introspection endpoint at startup to validate that hardcoded query shapes are compatible with the current schema version |
| **Fixed query templates** | Tool handlers MUST use fixed named queries or persisted queries — accepting arbitrary GraphQL query strings from the agent is an injection vector |
| **Depth / complexity limits** | Enforce query depth and complexity limits server-side; configure them explicitly rather than relying on server defaults |
| **N+1 detection** | Test each tool handler query for N+1 patterns (a list query that triggers one query per item); use DataLoader batching or single-query joins |

### Design Rules

| Rule ID | Rule |
|---|---|
| `ENT-REST-001` | All REST and GraphQL tool handlers MUST pin the API version explicitly in every request — implicit "latest" version requests are not permitted in production. |
| `ENT-REST-002` | REST adapters that return lists MUST enforce a `maxPages` cap — unbounded pagination can overflow the agent's context window and exhaust its token budget. |
| `ENT-REST-003` | On HTTP 429, REST adapters MUST honor the `Retry-After` response header — fixed-interval retry ignoring the server's recovery window is a correctness defect. |
| `ENT-REST-004` | GraphQL tool handlers MUST use fixed query templates or persisted queries — accepting arbitrary query strings from the agent reasoning loop is a prompt-injection vector. |
| `ENT-REST-005` | Response validation against the output schema (chap32 Stage 5) MUST run before returning REST or GraphQL results to the agent — schema drift MUST surface as `TOOL_SCHEMA_VIOLATION`, not as silent data corruption. |

---

## Section 33.4 — Relational Database Integration

### Safe Database Tool Handler Design

Database tools are among the highest-risk integration surface in an agent system. A single unsafe query can expose entire tables, corrupt transactional data, or cascade into dependent systems. Four non-negotiable design constraints:

**1. Structured parameters, never raw SQL**
Tool handlers MUST accept structured arguments (field names and values) and construct queries internally using parameterized statements. Accepting a raw SQL string as a tool argument is an injection vector regardless of input sanitization.

```java
@Component
public class DatabaseToolHandler {
    private final DataSource readPool;    // Read replica connection pool
    private final DataSource writePool;   // Primary connection pool

    // read-only tool — uses read replica (ENT-DB-003)
    public List<Map<String, Object>> queryByField(
            String table, String field, Object value, int limit) {

        // ENT-DB-001: parameterized query — never string concatenation
        String sql = "SELECT * FROM " + sanitizeIdentifier(table)
                   + " WHERE " + sanitizeIdentifier(field) + " = ? LIMIT ?";

        try (Connection conn = readPool.getConnection();           // ENT-DB-003
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            stmt.setObject(1, value);
            stmt.setInt(2, Math.min(limit, 100));                  // ENT-DB-002: row cap
            return ResultSetMapper.toList(stmt.executeQuery());
        } catch (SQLException e) {
            throw new ToolExecutionException("DB query failed", e);
        }
    }

    // write tool — uses primary, requires idempotency key (ENT-DB-004)
    public void upsertRecord(String table, Map<String, Object> record,
                              String idempotencyKey) {
        // Check idempotency store before executing write
        if (idempotencyStore.seen(idempotencyKey)) return;

        try (Connection conn = writePool.getConnection()) {
            // ... parameterized UPSERT implementation
            idempotencyStore.mark(idempotencyKey);
        } catch (SQLException e) {
            throw new ToolExecutionException("DB write failed", e);
        }
    }

    private String sanitizeIdentifier(String identifier) {
        // Allow only alphanumeric + underscore to prevent SQL identifier injection
        if (!identifier.matches("[a-zA-Z][a-zA-Z0-9_]*")) {
            throw new IllegalArgumentException("Invalid identifier: " + identifier);
        }
        return identifier;
    }
}
```

**2. Row caps on all result sets**
Every query that returns multiple rows MUST enforce an explicit row cap. Default caps should be conservative (e.g., 100 rows) to prevent context window flooding.

**3. Read replica routing**
Read-only tool handlers MUST connect to read replica pools; write tool handlers MUST connect to the primary pool. This reduces primary load, improves query performance, and creates a natural enforcement point for the `sideEffects` classification.

**4. Connection pool configuration**

| Parameter | Guideline |
|---|---|
| `maxPoolSize` | Database connection limit ÷ number of agent instances |
| `acquireTimeout` | Shorter than tool `timeoutMs` — allows error to propagate cleanly |
| `maxConnectionLifetime` | Set to prevent stale connections from idle timeouts or middlebox resets |
| `healthCheckQuery` | `SELECT 1` run before use to detect silently broken connections |

> 💡 **Create dedicated database roles for agent tool handlers** — one read-only role bound to the read-only tools, one restricted write role bound to the specific tables and operations the write tools need. Never connect agent tools as the application's primary service account, and never connect as a database superuser. Least-privilege database roles are your last line of defense when authorization at the tool layer fails.

### Design Rules

| Rule ID | Rule |
|---|---|
| `ENT-DB-001` | Database tool handlers MUST use driver-level parameterized statements — raw SQL string construction from tool arguments is prohibited regardless of input sanitization. |
| `ENT-DB-002` | All database queries that return rows MUST enforce an explicit row cap — uncapped queries can overflow the agent's context window. |
| `ENT-DB-003` | Read-only database tools MUST connect to read replica pools — connecting read-only tools to the primary is a correctness and capacity defect. |
| `ENT-DB-004` | Database write tools MUST implement idempotency via deduplication keys — non-idempotent writes that are retried by the abstraction layer will corrupt data. |
| `ENT-DB-005` | Agent database roles MUST be least-privilege — dedicated read and write roles scoped to required tables only; no shared service account or superuser credentials. |

---

## Section 33.5 — Message Queues and Event Buses

### Agents as Event Producers

When an agent needs to trigger a downstream process without waiting for a result — sending a notification, kicking off a long-running workflow, updating an audit log — producing an event to a message queue is the correct pattern. The tool handler publishes a message and returns immediately; the downstream consumer processes it asynchronously.

```java
@Component
public class EventProducerToolHandler {
    private final KafkaProducer<String, String> producer;
    private final IdempotencyStore idempotencyStore;

    // sideEffects: write, idempotent: true (with deduplication key)
    public void publishEvent(String topic, String eventType,
                              Map<String, Object> payload, String idempotencyKey) {

        if (idempotencyStore.seen(idempotencyKey)) return;  // ENT-MQ-001: idempotency

        String messageId = UUID.randomUUID().toString();
        ProducerRecord<String, String> record = new ProducerRecord<>(topic,
            messageId,
            Json.serialize(Map.of(
                "messageId", messageId,
                "eventType", eventType,
                "payload", payload,
                "producedAt", Instant.now().toString()
            )));

        // ENT-MQ-002: synchronous send confirmation before returning to agent
        try {
            producer.send(record).get(5, TimeUnit.SECONDS);
            idempotencyStore.mark(idempotencyKey);
        } catch (Exception e) {
            throw new ToolExecutionException("Event publish failed", e);
        }
    }
}
```

### Agents as Event Consumers

When agents react to enterprise events — a new support ticket, a completed order, a triggered alert — an event consumer pattern is required. In this model, a lightweight consumer bridge translates queue messages into agent task submissions.

> ⚠️ **At-least-once delivery means the agent MUST be idempotent at the task level.** A message that is delivered twice must produce the same observable outcome as a message delivered once. The deduplication key is the event's `messageId`.

### Key Design Constraints

| Concern | Requirement |
|---|---|
| **Idempotency** | Every write/destructive action triggered by a queue message MUST be guarded by a deduplication key |
| **Poison message handling** | Messages that consistently cause tool failures MUST be routed to a dead-letter queue after a configurable retry threshold — they MUST NOT block the consumer |
| **Backpressure** | Consumer tools MUST respect queue backpressure signals — unbounded consumption can overload downstream systems |
| **Ordering guarantees** | If the tool requires ordered processing, it MUST use the queue's partition key mechanism to ensure in-order delivery per entity |

### Design Rules

| Rule ID | Rule |
|---|---|
| `ENT-MQ-001` | Message queue producer tool handlers MUST implement idempotency via deduplication keys — duplicate publishes on retry MUST be suppressed. |
| `ENT-MQ-002` | Producers MUST await synchronous confirmation of message acceptance before returning success to the agent — fire-and-forget publishing hides delivery failures from the reasoning loop. |
| `ENT-MQ-003` | Consumer bridges MUST route poison messages (consistently failing) to a dead-letter queue after a configurable threshold — they MUST NOT block the consumer indefinitely. |
| `ENT-MQ-004` | Agent actions triggered by queue messages MUST be idempotent at the task level — at-least-once delivery is a fundamental guarantee of every message queue; the agent layer MUST be designed for it. |

---

## Section 33.6 — SaaS Platform Integration

### Salesforce

Salesforce exposes two primary integration surfaces relevant to agents:

- **REST API** — standard CRUD operations on Salesforce Objects (Accounts, Contacts, Opportunities, Cases, custom objects). Use the Connected App OAuth flow with the client credentials grant. Respect the API usage limits — a rolling 24-hour window of API calls per org, shared across all integrations.
- **SOQL** — Salesforce Object Query Language, a SQL-like language for querying object relationships. SOQL is safer than raw SQL for agent use because it is read-only by design — it cannot execute DML statements.

The most important Salesforce-specific concern for agent tool handlers is **governor limits**: per-transaction limits on SOQL queries, DML rows, heap size, and CPU time enforced by the Salesforce runtime. Handlers that batch operations MUST respect these limits or face partial-failure scenarios that are difficult to recover from.

```java
@Component
public class SalesforceToolHandler {
    private static final int MAX_SOQL_ROWS = 200;
    private static final int MAX_DML_ROWS = 200;     // Salesforce governor limit

    public List<Map<String, Object>> queryContacts(String accountId, int limit) {
        int safeLimit = Math.min(limit, MAX_SOQL_ROWS);  // ENT-SF-001
        String soql = "SELECT Id, Name, Email, Title FROM Contact "
                    + "WHERE AccountId = ? LIMIT " + safeLimit;
        return salesforceClient.query(soql, accountId);
    }
}
```

### SAP

SAP integration exposes three main surfaces:

| Surface | When to Use | Notes |
|---|---|---|
| **OData** | New S/4HANA deployments | REST-compatible; preferred for new integrations |
| **BAPI/RFC** | Legacy ECC deployments | Requires SAP JCo (Java); synchronous; high latency |
| **SAP Integration Suite** | Enterprise abstraction layer | Exposes SAP business processes as REST or event APIs; preferred over direct RFC calls |

> ⚠️ **SAP write operations are not idempotent by default.** A BAPI call to post a goods receipt will post it twice if retried. Mark all SAP write tool handlers as non-retryable and require explicit human confirmation (chap12 HITL) for any tool that triggers financial posting, inventory change, or master data modification.

### ServiceNow

ServiceNow exposes two relevant API surfaces:

- **Table API** — REST API for CRUD operations on any ServiceNow table (Incidents, Problems, Change Requests, Configuration Items). Authenticate via OAuth 2.0 using a per-instance token endpoint.
- **Flow Designer API** (Vancouver release and later) — REST-triggerable interface for pre-defined automation flows. **Prefer this over direct Table API writes for business process operations** — flows encapsulate approval routing, notifications, and SLA tracking that the Table API bypasses.

Key ServiceNow-specific concerns:
- Every record has a `sys_id` (UUID) as its primary key — tool handlers MUST accept and return `sys_id` values rather than human-readable display names, which are not unique
- ACL enforcement is per-user — the service account used by the tool handler MUST have the correct ACLs for the tables it accesses; ACL failures return HTTP 200 with an empty result set rather than HTTP 403, making them easy to miss in testing

### Microsoft 365 / Microsoft Graph

Microsoft Graph provides a unified REST API for Microsoft 365 services: Teams, Exchange, SharePoint, OneDrive, Calendar, and more. The defining engineering concern is its **fine-grained OAuth scope model**: over 900 documented scopes, each controlling access to a specific resource and operation.

| Concern | Requirement |
|---|---|
| **Scope minimization** | Request only the specific scopes required by each tool — never request `Mail.ReadWrite` when `Mail.Read` is sufficient |
| **Throttling** | Microsoft Graph enforces per-app, per-user, and per-resource throttling — implement `Retry-After` header handling for all Graph tool handlers |
| **Outbound communication tools** | Email send and Teams message tools MUST be declared as `destructive` side-effect tools and MUST require human confirmation before production sends (see chap12) |
| **Delta queries** | Use Graph delta queries for change-detection patterns — polling full lists repeatedly is expensive and may exhaust throttle quotas |

### Design Rules

| Rule ID | Rule |
|---|---|
| `ENT-SAAS-001` | Salesforce tool handlers MUST enforce SOQL and DML row caps at or below Salesforce governor limits — exceeding governor limits produces partial failures that are difficult to reason about. |
| `ENT-SAAS-002` | SAP BAPI/RFC write tool handlers MUST be declared non-retryable and MUST require HITL confirmation (chap12) for financial postings, inventory changes, and master data modifications. |
| `ENT-SAAS-003` | ServiceNow tool handlers MUST use `sys_id` as the primary key for all record operations — human-readable display names are not unique and produce silent data errors. |
| `ENT-SAAS-004` | ServiceNow tool handlers MUST test ACL rejection explicitly — ACL failures return HTTP 200 with empty result sets, not HTTP 403, making them invisible to standard error handling. |
| `ENT-SAAS-005` | Microsoft Graph tool handlers MUST request the minimum OAuth scope required for their declared `sideEffects` — broad scopes such as `Mail.ReadWrite` MUST NOT be requested when read-only access suffices. |
| `ENT-SAAS-006` | Email, Teams message, and other outbound communication tools MUST be declared `sideEffects: destructive` and MUST require HITL confirmation before any production send operation. |

---

## Section 33.7 — Data Governance and Compliance at the Tool Layer

### Why Governance Belongs at the Tool Layer

An agent that retrieves customer PII from a CRM and sends it to an LLM inference endpoint is performing a data transfer that may be regulated under GDPR, CCPA, HIPAA, or sector-specific data residency requirements. The agent's reasoning loop does not know about data residency. The LLM does not know about PII classifications. **The only place where governance controls can be applied systematically is at the tool layer** — on the data as it passes between enterprise systems and the agent.

### PII Minimization

PII minimization means returning only the personal data fields the agent actually needs to complete its task, stripping fields that are present in the backend response but irrelevant to the tool's declared output schema. This is the output filtering stage (chap32 Stage 6) applied with a governance lens.

```java
@Component
public class PiiOutputFilter {
    // Table-specific PII fields that MUST be stripped before returning to agent
    private static final Map<String, Set<String>> PII_FIELDS = Map.of(
        "customers", Set.of("ssn", "dateOfBirth", "creditCardNumber",
                            "bankAccount", "passportNumber"),
        "employees", Set.of("salary", "performanceRating", "homeAddress",
                            "personalEmail", "nationalId")
    );

    public Map<String, Object> strip(String table, Map<String, Object> row) {
        Set<String> forbidden = PII_FIELDS.getOrDefault(table, Set.of());
        return row.entrySet().stream()
            .filter(e -> !forbidden.contains(e.getKey()))
            .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));
    }
}
```

> ⚠️ **PII field stripping is necessary but not sufficient for GDPR compliance.** The legal basis for processing, purpose limitation, and data subject rights (access, erasure, portability) are governed by the broader data processing architecture, not the tool layer alone. See Volume 2, Chapter 20: Enterprise Governance, GDPR, and Compliance.

### Data Residency Enforcement

Some enterprise datasets cannot leave a specific geographic region. Data residency constraints MUST be modeled as a property of the tool contract and enforced at the abstraction layer's authorization stage (chap32 Stage 2) **before dispatch** — never as a post-processing filter.

```java
// Data residency enforcement at Stage 2 (authorization check)
public class DataResidencyPolicy {

    public AuthDecision evaluate(ToolContract contract, AgentCallingContext ctx) {
        String toolResidency = contract.dataResidencyZone();   // e.g., "EU", "US"
        Set<String> sessionPermitted = ctx.permittedResidencyZones();

        if (!sessionPermitted.contains(toolResidency)) {
            return AuthDecision.deny(
                "Data residency violation: tool zone " + toolResidency
                + " not permitted for session zones " + sessionPermitted);
        }
        return AuthDecision.permit();
    }
}
```

> ⚠️ **Data residency enforcement MUST run at the authorization check stage — before dispatch — not as a post-processing filter.** By the time tool output has been generated, the data has already been accessed and potentially logged in transit. Pre-dispatch enforcement is the only effective control.

### Tenant Isolation at the Tool Layer

Enterprise agents frequently serve multiple tenants on shared infrastructure. Tenant isolation requires three distinct controls:

| Control | Mechanism |
|---|---|
| **Tenant-scoped credentials** | Each tenant has a separate credential set in the secrets store — tool handlers retrieve the credential for the calling session's tenant, never a shared cross-tenant credential |
| **Argument-level scope enforcement** | Tool handlers validate that query arguments (e.g., `customerId`, `orgId`) belong to the calling session's tenant before dispatching — prevents cross-tenant data access even when the agent is compromised |
| **Tenant-scoped audit log** | Every tool call MUST be logged with the tenant identifier — compliance-grade audit trails require per-tenant query capability |

### Compliance-Grade Audit Logging

Every tool call that touches regulated data MUST emit a structured audit log event containing:

```json
{
  "eventType": "tool.data_access",
  "traceId": "t-abc123",
  "agentId": "agent-001",
  "sessionId": "session-xyz",
  "tenantId": "tenant-acme",
  "toolName": "crm.getContact",
  "dataClassification": "PII",
  "dataResidencyZone": "EU",
  "argumentsHash": "sha256:...",
  "resultFieldsReturned": ["id", "name", "email"],
  "piiFieldsStripped": ["ssn", "dateOfBirth"],
  "timestamp": "2026-06-08T15:30:00Z",
  "outcome": "success"
}
```

### Design Rules

| Rule ID | Rule |
|---|---|
| `ENT-GOV-001` | PII field stripping MUST run as a non-optional output filter (chap32 Stage 6) for every tool that accesses tables containing personal data — it MUST be declared in the tool contract, not left to individual handler implementations. |
| `ENT-GOV-002` | Data residency enforcement MUST run at the authorization check stage (chap32 Stage 2) — before dispatch — enforcing it post-dispatch is insufficient because the data transfer has already occurred. |
| `ENT-GOV-003` | Tenant isolation MUST be enforced at three levels: tenant-scoped credentials, argument-level scope validation, and tenant-scoped audit logging — tool-level ACL alone is insufficient. |
| `ENT-GOV-004` | Every tool call that accesses regulated data (PII, financial, health) MUST emit a structured audit log event including tenant ID, data classification, fields returned, and fields stripped. |
| `ENT-GOV-005` | The `dataResidencyZone` field MUST be declared in the tool contract for every tool whose backend is geographically constrained — undeclared residency is treated as a governance defect, not a default. |

---

## Section 33.8 — Connection Management and Operational Reliability

### Connection Lifecycle at Scale

An enterprise agent integrating with five or more backend systems maintains five or more categories of connections: database pools, HTTP client sessions, MCP server connections, message producer/consumer clients, and secrets store clients. In a multi-instance deployment, these multiply by the number of running agent instances. Three management disciplines are required:

1. **Shared connection pools** — where the backend enforces a connection limit (databases, SAP), use a shared connection pool (PgBouncer for PostgreSQL, or a proxy layer) rather than per-instance pools that collectively exceed the limit
2. **Lazy initialization** — establish connections only when first needed, not at agent startup, to prevent cascade failures during deployment where all instances start simultaneously and attempt to connect to all backends at once
3. **Health monitoring** — a background health check loop probes each backend at low frequency and updates a shared health registry; tool dispatch checks the health registry before attempting a call, failing fast to `TOOLUNAVAILABLE` rather than waiting for a connection timeout

### Bulkhead Isolation per Backend

In a system integrating with many enterprise backends, a failure in one backend MUST NOT degrade tool calls to healthy backends. Implement this with separate `BackendAdapter` objects per integration, each with its own pool and circuit breaker:

```java
@Component
public class BackendAdapter {
    private final String name;
    private final ConnectionPool pool;
    private final CircuitBreaker circuitBreaker;
    private final AgentMetrics metrics;

    public <T> T call(String operation, Map<String, Object> args,
                       Class<T> responseType) throws ToolExecutionException {

        if (circuitBreaker.isOpen()) {
            metrics.recordCircuitOpen(name);
            throw new ToolUnavailableException(name + " circuit is open");
        }

        try (Connection conn = pool.acquire()) {
            T result = conn.execute(operation, args, responseType);
            circuitBreaker.recordSuccess();
            return result;

        } catch (Exception e) {
            circuitBreaker.recordFailure();
            metrics.recordBackendFailure(name, e.getClass().getSimpleName());
            throw new ToolExecutionException(name + " call failed", e);
        }
    }
}

// One adapter per backend — bulkhead isolation (ENT-CONN-003)
@Configuration
public class BackendAdapterConfig {
    @Bean public BackendAdapter crmAdapter(SecretsClient secrets) {
        return new BackendAdapter("crm",
            ConnectionPool.of(secrets.getDbConfig("crm"), 10),
            CircuitBreaker.of("crm", 5, Duration.ofSeconds(30)),
            metrics);
    }

    @Bean public BackendAdapter erpAdapter(SecretsClient secrets) {
        return new BackendAdapter("erp",
            ConnectionPool.of(secrets.getDbConfig("erp"), 5),
            CircuitBreaker.of("erp", 3, Duration.ofSeconds(60)),
            metrics);
    }
}
```

### API Gateway as an Integration Topology

Many enterprises expose all internal services through a single API gateway (Kong, AWS API Gateway, Azure APIM, Apigee). This topology changes several integration decisions:

- **Authentication consolidation** — the gateway may accept a single OAuth token for all services; this simplifies secrets management but concentrates blast radius — a compromised gateway token grants access to all gateway-fronted services
- **Rate limit behavior** — limits are enforced at the gateway level; the rate limit counter MUST track against the gateway's aggregate quota rather than per-endpoint limits
- **Gateway as tool discovery** — enterprise API gateways often publish an OpenAPI catalog; this catalog can populate the Tool Registry at startup rather than from static configuration

> ⚠️ **Apply the principle of least privilege at the gateway authorization level.** If the gateway supports fine-grained OAuth scopes per downstream service, request only the scopes required by the agent's tool set rather than a blanket administrator grant.

### Graceful Degradation

When a backend becomes unavailable, the agent MUST degrade gracefully rather than fail entirely:

| Strategy | When to Apply |
|---|---|
| **Fallback tool** | Declared in tool contract (chap32 `ENT-ERR-004`) — live CRM lookup falls back to cached CRM snapshot |
| **Partial results** | Multi-tool operation loses one backend — return results from available backends with a structured annotation indicating which data is missing |
| **Human escalation** | No fallback exists for a required data source — agent surfaces the failure to the human operator rather than hallucinating a result from incomplete data |

> 💡 **Design degradation paths before deployment, not during incidents.** For each enterprise backend integration, define explicitly what the agent does if this system is unavailable. The answer should be in the tool contract and in runbook documentation — not improvised at 2AM.

### Design Rules

| Rule ID | Rule |
|---|---|
| `ENT-CONN-001` | Connection pools for capacity-limited backends MUST be shared across agent instances via a proxy layer (e.g., PgBouncer) — per-instance pools that collectively exceed the backend connection limit cause cascade failures under deployment load. |
| `ENT-CONN-002` | Connections MUST be initialized lazily (on first use) rather than at agent startup — eager initialization of all connections during deployment causes thundering-herd failures when all instances start simultaneously. |
| `ENT-CONN-003` | Each enterprise backend MUST have its own `BackendAdapter` with an independent connection pool and circuit breaker — bulkhead isolation MUST prevent one failing backend from degrading tool calls to healthy backends. |
| `ENT-CONN-004` | A background health check loop MUST maintain a per-backend health registry — tool dispatch MUST fail fast to `TOOLUNAVAILABLE` on an unhealthy backend rather than waiting for a connection timeout. |
| `ENT-CONN-005` | Graceful degradation paths (fallback tool, partial results, human escalation) MUST be defined and tested for every enterprise backend integration before the system goes to production. |

---

## Section 33.9 — Design Rules Master Table

| Rule ID | Scope | Rule Summary |
|---|---|---|
| `ENT-LAND-001` | Landscape | Classify integration surface type before implementation |
| `ENT-LAND-002` | Landscape | Handlers hide backend complexity from agent reasoning loop |
| `ENT-AUTH-001` | Auth | No credentials in code, config files, images, or logs |
| `ENT-AUTH-002` | Auth | Least-privilege credential scope per sideEffects classification |
| `ENT-AUTH-003` | Auth | OAuth token refresh protected by single-writer lock |
| `ENT-AUTH-004` | Auth | Secrets client uniform interface — no SDK calls in handlers |
| `ENT-AUTH-005` | Auth | Dynamic database credentials where secrets store supports them |
| `ENT-REST-001` | REST/GraphQL | Explicit API version pinning on every request |
| `ENT-REST-002` | REST/GraphQL | Pagination cap enforced in all list-returning adapters |
| `ENT-REST-003` | REST/GraphQL | `Retry-After` header honored on HTTP 429 |
| `ENT-REST-004` | REST/GraphQL | Fixed query templates in GraphQL handlers — no arbitrary query strings |
| `ENT-REST-005` | REST/GraphQL | Response schema validation before returning to agent |
| `ENT-DB-001` | Database | Driver-level parameterized statements — no raw SQL construction |
| `ENT-DB-002` | Database | Explicit row cap on all result sets |
| `ENT-DB-003` | Database | Read-only tools connect to read replica pools |
| `ENT-DB-004` | Database | Write tools implement idempotency via deduplication keys |
| `ENT-DB-005` | Database | Least-privilege database roles per tool type |
| `ENT-MQ-001` | Messaging | Producer idempotency via deduplication keys |
| `ENT-MQ-002` | Messaging | Synchronous send confirmation before returning success |
| `ENT-MQ-003` | Messaging | Poison message routing to dead-letter queue |
| `ENT-MQ-004` | Messaging | Agent actions triggered by queue messages are idempotent at task level |
| `ENT-SAAS-001` | SaaS | Salesforce SOQL/DML row caps at or below governor limits |
| `ENT-SAAS-002` | SaaS | SAP BAPI write tools: non-retryable + HITL confirmation |
| `ENT-SAAS-003` | SaaS | ServiceNow: `sys_id` as primary key, not display names |
| `ENT-SAAS-004` | SaaS | ServiceNow: test ACL rejection explicitly (HTTP 200 + empty set) |
| `ENT-SAAS-005` | SaaS | Microsoft Graph: minimum OAuth scope per tool |
| `ENT-SAAS-006` | SaaS | Outbound communication tools: `sideEffects: destructive` + HITL |
| `ENT-GOV-001` | Governance | PII field stripping as non-optional output filter |
| `ENT-GOV-002` | Governance | Data residency enforcement at Stage 2 (before dispatch) |
| `ENT-GOV-003` | Governance | Tenant isolation: credentials + argument scope + audit log |
| `ENT-GOV-004` | Governance | Compliance-grade audit log for regulated data access |
| `ENT-GOV-005` | Governance | `dataResidencyZone` declared in tool contract for constrained backends |
| `ENT-CONN-001` | Connection | Shared connection pools via proxy for capacity-limited backends |
| `ENT-CONN-002` | Connection | Lazy connection initialization |
| `ENT-CONN-003` | Connection | Per-backend `BackendAdapter` with independent pool + circuit breaker |
| `ENT-CONN-004` | Connection | Background health registry; fail fast on unhealthy backends |
| `ENT-CONN-005` | Connection | Graceful degradation paths tested before production |

---

## Section 33.10 — Chapter Boundary: Relationship to Adjacent Chapters

| Chapter | Relationship |
|---|---|
| **chap32** (Tool Abstraction Layer) | chap33 assumes all `TOOL-*` rules are in place — every enterprise handler uses the chap32 dispatch pipeline, contract model, error envelopes, and security stages. chap33 fills the pipeline with concrete backend implementations. |
| **chap12** (HITL) | `ENT-SAAS-002` and `ENT-SAAS-006` reference HITL confirmation gates for SAP write operations and outbound communication tools — chap12 defines the implementation. |
| **chap23** (Observability) | Audit log events defined in Section 33.7 feed the structured logging schema (`OBS-LOG-001–005`). Backend health metrics feed `OBS-METRIC-001`. |
| **chap34** (Advanced Tool Patterns) | chap34 builds on the enterprise handlers defined here to add parallel fan-out across multiple backends, dynamic tool selection from large enterprise tool registries, tool chaining with data flow safety, long-running async enterprise operations, and tool versioning/schema evolution. |
| **Vol 2, chap20** (Enterprise Governance) | PII minimization and data residency in Section 33.7 are the tool-layer controls. Full GDPR/CCPA/HIPAA compliance obligations, legal basis for processing, data subject rights, and cross-border transfer mechanisms are covered in Vol 2, chap20. |
