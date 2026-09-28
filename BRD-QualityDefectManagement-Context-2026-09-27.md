# Quality Defect Management App — Context & Non-Functional Requirements

**Date:** 2026-09-27  
**Status:** Draft (Ready for Mentor Generation)  
**File:** Use only for reference — DO NOT upload to Mentor

This document provides background context, compliance requirements, and success metrics. The **primary BRD** for Mentor upload is `BRD-QualityDefectManagement-2026-09-27.md`.

---

## Executive Summary

The Quality Defect Management app demonstrates how OutSystems excels at combining **real-time API integration** with **BPM workflow orchestration**. For enterprise engineers evaluating OutSystems vs. traditional .NET/React stacks, this app highlights:

1. **API Integration:** How OutSystems REST connectors handle external data without boilerplate HTTP client code
2. **Async Patterns:** How workflows manage state persistence without manual async/await or custom task scheduling
3. **Human Tasks:** Built-in task assignment, escalation, and audit logging (vs. custom database task tables in .NET)
4. **Productivity:** Development speed — what takes weeks in traditional code takes hours in OutSystems

**Business value:** Reduces defect investigation time from 4 hours to <1 hour through intelligent routing, auto-escalation, and real-time QMS data enrichment.

---

## Problem Statement

Manufacturing facilities struggle with quality defect management:
- **Current state:** Manual Excel tracking, email escalations, no audit trail
- **Pain points:**
  - Defect resolution takes 4+ hours (high process overhead)
  - No visibility into investigation status (supervisors chase via email)
  - Compliance risk: no audit trail of who approved what, when
  - Manual escalation decisions (late escalations → production delays)

- **Desired state:** Real-time defect tracking, automated routing & escalation, audit compliance, reduced resolution time

---

## Target Users & Scale

**Primary users:**
- QA Team: 5–10 investigators at a manufacturing facility
- Supervisors: 2–3 quality supervisors
- Admin: 1 platform admin

**Scale:**
- Concurrent users: 15 (all shifts)
- Defects/day: 50–100
- Investigation tasks/day: 50–100
- Peak load: 10–15 concurrent investigations
- Data retention: 7 years (regulatory requirement)

---

## Technical Context

**Why OutSystems for this use case:**

| Requirement | .NET Approach | OutSystems Approach | Winner |
|---|---|---|---|
| REST API integration | HttpClient + async/await + error handling | Built-in REST connector, auto-retry | OutSystems (less code) |
| Task assignment & tracking | Custom task table + SQL queries + role-checking code | Native human activity nodes + auto-assignment | OutSystems (30x less code) |
| Escalation logic | Timer job polling + scheduled task + custom notifications | Workflow conditional start + time-based escalation | OutSystems (no job setup) |
| State persistence | Database + event sourcing pattern | Workflow instance table (built-in) | OutSystems (automatic) |
| Audit trail | Custom logging table + triggers + code | Built-in workflow audit logs | OutSystems (automatic compliance) |
| Deployment | Docker + Kubernetes + CI/CD pipeline | One-click publish in OutSystems | OutSystems (speed) |
| Development time (MVP) | 6–8 weeks (team of 2) | 2–3 weeks (single developer) | OutSystems (3x faster) |

---

## Non-Functional Requirements

### Performance & Scalability

- **Response time SLA:**
  - API fetch (QMS defect details): < 2 seconds
  - Screen load (Defect List): < 3 seconds
  - Investigation task submission: < 1 second
  - Dashboard KPI update: < 5 seconds

- **Throughput:**
  - Support 50–100 defects/day
  - 10–15 concurrent investigations
  - 5 concurrent dashboard views

- **Database:**
  - Initial size: ~10,000 defects (year 1)
  - Growth: 50,000+ defects (year 3)
  - Indexing: On Defect.Status, Defect.Severity, Defect.CreatedDate, InvestigationTask.AssignedTo
  - Partitioning: Optional by year for historical data (post-MVP)

### Reliability & Availability

- **Uptime SLA:** 99.5% during business hours (8am–6pm, Mon–Fri)
- **Acceptable downtime:** < 43 minutes/month
- **RTO (Recovery Time Objective):** < 1 hour
- **RPO (Recovery Point Objective):** < 15 minutes (max data loss)
- **Backup:** Daily automatic backups retained for 30 days

### Security & Compliance

- **Data classification:** Defect data = Confidential (internal only)
- **Encryption:**
  - At rest: AES-256 (OutSystems managed)
  - In transit: TLS 1.3 (HTTPS required)
  - API credentials: Stored encrypted in ODC (not in code)

- **Authentication:**
  - Single Sign-On (SSO) via company directory (Azure AD)
  - Session timeout: 30 minutes of inactivity
  - MFA: Optional for admin accounts

- **Authorization:**
  - Role-based access control (RBAC): QA Team, Supervisor, Admin
  - Row-level security: QA Team sees only assigned defects
  - No sharing of defect data outside assigned role

- **Compliance & Audit:**
  - Manufacturing regulatory: ISO 9001 (quality management)
  - Data residency: All defect data stored in US region (company policy)
  - Audit trail: All status changes, approvals, updates logged with timestamp + user
  - Data retention: 7 years (per manufacturing regulations)
  - Right to be forgotten: Soft-delete supported (archive instead of hard-delete)
  - Log retention: Audit logs retained for 10 years
  - Compliance reports: Exportable for internal/external audits

---

## Integration Architecture

```
┌─────────────────────────────────────────┐
│ External Quality Management System (QMS) │
│ (e.g., quality-control.com QMS platform) │
└────────────────┬────────────────────────┘
                 │
         REST API (over HTTPS)
                 │
    ┌────────────▼─────────────┐
    │ OutSystems ODC App       │
    │ ├─ Defect entity         │
    │ ├─ Investigation workflow │
    │ ├─ REST connector        │
    │ └─ BPM orchestration     │
    └────────────┬─────────────┘
                 │
    ┌────────────▼─────────────┐
    │ Local Database           │
    │ (OutSystems managed)     │
    │ ├─ Defects (cached)      │
    │ ├─ Investigation tasks   │
    │ └─ Audit trail           │
    └─────────────────────────┘
```

**Data sync strategy:**
- Fetch on-demand: When defect details page opens, call QMS API to get latest data
- Cache locally: Store in local Defect entity for quick access
- Sync back: When investigation completes, update QMS with resolution status
- Offline fallback: If QMS API unavailable, use cached data + notify user

---

## Dependencies & Risks

### External Dependencies

| System | Dependency | Criticality | Mitigation |
|---|---|---|---|
| QMS API | Provides defect data | Critical | Fallback to cached data; notify user if API down |
| Azure AD | User authentication | Critical | Manual local user creation if SSO unavailable (emergency only) |
| Email service | Send escalation notifications | High | Queue notifications; retry if email fails |
| ODC platform | Hosting + deployment | Critical | Use ODC's built-in disaster recovery |

### Technical Risks

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| **QMS API downtime** | Medium | High | Implement retry logic with exponential backoff; cache recent data |
| **High defect volume (spike)** | Low | Medium | Implement query pagination; use indexes; monitor database performance |
| **Concurrent task conflicts** | Low | Low | Use optimistic locking on task updates |
| **Workflow state corruption** | Very Low | Critical | OutSystems auto-manages workflow state; no custom code needed |
| **Audit trail gaps** | Very Low | Critical | Built-in workflow audit logs; verify in testing |

---

## Testing & Acceptance Criteria

### Functional Testing

- [ ] Defect list loads and displays all open defects
- [ ] Defect details page fetches data from QMS within 2 seconds
- [ ] QA Team can create new investigation task
- [ ] Investigation task shows up in QA Team's task list (role-based filtering)
- [ ] QA Team can complete investigation, add findings
- [ ] Supervisor receives notification when defect is escalated
- [ ] Supervisor can approve/reject escalation with comments
- [ ] Investigation workflow auto-escalates after 2 hours (test with timer override)
- [ ] QMS API update succeeds when investigation is marked resolved
- [ ] Audit trail shows all changes (who, when, what)

### Performance Testing

- [ ] Dashboard KPIs load in < 5 seconds with 100 defects
- [ ] Defect list loads in < 3 seconds with pagination
- [ ] API call to QMS completes in < 2 seconds (95th percentile)
- [ ] Concurrent 10 investigations don't degrade performance

### Security Testing

- [ ] QA Team cannot view other investigators' assigned defects
- [ ] Supervisor cannot edit investigation findings
- [ ] Admin role can view/edit/delete all data
- [ ] SSO login works; logout clears session
- [ ] API credentials not visible in app code

### Compliance Testing

- [ ] Audit trail captures all status changes
- [ ] Data is encrypted in transit (SSL/TLS)
- [ ] Soft-delete (archive) works for historical data
- [ ] Export of audit trail works for compliance review

---

## Workshop Learning Objectives

**For the attendees (experienced .NET/React developers):**

1. **Understand OutSystems REST integration** vs. HttpClient in .NET
   - No boilerplate HTTP setup
   - Built-in retry + error handling
   - Connection credentials managed securely in ODC

2. **Understand OutSystems workflows** vs. async/await + custom task management
   - No manual state persistence (workflow engine handles it)
   - Human tasks are first-class (no database task table needed)
   - Auto-escalation rules without scheduled jobs

3. **See the productivity gain**
   - Demo: Same app in .NET takes 6–8 weeks; OutSystems takes 2–3 weeks
   - No framework setup, no database migrations, no deployment pipeline setup

4. **Compare error handling patterns**
   - OutSystems: Declarative (set retry count, fallback data)
   - .NET: Imperative (try/catch + retry loops + logging)

5. **Understand audit compliance**
   - Built-in workflow audit logs (no custom logging code needed)
   - Perfect for regulated industries

---

## Success Metrics (Post-Launch)

| Metric | Target | Measurement |
|---|---|---|
| Defect investigation time | < 1 hour average | Time from assignment → resolution |
| Investigation completion rate (without escalation) | > 80% | % of investigations resolved by QA (no supervisor escalation) |
| Escalation resolution time | < 30 minutes | Time from escalation → supervisor approval |
| Audit trail completeness | 100% | All status changes logged; no gaps |
| User adoption | > 90% within 2 weeks | % of QA Team actively using |
| System uptime | 99.5% | Monitored by ODC; monthly report |
| QMS API success rate | > 98% | Successful API calls / total calls |

---

## Post-MVP Enhancements

- **Phase 2:** Mobile app for field investigators
- **Phase 3:** Predictive analytics (ML model to predict defect severity)
- **Phase 4:** Integration with ERP for material traceability
- **Phase 5:** Advanced reporting & dashboards (Tableau/Power BI)

---

## Team & Timeline

**Development team:** 1 OutSystems developer (for MVP generation)

**Timeline:**
1. **Week 1:** Mentor generates app from BRD → review blueprint → generate code
2. **Week 2:** Manual refinements in ODC Studio (custom business logic, validation)
3. **Week 3:** Testing, UAT, deploy to production

**Total:** 3 weeks (vs. 6–8 weeks for traditional .NET approach)

---

## Sign-Off

**Document Owner:** Donnie Prakoso  
**Date:** 2026-09-27  
**Status:** Ready for Mentor Generation  

**Next steps:**
1. ✅ Review this context document
2. ✅ Upload `BRD-QualityDefectManagement-2026-09-27.md` to Mentor App Generator
3. ⏳ Let Mentor generate the app scaffold
4. ⏳ Review the blueprint and approve
5. ⏳ Generate code
6. ⏳ Manual refinements in ODC Studio
7. ⏳ Testing & UAT
