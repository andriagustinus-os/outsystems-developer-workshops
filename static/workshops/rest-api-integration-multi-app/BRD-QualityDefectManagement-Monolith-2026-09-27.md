# Quality Management System — Monolithic Application (QMSMonolith)

> **For:** Mentor App Generator (ODC)  
> **Generated:** 2026-09-27  
> **App Type:** Web Application (ODC)  
> **Architecture:** Monolithic (UI + Data + Logic in single app)  
> **Status:** Ready for Development  
> **Alternative to:** QMSBackend + QMSWebapp (microservices architecture)

---

## Overview

QMSMonolith is a **single monolithic application** for the Quality Management System that combines all functionality in one app: user interface, business logic, and data persistence. All components share the same database directly (no REST API layer).

**Same features as the microservices version (QMSBackend + QMSWebapp), but simpler architecture and deployment.**

---

## Architectural Difference

### Microservices (Current Alternative)
```
QMSWebapp (UI) → REST API → QMSBackend (Data)
```
- Two separate apps
- API authentication (X-API-Key)
- REST client integration
- More complex deployment

### Monolithic (This BRD)
```
QMSMonolith (UI + Data + Logic in one app)
```
- Single app
- Standard OutSystems authentication only
- Direct database access
- Simpler deployment and operations

**Feature parity:** Both versions provide identical user-facing functionality.

---

## 1. Data Model

### Entity: Defect

| Attribute | Type | Constraints | Description |
|---|---|---|---|
| DefectId | Text (36) | Primary key, mandatory | Unique identifier for the defect |
| Status | DefectStatus Identifier | Mandatory | Current defect state (1=Open, 2=Under Investigation, 3=Escalated, 4=Resolved, 5=Closed) |
| Severity | Text (50) | — | Values: Critical, High, Medium, Low |
| ProductLine | Text (200) | — | Manufacturing product line affected |
| Description | Text (1000) | — | Defect description |
| CreatedDate | DateTime | Mandatory | Timestamp when defect was created |
| UpdatedDate | DateTime | — | Timestamp of last update |
| ReportedBy | Text (200) | — | Name or ID of person who reported the defect |
| ResolutionDate | DateTime | — | When the defect was resolved (optional) |
| IsDeleted | Boolean | Mandatory, default False | Soft-delete flag for audit compliance |

**Relationships:**
- One Defect to many DefectHistory (One-to-Many)
- Cascade delete DefectHistory records when a Defect is deleted

### Entity: DefectHistory

| Attribute | Type | Constraints | Description |
|---|---|---|---|
| HistoryId | Text (36) | Primary key | Unique identifier for the history record |
| DefectId | Defect Identifier | Mandatory, cascade delete | Reference to the parent Defect |
| PreviousDefectId | Text (36) | — | ID of a related or similar previous defect |
| Severity | Text (50) | — | Severity of the previous defect |
| ResolutionDate | DateTime | — | When the previous defect was resolved |
| Resolution | Text (500) | — | How the previous defect was resolved (for pattern matching) |
| RecurrenceCount | Integer | — | How many times this pattern has occurred |

### Static Entity: DefectStatus

| Id | Identifier | Label |
|---|---|---|
| 1 | Open | Open |
| 2 | UnderInvestigation | Under Investigation |
| 3 | Escalated | Escalated |
| 4 | Resolved | Resolved |
| 5 | Closed | Closed |

---

## 2. Authentication & Authorization

### Built-in OutSystems Authentication

- Uses ODC's standard login (username + password)
- No custom API key management needed
- Session timeout: 60 minutes (default)

### Roles and Permissions

| Role | Description | Permissions |
|---|---|---|
| **QualityEngineer** | Front-line QA staff | View list, view detail, view history, create defect |
| **QualityManager** | Supervisors; can manage defect lifecycle | All of above, plus update status, soft-delete |
| **Admin** | System administrators (optional future) | All permissions |

**Permission enforcement:**
- UI-level: Hide buttons/screens for unauthorized roles
- Server-action-level: Raise Security Exception if called without required role

---

## 3. Main Screens

All screens use direct database access (no REST API calls). Same UI/UX as microservices version.

### Screen 1: Login
- Standard ODC login with username/password
- Redirect to DefectList on success

### Screen 2: Defect List
- Paginated table of all defects (non-deleted)
- Filters: Status, Product Line
- Actions: View detail, Create new, Refresh

### Screen 3: Defect Detail
- Full defect information
- History section (related defects)
- Actions: Back, Update Status (QualityManager only), Delete (QualityManager only)

### Screen 4: Create Defect
- Form: DefectId, Status, Severity, ProductLine, Description, ReportedBy
- Validation: All fields mandatory, unique DefectId
- Actions: Save (creates defect), Cancel

### Screen 5: Update Defect Status
- Pre-load current defect
- Form: New Status (mandatory), ResolutionDate (optional), Resolution Notes (optional)
- Actions: Save, Cancel
- Access: QualityManager only

### Screen 6: Logout
- Clear session, redirect to Login

---

## 4. Features and Workflows

### Feature 1: List Defects with Pagination
- Display 20 defects per page
- Support Status and ProductLine filters
- Show summary fields: DefectId, Status (badge), Severity (badge), ProductLine, ReportedBy, CreatedDate
- Empty state message when no results

### Feature 2: View Defect Detail
- Show all defect fields
- Show related history records
- Empty state for no history
- Show "Not found" message if defect doesn't exist or is deleted

### Feature 3: Create Defect
- Form validation (client-side + server-side)
- Mandatory fields: DefectId (unique), Status, Severity, ProductLine, Description, ReportedBy
- On success: Redirect to DefectDetail, show success message
- On conflict (duplicate ID): Show inline error
- On validation failure: Show field-level error messages

### Feature 4: Update Defect Status
- Change status and optionally add resolution info
- Only QualityManager can access
- On success: Redirect to DefectDetail
- On not-found: Show error message
- On validation failure: Show inline errors

### Feature 5: Soft-Delete Defect
- Only QualityManager can delete
- Confirmation dialog before deletion
- Sets IsDeleted = True, updates UpdatedDate
- Deleted defect removed from all lists immediately
- Record retained in database for audit trail

### Feature 6: Audit Trail
- All CREATE, UPDATE, DELETE operations log timestamp and user
- Soft-deleted records remain in database (not visible in UI)
- Supports compliance requirements (GDPR, ISO 9001)

---

## 5. Input Validation

### DefectId
- Mandatory, max 36 characters
- Pattern: Alphanumeric + hyphen only
- Uniqueness: No duplicates allowed
- Error: "Defect ID must be alphanumeric, max 36 characters"

### Status
- Mandatory dropdown
- Valid values: 1–5 (from DefectStatus static entity)
- Error: "Please select a valid status"

### Severity
- Mandatory dropdown
- Valid values: Critical, High, Medium, Low
- Error: "Please select a severity level"

### ProductLine
- Mandatory, max 200 characters
- Pattern: Letters, numbers, spaces, hyphens only
- Error: "Product line must be max 200 characters"

### Description
- Mandatory, max 1000 characters
- Plain text (no HTML)
- Error: "Description must be 1–1000 characters"

### ReportedBy
- Mandatory, max 200 characters
- Pattern: Letters, numbers, spaces, hyphens only
- Error: "Reporter name must be max 200 characters"

### ResolutionDate (optional, update only)
- Must be ≤ today (no future dates)
- Error: "Resolution date cannot be in the future"

### Resolution (optional, update only)
- Max 500 characters
- Plain text
- Error: "Resolution must be max 500 characters"

---

## 6. Error Handling

| Scenario | UI Behavior | User Message |
|---|---|---|
| Database connection error | Show inline error, offer Retry | "Database error. Please try again." |
| Not found (deleted defect) | Show error, offer Back | "Defect not found or has been deleted" |
| Duplicate DefectId on create | Show inline error, keep form | "A defect with this ID already exists" |
| Unauthorized role access | Redirect or hide controls | "You do not have permission to perform this action" |
| Session timeout | Redirect to Login | "Your session has expired. Please log in again." |
| Validation failure | Show field-level error | Field-specific error message |

---

## 7. UI and Design

- **Theme:** OutSystems UI default theme
- **Layout:** LayoutTopMenu for all authenticated screens
- **Status badges:** Color-coded (Open=Blue, Under Investigation=Yellow, Escalated=Red, Resolved=Green, Closed=Grey)
- **Severity badges:** Color-coded (Critical=Red, High=Orange, Medium=Yellow, Low=Green)
- **Responsive:** Tablet and desktop (mobile not required for MVP)
- **Navigation:** Top menu with Defects link, New Defect link, logged-in user, Logout

---

## 8. Performance Considerations

- **Page load time:** < 2 seconds
- **List queries:** Index on Status, Severity, ProductLine, CreatedDate for fast filtering
- **Pagination:** 20 records per page
- **Debounce:** Text filters debounced by 300ms before triggering query
- **Caching:** DefectStatus static entity cached in memory (not queried every time)

---

## 9. Acceptance Criteria

- ✅ Users log in and are redirected to DefectList
- ✅ DefectList displays paginated, filterable table
- ✅ Clicking defect row opens DefectDetail with full info + history
- ✅ QualityEngineer can create new defect; all mandatory fields validated
- ✅ QualityManager can update defect status with optional resolution info
- ✅ QualityManager can soft-delete defect with confirmation; removed from list
- ✅ QualityEngineer cannot see Update Status or Delete buttons
- ✅ All errors surface as user-friendly inline messages (no raw errors)
- ✅ Duplicate DefectId on create shows error: "A defect with this ID already exists"
- ✅ Not-found defect shows error: "Defect not found or has been deleted"
- ✅ Role-based access control enforced in UI and server actions
- ✅ All form fields have client-side and server-side validation
- ✅ Pagination, filters work correctly
- ✅ Status and Severity badges display with correct colors
- ✅ Created date formatted as human-readable (YYYY-MM-DD HH:MM)
- ✅ Logout works, redirects to Login

---

## 10. Success Metrics

- **Defect list load time:** < 2 seconds
- **Create defect time:** < 3 seconds (form fill + submit)
- **User adoption:** 90% of team using within first week
- **Error rate:** < 1% of operations fail (excluding user input errors)
- **Uptime:** 99.5% availability during business hours
- **User satisfaction:** Positive feedback from QA team

---

## 11. Implementation Order

1. **Phase 1: Setup** (Day 1)
   - Create app in ODC Studio
   - Create data model (Defect, DefectHistory, DefectStatus)
   - Create Login screen

2. **Phase 2: List and Detail** (Days 2–3)
   - Create DefectList screen with table, filters, pagination
   - Create server actions: GetDefectList, GetDefectDetail, GetDefectHistory
   - Create DefectDetail screen
   - Test filtering and pagination

3. **Phase 3: Create and Update** (Days 4–5)
   - Create CreateDefect screen and server action
   - Create UpdateDefectStatus screen and server action
   - Implement validation (client + server)
   - Test duplicate handling

4. **Phase 4: Delete and Permissions** (Day 6)
   - Implement soft-delete with confirmation dialog
   - Implement role-based access control
   - Hide/show buttons based on role
   - Test permissions

5. **Phase 5: Error Handling and Polish** (Days 7–8)
   - Implement error handling for all scenarios
   - Add loading states and spinners
   - Add toast notifications
   - Refine UI/UX (colors, spacing, responsiveness)

6. **Phase 6: UAT and Deployment** (Days 9–10)
   - Execute manual test cases
   - Fix bugs and edge cases
   - Deploy to QA environment
   - Execute UAT
   - Deploy to production

---

## 12. Comparison: Monolith vs Microservices

| Aspect | Monolith (This BRD) | Microservices (QMSBackend + QMSWebapp) |
|---|---|---|
| **Apps** | 1 | 2 |
| **Database Access** | Direct | Via REST API |
| **Authentication** | Built-in OutSystems | X-API-Key + Built-in |
| **Complexity** | Lower | Higher |
| **Deployment** | Single publish | Two publishes + coordination |
| **Scalability** | Vertical (single app) | Horizontal (independent apps) |
| **Feature Parity** | 100% same | 100% same |
| **Testing** | Standard app testing | App testing + API testing |
| **When to Use** | Small teams, simple use case | Large teams, independent services needed |

---

## 13. When to Choose This Approach

✅ **Good fit:**
- Small to medium team (< 10 developers)
- Single deployment environment
- Features tightly coupled
- Operational simplicity is priority
- No need for independent scaling

❌ **Not a good fit:**
- Large distributed teams
- Need to scale components independently
- Different teams own different components
- Need to reuse API elsewhere
- High-frequency deployments of individual features

---

## 14. Notes

**No REST API:** Unlike the microservices version, this monolith has no exposed REST API. All operations are server-side actions called from the UI. If a separate consumer app needs access to QMS data in the future, you would:
1. Extract a backend service (refactor monolith → microservices)
2. Or add REST endpoints to this monolith (add API layer later)

**Same data model:** The Defect and DefectHistory entities are identical to the microservices version. Migration path exists if needed.

**Same features:** End-users experience the same functionality, screens, and workflows in both architectures.

---

## 15. Files & Resources

- **Primary BRD:** This document (monolithic architecture)
- **Alternative BRD:** `BRD-QMSBackendService-2026-09-27.md` (backend service)
- **Alternative BRD:** `BRD-QualityDefectManagement-2026-09-27.md` (web app)
- **Workshop:** See quality-defect-management workshop (same flows, adapted for monolith)
