# TDR-004 — Domain Rules: Evaluations & Grades

## 1. Purpose and Scope

This document defines the domain rules for the **most constrained** part of the School Management System: evaluations and grades (Étape 4).

This domain has:
- **Legal compliance requirements** (report cards, transcripts, diplomas)
- **Academic integrity rules** (grade security, audit trails)
- **Mathematical precision** (weighted averages, coefficients, rounding)
- **Strict workflows** (submission, review, finalization, appeals)
- **Multi-scale support** (0-20, 0-10, percentages, letter grades)

**Critical Principle**: Grade data is **immutable once finalized** except through formal appeal process with audit trail.

---

## 2. EvaluationType Domain Rules

### 2.1 Definition

**EvaluationType** classifies evaluations into two major categories:

| Type | Purpose | Examples |
|------|---------|----------|
| ACADEMIC | Curriculum-based evaluations that count toward promotion | Quizzes, Tests, Exams, Projects |
| INTERNAL | Administrative/behavioral assessments that don't affect GPA | Participation, Conduct, Attendance, Effort |

### 2.2 Domain Rules

1. **R-ET-001**: EvaluationType is **IMMUTABLE** after creation
   - Reason: Grade calculations and reporting depend on type classification
   - Only system administrator can create new types during initialization

2. **R-ET-002**: Standard EvaluationTypes that MUST exist:
   - **ACADEMIC**:
     - "QUIZ" — short evaluation, low weight
     - "TEST" — test/surprise test
     - "COMPOSITION" — formal written exam
     - "PROJECT" — extended project with rubric
     - "PRACTICAL" — lab/workshop evaluation
     - "ORAL" — oral examination
     - "EXAM" — final cumulative exam
   - **INTERNAL**:
     - "PARTICIPATION" — class engagement
     - "CONDUCT" — behavior/discipline
     - "ATTENDANCE" — attendance score
     - "HOMEWORK" — homework completion (not graded for quality)

3. **R-ET-003**: EvaluationType determines:
   - Whether the evaluation affects `student_average`
   - Whether it requires teacher signature
   - Inclusion in report card templates
   - Weight multipliers (final exams may have 2x weight)

4. **R-ET-004**: `is_weighted_average` flag:
   - If `true`: Evaluation's grade contributes to GPA via coefficient
   - If `false`: Evaluation is informational only (e.g., attendance)

5. **R-ET-005**: `requires_justification` flag:
   - If `true`: Teacher must enter comments justifying grade
   - Applies to final exams and any grade below minimum threshold

**Fields**:
- `id` (UUID)
- `code` (VARCHAR[20], unique) — "QUIZ", "TEST", "COMPOSITION"
- `name` (VARCHAR[100])
- `type_category` (VARCHAR[20]) — "ACADEMIC", "INTERNAL"
- `is_weighted_average` (BOOLEAN, default TRUE for ACADEMIC)
- `requires_justification` (BOOLEAN)
- `weight_multiplier` (DECIMAL) — default 1.0, can be 2.0 for final exams
- `is_active` (BOOLEAN)
- Audit trail

---

## 3. EvaluationCategory Domain Rules

### 3.1 Definition

**EvaluationCategory** provides a second-level classification within an EvaluationType, typically aligned with official curriculum divisions.

### 3.2 Domain Rules

1. **R-EC-001**: EvaluationCategory groups evaluations for reporting:
   ```
   EvaluationType = "COMPOSITION"
   EvaluationCategory choices: "MIDTERM", "FINAL"

   EvaluationType = "PROJECT"
   EvaluationCategory choices: "INDIVIDUAL", "GROUP", "RESEARCH"

   EvaluationType = "TEST"
   EvaluationCategory choices: "CHAPTER", "UNIT", "REVIEW"
   ```

2. **R-EC-002**: Categories are scoped to EvaluationType:
   - Not all categories apply to all types
   - Predefined mappings during system setup
   - Cannot create arbitrary category-type combinations

3. **R-EC-003**: Standard mandatory categories by EvaluationType:
   - COMPOSITION: "MIDTERM", "FINAL"
   - TEST: "CHAPTER", "UNIT"
   - PROJECT: "INDIVIDUAL", "GROUP"
   - PRACTICAL: "LAB", "FIELD"

4. **R-EC-004**: `order` field determines display sequence in report cards
   - e.g., FINAL appears after MIDTERM
   - Unique within EvaluationType

5. **R-EC-005**: Some categories are promotion-determining:
   - `is_promotion_critical = true` for final exams
   - Missing grade in promotion-critical category triggers review

**Fields**:
- `id` (UUID)
- `code` (VARCHAR[20], unique) — scoped: "COMPOSITION_FINAL"
- `name` (VARCHAR[100])
- `evaluation_type_id` (UUID, foreign key to EvaluationType)
- `description` (TEXT)
- `order` (INTEGER)
- `is_promotion_critical` (BOOLEAN, default FALSE)
- `is_active` (BOOLEAN)
- Audit trail

---

## 4. Evaluation Domain Rules

### 4.1 Definition

**Evaluation** represents a specific assessment instrument administered to a Classroom (group of students) for a specific SYCT (term period).

This is the **central transactional entity** in the grading domain.

### 4.2 Domain Rules

1. **R-EVAL-001**: Evaluation creation requirements:
   - Must belong to an `OPEN` SYCT (SchoolYearCategoryTerm)
   - Must have a valid `classroom_subject_id` (the Subject being evaluated)
   - Must specify `evaluation_date` within SYCT's date range
   - Teacher creating it must be assigned to that ClassroomSubject

2. **R-EVAL-002**: Evaluation code format: `[school_year_code]-[class_subj_hash]-[seq]`
   - Example: "2024-2025-MATH5A-001"
   - Auto-generated sequential within the classroom_subject + term

3. **R-EVAL-003**: Evaluation lifecycle state machine:
   ```
   DRAFT → PUBLISHED → IN_PROGRESS → SUBMITTED → REVIEW → FINALIZED
   ```

   **State Transitions**:
   | From | To | Allowed Conditions | Who Can Execute |
   |------|-----|-------------------|-----------------|
   | DRAFT | PUBLISHED | All students in classroom enrolled, evaluation instruments ready | Teacher |
   | PUBLISHED | IN_PROGRESS | Evaluation date arrives (or early for special cases) | Teacher |
   | IN_PROGRESS | SUBMITTED | All grades entered and justified | Teacher |
   | SUBMITTED | REVIEW | Auto upon submission | System |
   | REVIEW | FINALIZED | Academic council approves or review timeout | Academic Inspector |
   | ANY | ARCHIVED | Only after FINALIZED and 1 year | System Admin |

4. **R-EVAL-004**: Evaluation configuration:
   - `max_score` — maximum possible grade (usually 20, or 100, etc.)
   - `passing_score` — minimum passing grade (varies by subject/level)
   - `coefficient` — weight multiplier (1 = normal, 2 = double weight)
   - `duration_minutes` — for scheduling exams
   - `is_average_computed` — if TRUE, contributes to student average

5. **R-EVAL-005**: Evaluation scope:
   - An Evaluation is given to **ALL students** in the Classroom
   - Exceptions: `allow_absences = true` for makeup exams
   - Individual accommodations noted in `special_instructions`

6. **R-EVAL-006**: Evaluation revision policy:
   - Once `PUBLISHED`, Evaluation's configuration (max_score, coefficient) CANNOT change
   - If configuration change needed: RETRACT, create new Evaluation
   - Reason: students may have already taken the evaluation

7. **R-EVAL-007**: Date constraints:
   - `evaluation_date` must be within `[SYCT.actual_start_date, SYCT.actual_end_date]`
   - `grade_submission_deadline` = `SYCT.evaluation_submission_deadline`
   - Late submissions require `late_submission_reason` and principal approval

8. **R-EVAL-008**: Evaluation groupings:
   - `group_evaluations` — multiple evaluations can be grouped for combined average
   - Used for: semester average = (midterm + final) / 2
   - `parent_evaluation_id` for grouping

**Fields**:
- `id` (UUID)
- `code` (VARCHAR[50], unique)
- `school_year_category_term_id` (UUID, foreign key to SYCT)
- `classroom_subject_id` (UUID, foreign key to ClassroomSubject)
- `evaluation_type_id` (UUID, foreign key to EvaluationType)
- `evaluation_category_id` (UUID, foreign key to EvaluationCategory)

- `name` (VARCHAR[200]) — "Test on Chapter 3 - Algebra"
- `description` (TEXT) — instructions, topics covered
- `evaluation_date` (DATE)
- `duration_minutes` (INTEGER)

- `max_score` (DECIMAL) — typically 20.00
- `passing_score` (DECIMAL)
- `coefficient` (DECIMAL, default 1.0)
- `is_average_computed` (BOOLEAN, default TRUE)

- `state` (VARCHAR[20]) — "DRAFT", "PUBLISHED", "IN_PROGRESS", "SUBMITTED", "REVIEW", "FINALIZED", "ARCHIVED"
- `grade_submission_deadline` (DATE)
- `allow_absences` (BOOLEAN, default FALSE)
- `allows_makeup` (BOOLEAN, default FALSE)

- `total_students_expected` (INTEGER) — snapshot of classroom count at publish time
- `total_students_completed` (INTEGER) — updates as grades enter
- `completion_percentage` (DECIMAL, computed)

- `teacher_notes` (TEXT, private to teacher)
- `special_instructions` (TEXT, visible to students)
- `parent_evaluation_id` (UUID, self-reference for grouping)

- `is_active` (BOOLEAN)
- Audit trail (submitted_at, submitted_by, finalized_at, finalized_by)

---

### 4.3 Critical Invariants

**I-EVAL-001 (Completion Tracking)**:
```
total_students_completed ≤ total_students_expected
```
Enforced via trigger on Grade insert.

**I-EVAL-002 (Finalized Integrity)**:
```
IF state = 'FINALIZED':
    total_students_completed = total_students_expected
    OR (allow_absences = TRUE AND exempted_count documented)
```
Finalization blocked if not all grades submitted (unless exempted).

**I-EVAL-003 (No Backdating)**:
```
evaluation_date ≤ TODAY()
```
Prevents future-dated evaluations (may confuse parents/students).

**I-EVAL-004 (Score Range)**:
All entered Grade.values must satisfy:
```
0 ≤ grade.value ≤ evaluation.max_score
```

**I-EVAL-005 (Parent-Child Coefficient)**:
If `parent_evaluation_id` exists (grouped evaluations):
```
parent_evaluation.coefficient = SUM(child_evaluation.coefficient)
    (or parent coefficient = 1 and children sum to equivalent weight)
```

---

## 5. Grade Domain Rules

### 5.1 Definition

**Grade** is the actual score recorded for a specific Student on a specific Evaluation.

This is the **most sensitive data** in the system — must be immutable once finalized, auditable, and legally defensible.

### 5.2 Grade State Machine

```
PENDING → ENTERED → SUBMITTED → REVIEWED → FINALIZED
      ↘ (rejected)  ↘ (returned)
```

**State Transitions**:

| From | To | Trigger | Who | Notes |
|------|-----|---------|-----|-------|
| PENDING | ENTERED | Teacher enters grade | Teacher | Grade can be saved as draft |
| ENTERED | SUBMITTED | Teacher submits for review | Teacher | Locks grade from teacher edit |
| SUBMITTED | REVIEWED | Academic review completes | Academic Inspector | May approve or return |
| REVIEWED | FINALIZED | Post-review approval or auto-timeout | System/Inspector | Grade becomes immutable |
| SUBMITTED/REVIEWED | RETURNED | Review finds issue | Inspector | Returns to teacher with comments |
| FINALIZED | (Appeal Started) | Formal grade appeal filed | Parent/Student | Creates audit trail, does NOT modify grade |

### 5.3 Domain Rules

1. **R-GRADE-001**: Grade creation requirements:
   - Student must be enrolled in the Evaluation's Classroom
   - Evaluation must be in `PUBLISHED` or later state
   - Student must be marked present OR have valid absence excuse with makeup permission

2. **R-GRADE-002**: Grade value rules:
   - Must be `0 ≤ value ≤ evaluation.max_score`
   - Cannot be NULL unless `is_exempt = TRUE`
   - Precision: 2 decimal places (0.01 granularity)
   - Default: `NULL` (pending)

3. **R-GRADE-003**: Exemption handling:
   - `is_exempt = TRUE`: Student excused from this evaluation
   - `exemption_reason` (TEXT) required
   - Excluded from average calculation
   - Final grade calculation: treat as N/A, not 0

4. **R-GRADE-004**: Grade modification after submission:
   - ENTERED → SUBMITTED: Teacher can edit (before submission deadline)
   - SUBMITTED → RETURNED: Inspector returns, teacher corrects
   - FINALIZED → **NO DIRECT MODIFICATION**
     - Only via formal Grade Appeal process
     - Creates new Grade record linked to original (audit trail)

5. **R-GRADE-005**: Grade Appeal process:
   ```
   Original Grade (FINALIZED)
        ↓
   Grade Appeal created (parent requests review)
        ↓
   Academic Council reviews
        ↓ [Approved]
   Amendment Grade created with override flag
        ↓
   Original grade preserved (read-only)
   Amendment grade used for calculations
   Appeal recorded in audit log
   ```
   - Appeals must be filed within 30 days of report card distribution
   - Grade amendments require principal + inspector dual approval
   - All appeal documentation retained for 10+ years

6. **R-GRADE-006**: Grade computation rules:
   ```
   Weighted Sum = Σ(grade.value × evaluation.coefficient) for all completed evaluations
   Total Coefficient = Σ(evaluation.coefficient) for all evaluated items
   Student Average = Weighted Sum / Total Coefficient
   ```
   - Rounded to 2 decimal places (hormonic rounding — 12.345 → 12.35)
   - Exempted evaluations excluded from both sum and total

7. **R-GRADE-007**: Grade scale mapping:
   - Numerical grades (0-20) stored as-is
   - Letter grades (A-F) must have corresponding numeric value in GradeType
   - Conversion happens at reporting layer, not storage

8. **R-GRADE-008**: `justification` field:
   - Required if `grade.value < passing_score`
   - Required if `is_exempt = TRUE`
   - Teacher enters rationale for low/exempt grade
   - Visible to parents in report card

**Fields**:
- `id` (UUID)
- `evaluation_id` (UUID, foreign key to Evaluation, NOT NULL)
- `student_profile_id` (UUID, foreign key to StudentProfile, NOT NULL)
- `enrollment_id` (UUID, foreign key to Enrollment) — denormalized for reporting

- `value` (DECIMAL(5,2)) — NULL if is_exempt
- `is_exempt` (BOOLEAN, default FALSE)
- `exemption_reason` (TEXT)

- `state` (VARCHAR[20]) — "PENDING", "ENTERED", "SUBMITTED", "REVIEWED", "FINALIZED", "RETURNED"
- `justification` (TEXT)

- `entered_by` (UUID, foreign key to TeacherProfile)
- `entered_at` (TIMESTAMPTZ)
- `submitted_at` (TIMESTAMPTZ)
- `reviewed_by` (UUID, foreign key to TeacherProfile/AcademicInspector)
- `reviewed_at` (TIMESTAMPTZ)
- `finalized_at` (TIMESTAMPTZ)
- `finalized_by` (UUID)

- `is_amendment` (BOOLEAN, default FALSE) — true if result of grade appeal
- `amends_grade_id` (UUID, self-reference) — links to original grade
- `grade_appeal_id` (UUID, foreign key to GradeAppeal table)

- `is_active` (BOOLEAN)
- Audit trail

**Unique Constraint**:
- `UNIQUE (evaluation_id, student_profile_id)` — one grade per evaluation per student

**Composite Indexes**:
- `(student_profile_id, evaluation_id)` — student grade lookup
- `(evaluation_id, state)` — find pending grades for teacher
- `(enrollment_id, evaluation_id)` — classroom grade sheet queries

---

### 5.4 Grade Immutability Pattern

Once a Grade is FINALIZED:

1. **Direct UPDATE/DELETE blocked** at database level:
```sql
CREATE TRIGGER grade_prevent_finalized_update
    BEFORE UPDATE ON grade
    FOR EACH ROW
    WHEN (OLD.state = 'FINALIZED')
    EXECUTE FUNCTION raise_exception('Finalized grades cannot be modified');
```

2. **Amendment via appeal**:
   - Original grade marked as `is_amended = TRUE`
   - New grade created with `is_amendment = TRUE` and `amends_grade_id`
   - Calculations use amendment grade, original preserved
   - Appeal record links both

3. **Audit trail complete**:
   - Every change logged with user, timestamp, reason
   - 10-year retention mandated

---

## 6. GradeType Domain Rules

### 6.1 Definition

**GradeType** defines the grading scale used for a particular Level (e.g., Primary uses 0-20, College may use 0-20 or percentages, University uses GPA 0-4).

### 6.2 Domain Rules

1. **R-GT-001**: GradeType is **level-specific** but **school-year-agnostic**
   - Defined at system initialization
   - Referenced by Level during setup
   - Cannot change for Level once students have grades

2. **R-GT-002**: Standard GradeTypes:
   - "TWENTY": 0.00 to 20.00 (French/ francophone system)
   - "TEN": 0.0 to 10.0 (some European countries)
   - "PERCENTAGE": 0 to 100%
   - "LETTER": A, B, C, D, F (US system) — requires mapping table
   - "GPA_4": 0.00 to 4.00 (US college)
   - "CUSTOM": user-defined scale

3. **R-GT-003**: GradeType defines:
   - `min_value` and `max_value` (validation bounds)
   - `passing_threshold` (e.g., 10.00 for 0-20 scale, 2.0 for GPA)
   - `rounding_method` ("STANDARD", "UP", "DOWN", "TRUNCATE")
   - `display_format` ("NUMERIC", "LETTER", "BOTH")

4. **R-GT-004**: Letter grade mapping for LETTER type:
   - Stored in separate table `grade_type_mapping`
   - Maps: A → 4.0, B → 3.0, C → 2.0, D → 1.0, F → 0.0
   - Used for GPA calculations and transcript conversion

5. **R-GT-005**: Level inherits GradeType, but can override:
   - Level.default_grade_type_id
   - SchoolYearCategory can set `override_grade_type_id` if needed
   - Evaluation may specify `grade_type_override` for special cases

**Fields (GradeType)**:
- `id` (UUID)
- `code` (VARCHAR[20], unique) — "TWENTY", "PERCENTAGE", "GPA_4"
- `name` (VARCHAR[100])
- `min_value` (DECIMAL)
- `max_value` (DECIMAL)
- `passing_threshold` (DECIMAL)
- `rounding_method` (VARCHAR[20])
- `display_format` (VARCHAR[20])
- `is_active` (BOOLEAN)
- Audit trail

**Fields (GradeTypeMapping for letter grades)**:
- `id` (UUID)
- `grade_type_id` (UUID)
- `letter_grade` (CHAR[1] or CHAR[2]) — "A", "B+", etc.
- `numeric_value` (DECIMAL) — GPA value
- `min_percent` (INTEGER) — for conversion (optional)
- `max_percent` (INTEGER)
- `is_passing` (BOOLEAN)

---

## 7. Calculation Domain Rules

### 7.1 Average Calculation

**Per-Term Average** (for a Student in a SYCT):
```
term_average = Σ(grade.value × evaluation.coefficient) / Σ(evaluation.coefficient)
              over all FINALIZED grades for that SYCT
```

**Year Average** (cumulative across all SYCTs in SchoolYear):
```
year_average = Σ(term_average × term_coefficient) / Σ(term_coefficient)
```
Where `term_coefficient` depends on AcademicYearType:
- TRIMESTER: T1=1, T2=1, T3=2 (final term weighted heavier)
- SEMESTER: S1=1, S2=1 (equal)
- Configurable per Category

**Class Average** (for a ClassroomSubject):
```
class_average = AVG(student_average) over all enrolled students
```
Used for class ranking and performance metrics.

### 7.2 Promotion Determination

At SchoolYearCategory FINALIZATION, for each Enrollment:

```
IF year_average ≥ promotion_threshold AND
   absences ≤ max_absences_allowed AND
   all promotion_critical evaluations have grade ≥ passing_score AND
   no disciplinary holds:
    promotion_decision = "PASS"
    IF level.next_level_id exists:
        promotion_to_level_id = level.next_level_id
    ELSE:
        promotion_decision = "GRADUATE"
ELSE IF year_average between retention_threshold_low and retention_threshold_high:
    promotion_decision = "RETAIN" (repeat same level)
ELSE IF year_average < retention_threshold_low AND absences > limit:
    promotion_decision = "EXIT" (leave school)
ELSE:
    promotion_decision = "REVIEW" (Academic Council decides case-by-case)
```

**Thresholds (per Category/Level)**:
- `promotion_threshold` = passing_average (e.g., 10.00/20)
- `retention_threshold_low` = 8.00 (below this = automatic retention)
- `retention_threshold_high` = 9.99 (between = council review)

---

## 8. Academic Integrity Rules

### 8.1 Grade Security

1. **Sealed Grades**: Once FINALIZED, grade value is cryptographically sealed
   - Hash stored: `SHA256(evaluation_id + student_id + value + finalized_at + finalized_by)`
   - Any modification invalidates the seal
   - Seal verification on report card generation

2. **Submission Deadlines**:
   - Grades must be submitted within `SYCT.grade_finalization_deadline`
   - Extensions require principal + inspector approval
   - Late submission flag logged

3. **Change Approval Workflow**:
   ```
   Grade Change Request (after finalization)
        ↓
   Justification documented
        ↓
   Principal approval required
        ↓
   Academic Inspector approval required
        ↓
   Amendment Grade created (original preserved)
   ```
   Max 3 amendments per grade (prevents grade inflation abuse)

### 8.2 Audit Requirements

All grade-related operations logged:
- Who entered/changed grade (user_id)
- When (timestamp)
- What changed (old_value, new_value)
- Why (justification/comment)
- IP address for online submissions

Retention: 10+ years (legal requirement for transcripts).

---

## 9. Reporting Constraints

### 9.1 Report Card Generation

Report cards generated when all evaluations in SYCT are FINALIZED:
- Student average per Subject
- Class ranking (optional)
- Teacher comments (from Evaluation.justification or separate)
- Attendance summary
- Promotion recommendation

**Report Card State**:
- `GENERATED` — PDF created
- `DISTRIBUTED` — given to parent/student
- `ACKNOWLEDGED` — parent signature received (digital or physical)

### 9.2 Transcript Generation

Transcript aggregates all FINALIZED grades across SchoolYears:
- Year-by-year breakdown
- Cumulative GPA (converted to standard scale)
- Promotion/graduation status
- Diplomas awarded

Transcripts are **read-only official documents**:
- Generated on-demand with digital signature
- Any subsequent grade appeals generate **amended transcript** with correction note

---

## 10. Offline-First Considerations

### 10.1 Offline Grade Entry

**Allowed**:
- Teachers can enter grades offline for already-PUBLISHED evaluations
- Grades stored locally with state = ENTERED
- Sync upon reconnection

**Constraints**:
- Evaluation must exist in local cache (downloaded while online)
- Cannot create new Evaluation offline
- Cannot SUBMIT grades offline (requires server validation of deadlines)

### 10.2 Sync Conflict Resolution

| Conflict Type | Resolution |
|---------------|------------|
| Two teachers edit same grade | Last-write-wins with conflict flag, notify admin |
| Offline grade contradicts max_score | Reject with error message |
| Offline submission after deadline | Server rejects, teacher must request extension |
| Evaluation modified on server after offline publish | Invalidate offline evaluation, teacher must re-publish |

### 10.3 Critical Offline Read-Only Data

Teachers must have cached:
- Current SYCTs with `state = OPEN`
- All PUBLISHED evaluations for their classrooms
- Student roster for those classrooms
- GradeType configuration for their subjects

---

## 11. Invariants Summary

| Invariant | Enforcement |
|-----------|--------------|
| One grade per (evaluation, student) | Unique constraint |
| Grade value within evaluation.max_score | CHECK constraint |
| Finalized grade immutable | Trigger + application guard |
| All classroom students represented | Business rule on evaluation publish |
| Term average calculated from FINALIZED only | Computed column trigger |
| Promotion decision based on year average | Application logic with audit |

---

## 12. Performance Targets

- Grade entry (per student): < 100ms
- Class average calculation (30 students): < 500ms
- Report card generation (100 students): < 10 seconds
- Transcript retrieval (5 years): < 2 seconds

---

**Next Step**: TDR-005 — Data Architecture: Evaluations & Grades Tables
