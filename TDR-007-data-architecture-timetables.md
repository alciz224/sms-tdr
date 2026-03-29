# TDR-007 — Data Architecture: Timetables Tables

## 1. Purpose and Scope

This document defines the physical data architecture for the timetabling domain (Étape 5). It specifies:

- Complete SQL DDL for TimeSlot, Timetable, TimetableEntry, and TimetableException
- Indexing strategies optimized for timetable queries
- Partitioning approach for timetable version history
- Conflict detection implementation
- Materialized views for common schedule queries

**Considerations**: Timetables are moderately sized but frequently queried. They must support real-time conflict checking during modification.

---

## 2. TimeSlot Table (time_slot)

```sql
CREATE TABLE time_slot (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Natural Key (globally unique across all schools)
    code VARCHAR(20) NOT NULL UNIQUE,  -- "MON_1", "TUE_3", "FRI_7"
    name VARCHAR(100) NOT NULL,

    -- Time Configuration
    day_of_week INTEGER NOT NULL CHECK (day_of_week BETWEEN 1 AND 7),  -- 1=Monday
    period_number INTEGER NOT NULL CHECK (period_number >= 1),

    start_time TIME NOT NULL,
    end_time TIME NOT NULL,
    duration_minutes INTEGER GENERATED ALWAYS AS (
        EXTRACT(EPOCH FROM (end_time - start_time)) / 60
    ) STORED,

    -- Scheduling Pattern
    rotation_pattern VARCHAR(20) NOT NULL DEFAULT 'DAILY' CHECK (
        rotation_pattern IN ('DAILY', 'A_WEEK', 'B_WEEK', 'MON_WED_FRI', 'TUE_THU', 'WEEKLY')
    ),

    -- Classification
    slot_type VARCHAR(20) NOT NULL DEFAULT 'REGULAR' CHECK (
        slot_type IN ('REGULAR', 'RECESS', 'LUNCH', 'EXAM', 'ASSEMBLY', 'SPORT')
    ),
    is_instructional BOOLEAN NOT NULL DEFAULT TRUE,  -- counts toward weekly hours

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
    CONSTRAINT time_slot_time_valid CHECK (end_time > start_time),
    CONSTRAINT time_slot_code_format CHECK (code ~ '^[A-Z]{3}_[1-9][0-9]*$'),

    -- Composite uniqueness: only one slot per (day, period)
    UNIQUE (day_of_week, period_number)
);

-- Indexes
CREATE INDEX idx_time_slot_code ON time_slot(code);
CREATE INDEX idx_time_slot_day_period ON time_slot(day_of_week, period_number);
CREATE INDEX idx_time_slot_rotation ON time_slot(rotation_pattern);
CREATE INDEX idx_time_slot_is_active ON time_slot(is_active) WHERE is_active = TRUE;

-- Typical data: ~50 rows (5 days × 8-10 periods + breaks)
-- No partitioning needed
```

---

## 3. Timetable Table (timetable)

```sql
CREATE TABLE timetable (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Foreign Key
    school_year_category_id UUID NOT NULL REFERENCES school_year_category(id) ON DELETE CASCADE,

    -- Natural Key
    code VARCHAR(50) NOT NULL,
    name VARCHAR(100) NOT NULL,

    -- Versioning
    version INTEGER NOT NULL DEFAULT 1,
    effective_from_date DATE NOT NULL,
    effective_to_date DATE,  -- NULL = current active

    -- Configuration (snapshots for reporting)
    total_weekly_periods INTEGER NOT NULL,  -- e.g., 30 (6 periods × 5 days)
    total_classroom_subjects INTEGER NOT NULL,  -- count of subjects scheduled

    -- Generation
    generation_method VARCHAR(20) NOT NULL CHECK (
        generation_method IN ('AUTOMATIC', 'MANUAL', 'HYBRID')
    ) DEFAULT 'AUTOMATIC',
    generation_parameters JSONB,  -- algorithm settings, weights, priorities
    generation_log TEXT,  -- any conflicts, backtracking info

    -- State
    status VARCHAR(20) NOT NULL CHECK (
        status IN ('DRAFT', 'ACTIVE', 'MODIFIED', 'ARCHIVED')
    ) DEFAULT 'DRAFT',

    -- Control: Only one ACTIVE timetable per school_year_category
    is_active BOOLEAN NOT NULL DEFAULT FALSE,

    -- Approval
    approved_by UUID REFERENCES teacher_profile(id),
    approved_at TIMESTAMPTZ,
    rejection_reason TEXT,

    -- Audit
    published_at TIMESTAMPTZ,
    published_by UUID REFERENCES teacher_profile(id),

    -- Audit Trail
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by UUID NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by UUID,
    archived_at TIMESTAMPTZ,
    archived_by UUID,

    -- Constraints
    CONSTRAINT timetable_code_unique UNIQUE (school_year_category_id, code),
    CONSTRAINT timetable_version_range CHECK (
        effective_from_date < COALESCE(effective_to_date, '9999-12-31')
    ),

    -- Only one active timetable per school_year_category
    CONSTRAINT timetable_one_active UNIQUE (school_year_category_id) WHERE is_active = TRUE
);

CREATE INDEX idx_timetable_sycat_id ON timetable(school_year_category_id);
CREATE INDEX idx_timetable_code ON timetable(code);
CREATE INDEX idx_timetable_status ON timetable(status);
CREATE INDEX idx_timetable_active ON timetable(is_active) WHERE is_active = TRUE;
CREATE INDEX idx_timetable_date_range ON timetable(effective_from_date, effective_to_date);
CREATE INDEX idx_timetable_sycat_active ON timetable(school_year_category_id, is_active)
    WHERE is_active = TRUE OR status IN ('DRAFT', 'MODIFIED');

-- Partitioning: Rarely needed. Timetable table is small (<100 rows per system)
-- Could partition by school_year_category_id if multi-tenant with many schools
```

### 3.1 Triggers

```sql
-- Validate weekly hours when timetable is generated
CREATE OR REPLACE FUNCTION timetable_validate_weekly_hours()
RETURNS TRIGGER AS $$
DECLARE
    subject_hours RECORD;
    missing_subjects TEXT[];
BEGIN
    -- Check that all ClassroomSubjects have required hours scheduled
    FOR subject_hours IN
        SELECT
            cs.id,
            cs.classroom_id,
            sub.weekly_hours as required_hours,
            COUNT(te.id) as scheduled_periods
        FROM classroom_subject cs
        JOIN subject sub ON cs.subject_id = sub.id
        LEFT JOIN timetable_entry te ON te.timetable_id = NEW.id
            AND te.classroom_subject_id = cs.id
            AND te.state = 'ACTIVE'
        WHERE cs.classroom_id IN (
            SELECT classroom_id
            FROM classroom
            WHERE school_year_category_id = NEW.school_year_category_id
        )
        GROUP BY cs.id, cs.classroom_id, sub.weekly_hours
    LOOP
        IF subject_hours.scheduled_periods != subject_hours.required_hours THEN
            missing_subjects := array_append(missing_subjects,
                format('ClassroomSubject %s: %s scheduled, %s required',
                    subject_hours.classroom_id,
                    subject_hours.scheduled_periods,
                    subject_hours.required_hours));
        END IF;
    END LOOP;

    IF array_length(missing_subjects, 1) IS NOT NULL THEN
        RAISE EXCEPTION 'Timetable validation failed: %',
            array_to_string(missing_subjects, '; ');
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Only validate on status change to ACTIVE or DRAFT to ACTIVE
CREATE TRIGGER trg_timetable_validate_hours
    BEFORE UPDATE OF status ON timetable
    FOR EACH ROW
    WHEN (NEW.status IN ('ACTIVE') AND OLD.status != NEW.status)
    EXECUTE FUNCTION timetable_validate_weekly_hours();
```

---

## 4. TimetableEntry Table (timetable_entry)

### 4.1 Physical Design

```sql
CREATE TABLE timetable_entry (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Foreign Keys
    timetable_id UUID NOT NULL REFERENCES timetable(id) ON DELETE CASCADE,

    time_slot_id UUID NOT NULL REFERENCES time_slot(id) ON DELETE RESTRICT,
    classroom_subject_id UUID NOT NULL REFERENCES classroom_subject(id) ON DELETE CASCADE,
    teacher_id UUID NOT NULL REFERENCES teacher_profile(id) ON DELETE RESTRICT,

    -- Optional room assignment
    room_code VARCHAR(20),  -- can reference rooms table if implemented

    -- Scheduling Pattern
    schedule_pattern VARCHAR(30) NOT NULL DEFAULT 'EVERY_WEEK' CHECK (
        schedule_pattern IN ('EVERY_WEEK', 'A_WEEK_ONLY', 'B_WEEK_ONLY',
                             'MON_WED_FRI', 'TUE_THU', 'MONDAY_ONLY', 'FRIDAY_ONLY')
    ),

    -- Effective Date Range (defaults from timetable)
    effective_from_date DATE NOT NULL,
    effective_to_date DATE,  -- NULL = until timetable ends

    -- Classification
    entry_type VARCHAR(20) NOT NULL DEFAULT 'REGULAR' CHECK (
        entry_type IN ('REGULAR', 'DOUBLE_PERIOD', 'LAB_SESSION', 'EXAM', 'STUDY_HALL')
    ),

    -- State
    state VARCHAR(20) NOT NULL DEFAULT 'ACTIVE' CHECK (
        state IN ('ACTIVE', 'CANCELLED', 'MOVED')
    ),

    -- Exception tracking
    is_exception BOOLEAN NOT NULL DEFAULT FALSE,
    exception_reason TEXT,
    original_entry_id UUID REFERENCES timetable_entry(id) ON DELETE SET NULL,

    -- Audit
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by UUID NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by UUID,
    archived_at TIMESTAMPTZ,
    archived_by UUID,

    -- Complex Uniqueness Constraint: No double bookings
    -- (timetable_id, time_slot_id, teacher_id) must be unique for ACTIVE entries
    -- (timetable_id, time_slot_id, classroom_subject_id) must be unique for ACTIVE entries
    -- Implemented via partial unique indexes
);

-- Composite unique indexes for conflict prevention
CREATE UNIQUE INDEX idx_timetable_entry_teacher_conflict
    ON timetable_entry(timetable_id, time_slot_id, teacher_id)
    WHERE state = 'ACTIVE' AND is_exception = FALSE;

CREATE UNIQUE INDEX idx_timetable_entry_subject_conflict
    ON timetable_entry(timetable_id, time_slot_id, classroom_subject_id)
    WHERE state = 'ACTIVE' AND is_exception = FALSE;

-- If room tracking enabled
CREATE UNIQUE INDEX idx_timetable_entry_room_conflict
    ON timetable_entry(timetable_id, time_slot_id, room_code)
    WHERE state = 'ACTIVE' AND is_exception = FALSE AND room_code IS NOT NULL;

-- Indexes for common queries
CREATE INDEX idx_timetable_entry_timetable_id ON timetable_entry(timetable_id);
CREATE INDEX idx_timetable_entry_teacher_id ON timetable_entry(teacher_id);
CREATE INDEX idx_timetable_entry_classroom_subject ON timetable_entry(classroom_subject_id);
CREATE INDEX idx_timetable_entry_time_slot ON timetable_entry(time_slot_id);

-- Composite indexes for schedule lookups
CREATE INDEX idx_timetable_entry_teacher_schedule
    ON timetable_entry(teacher_id, timetable_id, effective_from_date, effective_to_date);

CREATE INDEX idx_timetable_entry_classroom_schedule
    ON timetable_entry(classroom_subject_id, timetable_id);

CREATE INDEX idx_timetable_entry_state_active
    ON timetable_entry(timetable_id, state)
    WHERE state = 'ACTIVE';

-- Date range index for conflict detection queries
CREATE INDEX idx_timetable_entry_date_range
    ON timetable_entry(timetable_id, time_slot_id, effective_from_date, effective_to_date);

-- Partitioning: This table can grow large
-- ~(50 time_slots × 200 classrooms × 4 years) ≈ 40,000 rows per school
-- Partition by timetable_id (list) or school_year derived from timetable
-- For single large school: no partitioning needed (40K rows is small)
-- For SaaS multi-tenant: partition by tenant_id with local indexes
```

### 4.2 Triggers for Conflict Prevention

```sql
-- 1. Check teacher scheduling limit
CREATE OR REPLACE FUNCTION timetable_entry_check_teacher_hours()
RETURNS TRIGGER AS $$
DECLARE
    teacher_total_hours INTEGER;
    teacher_max_hours INTEGER;
BEGIN
    -- Get teacher's max weekly hours
    SELECT weekly_teaching_hours INTO teacher_max_hours
    FROM teacher_profile
    WHERE id = NEW.teacher_id;

    -- Calculate current scheduled hours for this teacher in this timetable
    -- Count each week's periods
    SELECT COUNT(*) * (
        SELECT duration_minutes FROM time_slot WHERE id = NEW.time_slot_id
    ) / 60 INTO teacher_total_hours
    FROM timetable_entry te
    JOIN time_slot ts ON te.time_slot_id = ts.id
    WHERE te.timetable_id = NEW.timetable_id
        AND te.teacher_id = NEW.teacher_id
        AND te.state = 'ACTIVE'
        AND te.id != COALESCE(OLD.id, NEW.id);  -- exclude self for UPDATE

    IF teacher_total_hours > teacher_max_hours THEN
        RAISE EXCEPTION 'Teacher exceeds maximum weekly hours (scheduled: %, max: %)',
            teacher_total_hours, teacher_max_hours;
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE CONSTRAINT TRIGGER trg_timetable_entry_teacher_hours
    AFTER INSERT OR UPDATE ON timetable_entry
    FOR EACH ROW
    WHEN (NEW.state = 'ACTIVE')
    EXECUTE FUNCTION timetable_entry_check_teacher_hours();

-- 2. Check classroom subject hours consistency
CREATE OR REPLACE FUNCTION timetable_entry_check_subject_hours()
RETURNS TRIGGER AS $$
DECLARE
    required_hours INTEGER;
    scheduled_hours INTEGER;
BEGIN
    -- Get required weekly hours from SubjectChoice
    SELECT weekly_hours INTO required_hours
    FROM subject_choice sc
    JOIN classroom_subject cs ON (
        cs.subject_id = sc.subject_id AND
        cs.required_track_id = sc.track_id
    )
    WHERE cs.id = NEW.classroom_subject_id;

    -- Count scheduled hours
    SELECT COUNT(*) INTO scheduled_hours
    FROM timetable_entry te
    WHERE te.classroom_subject_id = NEW.classroom_subject_id
        AND te.timetable_id = NEW.timetable_id
        AND te.state = 'ACTIVE'
        AND te.id != COALESCE(OLD.id, NEW.id);

    IF scheduled_hours > required_hours THEN
        RAISE EXCEPTION 'ClassroomSubject exceeds required weekly hours (scheduled: %, required: %)',
            scheduled_hours, required_hours;
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE CONSTRAINT TRIGGER trg_timetable_entry_subject_hours
    AFTER INSERT OR UPDATE ON timetable_entry
    FOR EACH ROW
    WHEN (NEW.state = 'ACTIVE')
    EXECUTE FUNCTION timetable_entry_check_subject_hours();
```

---

## 5. TimetableException Table (timetable_exception)

```sql
CREATE TABLE timetable_exception (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Foreign Keys
    timetable_id UUID NOT NULL REFERENCES timetable(id) ON DELETE CASCADE,
    affected_entry_id UUID NOT NULL REFERENCES timetable_entry(id) ON DELETE CASCADE,

    -- Exception Type
    exception_type VARCHAR(20) NOT NULL CHECK (
        exception_type IN ('CANCELLATION', 'SUBSTITUTION', 'RESCHEDULE', 'SPECIAL_EVENT')
    ),

    -- Date Range
    exception_date DATE NOT NULL,  -- single date for most exceptions
    -- For date ranges:
    exception_start_date DATE,
    exception_end_date DATE,

    -- Changes (depends on exception_type)
    new_time_slot_id UUID REFERENCES time_slot(id),
    new_teacher_id UUID REFERENCES teacher_profile(id),
    new_room_code VARCHAR(20),

    -- Details
    reason TEXT NOT NULL,
    approved_by UUID REFERENCES teacher_profile(id),
    approved_at TIMESTAMPTZ,

    -- Notification tracking
    notified_teacher BOOLEAN NOT NULL DEFAULT FALSE,
    notified_students BOOLEAN NOT NULL DEFAULT FALSE,
    notified_parents BOOLEAN NOT NULL DEFAULT FALSE,
    notification_sent_at TIMESTAMPTZ,

    -- Audit
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by UUID NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by UUID,
    archived_at TIMESTAMPTZ,
    archived_by UUID,

    -- Constraints
    CONSTRAINT timetable_exception_dates CHECK (
        (exception_end_date IS NULL AND exception_start_date IS NULL) OR
        (exception_end_date IS NOT NULL AND exception_start_date IS NOT NULL AND
         exception_end_date >= exception_start_date)
    ),
    CONSTRAINT timetable_exception_range_matches_timetable CHECK (
        exception_date >= (
            SELECT effective_from_date FROM timetable WHERE id = timetable_id
        ) AND
        exception_date <= COALESCE(
            (SELECT effective_to_date FROM timetable WHERE id = timetable_id),
            '9999-12-31'::DATE
        )
    ),

    -- Reschedule requires new time slot
    CONSTRAINT timetable_exception_reschedule_requires_slot CHECK (
        exception_type != 'RESCHEDULE' OR new_time_slot_id IS NOT NULL
    )
);

-- Indexes
CREATE INDEX idx_timetable_exception_timetable ON timetable_exception(timetable_id);
CREATE INDEX idx_timetable_exception_entry ON timetable_exception(affected_entry_id);
CREATE INDEX idx_timetable_exception_date ON timetable_exception(timetable_id, exception_date);
CREATE INDEX idx_timetable_exception_date_range ON timetable_exception(
    timetable_id,
    exception_date,
    exception_start_date,
    exception_end_date
);
CREATE INDEX idx_timetable_exception_notified ON timetable_exception(
    notified_teacher, notified_students, notified_parents
) WHERE notified_teacher = FALSE OR notified_students = FALSE OR notified_parents = FALSE;

-- Partitioning: ~365 × 30 classrooms = ~11K rows per year per school
-- For single school: no partitioning needed
-- For SaaS: partition by timetable_id or by year
```

---

## 6. Materialized Views for Schedule Queries

### 6.1 Complete Schedule View

```sql
CREATE MATERIALIZED VIEW full_schedule_view AS
SELECT
    t.id as timetable_id,
    t.code as timetable_code,
    t.school_year_category_id,
    ts.id as time_slot_id,
    ts.code as time_slot_code,
    ts.day_of_week,
    ts.period_number,
    ts.start_time,
    ts.end_time,
    te.id as timetable_entry_id,
    te.state as entry_state,
    cs.id as classroom_subject_id,
    c.id as classroom_id,
    c.code as classroom_code,
    l.code as level_code,
    tr.code as track_code,
    sub.id as subject_id,
    sub.code as subject_code,
    sub.name as subject_name,
    tp.id as teacher_id,
    tp.user_id as teacher_user_id,
    u.full_name as teacher_name,
    te.room_code
FROM timetable t
JOIN timetable_entry te ON t.id = te.timetable_id
JOIN time_slot ts ON te.time_slot_id = ts.id
JOIN classroom_subject cs ON te.classroom_subject_id = cs.id
JOIN classroom c ON cs.classroom_id = c.id
JOIN level l ON c.level_id = l.id
LEFT JOIN track tr ON c.track_id = tr.id
JOIN subject sub ON cs.subject_id = sub.id
JOIN teacher_profile tp ON te.teacher_id = tp.id
JOIN customuser u ON tp.user_id = u.id
WHERE t.is_active = TRUE
    AND te.state = 'ACTIVE'
    AND te.is_exception = FALSE;

-- Indexes for the materialized view
CREATE INDEX idx_full_schedule_day_period ON full_schedule_view(day_of_week, period_number);
CREATE INDEX idx_full_schedule_teacher ON full_schedule_view(teacher_id, day_of_week, period_number);
CREATE INDEX idx_full_schedule_classroom ON full_schedule_view(classroom_id, day_of_week, period_number);
CREATE INDEX idx_full_schedule_timetable ON full_schedule_view(timetable_id);

-- Refresh on timetable activation
```

### 6.2 Student-Specific Schedule View

```sql
CREATE MATERIALIZED VIEW student_schedule_view AS
SELECT
    e.student_profile_id,
    sp.user_id as student_user_id,
    u.full_name as student_name,
    e.enrollment_number,
    c.code as classroom_code,
    syc.code as school_year_category_code,
    t.code as timetable_code,
    ts.day_of_week,
    ts.period_number,
    ts.start_time,
    ts.end_time,
    te.time_slot_id,
    sub.code as subject_code,
    sub.name as subject_name,
    tp.full_name as teacher_name,
    te.room_code
FROM enrollment e
JOIN student_profile sp ON e.student_profile_id = sp.id
JOIN customuser u ON sp.user_id = u.id
JOIN classroom c ON e.classroom_id = c.id
JOIN school_year_category syc ON e.school_year_category_id = syc.id
JOIN timetable t ON syc.id = t.school_year_category_id AND t.is_active = TRUE
JOIN timetable_entry te ON t.id = te.timetable_id
JOIN time_slot ts ON te.time_slot_id = ts.id
JOIN classroom_subject cs ON te.classroom_subject_id = cs.id
    AND cs.classroom_id = c.id
JOIN subject sub ON cs.subject_id = sub.id
LEFT JOIN teacher_profile tp ON te.teacher_id = tp.id
WHERE e.state = 'ACTIVE'
    AND te.state = 'ACTIVE';

CREATE INDEX idx_student_schedule_student_day ON student_schedule_view(
    student_profile_id, day_of_week, period_number
);
CREATE INDEX idx_student_schedule_timetable ON student_schedule_view(
    student_profile_id, timetable_code
);
```

---

## 7. Conflict Detection Queries

### 7.1 Teacher Conflict Query

```sql
-- Find all teacher conflicts in a timetable
SELECT
    t.id as timetable_id,
    te1.teacher_id,
    tp.full_name as teacher_name,
    ts1.day_of_week,
    ts1.period_number,
    COUNT(*) as conflict_count
FROM timetable_entry te1
JOIN timetable_entry te2 ON
    te1.timetable_id = te2.timetable_id AND
    te1.teacher_id = te2.teacher_id AND
    te1.time_slot_id = te2.time_slot_id AND
    te1.id < te2.id AND  -- avoid duplicate pairs
    te1.state = 'ACTIVE' AND
    te2.state = 'ACTIVE'
JOIN time_slot ts1 ON te1.time_slot_id = ts1.id
JOIN teacher_profile tp ON te1.teacher_id = tp.id
WHERE te1.timetable_id = $1
GROUP BY t.id, te1.teacher_id, tp.full_name, ts1.day_of_week, ts1.period_number
HAVING COUNT(*) > 0
ORDER BY teacher_name, day_of_week, period_number;
```

### 7.2 Room Conflict Query

```sql
-- Find all room double-bookings
SELECT
    t.id,
    te1.room_code,
    ts.day_of_week,
    ts.period_number,
    COUNT(*) as double_bookings
FROM timetable_entry te1
JOIN timetable_entry te2 ON
    te1.timetable_id = te2.timetable_id AND
    te1.room_code = te2.room_code AND
    te1.time_slot_id = te2.time_slot_id AND
    te1.id < te2.id
JOIN time_slot ts ON te1.time_slot_id = ts.id
JOIN timetable t ON te1.timetable_id = t.id
WHERE te1.room_code IS NOT NULL
    AND te1.state = 'ACTIVE'
    AND te2.state = 'ACTIVE'
    AND te1.timetable_id = $1
GROUP BY t.id, te1.room_code, ts.day_of_week, ts.period_number
HAVING COUNT(*) > 0;
```

---

## 8. Timetable Generation Stored Procedure (Outline)

```sql
-- Auto-generation algorithm using Greedy + Backtracking
CREATE OR REPLACE FUNCTION generate_timetable(
    p_school_year_category_id UUID,
    p_generation_method VARCHAR(20) DEFAULT 'AUTOMATIC'
) RETURNS UUID AS $$
DECLARE
    v_timetable_id UUID;
    v_classroom_ids UUID[];
    v_time_slot_ids UUID[];
    v_subjects RECORD;
    v_assignment_count INTEGER;
BEGIN
    -- Create timetable
    INSERT INTO timetable (school_year_category_id, code, name, status, generation_method)
    VALUES (
        p_school_year_category_id,
        (SELECT code || '-TT01' FROM school_year_category WHERE id = p_school_year_category_id),
        'Auto-generated Timetable',
        'DRAFT',
        p_generation_method
    ) RETURNING id INTO v_timetable_id;

    -- Get all classrooms requiring schedule
    SELECT array_agg(id) INTO v_classroom_ids
    FROM classroom
    WHERE school_year_category_id = p_school_year_category_id
        AND state = 'ACTIVE';

    -- Get all time slots for the school
    SELECT array_agg(id) INTO v_time_slot_ids
    FROM time_slot
    WHERE is_active = TRUE AND slot_type IN ('REGULAR', 'LAB_SESSION')
    ORDER BY day_of_week, period_number;

    -- For each classroom_subject, assign time slots
    FOR v_subjects IN
        SELECT
            cs.id,
            cs.classroom_id,
            sub.weekly_hours,
            cs.teacher_id,
            sub.has_practical
        FROM classroom_subject cs
        JOIN subject sub ON cs.subject_id = sub.id
        WHERE cs.classroom_id = ANY(v_classroom_ids)
          AND cs.is_active = TRUE
        ORDER BY
            CASE WHEN sub.has_practical THEN 0 ELSE 1 END,  -- labs first
            sub.weekly_hours DESC  -- subjects with more hours first
    LOOP
        -- Assignment algorithm: find available slots for this (classroom_subject, teacher)
        -- Implementation uses recursive CTE or application-level solver
        -- For DDL doc: outline approach only
        NULL;
    END LOOP;

    RETURN v_timetable_id;
END;
$$ LANGUAGE plpgsql;
```

---

## 9. Data Retention and Archival

### 9.1 Timetable History

Old timetables are retained for audit and dispute resolution:

- **Active timetable**: kept in main table
- **Previous timetable versions**: moved to `archive_timetable` table after 2 years
- **Timetable entries**: archived with timetable
- **Exceptions**: retained for current + 2 previous years (legal requirement)

### 9.2 Archival Procedure

```sql
-- Archive old timetables (run annually)
CREATE TABLE archive.timetable_y2023 PARTITION OF timetable
    FOR VALUES IN ('2023-2024')  -- sycat code pattern
    TABLESPACE archive_tsd;

CREATE TABLE archive.timetable_entry_y2023 PARTITION OF timetable_entry
    FOR VALUES IN ('2023-2024');

-- After archival, detach from main and attach to archive
```

---

## 10. Sync Strategy for Timetables

### 10.1 Distribution Pattern

```
Timeline:
T-2 weeks: Timetable DRAFT created (admin-only)
T-1 week: Timetable ACTIVE → PUSH to all affected devices
          Teachers receive teacher-specific schedule
          Students receive class schedules
          Devices cache locally
T (school start): Timetable enforced for attendance
```

### 10.2 Offline Considerations

**Device Caching Requirements**:
- Full timetable + all entries for active timetables
- Current exceptions
- TimeSlot definitions

**Offline Modifications**:
- Not allowed (timetable changes require admin and conflict checking)
- Queue changes for sync (rare)

### 10.3 Conflict Resolution

Timetable modifications use optimistic locking:
```sql
UPDATE timetable
SET status = 'MODIFIED', version = version + 1, ...
WHERE id = $1 AND version = $2;  -- check version

-- If rows affected = 0, someone else modified → conflict
```

---

## 11. Performance Targets

| Operation | Target |
|-----------|--------|
| Load teacher schedule (full week) | < 150ms |
| Check room availability (slot) | < 50ms |
| Conflict detection (full timetable) | < 5s |
| Timetable generation (30 classrooms) | < 60s |
| Get student daily schedule | < 100ms |

---

## 12. Complete Data Model

```
time_slot (N) <─ timetable_entry (N) >─ classroom_subject (1)
                    │                           │
                    │                           ├─ classroom (1)
                    │                           ├─ subject (1)
                    │                           └─ teacher (1)
                    │
                    └─> timetable (1) ── school_year_category (1)

timetable_exception (N) ── timetable_entry (N)
```

---

**Next Step**: TDR-008 — Domain Rules: User & Role System (CustomUser, TeacherProfile, StudentProfile, ParentProfile, AdminProfile)
