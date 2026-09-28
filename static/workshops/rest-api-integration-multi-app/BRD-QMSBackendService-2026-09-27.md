# Quality Management System (QMS) Backend Service — Requirement Document

> **For:** Mentor App Generator (ODC)
> **Generated:** 2026-09-27
> **Research basis:** OutSystems Mentor documentation (synced 2026-07-21)
> **App Type:** Service/Library (backend-only, no UI)

---

## App Overview

A headless backend service (API-only, no user interface) that stores and manages manufacturing quality defect data. This app exposes REST APIs that other applications (like the Quality Investigation app) consume to list defects, fetch defect details, retrieve defect history, create new defects, update defect status, and delete defects.

**IMPORTANT: This app has NO screens, NO forms, NO UI components of any kind.** It is a pure backend service accessed only through REST API endpoints.

Built to demonstrate OutSystems multi-app architecture: decomposing backend services from UI apps, encapsulating data access, and exposing clean REST interfaces for frontend consumption.

---

## General App Settings

- App Type: Service/Library (backend-only)
- No theme settings (no UI)
- No screens or user interface
- **This is a headless backend service** — do NOT generate any UI components, screens, forms, or user-facing pages

## UI Requirements

**NONE — This app is API-only with no user interface.**

- Do NOT generate any screens
- Do NOT generate any forms
- Do NOT generate any UI components
- Do NOT generate any navigation
- Do NOT generate any dashboards or reports
- This is a pure backend/headless service

This app exposes REST APIs for other applications to consume. All interaction happens through API calls, not through a user interface.

---

## Data Model

### Entity: Defect

Stores manufacturing quality defect records.

| Attribute | Type | Constraints | Description |
|---|---|---|---|
| DefectId | Text (36) | Primary key, mandatory | Unique identifier for the defect |
| Status | DefectStatus Identifier | Mandatory | Current defect state (Open, Under Investigation, Escalated, Resolved, Closed) |
| Severity | Text (50) | — | Values: Critical, High, Medium, Low |
| ProductLine | Text (200) | — | Manufacturing product line affected |
| Description | Text (1000) | — | Defect description |
| QMSExternalId | Text (100) | — | Reference ID in external QMS system |
| CreatedDate | DateTime | Mandatory | Timestamp when defect was created |
| UpdatedDate | DateTime | — | Timestamp of last update |
| ReportedBy | Text (200) | — | Name or ID of person who reported the defect |
| ResolutionDate | DateTime | — | When the defect was resolved (optional) |
| IsDeleted | Boolean | Mandatory, default False | Soft-delete flag for audit compliance |

**Relationships:**
- One Defect to many DefectHistory (One-to-Many)
- Cascade delete DefectHistory records when a Defect is deleted
- **Soft-delete behaviour:** Deleting a defect sets IsDeleted = True and updates UpdatedDate. The record is retained in the database for audit trail compliance. All list and detail queries exclude records where IsDeleted = True.

### Entity: DefectHistory

Stores historical records of similar or related defects on the same product line.

| Attribute | Type | Constraints | Description |
|---|---|---|---|
| HistoryId | Text (36) | Primary key | Unique identifier for the history record |
| DefectId | Defect Identifier | Mandatory, cascade delete | Reference to the parent Defect |
| PreviousDefectId | Text (36) | — | ID of a related or similar previous defect |
| Severity | Text (50) | — | Severity of the previous defect |
| ResolutionDate | DateTime | — | When the previous defect was resolved |
| Resolution | Text (500) | — | How the previous defect was resolved (for pattern matching) |
| RecurrenceCount | Integer | — | How many times this pattern has occurred |

**Relationships:**
- Many DefectHistory to one Defect (Many-to-One)
- Cascade delete when parent Defect is deleted

### Static Entity: DefectStatus

Fixed status values for defects.

| Id | Identifier | Label |
|---|---|---|
| 1 | Open | Open |
| 2 | UnderInvestigation | Under Investigation |
| 3 | Escalated | Escalated |
| 4 | Resolved | Resolved |
| 5 | Closed | Closed |

**Important for API consumers:** The Status field is serialised as its integer Id in JSON requests and responses, not as the label text. For example, `"status": 1` means Open.

---

## Roles and Permissions

**NONE** — This is a backend service with no user-facing screens. All access control is enforced at the API level (authentication + authorization handled by consuming apps or API gateway).

---

## Main Features and Screens

**NONE — NO SCREENS, NO UI**

This is a headless backend service with no user interface. Do not generate any screens, forms, or UI components.

The service exposes the following REST API endpoints only:

---

## REST API: QMSDefectAPI

**Base path:** `https://{your-domain}/QMSBackend/rest/QMSDefectAPI`

All endpoints require authentication via the X-API-Key header (see Authentication section).

### Endpoint 0 — List Defects

**Method:** GET  
**Path:** `/defects`  
**Description:** Returns a paginated list of non-deleted defects, optionally filtered by status and product line.

**Query Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| Status | DefectStatus Id (Integer) | No | — | Filter by status (1=Open, 2=Under Investigation, etc.) |
| ProductLine | Text | No | — | Filter by product line |
| Page | Integer | No | 1 | Page number |
| PageSize | Integer | No | 20 | Records per page |

**Response (200 OK):** Array of defect summary objects.

```json
[
  {
    "DefectId": "DEF-2026-001",
    "Status": 1,
    "Severity": "Critical",
    "ProductLine": "Assembly Line A",
    "CreatedDate": "2026-09-27T08:30:00",
    "ReportedBy": "qa-user-001"
  }
]
```

**HTTP Status Codes:**
- **200** (success, may be empty array)
- **500** (error)

**Authentication:** Required (X-API-Key)

---

### Endpoint 1 — Get Defect Details

**Method:** GET  
**Path:** `/defects/{DefectId}`  
**Description:** Fetches complete defect information for a single non-deleted defect.

**Path Parameter:** DefectId (Text, 36 chars)

**Response (200 OK):**

```json
{
  "DefectId": "DEF-2026-001",
  "Status": 1,
  "Severity": "Critical",
  "ProductLine": "Assembly Line A",
  "Description": "Solder joint failure in PCB assembly",
  "CreatedDate": "2026-09-27T08:30:00",
  "UpdatedDate": "2026-09-27T08:30:00",
  "ReportedBy": "qa-user-001",
  "ResolutionDate": null
}
```

**HTTP Status Codes:**
- **200** (success)
- **404** (not found or deleted)
- **500** (error)

**Authentication:** Required (X-API-Key)

**Known gap:** The Found output from the underlying server action is not yet mapped to a 404 HTTP response. This is a known limitation — the endpoint currently returns 200 with an empty body when the defect is not found. Requires explicit HTTP status code handling in implementation.

---

### Endpoint 2 — Get Defect History

**Method:** GET  
**Path:** `/defects/{DefectId}/history`  
**Description:** Fetches all history records linked to a given defect.

**Path Parameter:** DefectId (Text, 36 chars)

**Response (200 OK):** Array of history objects.

```json
[
  {
    "PreviousDefectId": "DEF-2025-042",
    "Severity": "High",
    "ResolutionDate": "2025-11-15T00:00:00",
    "Resolution": "Replaced solder paste batch",
    "RecurrenceCount": 3
  }
]
```

**HTTP Status Codes:**
- **200** (success, may be empty array)
- **404** (defect not found)
- **500** (error)

**Authentication:** Required (X-API-Key)

**Known gap:** Same as Endpoint 1 — 404 is not natively enforced at the HTTP level.

---

### Endpoint 3 — Update Defect Status

**Method:** PATCH  
**Path:** `/defects/{DefectId}`  
**Description:** Updates the status and resolution information of a non-deleted defect. Sets UpdatedDate to the current timestamp automatically.

**Path Parameter:** DefectId (Text, 36 chars)

**Request Body:**

```json
{
  "Status": 4,
  "ResolutionDate": "2026-09-28T10:00:00",
  "Resolution": "Root cause identified and corrected"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| Status | DefectStatus Id (Integer) | Yes | New status value |
| ResolutionDate | DateTime | No | When the defect was resolved |
| Resolution | Text (500) | No | Investigation findings |

**Response (200 OK):** Updated defect object (same shape as Endpoint 1 response).

**HTTP Status Codes:**
- **200** (success)
- **404** (not found or deleted)
- **500** (error)

**Authentication:** Required (X-API-Key)

---

### Endpoint 4 — Create Defect

**Method:** POST  
**Path:** `/defects`  
**Description:** Creates a new defect record. Sets CreatedDate and UpdatedDate to the current timestamp automatically. IsDeleted defaults to False.

**Request Body:**

```json
{
  "DefectId": "DEF-2026-001",
  "Status": 1,
  "Severity": "Critical",
  "ProductLine": "Assembly Line A",
  "Description": "Solder joint failure in PCB assembly",
  "ReportedBy": "qa-user-001"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| DefectId | Text (36) | Yes | Unique identifier — caller-supplied |
| Status | DefectStatus Id (Integer) | Yes | Initial status |
| Severity | Text (50) | Yes | Critical, High, Medium, or Low |
| ProductLine | Text (200) | Yes | Affected product line |
| Description | Text (1000) | Yes | Defect description |
| ReportedBy | Text (200) | Yes | Name or ID of reporter |

**Response (200 OK):** Created defect object (same shape as Endpoint 1 response).

**HTTP Status Codes:**
- **200** (success)
- **409** (conflict — DefectId already exists, returns empty body)
- **500** (error)

**Authentication:** Required (X-API-Key)

**Known gap:** The BRD originally specified 201 for success and a proper 409 response body. ODC does not natively return 201 from REST endpoints, and the 409 conflict is currently a silent rejection (empty 200 response). Both require additional implementation to enforce at the HTTP level.

---

### Endpoint 5 — Delete Defect

**Method:** DELETE  
**Path:** `/defects/{DefectId}`  
**Description:** Soft-deletes a defect by setting IsDeleted = True and updating UpdatedDate. The record is retained in the database for audit compliance.

**Path Parameter:** DefectId (Text, 36 chars)

**Response:** Empty body.

**HTTP Status Codes:**
- **200** (success)
- **404** (not found or already deleted)
- **500** (error)

**Authentication:** Required (X-API-Key)

**Known gap:** The BRD originally specified 204 No Content. ODC returns 200 by default — returning 204 requires additional implementation.

---

## External Integrations

**NONE** — This service doesn't integrate with external systems. It's the backend data store that other apps consume.

---

## API Authentication

All endpoints are protected by the OnAuthentication callback inside QMSDefectAPI.

### Server-side (QMS Backend)

- **Setting:** `APIKey` (type: Text, Secret) — configured per stage in the ODC Portal
- **OnAuthentication callback** runs before every request and:
  1. Calls `Request_GetHeader` (from the HTTP extension) with header name `X-API-Key`
  2. Assigns the returned value to a local variable `ReceivedKey`
  3. Compares `ReceivedKey` against the `APIKey` Setting
  4. If they match → allows the request to proceed
  5. If they don't match or the header is missing → raises an Invalid Login exception, returning HTTP 401

### Client-side (consuming app)

- Store the shared key in a Secret Setting named `QMSAPIKey`
- Include the header `X-API-Key: {QMSAPIKey}` on every outbound request to this API
- Configure the Setting value per stage in the ODC Portal — it must match the `APIKey` value set in QMSBackend for the same stage

### Security Requirements

- Keys must never be hardcoded in logic
- Keys must differ per stage (Development ≠ QA ≠ Production)
- Keys must be at least 32 characters, randomly generated
- Settings must be marked as Secret

---

## Platform Implementation Notes

### OnAuthentication Pattern

The OnAuthentication callback has a restricted expression scope in ODC — Settings and request headers cannot be read directly in expressions. The correct pattern is:

1. Add `Request_GetHeader` from the HTTP extension as a public element (Ctrl+Q in ODC Studio)
2. Inside OnAuthentication, use an **Execute Action node** to call `Request_GetHeader("X-API-Key")` — never call it inline in an expression
3. Use an **Assign node** to store the output: `ReceivedKey = GetAPIKeyHeader.Value`
4. Use an **If node** with condition `ReceivedKey = Settings.APIKey` (use the Settings. prefix in the expression editor)
5. **True branch** → End; **False branch** → Raise Invalid Login exception (returns HTTP 401)

**Note on OnRequest callback:** When you set `Authentication = Custom` on the REST API, ODC automatically generates both an **OnAuthentication** and an **OnRequest** callback. Only **OnAuthentication** enforces authentication. **OnRequest** is a pass-through request-transformation hook that runs before every method call — it is irrelevant to authentication and can be left as-is. Do not attempt to implement authentication logic in OnRequest; use only OnAuthentication.

### Status Field Serialisation

DefectStatus identifiers are serialised as integers in JSON. Consumers must use the integer Id, not the label text.

Example: `"status": 1` = Open, `"status": 2` = Under Investigation, etc.

### Server Action List Building Pattern (Critical)

When implementing `ListDefects` and `FetchDefectHistory` server actions, two rules must BOTH be satisfied to avoid empty objects in the API response array:

**Broken pattern (do NOT use):**
```
ForEach row in aggregate
  ListAppend(OutputList, {})
  Assign OutputList.Current.Field = row.Field
```

This pattern has two compounding defects:
1. **Wrong element argument:** `ListAppend` is called with `{}` (an empty record) — so the appended snapshot has no data
2. **Wrong order:** `ListAppend` runs BEFORE the Assign nodes — so even if the element argument were correct, `.Current` has not been populated yet

**Result:** API response with correct item count but all fields blank, e.g., `[{"DefectId": "DEF-001", ...}, {}, {}]`

**Correct pattern:**
```
ForEach row in aggregate
  Assign OutputList.Current.Field = row.Field
  ListAppend(OutputList, OutputList.Current)
```

**Why this works:**
- The Assign nodes run FIRST and populate all fields on `.Current`
- `ListAppend` is then called with `OutputList.Current` (the fully populated record) as its element argument
- The appended snapshot contains all field values
- The API response contains correctly populated objects

**Two things must be correct simultaneously:**
1. **Order** — Assign nodes must run BEFORE ListAppend
2. **Element argument** — ListAppend must receive `OutputList.Current`, not `{}`

**Critical:** Fixing only one of the two is not sufficient. Both defects independently produce empty objects in the response.

**Actions affected:**
- `ListDefects` — builds DefectList from GetDefects aggregate
- `FetchDefectHistory` — builds HistoryList from GetHistoryByDefectId aggregate

If API endpoints return 200 OK with an array where items are empty objects `[{}, {}, {}]` or partially empty, one or both of these rules were not followed.

---

## Known Gaps (Pending Implementation)

| Gap | Description |
|---|---|
| HTTP 201 on POST | ODC returns 200 by default; 201 requires explicit workaround |
| HTTP 204 on DELETE | ODC returns 200 by default; 204 requires explicit workaround |
| HTTP 409 response body | Conflict on duplicate DefectId returns empty 200 instead of 409 |
| HTTP 404 enforcement | Not-found cases return empty 200 instead of 404 for Endpoints 1 and 2 |

---

## Acceptance Criteria

- GET /defects returns paginated list with optional status and product line filters
- GET /defects/{DefectId} returns complete defect data for non-deleted records
- GET /defects/{DefectId}/history returns all history records for a defect
- POST /defects creates a new defect with auto-generated timestamps; rejects duplicate DefectId
- PATCH /defects/{DefectId} updates status and resolution atomically; updates UpdatedDate
- DELETE /defects/{DefectId} soft-deletes the record; retains it for audit trail
- All endpoints reject requests with a missing or invalid X-API-Key header (HTTP 401)
- Soft-deleted records are excluded from all list and detail queries
- DefectHistory maintains referential integrity with its parent Defect
- API documentation is auto-generated and available (OpenAPI/Swagger via ODC Portal)

---

## Success Metrics

- Quality Investigation app successfully consumes all 6 endpoints
- API response time consistently under 500ms
- 100% uptime during integration testing
- API supports 10+ concurrent requests without degradation
- Zero data loss or corruption
