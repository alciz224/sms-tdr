# School Management System — Architecture Overview

## Executive Summary

The **School Management System (SMS)** is an offline-first, 10+ year lifecycle platform designed to manage all aspects of school operations — from student enrollment and academic grading to timetabling and parent communication.

**Key Attributes**:
- ✅ **Offline-First**: Teachers and students can work without internet connectivity; changes sync when reconnected
- ✅ **10-Year Design**: Built to last through multiple school generations with partitioning, archival, and schema evolution
- ✅ **Audit-Complete**: Every change tracked for compliance and legal requirements
- ✅ **Conflict-Resilient**: Deterministic resolution when multiple devices modify the same data
- ✅ **Scalable**: Single database supports up to 100+ schools; partitioning handles 10+ years of data

---

## System at a Glance

```
┌─────────────────────────────────────────────────────────────────────┐
│                         USERS & DEVICES                              │
├─────────────┬─────────────┬──────────────┬──────────────┬──────────┤
│  Teacher    │   Student   │    Parent    │   Registrar  │ Principal│
│  (Tablet)   │  (Mobile)   │  (Phone)     │  (Desktop)   │ (Admin)  │
└─────────────┴─────────────┴──────────────┴──────────────┴──────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    API GATEWAY & AUTHENTICATION                      │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐       │
│  │   REST API │ │  GraphQL   │ │  WebSocket │ │   Sync     │       │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘       │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        APPLICATION LAYER                             │
├─────────────────────────────────────────────────────────────────────┤
│  EnrollmentService │ GradeService │ TimetableService │ SyncService   │
│  PromotionService  │ ReportingSvc │ NotificationSvc  │ AuditService  │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        SELECTOR / REPOSITORY LAYER                   │
│  Queries optimized for performance + Materialized Views for reports │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      POSTGRESQL DATABASE                             │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  14 Core Tables + Partitioning (by school_year) + Caching   │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   OFFLINE DEVICE LOCAL DATABASE                     │
│              (SQLite / Realm on Mobile & Desktop)                   │
│  • Offline Queue  • Cached Reference Data  • Local Operations      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Core Capabilities

### 1. Student Lifecycle Management
- **Enrollment**: Application → Pending → Active → Withdrawn/Completed
- **Academic Progression**: Level/Track assignment, promotion decisions, graduation
- **Profile**: Demographics, guardians, health records, media consent

### 2. Academic Operations
- **Evaluations**: Create, publish, submit, finalize assessments
- **Grades**: Entry, justification, amendment (via appeals), cryptographic sealing
- **Report Cards**: Auto-generated when term finalized
- **Transcripts**: Aggregated academic history with official signatures

### 3. Scheduling
- **Timetables**: Weekly class schedules with conflict detection
- **Classroom-Subject Assignments**: Teacher + room + time slot
- **Exceptions**: Date-specific changes (substitutions, cancellations)
- **Schedule Views**: Student daily schedule, teacher weekly schedule

### 4. Offline Operations
- **Queue-Based**: All writes queued locally when offline
- **Sync Protocol**: Incremental changes with conflict resolution
- **Idempotency**: Safe retry of failed operations
- **Cache**: Reference data + active schedules cached locally

---

## Document Map (How to Navigate This TDR)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        TDR DOCUMENT STRUCTURE                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  📘 START HERE: INDEX.md (navigation guide)                        │
│                                                                     │
│  ┌─ Domain Layer (Business Rules) ──────────────────────────────┐  │
│  │  TDR-001: Master Referentials    (Region, Category, Subject)  │  │
│  │  TDR-003: School Structure       (Year, Classroom, Enrollment)│  │
│  │  TDR-004: Evaluations & Grades   (Grade immutability, appeals)│  │
│  │  TDR-006: Timetables             (Scheduling, conflicts)       │  │
│  │  TDR-008: User & Roles           (Authentication, RBAC)        │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌─ Data Layer (Physical Design) ─────────────────────────────────┐  │
│  │  TDR-002: Master Tables DDL       (Partitioning, indexes)       │  │
│  │  TDR-005: Evaluations/Grades DDL  (Grade security)             │  │
│  │  TDR-007: Timetables DDL          (Conflict-prevention indexes)│  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌─ Service & API Layer ──────────────────────────────────────────┐  │
│  │  TDR-009: Service Layer          (Enrollment, Grade services)  │  │
│  │  TDR-010: API Contracts          (REST endpoints, GraphQL)     │  │
│  │  TDR-011: Selectors & Queries    (Repository patterns, cache)  │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌─ Offline & Operations ─────────────────────────────────────────┐  │
│  │  TDR-012: Offline Sync Strategy (Queue, conflicts, recovery)  │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌─ Validation & Scenarios ───────────────────────────────────────┐  │
│  │  TDR-013: Service Scenarios      (10 real-world narratives)    │  │
│  │  TDR-014: Architecture Decisions(12 ADRs explaining "why")    │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Key Architectural Decisions (at a glance)

| Decision | What | Why |
|----------|------|-----|
| **Offline-First** | Devices work without internet; queue changes for sync | Schools have unreliable connectivity |
| **Eventual Consistency** | Changes propagate asynchronously | Enables offline operations |
| **PostgreSQL** | Relational database with partitioning | Strong integrity + long-term scale |
| **SchoolYear Anchor** | All data scoped to academic year | Natural temporal boundary, easy archival |
| **Grade Appending** | Corrections create new records, don't modify | Audit trail, legal compliance |
| **Timetable Versioning** | Each change creates new version | Historical queries preserved |
| **Tenant Scoping** | All tables have tenant_id | Multi-school isolation in shared DB |
| **Materialized Views** | Pre-computed aggregates for reports | Fast dashboards (50ms vs 1000ms) |
| **REST Primary API** | Standard endpoints for mobile SDK | Simplicity, caching, sync alignment |

**Full details**: See `TDR-014-architecture-decision-records.md`

---

## Data Model Summary

### 14 Core Tables (organized by step)

| Step | Tables | Purpose |
|------|--------|---------|
| **1 — Referentials** | Region, Prefecture, SousPrefecture, Quartier, AcademicYearType, TermType, Term, Category, Level, Track, Subject, SubjectChoice | Permanent structure, geographic & academic |
| **2 — Users** | CustomUser, TeacherProfile, StudentProfile, ParentProfile, AdminProfile | Identity & roles |
| **3 — School Structure** | SchoolYear, SchoolYearCategory, SchoolYearCategoryTerm, Classroom, Enrollment, ClassroomSubject | Year organization & student placement |
| **4 — Evaluations** | EvaluationType, EvaluationCategory, Evaluation, Grade, GradeType, GradeAppeal | Assessments & scoring |
| **5 — Scheduling** | TimeSlot, Timetable, TimetableEntry, TimetableException | Class schedules |
| **6 — Cross-Cutting** | AuditModel (base), StudentIdentifier, ClassroomSubjectCoefficient | Supporting concerns |

**Entity Relationship**: See `table.md` for complete entity diagram

---

## Non-Functional Requirements

### 10+ Year Lifespan
- **Data retention**: All academic records kept for 10+ years
- **Archival strategy**: Partition by school_year; move old years to cold storage
- **Schema evolution**: Backwards-compatible changes, versioned API
- **Deprecation policy**: 2-year deprecation period for removed features

### Offline-First
- **Local database**: SQLite/Realm on device
- **Queue operations**: All writes queued when offline
- **Sync protocol**: Incremental changes with conflict resolution
- **Capability matrix**: Read-only offline, writes queued for certain operations

### Security & Compliance
- **Authentication**: JWT + MFA (TOTP/SMS/Email)
- **Authorization**: RBAC with scoped permissions
- **Encryption**: TLS in transit, field-level encryption for PII at rest
- **Audit**: Immutable audit log for all sensitive operations
- **GDPR**: Right to erasure, data export, consent tracking

### Performance Targets

| Operation | Target Latency (p95) |
|-----------|---------------------|
| User login | < 500ms |
| Enrollment activation | < 2s |
| Grade entry (online) | < 150ms |
| Report card generation (cached) | < 50ms |
| Teacher schedule load | < 150ms |
| Student schedule load | < 100ms |
| Incremental sync | < 500ms |

---

## Integration Points

### External Systems

| System | Integration | Purpose |
|--------|-------------|---------|
| Payment Gateway | REST API + Webhooks | Tuition collection, fee tracking |
| Government SIS | Data import/export | Student records, official transcripts |
| Email/SMS | SMTP + Twilio/MessageBird | Parent notifications, alerts |
| Document Storage | S3-compatible | PDFs, uploads, report cards |
| SSO Provider | SAML/OAuth | Single sign-on for district systems |

---

## Deployment Topology

```
┌─────────────────────────────────────────────────────────────┐
│                    LOAD BALANCER (HTTPS)                    │
│              (TLS termination, rate limiting)               │
└─────────────────────────────┬───────────────────────────────┘
                              │
                 ┌────────────┴────────────┐
                 ▼                         ▼
         ┌──────────────┐        ┌──────────────┐
         │  API Server  │        │  API Server  │
         │   (Pod 1)    │        │   (Pod 2)    │
         └──────────────┘        └──────────────┘
                 │                         │
                 └────────────┬────────────┘
                              │
                 ┌────────────▼────────────┐
                 │   PostgreSQL Cluster    │
                 │  Primary + 2 Replicas   │
                 │  + Partitioned Tables   │
                 └─────────────────────────┘
                              │
                 ┌────────────┴────────────┐
                 ▼                         ▼
         ┌──────────────┐        ┌──────────────┐
         │   Backup     │        │  Object Store│
         │  (PITR)      │        │  (Documents) │
         └──────────────┘        └──────────────┘
```

**Infrastructure as Code**: The above topology should be defined in Terraform/CloudFormation (not in this TDR but in deployment docs).

---

## Getting Started as a Developer

### Quick Orientation Path (30 minutes)

1. **Conceptual overview**: Read this document (5 min)
2. **Big decisions**: Skim `TDR-014-architecture-decision-records.md` (5 min)
3. **Your domain**: Find your area in `INDEX.md` and read the corresponding TDR
4. **See it work**: Read relevant scenarios in `TDR-013-service-scenarios.md` (10 min)
5. **Understand data**: Look up tables in `table.md` and corresponding DDL

### Deep Dive Path (1 week per domain)

```
Week 1: Master referentials + school structure (TDR-001, TDR-002, TDR-003)
Week 2: Evaluations & grades (TDR-004, TDR-005)
Week 3: Timetables (TDR-006, TDR-007)
Week 4: Users & roles (TDR-008)
Week 5: Service layer patterns (TDR-009)
Week 6: API design (TDR-010)
Week 7: Offline sync (TDR-012) ← CRITICAL
Week 8: Putting it together (TDR-011, TDR-013)
```

---

## Quality Attributes

### What This System Does Well

| Attribute | Strength | Evidence |
|-----------|----------|----------|
| **Offline operation** | Full daily work without internet | Queue-based writes, cached reads |
| **Data integrity** | Grades never lost or corrupted | Grade immutability, audit trail |
| **Historical accuracy** | What happened can be reconstructed | Append-only, versioned timetables |
| **Conflict handling** | Concurrent edits resolved predictably | Resolution policies by table |
| **Scale management** | 10+ years manageable | Partitioning by school_year |

### Known Trade-offs

| Trade-off | Chosen Path | Alternative Considered |
|-----------|-------------|-----------------------|
| Offline vs immediate consistency | Eventual consistency (sync later) | Pessimistic locking (requires online) |
| Performance vs freshness | Materialized views (stale ~15min) | Live queries (slower) |
| Single DB vs multi-tenant DB | Shared DB with tenant_id | Separate DB per tenant |
| Rich domain vs anemic | Service layer (orchestration) | Entities with behavior |
| REST vs GraphQL only | REST primary, GraphQL optional | GraphQL only |

---

## Document Status

| Document | Status | Last Updated |
|----------|--------|--------------|
| `prompt.md` | Stable | 2025-03-29 |
| `table.md` | Stable | 2025-03-29 |
| `INDEX.md` | Stable | 2025-03-29 |
| TDR-001 through TDR-014 | Complete | 2025-03-29 |
| This document (`ARCHITECTURE-OVERVIEW.md`) | Draft | 2025-03-29 |

**Total Documentation**: ~54,000 words across 15 documents

---

## Next Steps for Implementation

### Phase 1: Foundation (Months 1-3)
- Set up PostgreSQL with partitioning infrastructure
- Implement CustomUser + authentication
- Build master referentials APIs
- Create device sync skeleton

### Phase 2: Core Operations (Months 4-6)
- Enrollment flow (online + offline queue)
- Grade entry and evaluation workflows
- Timetable generation basics
- Report card generation

### Phase 3: Polish & Scale (Months 7-9)
- Conflict resolution refinement
- Performance optimization
- Materialized view tuning
- Integration adapters (payment, SIS)

### Phase 4: Hardening (Months 10-12)
- Security audit
- Load testing
- Disaster recovery procedures
- Documentation completion

*(This roadmap to be detailed in separate `IMPLEMENTATION-ROADMAP.md` document)*

---

## Questions?

Refer to `INDEX.md` for topic-based navigation, or dive into the specific TDR documents for detailed specifications.

**For architects**: Start with `TDR-014` (ADRs)
**For developers**: Start with `INDEX.md` → your relevant domain
**For product**: Read `TDR-013` (Scenarios) to understand user workflows

---

*This overview reflects the architecture as of March 29, 2025. All 14 TDR documents are complete and form the authoritative specification for implementation.*
