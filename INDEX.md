# SMS TDR Repository — Document Index

## Quick Navigation

This index helps you find the right TDR document based on your context, use case, or topic of interest.

---

## By Implementation Phase

### 📋 **Starting Out — Understanding the System**
- **Read First**: `prompt.md` — Meta-prompt explaining the TDR philosophy and usage
- **Overview**: `table.md` — Complete table/entity roadmap by steps
- **Big Picture**: `TDR-014-architecture-decision-records.md` — 12 key architectural decisions

### 🏗️ **Design Phase — Domain Modeling**
1. `TDR-001-domain-rules-master-referentials.md` — Business rules for geographic & academic structure
2. `TDR-003-domain-rules-school-structure.md` — SchoolYear, SchoolYearCategory, Enrollment

### 📊 **Data Design Phase — Database Architecture**
3. `TDR-002-data-architecture-master-tables.md` — DDL, indexes, denormalization for masters
4. `TDR-005-data-architecture-evaluations-grades.md` — DDL for grades & evaluation tables
5. `TDR-007-data-architecture-timetables.md` — DDL for scheduling tables

### 🔧 **Service Design Phase**
6. `TDR-009-domain-rules-service-layer.md` — All service operations (Enrollment, Grade, Timetable, etc.)
7. `TDR-011-selectors-queries.md` — Repository patterns, query implementations, cache strategy

### 🌐 **API & Integration Phase**
8. `TDR-010-api-contracts.md` — REST endpoints, GraphQL schema, webhooks, error codes

### 🔄 **Offline Capability Phase**
9. `TDR-012-offline-sync-strategy.md` — Complete sync protocol, conflict resolution, queue management

### 📖 **Use Case Validation Phase**
10. `TDR-013-service-scenarios.md` — 10 narrative scenarios with acceptance criteria

### 👥 **User & Security Phase**
11. `TDR-008-domain-rules-user-roles.md` — CustomUser, profiles, RBAC

---

## By Topic / Question

### "How does enrollment work?"
→ **Start here**: `TDR-003-domain-rules-school-structure.md` (Enrollment domain rules)
→ **Then**: `TDR-009-domain-rules-service-layer.md` (EnrollmentService operations)
→ **See it in action**: `TDR-013-service-scenarios.md` (Scenarios 1, 5)

### "How are grades secured and immutable?"
→ **Rules**: `TDR-004-domain-rules-evaluations-grades.md` (Grade immutability via appending)
→ **Schema**: `TDR-005-data-architecture-evaluations-grades.md` (triggers, constraints)
→ **Operations**: `TDR-009-domain-rules-service-layer.md` (GradeService)
→ **Appeals**: `TDR-013-service-scenarios.md` (Scenario 6)

### "How does offline sync work?"
→ **Strategy**: `TDR-012-offline-sync-strategy.md` (complete sync architecture)
→ **Queue design**: `TDR-012` Section 4 (offline queue schema)
→ **Conflict resolution**: `TDR-012` Section 6 (resolution policies)
→ **See it**: `TDR-013-service-scenarios.md` (Scenarios 2, 7)

### "What's the timetable design?"
→ **Rules**: `TDR-006-domain-rules-timetables.md` (conflict prevention, exceptions)
→ **Schema**: `TDR-007-data-architecture-timetables.md` (DDL, indexes, triggers)
→ **Operations**: `TDR-009` (TimetableService)
→ **Conflict example**: `TDR-013-service-scenarios.md` (Scenario 4)

### "What are the core architectural decisions?"
→ `TDR-014-architecture-decision-records.md` (12 ADRs)

### "What APIs are available?"
→ `TDR-010-api-contracts.md` (REST endpoints, GraphQL, error codes)

### "How do we query efficiently?"
→ `TDR-011-selectors-queries.md` (Selector pattern, materialized views, N+1 prevention)

### "How does user authentication and roles work?"
→ `TDR-008-domain-rules-user-roles.md` (CustomUser, profiles, RBAC)

### "How is long-term data retention handled?"
→ `TDR-002` & `TDR-005` & `TDR-007` (partitioning strategies)
→ `TDR-009` (archival processes in services)

---

## By Entity / Table

| Table / Entity | Primary Doc(s) | Also See |
|----------------|----------------|----------|
| SchoolYear | TDR-003, TDR-002 | TDR-009 (EnrollmentService) |
| SchoolYearCategory | TDR-003 | - |
| SchoolYearCategoryTerm | TDR-003 | - |
| Enrollment | TDR-003, TDR-001 | TDR-009, TDR-013 (Scenarios 1, 5) |
| Classroom | TDR-003 | - |
| ClassroomSubject | TDR-001 (SubjectChoice) | TDR-006 (timetable) |
| Evaluation | TDR-004, TDR-005 | TDR-009 (GradeService) |
| Grade | TDR-004, TDR-005 | TDR-009, TDR-012, TDR-013 (Scenario 6) |
| Timetable | TDR-006, TDR-007 | TDR-009, TDR-013 (Scenario 4) |
| TimetableEntry | TDR-006, TDR-007 | - |
| CustomUser | TDR-008 | TDR-010 (API) |
| Teacher/Student/Parent Profiles | TDR-008 | - |
| Region/Prefecture/etc. | TDR-001, TDR-002 | - |
| Category/Level/Track | TDR-001 | - |
| Subject/SubjectChoice | TDR-001 | - |

---

## By Role / User Type

### **For System Architects**
- Start: `TDR-014-architecture-decision-records.md`
- Then: `TDR-012-offline-sync-strategy.md` (critical complexity)
- Review: `TDR-009-domain-rules-service-layer.md`

### **For Backend Developers (Services)**
1. `TDR-009-domain-rules-service-layer.md` (what to implement)
2. `TDR-011-selectors-queries.md` (how to query efficiently)
3. `TDR-004-domain-rules-evaluations-grades.md` (business rules)

### **For Database Engineers**
1. `TDR-002-data-architecture-master-tables.md`
2. `TDR-005-data-architecture-evaluations-grades.md`
3. `TDR-007-data-architecture-timetables.md`
4. `TDR-011-selectors-queries.md` (index guidelines)

### **For API Engineers**
- `TDR-010-api-contracts.md` (complete API spec)
- `TDR-009-domain-rules-service-layer.md` (service interfaces)
- `TDR-008-domain-rules-user-roles.md` (authentication/authorization)

### **For Mobile Developers (Offline App)**
1. `TDR-012-offline-sync-strategy.md` (critical!)
2. `TDR-010-api-contracts.md` (sync endpoint, idempotency)
3. `TDR-009-domain-rules-service-layer.md` (operations to support offline)
4. `TDR-013-service-scenarios.md` (Scenario 7: daily teacher workflow)

### **For QA / Test Engineers**
- `TDR-013-service-scenarios.md` (acceptance criteria)
- `TDR-011-selectors-queries.md` (query expectations)
- `TDR-012-offline-sync-strategy.md` (sync testing scenarios)

---

## Document Metadata Summary

| Doc | Type | Pages | Priority | Audience |
|-----|------|-------|----------|----------|
| TDR-001 | Domain Rules | ~60 | High | Architects, Devs |
| TDR-002 | Data Architecture | ~40 | High | DB Engineers |
| TDR-003 | Domain Rules | ~50 | High | Arch, Devs |
| TDR-004 | Domain Rules | ~50 | High | Arch, Devs |
| TDR-005 | Data Architecture | ~35 | High | DB Engineers |
| TDR-006 | Domain Rules | ~30 | Medium | Arch, Devs |
| TDR-007 | Data Architecture | ~25 | Medium | DB Engineers |
| TDR-008 | Domain Rules | ~45 | High | Arch, Devs |
| TDR-009 | Service Layer | ~60 | High | Backend Devs |
| TDR-010 | API Contracts | ~45 | High | API Engineers |
| TDR-011 | Selectors & Queries | ~35 | Medium | Backend Devs |
| TDR-012 | Offline Sync Strategy | ~60 | **Critical** | Mobile, Backend |
| TDR-013 | Scenarios | ~40 | High | QA, All |
| TDR-014 | ADRs | ~25 | Reference | Architects |

---

## Cross-Reference Quick Lookup

| You Need... | Read This |
|-------------|-----------|
| **Database schema for X** | TDR-00X (data architecture for that domain) |
| **Business rules for X** | TDR-00X (domain rules for that domain) |
| **How to implement the service** | TDR-009 |
| **API endpoint for X operation** | TDR-010 (search by operation) |
| **How to write performant query** | TDR-011 |
| **Offline capability guarantees** | TDR-012 |
| **Test scenarios to verify** | TDR-013 |
| **Why was this decision made?** | TDR-014 (search ADR title) |

---

## Reading Order Templates

### **Template A: New Architect**
```
Week 1:
  - prompt.md
  - table.md
  - TDR-014 (skim ADRs)
  - TDR-001
  - TDR-003

Week 2:
  - TDR-002
  - TDR-004
  - TDR-005

Week 3:
  - TDR-012 (critical!)
  - TDR-009
  - TDR-010

Week 4:
  - TDR-006, TDR-007
  - TDR-008
  - TDR-011
  - TDR-013
```

### **Template B: Backend Developer**
```
Sprint 1 (Services):
  - Domain rules for assigned tables (TDR-00X)
  - TDR-009 (service patterns)
  - TDR-011 (selectors)

Sprint 2 (API):
  - TDR-010 (your endpoints)
  - TDR-008 (auth)

Sprint 3 (Integration):
  - TDR-012 (sync implications)
  - TDR-013 (scenarios for your services)
```

### **Template C: Mobile Developer**
```
Critical Path:
  - TDR-012 (ENTIRE DOCUMENT — most important)
  - TDR-010 (sync API, idempotency)
  - TDR-009 (operations to support)
  - TDR-013 (Scenario 7: daily teacher workflow)
```

---

## Search Tips

**Find by keyword** (use grep or IDE search):
- "enrollment" → TDR-003, TDR-009
- "grade" → TDR-004, TDR-005, TDR-009
- "sync" → TDR-012 (entire doc)
- "conflict" → TDR-012 (Section 6), TDR-005 (triggers)
- "offline" → TDR-012 (entire doc)
- "partition" → TDR-002, TDR-005, TDR-007, TDR-009 (partitioning sections)
- "API" → TDR-010
- "RBAC" → TDR-008, TDR-011
- "materialized view" → TDR-005, TDR-007, TDR-011
- "scenario" → TDR-013

---

**Last Updated**: 2025-03-29
**Repository Version**: 1.0
**Total Documents**: 14
