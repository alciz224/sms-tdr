# TDR-009 — Domain Rules: Service Layer

## 1. Purpose and Scope

This document defines the **Service Layer** architecture (cross-cutting concern). The Service Layer encapsulates complex business processes that span multiple domain entities, implementing:

- **Transaction orchestration** across multiple tables
- **Business rule validation** beyond simple database constraints
- **Event-driven workflows** with state transitions
- **Offline-capable operations** with eventual consistency
- **Idempotency guarantees** for unreliable connectivity

**Key Principle**: Services are stateless (per request) and orchestrate domain entities. They are the primary entry point for API/CLI interactions.

---

## 2. Service Layer Architecture

### 2.1 Service Categories

```
┌─────────────────────────────────────────────────────────────┐
│                    SERVICE LAYER                             │
├─────────────────────────────────────────────────────────────┤
│  Transactional Services   │   Query Services                 │
│  (write operations)       │   (read-only, optimized)         │
├───────────────────────────┼─────────────────────────────────┤
│ • EnrollmentService       │ • ReportingService              │
│ • GradeService            │ • SearchService                 │
│ • TimetableService        │ • DashboardService              │
│ • PromotionService        │ • AnalyticsService              │
├───────────────────────────┼─────────────────────────────────┤
│  Infrastructure Services  │   Sync Services                 │
│  (cross-cutting)          │   (offline-first)               │
├───────────────────────────┼─────────────────────────────────┤
│ • NotificationService     │ • SyncOrchestrator              │
│ • DocumentService         │ • ConflictResolver              │
│ • AuditService            │ • OfflineQueueManager           │
│ • ValidationService       │                                 │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Service Design Principles

1. **Single Responsibility**: Each service handles one business domain
2. **Idempotency**: Operations can be retried safely (critical for offline sync)
3. **Event Emission**: State changes emit domain events for cross-cutting reactions
4. **Compensation**: Failed operations have explicit undo paths
5. **Validation Pipeline**: Pre-checks → Business Rules → Post-checks

---

## 3. EnrollmentService

**Purpose**: Manage the complete student enrollment lifecycle from application to activation.

### 3.1 Core Operations

#### Operation: `createEnrollment(request: CreateEnrollmentRequest): Enrollment`

**Preconditions**:
1. SchoolYearCategory exists and is in `DRAFT` or `ACTIVE` state
2. StudentProfile exists and `is_active = true`
3. Level/Track combination is valid for the Category
4. Student age is within Level.min_age and Level.max_age
5. No existing active enrollment for student in this SchoolYearCategory

**Process**:
```
1. Validate prerequisites (R-ENR-006)
   ↓ [FAIL]
   Raise Enrollment Validation Error with specific failure reason
   ↓ [PASS]
2. Determine eligible classrooms for (Level, Track)
   ↓
3. Find classroom with capacity:
   IF capacity_available:
       assign classroom_id, enrollment.state = PENDING
   ELSE:
       enrollment.state = WAITLISTED, position = waitlist_position
       emit WaitlistAdded event
       notify parent of waitlist status
   ↓
4. Generate enrollment_number (sequential per SchoolYear)
5. Calculate tuition based on:
   - Category fee structure
   - Scholarship percentage (if applicable)
   - Sibling discounts
   - Payment plan selection
6. Create Enrollment record
7. Create FeeInvoice linked to enrollment
8. Emit EnrollmentCreated event
9. Return Enrollment with pre-signed URLs for document upload
```

**Post-conditions**:
- Enrollment record created with `state = PENDING`
- FeeInvoice created
- Audit log entry
- Event published: `Enrollment.created`

**Idempotency Key**: `(student_profile_id, school_year_category_id)` — duplicate requests return same enrollment

---

#### Operation: `activateEnrollment(enrollment_id: UUID, operator: User): Enrollment`

**Preconditions**:
1. Enrollment exists and `state = PENDING`
2. All documents uploaded and validated
3. Fee status = PAID or PAYMENT_PLAN_APPROVED
4. No prerequisite holds (disciplinary, previous year grades)

**Process**:
```
1. Re-validate capacity (last check):
   classroom.current_enrollment_count < classroom.max_capacity
   ↓ [FULL]
   Raised enrollment to WAITLISTED, notify admin
   ↓
2. BEGIN TRANSACTION:
   a. Enrollment.state = ACTIVE
   b. classroom.current_enrollment_count += 1
   c. Create ClassroomSubject assignments (if not exists)
   d. Generate StudentIdentifier (QR code)
   e. Create initial attendance records
   f. Emit StudentActivated event
3. COMMIT
4. Send welcome notification to parent/student
5. Add to class roster for teacher
```

**Post-conditions**:
- Enrollment `state = ACTIVE`
- Classroom enrollment count incremented
- Student appears in attendance app
- ClassroomSubject assignments exist

**Compensation**: If failure after transaction begins, rollback and emit `EnrollmentActivationFailed`

---

#### Operation: `withdrawEnrollment(enrollment_id: UUID, reason: WithdrawalReason, operator: User): Enrollment`

**Preconditions**:
1. Enrollment exists
2. State is ACTIVE or PENDING
3. If ACTIVE: check if any final grades exist (promotion implications)

**Process**:
```
1. Check financial balance:
   IF unpaid_invoices > 0:
       Raise FinancialHold error (must settle first)
   ↓
2. Return or transfer classroom seat:
   classroom.current_enrollment_count -= 1
   notify waitlist #1 student (if exists)
   ↓
3. State transition:
   PENDING → WITHDRAWN (full refund if no invoices posted)
   ACTIVE → WITHDRAWN (pro-rated refund calculation)
   ↓
4. Handle academic implications:
   IF SchoolYearCategory.term.state = OPEN:
       delete pending evaluations
   IF evaluations SUBMITTED or higher:
       preserve grades, mark enrollment WITHDRAWN
       require grade withdrawal justification
   ↓
5. Generate withdrawal certificate (if requested)
6. Archive withdrawal reason
```

**Post-conditions**:
- Enrollment `state = WITHDRAWN`
- Financial settlement initiated
- Classroom seat available
- Event: `Enrollment.withdrawn`

---

### 3.2 EnrollmentService Invariants

**I-ENR-SVC-001 (Capacity Integrity)**:
```
During activation: classroom.current_enrollment_count ≤ max_capacity
```
Enforced via database constraint + optimistic locking check.

**I-ENR-SVC-002 (Duplication Prevention)**:
```
For (student, school_year_category):
    COUNT(active Enrollment) = 1
```
Enforced via unique index + pre-check.

**I-ENR-SVC-003 (Academic Integrity)**:
```
IF enrollment.withdrawn AND evaluations_finalized_exist:
    grades preserved, not deleted
```
Soft-delete pattern for grades.

**I-ENR-SVC-004 (Financial Consistency)**:
```
Enrollment.fee_status matches Invoice.status
```
Eventual consistency check.

---

### 3.3 EnrollmentService Error Handling

| Error Code | Condition | Recovery |
|------------|-----------|----------|
| ENR001 | Student already enrolled | Check existing enrollment, possibly merge |
| ENR002 | Classroom capacity exceeded | Waitlist or request additional section |
| ENR003 | Age ineligible for level | Suggest appropriate level |
| ENR004 | Prerequisite documents missing | Return list of missing items |
| ENR005 | Financial hold | Process payment first |
| ENR006 | Invalid track for level | Suggest valid tracks |

All errors include: `error_code`, `message`, `field` (which input failed), `suggestion`

---

### 3.4 EnrollmentService Events

```json
{
  "event_type": "enrollment.created",
  "timestamp": "2025-03-29T10:30:00Z",
  "payload": {
    "enrollment_id": "...",
    "student_profile_id": "...",
    "school_year_category_id": "...",
    "classroom_id": "...",
    "state": "PENDING",
    "triggered_by": "REGISTRAR"
  }
}

{
  "event_type": "enrollment.activated",
  "payload": {
    "enrollment_id": "...",
    "activation_date": "2025-03-29",
    "assigned_classroom": "...",
    "notifications_sent": ["parent", "teacher"]
  }
}

{
  "event_type": "enrollment.waitlisted",
  "payload": {
    "enrollment_id": "...",
    "waitlist_position": 3,
    "estimated_availability": "2025-04-15"
  }
}

{
  "event_type": "enrollment.withdrawn",
  "payload": {
    "enrollment_id": "...",
    "withdrawal_reason": "FAMILY_MOVED",
    "refund_amount": 1250.00,
    "academic_impact": "GRADES_PRESERVED"
  }
}
```

---

## 4. GradeService

**Purpose**: Manage grade entry, calculation, finalization, and appeals with strict integrity guarantees.

### 4.1 Core Operations

#### Operation: `enterGrade(evaluation_id: UUID, student_id: UUID, value: Decimal, entered_by: User): Grade`

**Preconditions**:
1. Evaluation exists and `state ∈ {PUBLISHED, IN_PROGRESS, SUBMITTED, REVIEW}`
2. Student is enrolled in the evaluation's classroom
3. `evaluation.evaluation_date ≤ TODAY()`
4. `state = FINALIZED` not allowed (use amendment process)

**Process**:
```
1. Fetch evaluation details (with FOR SHARE to lock)
   ↓
2. Check student eligibility:
   SELECT 1 FROM enrollment
   WHERE student_profile_id = student_id
     AND classroom_id = evaluation.classroom_subject.classroom_id
     AND state = 'ACTIVE'
   ↓ [NOT FOUND]
   Raise GradeEntryError("Student not enrolled in this classroom")
   ↓
3. Check for existing grade:
   IF grade exists:
       IF grade.state = FINALIZED:
           Raise AmendmentRequiredError (use appeal process)
       ELSE:
           UPDATE existing grade (idempotent retry)
   ELSE:
       INSERT new Grade(state = ENTERED)
   ↓
4. Validate grade value:
   IF value < 0 OR value > evaluation.max_score:
       Raise GradeValidationError
   IF value < evaluation.passing_score AND
      evaluation.evaluation_type.requires_justification:
       Require justification parameter
   ↓
5. Update grade table
   ↓
6. Update evaluation counters:
       total_students_completed += 1 (if first grade for this student)
   ↓
7. IF evaluation.completion_percentage >= 90% AND all grades entered:
       suggest SUBMIT
   ↓
8. Return Grade
```

**Idempotency**: Same `(evaluation_id, student_id, value)` can be retried safely

---

#### Operation: `submitEvaluation(evaluation_id: UUID, submitted_by: User): Evaluation`

**Preconditions**:
1. Evaluation `state ∈ {PUBLISHED, IN_PROGRESS}`
2. `total_students_completed = total_students_expected` (or allowances documented)
3. Not past `grade_submission_deadline` (unless extension granted)

**Process**:
```
1. Lock evaluation row (SELECT FOR UPDATE)
   ↓
2. Verify all students have grades (or exempt documentation)
   IF missing_grades > 0:
       Raise IncompleteGradesError(list of missing students)
   ↓
3. State transition:
       evaluation.state = SUBMITTED
       evaluation.submitted_at = NOW()
       evaluation.submitted_by = operator.id
   ↓
4. Emit EvaluationSubmitted event
   ↓
5. Notify academic council of submission
   ↓
6. If auto-approval enabled AND deadline passed:
       transition to REVIEW
   Else:
       await manual review
```

**Compensation**: Can return to `IN_PROGRESS` if review finds issues

---

#### Operation: `finalizeEvaluation(evaluation_id: UUID, finalized_by: User): Evaluation`

**Preconditions**:
1. Evaluation `state = REVIEW`
2. Academic inspector approval recorded
3. No pending grade amendments

**Process**:
```
1. Verify all grades are FINALIZED
   ↓ [not all finalized]
   Raise IncompleteFinalizationError
   ↓
2. For each grade in evaluation:
       IF grade.state != FINALIZED:
           grade.state = FINALIZED
           grade.finalized_at = NOW()
           grade.finalized_by = operator.id
           Compute cryptographic seal (optional)
   ↓
3. Lock and update evaluation:
       evaluation.state = FINALIZED
       evaluation.finalized_at = NOW()
       evaluation.finalized_by = operator.id
   ↓
4. Update aggregate caches:
       Update student term averages (async job)
       Update classroom subject statistics
       Update report card materialized view (async)
   ↓
5. IF evaluation.evaluation_category.is_promotion_critical:
       Trigger promotion review for affected students
   ↓
6. Emit EvaluationFinalized event
   ↓
7. Generate report card PDFs (async)
```

---

#### Operation: `createGradeAppeal(grade_id: UUID, reason: Text, requested_by: User): GradeAppeal`

**Preconditions**:
1. Grade `state = FINALIZED`
2. Appeal within 30 days of report card distribution
3. Student has not exceeded appeal limit (max 3 appeals per year)

**Process**:
```
1. Create GradeAppeal record with status = PENDING
   ↓
2. Lock original grade (prevent amendment by others)
   ↓
3. Notify:
   - Academic council
   - Subject teacher
   - Parent (acknowledgement)
   ↓
4. Review process (outside this method):
   - Council reviews within 14 days
   - May request meeting
   - Decision: APPROVE or DENY
   ↓
5. [IF APPROVED]:
   Create amendment grade (value adjusted)
   Link grade_appeal.amendment_grade_id
   Recalculate affected term averages
   Generate amended transcript
   Emit GradeAmended event
   ↓
6. [IF DENIED]:
   Update appeal.status = DENIED
   Notify parent with explanation
   Emit GradeAppealDenied event
```

---

### 4.2 Grade Calculation Service

#### Operation: `calculateStudentAverage(student_id: UUID, syct_id: UUID): StudentAverageResult`

**Formula**:
```
term_average = Σ(Evaluation.grade × Evaluation.coefficient) / Σ(Evaluation.coefficient)
WHERE:
  - Evaluation in this SYCT with FINALIZED grades
  - Grade.is_exempt = FALSE
  - Evaluation.is_average_computed = TRUE
```

**Process**:
```sql
-- Implementation query (simplified)
SELECT
    student_id,
    ROUND(
        SUM(g.value * e.coefficient) /
        NULLIF(SUM(e.coefficient), 0),
        2
    ) as term_average,
    COUNT(*) as grade_count
FROM grade g
JOIN evaluation e ON g.evaluation_id = e.id
JOIN school_year_category_term syct ON e.school_year_category_term_id = syct.id
WHERE g.student_profile_id = student_id
  AND g.state = 'FINALIZED'
  AND g.is_exempt = FALSE
  AND e.is_average_computed = TRUE
  AND syct.id = syct_id
GROUP BY student_id;
```

**Caching**: Results stored in `student_term_average` table, invalidated on grade changes.

---

### 4.3 PromotionService

**Purpose**: Determine student progression at SchoolYearCategory finalization.

#### Operation: `computePromotionDecisions(sycat_id: UUID, computed_by: User): PromotionBatch`

**Process**:
```
FOR EACH enrollment in school_year_category:
    IF enrollment.state != ACTIVE:
        promotion_decision = N/A (inactive)
    ELSE:
        Retrieve student's term_average for each SYCT
        year_average = weighted average of term_averages

        absences = attendance.summary.unexcused_count
        critical_evaluations_missing = check missing promotion-critical grades

        promotion_rules = get_rules_for(enrollment.level, enrollment.track)

        IF year_average >= promotion_rules.pass_threshold AND
           absences <= promotion_rules.max_absences AND
           critical_evaluations_missing = 0:
            IF enrollment.level.has_next_level:
                promotion_decision = PASS
                promotion_to_level = enrollment.level.next_level
            ELSE:
                promotion_decision = GRADUATE
        ELSEIF year_average >= promotion_rules.review_threshold_low:
            promotion_decision = REVIEW (council decides)
        ELSE:
            promotion_decision = RETAIN (repeat level)

        Record decision in enrollment:
            promotion_decision
            promotion_to_level_id
            promotion_to_track_id
            academic_council_notes
            computed_at
            computed_by
    END IF
END FOR

Emit PromotionDecisionsComputed event (with student IDs affected)
Generate promotion reports (async)
```

**Manual Override**: Academic council can override automated decisions via `overridePromotionDecision()`

---

## 5. TimetableService

**Purpose**: Generate, validate, and manage timetable schedules.

### 5.1 Core Operations

#### Operation: `generateTimetable(sycat_id: UUID, options: GenerationOptions): Timetable`

**Input Options**:
```json
{
  "method": "AUTOMATIC",
  "constraints": {
    "respect_teacher_preferences": true,
    "balance_classroom_hours": true,
    "group_core_subjects_morning": false,
    "max_consecutive_periods": 4
  },
  "priority_subjects": ["MATHEMATICS", "LANGUAGE"],
  "excluded_time_slots": ["FRI_7", "FRI_8"]
}
```

**Algorithm (Constraint Programming)**:
```
1. Load problem space:
   - All ClassroomSubjects with required weekly_hours
   - All Teachers with availability constraints
   - All TimeSlots (active, instructional)
   - Room availability if tracked

2. Sort subjects by priority:
   - Practical/lab subjects first (special room requirements)
   - Core subjects with many hours
   - Electives last

3. For each (ClassroomSubject):
   a. Find candidate (time_slot, teacher, room) tuples
      that satisfy:
      - Teacher available at slot (no conflict)
      - Room available (if specified)
      - Time slot not used by this classroom's other subjects
      - Within teacher's max hours when completed
   b. Assign slot using heuristic:
      - Prefer teacher's preferred slots
      - Distribute across week (not all on one day)
      - Morning slots for priority subjects
   c. IF no candidate found:
      Mark as UNSCHEDULED for manual resolution

4. After all assigned:
   Validate total hours per subject match requirement
   Validate no conflicts exist
   Validate teacher hour limits

5. IF validation fails:
   Return unscheduled list + conflict report

6. Create Timetable + TimetableEntries
   Status = DRAFT

7. Return timetable_id and unscheduled items
```

**Result**: Timetable with entries; any unscheduled items require manual resolution.

---

#### Operation: `publishTimetable(timetable_id: UUID, publisher: User): Timetable`

**Preconditions**:
1. Timetable `status = DRAFT`
2. No unscheduled classroom subjects (or manual override flag)
3. No conflicts detected
4. Teacher notification period elapsed (optional)

**Process**:
```
1. Validate timetable completeness
   ↓
2. Set effective dates:
   effective_from_date = SCHOOL_YEAR_START
   effective_to_date = SCHOOL_YEAR_END
   ↓
3. Update status:
       timetable.status = ACTIVE
       timetable.is_active = TRUE
       timetable.approved_by = publisher.id
       timetable.approved_at = NOW()
   ↓
4. Create TimetableExceptions for any known deviations
   ↓
5. Invalidate previous active timetable:
       previous.status = ARCHIVED
       previous.is_active = FALSE
       previous.effective_to_date = effective_from_date - 1 day
   ↓
6. Emit TimetablePublished event
   ↓
7. Trigger notification job:
       - Teachers: "Your schedule is ready"
       - Students/Parents: "Class schedule available"
       - Admin: "Timetable published successfully"
   ↓
8. Invalidate all cached schedules
```

---

#### Operation: `createTimetableException(exception: ExceptionRequest): TimetableException`

**Process**:
```
1. Validate affected entry exists and is ACTIVE
   ↓
2. Check exception date within timetable validity
   ↓
3. Check for cascade conflicts:
   If CANCELLATION: verify no dependent entries
   If SUBSTITUTION: substitute teacher available?
   If RESCHEDULE: target slot available?
   ↓
4. Create exception record
   ↓
5. Update affected TimetableEntry state:
       original.state = CANCELLED (for reschedule)
       new exception-linked entry created if rescheduled
   ↓
6. Notify affected parties
   ↓
7. Invalidate cached schedules for affected classes
```

---

## 6. ReportingService

**Purpose**: Generate academic reports (report cards, transcripts, class lists).

### 6.1 Report Card Generation

#### Operation: `generateReportCard(enrollment_id: UUID, term_code: String): ReportCard`

**Process**:
```
1. Fetch enrollment with relationships:
   - Student profile
   - Classroom
   - SchoolYearCategory
   - Term (SYCT)

2. Validate prerequisites:
   - All evaluations for term are FINALIZED
   - No pending grade amendments
   ↓

3. Gather data:
   a. Subject grades (from grade table, FINALIZED only)
   b. Term average per subject
   c. Class ranking (optional, per school policy)
   d. Teacher comments (from evaluation.justification)
   e. Attendance summary (excused/unexcused)
   f. Conduct/participation (internal evaluations)
   g. Promotion recommendation

4. Apply grading scale:
   Convert numeric to letter grade (if display_format includes LETTER)

5. Format according to template:
   - Template selected by school
   - Include school logo, branding
   - Principal signature line

6. Generate PDF:
   - Use template engine (Jasper, PDFKit)
   - Store in document storage
   - Record ReportCard record with:
       report_card_id
       enrollment_id
       term_code
       pdf_url
       generated_at
       generated_by

7. Update enrollment:
       last_report_card_generated = term_code
       report_card_url = pdf_url

8. Notify parent:
       "Report card for {student} ({term}) is available"
       Include secure download link

9. Emit ReportCardGenerated event
```

**Performance**: Generate asynchronously; notify when ready (may take 5-10 seconds for 100-page batch).

---

### 6.2 Transcript Generation

#### Operation: `generateTranscript(student_id: UUID, options: TranscriptOptions): Transcript`

**Options**:
```json
{
  "school_years": ["2022-2023", "2023-2024", "2024-2025"],
  "include_social_conduct": true,
  "include_standardized_test_scores": true,
  "format": "PDF",
  "seal_and_sign": true
}
```

**Process**:
```
1. Fetch student's complete academic history:
   - All enrollments across years
   - All FINALIZED grades
   - All promotion decisions
   - Diplomas/certificates earned

2. Calculate cumulative statistics:
   - Overall GPA (converted to standard scale)
   - Best year, worst year
   - Athletics eligibility (if tracked)
   - Honors/recognition list

3. Apply transcript template:
   - Official seal if authenticated
   - Principal signature
   - Registrar signature
   - Issue date

4. Storage and distribution:
   - Store in document storage with tamper-evident hash
   - Return to caller with download URL (expiring)
   - Log transcript issue in audit (who requested, when)

5. Emit TranscriptGenerated event
```

---

## 7. SyncService (Offline-First)

**Purpose**: Orchestrate data synchronization between offline devices and central server with conflict resolution.

### 7.1 Sync Orchestration

#### Operation: `syncDevice(device_id: UUID, sync_request: SyncRequest): SyncResponse`

**SyncRequest structure**:
```json
{
  "device_id": "device-uuid",
  "last_sync_version": 12345,
  "pending_changes": [
    {
      "operation": "INSERT",
      "table": "grade",
      "record": { ... },
      "local_id": "temp-uuid"
    }
  ],
  "capabilities": {
    "storage_available_mb": 500,
    "last_online": "2025-03-28T14:30:00Z"
  }
}
```

**Process**:
```
1. Validate device registration and sync token
   ↓

2. Server version check:
   current_version = get_current_sync_version()
   IF device.last_sync_version == current_version:
       No server changes; proceed to client upload
   ELSE:
       Fetch all changes since last_sync_version:
           SELECT * FROM all_tables
           WHERE updated_at > device_last_sync
   ↓

3. Process client pending changes (in order by timestamp):
   FOR EACH change in pending_changes:
       a. Validate operation type and table
       b. Check user permissions for operation
       c. Run business rule validation (same as online)
       d. Conflict detection:
           - If record modified on server since last_sync:
               Conflict detected
               → Resolve based on conflict policy
               OR → queue for manual resolution
       e. Apply change:
           - INSERT → new record
           - UPDATE → apply if version matches
           - DELETE → soft delete if allowed
       f. Record sync log entry
   ↓

4. Generate SyncResponse:
   {
     "new_sync_version": current_version,
     "server_changes": [...],
     "conflicts": [...],
     "rejected_changes": [...],
     "next_sync_recommended": "2025-03-29T02:00:00Z"
   }

5. Emit SyncCompleted event
```

---

### 7.2 Conflict Resolution Policies

| Conflict Type | Resolution Strategy |
|---------------|---------------------|
| Grade value changed on server and client | Server wins if FINALIZED; client wins if ENTERED (but log conflict) |
| Enrollment created offline but classroom full on server | Client stays PENDING, notify admin |
| Student record modified offline and online | Last-write-wins, merge contact fields, keep server security fields |
| Evaluation modified/retracted | Client changes rejected, device must re-download |

---

## 8. NotificationService

**Purpose**: Send communications via multiple channels (email, SMS, push, in-app).

### 8.1 Notification Types

| Event | Channels | Template |
|-------|----------|----------|
| Enrollment.created | Email, SMS, Push | enrollment_invite |
| Grade.submitted | Email, Push, In-app | grade_available |
| ReportCard.generated | Email, Push | report_card_ready |
| Timetable.published | Email, Push, In-app | timetable_ready |
| Attendance.absent | SMS, Email | absence_alert |
| Fee.payment_due | Email, SMS | payment_reminder |

### 8.2 Delivery Guarantees

- **Critical notifications** (grade finalization, attendance): At least once delivery, retry up to 3 times
- **Informational** (timetable change): At most once (duplicates acceptable)
- **Bulk notifications** (report card batch): Async processing with batch tracking

---

## 9. AuditService

**Purpose**: Immutable audit trail for all sensitive operations.

### 9.1 Audit Events

All operations logged with:
```json
{
  "event_id": "uuid",
  "timestamp": "2025-03-29T10:30:00Z",
  "user_id": "...",
  "action": "GRADE_UPDATE",
  "resource_type": "grade",
  "resource_id": "...",
  "old_values": {"value": 12.5},
  "new_values": {"value": 14.0, "justification": "recalculation error"},
  "ip_address": "192.168.1.1",
  "user_agent": "Claude Code CLI/1.0",
  "tenant_id": "...",
  "reason": "Teacher correction after review"
}
```

### 9.2 Retention Policy

- Grade changes: 10 years (legal requirement)
- Enrollment changes: 10 years after student graduates
- General audit: 7 years
- Security events (login failures): 3 years

---

## 10. ValidationService

**Purpose**: Centralize all business rule validation.

### 10.1 Validation Rules Catalog

```python
# Conceptual rule definitions (pseudocode)
rules = {
    "enrollment": [
        ValidateAgeEligibility(level, student_dob, school_year_start),
        ValidateNoDuplicateEnrollment(student, school_year_category),
        ValidateClassroomCapacity(classroom),
        ValidatePrerequisiteDocuments(required_docs, uploaded_docs),
        ValidateFinancialClearance(student),
    ],
    "grade": [
        ValidateGradeRange(grade_value, max_score),
        ValidateJustificationRequired(grade, evaluation_type),
        ValidateEvaluationOpen(evaluation),
        ValidateGradeImmutable(grade),
    ],
    "timetable": [
        ValidateNoTeacherConflict(teacher, time_slot, timetable),
        ValidateSubjectHours(classroom_subject, scheduled_hours),
    ]
}
```

### 10.2 Validation Pipeline

Each service operation:
```
1. Syntactic validation (input types, required fields)
2. Business rule validation (call appropriate validators)
3. Cross-entity consistency checks
4. Permission validation (user can perform this operation)
5. All pass → proceed
   Any fail → raise ValidationError with field-specific messages
```

---

## 11. Error Handling & Resilience

### 11.1 Error Taxonomy

| Category | Retry Strategy | User Impact |
|----------|----------------|-------------|
| ValidationError | No retry | Show error, ask for correction |
| ConcurrencyError | Exponential backoff | Show "try again" message |
| NotFoundError | No retry | Show "not found", verify input |
| PermissionError | No retry | Show "access denied" |
| ServiceUnavailable | Circuit breaker, retry | Show "temporarily unavailable" |
| SyncConflict | Manual resolution | Notify admin, queue for review |

### 11.2 Compensation Patterns

For multi-step transactions:
```
Operation:
  Step 1: Create Enrollment
  Step 2: Create Invoice
  Step 3: Send notification

IF failure at Step 3:
  Compensation:
    - Enrollment and Invoice remain (no rollback)
    - Notification can be retried separately
    - Log compensation for audit
```

---

## 12. Service Contracts (Interfaces)

### 12.1 EnrollmentService Interface

```typescript
interface EnrollmentService {
  createEnrollment(request: CreateEnrollmentRequest): Promise<Enrollment>;
  activateEnrollment(id: UUID, operator: UserContext): Promise<Enrollment>;
  withdrawEnrollment(id: UUID, reason: WithdrawalReason, operator: UserContext): Promise<Enrollment>;
  getEnrollment(id: UUID): Promise<Enrollment>;
  listEnrollments(filter: EnrollmentFilter): Promise<Enrollment[]>;
  transferEnrollment(id: UUID, targetClassroomId: UUID, operator: UserContext): Promise<Enrollment>;
  waitlistTo Enrollment(waitlistId: UUID, operator: UserContext): Promise<Enrollment>;
}

interface CreateEnrollmentRequest {
  studentProfileId: UUID;
  schoolYearCategoryId: UUID;
  levelId?: UUID;  // optional, auto-determined if not provided
  trackId?: UUID;
  enrollmentType: 'NEW' | 'TRANSFER' | 'RETURNING';
  preferredClassroomIds?: UUID[];
  scholarshipPercentage?: number;
  paymentPlanId?: UUID;
}
```

---

### 12.2 GradeService Interface

```typescript
interface GradeService {
  enterGrade(evaluationId: UUID, studentId: UUID, value: number, justification?: string, operator: UserContext): Promise<Grade>;
  submitEvaluation(evaluationId: UUID, operator: UserContext): Promise<Evaluation>;
  finalizeEvaluation(evaluationId: UUID, operator: UserContext): Promise<Evaluation>;
  createGradeAppeal(gradeId: UUID, reason: string, requestedBy: UserContext): Promise<GradeAppeal>;
  getStudentTermAverage(studentId: UUID, syctId: UUID): Promise<StudentAverageResult>;
  getClassroomGrades(classroomSubjectId: UUID, syctId: UUID): Promise<ClassroomGradeSheet>;
  amendGrade(gradeId: UUID, newValue: number, reason: string, operator: UserContext): Promise<Grade>;
}
```

---

### 12.3 TimetableService Interface

```typescript
interface TimetableService {
  generateTimetable(sycatId: UUID, options: GenerationOptions): Promise<TimetableGenerationResult>;
  publishTimetable(timetableId: UUID, publisher: UserContext): Promise<Timetable>;
  createTimetableException(timetableId: UUID, exception: ExceptionRequest): Promise<TimetableException>;
  getTeacherSchedule(teacherId: UUID, timetableId: UUID): Promise<Schedule>;
  getClassroomSchedule(classroomId: UUID, timetableId: UUID): Promise<Schedule>;
  getStudentSchedule(studentId: UUID, date: Date): Promise<DailySchedule>;
  checkConflicts(timetableId: UUID): Promise<ConflictReport>;
}
```

---

## 13. Service Layer Event Catalog

### 13.1 Domain Events

| Event | When Published | Subscribers |
|-------|----------------|-------------|
| `enrollment.created` | New enrollment initiated | Notification, Analytics |
| `enrollment.activated` | Student officially enrolled | Attendance, Classroom roster |
| `enrollment.withdrawn` | Student withdraws | Finance (refund), Waitlist |
| `grade.entered` | Teacher entered grade | Dashboard updates |
| `grade.submitted` | Teacher submitted grades for review | Academic inspector |
| `grade.finalized` | Grade locked | Report card generator, Parent notification |
| `grade.appealed` | Appeal filed | Academic council |
| `grade.amended` | Original grade corrected | Transcript updater |
| `evaluation.finalized` | All grades entered and approved | Promotion service |
| `timetable.published` | Schedule released | Notification, Cache invalidation |
| `report_card.generated` | PDF ready | Parent notification |
| `sync.completed` | Device sync finished | Device registry update |

---

## 14. Performance Considerations

**Service Operation SLAs**:
- Enrollment activation: < 2 seconds
- Grade entry: < 500ms
- Timetable generation (30 classrooms): < 60 seconds (async)
- Report card generation (batch 100): < 10 seconds per batch

**Concurrency Controls**:
- Classroom enrollment: Row-level locking or optimistic concurrency on `current_enrollment_count`
- Grade finalization: Advisory locks to prevent concurrent modifications
- Timetable publication: Exclusive lock on SchoolYearCategory

**Caching Strategy**:
- Grade averages: denormalized columns refreshed asynchronously
- Student enrollment info: cached in Redis (1 hour TTL)
- Teacher schedules: cached in memory (invalidate on timetable change)

---

## 15. Testing Strategy for Services

Each service should have:

1. **Unit tests** for each operation (happy path + edge cases)
2. **Integration tests** with database (test constraints, transactions)
3. **Idempotency tests**: retry same request, verify same result
4. **Concurrency tests**: simulate two users operating on same resource
5. **Offline simulation tests**: using offline queue patterns

---

## 16. Cross-Service Coordination

### 16.1 Enrollment → ClassroomSubject

When enrollment activates:
1. EnrollmentService calls `getOrCreateClassroomSubject(classroom, subject)`
2. ClassroomSubject ensures teacher assignment exists
3. GradeService backfills any required baseline entries

### 16.2 Grade Finalization → Report Card

1. GradeService emits `grade.finalized`
2. ReportGenerator subscribes, batches by SYCT
3. When all grades in SYCT finalized, generates report cards
4. Notifies parents via NotificationService

### 16.3 Promotion → Next Year Enrollment

1. PromotionService publishes `promotion.decisions_computed`
2. EnrollmentService auto-creates next year enrollment for passers
3. NotificationService sends re-enrollment instructions

---

**Next Step**: TDR-010 — API Contracts (REST/GraphQL specifications)
