# TDR-011 — Selectors & Queries

## 1. Purpose and Scope

This document defines the **data access layer** architecture, specifying:

- **Selector pattern** for read operations (query objects)
- **Repository interfaces** for write operations (aggregate persistence)
- **Common query patterns** for each domain
- **Performance optimization techniques** (indexes, caching, materialized views)
- **Pagination, filtering, sorting** conventions
- **N+1 query prevention** strategies
- **Batch operations** for bulk processing

**Key Principle**: The data access layer is the contract between services and the database. It must be optimized for both online and offline patterns, with clear separation between reads (selectors) and writes (repositories).

---

## 2. Architecture Overview

### 2.1 Layer Structure

```
┌─────────────────────────────────────────────────────────────┐
│                    SERVICE LAYER                             │
│  (EnrollmentService, GradeService, etc.)                    │
└───────────────────────────┬─────────────────────────────────┘
                            │ uses
┌───────────────────────────▼─────────────────────────────────┐
│                  DATA ACCESS LAYER                           │
├─────────────────────────────────────────────────────────────┤
│  Selectors (Read)              │  Repositories (Write)      │
│  • StudentSelector            │  • EnrollmentRepository    │
│  • GradeSelector              │  • GradeRepository         │
│  • ClassroomSelector          │  • TimetableRepository    │
│  • ReportSelector             │  • UserRepository          │
│  • SearchSelector             │  • SyncRepository          │
└───────────────────────────┬─────────────────────────────────┘
                            │ uses
┌───────────────────────────▼─────────────────────────────────┐
│                   DATABASE / CACHE                          │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Design Patterns

**Selector Pattern** (Query Object):
- Stateless, composable query builders
- Return DTOs (not entities) to prevent lazy loading issues
- Self-contained with all required joins
- Optimized for specific use cases

**Repository Pattern**:
- One repository per aggregate root (Enrollment, Grade, Classroom)
- Handles unit of work, transactions
- Provides CRUD + aggregate-specific operations
- Domain events published on changes

---

## 3. Common Query Conventions

### 3.1 Pagination

**Cursor-based** (preferred for large datasets):
```typescript
interface PaginatedResult<T> {
  data: T[];
  cursor?: string;  // base64 encoded last ID or timestamp
  has_more: boolean;
  total?: number;   // optional, expensive to compute
}
```

**Offset-based** (for admin lists):
```typescript
interface OffsetPaginatedResult<T> {
  data: T[];
  page: number;
  per_page: number;
  total: number;
  total_pages: number;
}
```

### 3.2 Filtering

Filters passed as typed objects:
```typescript
interface EnrollmentFilter {
  school_year_category_id?: UUID;
  classroom_id?: UUID;
  state?: EnrollmentState;
  student_id?: UUID;
  created_after?: Date;
  created_before?: Date;
}
```

Converted to SQL with parameterized queries (prevent injection).

### 3.3 Sorting

```typescript
type SortOrder = 'asc' | 'desc';
type SortSpec = Array<{ field: string; order: SortOrder }>;

// Example: sort by created_at descending, then student name ascending
const sort: SortSpec = [{ field: 'created_at', order: 'desc' }, { field: 'student_name', order: 'asc' }];
```

### 3.4 Field Selection (GraphQL-style projection)

```typescript
// Request only needed fields to reduce payload
const fields = ['id', 'enrollment_number', 'student.name', 'classroom.code', 'state'];
```

---

## 4. Selector Definitions

### 4.1 EnrollmentSelector

**Purpose**: Read queries for enrollments with associated data.

#### Get Enrollment by ID

```typescript
async getEnrollment(id: UUID): Promise<EnrollmentDetail | null> {
  return this.db.query(`
    SELECT
      e.id,
      e.enrollment_number,
      e.state,
      e.admission_date,
      e.level_id,
      l.code as level_code,
      l.name as level_name,
      e.track_id,
      t.code as track_code,
      t.name as track_name,
      e.promotion_decision,
      e.promotion_to_level_id,
      c.id as classroom_id,
      c.code as classroom_code,
      c.name as classroom_name,
      sp.id as student_id,
      sp.user_id,
      u.full_name as student_name,
      u.email as student_email,
      sp.student_number,
      syc.id as syc_id,
      syc.code as syc_code,
      sy.name as school_year_name
    FROM enrollment e
    JOIN school_year_category syc ON e.school_year_category_id = syc.id
    JOIN school_year sy ON syc.school_year_id = sy.id
    JOIN classroom c ON e.classroom_id = c.id
    JOIN student_profile sp ON e.student_profile_id = sp.id
    JOIN customuser u ON sp.user_id = u.id
    LEFT JOIN level l ON e.level_id = l.id
    LEFT JOIN track t ON e.track_id = t.id
    WHERE e.id = $1 AND e.is_active = TRUE
  `, [id]).then(this.mapEnrollmentDetail);
}
```

#### List Enrollments with Filter

```typescript
async listEnrollments(
  filter: EnrollmentFilter,
  options: PaginationOptions & SortOptions
): Promise<PaginatedResult<EnrollmentListItem>> {
  const conditions: string[] = ['e.is_active = TRUE'];
  const params: any[] = [];
  let paramCount = 0;

  if (filter.school_year_category_id) {
    paramCount++;
    conditions.push(`e.school_year_category_id = $${paramCount}`);
    params.push(filter.school_year_category_id);
  }

  if (filter.classroom_id) {
    paramCount++;
    conditions.push(`e.classroom_id = $${paramCount}`);
    params.push(filter.classroom_id);
  }

  if (filter.state) {
    paramCount++;
    conditions.push(`e.state = $${paramCount}`);
    params.push(filter.state);
  }

  if (filter.student_id) {
    paramCount++;
    conditions.push(`e.student_profile_id = $${paramCount}`);
    params.push(filter.student_id);
  }

  const whereClause = conditions.length > 0 ? `WHERE ${conditions.join(' AND ')}` : '';

  // Build ORDER BY
  const orderBy = this.buildOrderClause(options.sort, {
    defaults: [{ field: 'e.created_at', order: 'desc' }]
  });

  // Pagination
  const limit = options.per_page || 20;
  const offset = options.page ? (options.page - 1) * limit : 0;
  paramCount++;
  params.push(limit);
  paramCount++;
  params.push(offset);

  const query = `
    SELECT
      e.id,
      e.enrollment_number,
      e.state,
      c.code as classroom_code,
      u.full_name as student_name,
      l.code as level_code,
      t.code as track_code,
      e.created_at
    FROM enrollment e
    JOIN classroom c ON e.classroom_id = c.id
    JOIN student_profile sp ON e.student_profile_id = sp.id
    JOIN customuser u ON sp.user_id = u.id
    LEFT JOIN level l ON e.level_id = l.id
    LEFT JOIN track t ON e.track_id = t.id
    ${whereClause}
    ${orderBy}
    LIMIT $${paramCount} OFFSET $${paramCount + 1}
  `;

  const [rows, totalRows] = await Promise.all([
    this.db.query(query, params),
    this.db.query(
      `SELECT COUNT(*) FROM enrollment e ${whereClause}`,
      params.slice(0, -2)  // remove limit/offset
    )
  ]);

  return {
    data: rows.map(this.mapEnrollmentListItem),
    has_more: rows.length === limit && totalRows.rows[0].count > offset + limit,
    total: parseInt(totalRows.rows[0].count),
    page: options.page || 1,
    per_page: limit
  };
}
```

#### Get Enrollment Waitlist

```typescript
async getWaitlist(
  sycId: UUID,
  levelId?: UUID,
  trackId?: UUID
): Promise<WaitlistEntry[]> {
  return this.db.query(`
    SELECT
      ROW_NUMBER() OVER (ORDER BY e.created_at) as waitlist_position,
      e.id,
      e.created_at,
      sp.id as student_id,
      u.full_name as student_name,
      u.email as parent_email,
      e.level_id,
      l.code as level_code,
      e.track_id,
      t.code as track_code,
      e.scholarship_percent
    FROM enrollment e
    JOIN student_profile sp ON e.student_profile_id = sp.id
    JOIN customuser u ON sp.user_id = u.id
    LEFT JOIN level l ON e.level_id = l.id
    LEFT JOIN track t ON e.track_id = t.id
    WHERE e.school_year_category_id = $1
      AND e.state = 'PENDING'
      AND e.classroom_id IS NULL
      AND ($2::uuid IS NULL OR e.level_id = $2)
      AND ($3::uuid IS NULL OR e.track_id = $3)
    ORDER BY e.created_at
  `, [sycId, levelId, trackId]).then(this.mapWaitlistEntry);
}
```

---

### 4.2 GradeSelector

**Purpose**: Queries for grades with complex calculations.

#### Get Student Term Grades

```typescript
async getStudentTermGrades(
  studentId: UUID,
  syctId: UUID
): Promise<StudentTermGrades> {
  return this.db.query(`
    SELECT
      sub.id as subject_id,
      sub.code as subject_code,
      sub.name as subject_name,
      cs.coefficient as classroom_subject_coefficient,
      e.id as evaluation_id,
      e.code as evaluation_code,
      e.name as evaluation_name,
      e.coefficient as evaluation_coefficient,
      e.evaluation_type_id,
      et.code as evaluation_type_code,
      g.id as grade_id,
      g.value,
      g.state as grade_state,
      g.is_exempt,
      g.justification,
      syct.code as term_code,
      -- Calculate running class average for context
      AVG(g2.value) FILTER (WHERE g2.state = 'FINALIZED') OVER (
        PARTITION BY e.classroom_subject_id
      ) as class_average
    FROM grade g
    JOIN evaluation e ON g.evaluation_id = e.id
    JOIN school_year_category_term syct ON e.school_year_category_term_id = syct.id
    JOIN classroom_subject cs ON e.classroom_subject_id = cs.id
    JOIN subject sub ON cs.subject_id = sub.id
    JOIN evaluation_type et ON e.evaluation_type_id = et.id
    LEFT JOIN grade g2 ON e.id = g2.evaluation_id
      AND g2.state = 'FINALIZED'
    WHERE g.student_profile_id = $1
      AND syct.id = $2
      AND g.is_active = TRUE
      AND e.is_active = TRUE
    ORDER BY sub.display_order, e.evaluation_date
  `, [studentId, syctId]).then(this.mapStudentTermGrades);
}
```

**Optimization**: Use materialized view `student_report_card_data` for frequent queries.

#### Get Classroom Grade Sheet

```typescript
async getClassroomGradeSheet(
  classroomSubjectId: UUID,
  syctId: UUID,
  includeAbsent: boolean = true
): Promise<ClassroomGradeSheet> {
  const query = `
    SELECT
      e.id as evaluation_id,
      e.code as evaluation_code,
      e.name as evaluation_name,
      e.coefficient,
      e.max_score,
      e.state as evaluation_state,
      s.id as student_id,
      sp.student_number,
      u.full_name as student_name,
      g.id as grade_id,
      g.value,
      g.state as grade_state,
      g.is_exempt,
      g.justification,
      g.entered_at,
      u_teacher.full_name as entered_by_teacher
    FROM evaluation e
    JOIN enrollment en ON e.classroom_subject_id = (
      SELECT classroom_subject_id FROM evaluation WHERE id = e.id
    ) AND en.school_year_category_id = (
      SELECT school_year_category_id FROM school_year_category_term WHERE id = syctId
    ) AND en.state = 'ACTIVE'
    JOIN student_profile s ON en.student_profile_id = s.id
    JOIN customuser u ON s.user_id = u.id
    LEFT JOIN grade g ON e.id = g.evaluation_id AND g.student_profile_id = s.id
    LEFT JOIN teacher_profile tp_enter ON g.entered_by = tp_enter.id
    LEFT JOIN customuser u_teacher ON tp_enter.user_id = u_teacher.id
    WHERE e.classroom_subject_id = $1
      AND e.school_year_category_term_id = $2
      AND e.is_active = TRUE
      AND (g.id IS NOT NULL OR $3 = TRUE)  -- include absent students if flag set
    ORDER BY u.last_name, u.first_name, e.evaluation_date
  `;

  return this.db.query(query, [classroomSubjectId, syctId, includeAbsent])
    .then(this.mapClassroomGradeSheet);
}
```

---

### 4.3 ClassroomSelector

**Purpose**: Queries for classes, rosters, subject assignments.

#### Get Classroom with Roster

```typescript
async getClassroomWithRoster(
  classroomId: UUID
): Promise<ClassroomWithRoster | null> {
  // Single query with subqueries for aggregates to avoid N+1
  return this.db.query(`
    SELECT
      c.*,
      l.code as level_code,
      l.name as level_name,
      tr.code as track_code,
      tr.name as track_name,
      syc.code as syc_code,
      sy.code as school_year_code,
      u_teacher.full_name as homeroom_teacher_name,
      COUNT(en.id) FILTER (WHERE en.state = 'ACTIVE') as enrollment_count,
      ARRAY_AGG(DISTINCT cs.subject_id) as subject_ids
    FROM classroom c
    JOIN school_year_category syc ON c.school_year_category_id = syc.id
    JOIN school_year sy ON syc.school_year_id = sy.id
    JOIN level l ON c.level_id = l.id
    LEFT JOIN track tr ON c.track_id = tr.id
    LEFT JOIN teacher_profile tp ON c.homeroom_teacher_id = tp.id
    LEFT JOIN customuser u_teacher ON tp.user_id = u_teacher.id
    LEFT JOIN enrollment en ON en.classroom_id = c.id AND en.is_active = TRUE
    LEFT JOIN classroom_subject cs ON cs.classroom_id = c.id AND cs.is_active = TRUE
    WHERE c.id = $1 AND c.is_active = TRUE
    GROUP BY c.id, l.id, tr.id, syc.id, sy.id, u_teacher.id
  `, [classroomId]).then(row => row[0] ? this.mapClassroomWithRoster(row[0]) : null);
}
```

#### Get Classroom Students

```typescript
async getClassroomStudents(
  classroomId: UUID,
  options?: { include_inactive?: boolean }
): Promise<StudentProfile[]> {
  const includeInactive = options?.include_inactive ? 'OR en.state != ''ACTIVE''' : '';

  return this.db.query(`
    SELECT
      s.id,
      s.student_number,
      s.date_of_birth,
      s.current_level_id,
      u.full_name,
      u.email,
      u.phone_number,
      en.state as enrollment_state,
      en.admission_date,
      CASE
        WHEN p_guard.id IS NOT NULL THEN true
        ELSE false
      END as has_guardian_consent
    FROM classroom c
    JOIN enrollment en ON c.id = en.classroom_id
      AND en.is_active = TRUE
      AND (en.state = 'ACTIVE' ${includeInactive})
    JOIN student_profile s ON en.student_profile_id = s.id
    JOIN customuser u ON s.user_id = u.id
    LEFT JOIN parent_profile p_guard ON s.id = p_guard.student_id
      AND p_guard.is_active = TRUE
    WHERE c.id = $1
    ORDER BY u.last_name, u.first_name
  `, [classroomId]).then(this.mapStudentProfile);
}
```

---

### 4.4 TimetableSelector

**Purpose**: Schedule queries for teachers, students, classrooms.

#### Get Teacher Schedule

```typescript
async getTeacherSchedule(
  teacherId: UUID,
  options: {
    timetableId?: UUID;
    sycatId?: UUID;
    date?: Date;
    expandExceptions?: boolean;
  }
): Promise<TeacherSchedule> {
  const conditions: string[] = [
    'te.teacher_id = $1',
    'te.state = ''ACTIVE''',
    'te.is_exception = FALSE'
  ];
  const params: any[] = [teacherId];

  if (options.timetableId) {
    conditions.push('te.timetable_id = $' + (params.length + 1));
    params.push(options.timetableId);
  } else if (options.sycatId) {
    conditions.push('t.school_year_category_id = $' + (params.length + 1));
    params.push(options.sycatId);
    conditions.push('t.is_active = TRUE');
  }

  if (options.date) {
    // Check if date is exception; if so, switch query
    return this.getScheduleWithExceptions(teacherId, options.date, options.expandExceptions);
  }

  const query = `
    SELECT
      te.id as entry_id,
      ts.day_of_week,
      ts.period_number,
      ts.start_time,
      ts.end_time,
      cs.id as classroom_subject_id,
      c.code as classroom_code,
      c.name as classroom_name,
      l.code as level_code,
      tr.code as track_code,
      sub.code as subject_code,
      sub.name as subject_name,
      te.room_code,
      s.name as school_name,
      t.code as timetable_code
    FROM timetable_entry te
    JOIN timetable t ON te.timetable_id = t.id
    JOIN time_slot ts ON te.time_slot_id = ts.id
    JOIN classroom_subject cs ON te.classroom_subject_id = cs.id
    JOIN classroom c ON cs.classroom_id = c.id
    JOIN level l ON c.level_id = l.id
    LEFT JOIN track tr ON c.track_id = tr.id
    JOIN subject sub ON cs.subject_id = sub.id
    JOIN school_year_category syc ON t.school_year_category_id = syc.id
    JOIN school_year sy ON syc.school_year_id = sy.id
    JOIN school s ON sy.school_id = s.id
    WHERE ${conditions.join(' AND ')}
    ORDER BY ts.day_of_week, ts.period_number
  `;

  return this.db.query(query, params).then(this.mapTeacherSchedule);
}
```

#### Get Student Schedule

```typescript
async getStudentSchedule(
  studentId: UUID,
  date: Date
): Promise<DailySchedule | null> {
  const dayOfWeek = date.getDay() || 7; // Monday=1, Sunday=7

  return this.db.query(`
    WITH current_enrollment AS (
      SELECT e.classroom_id, e.school_year_category_id, syc.school_year_id
      FROM enrollment e
      JOIN school_year_category syc ON e.school_year_category_id = syc.id
      WHERE e.student_profile_id = $1
        AND e.state = 'ACTIVE'
        AND e.is_active = TRUE
        AND syc.school_year_id = (
          SELECT id FROM school_year
          WHERE start_date <= $2 AND end_date >= $2
          AND is_active = TRUE
          LIMIT 1
        )
      LIMIT 1
    ),
    active_timetable AS (
      SELECT t.id as timetable_id
      FROM timetable t
      WHERE t.school_year_category_id = (SELECT school_year_category_id FROM current_enrollment)
        AND t.is_active = TRUE
        AND ($2 BETWEEN t.effective_from_date AND COALESCE(t.effective_to_date, '9999-12-31'::DATE))
      LIMIT 1
    )
    SELECT
      ts.day_of_week,
      ts.period_number,
      ts.start_time,
      ts.end_time,
      c.code as classroom_code,
      sub.code as subject_code,
      sub.name as subject_name,
      tp.full_name as teacher_name,
      te.room_code,
      te.entry_type,
      te.is_exception,
      te.exception_reason
    FROM active_timetable at
    JOIN timetable_entry te ON at.timetable_id = te.timetable_id
      AND te.state = 'ACTIVE'
    JOIN time_slot ts ON te.time_slot_id = ts.id
      AND ts.day_of_week = $3
    JOIN classroom_subject cs ON te.classroom_subject_id = cs.id
    JOIN classroom c ON cs.classroom_id = c.id
    JOIN subject sub ON cs.subject_id = sub.id
    LEFT JOIN teacher_profile tp ON te.teacher_id = tp.id
    WHERE c.id = (SELECT classroom_id FROM current_enrollment)
      AND (te.is_exception = FALSE OR te.exception_date = $2)
    ORDER BY ts.period_number
  `, [studentId, date, dayOfWeek]).then(this.mapDailySchedule);
}
```

---

### 4.5 ReportSelector

**Purpose**: Reporting queries with aggregates.

#### Generate Report Card Data

```typescript
async getReportCardData(
  enrollmentId: UUID,
  termCode: string
): Promise<ReportCardData> {
  // Leverage materialized view for cached term averages where possible
  return this.db.query(`
    WITH syct AS (
      SELECT id FROM school_year_category_term WHERE code = $2
    ),
    student_evaluations AS (
      SELECT
        cs.id as classroom_subject_id,
        sub.id as subject_id,
        sub.code as subject_code,
        sub.name as subject_name,
        sub.category as subject_category,
        cs.coefficient as subject_coefficient,
        u_teacher.full_name as teacher_name,
        ARRAY_AGG(
          jsonb_build_object(
            'evaluation_id', e.id,
            'name', e.name,
            'coefficient', e.coefficient,
            'max_score', e.max_score,
            'grade', g.value,
            'grade_state', g.state,
            'is_exempt', g.is_exempt,
            'justification', g.justification
          )
        ) FILTER (WHERE g.id IS NOT NULL) as evaluations,
        COALESCE(AVG(g.value) FILTER (WHERE g.state = 'FINALIZED' AND g.is_exempt = FALSE), 0) as term_average,
        COUNT(g.id) FILTER (WHERE g.state = 'FINALIZED') as grades_count
      FROM enrollment e_main
      JOIN classroom c ON e_main.classroom_id = c.id
      JOIN classroom_subject cs ON c.id = cs.classroom_id AND cs.is_active = TRUE
      JOIN subject sub ON cs.subject_id = sub.id
      JOIN evaluation e ON cs.id = e.classroom_subject_id
        AND e.school_year_category_term_id = (SELECT id FROM syct)
        AND e.state = 'FINALIZED'
        AND e.is_active = TRUE
      LEFT JOIN grade g ON e.id = g.evaluation_id
        AND g.student_profile_id = e_main.student_profile_id
        AND g.is_active = TRUE
      LEFT JOIN teacher_profile tp ON cs.teacher_id = tp.id
      LEFT JOIN customuser u_teacher ON tp.user_id = u_teacher.id
      WHERE e_main.id = $1
        AND e_main.state = 'ACTIVE'
        AND e_main.is_active = TRUE
      GROUP BY cs.id, sub.id, u_teacher.full_name
      ORDER BY sub.display_order
    )
    SELECT
      se.*,
      -- Class average for subject
      (SELECT AVG(g2.value)
       FROM grade g2
       JOIN evaluation e2 ON g2.evaluation_id = e2.id
       WHERE e2.classroom_subject_id = se.classroom_subject_id
         AND g2.state = 'FINALIZED'
         AND g2.is_exempt = FALSE) as class_average,
      -- Class rank
      (SELECT COUNT(*) + 1
       FROM (
         SELECT g3.student_profile_id, AVG(g3.value) as avg_grade
         FROM grade g3
         JOIN evaluation e3 ON g3.evaluation_id = e3.id
         WHERE e3.classroom_subject_id = se.classroom_subject_id
           AND g3.state = 'FINALIZED'
           AND g3.is_exempt = FALSE
         GROUP BY g3.student_profile_id
       ) ranked
       WHERE ranked.avg_grade > (
         SELECT AVG(g4.value)
         FROM grade g4
         JOIN evaluation e4 ON g4.evaluation_id = e4.id
         WHERE e4.classroom_subject_id = se.classroom_subject_id
           AND g4.student_profile_id = e_main.student_profile_id
           AND g4.state = 'FINALIZED'
           AND g4.is_exempt = FALSE
       )
      ) as class_rank,
      (SELECT COUNT(*) FROM enrollment WHERE classroom_id = c.id AND state = 'ACTIVE') as class_size
    FROM student_evaluations se
    CROSS JOIN enrollment e_main
    JOIN classroom c ON e_main.classroom_id = c.id
  `, [enrollmentId, termCode]).then(this.mapReportCardData);
}
```

---

### 4.6 SearchSelector

**Purpose**: Full-text and faceted search across entities.

#### Global Search

```typescript
async globalSearch(
  query: string,
  filters: {
    entity_type?: 'student' | 'teacher' | 'parent';
    school_year_category_id?: UUID;
  },
  options: PaginationOptions
): Promise<SearchResults> {
  // Use PostgreSQL full-text search with trigram similarity
  return this.db.query(`
    SELECT
      sqlc.record_id as id,
      sqlc.record_type as type,
      sqlc.rank,
      u.full_name,
      u.email,
      sp.student_number,
      tp.employee_number,
      jsonb_build_object(
        'id', sqlc.record_id,
        'type', sqlc.record_type,
        'name', u.full_name,
        'email', u.email,
        'student_number', sp.student_number,
        'employee_number', tp.employee_number
      ) as result
    FROM (
      SELECT
        record_id,
        record_type,
        ts_rank_cd(
          to_tsvector('french', searchable_text),
          plainto_tsquery('french', $1)
        ) as rank
      FROM search_index
      WHERE searchable_text @@ plainto_tsquery('french', $1)
        AND ($2::varchar IS NULL OR record_type = $2)
      ORDER BY rank DESC
      LIMIT $3 OFFSET $4
    ) sqlc
    JOIN customuser u ON sqlc.record_id = u.id
    LEFT JOIN student_profile sp ON sqlc.record_id = sp.id AND sqlc.record_type = 'student'
    LEFT JOIN teacher_profile tp ON sqlc.record_id = tp.id AND sqlc.record_type = 'teacher'
  `, [query, filters.entity_type, options.per_page, (options.page - 1) * options.per_page])
    .then(this.mapSearchResults);
}
```

**Search Index Population**:
```sql
-- Materialized view for search optimization
CREATE MATERIALIZED VIEW search_index AS
SELECT
  u.id as record_id,
  'student' as record_type,
  u.full_name || ' ' || COALESCE(sp.student_number, '') || ' ' ||
  COALESCE(u.email, '') as searchable_text
FROM customuser u
JOIN student_profile sp ON u.id = sp.user_id
WHERE sp.is_active = TRUE
UNION ALL
SELECT
  u.id,
  'teacher' as record_type,
  u.full_name || ' ' || COALESCE(tp.employee_number, '') || ' ' ||
  COALESCE(u.email, '') || ' ' || array_to_string(tp.taught_subjects, ' ')
FROM customuser u
JOIN teacher_profile tp ON u.id = tp.user_id
WHERE tp.is_active = TRUE;

CREATE INDEX idx_search_index_fts ON search_index USING GIN(to_tsvector('french', searchable_text));
CREATE INDEX idx_search_index_trigger ON search_index(record_type, record_id);
```

Refresh hourly or on user change.

---

## 5. Repository Interfaces

### 5.1 EnrollmentRepository

```typescript
interface EnrollmentRepository {
  // CRUD
  findById(id: UUID): Promise<Enrollment | null>;
  save(enrollment: Enrollment): Promise<Enrollment>;
  delete(id: UUID): Promise<void>;

  // Aggregate-specific
  createFromApplication(
    application: EnrollmentApplication,
    operator: UserContext
  ): Promise<Enrollment>;

  activate(id: UUID, operator: UserContext): Promise<Enrollment>;

  withdraw(id: UUID, reason: WithdrawalReason, operator: UserContext): Promise<Enrollment>;

  transfer(id: UUID, targetClassroomId: UUID, operator: UserContext): Promise<Enrollment>;

  promote(
    enrollmentId: UUID,
    toLevelId: UUID,
    toTrackId?: UUID,
    decision: PromotionDecision,
    notes?: string
  ): Promise<Enrollment>;

  getByStudentAndYear(
    studentId: UUID,
    schoolYearId: UUID
  ): Promise<Enrollment | null>;

  getClassroomRoster(classroomId: UUID): Promise<Enrollment[]>;

  getWaitlistPosition(enrollmentId: UUID): Promise<number>;

  applyWaitlistToVacancy(waitlistId: UUID): Promise<Enrollment>;
}
```

**Implementation Notes**:
- Uses database transactions for multi-step operations
- Emits domain events after commit
- Handles optimistic locking with `version` column
- Retry logic for deadlock detection

---

### 5.2 GradeRepository

```typescript
interface GradeRepository {
  // Grade operations
  findById(id: UUID): Promise<Grade | null>;
  findByEvaluationAndStudent(evaluationId: UUID, studentId: UUID): Promise<Grade | null>;

  save(grade: Grade): Promise<Grade>;

  enterGrade(
    evaluationId: UUID,
    studentId: UUID,
    value: number,
    enteredBy: UUID,
    justification?: string
  ): Promise<Grade>;

  submitEvaluation(evaluationId: UUID, submittedBy: UUID): Promise<Evaluation>;

  finalizeEvaluation(evaluationId: UUID, finalizedBy: UUID): Promise<Evaluation>;

  // Grade amendments (appeals)
  createAmendment(
    originalGradeId: UUID,
    newValue: number,
    appealId: UUID,
    amendedBy: UUID
  ): Promise<Grade>;

  // Queries
  getStudentTermGrades(
    studentId: UUID,
    syctId: UUID
  ): Promise<StudentTermGrades>;

  getClassroomGradeSheet(
    classroomSubjectId: UUID,
    syctId: UUID
  ): Promise<ClassroomGradeSheet>;

  getEvaluationGrades(evaluationId: UUID): Promise<Grade[]>;

  getPendingFinalization(evaluationId: UUID): Promise<Grade[]>;

  // Bulk operations (for batch grade entry by teacher)
  batchEnterGrades(
    entries: Array<{ evaluationId: UUID; studentId: UUID; value: number; justification?: string }>,
    enteredBy: UUID
  ): Promise<Grade[]>;

  // Statistics
  getEvaluationStatistics(evaluationId: UUID): Promise<EvaluationStatistics>;
  getSubjectStatistics(
    classroomSubjectId: UUID,
    syctId: UUID
  ): Promise<SubjectStatistics>;
}
```

---

### 5.3 TimetableRepository

```typescript
interface TimetableRepository {
  // CRUD
  findById(id: UUID): Promise<Timetable | null>;
  findActiveBySycat(sycatId: UUID): Promise<Timetable | null>;
  save(timetable: Timetable): Promise<Timetable>;

  // Generation
  generate(
    sycatId: UUID,
    method: GenerationMethod,
    options: GenerationOptions
  ): Promise<TimetableGenerationResult>;

  publish(timetableId: UUID, publisherId: UUID): Promise<Timetable>;

  // Entries
  addEntry(entry: TimetableEntryCreate): Promise<TimetableEntry>;
  updateEntry(id: UUID, updates: TimetableEntryUpdate): Promise<TimetableEntry>;
  deleteEntry(id: UUID): Promise<void>;

  // Exceptions
  createException(exception: TimetableExceptionCreate): Promise<TimetableException>;

  // Queries
  getTeacherSchedule(
    teacherId: UUID,
    options: ScheduleQueryOptions
  ): Promise<TeacherSchedule>;

  getClassroomSchedule(
    classroomId: UUID,
    options: ScheduleQueryOptions
  ): Promise<ClassroomSchedule>;

  getStudentSchedule(
    studentId: UUID,
    date: Date
  ): Promise<DailySchedule>;

  checkConflicts(timetableId: UUID): Promise<ConflictReport>;

  // Versioning
  getHistory(sycatId: UUID): Promise<Timetable[]>;
  rollbackToVersion(timetableId: UUID): Promise<void>;
}
```

---

## 6. Materialized Views & Cache Strategy

### 6.1 Materialized Views

**Purpose**: Pre-compute expensive aggregates for reporting.

#### Report Card Materialized View

```sql
CREATE MATERIALIZED VIEW report_card_cache AS
SELECT
    e.enrollment_id,
    e.student_profile_id,
    syc.id as syc_id,
    syc.code as syc_code,
    ts.code as term_code,
    cs.id as classroom_subject_id,
    sub.id as subject_id,
    sub.code as subject_code,
    sub.name as subject_name,
    sub.category as subject_category,
    COUNT(DISTINCT e2.id) as evaluation_count,
    SUM(e2.coefficient) FILTER (WHERE g.state = 'FINALIZED') as weighted_sum,
    SUM(e2.coefficient) FILTER (WHERE g.state = 'FINALIZED' AND g.is_exempt = FALSE) as total_coefficient,
    ROUND(
        SUM(g.value * e2.coefficient) FILTER (WHERE g.state = 'FINALIZED' AND g.is_exempt = FALSE) /
        NULLIF(SUM(e2.coefficient) FILTER (WHERE g.state = 'FINALIZED' AND g.is_exempt = FALSE), 0),
        2
    ) as subject_average,
    MAX(g.finalized_at) as last_grade_finalized
FROM enrollment e
JOIN school_year_category syc ON e.school_year_category_id = syc.id
JOIN school_year_category_term ts ON syc.id = ts.school_year_category_id
JOIN classroom c ON e.classroom_id = c.id
JOIN classroom_subject cs ON c.id = cs.classroom_id AND cs.is_active = TRUE
JOIN subject sub ON cs.subject_id = sub.id
JOIN evaluation e2 ON cs.id = e2.classroom_subject_id
    AND e2.school_year_category_term_id = ts.id
    AND e2.state = 'FINALIZED'
    AND e2.is_active = TRUE
LEFT JOIN grade g ON e2.id = g.evaluation_id
    AND g.student_profile_id = e.student_profile_id
    AND g.state = 'FINALIZED'
    AND g.is_active = TRUE
WHERE e.state = 'ACTIVE'
    AND e.is_active = TRUE
    AND ts.is_reporting_term = TRUE
GROUP BY e.enrollment_id, e.student_profile_id, syc.id, syc.code, ts.code,
         cs.id, sub.id, sub.code, sub.name, sub.category;

-- Indexes
CREATE UNIQUE INDEX idx_report_card_cache_unique ON report_card_cache(
    enrollment_id, syc_id, term_code, subject_id
);
CREATE INDEX idx_report_card_cache_student ON report_card_cache(student_profile_id, syc_id);
```

**Refresh Strategy**:
- Full refresh: nightly at 2 AM
- Incremental refresh: when evaluation finalization triggers (async job)
- Refresh on demand: `REFRESH MATERIALIZED VIEW CONCURRENTLY report_card_cache`

---

#### Class Average Materialized View

```sql
CREATE MATERIALIZED VIEW class_average_cache AS
SELECT
    c.id as classroom_id,
    syc.id as syc_id,
    cs.id as classroom_subject_id,
    sub.id as subject_id,
    COUNT(DISTINCT e.student_profile_id) as student_count,
    ROUND(AVG(g.value) FILTER (WHERE g.state = 'FINALIZED' AND g.is_exempt = FALSE), 2) as class_average,
    ROUND(STDDEV(g.value) FILTER (WHERE g.state = 'FINALIZED' AND g.is_exempt = FALSE), 2) as standard_deviation,
    MIN(g.value) FILTER (WHERE g.state = 'FINALIZED' AND g.is_exempt = FALSE) as min_score,
    MAX(g.value) FILTER (WHERE g.state = 'FINALIZED' AND g.is_exempt = FALSE) as max_score,
    COUNT(*) FILTER (WHERE g.value >= 10) as passing_count,
    ROUND(
        COUNT(*) FILTER (WHERE g.value >= 10)::numeric /
        NULLIF(COUNT(*) FILTER (WHERE g.state = 'FINALIZED' AND g.is_exempt = FALSE), 0) * 100,
        1
    ) as passing_percentage
FROM classroom c
JOIN school_year_category syc ON c.school_year_category_id = syc.id
JOIN classroom_subject cs ON c.id = cs.classroom_id AND cs.is_active = TRUE
JOIN subject sub ON cs.subject_id = sub.id
JOIN enrollment e ON c.id = e.classroom_id AND e.state = 'ACTIVE' AND e.is_active = TRUE
JOIN grade g ON e.student_profile_id = g.student_profile_id
JOIN evaluation ev ON g.evaluation_id = ev.id
    AND ev.classroom_subject_id = cs.id
    AND ev.school_year_category_term_id IN (
        SELECT id FROM school_year_category_term WHERE school_year_category_id = syc.id
    )
    AND ev.state = 'FINALIZED'
WHERE g.state = 'FINALIZED' AND g.is_exempt = FALSE
GROUP BY c.id, syc.id, cs.id, sub.id;
```

Used for:
- Class ranking in report cards
- Teacher performance metrics
- School-wide performance dashboards

---

### 6.2 Redis Cache Pattern

**Cache Keys**:

```
# User permissions (5 min TTL)
perm:{tenant_id}:{user_id} → serialized permissions array

# Student schedule (1 hour TTL, invalidated on timetable change)
schedule:student:{student_id}:{date} → DailySchedule JSON

# Teacher schedule (1 hour TTL)
schedule:teacher:{teacher_id}:{timetable_id} → TeacherSchedule JSON

# Classroom roster (30 min TTL)
roster:classroom:{classroom_id} → Enrollment[] JSON

# Grade evaluation completion (5 min TTL)
grade:completion:{evaluation_id} → {completed: 23, total: 30, percentage: 76.66}

# Student term average (computed, 15 min TTL)
grade:average:{student_id}:{syct_id} → {average: 14.5, rank: 5, class_size: 30}
```

**Invalidation Strategy**:
- Write-through: Updates also invalidate relevant cache keys
- Event-driven: Domain events trigger cache invalidation
- TTL-based: Stale data naturally expires

---

## 7. N+1 Query Prevention

### 7.1 The Problem

```typescript
// BAD: N+1 query pattern
const enrollments = await enrollmentRepo.list(filter);
for (const enrollment of enrollments) {
  const student = await studentRepo.findById(enrollment.studentId); // N queries!
  const classroom = await classroomRepo.findById(enrollment.classroomId); // N queries!
}
```

### 7.2 The Solution

**Eager loading via joins** (single query):

```typescript
// GOOD: Single query with all joins
const enrollments = await enrollmentRepo.listWithRelations(filter, {
  include: ['student', 'classroom', 'level', 'track']
});

// Implementation uses LEFT JOINs and maps results properly,
// returning hydrated objects without additional queries
```

**Repository method with includes**:
```typescript
list(
  filter: EnrollmentFilter,
  options: {
    include?: ('student' | 'classroom' | 'level' | 'track')[];
  } = {}
): Promise<Enrollment[]> {
  const joins = this.buildJoins(options.include || []);
  const query = `
    SELECT e.*, ${this.selectFieldsForIncludes(options.include)}
    FROM enrollment e
    ${joins}
    WHERE ${whereClause}
    ORDER BY ...
  `;
  // Map rows, grouping child records as needed
}
```

---

## 8. Batch Operations

### 8.1 Batch Grade Entry (Teacher UX)

```typescript
async batchEnterGrades(
  evaluationId: UUID,
  grades: Array<{ studentId: UUID; value: number; justification?: string }>,
  operatorId: UUID
): Promise<BatchResult<Grade>> {
  return await this.db.transaction(async trx => {
    const results: BatchResult<Grade> = { succeeded: [], failed: [] };

    // Lock evaluation to prevent concurrent modification
    const evaluation = await trx.evaluation.find({ id: evaluationId, forUpdate: true });

    if (evaluation.state !== 'PUBLISHED' && evaluation.state !== 'IN_PROGRESS') {
      throw new Error('Evaluation not open for grade entry');
    }

    if (evaluation.grade_submission_deadline < new Date()) {
      throw new Error('Grade submission deadline has passed');
    }

    for (const gradeData of grades) {
      try {
        const grade = await this.gradeRepository.enterGrade(
          evaluationId,
          gradeData.studentId,
          gradeData.value,
          operatorId,
          gradeData.justification
        );
        results.succeeded.push(grade);
      } catch (error) {
        results.failed.push({
          studentId: gradeData.studentId,
          error: error.message,
          errorCode: error.code
        });
      }
    }

    // Update evaluation completion count
    await trx.evaluation.update(
      { id: evaluationId },
      {
        total_students_completed: results.succeeded.length,
        completion_percentage: (results.succeeded.length / evaluation.total_students_expected) * 100
      }
    );

    return results;
  });
}
```

---

## 9. Query Performance Checklist

### 9.1 Every Query Should:

- ✅ Use parameterized queries (prevent SQL injection)
- ✅ Have appropriate indexes on WHERE, JOIN, ORDER BY columns
- ✅ Use `EXPLAIN ANALYZE` to verify plan during development
- ✅ Limit result sets (pagination)
- ✅ Avoid SELECT * (explicit column lists)
- ✅ Consider materialized view for >3 joins

### 9.2 Critical Queries Must Have:

- ✅ Performance tests with production-scale data
- ✅ Execution time < 200ms (p95)
- ✅ Index usage verified (no sequential scans)
- ✅ Connection pool sizing appropriate
- ✅ Monitoring alerts on slow queries (>500ms)

---

## 10. Index Design Guidelines

| Query Pattern | Recommended Index |
|---------------|-------------------|
| Find by UUID (primary lookup) | PRIMARY KEY (id) |
| Foreign key lookups | `INDEX(table_fk_id)` |
| Composite filters | `INDEX(fk1, fk2, created_at)` |
| Active records only | `PARTIAL INDEX WHERE is_active = TRUE` |
| Date range queries | `INDEX(created_at DESC)` |
| Multi-column WHERE | `INDEX(col1, col2, col3)` |
| Text search | `GIN(to_tsvector(...))` |

**Example**: Enrollment classroom access pattern:
```sql
-- Common: Find all ACTIVE enrollments in a classroom
CREATE INDEX idx_enrollment_classroom_active ON enrollment(classroom_id)
WHERE is_active = TRUE AND state = 'ACTIVE';

-- Common: Find student's enrollment history
CREATE INDEX idx_enrollment_student ON enrollment(student_profile_id, school_year_category_id DESC);
```

---

## 11. Selector/Repository Testing

### 11.1 Unit Tests

```typescript
describe('EnrollmentSelector', () => {
  it('should return enrollment with all joins', async () => {
    const result = await selector.getEnrollment(enrollmentId);
    expect(result).toMatchObject({
      student: { name: expect.any(String) },
      classroom: { code: expect.any(String) },
      level: { code: expect.any(String) }
    });
  });

  it('should paginate correctly', async () => {
    const result = await selector.listEnrollments(filter, { page: 2, per_page: 10 });
    expect(result.data).toHaveLength(10);
    expect(result.has_more).toBe(true);
  });
});
```

### 11.2 Integration Tests (with real DB)

- Use test database with fixtures
- Verify query plans (no bad indexes)
- Test pagination edge cases (last page, empty results)
- Concurrent modification scenarios

---

## 12. Database Connection Pooling

**Configuration** (example for PostgreSQL):

```typescript
const pool = new Pool({
  max: 20,                    // max connections
  min: 2,                     // min idle connections
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
  // Separate pools for different workloads:
  // - High concurrency: grade entry (50 connections)
  // - Reporting: long-running queries (10 connections)
});
```

**Query Timeouts**:
```typescript
// Set per-query timeout to prevent runaway queries
const result = await pool.query('SELECT ...', [], { timeout: '5s' });
```

---

**Next Step**: TDR-012 — Offline Sync Strategy (conflict resolution, sync flows, queue management)
