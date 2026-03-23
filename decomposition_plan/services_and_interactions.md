# Spring Music – Services & Interaction Overview

---

## Existing Services & Components

### 1. `AlbumController` — Album Catalog REST API
**File:** `src/main/java/.../web/AlbumController.java`

The primary API surface of the application. Exposes full CRUD over albums.

| Method | Endpoint | Action |
|---|---|---|
| `GET` | `/albums` | Return all albums |
| `GET` | `/albums/{id}` | Return a single album by ID |
| `PUT` | `/albums` | Create a new album |
| `POST` | `/albums` | Update an existing album |
| `DELETE` | `/albums/{id}` | Delete an album by ID |

Delegates all data access to the single `CrudRepository<Album, String>` bean wired at startup — it has no knowledge of which database is active.

---

### 2. `InfoController` — Application Metadata API
**File:** `src/main/java/.../web/InfoController.java`

Exposes runtime environment information, primarily to show which database profile and CF services are active.

| Method | Endpoint | Action |
|---|---|---|
| `GET` | `/appinfo` | Returns active Spring profiles + bound CF service names |
| `GET` | `/service` | Returns full CF service binding details from `VCAP_SERVICES` |

Uses `CfEnv` (Java CF Env library) to read Cloud Foundry environment variables. Works locally too — returns empty service list when not on CF.

---

### 3. `ErrorController` — Failure Simulation API
**File:** `src/main/java/.../web/ErrorController.java`

A demo/testing utility that intentionally crashes the JVM in different ways. Used to demonstrate Cloud Foundry's self-healing (automatic app restart).

| Method | Endpoint | Action |
|---|---|---|
| `GET` | `/errors/kill` | Calls `System.exit(1)` — hard JVM shutdown |
| `GET` | `/errors/fill-heap` | Allocates infinite arrays until `OutOfMemoryError` |
| `GET` | `/errors/throw` | Throws a `NullPointerException` |

---

### 4. `SpringApplicationContextInitializer` — Profile Resolver
**File:** `src/main/java/.../config/SpringApplicationContextInitializer.java`

Runs before any beans are created. Inspects CF-bound services and activates the matching Spring profile. Also dynamically excludes Spring Boot auto-configurations that aren't needed (e.g., suppresses MongoDB auto-config when using JPA).

- Validates only one database profile is active (throws on conflict)
- Maps CF service tags → profiles: `mongodb`, `postgres`, `mysql`, `redis`, `oracle`, `sqlserver`
- Falls back to default (H2) if no CF service is bound and no profile is manually set

---

### 5. `AlbumRepositoryPopulator` — Seed Data Loader
**File:** `src/main/java/.../repositories/AlbumRepositoryPopulator.java`

Listens for `ApplicationReadyEvent` (fires once the app is fully started). If the repository is empty (`count() == 0`), it deserialises `albums.json` from the classpath and saves each album via the active repository. Runs exactly once per fresh database.

---

### 6. Repository Implementations — Data Access Layer

Three implementations of `CrudRepository<Album, String>`, exactly one is active at runtime based on the Spring profile:

| Class | Profile | Storage | Mechanism |
|---|---|---|---|
| `JpaAlbumRepository` | default / `mysql` / `postgres` / `oracle` / `sqlserver` | Relational DB | Spring Data JPA + Hibernate; DDL auto-generated |
| `MongoAlbumRepository` | `mongodb` | MongoDB | Spring Data MongoDB |
| `RedisAlbumRepository` | `redis` | Redis | Manual `HashOperations` on a single `albums` hash key |

---

### 7. AngularJS SPA — Frontend
**Location:** `src/main/resources/static/`

A single-page app served as static files from the same JAR. Calls the backend REST API (`/albums`, `/appinfo`) directly. Bundled via Webjars (Bootstrap, AngularJS, jQuery).

---

## Interaction Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                          Browser                                │
│                    (AngularJS SPA)                              │
│         served from /static/ inside the JAR                    │
└───────┬──────────────────────┬──────────────────────┬──────────┘
        │ GET/PUT/POST/DELETE  │ GET /appinfo         │ GET /errors/*
        │ /albums              │ GET /service         │
        ▼                      ▼                      ▼
┌────────────────┐   ┌──────────────────┐   ┌─────────────────────┐
│ AlbumController│   │  InfoController  │   │   ErrorController   │
│  /albums       │   │  /appinfo        │   │   /errors/kill      │
│  /albums/{id}  │   │  /service        │   │   /errors/fill-heap │
└───────┬────────┘   └───────┬──────────┘   │   /errors/throw     │
        │                    │              └─────────────────────┘
        │ CrudRepository     │ CfEnv
        │ (interface)        │ (reads VCAP_SERVICES)
        ▼                    ▼
┌───────────────────────────────────────────────┐
│           Active Repository (one of):         │
│                                               │
│  [default/jpa]  JpaAlbumRepository            │
│  [mongodb]      MongoAlbumRepository          │
│  [redis]        RedisAlbumRepository          │
└───────┬───────────────────────────────────────┘
        │
        │ writes on ApplicationReadyEvent (if empty)
        │ ◄── AlbumRepositoryPopulator ◄── albums.json
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Active Database                              │
│                                                                 │
│   H2 (default) │ MySQL │ PostgreSQL │ MongoDB │ Redis           │
│   Oracle │ SQL Server                                           │
└─────────────────────────────────────────────────────────────────┘
        ▲
        │  profile auto-detected from CF service tags
        │
┌───────┴────────────────────────────────────────────────────────┐
│           SpringApplicationContextInitializer                  │
│  (runs at startup, before beans are created)                   │
│  · reads VCAP_SERVICES via CfEnv                               │
│  · activates matching profile (mongodb/postgres/mysql/redis…)  │
│  · excludes conflicting Spring Boot auto-configurations        │
│  · validates only one DB profile is active                     │
└────────────────────────────────────────────────────────────────┘
```

---

## Key Design Note

`AlbumController` is completely decoupled from the database — it only depends on the `CrudRepository` interface. The entire database-switching mechanism lives in `SpringApplicationContextInitializer` (profile selection at startup) and the three `@Profile`-annotated repository beans (only one is registered per run).
