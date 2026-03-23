# Use Case Document – Add Album

---

## Context
- **System / Application Name**: Spring Music
- **Business Domain**: Music Catalog Management
- **Objective / Problem Statement**: Allow users to add a new album entry to the catalog
- **Intended Audience**: Business, Tech, QA

---

## 1. Use Case Overview
- **Use Case ID**: UC-002
- **Use Case Name**: Add Album
- **Description**: A user submits a new album with its details; the system persists it to the active data store and returns the created record
- **Scope**: Album catalog write operations
- **Level**: User Goal

---

## 2. Actors
- **Primary Actor**: End User (via browser)
- **Secondary Actors**: Spring Music Backend, Active Database (H2 / MySQL / PostgreSQL / MongoDB / Redis)
- **Stakeholders & Interests**: Users want to expand the music catalog with new entries

---

## 3. Preconditions
1. The Spring Music application is running and accessible
2. A database profile is active and the repository is reachable

---

## 4. Trigger
The user fills in the album form and submits a new album, which sends a `PUT /albums` request

---

## 5. Main Success Scenario (Basic Flow)

1. User clicks the "Add Album" action in the UI
2. User fills in the album details: title, artist, release year, genre, and track count
3. The frontend sends `PUT /albums` with the album data as JSON in the request body
4. `AlbumController` receives the request and passes the payload to the active `AlbumRepository`
5. The repository assigns a randomly generated UUID as the album ID
6. The album record is persisted to the active database
7. The controller returns HTTP 200 with the created album (including its generated ID) as JSON
8. The frontend refreshes the album list to include the new entry

---

## 6. Alternate Flows

- **AF-1** (From Step 2): User submits the form with only some fields populated — optional fields are stored as null/empty; required fields (title, artist) must be provided

---

## 7. Exception / Error Flows

- **EF-1**: Request body is missing or malformed JSON — the system returns HTTP 400 Bad Request
- **EF-2**: Validation fails (e.g., required field is blank) — the system returns HTTP 400 with validation error details
- **EF-3**: Database write fails — the system returns HTTP 500; no record is created

---

## 8. Postconditions
- **Success Postconditions**: A new album record exists in the active database; the album is visible in subsequent `GET /albums` calls
- **Failure Postconditions**: No new record is created; the catalog state is unchanged

---

## 9. Business Rules
1. Each album is assigned a unique, randomly generated UUID by `RandomIdGenerator` at creation time
2. Duplicate albums (same title and artist) are not prevented by the system
3. The `albumId` field is a secondary identifier separate from the internal primary key

---

## 10. Acceptance Criteria ✅

### Acceptance Criteria

- **AC-1 (Happy Path)**
  - Given the application is running and the database is reachable
  - When the user sends `PUT /albums` with a valid album JSON body
  - Then the response is HTTP 200 with the created album including a generated ID, and the album appears in `GET /albums`

- **AC-2 (Validation Error)**
  - Given the application is running
  - When the user sends `PUT /albums` with a missing required field (e.g., no title)
  - Then the response is HTTP 400 and no record is created in the database

- **AC-3 (Malformed Request)**
  - Given the application is running
  - When the user sends `PUT /albums` with an invalid or empty request body
  - Then the response is HTTP 400 and the catalog is unchanged

---

## 11. Non‑Functional Requirements (if applicable)
- Performance: Album creation should complete within acceptable latency
- Audit / Logging: No audit logging is currently implemented

---

## 12. Assumptions & Dependencies
- Assumptions: ID generation is handled server-side; clients must not supply an ID
- External systems / integrations: Active database (H2, MySQL, PostgreSQL, MongoDB, or Redis)

---

## 13. Open Issues / Notes
- No deduplication logic exists; identical albums can be added multiple times
- No authentication or authorization controls write access
