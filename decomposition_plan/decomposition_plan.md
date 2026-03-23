# Monolith Decomposition Plan – Spring Music

---

## 1. Executive Summary

**Objective:**
Spring Music is a single-deployable Spring Boot monolith that bundles the frontend (AngularJS), REST API, domain logic, and data access into one JAR. As the application evolves, the current design creates friction:

- **Scale**: Album catalog reads and writes cannot be scaled independently from UI asset serving or application metadata endpoints
- **Reliability**: A failure in any layer (e.g., database connectivity) brings down the entire application
- **Speed**: All developers work in a single codebase; changes to the data layer require rebuilding and redeploying the full application
- **Database flexibility**: The profile-based database selection is a compile-time coupling that prevents runtime database switching or multi-database operation
- **Team autonomy**: Frontend, API, and data teams cannot release independently

**Non-Goals:**
- Migrating away from Cloud Foundry as the deployment platform
- Replacing the AngularJS frontend with a new framework
- Changing the Album domain model or adding new business capabilities
- Introducing an API gateway or service mesh in this phase
- Addressing authentication and authorization (currently absent)

**Success Metrics:**
- Deployment frequency: Each service can be deployed independently at least once per day without coordinating with other services
- Lead time for change: Time from code commit to production reduced by eliminating cross-service rebuild dependencies
- MTTR: A failure in the frontend or info service does not affect album catalog availability
- Incident rate: Reduction in catalog downtime caused by frontend or metadata service restarts
- Cost / performance targets: Album catalog service can be scaled horizontally without scaling frontend or info service instances

**Constraints:**
- Regulatory / Compliance: None currently documented
- Hosting / Infrastructure: Cloud Foundry; each service deployed as a separate `cf push` with its own manifest
- Release windows: No formal release freeze currently; services must remain backward-compatible during migration
- Security requirements: No authentication in place; this must be addressed before exposing individual services externally

---

## 2. Current State Overview

### 2.1 Business Capability Map

- **Album Catalog Management**: Create, read, update, and delete music album records
- **Catalog Seeding**: Load an initial set of albums from a static JSON file on first startup
- **Application Metadata**: Expose runtime information — active Spring profiles, bound CF services, environment details
- **UI Presentation**: Serve a single-page AngularJS frontend that consumes the REST API
- **Health & Observability**: Expose Spring Boot Actuator endpoints (health, metrics, info)

### 2.2 Application Architecture

**Tech Stack:**
- Language / Framework: Java 8+, Spring Boot 2.4.0, Spring Data (JPA / MongoDB / Redis), Spring Web MVC
- Deployment model: Single fat JAR (`spring-music-1.0.jar`) deployed to Cloud Foundry via `cf push`; 1 GB memory allocation
- Databases: Pluggable at startup via Spring profiles — H2 (default), MySQL, PostgreSQL, MongoDB, Redis, Oracle, SQL Server
- Messaging / Batch: None; `AlbumRepositoryPopulator` runs synchronously on `ApplicationReadyEvent`

**High-Level Diagram:**

```
Browser (AngularJS SPA)
        │
        ▼
┌──────────────────────────────┐
│     Spring Boot Monolith     │
│                              │
│  ┌────────────┐  ┌────────┐  │
│  │AlbumCtrl   │  │InfoCtrl│  │
│  └─────┬──────┘  └────────┘  │
│        │                     │
│  ┌─────▼──────────────────┐  │
│  │   AlbumRepository      │  │
│  │ (JPA / Mongo / Redis)  │  │
│  └─────┬──────────────────┘  │
│        │                     │
└────────┼─────────────────────┘
         │
    [Active DB]
(H2 / MySQL / PG / Mongo / Redis)
```

### 2.3 Dependency & Coupling Summary

- **Shared libraries**: All capabilities share a single Spring application context and the `Album` domain class; there is no module boundary enforced by the build tool
- **Cyclic dependencies**: No explicit cycles, but `AlbumController`, `InfoController`, `AlbumRepositoryPopulator`, and all repository implementations are wired in the same context
- **Shared database tables/collections**: All capabilities share the same single database instance and schema; there is no schema-per-capability isolation
- **Cross-module transactions**: `AlbumRepositoryPopulator` writes to the same repository used by `AlbumController`; both participate in the same data store with no transactional boundary between them
- **Frontend coupling**: Static web assets (AngularJS, Bootstrap, jQuery) are served from the same JAR via Webjars and classpath resources, coupling frontend releases to backend releases

### 2.4 Operational & Change Hotspots

- **High incident modules**: Database connectivity issues affect the entire application including unrelated UI and metadata endpoints
- **High churn modules**: `AlbumController` and repository implementations are the primary areas of change when adding new database support or modifying the catalog API
- **Performance bottlenecks**: All traffic — static assets, API calls, and actuator scrapes — is handled by the same JVM process and thread pool; heavy catalog load degrades frontend responsiveness
- **Profile management complexity**: Adding a new database requires changes to `SpringApplicationContextInitializer`, a new repository implementation, new application.yml profile block, and a new dependency in `build.gradle` — all in one deployment unit

---

## 3. Target State Principles

- Services aligned to **business capabilities (bounded contexts)**:
  - `album-catalog-service` — owns Album CRUD and seeding
  - `app-info-service` — owns runtime metadata and actuator exposure
  - `spring-music-ui` — serves the AngularJS SPA; calls catalog service API
- **Independent deployment and scaling**: Each service has its own `manifest.yml` and can be pushed, scaled, and restarted without affecting others
- **Database per service (end-state)**: `album-catalog-service` owns its database binding exclusively; no other service accesses it directly
- **Backward compatibility during migration**: The existing `GET/PUT/POST/DELETE /albums` API contract is preserved throughout; the UI requires no changes during backend extraction
- **No distributed transactions by default**: Catalog seeding (`AlbumRepositoryPopulator`) moves inside `album-catalog-service`; there is no cross-service write coordination
- **Strong observability (logs, metrics, traces)**: Each service retains Spring Boot Actuator; structured logging and a consistent correlation ID header are introduced to trace requests across `spring-music-ui` → `album-catalog-service`
