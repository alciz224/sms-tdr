# TDR-014 — Architecture Decision Records (ADRs)

## 1. Purpose and Scope

This document catalogs the **Architectural Decision Records (ADRs)** for the School Management System. Each ADR captures a significant architectural choice, its context, the decision made, and its consequences.

ADRs are maintained in this central location following the [lightweight ADR](https://adr.github.io/) format.

---

## 2. ADR Index

| ID | Title | Status | Date |
|-----|-------|--------|------|
| [ADR-001](#adr-001) | Offline-First Architecture | Accepted | 2025-03-29 |
| [ADR-002](#adr-002) | PostgreSQL as Primary Database | Accepted | 2025-03-29 |
| [ADR-003](#adr-003) | Eventual Consistency Model | Accepted | 2025-03-29 |
| [ADR-004](#adr-004) | Multi-Tenancy via Tenant Scoping | Accepted | 2025-03-29 |
| [ADR-005](#adr-005) | Service Layer over Rich Domain Model | Accepted | 2025-03-29 |
| [ADR-006](#adr-006) | SchoolYear as Temporal Anchor | Accepted | 2025-03-29 |
| [ADR-007](#adr-007) | Grade Immutability via Appending | Accepted | 2025-03-29 |
| [ADR-008](#adr-008) | Timetable Versioning Strategy | Accepted | 2025-03-29 |
| [ADR-009](#adr-009) | Partitioning by School Year | Accepted | 2025-03-29 |
| [ADR-010](#adr-010) | Materialized Views for Reports | Accepted | 2025-03-29 |
| [ADR-011](#adr-011) | RBAC with Scoped Permissions | Accepted | 2025-03-29 |
| [ADR-012](#adr-012) | REST API over GraphQL Primary | Accepted | 2025-03-29 |

---

## ADR-001: Offline-First Architecture

### Status
Accepted

### Date
2025-03-29

### Context

School environments in many regions have unreliable internet connectivity:
- Rural schools: no reliable broadband
- Urban schools: intermittent outages during storms
- Mobile teachers: teach at multiple locations

The system **must** support full daily operations without connectivity for periods up to days or weeks.

### Decision

Adopt an **offline-first architecture** where:

1. All devices maintain a local database (SQLite/Realm)
2. All write operations succeed locally first
3. Changes are queued for asynchronous synchronization
4. Read operations prefer local data, fall back to server
5. Conflict resolution is deterministic and user-visible

### Consequences

**Positive**:
- Teachers can enter grades offline, students can view schedules without internet
- Poor connectivity does not disrupt school operations
- Better UX: no "saving..." spinners, instant feedback

**Negative**:
- Added complexity: sync orchestration, conflict resolution
- Larger mobile app size (embedded database)
- More testing scenarios (online, offline, sync states)

**Mitigations**:
- Clear sync status indicator in UI
- Conflict resolution UI for manual cases
- Comprehensive offline testing automation

---

## ADR-002: PostgreSQL as Primary Database

### Status
Accepted

### Date
2025-03-29

### Context

The system requires:
- Strong transactional guarantees (grade integrity)
- Complex query capabilities (reporting, aggregates)
- 10+ year data retention with partitioning
- Referential integrity across 30+ tables
- JSON capabilities for flexible fields

### Decision

Use **PostgreSQL 15+** as the primary database:

- ACID compliance for critical operations
- Native partitioning for long-term retention
- GIN indexes for full-text search
- JSONB for flexible profile fields
- Row-level security for multi-tenancy
- Extensive tooling and operational maturity

### Consequences

**Positive**:
- Single relational database, no polyglot persistence complexity
- Strong data integrity guarantees
- Rich query capabilities
- Good performance with proper indexing
- Open source, no licensing costs

**Negative**:
- Scale limited to single primary (read replicas possible)
- Vertical scaling ceiling (but 10-year horizon is manageable)
- Mobile sync requires different storage on devices

**Alternatives Considered**:
- **MySQL/MariaDB**: Similar but less advanced partitioning, JSON support
- **MongoDB**: Document model tempting, but lacks transactional joins needed for grades
- **CockroachDB**: Distributed SQL promising but immature, operational complexity

---

## ADR-003: Eventual Consistency Model

### Status
Accepted

### Date
2025-03-29

### Context

Offline-first means devices can be disconnected. The system cannot require all devices to be online for operations (that would violate offline-first).

However, some data (grades, enrollments) must be consistent across devices for integrity.

### Decision

Adopt **eventual consistency** with:

1. Local operations succeed immediately (optimistic)
2. Changes propagate asynchronously via sync
3. Conflicts resolved using deterministic rules
4. Short sync intervals (5 min foreground, 24 hr background)
5. Critical operations (grade finalization) have server validation

Consistency window: typically < 5 minutes, maximum 24 hours.

### Consequences

**Positive**:
- Offline devices work independently
- No blocking network calls
- Better user experience

**Negative**:
- Temporary inconsistencies possible (e.g., two devices modify same grade)
- Conflict resolution needed
- Cannot rely on immediate server state for validation

**Mitigations**:
- Clear conflict resolution policies (server authoritative for grades)
- Pessimistic locking for extremely sensitive operations
- User notifications when conflicts occur

---

## ADR-004: Multi-Tenancy via Tenant Scoping

### Status
Accepted

### Date
2025-03-29

### Context

The system will serve multiple schools (clients). Each school's data must be isolated.

Considerations:
- Some users (system admins) need cross-school visibility
- Some teachers teach at multiple schools
- Reporting may need district-level aggregation

### Decision

Implement **tenant scoping** at the database level:

1. Every table has `tenant_id` (except truly global referentials)
2. All queries automatically filter by `tenant_id` from user's current context
3. User-tenant associations stored in `user_tenant` junction
4. One database, tenant_id as partition key
5. Row-level security policies (optional additional guard)

### Consequences

**Positive**:
- Single database cluster for all tenants (operationally simpler)
- Tenant isolation enforced at query level
- Multi-school users natural (row belongs to tenant A, user has access to both A and B)

**Negative**:
- Must remember tenant_id in every query
- Accidental cross-tenant query could leak data (mitigate with RLS)
- Scaling: all tenants share resources (but typical deployment: < 100 tenants)

**Alternatives Considered**:
- **Separate databases per tenant**: Better isolation, but operational overhead
- **Separate schemas per tenant**: Similar isolation, easier cross-tenant queries
- **Chosen approach**: Single DB, tenant_id column (simplest for < 100 tenants)

---

## ADR-005: Service Layer over Rich Domain Model

### Status
Accepted

### Date
2025-03-29

### Context

The application has complex business logic spanning multiple entities:
- Enrollment involves students, classrooms, fees, documents
- Grade finalization triggers report cards, affects averages
- Promotion requires calculating across terms

Where should this logic live?

### Decision

Implement a **Service Layer** that orchestrates domain entities:

1. Entities are relatively simple (data + basic validation)
2. Services handle multi-entity transactions and workflows
3. Services emit domain events for cross-cutting concerns
4. API layer calls services, never repositories directly

Structure:
```
API → Service → Repository → Database
        ↓
   Domain Events → Event Handlers
```

### Consequences

**Positive**:
- Clear separation: API (protocol), Service (business), Repository (persistence)
- Services easily unit tested (mock repositories)
- Domain events decouple concerns (notifications, audit)
- Orchestration explicit, not hidden in controllers

**Negative**:
- More layers, more classes
- Services can become "god objects" if not disciplined

**Mitigations**:
- One service per aggregate (EnrollmentService, GradeService)
- Keep services focused (Single Responsibility)
- Domain services for complex domain logic that doesn't fit entities

**Alternatives Considered**:
- **Domain-Driven Design with rich entities**: Entities contain behavior, services thin
  - Rejected: Business logic would be scattered across many entity methods
- **CQRS (Command Query Responsibility Segregation)**: Separate models for read/write
  - Too complex for current needs; may adopt later if read scaling becomes issue

---

## ADR-006: SchoolYear as Temporal Anchor

### Status
Accepted

### Date
2025-03-29

### Context

School data is inherently temporal:
- Students belong to a specific SchoolYear
- Classrooms exist per year
- Grades are per term per year
- Historical reports need year context

How to structure this temporal dimension?

### Decision

Use **SchoolYear as the primary temporal anchor**:

```
SchoolYear (2024-2025)
├── SchoolYearCategory (Primaire, Collège, Lycée)
│   ├── SchoolYearCategoryTerm (T1, T2, T3)
│   ├── Classroom (5ème S A)
│   └── Enrollment (student-year contract)
└── Term (T1, T2, T3)
    └── Evaluation (test within term)
        └── Grade (student score)
```

All key operations scoped to a SchoolYearCategory:
- Enrollment: one per student per SchoolYearCategory
- Classroom: per SchoolYearCategory
- Grades: tied back to SchoolYearCategory through Evaluation

### Consequences

**Positive**:
- Natural temporal boundary (easy to archive old years)
- Year-over-year comparisons straightforward
- Clear data lifecycle: DRAFT → ACTIVE → CLOSED → ARCHIVED
- Partitioning by school_year_code efficient

**Negative**:
- Cannot easily have student in two grades in same year (edge case)
- Some queries need joins across year layers
- Schema has temporal complexity

**Alternatives Considered**:
- **Flat model without explicit year**: harder to archive, harder to partition
- **Semester-based only**: too rigid, some schools use trimesters

---

## ADR-007: Grade Immutability via Appending

### Status
Accepted

### Date
2025-03-29

### Context

Grades are legally significant. Once a grade is finalized and a report card issued, that grade should not change.

However, errors happen. Corrections must be:
- Auditable (original preserved)
- Traceable (why changed, who approved)
- Transparent (report cards show amendments)

### Decision

Implement **grade immutability via appending**:

1. `Grade` table has `is_amendment` flag and `amends_grade_id` self-reference
2. FINALIZED grades cannot be UPDATED (database trigger prevents it)
3. To correct: create NEW grade with `is_amendment = true`
4. Calculations use amendment grade if exists, otherwise original
5. GradeAppeal table tracks the request and approval workflow

```
Original Grade: value=12.5, state=FINALIZED, is_amendment=false
Amendment Grade: value=14.0, state=FINALIZED, is_amendment=true, amends_grade_id=original
Result: Student's official grade is 14.0, but audit shows 12.5 → 14.0 correction
```

### Consequences

**Positive**:
- Full audit trail (original preserved)
- Compliance with education record retention laws
- Transparent corrections on transcripts
- Easy to revert if amendment itself wrong

**Negative**:
- More complex queries (need to check for amendments)
- Larger grade table over 10+ years (amendment duplicates)
- Training: users must understand amendment pattern

**Mitigations**:
- Database views that automatically resolve to amendment if exists
- UI shows amendment flag prominently
- Limit amendments (max 3 per grade per year)

---

## ADR-008: Timetable Versioning Strategy

### Status
Accepted

### Date
2025-03-29

### Context

Timetables change mid-year (teacher absence, room change, schedule adjustment). History must be preserved:
- What was the schedule on March 15?
- Who taught what when?
- Attendance records reference schedule

But we also need to know the current active schedule.

### Decision

Implement **versioned timetables with single active version**:

1. Timetable has `version` (integer, increments)
2. Timetable has `effective_from_date` and `effective_to_date`
3. Only ONE active timetable per SchoolYearCategory (`is_active = true` unique constraint)
4. TimetableEntry belongs to a specific timetable version
5. Exceptions (date-specific changes) link to timetable entry but stand alone

When modification needed:
1. Create NEW timetable version (increment)
2. Copy entries from old timetable, apply modifications
3. Update `effective_to_date` of old timetable
4. Set new timetable `effective_from_date`, `is_active = true`
5. Invalidates cached schedules

### Consequences

**Positive**:
- Historical schedule queries straightforward (query by effective date)
- Current schedule always available via `is_active = true` index
- Audit trail preserved naturally (different timetable rows)
- No UPDATE of existing entries (appending changes safely)

**Negative**:
- Duplicate entries when copying (storage, ~2x)
- Need cleanup/archival of old versions
- More complex queries to find "the schedule for date X"

**Mitigations**:
- Old timetables archived to cold storage after 2 years
- Materialized view or function to get schedule for date (hides complexity)
- Automated archival cron job

---

## ADR-009: Partitioning by School Year

### Status
Accepted

### Date
2025-03-29

### Context

10+ years of data will accumulate:
- Grade table: potentially 100M+ rows (30 students × 20 subjects × 50 evaluations × 10 years)
- Evaluation table: 10M+ rows
- Timetable, Enrollment, etc.

Performance on recent queries must remain fast. Old data needs archival.

### Decision

Use **time-based declarative partitioning**:

1. Partition key: `school_year_code` (denormalized string like "2024-2025")
2. Create one partition per school year:
   ```
   grade_y2024 PARTITION OF grade FOR VALUES IN ('2024-2025')
   grade_y2025 PARTITION OF grade FOR VALUES IN ('2025-2026')
   ```
3. Partitions on different tablespaces to manage I/O:
   - Recent (0-3 years): SSD, high performance
   - Middle (3-7 years): standard storage
   - Archive (7-10 years): cold storage tablespace
4. Retention policy: drop partitions older than 10 years (after export)

Queries filtered by school_year_code automatically prune partitions.

### Consequences

**Positive**:
- Query performance: partition pruning reduces scan
- Archival simple: DETACH old partition, ATTACH to archive tablespace
- Data management: can vacuum/optimize partitions independently
- Scale: manageable partition count (~12 partitions for 12 years)

**Negative**:
- Must denormalize school_year_code to every table needing partition
- Partition key must be in WHERE clause for pruning
- Management overhead (creating new partitions annually)

**Alternatives Considered**:
- **Hash partitioning**: Better distribution, but no temporal archival
- **No partitioning**: Bad for query performance over time
- **Chosen**: Time-based + denormalization trade-off reasonable

---

## ADR-010: Materialized Views for Reports

### Status
Accepted

### Date
2025-03-29

### Context

Report card generation involves complex queries:
- Student term average across all subjects
- Class rank per subject
- Subject averages across classroom

These queries join 6-8 tables, aggregate, and can be slow (500ms-2s) on large datasets. Teachers expect instant report viewing.

### Decision

Use **materialized views** for pre-computed aggregates:

1. `report_card_cache`: per-student, per-term, per-subject aggregates
2. `class_average_cache`: classroom subject statistics
3. `student_term_average`: quick lookup of term average

Views:
- Refreshed on demand (when evaluation finalized)
- Refreshed in background (nightly batch)
- `REFRESH MATERIALIZED VIEW CONCURRENTLY` for zero downtime

Application queries the cache; falls back to live query if cache stale.

### Consequences

**Positive**:
- Report card queries: 50ms instead of 1000ms+
- Dashboard statistics fast
- Scalable to many concurrent users

**Negative**:
- Stale data window (refresh lag)
- Storage overhead (duplicate aggregates)
- Complexity: cache invalidation logic

**Mitigations**:
- Trigger refresh on evaluation finalization (fresh data for report generation)
- TTL-based staleness check (refresh if > 1 hour old)
- Acceptable staleness: < 15 minutes acceptable for most reports

---

## ADR-011: RBAC with Scoped Permissions

### Status
Accepted

### Date
2025-03-29

### Context

Users have different capabilities:
- Teacher: view own classes, enter grades for own evaluations
- Parent: view own children only
- Registrar: view all students in school, manage enrollments
- Principal: all school operations

Permissions must be:
- Granular (per resource, per action)
- Scoped (school, classroom, self)
- Configurable (admin can adjust)

### Decision

Implement **Role-Based Access Control with scope-based permissions**:

1. **Roles**: TEACHER, PARENT, REGISTRAR, PRINCIPAL, SYSTEM_ADMIN
2. **Permissions**: `resource:action:scope` format
   - `grade:read:classroom` — read grades in assigned classrooms
   - `grade:create:own_evaluations` — create grades only for own evaluations
   - `enrollment:read:school` — read all enrollments in school
   - `student:read:self` — read own profile (for student role)
3. **Scope evaluation**: Check user's role + context at runtime
4. **Row-level filtering**: Enforce scope via query WHERE clauses

Permission computed at login, cached in session (5 min TTL).

### Consequences

**Positive**:
- Fine-grained access control
- Easy to audit (list roles → list permissions)
- Flexible: new roles can be added
- Cached permissions good performance

**Negative**:
- Complex to implement correctly (many permission checks)
- Harder to debug "why user can't access X"
- Migration challenge if permissions change

**Mitigations**:
- Permissions evaluation in central middleware (single point)
- Debug logs for permission denials
- Admin UI to view user's effective permissions

---

## ADR-012: REST API over GraphQL Primary

### Status
Accepted

### Date
2025-03-29

### Context

Clients need data access:
- Mobile apps (teacher, parent, student)
- Web admin portal
- Third-party integrations (payment, SIS)
- Offline sync protocol

What API style? REST is standard, GraphQL allows flexible queries.

### Decision

**REST as primary API**, GraphQL as optional supplement:

1. REST endpoints for predictable, well-defined operations
   - `/v1/enrollments`, `/v1/grades`, etc.
2. GraphQL available for admin dashboards needing flexible queries
3. REST used by mobile SDKs, offline sync
4. Both share same service layer backend

Rationale:
- REST simpler for common CRUD operations
- REST easier to cache (URL-based)
- REST better aligned with sync protocol (predictable endpoints)
- GraphQL complexity not needed for 80% of use cases

### Consequences

**Positive**:
- REST straightforward, well-understood
- Mobile SDK generation easy from OpenAPI spec
- GraphQL available for complex admin queries
- Clear API boundaries

**Negative**:
- Maintaining two API surfaces (some duplication)
- REST versioning overhead
- GraphQL requires separate schema management

**Alternatives Considered**:
- **GraphQL only**: Flexible but adds complexity for simple use cases
- **REST only**: Decided to offer GraphQL as optional for flexibility

---

## 13. Summary of Key Decisions

These ADRs collectively define the architecture's character:

| Aspect | Decision | Rationale |
|--------|----------|-----------|
| Offline capability | Offline-first with sync | Schools have unreliable connectivity |
| Data consistency | Eventual consistency | Trade immediate consistency for offline availability |
| Data store | PostgreSQL + SQLite (mobile) | Strong relational + cache on device |
| Temporal model | SchoolYear anchor | Natural academic year boundary |
| Multi-tenancy | Tenant scoping in shared DB | Operational efficiency for <100 schools |
| Service architecture | Service layer + repositories | Separation of concerns, testable |
| Grade integrity | Immutability via appending | Audit trail, legal compliance |
- | Scheduling | Versioned timetables | Historical queries, audit naturally |
| Scale | Time-based partitioning | Manage 10+ years of data |
- | Reports | Materialized views | Fast aggregates for dashboards |
| Security | RBAC with scopes | Fine-grained access control |
| API | REST primary, GraphQL optional | Balance simplicity and flexibility |

---

**TDR Repository Complete**

All 14 documents have been created:
1. ✅ Domain Rules — Master Referentials
2. ✅ Data Architecture — Master Tables
3. ✅ Domain Rules — School Structure
4. ✅ Domain Rules — Evaluations & Grades
5. ✅ Data Architecture — Evaluations & Grades
6. ✅ Domain Rules — Timetables
7. ✅ Data Architecture — Timetables
8. ✅ Domain Rules — User & Role System
9. ✅ Domain Rules — Service Layer
10. ✅ API Contracts
11. ✅ Selectors & Queries
12. ✅ Offline Sync Strategy
13. ✅ Service Scenarios
14. ✅ Architecture Decision Records
