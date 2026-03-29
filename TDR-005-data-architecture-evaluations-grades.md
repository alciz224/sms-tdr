# TDR-005 — Data Architecture: Evaluations & Grades Tables

## 1. Purpose and Scope

This document defines the physical data architecture for the Evaluations & Grades domain (Étape 4). It specifies:

- Complete SQL DDL for EvaluationType, EvaluationCategory, Evaluation, Grade, and GradeType
- Comprehensive indexing strategy for high-volume grade queries
- Partitioning strategy for long-term grade retention
- Synchronization patterns for offline-first grade entry
- Security mechanisms for grade integrity

**Key Challenge**: This domain handles the highest-volume transactional data (30+ grades per student per term) and requires strict integrity for 10+ years.

---

## 2. EvaluationType Table (evaluation_type)

```sql
CREATE TABLE evaluation_type (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Natural Key (immutable)
    code VARCHAR(20) NOT NULL UNIQUE,  -- "QUIZ", "TEST", "COMPOSITION", "PROJECT", "PARTICIPATION"

    -- Descriptive
    name VARCHAR(100) NOT NULL,
    description TEXT,

    -- Classification
    type_category VARCHAR(20) NOT NULL CHECK (
        type_category IN ('ACADEMIC', 'INTERNAL')
    ),

    -- Evaluation Behavior
    is_weighted_average BOOLEAN NOT NULL DEFAULT TRUE,
    weight_multiplier DECIMAL(4,2) NOT NULL DEFAULT 1.00 CHECK (weight_multiplier > 0),

    -- Workflow
    requires_justification BOOLEAN NOT NULL DEFAULT FALSE,
    requires_signature BOOLEAN NOT NULL DEFAULT FALSE,

    -- Reporting
    appears_on_report_card BOOLEAN NOT NULL DEFAULT TRUE,
    display_order INTEGER NOT NULL DEFAULT 0,

    -- Control
    is_active BOOLEAN NOT NULL DEFAULT TRUE,

    -- Audit Trail
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by UUID NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by UUID,
    archived_at TIMESTAMPTZ,
    archived_by UUID,

    -- Constraints
    CONSTRAINT evaluation_type_code_check CHECK (code ~ '^[A-Z0-9_]+$')
);

-- Indexes
CREATE INDEX idx_evaluation_type_code ON evaluation_type(code);
CREATE INDEX idx_evaluation_type_category ON evaluation_type(type_category);
CREATE INDEX idx_evaluation_type_is_active ON evaluation_type(is_active) WHERE is_active = TRUE;
CREATE INDEX idx_evaluation_type_weighted ON evaluation_type(is_weighted_average) WHERE is_weighted_average = TRUE;

-- This table is tiny (<30 rows), no partitioning needed
```

---

## 3. EvaluationCategory Table (evaluation_category)

```sql
CREATE TABLE evaluation_category (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Natural Key: scoped to EvaluationType
    code VARCHAR(30) NOT NULL,  -- e.g., "COMPOSITION_FINAL", "TEST_CHAPTER"
    name VARCHAR(100) NOT NULL,

    -- Foreign Key
    evaluation_type_id UUID NOT NULL REFERENCES evaluation_type(id) ON DELETE RESTRICT,

    -- Denormalized
    evaluation_type_code VARCHAR(20) NOT NULL,

    -- Configuration
    description TEXT,
    display_order INTEGER NOT NULL DEFAULT 0,
    is_promotion_critical BOOLEAN NOT NULL DEFAULT FALSE,

    -- Control
    is_active BOOLEAN NOT NULL DEFAULT TRUE,

    -- Audit Trail
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by UUID NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by UUID,
    archived_at TIMESTAMPTZ,
    archived_by UUID,

    -- Uniqueness: Code unique within EvaluationType, or globally unique?
    -- Approach: Global codes like "COMPOSITION_FINAL" to simplify
    UNIQUE (code),

    -- Also ensure (evaluation_type_id, name) uniqueness for duplicate prevention
    UNIQUE (evaluation_type_id, name)
);

CREATE INDEX idx_evaluation_category_code ON evaluation_category(code);
CREATE INDEX idx_evaluation_category_type_id ON evaluation_category(evaluation_type_id);
CREATE INDEX idx_evaluation_category_promotion ON evaluation_category(is_promotion_critical) WHERE is_promotion_critical = TRUE;
CREATE INDEX idx_evaluation_category_is_active ON evaluation_category(is_active) WHERE is_active = TRUE;

-- Trigger to denormalize evaluation_type_code
```

---

## 4. Evaluation Table (evaluation)

### 4.1 Physical Design

```sql
CREATE TABLE evaluation (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Natural Business Key (auto-generated)
    code VARCHAR(50) NOT NULL UNIQUE,  -- e.g., "2024-2025-MATH1A-001"

    -- Foreign Keys (core relationships)
    school_year_category_term_id UUID NOT NULL REFERENCES school_year_category_term(id) ON DELETE RESTRICT,
    classroom_subject_id UUID NOT NULL REFERENCES classroom_subject(id) ON DELETE CASCADE,
    evaluation_type_id UUID NOT NULL REFERENCES evaluation_type(id) ON DELETE RESTRICT,
    evaluation_category_id UUID REFERENCES evaluation_category(id) ON DELETE SET NULL,

    -- Denormalized for reporting (snapshots at creation)
    school_year_code VARCHAR(20) NOT NULL,
    term_code VARCHAR(10) NOT NULL,
    classroom_code VARCHAR(50) NOT NULL,
    subject_code VARCHAR(20) NOT NULL,
    teacher_id UUID REFERENCES teacher_profile(id),  -- denormalized from classroom_subject

    -- Core Configuration
    name VARCHAR(200) NOT NULL,
    description TEXT,
    evaluation_date DATE NOT NULL,
    duration_minutes INTEGER,

    -- Grading Configuration
    max_score DECIMAL(6,2) NOT NULL CHECK (max_score > 0),  -- typically 20.00
    passing_score DECIMAL(6,2) NOT NULL,
    coefficient DECIMAL(4,2) NOT NULL DEFAULT 1.00 CHECK (coefficient > 0),
    is_average_computed BOOLEAN NOT NULL DEFAULT TRUE,

    -- State
    state VARCHAR(20) NOT NULL CHECK (
        state IN ('DRAFT', 'PUBLISHED', 'IN_PROGRESS', 'SUBMITTED', 'REVIEW', 'FINALIZED', 'ARCHIVED')
    ),

    -- Dates & Deadlines
    published_at TIMESTAMPTZ,
    grade_submission_deadline DATE NOT NULL,
    grade_finalization_deadline DATE,

    -- Options
    allow_absences BOOLEAN NOT NULL DEFAULT FALSE,
    allows_makeup BOOLEAN NOT NULL DEFAULT FALSE,
    requires_individual_accommodations BOOLEAN NOT NULL DEFAULT FALSE,

    -- Grouping (for composite evaluations like midterm+final)
    parent_evaluation_id UUID REFERENCES evaluation(id) ON DELETE SET NULL,
    is_group_parent BOOLEAN NOT NULL DEFAULT FALSE,

    -- Progress Tracking (denormalized for quick dashboard)
    total_students_expected INTEGER NOT NULL,
    total_students_completed INTEGER NOT NULL DEFAULT 0,
    total_students_with_grade INTEGER NOT NULL DEFAULT 0,
    completion_percentage DECIMAL(5,2) GENERATED ALWAYS AS (
        CASE
            WHEN total_students_expected > 0
            THEN ROUND((total_students_completed::NUMERIC / total_students_expected) * 100, 2)
            ELSE 0
        END
    ) STORED,

    -- Content
    instructions TEXT,  -- visible to students
    teacher_notes TEXT,  -- private
    special_instructions TEXT,  -- accommodations

    -- Audit (state transition tracking)
    submitted_at TIMESTAMPTZ,
    submitted_by UUID REFERENCES teacher_profile(id),
    finalized_at TIMESTAMPTZ,
    finalized_by UUID REFERENCES teacher_profile(id),
    reviewed_at TIMESTAMPTZ,
    reviewed_by UUID REFERENCES teacher_profile(id),

    -- Control
    is_active BOOLEAN NOT NULL DEFAULT TRUE,

    -- Audit Trail
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by UUID NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by UUID,
    archived_at TIMESTAMPTZ,
    archived_by UUID
);

-- Indexes
CREATE INDEX idx_evaluation_code ON evaluation(code);
CREATE INDEX idx_evaluation_syct_id ON evaluation(school_year_category_term_id);
CREATE INDEX idx_evaluation_classroom_subject_id ON evaluation(classroom_subject_id);
CREATE INDEX idx_evaluation_teacher_id ON evaluation(teacher_id);
CREATE INDEX idx_evaluation_state ON evaluation(state);
CREATE INDEX idx_evaluation_syct_state ON evaluation(school_year_category_term_id, state);

-- Composite indexes for common queries
CREATE INDEX idx_evaluation_classroom_subject_term ON evaluation(classroom_subject_id, school_year_category_term_id);
CREATE INDEX idx_evaluation_published ON evaluation(state, published_at) WHERE state IN ('PUBLISHED', 'IN_PROGRESS', 'SUBMITTED');
CREATE INDEX idx_evaluation_teacher_pending ON evaluation(teacher_id, state) WHERE state = 'SUBMITTED' OR state = 'REVIEW';

-- For deadline monitoring
CREATE INDEX idx_evaluation_submission_deadline ON evaluation(grade_submission_deadline) WHERE state NOT IN ('FINALIZED', 'ARCHIVED');

-- Denormalized columns for reporting
CREATE INDEX idx_evaluation_school_year_code ON evaluation(school_year_code);
CREATE INDEX idx_evaluation_subject_code ON evaluation(subject_code);

-- Partitioning Strategy: By school_year_code for long-term retention
-- Create partitions: evaluation_y2024, evaluation_y2025, etc.
-- Pattern: Value = start year of school year (e.g., "2024" for "2024-2025")
```

### 4.2 Triggers and Constraints

```sql
-- 1. Prevent date backdating
CREATE OR REPLACE FUNCTION evaluation_prevent_backdate()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.evaluation_date > CURRENT_DATE THEN
        RAISE EXCEPTION 'Evaluation date cannot be in the future';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_evaluation_prevent_backdate
    BEFORE INSERT OR UPDATE ON evaluation
    FOR EACH ROW EXECUTE FUNCTION evaluation_prevent_backdate();

-- 2. Auto-calculate completion percentage trigger (redundant with generated column, but for migration)
-- 3. Denormalize school_year_code, term_code, classroom_code, subject_code, teacher_id
--    via trigger from joined tables

-- 4. Prevent state regression
CREATE OR REPLACE FUNCTION evaluation_prevent_state_regression()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.state = 'DRAFT' AND OLD.state != 'DRAFT' THEN
        RAISE EXCEPTION 'Cannot regress evaluation to DRAFT';
    END IF;
    -- Check state order valid transition
    IF NEW.state != OLD.state AND
       NOT is_valid_state_transition(OLD.state, NEW.state) THEN
        RAISE EXCEPTION 'Invalid state transition: % -> %', OLD.state, NEW.state;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_evaluation_state_transition
    BEFORE UPDATE OF state ON evaluation
    FOR EACH ROW EXECUTE FUNCTION evaluation_prevent_state_regression();

-- 5. Finalized evaluation protection
CREATE OR REPLACE FUNCTION evaluation_prevent_finalized_modification()
RETURNS TRIGGER AS $$
BEGIN
    IF OLD.state = 'FINALIZED' THEN
        RAISE EXCEPTION 'Cannot modify finalized evaluation';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_evaluation_finalized_protect
    BEFORE UPDATE ON evaluation
    FOR EACH ROW
    WHEN (OLD.state = 'FINALIZED')
    EXECUTE FUNCTION evaluation_prevent_finalized_modification();
```

---

## 5. Grade Table (grade)

### 5.1 Physical Design

```sql
CREATE TABLE grade (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Composite Foreign Keys
    evaluation_id UUID NOT NULL REFERENCES evaluation(id) ON DELETE CASCADE,
    student_profile_id UUID NOT NULL REFERENCES student_profile(id) ON DELETE CASCADE,

    -- Denormalized (for reporting without joins)
    enrollment_id UUID NOT NULL REFERENCES enrollment(id) ON DELETE CASCADE,
    school_year_category_term_id UUID NOT NULL,  -- denormalized from evaluation
    classroom_subject_id UUID NOT NULL,  -- denormalized from evaluation
    teacher_id UUID REFERENCES teacher_profile(id),  -- denormalized from evaluation

    -- Grade Value
    value DECIMAL(5,2),  -- NULL when is_exempt = TRUE
    is_exempt BOOLEAN NOT NULL DEFAULT FALSE,
    exemption_reason TEXT,

    -- State Machine
    state VARCHAR(20) NOT NULL CHECK (
        state IN ('PENDING', 'ENTERED', 'SUBMITTED', 'REVIEWED', 'FINALIZED', 'RETURNED')
    ),

    -- Justification (required for certain cases)
    justification TEXT,

    -- Audit Trail (state transitions)
    entered_by UUID REFERENCES teacher_profile(id),
    entered_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    submitted_at TIMESTAMPTZ,
    submitted_by UUID REFERENCES teacher_profile(id),
    reviewed_at TIMESTAMPTZ,
    reviewed_by UUID REFERENCES teacher_profile(id),
    finalized_at TIMESTAMPTZ,
    finalized_by UUID REFERENCES teacher_profile(id),

    -- Grade Amendment (appeals)
    is_amendment BOOLEAN NOT NULL DEFAULT FALSE,
    amends_grade_id UUID REFERENCES grade(id) ON DELETE SET NULL,
    grade_appeal_id UUID,  -- reference to grade_appeal table

    -- Control
    is_active BOOLEAN NOT NULL DEFAULT TRUE,

    -- Audit Trail
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by UUID NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by UUID,
    archived_at TIMESTAMPTZ,
    archived_by UUID,

    -- Constraints
    CONSTRAINT grade_value_range CHECK (
        is_exempt = TRUE OR (value IS NOT NULL AND value >= 0)
    ),
    CONSTRAINT grade_exemption_justification CHECK (
        is_exempt = FALSE OR exemption_reason IS NOT NULL
    ),
    CONSTRAINT grade_submission_requires_value CHECK (
        state IN ('PENDING', 'ENTERED') OR (is_exempt = FALSE AND value IS NOT NULL)
    ),

    -- Uniqueness: One grade per student per evaluation
    UNIQUE (evaluation_id, student_profile_id)
);

-- Indexes (heavily queried table)
CREATE INDEX idx_grade_evaluation_id ON grade(evaluation_id);
CREATE INDEX idx_grade_student_id ON grade(student_profile_id);
CREATE INDEX idx_grade_enrollment_id ON grade(enrollment_id);
CREATE INDEX idx_grade_state ON grade(state);
CREATE INDEX idx_grade_finalized ON grade(state, finalized_at) WHERE state = 'FINALIZED';

-- Composite indexes for common access patterns
-- 1. Teacher enters grades: get all PENDING/ENTERED for their evaluation
CREATE INDEX idx_grade_evaluation_state ON grade(evaluation_id, state);

-- 2. Student grade lookup: get all FINALIZED grades for student
CREATE INDEX idx_grade_student_finalized ON grade(student_profile_id, state) WHERE state = 'FINALIZED';

-- 3. Classroom grade sheet: get grades for all students in classroom for evaluation
CREATE INDEX idx_grade_classroom_subject ON grade(classroom_subject_id, evaluation_id);

-- 4. Report card generation: get grades by enrollment + state
CREATE INDEX idx_grade_enrollment_finalized ON grade(enrollment_id, state) WHERE state = 'FINALIZED';

-- 5. Grade audit trail
CREATE INDEX idx_grade_created_at ON grade(created_at);
CREATE INDEX idx_grade_updated_at ON grade(updated_at);
CREATE INDEX idx_grade_amendment ON grade(amends_grade_id) WHERE is_amendment = TRUE;

-- Denormalized columns
CREATE INDEX idx_grade_syct_id ON grade(school_year_category_term_id);
CREATE INDEX idx_grade_teacher_id ON grade(teacher_id);

-- Partitioning Strategy: HIGH PRIORITY
-- This table will be massive (millions of rows after 10 years)
-- Partition by school_year_code (denormalized) for time-based pruning
-- Also consider hash partition on student_id within each year for parallelism

-- Create partitions:
-- grade_y2024 PARTITION OF grade FOR VALUES IN ('2024')
-- grade_y2025 PARTITION OF grade FOR VALUES IN ('2025')
-- Partition key uses school_year_code extracted from enrollment

-- Alternative: Range partition on created_at for live data, archive old partitions
```

### 5.2 Triggers and Constraints

```sql
-- 1. Denormalize school_year_category_term_id, classroom_subject_id, teacher_id from evaluation on grade insert
CREATE OR REPLACE FUNCTION grade_denormalize_from_evaluation()
RETURNS TRIGGER AS $$
DECLARE
    eval_record evaluation%ROWTYPE;
BEGIN
    SELECT * INTO eval_record FROM evaluation WHERE id = NEW.evaluation_id;
    NEW.school_year_category_term_id = eval_record.school_year_category_term_id;
    NEW.classroom_subject_id = eval_record.classroom_subject_id;
    NEW.teacher_id = eval_record.teacher_id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_grade_denormalize
    BEFORE INSERT ON grade
    FOR EACH ROW EXECUTE FUNCTION grade_denormalize_from_evaluation();

-- 2. Denormalize enrollment_id (from student + school_year_category)
CREATE OR REPLACE FUNCTION grade_denormalize_enrollment()
RETURNS TRIGGER AS $$
DECLARE
    enroll_id UUID;
BEGIN
    SELECT id INTO enroll_id
    FROM enrollment
    WHERE student_profile_id = NEW.student_profile_id
      AND school_year_category_id = (
          SELECT school_year_category_id
          FROM school_year_category_term
          WHERE id = NEW.school_year_category_term_id
      )
      AND is_active = TRUE
    LIMIT 1;

    IF enroll_id IS NULL THEN
        RAISE EXCEPTION 'No active enrollment found for student in this school year category';
    END IF;

    NEW.enrollment_id = enroll_id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_grade_set_enrollment
    BEFORE INSERT ON grade
    FOR EACH ROW EXECUTE FUNCTION grade_denormalize_enrollment();

-- 3. Prevent FINALIZED grade modification
CREATE OR REPLACE FUNCTION grade_prevent_finalized_update()
RETURNS TRIGGER AS $$
BEGIN
    IF OLD.state = 'FINALIZED' AND
       (OLD.value != NEW.value OR OLD.state != NEW.state) THEN
        RAISE EXCEPTION 'Cannot modify finalized grade. Use amendment process via GradeAppeal.';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_grade_finalized_protect
    BEFORE UPDATE ON grade
    FOR EACH ROW
    WHEN (OLD.state = 'FINALIZED')
    EXECUTE FUNCTION grade_prevent_finalized_update();

-- 4. Auto-calculate evaluation completion count
CREATE OR REPLACE FUNCTION evaluation_update_completion()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE evaluation
    SET total_students_completed = (
        SELECT COUNT(*) FROM grade
        WHERE evaluation_id = NEW.evaluation_id
          AND state IN ('ENTERED', 'SUBMITTED', 'REVIEWED', 'FINALIZED')
    ),
    total_students_with_grade = (
        SELECT COUNT(*) FROM grade
        WHERE evaluation_id = NEW.evaluation_id
          AND state = 'FINALIZED'
    )
    WHERE id = NEW.evaluation_id;
    RETURN NULL;  -- AFTER trigger returns NULL
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_evaluation_update_completion
    AFTER INSERT OR UPDATE OF state ON grade
    FOR EACH ROW EXECUTE FUNCTION evaluation_update_completion();

-- 5. Cryptographic seal for finalized grades (optional enhanced security)
-- CREATE TABLE grade_seal (
--     grade_id UUID PRIMARY KEY REFERENCES grade(id),
--     seal_hash CHAR(64),  -- SHA256(evaluation_id + student_id + value + finalized_at)
--     sealed_at TIMESTAMPTZ DEFAULT NOW(),
--     sealed_by UUID
-- );
```

---

## 6. GradeType Table (grade_type)

```sql
CREATE TABLE grade_type (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Natural Key
    code VARCHAR(20) NOT NULL UNIQUE,  -- "TWENTY", "PERCENTAGE", "GPA_4", "LETTER"
    name VARCHAR(100) NOT NULL,

    -- Scale Definition
    min_value DECIMAL(6,2) NOT NULL,
    max_value DECIMAL(6,2) NOT NULL,
    passing_threshold DECIMAL(6,2) NOT NULL,

    -- Configuration
    rounding_method VARCHAR(20) NOT NULL CHECK (
        rounding_method IN ('STANDARD', 'UP', 'DOWN', 'TRUNCATE')
    ) DEFAULT 'STANDARD',
    display_format VARCHAR(20) NOT NULL CHECK (
        display_format IN ('NUMERIC', 'LETTER', 'BOTH')
    ),
    precision_digits INTEGER NOT NULL DEFAULT 2,

    -- Control
    is_active BOOLEAN NOT NULL DEFAULT TRUE,

    -- Audit Trail
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by UUID NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by UUID,
    archived_at TIMESTAMPTZ,
    archived_by UUID,

    -- Constraints
    CONSTRAINT grade_type_range_valid CHECK (min_value < max_value),
    CONSTRAINT grade_type_threshold_valid CHECK (
        passing_threshold >= min_value AND passing_threshold <= max_value
    )
);

CREATE INDEX idx_grade_type_code ON grade_type(code);
CREATE INDEX idx_grade_type_is_active ON grade_type(is_active) WHERE is_active = TRUE;

-- Grade Type Mapping for LETTER grades
CREATE TABLE grade_type_mapping (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    grade_type_id UUID NOT NULL REFERENCES grade_type(id) ON DELETE CASCADE,
    letter_grade VARCHAR(5) NOT NULL,  -- "A", "B+", "F"
    numeric_value DECIMAL(6,2) NOT NULL,  -- GPA equivalent
    min_percent INTEGER,  -- optional percentage range
    max_percent INTEGER,
    is_passing BOOLEAN NOT NULL DEFAULT TRUE,

    UNIQUE (grade_type_id, letter_grade),
    UNIQUE (grade_type_id, numeric_value)
);

CREATE INDEX idx_grade_type_mapping_type_id ON grade_type_mapping(grade_type_id);
CREATE INDEX idx_grade_type_mapping_letter ON grade_type_mapping(letter_grade);

-- Level links to GradeType
-- (Level table from TDR-002 would have default_grade_type_id FK)
```

---

## 7. Grade Appeal Table (grade_appeal)

```sql
CREATE TABLE grade_appeal (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- References
    grade_id UUID NOT NULL REFERENCES grade(id),  -- the grade being appealed
    student_profile_id UUID NOT NULL REFERENCES student_profile(id),
    enrollment_id UUID NOT NULL REFERENCES enrollment(id),

    -- Appeal Details
    appeal_date DATE NOT NULL DEFAULT CURRENT_DATE,
    appeal_reason TEXT NOT NULL,
    requested_action VARCHAR(50) NOT NULL CHECK (
        requested_action IN ('REVIEW', 'RECALCULATION', 'REGrading', 'OTHER')
    ),

    -- Status
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING' CHECK (
        status IN ('PENDING', 'UNDER_REVIEW', 'APPROVED', 'DENIED', 'WITHDRAWN')
    ),

    -- Review Process
    assigned_to UUID REFERENCES teacher_profile(id),  -- reviewing teacher
    reviewed_by UUID REFERENCES academic_inspector_profile(id),
    reviewed_at TIMESTAMPTZ,
    review_notes TEXT,

    -- Resolution
    resolution_type VARCHAR(50),  -- 'NO_CHANGE', 'GRADE_AMENDMENT', 'EXEMPTION_GRANTED'
    amendment_grade_id UUID REFERENCES grade(id),  -- the correction grade if approved

    -- Control
    is_active BOOLEAN NOT NULL DEFAULT TRUE,

    -- Audit
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by UUID NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by UUID,
    archived_at TIMESTAMPTZ,
    archived_by UUID
);

CREATE INDEX idx_grade_appeal_grade_id ON grade_appeal(grade_id);
CREATE INDEX idx_grade_appeal_student_id ON grade_appeal(student_profile_id);
CREATE INDEX idx_grade_appeal_status ON grade_appeal(status);
CREATE INDEX idx_grade_appeal_created_at ON grade_appeal(created_at);
```

---

## 8. Materialized Views for Performance

### 8.1 Student Report Card View

```sql
CREATE MATERIALIZED VIEW student_report_card_data AS
SELECT
    e.enrollment_id,
    e.student_profile_id,
    syc.code as school_year_category_code,
    syct.code as term_code,
    c.code as classroom_code,
    sub.code as subject_code,
    sub.name as subject_name,
    COUNT(DISTINCT eval.id) as total_evaluations,
    SUM(CASE WHEN g.value IS NOT NULL THEN eval.coefficient ELSE 0 END) as total_coefficient,
    SUM(CASE WHEN g.value IS NOT NULL THEN g.value * eval.coefficient ELSE 0 END) as weighted_sum,
    CASE
        WHEN SUM(CASE WHEN g.value IS NOT NULL THEN eval.coefficient ELSE 0 END) > 0
        THEN ROUND(
            SUM(CASE WHEN g.value IS NOT NULL THEN g.value * eval.coefficient ELSE 0 END) /
            SUM(CASE WHEN g.value IS NOT NULL THEN eval.coefficient ELSE 0 END),
            2
        )
        ELSE NULL
    END as subject_average,
    MAX(eval.evaluation_date) as last_evaluation_date
FROM enrollment e
JOIN school_year_category syc ON e.school_year_category_id = syc.id
JOIN school_year sy ON syc.school_year_id = sy.id
JOIN classroom c ON e.classroom_id = c.id
JOIN classroom_subject cs ON c.id = cs.classroom_id
JOIN subject sub ON cs.subject_id = sub.id
JOIN evaluation eval ON cs.id = eval.classroom_subject_id
JOIN school_year_category_term syct ON eval.school_year_category_term_id = syct.id
LEFT JOIN grade g ON eval.id = g.evaluation_id
    AND g.student_profile_id = e.student_profile_id
    AND g.state = 'FINALIZED'
WHERE e.state = 'ACTIVE'
    AND eval.state = 'FINALIZED'
    AND syct.is_reporting_term = TRUE
GROUP BY e.enrollment_id, e.student_profile_id, syc.code, syct.code, c.code, sub.code, sub.name;

-- Indexes on materialized view
CREATE UNIQUE INDEX idx_student_report_card_data_unique ON student_report_card_data(
    enrollment_id, school_year_category_code, term_code, subject_code
);
CREATE INDEX idx_student_report_card_student ON student_report_card_data(student_profile_id, school_year_category_code);

-- Refresh strategy: incremental refresh at term finalization
-- REFRESH MATERIALIZED VIEW CONCURRENTLY student_report_card_data;
```

### 8.2 Class Ranking View

```sql
CREATE MATERIALIZED VIEW class_ranking_data AS
SELECT
    e.enrollment_id,
    e.student_profile_id,
    c.id as classroom_id,
    c.code as classroom_code,
    syc.code as school_year_code,
    AVG(g.value) FILTER (WHERE g.state = 'FINALIZED') as class_average,
    RANK() OVER (
        PARTITION BY c.id, syc.id
        ORDER BY AVG(g.value) FILTER (WHERE g.state = 'FINALIZED') DESC NULLS LAST
    ) as class_rank,
    COUNT(*) FILTER (WHERE g.value IS NOT NULL) as grades_count
FROM enrollment e
JOIN classroom c ON e.classroom_id = c.id
JOIN school_year_category syc ON c.school_year_category_id = syc.id
JOIN grade g ON e.enrollment_id = g.enrollment_id
WHERE e.state = 'ACTIVE'
GROUP BY e.enrollment_id, e.student_profile_id, c.id, c.code, syc.id, syc.code;

CREATE INDEX idx_class_ranking_classroom ON class_ranking_data(classroom_id, school_year_code);
CREATE INDEX idx_class_ranking_student ON class_ranking_data(student_profile_id);
```

---

## 9. Sync Strategy for Evaluations & Grades

### 9.1 Offline Evaluation Constraints

**Evaluations can be CREATED only by teachers with connectivity**:
- Requires server-side check: SYCT state = OPEN
- Requires teacher assignment validation
- Not allowed offline (new evaluation)

**Evaluation PUBLISH can be queued offline**:
- Teacher marks evaluation as ready offline
- Sync pushes PUBLISH event when online
- Server validates before publishing

### 9.2 Grade Sync Patterns

```
Scenario 1: Entry (online)
Teacher → ENTER grade → SUBMIT → Server validates → ACCEPTED

Scenario 2: Entry (offline)
Teacher → ENTER grade → Save locally (state=ENTERED)
    On sync:
        Validate evaluation exists, not FINALIZED, deadline not passed
        Grade uploaded as ENTERED
        Server may require SUBMIT → separate step

Scenario 3: Conflict (two teachers edit same grade)
    Last-write-wins with conflict flag
    Both versions preserved in audit
    Admin notified to resolve

Scenario 4: Finalized grade amendment
    Original grade protected
    New amendment grade created with is_amendment = TRUE
    Both visible in audit trail
```

### 9.3 Sync Order Constraints

Grades sync in this order:
1. Evaluation metadata (must exist before grades)
2. Evaluation state transitions
3. Grade records (INSERT/UPDATE)

**Offline devices must maintain local reference to pending sync queue**:
```json
{
  "pending_grade_sync": [
    {
      "grade_id": "local-uuid",
      "evaluation_id": "server-uuid",
      "value": 15.50,
      "action": "INSERT",
      "timestamp": "2025-03-29T10:30:00Z"
    }
  ]
}
```

### 9.4 Merge Conflict Resolution Table

```sql
-- Track sync conflicts for manual resolution
CREATE TABLE grade_sync_conflict (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    grade_id UUID NOT NULL,  -- local grade that conflicted
    evaluation_id UUID NOT NULL,
    student_profile_id UUID NOT NULL,
    local_value DECIMAL(5,2),
    server_value DECIMAL(5,2),
    conflict_type VARCHAR(50),  -- "CONCURRENT_EDIT", "VALIDATION_FAILED", "DEADLINE_EXCEEDED"
    resolution_status VARCHAR(20) DEFAULT 'PENDING',  -- PENDING, RESOLVED, IGNORED
    resolved_by UUID,
    resolved_at TIMESTAMPTZ,
    resolution_notes TEXT
);
```

---

## 10. Data Retention & Archival

### 10.1 Archive Policy

| Table | Retention | Archive Age | Archive Method |
|-------|-----------|-------------|----------------|
| evaluation_type | Permanent | N/A | Live table |
| evaluation_category | Permanent | N/A | Live table |
| evaluation | 10 years online | 5 years | Move to archive partition |
| grade | Permanent | 3 years old | Move to archive partition |
| grade_type | Permanent | N/A | Live table |
| grade_appeal | 10 years | 7 years | Move to archive schema |

### 10.2 Partition Archival Jobs

```sql
-- Move evaluation_y2018 to archive after 5 years
CREATE TABLE archive.evaluation_y2018 PARTITION OF evaluation FOR VALUES IN ('2018');

-- Move grade_y2015 to archive after 3 years
CREATE TABLE archive.grade_y2015 PARTITION OF grade FOR VALUES IN ('2015');

-- Archive cleanup: drop partitions older than 10 years
```

---

## 11. Integrity Constraints Summary

| Constraint | Enforcement Mechanism |
|------------|----------------------|
| One grade per (evaluation, student) | UNIQUE constraint |
| Grade value ≤ evaluation.max_score | CHECK constraint + trigger |
| Finalized grade immutability | Trigger + is_active flag |
| Evaluation state transitions | Trigger with state machine validation |
| Evaluation completion tracking | Trigger on grade insert/update |
| Grade denormalization | BEFORE INSERT triggers |
| Backdate prevention | Trigger on evaluation_date |

---

## 12. Performance Optimizations for Large Scale

1. **Index Design**:
   - Composite `(evaluation_id, state)` for teacher grade entry
   - Composite `(student_profile_id, state)` for student transcript lookup
   - Partial indexes for active records only

2. **Partitioning**:
   - Grade table: hash partition by student_id within each school_year
   - Evaluation table: range partition by school_year_code + school_year
   - Reports query recent years; archive pruning effective

3. **Materialized Views**:
   - Student report card data refreshed at term finalization
   - Class ranking refreshed daily for active terms

4. **Connection Pooling**:
   - Grade entry is write-heavy; use separate pool for teacher connections
   - Reporting queries use read replicas

---

**Next Step**: TDR-006 — Domain Rules: Timetables (TimeSlot, Timetable, TimetableEntry)
