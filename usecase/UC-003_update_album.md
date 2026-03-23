# Use Case Document – Update Album

---

## Context
- **System / Application Name**: Spring Music
- **Business Domain**: Music Catalog Management
- **Objective / Problem Statement**: Allow users to modify the details of an existing album in the catalog
- **Intended Audience**: Business, Tech, QA

---

## 1. Use Case Overview
- **Use Case ID**: UC-003
- **Use Case Name**: Update Album
- **Description**: A user edits an existing album's details; the system persists the changes to the active data store and returns the updated record
- **Scope**: Album catalog write operations
- **Level**: User Goal

---

## 2. Actors
- **Primary Actor**: End User (via browser)
- **Secondary Actors**: Spring Music Backend, Active Database (H2 / MySQL / PostgreSQL / MongoDB / Redis)
- **Stakeholders & Interests**: Users want to correct or update album metadata in the catalog

---

## 3. Preconditions
1. The Spring Music application is running and accessible
2. A database profile is active and the repository is reachable
3. The album to be updated already exists in the catalog (has a valid ID)

---

## 4. Trigger
The user edits an album's fields in the UI and submits the changes, sending a `POST /albums` request

---

## 5. Main Success Scenario (Basic Flow)

1. User selects an existing album from the list
2. The UI pre-populates the edit form with the album's current values
3. User modifies one or more fields (title, artist, release year, genre, track count)
4. The frontend sends `POST /albums` with the full album JSON (including the existing album ID) in the request body
5. `AlbumController` receives the request and passes the payload to the active `AlbumRepository`
6. The repository locates the record by ID and overwrites it with the new values
7. The controller returns HTTP 200 with the updated album as JSON
8. The frontend refreshes the album list to reflect the changes

---

## 6. Alternate Flows

- **AF-1** (From Step 3): User makes no changes and submits — the system performs an idempotent update; the record is saved with the same values and HTTP 200 is returned

---

## 7. Exception / Error Flows

- **EF-1**: The album ID in the request body does not match any existing record — behaviour depends on the active repository implementation; the record may be inserted as new (upsert) rather than updated
- **EF-2**: Request body is missing or malformed JSON — the system returns HTTP 400 Bad Request
- **EF-3**: Validation fails (e.g., required field is blank) — the system returns HTTP 400 with validation error details
- **EF-4**: Database write fails — the system returns HTTP 500; the record is unchanged

---

## 8. Postconditions
- **Success Postconditions**: The album record in the database reflects the submitted values; changes are visible in subsequent `GET /albums` calls
- **Failure Postconditions**: The album record is unchanged in the database

---

## 9. Business Rules
1. The album ID must be present in the request body; it identifies the record to update
2. All fields are replaced with the submitted values (full replacement, not partial patch)
3. The album ID itself cannot be changed through this operation

---

## 10. Acceptance Criteria ✅

### Acceptance Criteria

- **AC-1 (Happy Path)**
  - Given an album with a known ID exists in the catalog
  - When the user sends `POST /albums` with the album's ID and updated field values
  - Then the response is HTTP 200 with the updated album, and `GET /albums` reflects the new values

- **AC-2 (Validation Error)**
  - Given an album with a known ID exists in the catalog
  - When the user sends `POST /albums` with a missing required field
  - Then the response is HTTP 400 and the album record is unchanged in the database

- **AC-3 (Malformed Request)**
  - Given the application is running
  - When the user sends `POST /albums` with an invalid or empty request body
  - Then the response is HTTP 400 and the catalog is unchanged

---

## 11. Non‑Functional Requirements (if applicable)
- Performance: Update operation should complete within acceptable latency
- Audit / Logging: No audit logging is currently implemented; previous values are not retained

---

## 12. Assumptions & Dependencies
- Assumptions: The client always sends the complete album object (full replace semantics); partial updates are not supported
- External systems / integrations: Active database (H2, MySQL, PostgreSQL, MongoDB, or Redis)

---

## 13. Open Issues / Notes
- Upsert behaviour when a non-existent ID is provided is not explicitly guarded against and may vary by repository implementation
- No optimistic locking or conflict detection is in place
- No authentication or authorization controls write access
