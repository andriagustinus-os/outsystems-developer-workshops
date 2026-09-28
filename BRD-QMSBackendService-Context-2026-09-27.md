# QMS Backend Service — Context & Non-Functional Requirements

**Date:** 2026-09-27  
**Status:** Draft (Ready for Mentor Generation)  
**File:** Use only for reference — DO NOT upload to Mentor

This document provides background context and technical details. The **primary BRD** for Mentor upload is `BRD-QMSBackendService-2026-09-27.md`.

---

## Executive Summary

The QMS Backend Service is a pure data management service with no user interface. Its role is to:
1. **Store defect data** in a relational database
2. **Expose REST APIs** for other applications to consume
3. **Demonstrate multi-app architecture** (separate backend from UI/workflow)

In the context of the workshop, this service demonstrates:
- **Separation of concerns:** Backend service ≠ UI app
- **API-driven architecture:** Other apps consume clean REST interfaces
- **Data encapsulation:** All database access goes through APIs
- **Enterprise pattern:** How OutSystems scales with multiple coordinated apps

This service is consumed by the **Quality Investigation app** (the UI/workflow app).

---

## Problem Statement

Manufacturing quality data lives in many silos:
- **Current state:** External QMS system has data, but it's not integrated into workflow apps
- **Gap:** UI apps must call external APIs directly, or custom code must be written to encapsulate API calls
- **Challenge:** Without a backend service, each consuming app duplicates API integration logic

**Solution:** Create a dedicated QMS Backend Service that:
- Centralizes all quality defect data access
- Provides clean, consistent REST APIs
- Acts as a single source of truth
- Can be reused by multiple apps

---

## Technical Architecture

```
┌─────────────────────────────────────────────────────┐
│ Quality Investigation App (Web UI + Workflows)       │
│ - User interface                                     │
│ - Investigation workflows                           │
│ - Calls QMS Backend APIs                            │
└───────────────────────┬─────────────────────────────┘
                        │ REST API calls
        ┌───────────────▼──────────────┐
        │ QMS Backend Service          │
        │ - Data model (Defect entity) │
        │ - REST APIs (3 endpoints)    │
        │ - Database access logic      │
        │ - No screens, no UI          │
        └───────────────┬──────────────┘
                        │ Database
        ┌───────────────▼──────────────┐
        │ OutSystems Database          │
        │ - Defect table               │
        │ - DefectHistory table        │
        │ - Audit logs                 │
        └──────────────────────────────┘
```

**Why this architecture:**
- **Scalability:** Multiple apps can consume the same backend service
- **Maintainability:** API changes managed in one place (backend service)
- **Security:** API-level authentication/authorization
- **Performance:** Caching can be done at API level
- **Flexibility:** Backend can evolve independently of UI apps

---

## API Design Details

### Authentication & Security

- **Authentication method:** API Key (header: `X-API-Key`) or Bearer token
- **CORS:** Enable cross-origin requests (for web apps)
- **Rate limiting:** Optional (not required for MVP)
- **Encryption:** TLS 1.3 for all HTTPS connections

### Data Formats

**Request/Response Content-Type:** application/json

**Defect object (JSON response):**
```json
{
  "defectId": "DEF-2026-001",
  "status": "Open",
  "severity": "Critical",
  "productLine": "Widget Assembly Line A",
  "description": "Solder joint failure in PCB assembly",
  "createdDate": "2026-09-27T08:30:00Z",
  "updatedDate": "2026-09-27T08:30:00Z",
  "reportedBy": "qa-user-001",
  "resolutionDate": null
}
```

**DefectHistory array (JSON response):**
```json
[
  {
    "historyId": "HIST-001",
    "defectId": "DEF-2026-001",
    "previousDefectId": "DEF-2025-145",
    "severity": "High",
    "resolutionDate": "2025-10-15T14:22:00Z",
    "resolution": "Reflow temperature corrected; passed re-test",
    "recurrenceCount": 3
  }
]
```

### Error Responses

**400 Bad Request:**
```json
{
  "error": "Invalid status value",
  "detail": "Status must be one of: Open, Under Investigation, Escalated, Resolved, Closed"
}
```

**404 Not Found:**
```json
{
  "error": "Defect not found",
  "detail": "DefectId DEF-2026-999 does not exist"
}
```

**500 Server Error:**
```json
{
  "error": "Internal server error",
  "detail": "Database connection failed. Please try again later."
}
```

---

## Non-Functional Requirements

### Performance

- **API Response Time:**
  - GET endpoints: < 500ms (95th percentile)
  - PATCH endpoint: < 1 second (includes database write)
  - Concurrent requests: Support 10+ simultaneous API calls

- **Database:**
  - Initial data volume: 1,000–5,000 defects
  - Growth: 50,000+ defects over 2 years
  - Indexes on: DefectId (PK), Status, Severity, CreatedDate, ProductLine

### Reliability & Availability

- **Uptime SLA:** 99.5% during business hours
- **Database backup:** Daily automatic backups
- **Data loss tolerance:** < 15 minutes (RPO)
- **Recovery time:** < 1 hour (RTO)

### Security & Compliance

- **Data classification:** Confidential (manufacturing defect data)
- **Encryption:**
  - At rest: AES-256 (OutSystems managed)
  - In transit: TLS 1.3
  - API credentials: Stored encrypted in OutSystems

- **Access control:**
  - API Key authentication required
  - Consuming apps verify API Key before calling
  - No direct database access (only via API)

- **Audit trail:**
  - All PATCH operations logged with timestamp + user
  - Soft-delete supported (archive old records)

- **Compliance:**
  - Manufacturing regulatory: ISO 9001
  - Data retention: 7 years
  - GDPR-compliant (supports right to be forgotten via soft-delete)

---

## Implementation Notes

### Database Schema

**Defect table:**
```
DefectId (PK, Text)
Status (FK to DefectStatus, Text)
Severity (Text)
ProductLine (Text)
Description (Text)
QMSExternalId (Text, nullable)
CreatedDate (DateTime)
UpdatedDate (DateTime)
ReportedBy (Text)
ResolutionDate (DateTime, nullable)
IsDeleted (Boolean, soft-delete flag)
```

**DefectHistory table:**
```
HistoryId (PK, Text)
DefectId (FK to Defect, Text)
PreviousDefectId (Text)
Severity (Text)
ResolutionDate (DateTime)
Resolution (Text)
RecurrenceCount (Integer)
```

### API Implementation in OutSystems

**Server Actions (Public, exposed as REST):**
- `GetDefect(defectId: Text) → DefectRecord`
- `GetDefectHistory(defectId: Text) → List<DefectHistoryRecord>`
- `UpdateDefectStatus(defectId: Text, newStatus: Text, resolutionDate: DateTime, resolution: Text) → DefectRecord`

**Service Action Structure:**
- Input parameters mapped to request body/path
- Output structures for JSON serialization
- Error handling with appropriate HTTP status codes

---

## Dependencies

### Internal Dependencies

- **Quality Investigation app** consumes this service's APIs

### External Dependencies

- **OutSystems database:** Data persists here (no external database)
- **Azure AD:** Optional, if token-based auth needed (future enhancement)

---

## Testing & Acceptance

### API Testing

- [ ] GET /defects/{id} returns correct defect data
- [ ] GET /defects/{id} returns 404 when defect doesn't exist
- [ ] GET /defects/{id}/history returns empty array if no history
- [ ] PATCH /defects/{id} updates status successfully
- [ ] PATCH /defects/{id} rejects invalid status values (400)
- [ ] All endpoints return proper HTTP status codes
- [ ] API response time < 500ms under normal load
- [ ] Concurrent 10 requests handled without errors

### Integration Testing

- [ ] Quality Investigation app successfully calls all 3 endpoints
- [ ] Quality Investigation app receives correct data from API
- [ ] Quality Investigation app's API errors handled gracefully

---

## Post-MVP Enhancements

- **Phase 2:** Add batch endpoints (GET /defects/batch)
- **Phase 3:** Add filtering/pagination to history endpoint
- **Phase 4:** Add webhook support (notify consuming apps of status changes)
- **Phase 5:** Add full-text search on defect descriptions

---

## Build Order (with Quality Investigation App)

1. **Week 1:** 
   - Generate QMS Backend Service from this BRD
   - Test APIs in isolation
   - Publish to development environment

2. **Week 1-2:**
   - Generate Quality Investigation app (consumes QMS APIs)
   - Integration testing between apps
   - Refine both apps as needed

3. **Week 2-3:**
   - UAT
   - Deploy to production

---

## Known Platform Constraints & Troubleshooting

### OnAuthentication Implementation

**Why you may see OS-BEW-50000 or OS-DPL-50205 errors:**

If Mentor attempts to auto-generate the full OnAuthentication logic (reading headers + Settings), the generated code will fail to compile because:

1. **OS-BEW-50000** — Builder Web error: OnAuthentication references undefined functions or Settings outside its restricted scope
2. **OS-DPL-50205** — Deployment error: Compilation fails due to invalid expressions in the OnAuthentication action

**Solution:**

Don't attempt to auto-generate the OnAuthentication logic. Instead:
1. Let Mentor generate the `APIKey` Setting ✅
2. Let Mentor generate the REST API endpoints ✅
3. **Manually implement the OnAuthentication action** in ODC Studio ✅ (see BRD section "Platform Implementation Constraints")

The BRD is intentionally written to separate the auto-generatable parts (data model, API endpoints, Settings) from the manually-completed parts (authentication logic).

**Note on OnRequest vs. OnAuthentication:**

When you set `Authentication = Custom` on the REST API, ODC automatically generates both callbacks:
- **OnAuthentication** — enforces the actual authentication (implement your header validation here)
- **OnRequest** — a pass-through request-transformation hook that runs before every method call (irrelevant to authentication, can be left as-is)

Only OnAuthentication matters for API key validation. Do not confuse the two callbacks.

### Why This Design?

This constraint reflects how ODC's runtime works:
- OnAuthentication runs **before** the normal app runtime initializes
- It operates in a sandboxed scope to prevent access to uninitialized resources
- Settings and dynamic functions are not accessible in this early stage
- The workaround is minimal: implement the logic once, then it's done

---

## Sign-Off

**Document Owner:** Donnie Prakoso  
**Date:** 2026-09-27  
**Status:** Ready for Mentor Generation  

**Build order:**
1. ✅ Generate QMS Backend Service from `BRD-QMSBackendService-2026-09-27.md`
2. ✅ Generate Quality Investigation app from `BRD-QualityDefectManagement-2026-09-27.md` (consumes this service)
3. ✅ Integration test between apps
4. ✅ Demo in workshop
