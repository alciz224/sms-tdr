# TDR-003 — Domain Rules: Core School Structure

## 1. Purpose and Scope

This document defines the domain rules governing the core school structure (Étape 3). These tables form the **heart of the School Management System**, implementing:

- **Annual cycle management** via SchoolYear
- **Period/term management** within years via SchoolYearCategoryTerm (the critical concept)
- **Classroom organization** combining year, category, level, track, and term
- **Student enrollment lifecycle** with academic progression

These rules embody the "strict annual logic" and "periods by cycle" principles from the system requirements.

---

## 2. SchoolYear Domain Rules

### 2.1 Definition and Lifecycle

**SchoolYear** represents a complete academic year cycle (e.g., "2024-2025").

**State Machine**:
```
DRAFT → ACTIVE → CLOSED → ARCHIVED
```

Transitions are strictly controlled:

| From | To | Allowed Conditions | Who Can Execute |
|------|-----|---------------------|-----------------|
| DRAFT | ACTIVE | Start date set, all Term dates configured, no enrollments yet | School Admin + District Supervisor |
| ACTIVE | CLOSED | All evaluations submitted, all grades finalized | School Admin + Academic Inspector |
| CLOSED | ARCHIVED | 1 year after CLOSED, zero active enrollments | System Admin only |

**Domain Rules**:

1. **R-SY-001**: SchoolYear code format: `[start_year]-[end_year]` (e.g., "2024-2025")
   - `end_year` MUST equal `start_year + 1`
   - Non-overlapping constraint: adjacent SchoolYears can overlap by at most 1 day for transition

2. **R-SY-002**: SchoolYear dates:
   - `start_date` = first day of instruction
   - `end_date` = last day of instruction
   - Duration must be between 150 and 220 days (configurable per country)

3. **R-SY-003**: AcademicYearType is assigned at SchoolYear creation and **IMMUTABLE** thereafter
   - Reason: changing from TRIMESTER to SEMESTER would require massive data restructuring

4. **R-SY-004**: SchoolYear.is_active flag indicates current operational year
   - At any time, **exactly one** SchoolYear must have `is_active = true` per School (or per system for district-level)
   - Enrollments can only happen in active SchoolYear
   - Evaluation creation restricted to active SchoolYear terms

5. **R-SY-005**: SchoolYear closure requirements (before transition to CLOSED):
   - All Term evaluations have `status = 'FINALIZED'`
   - All Grade records have passed academic council review
   - All student reports generated and distributed
   - No pending grade change requests

6. **R-SY-006**: An ARCHIVED SchoolYear:
   - Cannot be modified in any way
   - All queries against it must be read-only
   - Can be moved to archive partition after 5 years
   - Must retain all data for 10+ years (legal requirement)

**Fields**:
- `id` (UUID)
- `school_id` (UUID, foreign key to School)
- `academic_year_type_id` (UUID, foreign key to AcademicYearType)
- `code` (VARCHAR[20]) — "2024-2025"
- `name` (VARCHAR[100]) — "Année Scolaire 2024-2025"
- `start_date` (DATE)
- `end_date` (DATE)
- `orientation_start_date` (DATE, optional) — before formal instruction
- `exams_end_date` (DATE) — final exam period end
- `state` (VARCHAR[20]) — "DRAFT", "ACTIVE", "CLOSED", "ARCHIVED"
- `is_active` (BOOLEAN) — convenience flag
- `total_school_days` (INTEGER, computed)
- `notes` (TEXT)
- Audit trail fields

---

## 3. SchoolYearCategory Domain Rules

### 3.1 Definition

**SchoolYearCategory** links the global Category (Primaire, Collège, Lycée) to a specific SchoolYear. It represents "the Primaire division for the 2024-2025 school year."

**Critical Insight**: This is NOT just a junction table. It's a **temporal binding** that says "This Primaire cycle operates within this SchoolYear with these specific configurations."

**Domain Rules**:

1. **R-SYC-001**: SchoolYearCategory is created when SchoolYear is activated
   - Automatically generates one SchoolYearCategory for each Category relevant to the School
   - Manual override allowed: may skip certain categories (e.g., school without Lycée)

2. **R-SYC-002**: SchoolYearCategory `code` format: `[school_year_code]-[category_code]`
   - Example: "2024-2025-PRIM", "2024-2025-COLL"
   - Must be unique within the School

3. **R-SYC-003**: SchoolYearCategory configuration:
   - `min_students_per_class` (default varies by Category)
   - `max_students_per_class` (default varies by Category)
   - `requires_uniform` (boolean)
   - `catalog_year` (reference to subject curriculum version)

4. **R-SYC-004**: SchoolYearCategory inherits global Category properties:
   - Grading scale (from Category → Level → subject overrides)
   - Age ranges (from Level definitions)
   - Mandatory subject requirements (from SubjectChoice templates)

5. **R-SYC-005**: SchoolYearCategory determines:
   - Which Levels are active this year (through Level.is_active flag per category)
   - Which Tracks are offered (may vary year to year)
   - Registration deadlines and capacity limits

**Fields**:
- `id` (UUID)
- `school_year_id` (UUID, foreign key to SchoolYear, ON DELETE CASCADE)
- `category_id` (UUID, foreign key to Category)
- `code` (VARCHAR[30], unique)
- `name` (VARCHAR[100]) — "Primaire 2024-2025"
- `min_students_per_class` (INTEGER, default 20)
- `max_students_per_class` (INTEGER, default 30)
- `requires_uniform` (BOOLEAN, default TRUE)
- `catalog_year` (INTEGER) — curriculum year reference
- `is_active` (BOOLEAN)
- Audit trail

---

## 4. SchoolYearCategoryTerm Domain Rules

### 4.1 The Critical Concept

**SchoolYearCategoryTerm (SYCT)** is the **most important table** for implementing "strict annual logic" and "periods by cycle."

**Definition**: SchoolYearCategoryTerm binds together:
- A **SchoolYearCategory** (the category within a specific year)
- A **Term** (the time period)
- The **operational configuration** for that period

It answers: "For the Primaire cycle in 2024-2025, what are the rules for Term 1?"

### 4.2 Domain Rules

1. **R-SYCT-001**: **ONE-TO-ONE Relationship**: Each Term in a SchoolYear has exactly ONE SchoolYearCategoryTerm per active SchoolYearCategory

   - Example: SchoolYear "2024-2025" has 3 Terms (T1, T2, T3)
   - Category "Primaire" creates 3 SYCT records: (Primaire-T1), (Primaire-T2), (Primaire-T3)
   - Category "Collège" also creates 3 SYCT records (Collège-T1), etc.

2. **R-SYCT-002**: SYCT `code` format: `[school_year_category_code]-[term_code]`
   - Example: "2024-2025-PRIM-T1", "2024-2025-COLL-T2"
   - Must be globally unique

3. **R-SYCT-003**: SYCT state follows Term+SchoolYearCategory state combination:
   - Cannot be `OPEN` if Term is outside date range
   - Cannot be `CLOSED` if SchoolYearCategory still accepts enrollments
   - State transitions coordinated with SchoolYear and Term states

4. **R-SYCT-004**: Evaluation windows controlled by SYCT:
   - `evaluation_submission_deadline` = Term.end_date - buffer (typically 3 days)
   - `grade_finalization_deadline` = evaluation_submission_deadline + 7 days
   - `makeup_evaluation_window_start/end` for students with valid excuses

5. **R-SYCT-005**: Promotion/Progression decision point:
   - At SYCT closure, determine which students:
     - Pass to next level within same track
     - Need academic support/repeat
     - Change tracks (with approval)
   - This decision is recorded against SYCT, not individual Term

6. **R-SYCT-006**: SYCT-specific configurations (overrides from SchoolYearCategory):
   - `allow_elective_changes` — can students change electives between terms?
   - `require_teacher_signature_on_reports` — varies by term (report cards vs progress sheets)
   - `final_term` flag — indicates terminal evaluation term (diploma/brevet/baccalaureate)

7. **R-SYCT-007**: Reporting structure:
   - Term 1: Progress report only (no final grades)
   - Term 2+: Full report card
   - Final term (T3 or T4): Includes promotion recommendation

8. **R-SYCT-008**: Date calculations inherit from Term but can have SYCT-specific adjustments:
   - `actual_start_date` = Term.start_date + (SYCT.delayed_start_days or 0)
   - `actual_end_date` = Term.end_date - (SYCT.early_closure_days or 0)
   - Used for schools with delayed openings or early exams

**Fields**:
- `id` (UUID)
- `school_year_category_id` (UUID, foreign key to SchoolYearCategory, ON DELETE CASCADE)
- `term_id` (UUID, foreign key to Term, ON DELETE RESTRICT)
- `code` (VARCHAR[40], unique)
- `name` (VARCHAR[100]) — "Primaire — Trimestre 1"
- `state` (VARCHAR[20]) — "SCHEDULED", "OPEN", "CLOSED", "GRADING", "FINALIZED"

- `is_final_term` (BOOLEAN) — this term determines promotion
- `is_reporting_term` (BOOLEAN) — generates report cards

- `evaluation_submission_deadline` (DATE)
- `grade_finalization_deadline` (DATE)
- `makeup_window_start` (DATE)
- `makeup_window_end` (DATE)

- `allow_elective_changes` (BOOLEAN, default FALSE)
- `require_teacher_signature` (BOOLEAN, default TRUE)
- `delayed_start_days` (INTEGER, default 0)
- `early_closure_days` (INTEGER, default 0)
- `special_instructions` (TEXT)

- `promotion_pass_threshold` (DECIMAL) — overrides default if set
- `max_absences_allowed` (INTEGER) — term-specific attendance policy

- `is_active` (BOOLEAN)
- Audit trail

**Unique Constraint**:
- `UNIQUE (school_year_category_id, term_id)` — one SYCT per term per category

**Composite Index**:
- `(school_year_category_id, state)` — for finding open SYCTs

---

### 4.3 Invariants

**I-SYCT-001 (Term Consumptions)**:
- Each Term's duration (start_date to end_date) must be fully consumed by exactly one SYCT per active SchoolYearCategory
- No gaps between consecutive SYCT terms within same SchoolYearCategory

**I-SYCT-002 (Final Terminal Term)**:
- Exactly one SYCT per SchoolYearCategory has `is_final_term = true`
- The final term is always the last Term (highest sequence number)

**I-SYCT-003 (Date Bounds)**:
```
SYCT.actual_start_date >= Term.start_date
SYCT.actual_end_date <= Term.end_date
SYCT.evaluation_submission_deadline <= Term.end_date
SYCT.grade_finalization_deadline >= SYCT.evaluation_submission_deadline
```

**I-SYCT-004 (Promotion Uniformity)**:
- All SYCTs within same SchoolYearCategory must share:
  - `promotion_pass_threshold` (if set explicitly)
  - `max_absences_allowed`
- These represent category-wide policies, not per-term variations

**I-SYCT-005 (State Consistency)**:
- SYCT state must be consistent with parent SchoolYearCategory state:
  - If SchoolYearCategory is CLOSED, all SYCTs must be FINALIZED
- SYCT state must be consistent with parent Term:
  - If Term is not within its date range, SYCT cannot be OPEN

---

### 4.4 SYCT State Transition Rules

```
          ┌─────┐
          │DRAFT│ (SchoolYearCategory created, Term not yet configured)
          └─────┘
             │ Term dates set, evaluation windows calculated
             ▼
        ┌────────┐
        │SCHEDULED│ (Ready but not yet open for instruction)
        └────────┘
             │ Term.start_date - delayed_start_days arrives
             ▼
        ┌───────┐
        │  OPEN │ (Evaluation creation, grade entry allowed)
        └───────┘
             │ Term.end_date arrives
             ▼
        ┌────────┐
        │ CLOSED │ (No more evaluations, grading phase)
        └────────┘
             │ grade_finalization_deadline passes + all grades verified
             ▼
     ┌──────────────┐
     │  FINALIZED   │ (Report cards generated, promotion decisions made)
     └──────────────┘
             │
             ▼ (Once all SYCTs in SchoolYearCategory are FINALIZED)
        ┌──────────┐
        │ARCHIVED  │
        └──────────┘
```

**State Transition Guards**:
- DRAFT → SCHEDULED: Only allowed by SchoolAdmin if Term dates are valid
- SCHEDULED → OPEN: Only when date condition met, automatic or manual trigger
- OPEN → CLOSED: Only after evaluation_submission_deadline, can be forced if needed
- CLOSED → FINALIZED: Only after all grades are approved by Academic Council
- Any → ARCHIVED: Only after SchoolYearCategory is CLOSED and 1 year has passed

---

## 5. Classroom Domain Rules

### 5.1 Definition

**Classroom** represents a specific group of students who move together through the curriculum for a SchoolYearCategory.

**Key Principle**: A Student is assigned to exactly ONE Classroom per SchoolYearCategory.

Classroom is the **anchor** for:
- Class roster (Enrollments)
- Classroom-Subject assignments (which teacher teaches Math to Class 5ème A)
- Timetable
- Report card generation

### 5.2 Domain Rules

1. **R-CLASS-001**: Classroom code format: `[school_year_category_code]-[track_code?]-[sequence]`
   - Examples:
     - "2024-2025-PRIM-CP-A" (Primary, CP class, section A)
     - "2024-2025-COLL-5E-S" (College, 5eme, Scientifique track)
     - "2024-2025-LYC-TERM-L" (Lycée, Terminale, Literary track)
   - Must be unique within SchoolYearCategory

2. **R-CLASS-002**: Classroom configuration:
   - `school_year_category_id` (not SchoolYear — important distinction)
   - `track_id` (nullable, NULL indicates mixed-track or undecided)
   - `level_id` (from Category/Level)
   - `max_capacity` (typically 30, configurable)
   - `current_enrollment_count` (computed, denormalized for quick lookup)

3. **R-CLASS-003**: Track consistency:
   - If `track_id IS NOT NULL`, all students in classroom must have that track
   - If `track_id IS NULL`, track must be assigned per-student during enrollment (allow late decision)

4. **R-CLASS-004**: Classroom creation:
   - Auto-generated based on projected enrollment (min/max per class)
   - Manual override allowed by SchoolAdmin
   - Classrooms can be added mid-year (with capacity expansion)
   - Classrooms CANNOT be deleted once enrollments exist (archive only)

5. **R-CLASS-005**: Classroom lifecycle:
   - `PLANNED` — before SchoolYear activation, based on projections
   - `ACTIVE` — during the school year, accepting enrollments
   - `FROZEN` — after add/drop deadline, no enrollment changes
   - `CLOSED` — at SchoolYear end, archived

6. **R-CLASS-006**: Classroom naming conventions:
   - Section letters: A, B, C, ... (for parallel classes at same level)
   - No "I", "O" to avoid confusion (configurable by country)
   - Special designations: "SPE" for special needs, "ALT" for alternative program

**Fields**:
- `id` (UUID)
- `school_year_category_id` (UUID, foreign key to SchoolYearCategory)
- `code` (VARCHAR[50], unique)
- `name` (VARCHAR[100]) — "CP A", "5ème Scientifique A"
- `level_id` (UUID, foreign key to Level)
- `track_id` (UUID, foreign key to Track, nullable)
- `max_capacity` (INTEGER, default 30)
- `current_enrollment_count` (INTEGER, default 0, denormalized)
- `homeroom_teacher_id` (UUID, foreign key to TeacherProfile) — primary advisor
- `classroom_characteristics` (JSONB) — {"needs_accommodation": true, "gifted_program": false}
- `state` (VARCHAR[20]) — "PLANNED", "ACTIVE", "FROZEN", "CLOSED"
- `is_active` (BOOLEAN)
- Audit trail

**Indexes**:
- `(school_year_category_id, code)` unique
- `(level_id, track_id)` for subject assignment queries
- `(homeroom_teacher_id)` for teacher dashboard

---

### 5.7 Invariants

**I-CLASS-001 (Capacity)**:
```
current_enrollment_count <= max_capacity
```
Enforced at enrollment time; if full, student may be placed on waitlist.

**I-CLASS-002 (Track Consistency)**:
- All students enrolled in a classroom with `track_id = X` must have track X in their Enrollment
- If `track_id IS NULL`, classroom allows mixed-track enrollment (special case)

**I-CLASS-003 (Level Match)**:
- Classroom's `level_id` must be a Level belonging to the SchoolYearCategory's Category
- Enrollment student's Academic Level (from previous year's promotion) determines eligible classrooms

---

## 6. Enrollment Domain Rules

### 6.1 Definition

**Enrollment** is the **contractual relationship** binding a Student to a Classroom for a SchoolYearCategory.

This is the most critical transactional table — it's where student academic journey begins each year.

**State Machine**:
```
PENDING → ACTIVE → WITHDRAWN → COMPLETED
```

### 6.2 Enrollment Creation Rules

1. **R-ENR-001**: Enrollment can be created ONLY when:
   - SchoolYearCategory exists and is in DRAFT or ACTIVE state
   - Target Classroom exists and `current_enrollment_count < max_capacity`
   - Student meets prerequisites for the Level (previous year's promotion or entrance exam)
   - All required documents submitted (medical, transcripts, guardian consent)
   - Fees paid or payment plan approved

2. **R-ENR-002**: Enrollment number format: `[school_year_code]-[sequential]`
   - Example: "2024-2025-000123"
   - Must be globally unique
   - Generated sequentially per SchoolYear
   - Never reused even if enrollment is withdrawn

3. **R-ENR-003**: Track determination at enrollment:
   - If student has confirmed track (entrance exam, previous track), use that
   - If undecided (new middle school student), track can be NULL initially
   - Track CAN change via promotion decision or transfer within first 30 days

4. **R-ENR-004**: Level determination at enrollment:
   - Based on student's age and previous academic record
   - Can be overridden by Academic Council for special cases
   - Must satisfy Level.min_age ≤ student.age ≤ Level.max_age at SchoolYear start

5. **R-ENR-005**: Classroom assignment logic:
   ```
   IF track IS NOT NULL:
       Find first available classroom with (level, track)
   ELSE:
       Find classroom with level that accepts mixed-track (track_id IS NULL)
   IF no classroom available:
       WAITLIST position assigned
   ```

6. **R-ENR-006**: Enrollment prerequisites validation (all must pass):
   - Age eligibility for Level
   - No delinquent fees from previous year
   - No disciplinary suspension carryover
   - Medical clearance if required by Category
   - Required uniform purchased (or waiver)

### 6.3 Enrollment State Transitions

| From | To | Trigger | Who Can Execute |
|------|-----|---------|-----------------|
| PENDING | ACTIVE | All prerequisites satisfied, fees cleared | Registrar |
| PENDING | WITHDRAWN | Student/parent withdraws before start date | Parent/Registrar |
| ACTIVE | WITHDRAWN | Student withdraws mid-year | Parent + Principal approval |
| ACTIVE | COMPLETED | SchoolYear CLOSED + all grades submitted | System (automatic) |

### 6.4 Domain Rules for Active Enrollment

1. **R-ENR-007**: Once `state = ACTIVE`:
   - Student appears in classroom roster
   - Student can attend classes
   - Evaluations can be created for the student
   - Grades can be entered
   - Attendance tracking enabled

2. **R-ENR-008**: Enrollment modifications:
   - `track_id` change requires:
     - All ClassroomSubject assignments recalculated
     - May trigger classroom change
     - Must happen within add/drop period (configurable, typically 30 days)
   - `level_id` change:
     - Requires Academic Council approval
     - Rare, for special promotion or retention
     - Triggers new ClassroomSubject setup

3. **R-ENR-009**: Withdrawal implications:
   - Attendance records preserved
   - Evaluations with submitted grades retained
   - Report cards generated for completed terms
   - Re-enrollment flag saved for returning students
   - Financial settlement required

4. **R-ENR-010**: Mid-year transfers:
   - Incoming: Enrollment created with state BLOCKED until records validated
   - Outgoing: Enrollment marked TRANSFERRED, generate transcript
   - Transfer requires both schools' approval (digital exchange)

### 6.5 Enrollment Fields

**Core Identification**:
- `id` (UUID)
- `enrollment_number` (VARCHAR[30], unique)
- `school_year_category_id` (UUID, foreign key, NOT NULL)
- `classroom_id` (UUID, foreign key, nullable until assigned)

**Student & Academic**:
- `student_profile_id` (UUID, foreign key to StudentProfile, NOT NULL)
- `admission_date` (DATE)
- `level_id` (UUID, foreign key to Level) — effective level for this enrollment
- `track_id` (UUID, foreign key to Track, nullable)

**Status**:
- `state` (VARCHAR[20]) — "PENDING", "ACTIVE", "WITHDRAWN", "COMPLETED", "TRANSFERRED"
- `withdrawal_reason` (TEXT)
- `withdrawal_date` (DATE)
- `withdrawal_approved_by` (UUID)

**Academic Tracking**:
- `promotion_decision` (VARCHAR[20]) — "PASS", "RETAIN", "TRANSFER", "GRADUATE"
- `promotion_to_level_id` (UUID, for next year)
- `promotion_to_track_id` (UUID, for next year)
- `academic_council_notes` (TEXT)

**Financial**:
- `tuition_fee` (DECIMAL)
- `fee_status` (VARCHAR[20]) — "UNPAID", "PARTIAL", "PAID", "WAIVED"
- `scholarship_percent` (DECIMAL)

**Control**:
- `is_active` (BOOLEAN)
- `enrollment_source` (VARCHAR[30]) — "NEW", "RETURNING", "TRANSFER_IN"
- Audit trail

---

### 6.6 Critical Invariants

**I-ENR-001 (One Enrollment per Year)**:
```
For a given (student_profile_id, school_year_id):
    COUNT(active Enrollment) ≤ 1
```
A student cannot be enrolled twice in the same SchoolYearCategory.

**I-ENR-002 (Classroom Consistency)**:
```
IF enrollment.classroom_id IS NOT NULL:
    enrollment.level_id = classroom.level_id
    enrollment.track_id = classroom.track_id OR classroom.track_id IS NULL
```

**I-ENR-003 (State Guard)**:
- `state = ACTIVE` requires non-NULL `classroom_id`
- `state = COMPLETED` requires all associated Grades finalized
- `promotion_decision` NULL if `state ≠ COMPLETED`

**I-ENR-004 (Waitlist)**:
- Student on waitlist has `state = PENDING` and `classroom_id IS NULL`
- When classroom space available, system auto-assigns and transitions to ACTIVE

**I-ENR-005 (Age Eligibility)**:
```
student.date_of_birth + Level.min_age years ≤ school_year.start_date
AND
student.date_of_birth + Level.max_age years ≥ school_year.end_date
```

---

### 6.7 Enrollment Business Processes

#### 6.7.1 New Student Registration Flow

```
1. Parent submits application with documents
   ↓
2. Registrar verifies prerequisites (R-ENR-006)
   ↓ [Fail]
   Application rejected, reason logged
   ↓ [Pass]
3. System determines eligible (Level, Track) based on:
   - Age date_of_birth
   - Previous transcripts (if returning/transfer)
   - Entrance exam results (if applicable)
   ↓
4. Find available classroom for (Level, Track)
   ↓ [Found]
   Create Enrollment with state=PENDING, assign classroom_id
   ↓
5. Generate tuition invoice
   ↓ [Paid]
   Enrollment → ACTIVE
   ↓
6. Generate student identifier (StudentIdentifier)
   Create ClassroomSubject assignments
   Add to classroom attendance roster
```

#### 6.7.2 Mid-Year Transfer (Incoming)

```
1. Request from receiving school
2. Validate:
   - Current enrollment at sending school exists
   - Student is in good standing (no disciplinary holds)
   - Sending school releases records (digital transcript)
3. Create Enrollment with state=BLOCKED
4. Validate against current SchoolYearCategory:
   - Age-appropriate Level?
   - Available classroom space?
5. If validation passes:
   - Transfer previous Grades (with source marking)
   - state → ACTIVE
   - Notify teachers of transfer-in
6. If validation fails:
   - state → REJECTED, reason logged
   - Notify sending school
```

#### 6.7.3 Year-End Promotion Process

```
1. SchoolYear closes → all SYCTs in SCHOOLYEAR move to GRADING
2. Academic Council reviews all ACTIVE enrollments
3. For each enrollment:
   - Calculate: average_grade, absences_count, teacher_recommendation
   - Apply promotion rules (per category/level)
   - Decision: PASS/FAIL/TRANSFER/GRADUATE
4. Record decision:
   enrollment.promotion_decision = X
   enrollment.promotion_to_level_id = (next level if PASS)
   enrollment.promotion_to_track_id = (next track if promoted with track change)
5. Generate promotion report
6. Year-end archival:
   - All enrollments → COMPLETED
   - Create next year's enrollment records for returning students (pre-populated)
```

---

## 7. ClassroomSubject Domain Rules

*(Preview for TDR-004)*

While ClassroomSubject is in Étape 3, its domain rules are derived from SubjectChoice:

1. **R-CS-001**: When Classroom is created OR enrollment batch processed:
   - For each SubjectChoice where `is_required = true`:
     Create ClassroomSubject with matching coefficient and weekly_hours
   - For each SubjectChoice where `is_elective = true`:
     Create ClassroomSubject but allow student to opt-out

2. **R-CS-002**: ClassroomSubject defines:
   - Which Teacher teaches this Subject to this Classroom
   - Room assignment for this Subject-Classroom combination
   - Special scheduling constraints (double periods, labs)

3. **R-CS-003**: Student-specific ClassroomSubject enrollment:
   - Electives: Student may select from available electives
   - Required: Automatically assigned
   - Creates Grade records linked to (Student, ClassroomSubject, Term)

---

## 8. Cross-Cutting Constraints for Core Structure

### 8.1 Temporal Integrity

**All dates must satisfy**:
```
SchoolYear.start_date ≤ SchoolYear.end_date
SchoolYearCategoryTerm.start_date ≥ Term.start_date
SchoolYearCategoryTerm.end_date ≤ Term.end_date
Consecutive SYCTs for same SchoolYearCategory:
    previous.end_date < next.start_date
```

### 8.2 Enrollment Caps

**System-wide constraints**:
```
Sum(current_enrollment_count) for all Classrooms in a SchoolYearCategory
    ≤ configured_max_total_students_for_category

Individual Classroom:
    current_enrollment_count ≤ max_capacity
```

### 8.3 Track Offering Consistency

If a Track exists for a Level, there must be at least one Classroom offering it per SchoolYear:
```
FOR EACH (Level, Track) where Track.is_active = true:
    EXISTS Classroom with level_id = Level.id AND track_id = Track.id
        AND classroom.school_year_category_id = current_category
```
Exception: Tracks with enrollment below threshold may be consolidated.

---

## 9. Offline-First Considerations

### 9.1 Critical Data Caching Requirements

Offline devices must cache:
- **All Active Classrooms** (for enrollment lookups)
- **Current SchoolYear** and its SchoolYearCategories
- **All Terms and SYCTs** for current SchoolYear
- **Level and Track definitions** (for enrollment determination)

### 9.2 Offline Enrollment Limitations

**Allowed Offline**:
- Initiate enrollment (PENDING state)
- Document upload (stored locally until sync)
- Capture signatures

**NOT Allowed Offline**:
- Finalize enrollment (ACTIVE) — requires seat availability check
- Change classroom assignment
- Withdraw enrollment
- Create new Classroom

### 9.3 Sync Merge Strategy

When syncing offline-created enrollments:

1. Check seat availability (conflict if classroom full)
2. Validate prerequisites (might conflict with server-side rules)
3. Resolution:
   - If conflict: enrollment remains PENDING, notify admin
   - If no conflict: enrollment → ACTIVE, propagate to server

### 9.4 Conflict Scenarios

| Scenario | Detection | Resolution |
|----------|-----------|------------|
| Two devices enroll same student in same classroom | Duplicate enrollment_number or (student+year) | Server rejects duplicate; use PENDING queue |
| Offline enrollment exceeds classroom capacity | Sync-time count check | Enrollment stays PENDING, admin notified |
| Offline track assignment conflicts with classroom | Classroom track mismatch | Reassign classroom or require admin approval |

---

## 10. Performance Expectations

- Get active classrooms for Category: < 50ms
- Check student eligibility for Level: < 25ms
- Enrollment creation (with validation): < 200ms
- Promotion decision batch (1000 students): < 30 seconds

---

## 11. Data Integrity Summary

| Invariant | Enforced By | Violation Consequence |
|-----------|-------------|---------------------|
| Non-overlapping SYCT dates | DB constraint (EXCLUDE) + app validation | Scheduling conflicts |
| Classroom capacity | DB check + trigger | Overcrowding |
| One enrollment per student per year | Unique constraint + app guard | Duplicate enrollments |
| Track consistency | DB foreign key + app validation | Students in wrong classes |
| Term count matches AcademicYearType | SchoolYear creation workflow | System instability |

---

**Next Step**: TDR-004 — Domain Rules: Evaluations & Grades (EvaluationType, EvaluationCategory, Evaluation, EvaluationSubject, GradeType, Grade)
