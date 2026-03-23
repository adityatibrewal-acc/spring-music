# Spring Music – Modernization Report

**Date:** 2026-03-23
**Source:** `spring-music/` (original monolith)
**Target:** `spring-music-new/` (modernized, decomposed)

---

## 1. Executive Summary

The original Spring Music application was a single Spring Boot 2.4.0 monolith bundling the REST API, multiple database implementations, application metadata endpoints, and an AngularJS frontend into one deployable JAR. The modernization effort addressed three goals:

1. **Technology upgrade** — Bring the stack to current supported versions (Spring Boot 3.2, Java 17, Gradle 8)
2. **Service decomposition** — Break the monolith into three independently deployable services aligned to distinct business capabilities
3. **Code quality** — Replace deprecated APIs, fix incorrect configuration, and add proper exception handling

---

## 2. Decomposition: Monolith → Three Services

| Service | Port | Responsibility |
|---|---|---|
| `album-catalog-service` | 8080 | Album CRUD, all database implementations, seed data loading |
| `app-info-service` | 8081 | Runtime metadata (`/appinfo`, `/service`), CF environment introspection |
| `spring-music-ui` | 8082 | AngularJS SPA (static assets) + reverse proxy to backend services |

### Why this split

- **album-catalog-service** is the highest-churn, highest-load component. Isolating it allows independent scaling and database binding without touching the UI or metadata service.
- **app-info-service** is operationally stable read-only metadata. Separating it means a catalog failure does not prevent operators from inspecting which services are bound.
- **spring-music-ui** owns the browser experience. Decoupling it allows UI changes and redeployments without any backend restart.

### Frontend compatibility

The AngularJS SPA uses relative URLs (`albums`, `appinfo`, `errors/*`). Rather than modifying the JavaScript, `spring-music-ui` includes a `ProxyController` that transparently forwards all API calls to the appropriate backend service. Zero changes were made to the frontend code.

---

## 3. Technology Upgrades

### 3.1 Spring Boot: 2.4.0 → 3.2.3

Spring Boot 3 is the current long-term-support release line. 2.4.x reached end-of-life in February 2023.

**Impact on code:**
- All `javax.*` imports replaced with `jakarta.*` (Spring Boot 3 requires Jakarta EE 10)
- Auto-configuration class names and package structure unchanged for the exclusions used in `SpringApplicationContextInitializer`
- `spring.profiles:` YAML key replaced with `spring.config.activate.on-profile:` (required since Spring Boot 2.4)

### 3.2 Java: 8 → 17

Java 17 is the current LTS release. Spring Boot 3 requires a minimum of Java 17.

**New language features used:**
- **Records** (`ApplicationInfo`) — replaces a mutable POJO with a concise, immutable data carrier
- **Enhanced switch / text blocks** — not used but available for future development
- **Local variable type inference (`var`)** — available where appropriate

### 3.3 Gradle: 6.7 → 8.7

Gradle 6.7 reached end-of-life. Gradle 8.x is the current stable release.

**Build script modernisation:**
- Replaced legacy `buildscript {}` block and `apply plugin:` statements with the modern `plugins {}` DSL block
- Removed `jcenter()` repository (deprecated and shut down); all dependencies now resolve from `mavenCentral()` only
- Removed `eclipse-wtp` and `idea` plugins (IDE integration is handled by IDE tooling, not the build script)
- Replaced `java { sourceCompatibility }` with `java { toolchain { languageVersion = JavaLanguageVersion.of(17) } }` for reproducible cross-machine builds

### 3.4 JUnit: 4 → 5 (Jupiter)

Spring Boot 3 test starter no longer includes JUnit 4 by default. `junit:junit` dependency removed; tests now use `org.junit.jupiter.api.Test`.

---

## 4. Code-Level Changes

### 4.1 Namespace: `javax.*` → `jakarta.*`

| File | Old import | New import |
|---|---|---|
| `Album.java` | `javax.persistence.*` | `jakarta.persistence.*` |
| `Album.java` | `javax.validation.constraints.*` | `jakarta.validation.constraints.*` |
| `AlbumController.java` | `javax.validation.Valid` | `jakarta.validation.Valid` |
| `ProxyController.java` (new) | — | `jakarta.servlet.http.HttpServletRequest` |

### 4.2 Database Driver: MySQL

| Old | New |
|---|---|
| `mysql:mysql-connector-java` | `com.mysql:mysql-connector-j` |
| Driver class `com.mysql.jdbc.Driver` | `com.mysql.cj.jdbc.Driver` |

The old artifact was renamed and the legacy driver class deprecated since MySQL Connector/J 8.0.

### 4.3 Hibernate Dialect Configuration Removed

The original `application.yml` specified:
```yaml
hibernate:
  dialect: org.hibernate.dialect.MySQL55Dialect    # outdated
  dialect: org.hibernate.dialect.ProgressDialect   # incorrect — Progress DB, not PostgreSQL
```

Both entries are removed. Hibernate 6 (bundled in Spring Boot 3) auto-detects the correct dialect from the JDBC URL and driver metadata. Explicit dialect configuration is only needed for non-standard databases.

### 4.4 YAML Multi-Document Profile Syntax

| Old (Spring Boot < 2.4) | New (Spring Boot 2.4+, required in 3.x) |
|---|---|
| `spring.profiles: mysql` | `spring.config.activate.on-profile: mysql` |

The old syntax was deprecated in Spring Boot 2.4 and removed in 3.0.

### 4.5 Hibernate 6 IdentifierGenerator API

`RandomIdGenerator` implemented `IdentifierGenerator` from Hibernate 5, which used a `SessionImplementor` parameter. In Hibernate 6 this was replaced with `SharedSessionContractImplementor`.

```java
// Before (Hibernate 5)
public Serializable generate(SessionImplementor session, Object object)

// After (Hibernate 6)
public Object generate(SharedSessionContractImplementor session, Object object)
```

### 4.6 HTTP Mapping Annotations

`@RequestMapping(method = RequestMethod.GET)` replaced throughout with purpose-specific annotations for clarity:

| Old | New |
|---|---|
| `@RequestMapping(method = GET)` | `@GetMapping` |
| `@RequestMapping(method = PUT)` | `@PutMapping` |
| `@RequestMapping(method = POST)` | `@PostMapping` |
| `@RequestMapping(method = DELETE)` | `@DeleteMapping` |

### 4.7 HTTP Status Codes Corrected

The original controller returned HTTP 200 for all operations. REST semantics corrected:

| Operation | Old status | New status |
|---|---|---|
| `PUT /albums` (create) | 200 OK | **201 Created** |
| `DELETE /albums/{id}` | 200 OK | **204 No Content** |
| `GET /albums` | 200 OK | 200 OK (unchanged) |
| `POST /albums` (update) | 200 OK | 200 OK (unchanged) |

### 4.8 Exception Handling Added

The original application had no centralised exception handling; validation errors and unexpected exceptions returned raw Spring error responses.

New `GlobalExceptionHandler` (`@RestControllerAdvice`) handles:

| Exception | HTTP Status | Response format |
|---|---|---|
| `MethodArgumentNotValidException` | 400 Bad Request | RFC 7807 `ProblemDetail` with field-level error messages |
| `Exception` (catch-all) | 500 Internal Server Error | RFC 7807 `ProblemDetail` with generic message; full stack trace logged |

RFC 7807 `ProblemDetail` is natively supported in Spring 6 (Spring Boot 3) with no additional dependency.

### 4.9 CORS Configuration Added

`album-catalog-service` now exposes a `WebConfig` that allows cross-origin requests from the UI service host. The allowed origins are configurable via `cors.allowed-origins` in `application.yml`, defaulting to `http://localhost:8082`.

### 4.10 `ApplicationInfo` Converted to Java Record

```java
// Before — mutable POJO, 28 lines
public class ApplicationInfo {
    private String[] profiles;
    private String[] services;
    public ApplicationInfo(String[] profiles, String[] services) { ... }
    public String[] getProfiles() { ... }
    public void setProfiles(String[] profiles) { ... }
    public String[] getServices() { ... }
    public void setServices(String[] services) { ... }
}

// After — immutable record, 3 lines
public record ApplicationInfo(String[] profiles, String[] services) {}
```

### 4.11 `RedisAlbumRepository` Cleaned Up

- `deleteAll(Iterable)` and `deleteAllById(Iterable)` implementations replaced with lambda forEach calls (removing manual iterator loops)
- `convertIterableToList` method inlined with `iterable.forEach(list::add)`
- `save()` now calls `idGenerator.generate(null, album).toString()` using the Hibernate 6 signature

### 4.12 Logger Calls

SLF4J parameterised logging used consistently throughout:
```java
// Before
logger.info("Adding album " + album.getId());   // string concatenation

// After
logger.info("Adding album {}", album.getTitle());  // lazy interpolation
```

---

## 5. New Files Added

| File | Service | Purpose |
|---|---|---|
| `config/WebConfig.java` | album-catalog-service | CORS configuration for cross-origin UI access |
| `web/GlobalExceptionHandler.java` | album-catalog-service | Centralised exception → RFC 7807 ProblemDetail mapping |
| `info/InfoApplication.java` | app-info-service | New standalone Spring Boot entry point |
| `info/domain/ApplicationInfo.java` | app-info-service | Java 17 record replacing the original POJO |
| `info/web/InfoController.java` | app-info-service | Extracted `InfoController` with own package |
| `ui/UiApplication.java` | spring-music-ui | Spring Boot entry point for UI service |
| `ui/web/ProxyController.java` | spring-music-ui | Reverse proxy forwarding browser calls to backend services |
| `ui/web/RestTemplateConfig.java` | spring-music-ui | `RestTemplate` bean for the proxy |

---

## 6. Unchanged Items

The following were intentionally left unchanged:

- All AngularJS JavaScript files (`albums.js`, `app.js`, `errors.js`, `info.js`, `status.js`)
- All HTML templates (`index.html`, `templates/*.html`)
- All CSS files
- `AlbumRepositoryPopulator` logic (seed data behaviour)
- `JpaAlbumRepository` and `MongoAlbumRepository` (interface-only, no code to change)
- `SpringApplicationContextInitializer` logic (CF profile detection algorithm unchanged)
- `albums.json` seed data

---

## 7. Build & Run Instructions

### Local development (all services)

```bash
# Terminal 1 — Album Catalog Service (default: H2 in-memory)
cd album-catalog-service
./gradlew clean assemble
java -jar build/libs/album-catalog-service-1.0.jar

# Terminal 2 — App Info Service
cd app-info-service
./gradlew clean assemble
java -jar build/libs/app-info-service-1.0.jar

# Terminal 3 — UI (proxy + static files)
cd spring-music-ui
./gradlew clean assemble
java -jar build/libs/spring-music-ui-1.0.jar
# Open http://localhost:8082
```

### With a specific database profile

```bash
java -jar -Dspring.profiles.active=postgres build/libs/album-catalog-service-1.0.jar
```

### Run tests

```bash
./gradlew test                                          # all tests
./gradlew test --tests '*.ApplicationTests'             # single test class
```

### Cloud Foundry deployment

```bash
# Deploy each service independently
cd album-catalog-service && cf push
cd ../app-info-service   && cf push
cd ../spring-music-ui    && cf push

# Set backend URLs in UI service after other services are pushed
cf set-env spring-music-ui SERVICES_CATALOG_URL https://<catalog-route>
cf set-env spring-music-ui SERVICES_INFO_URL     https://<info-route>
cf restart spring-music-ui
```

---

## 8. Known Limitations & Next Steps

| Item | Notes |
|---|---|
| Authentication / Authorization | Still absent. Recommended next step: add Spring Security with OAuth2 resource server to `album-catalog-service` |
| AngularJS version | Still on 1.2.16 (EOL). Recommend migrating to a current framework (Angular 17, React, Vue) in a follow-up iteration |
| `@GenericGenerator` deprecation | Still used in `Album.java`. Can be replaced with `@UuidGenerator` from Hibernate 6 once the custom UUID format (no dashes) is no longer required |
| Observability | Actuator endpoints exposed but no distributed tracing. Recommend adding Micrometer Tracing with a correlation ID header propagated through `ProxyController` |
| Service discovery | Backend URLs in `spring-music-ui` are static config. Recommend Spring Cloud Discovery or environment-based routing for production CF deployments |
