---
title: "REST API Integration: Building Multi-App Architecture"
description: Learn REST API design, authentication, and multi-app integration patterns. Build a backend service that exposes REST APIs and a web application that consumes them. Demonstrates service-oriented architecture (SOA) and how OutSystems simplifies distributed app communication compared to traditional .NET approaches.
weight: 25
type: docs
layout: single
created: "2026-09-27"
updated: "2026-09-27"
authors: ["donnie"]
prerequisites: ["setting-up"]
duration: "240 minutes"
difficulty: "intermediate"
tags: ["api", "rest", "integration", "multi-app", "architecture", "backend", "authentication"]
---

## Overview

In this workshop, you'll build a **multi-app quality management system** that demonstrates **REST API design, authentication, and integration patterns**. You'll learn how to decompose a monolithic application into separate backend and frontend services that communicate via REST APIs.

You'll create two coordinated apps:
1. **QMS Backend Service** — A pure API backend that exposes 6 REST endpoints for defect data management with X-API-Key authentication
2. **QMS Web Application** — A web UI that consumes the backend APIs and provides a complete user interface for defect management

By the end, you'll understand service-oriented architecture (SOA) and how OutSystems simplifies distributed app communication compared to traditional .NET approaches.

## What You'll Learn

- How to design and implement a backend service with REST APIs (6 endpoints: list, detail, history, create, update, delete)
- How to expose APIs with proper authentication (X-API-Key pattern with Settings marked as Secret)
- How OutSystems REST connectors eliminate boilerplate HTTP client code
- How to test APIs comprehensively with Postman and automated Pytest test suites
- Service-oriented architecture (SOA) patterns: when to use multi-app vs monolithic
- Why OutSystems is 3x faster than traditional .NET for building and deploying integrated systems

## Prerequisites

- Complete {{< workshop slug="setting-up" label="Setting Up for OutSystems Workshops" >}}
- <a href="https://www.postman.com/downloads/" target="_blank">Postman</a> (or any REST client) installed
- Basic understanding of REST APIs and HTTP headers
- Access to OutSystems ODC Studio and ODC Portal

---

## Resources

Download and reference these Requirement Documents (BRDs) throughout the workshop:

### Backend Service
- **[BRD-QMSBackendService-2026-09-27.md](/outsystems-developer-workshops/workshops/rest-api-integration-multi-app/BRD-QMSBackendService-2026-09-27.md)** — Complete specification for the REST API backend service (data model, 6 endpoints, authentication, implementation notes)

### Web Application (Multi-App Architecture)
- **[BRD-QualityDefectManagement-2026-09-27.md](/outsystems-developer-workshops/workshops/rest-api-integration-multi-app/BRD-QualityDefectManagement-2026-09-27.md)** — Complete specification for the web app that consumes the REST API (6 screens, validation rules, error handling, testing strategy)

### Alternative: Monolithic Architecture
- **[BRD-QualityDefectManagement-Monolith-2026-09-27.md](/outsystems-developer-workshops/workshops/rest-api-integration-multi-app/BRD-QualityDefectManagement-Monolith-2026-09-27.md)** — Alternative single-app architecture (same features, no REST API layer). Use this to compare multi-app vs monolith approaches.

---

## Context

A manufacturing facility produces widgets on multiple assembly lines. Quality inspectors identify defects during testing, but currently:
- Defects are logged in an external Quality Management System (QMS)
- Investigation assignments are done manually via email
- Escalations are unreliable and untracked
- No audit trail of who approved what, when

Your job is to:
1. Build a **QMS Backend Service** that exposes the defect data via REST APIs
2. Build a **Quality Investigation app** that orchestrates the investigation workflow
3. Demonstrate how OutSystems handles the integration and workflow orchestration that would take weeks in traditional .NET

---

## Part 1: Building the QMS Backend Service

### Step 1: Download the BRD and Review

Before creating the app, download and review the requirement document:

1. Download: `BRD-QMSBackendService-2026-09-27.md` from this workshop's resources
2. Review the structure:
   - **Data Model:** Two entities (Defect, DefectHistory) with explicit relationships
   - **API Endpoints:** Four REST endpoints (List, Get, GetHistory, Update)
   - **Authentication:** X-API-Key header validation with OnAuthentication action
   - **Security:** API Key stored in Settings marked as Secret, different per stage

> 💡 **Tip**: The BRD explicitly states "backend service" and "NO UI components" — this teaches how Mentor handles backend-only apps, and how to prevent it from auto-generating unnecessary screens.

---

### Step 2: Create the Backend App in ODC Studio

1. Open <a href="https://www.outsystems.com/low-code-platform/developer-cloud" target="_blank">ODC Studio</a>
2. Click **Create** → Select **Generate with Mentor**
3. Name the app: **QMS-Backend**
4. Description: *"Backend service for manufacturing quality defect management. Exposes REST APIs for defect CRUD operations and authentication via X-API-Key header."*

---

### Step 3: Upload BRD to Mentor IDE

1. In Mentor IDE, click the **attachment icon** (clip symbol)
2. Select the downloaded `BRD-QMSBackendService-2026-09-27.md` file
3. Review Mentor's blueprint interpretation:
   - Check that data model shows: **Defect** and **DefectHistory** entities
   - Check that 4 API endpoints are recognized: List, Get, GetHistory, Update
   - Verify **NO screens** are being generated (backend-only warning should appear)

> ⚠️ **Warning**: If Mentor tries to generate screens, this is expected — the BRD explicitly says "no UI." In the blueprint review, you can delete any generated screens before approval. This is a quirk of how code generators work.

4. Review the blueprint and click **Approve**
5. Click **Generate** and wait for code generation (~2-3 minutes)

---

### Step 4: Publish the Generated App

1. Once generation completes, review the generated code structure:
   - **Data Model:** Defect and DefectHistory entities in the database tab
   - **REST Connectors:** Look for the exposed REST service actions
   - **Logic:** OnAuthentication action (may need manual creation — see Step 5)

2. Click **1-Click Publish** to publish to your development stage

> 💡 **Tip**: Keep ODC Studio open — you'll review a critical implementation pattern and add the OnAuthentication logic in the next steps.

---

### Step 4.5: Critical Implementation Pattern — Building List Responses

⚠️ **Important:** Before implementing the `ListDefects` and `FetchDefectHistory` server actions, you need to understand how to correctly build list responses in OutSystems. A common mistake makes endpoints return empty lists silently.

#### The Problem: ForEach Without ListAppend

When building a server action that returns a list, developers often make this mistake:

```
ForEach row in aggregate
  Assign OutputList.Current.Field = row.Field
```

**This silently returns an empty list** — the loop runs, but nothing is accumulated because `.Current` on an empty list has nowhere to write.

#### The Solution: ListAppend Inside ForEach

The correct pattern adds a blank item to the list BEFORE assigning field values:

```
ForEach row in aggregate
  Execute Action: ListAppend(OutputList, {})
  Assign OutputList.Current.Field = row.Field
```

**Step by step:**

1. **ForEach loop** iterates over your aggregate (e.g., GetDefects)
2. **Execute Action node** calls `ListAppend`
   - List argument: Your output list (e.g., `DefectList`)
   - Element argument: `{}` (empty record of the list's item type)
3. **Assign node** fills the fields of the newly appended item
   - `OutputList.Current.Field = row.Field`

#### Why {} Works

`{}` is an empty record literal in OutSystems. When passed to ListAppend:
- Creates a new blank item at the end of the list
- `.Current` now correctly points to that new slot
- Subsequent Assign nodes populate the item's fields

#### Actions Affected

These server actions must follow this pattern:

- **ListDefects** — builds DefectList from the GetDefects aggregate
- **FetchDefectHistory** — builds HistoryList from the GetHistoryByDefectId aggregate

> 💡 **Tip**: If you test your API endpoints and they return empty lists (200 OK but `[]` in response), this is likely the cause. Check that ListAppend is inside your ForEach loops.

---

### Step 5: Set Up API Authentication (Manual Post-Generation Step)

⚠️ **Important:** Mentor cannot auto-generate the OnAuthentication logic due to ODC's restricted expression scope. You must implement this **manually in ODC Studio** after the app is generated.

#### 5.1: Create the APIKey Setting in ODC Portal

1. Go to <a href="https://www.outsystems.com/low-code-platform/developer-cloud" target="_blank">ODC Portal</a>
2. Navigate to **Apps** → **QMS-Backend** → **Settings**
3. Click **Create Setting**
4. Configure:
   - **Name:** `APIKey`
   - **Type:** Text
   - **Access:** Mark as **Secret** (encrypts the value)
   - **Scope:** Application
   - **Value (Development stage):** Use this example key:
     ```
     dev_k9mZ7xQ2pL5vN8bR4tY1wE6jH3aS0cF7
     ```

> 🔑 **API Key Format & Generation**: 
> - Keys must be at least 32 characters long
> - Keys should contain random alphanumeric characters (use `openssl rand -hex 32` on Mac/Linux to generate)
> - Keys should have a stage prefix: `dev_`, `qa_`, `prod_` (so `dev_k9mZ7xQ2pL5vN8bR4tY1wE6jH3aS0cF7` is 32 chars post-prefix)
> - Never commit keys to version control
> - Store only in ODC Portal Settings (never in app code)
> 
> **To generate a random 32-character key on Mac/Linux:**
> ```bash
> openssl rand -hex 32
> ```
> Then prepend the stage prefix (e.g., `dev_` + output)

#### 5.2: Implement OnAuthentication in the REST API

⚠️ **Important:** You must add `Request_GetHeader` as a public element before using it.

**Step A: Add Request_GetHeader to your app**

1. In ODC Studio, click **Add public elements** (top toolbar) or press **Ctrl+Q**
2. Search for: `Request_GetHeader`
3. Select it from the results
4. Click **Add** to add it to your app
5. Now `Request_GetHeader` is available in your expression editor

**Step B: Edit the OnAuthentication callback**

1. In ODC Studio, navigate to the **REST API** section (under Integrations)
2. Find your exposed API (e.g., **QMSDefectAPI**)
3. Edit the **OnAuthentication** callback (built-in to the REST API)

**Step C: Build the authentication flow**

Inside the OnAuthentication flow, add:

   **i. Add a local variable**
   - Name: `ReceivedKey`
   - Type: `Text`

   **ii. Add an Execute Action node after Start**
   - ⚠️ **Critical:** `Request_GetHeader` is a server action (not a function) — it must be called via Execute Action
   - Select: `Request_GetHeader` from available actions
   - Input parameter: `Name = "X-API-Key"` (the header name to read)
   - Output: Creates result `GetAPIKeyHeader` with property `Value`

   **iii. Add an Assign node after the Execute Action**
   - Assignment: `ReceivedKey = GetAPIKeyHeader.Value`
   - This captures the header value from the action output into your variable

   **iv. Add an If node after the Assign**
   - Condition: `ReceivedKey = Settings.APIKey`
   - Use the expression editor's Settings scope tree to select `APIKey`
   - This compares the received header against the stored API key

   **v. Wire the branches**
   - **True branch** → End (request proceeds)
   - **False branch** → Raise Exception
     - Exception type: `Invalid Login` (OutSystems built-in, returns HTTP **401**)
     - Message: `"Invalid or missing X-API-Key header"`

6. Save and **Publish the app**

> ⚠️ **Critical Pattern for ODC**: 
> - Server actions like `Request_GetHeader` **must be called via Execute Action nodes**, not inline in expressions
> - Assign nodes are for variable assignment only (ReceivedKey = value)
> - If nodes use the assigned variables in conditions
> - This Execute Action → Assign → If pattern is the standard ODC flow for any server action that returns a value

---

### Step 6: Test API with cURL (Phase 1: List Defects)

First, verify that your API is responding correctly by testing the List Defects endpoint. It should return an empty list initially (or minimal data).

#### 6.1: Test GET /defects (List)

Replace `{your-domain}` with your actual OutSystems environment domain (e.g., `mycompany-dev.outsystems.app`).

```bash
curl -X GET "https://{your-domain}/QMSBackend/rest/QMSDefectAPI/defects?Page=1&PageSize=20" \
  -H "X-API-Key: dev_k9mZ7xQ2pL5vN8bR4tY1wE6jH3aS0cF7" \
  -H "Content-Type: application/json"
```

**Expected Response (HTTP 200):**
```json
[]
```

If you get:
- ✅ `[]` (empty array) — Correct! API is working.
- ❌ `401` — API key mismatch. Check that QMSAPIKey Setting matches backend's APIKey.
- ❌ `404` — Wrong base URL. Verify the domain and app name.

---

### Step 7: Create Sample Test Data Using POST

Now create some defects using the POST endpoint. These defects will be the data you test against in the next step.

#### 7.1: Using cURL to Create Defects

Copy and paste each cURL command below. Replace `{your-domain}` with your OutSystems domain.

**Defect 1:**
```bash
curl -X POST "https://{your-domain}/QMSBackend/rest/QMSDefectAPI/defects" \
  -H "X-API-Key: dev_k9mZ7xQ2pL5vN8bR4tY1wE6jH3aS0cF7" \
  -H "Content-Type: application/json" \
  -d '{
    "defectId": "DEF-2026-001",
    "status": 1,
    "severity": "Critical",
    "productLine": "Assembly Line A",
    "description": "Solder joint failure in PCB assembly",
    "reportedBy": "qa-user-001"
  }'
```

**Defect 2:**
```bash
curl -X POST "https://{your-domain}/QMSBackend/rest/QMSDefectAPI/defects" \
  -H "X-API-Key: dev_k9mZ7xQ2pL5vN8bR4tY1wE6jH3aS0cF7" \
  -H "Content-Type: application/json" \
  -d '{
    "defectId": "DEF-2026-002",
    "status": 1,
    "severity": "High",
    "productLine": "Widget Press B",
    "description": "Misalignment in press calibration",
    "reportedBy": "qa-user-002"
  }'
```

**Defect 3:**
```bash
curl -X POST "https://{your-domain}/QMSBackend/rest/QMSDefectAPI/defects" \
  -H "X-API-Key: dev_k9mZ7xQ2pL5vN8bR4tY1wE6jH3aS0cF7" \
  -H "Content-Type: application/json" \
  -d '{
    "defectId": "DEF-2026-003",
    "status": 2,
    "severity": "Medium",
    "productLine": "Paint Shop C",
    "description": "Color inconsistency in batch #5432",
    "reportedBy": "qa-user-001"
  }'
```

**Defect 4:**
```bash
curl -X POST "https://{your-domain}/QMSBackend/rest/QMSDefectAPI/defects" \
  -H "X-API-Key: dev_k9mZ7xQ2pL5vN8bR4tY1wE6jH3aS0cF7" \
  -H "Content-Type: application/json" \
  -d '{
    "defectId": "DEF-2026-004",
    "status": 3,
    "severity": "Critical",
    "productLine": "Assembly Line A",
    "description": "Missing component in final assembly",
    "reportedBy": "qa-user-003"
  }'
```

**Defect 5:**
```bash
curl -X POST "https://{your-domain}/QMSBackend/rest/QMSDefectAPI/defects" \
  -H "X-API-Key: dev_k9mZ7xQ2pL5vN8bR4tY1wE6jH3aS0cF7" \
  -H "Content-Type: application/json" \
  -d '{
    "defectId": "DEF-2026-005",
    "status": 4,
    "severity": "Low",
    "productLine": "Quality Lab",
    "description": "Measurement tool drift (recalibrated)",
    "reportedBy": "qa-user-002"
  }'
```

> ℹ️ **Note:** Status is an integer (1=Open, 2=Under Investigation, 3=Escalated, 4=Resolved, 5=Closed). See BRD for the full mapping.

> ✅ **Success:** Each POST should return HTTP 200 with the created defect object. If you get a 409 or error, check that the DefectId is unique.

---

#### 7.2: Using Postman (Alternative)

1. Create a new **POST** request in Postman
2. URL: `https://{your-domain}/QMSBackend/rest/QMSDefectAPI/defects`
3. Headers:
   - `X-API-Key`: `dev_k9mZ7xQ2pL5vN8bR4tY1wE6jH3aS0cF7`
   - `Content-Type`: `application/json`
4. Body (raw JSON):
   ```json
   {
     "defectId": "DEF-2026-001",
     "status": 1,
     "severity": "Critical",
     "productLine": "Assembly Line A",
     "description": "Solder joint failure in PCB assembly",
     "reportedBy": "qa-user-001"
   }
   ```
5. Click **Send** → You should get HTTP **200** with the created defect

---

### Step 8: Verify Data with GET /defects (Phase 2: Confirm Seeding)

---

Now verify that your POST operations succeeded by calling the List Defects endpoint again:

```bash
curl -X GET "https://{your-domain}/QMSBackend/rest/QMSDefectAPI/defects?Page=1&PageSize=20" \
  -H "X-API-Key: dev_k9mZ7xQ2pL5vN8bR4tY1wE6jH3aS0cF7" \
  -H "Content-Type: application/json"
```

**Expected Response (HTTP 200):**
```json
[
  {
    "defectId": "DEF-2026-001",
    "status": 1,
    "severity": "Critical",
    "productLine": "Assembly Line A",
    "createdDate": "2026-09-27T08:30:00Z",
    "reportedBy": "qa-user-001"
  },
  {
    "defectId": "DEF-2026-002",
    "status": 1,
    "severity": "High",
    "productLine": "Widget Press B",
    "createdDate": "2026-09-27T09:15:00Z",
    "reportedBy": "qa-user-002"
  },
  {
    "defectId": "DEF-2026-003",
    "status": 2,
    "severity": "Medium",
    "productLine": "Paint Shop C",
    "createdDate": "2026-09-27T09:30:00Z",
    "reportedBy": "qa-user-001"
  },
  {
    "defectId": "DEF-2026-004",
    "status": 3,
    "severity": "Critical",
    "productLine": "Assembly Line A",
    "createdDate": "2026-09-27T10:00:00Z",
    "reportedBy": "qa-user-003"
  },
  {
    "defectId": "DEF-2026-005",
    "status": 4,
    "severity": "Low",
    "productLine": "Quality Lab",
    "createdDate": "2026-09-27T10:15:00Z",
    "reportedBy": "qa-user-002"
  }
]
```

> ✅ **Success:** You now have 5 defects in the system. The API is working correctly. The remaining endpoints (GET detail, GET history, PATCH update, DELETE) are tested in the automated test suite (see next section).
```

#### 7.6: Test Endpoint 5: Delete Defect

**Request Details:**

```
Method: DELETE
URL: https://{your-domain}/QMSBackend/rest/QMSDefectAPI/defects/DEF-2026-005
Headers:
  X-API-Key: {{api_key}}
```

**In Postman:**

1. Create a new **DELETE** request
2. URL: `https://[your-env].outsystems.app/qms-backend/rest/defects/DEF-2026-005`
3. Add **X-API-Key** header: `{{api_key}}`
4. Click **Send**

**Expected Response (204 No Content):**
- Status: **204 No Content**
- Body: Empty (no response body for successful deletion)

> ℹ️ **Info**: The defect is soft-deleted (archived) for audit compliance, not permanently removed from the database.

#### 7.7: Verify Authentication (Negative Test)

**Request Details:**

```
Method: GET
URL: https://{your-domain}/QMSBackend/rest/QMSDefectAPI/defects
Headers:
  X-API-Key: invalid_key_12345
```

**In Postman:**

1. Duplicate the "List Defects" request
2. Change the `X-API-Key` header value to: `invalid_key_12345`
3. Click **Send**

**Expected Response (401 Unauthorized):**

```json
{
  "error": "Invalid or missing X-API-Key header"
}
```

> ✅ **Success**: All 6 endpoints return correct status codes (200, 201, 204, 404, 401) and proper authentication is enforced.

---

## Summary: QMS Backend Service Complete ✅

You've now:
- ✅ Created a backend service using Mentor App Generator
- ✅ Exposed 6 REST API endpoints:
  - GET /defects (List with pagination & filters)
  - GET /defects/{defectId} (Get single defect)
  - GET /defects/{defectId}/history (Get related defects)
  - POST /defects (Create new defect)
  - PATCH /defects/{defectId} (Update status)
  - DELETE /defects/{defectId} (Soft-delete defect)
- ✅ Implemented X-API-Key authentication with Settings marked as Secret
- ✅ Created sample data using the POST endpoint
- ✅ Tested all endpoints in Postman with valid and invalid authentication

**The QMS Backend Service is complete and ready to be consumed by the Quality Investigation app.**

---

## Next: Quality Investigation App (Coming Soon)

In Part 2, you'll build the UI + workflow app that:
- Consumes these APIs
- Implements the 9-step BPM workflow
- Shows how OutSystems orchestrates async operations without manual state management
- Demonstrates role-based access control

**Ready to proceed to Part 2?**
