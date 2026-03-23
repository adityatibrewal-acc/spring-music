# Use Case Document – Delete Album

---

## Context
- **System / Application Name**: Spring Music
- **Business Domain**: Music Catalog Management
- **Objective / Problem Statement**: Allow users to permanently remove an album from the catalog
- **Intended Audience**: Business, Tech, QA

---

## 1. Use Case Overview
- **Use Case ID**: UC-004
- **Use Case Name**: Delete Album
- **Description**: A user selects an album and deletes it; the system permanently removes the record from the active data store
- **Scope**: Album catalog write operations
- **Level**: User Goal

---

## 2. Actors
- **Primary Actor**: End User (via browser)
- **Secondary Actors**: Spring Music Backend, Active Database (H2 / MySQL / PostgreSQL / MongoDB / Redis)
- **Stakeholders & Interests**: Users want to remove unwanted or incorrect album entries from the catalog

---

## 3. Preconditions
1. The Spring Music application is running and accessible
2. A database profile is active and the repository is reachable
3. The album to be deleted exists in the catalog (has a valid ID)

---

## 4. Trigger
The user clicks the delete action on an album, which sends a `DELETE /albums/{id}` request

---

## 5. Main Success Scenario (Basic Flow)

1. User views the album list and selects an album to delete
2. The frontend sends `DELETE /albums/{id}` where `{id}` is the album's unique identifier
3. `AlbumController` receives the request and passes the ID to the active `AlbumRepository`
4. The repository locates the record by ID and removes it from the database
5. The controller returns HTTP 200
6. The frontend refreshes the album list; the deleted album no longer appears

---

## 6. Alternate Flows

- **AF-1** (From Step 2): User attempts to delete an album that was already deleted (stale UI state) — see EF-1

---

## 7. Exception / Error Flows

- **EF-1**: The provided ID does not match any existing record — the repository's `deleteById` call completes without error (Spring Data does not throw on missing ID); HTTP 200 is returned and the catalog is unchanged
- **EF-2**: Database operation fails — the system returns HTTP 500; the record is not deleted
- **EF-3**: The `{id}` path variable is missing or malformed — the system returns HTTP 400 or HTTP 404 (no matching route)

---

## 8. Postconditions
- **Success Postconditions**: The album record is permanently removed from the active database; it no longer appears in `GET /albums`
- **Failure Postconditions**: The album record remains in the database; no data is lost

---

## 9. Business Rules
1. Deletion is permanent; there is no soft-delete or recycle-bin mechanism
2. Only a single album can be deleted per request (by ID)
3. Deleting a non-existent ID does not raise an error (idempotent delete)

---

## 10. Acceptance Criteria ✅

### Acceptance Criteria

- **AC-1 (Happy Path)**
  - Given an album with a known ID exists in the catalog
  - When the user sends `DELETE /albums/{id}` with that ID
  - Then the response is HTTP 200, and the album no longer appears in `GET /albums`

- **AC-2 (Already Deleted / Non-existent ID)**
  - Given no album exists with the provided ID
  - When the user sends `DELETE /albums/{id}`
  - Then the response is HTTP 200 and the catalog is unchanged

- **AC-3 (Database Failure)**
  - Given the configured database is unreachable or the delete operation fails
  - When the user sends `DELETE /albums/{id}`
  - Then the response is HTTP 500 and the album record remains in the database

---

## 11. Non‑Functional Requirements (if applicable)
- Performance: Delete operation should complete within acceptable latency
- Audit / Logging: No audit logging is currently implemented; deleted records are not recoverable

---

## 12. Assumptions & Dependencies
- Assumptions: Deletion is irreversible; there is no confirmation step enforced server-side
- External systems / integrations: Active database (H2, MySQL, PostgreSQL, MongoDB, or Redis)

---

## 13. Open Issues / Notes
- No soft-delete capability; once deleted, the record cannot be recovered without a database backup
- No authentication or authorization controls delete access
- No cascade rules apply since Album has no child relationships in the current data model
