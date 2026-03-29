# TDR-006 — Domain Rules: Timetables

## 1. Purpose and Scope

This document defines the domain rules for the timetabling system (Étape 5). Timetables represent one of the most complex scheduling challenges in school management:

- **Multi-dimensional constraints** (room availability, teacher schedules, subject hours)
- **Recurring patterns** across days, weeks, and terms
- **Conflict detection** (no teacher in two places, no classroom double-booked)
- **Change propagation** when modifications occur mid-year
- **Offline access** for teachers and students to view schedules

**Key Principle**: Timetables are derived from ClassroomSubject assignments but can have exceptions and adjustments.

---

## 2. TimeSlot Domain Rules

### 2.1 Definition

**TimeSlot** represents a standardized time period in the school day/week. It's the atomic unit of scheduling.

### 2.2 Domain Rules

1. **R-TS-001**: TimeSlot is a **global referential table** (school-agnostic)
   - Defined once during system initialization
   - All schools in the system share same TimeSlot definitions
   - Typical pattern: 8 periods per day, each 45-55 minutes

2. **R-TS-002**: TimeSlot code format must be globally unique:
   ```
   [day_code]_[period_number]
   Examples: "MON_1", "MON_2", ..., "FRI_8"
   ```
   - `day_code`: "MON", "TUE", "WED", "THU", "FRI", "SAT"
   - `period_number`: 1-8 (or up to 12 for some systems)

3. **R-TS-003**: Standard school day structure:
   - Periods 1-6: Core instruction (8:00-14:30 typical)
   - Period 7: Afternoon classes (14:45-16:30)
   - Period 8: Optional/extra classes (16:45-18:30)
   - Break slots: "RECESS_1", "RECESS_2", "LUNCH" between periods

4. **R-TS-004**: TimeSlot duration configuration:
   - `start_time` and `end_time` are consistent across all school days
   - Some slots may be shorter (recess = 15 min, lunch = 45 min)
   - Total instructional time per week computed by summing subject time slots

5. **R-TS-005**: TimeSlot validity:
   - `is_active` flag determines if slot can be scheduled
   - During exams: regular slots deactivated, exam slots created
   - Special events: substitute slots created for that date

6. **R-TS-006**: Week rotation pattern:
   - Some schools use A/B weeks (Week A / Week B alternating)
   - TimeSlot can have `rotation_pattern` indicating:
     - "DAILY" — every day
     - "A_WEEK" — only on A weeks
     - "B_WEEK" — only on B weeks
     - "MON_WED_FRI" — specific days only

**Fields**:
- `id` (UUID)
- `code` (VARCHAR[20], unique) — "MON_1", "TUE_3", "FRI_7"
- `name` (VARCHAR[100]) — "Monday Period 1", "Tuesday 3rd Hour"
- `day_of_week` (INTEGER, 1=Monday, 7=Sunday)
- `period_number` (INTEGER)
- `start_time` (TIME) — e.g., "08:00:00"
- `end_time` (TIME)
- `duration_minutes` (INTEGER, computed) = end_time - start_time
- `rotation_pattern` (VARCHAR[20]) — "DAILY", "A_WEEK", "B_WEEK", "MON_WED_FRI"
- `slot_type` (VARCHAR[20]) — "REGULAR", "RECESS", "LUNCH", "EXAM", "ASSEMBLY"
- `is_active` (BOOLEAN)
- `is_instructional` (BOOLEAN) — does this count toward required hours?
- Audit trail

**Constraints**:
- `UNIQUE(day_of_week, period_number)` — only one slot per day/period combination
- `CHECK(end_time > start_time)` — valid duration

---

## 3. Timetable Domain Rules

### 3.1 Definition

**Timetable** represents the scheduled arrangement of classes for a specific SchoolYearCategory. It's the container for all TimeSlot → ClassroomSubject assignments.

### 3.2 Domain Rules

1. **R-TT-001**: Timetable lifecycle:
   ```
   DRAFT → ACTIVE → MODIFIED → ACTIVE → ARCHIVED
   ```
   - DRAFT: Initial creation, can be modified freely
   - ACTIVE: Current published schedule, enforced for attendance
   - MODIFIED: Changes made to active timetable (requires approval)
   - ARCHIVED: Previous timetable retained for history

2. **R-TT-002**: **One timetable per SchoolYearCategory per academic period**
   - Default: One timetable covers entire SchoolYearCategory
   - Some schools create separate timetables per term (if schedules change)
   - Timetable uniqueness: `(school_year_category_id, effective_from_date)`

3. **R-TT-003**: Timetable creation requirements:
   - Must have all TimeSlots defined for the school
   - All ClassroomSubject assignments must exist for the SchoolYearCategory
   - Total weekly hours per ClassroomSubject must match SubjectChoice.weekly_hours
   - No scheduling conflicts (checked by algorithm)

4. **R-TT-004**: Timetable modification policy:
   - Changes to ACTIVE timetable require:
     - Reason documented
     - Approval by School Admin + affected teachers
     - Notification to students/parents
   - Minor changes (room swaps) may be fast-tracked
   - Major changes (period shifts) require 1-week notice

5. **R-TT-005**: Timetable versioning:
   - Each modification creates new version (incremented)
   - Previous versions retained for conflict resolution ("what was the schedule on March 15?")
   - Version 1 = initial creation
   - Version.active = true for current version only

6. **R-TT-006**: Timetable scope:
   - Covers all classrooms in the SchoolYearCategory
   - Includes teacher assignments (which teacher teaches which subject when)
   - Includes room assignments (which classroom/lab)
   - May include special scheduling (half-days, exam weeks, events)

7. **R-TT-007**: **Timetable generation modes**:
   - **AUTOMATIC**: System generates conflict-free schedule based on constraints
   - **MANUAL**: Admin manually creates entries (flexible, risk of conflicts)
   - **HYBRID**: Auto-generated base + manual adjustments

**Fields**:
- `id` (UUID)
- `school_year_category_id` (UUID, foreign key to SchoolYearCategory)
- `code` (VARCHAR[50]) — "2024-2025-PRIM-TT01"
- `name` (VARCHAR[100]) — "Timetable Primaire 2024-2025"
- `version` (INTEGER, default 1)
- `effective_from_date` (DATE) — when this version becomes active
- `effective_to_date` (DATE) — when replaced (NULL for current)

- `generation_method` (VARCHAR[20]) — "AUTOMATIC", "MANUAL", "HYBRID"
- `generation_parameters` (JSONB) — algorithm settings

- `status` (VARCHAR[20]) — "DRAFT", "ACTIVE", "MODIFIED", "ARCHIVED"
- `is_active` (BOOLEAN) — true for current active version only

- `total_periods_per_week` (INTEGER) — computed
- `notes` (TEXT)
- `approved_by` (UUID)
- `approved_at` (TIMESTAMPTZ)

- Audit trail

**Unique Constraints**:
- `UNIQUE(school_year_category_id, effective_from_date)` — one timetable per period
- Only one active timetable per SchoolYearCategory: partial unique index on `(school_year_category_id) WHERE is_active = TRUE`

---

### 3.8 Invariants

**I-TT-001 (Single Active)**:
```
For each SchoolYearCategory:
    COUNT(active timetable) = 1
```

**I-TT-002 (No Overlap)**:
Active timetable's date range must not overlap with another timetable for same category:
```
NOT EXISTS (
    SELECT 1 FROM timetable t2
    WHERE t2.school_year_category_id = NEW.school_year_category_id
      AND t2.id != NEW.id
      AND t2.is_active = TRUE
      AND t2.effective_from_date <= NEW.effective_to_date
      AND COALESCE(t2.effective_to_date, '9999-12-31') >= NEW.effective_from_date
)
```

**I-TT-003 (Weekly Hours)**:
Sum of all TimetableEntry durations for a ClassroomSubject must equal SubjectChoice.weekly_hours.

---

## 4. TimetableEntry Domain Rules

### 4.1 Definition

**TimetableEntry** is the atomic schedule item: "Class 5ème A has Mathematics with Prof. Diallo on Monday Period 1 in Room 102."

### 4.2 Domain Rules

1. **R-TE-001**: Each TimetableEntry represents one scheduled instance per week
   - Regular classes: "every Monday Period 1"
   - Double periods: separate entries for Period 1 and Period 2 (but conceptually linked)
   - Lab sessions: may be in different room than regular class

2. **R-TE-002**: Entry uniqueness per timetable:
   ```
   UNIQUE(timetable_id, time_slot_id, classroom_subject_id)
       — No same subject scheduled in same slot twice
   UNIQUE(timetable_id, time_slot_id, teacher_id)
       — No teacher double-booked
   UNIQUE(timetable_id, time_slot_id, room_number) if rooms tracked
       — No room double-booked
   ```

3. **R-TE-003**: Subject assignment constraints:
   - Each ClassroomSubject must have minimum weekly hours allocated
   - Total weekly slots for a ClassroomSubject must equal SubjectChoice.weekly_hours
   - Subjects with practical hours must have at least one lab/different room slot

4. **R-TE-004**: Teacher assignment:
   - Teacher must be assigned to the ClassroomSubject (classroom_subject.teacher_id)
   - Teacher's schedule cannot exceed max hours per week (teacher profile setting)
   - Teacher's subject competence must match (taught_subjects includes this subject)

5. **R-TE-005**: Room assignment (optional but recommended):
   - If Subject.has_practical = TRUE, must assign a lab/practical room for at least one slot
   - Classroom capacity must ≥ classroom.max_capacity
   - Special equipment requirements (computers, science lab, gym) must match subject needs

6. **R-TE-006**: Scheduling patterns:
   ```
   Pattern 1: Same slot every week (standard)
      time_slot_id references Mon_1 → every Monday Period 1

   Pattern 2: Alternate weeks (A/B schedule)
      time_slot_id references Mon_1 + rotation=A_WEEK

   Pattern 3: Block scheduling
      multiple consecutive time slots allocated as single block
      (represented as separate entries but displayed as block)
   ```

7. **R-TE-007**: Exceptions and substitutions:
   - Regular timetable entry provides default schedule
   - TimetableException table records deviations:
     - Teacher absence → substitute teacher assigned
     - Room change for special activity
     - Cancelled class
   - Exceptions viewed on a date-specific basis

8. **R-TE-008**: TimetableEntry state:
   - `ACTIVE`: Normal scheduled entry
   - `CANCELLED`: Entry cancelled for specific date range
   - `MOVED`: Entry rescheduled to different slot (original marked CANCELLED, new created)
   - `EXCEPTION`: One-off modified entry (linked to exception record)

**Fields**:
- `id` (UUID)
- `timetable_id` (UUID, foreign key to Timetable, ON DELETE CASCADE)

- `time_slot_id` (UUID, foreign key to TimeSlot)
- `classroom_subject_id` (UUID, foreign key to ClassroomSubject)
- `teacher_id` (UUID, foreign key to TeacherProfile)
- `room_code` (VARCHAR[20]) — optional, for room booking systems

- `schedule_pattern` (VARCHAR[30]) — "EVERY_WEEK", "A_WEEK_ONLY", "B_WEEK_ONLY", "MON_WED_FRI"
- `effective_from_date` (DATE, default = timetable.effective_from_date)
- `effective_to_date` (DATE, nullable for open-ended schedule)

- `is_exception` (BOOLEAN, default FALSE)
- `exception_reason` (TEXT)
- `original_entry_id` (UUID, self-reference for moves)

- `state` (VARCHAR[20]) — "ACTIVE", "CANCELLED", "MOVED"
- `is_active` (BOOLEAN)

- Audit trail

**Unique Constraints**:
- `UNIQUE(timetable_id, time_slot_id, classroom_subject_id, schedule_pattern)`
- `UNIQUE(timetable_id, time_slot_id, teacher_id, schedule_pattern)` — no teacher conflicts
- Index on `(timetable_id, classroom_subject_id)` for loading full subject schedule

---

### 4.3 Scheduling Conflict Detection

**Conflict types to prevent**:

| Conflict | Detection | Impact |
|----------|-----------|--------|
| Teacher double-booked | Check other entries: teacher_id same, overlapping time_slot | Teacher cannot be in two places |
| Room double-booked | Check room booking: room_code same, overlapping time_slot | No room available |
| Student timetable collision | Check classroom's student roster against other classroom at same slot | Student in two classes |
| Teacher max hours exceeded | SUM(duration) for teacher > teacher.weekly_hour_limit | Teacher overload |

**Conflict prevention strategies**:
1. **Algorithmic generation**: Constraint-based solver (OR-Tools, custom backtracking)
2. **Manual override with validation**: Admin-assigned entries checked for conflicts
3. **Gradual rollout**: Generate timetable, allow conflict marking, resolve iteratively

---

## 5. TimetableException Domain Rules

### 5.1 Definition

**TimetableException** records deviations from the regular timetable for specific dates.

### 5.2 Domain Rules

1. **R-EX-001**: Exception types:
   - **CANCELLATION**: Regular class cancelled, no replacement
   - **SUBSTITUTION**: Different teacher or room for that day
   - **RESCHEDULE**: Moved to different time slot (original cancelled, new entry created)
   - **SPECIAL_EVENT**: Assembly, exam, field trip replaces regular class

2. **R-EX-002**: Exception date range:
   - Single date: most common (snow day, teacher absence)
   - Date range: exam periods, sports events
   - Cannot extend beyond timetable's effective_to_date

3. **R-EX-003**: Exception creation workflow:
   - Requires timetable_admin or higher role
   - Must reference affected TimetableEntry (or multiple via pattern)
   - Affected parties notified automatically:
     - Teacher notified of substitution/cancellation
     - Students notified of schedule change
     - Room booking system updated

4. **R-EX-004**: Exception visibility:
   - Current/future exceptions visible on all devices
   - Past exceptions retained for audit
   - Students see exceptions on their schedule view

**Fields**:
- `id` (UUID)
- `timetable_id` (UUID, foreign key to Timetable)
- `affected_timetable_entry_id` (UUID, foreign key to TimetableEntry)
- `exception_type` (VARCHAR[20]) — "CANCELLATION", "SUBSTITUTION", "RESCHEDULE", "SPECIAL_EVENT"

- `exception_date` (DATE) — single date
- `exception_start_date` (DATE) — for date ranges
- `exception_end_date` (DATE, nullable)

- `new_time_slot_id` (UUID, foreign key to TimeSlot) — for reschedule
- `new_teacher_id` (UUID) — for substitution
- `new_room_code` (VARCHAR[20]) — for room change

- `reason` (TEXT)
- `notified_users` (JSONB) — records of notification sent

- `created_by` (UUID)
- `created_at` (TIMESTAMPTZ)

- `is_active` (BOOLEAN)
- Audit trail

**Indexes**:
- `(timetable_id, exception_date)` — for checking exceptions on a date
- `(affected_timetable_entry_id, exception_date)` — for finding exceptions for an entry

---

## 6. Scheduling Workflows

### 6.1 Automatic Timetable Generation

```
Input:
- SchoolYearCategory with all ClassroomSubjects
- Teacher availability constraints
- Room availability
- Weekly hour requirements per subject

Algorithm (simplified):
1. Create list of all (ClassroomSubject, weekly_hours) pairs
2. For each ClassroomSubject:
   - Find available teacher (with competence)
   - Find available rooms (with equipment if needed)
   - Assign to time slots respecting:
        a. No teacher conflict
        b. No room conflict
        c. Student timetable cohesion (same classroom gets coherent schedule)
   - If constraint failure, backtrack or flag for manual resolution
3. Validate: all subjects meet weekly_hours, no conflicts exist
4. Create Timetable + TimetableEntries
5. Mark as DRAFT for admin review
```

**Conflict resolution techniques**:
- Priority-based (core subjects scheduled first)
- Room pooling (multiple classrooms can share flexible rooms)
- Teacher sharing (part-time teachers' schedules optimized)

### 6.2 Timetable Approval Process

```
1. Auto-generated timetable → DRAFT
2. Admin reviews conflicts list (if any)
3. Manual adjustments made
4. Teachers notified to review their schedule
5. Teachers can flag:
   - Conflicts they see
   - Unreasonable gaps between classes
   - Room inadequacies
6. Admin resolves flags
7. Principal approves
8. Timetable → ACTIVE
9. All devices sync new timetable
```

---

## 7. Offline-First Considerations

### 7.1 Timetable Distribution

**Critical**: Timetables MUST be cached on all devices before SchoolYear activation.

Distribution flow:
```
Server (ACTIVE timetable)
    ↓ push to all devices in SchoolYearCategory
Device cache (timetable + all entries)
    ↓ used for:
        - Teacher schedule view
        - Student timetable display
        - Attendance session (class subject lookup)
```

### 7.2 Offline View Limitations

**Allowed Offline**:
- View own teacher schedule
- View assigned classroom's schedule
- View student's daily timetable
- Check room assignments

**NOT Allowed Offline**:
- Create/modify Timetable (requires admin)
- Create TimetableExceptions (may conflict)
- Generate new Timetable (computationally intensive)

### 7.3 Sync Merge Strategy

Timetables use **last-write-wins** with version check:
- Each Timetable has version number
- Device sending modification must have latest version
- Conflict detected if:
  - Same timetable modified on two devices simultaneously
  - Resolution: most recent modification wins, other changes lost (unlikely due to role restrictions)

---

## 8. Query Patterns and Performance

### 8.1 Common Queries

| Query | Required Indexes |
|-------|------------------|
| Get teacher's schedule for week | `(teacher_id, timetable_id)` on TimetableEntry |
| Get classroom's schedule for date | `(classroom_subject_id, timetable_id)` + TimeSlot join |
| Check room availability for slot | `(time_slot_id, room_code)` on TimetableEntry |
| Find student's daily timetable | Enrollment → Classroom → TimetableEntry chain |
| List teacher conflicts | Self-join on TimetableEntry with overlapping time_slot |

### 8.2 Materialized Views

```sql
-- Teacher schedule view (refreshed on timetable change)
CREATE MATERIALIZED VIEW teacher_schedule AS
SELECT
    t.id as teacher_id,
    t.user_id,
    ts.day_of_week,
    ts.period_number,
    ts.start_time,
    ts.end_time,
    te.timetable_id,
    cs.classroom_id,
    c.code as classroom_code,
    sub.code as subject_code,
    sub.name as subject_name,
    te.room_code
FROM teacher_profile t
JOIN timetable_entry te ON t.id = te.teacher_id
JOIN time_slot ts ON te.time_slot_id = ts.id
JOIN classroom_subject cs ON te.classroom_subject_id = cs.id
JOIN classroom c ON cs.classroom_id = c.id
JOIN subject sub ON cs.subject_id = sub.id
WHERE te.state = 'ACTIVE'
  AND te.is_active = TRUE;

CREATE INDEX idx_teacher_schedule_teacher_day ON teacher_schedule(teacher_id, day_of_week, period_number);
```

---

## 9. Timetable Change Impact Analysis

When modifying a timetable (even minor change), system must compute impact:

**Affected entities**:
- All enrolled students (class schedules change)
- Assigned teachers (schedule changes)
- Room bookings (if room changed)
- Attendance sessions (if timeslot changes)
- Substitution requests already planned

**Impact report generated automatically**:
```
Change: Move Math 5ème A from Monday 1st to Monday 2nd
────────────────────────────────────────────────────────
Affected Students: 30
Affected Teachers: 1 (Prof. Diallo)
Affected Rooms: 1 (Room 102)
Attendance Sessions to Reschedule: 15 upcoming dates
Parent Notifications Required: 30
────────────────────────────────────────────────────────
[PROCEED] [CANCEL]
```

---

## 10. Invariants Summary

| Invariant | Enforced By | Consequence |
|-----------|-------------|-------------|
| No teacher double-booked | Unique constraint + conflict detection | Teacher unavailable error |
| All subjects meet weekly hours | Sum check on TimetableEntries | Incomplete curriculum |
| One active timetable per category | Partial unique index | Schedule ambiguity |
| Timetable not modified when ACTIVE without approval | State check + role requirement | Blocked modification |
| Exceptions within timetable validity | Date range check | Invalid exception |

---

## 11. Special Scheduling Patterns

### 11.1 Block Scheduling
Some schools have longer blocks (90-120 minutes) instead of 45-minute periods:
- Implemented as multiple consecutive TimeSlots linked conceptually
- Same classroom_subject scheduled in TimeSlot 1+2 as "block"

### 11.2 Rotating Schedules
A/B week pattern:
- TimeSlot rotation field indicates week type
- Student schedule: Monday Period 1 = Science (A), Math (B) alternating

### 11.3 Staggered Start Times
Different grade levels start at different times:
- Primary: 8:00 AM
- Middle: 8:30 AM
- High School: 9:00 AM
- Handled by having different TimeSlot sets per category (rare)

---

## 12. Performance Targets

- Teacher schedule load: < 200ms
- Student daily timetable: < 100ms
- Conflict detection for full timetable: < 5 seconds
- Timetable generation for 30 classrooms: < 60 seconds

---

## 13. Data Model Relationships

```
TimeSlot (1) ────< TimetableEntry (N) ────< Timetable (1)
                      │                              │
                      ├─── ClassroomSubject (1)      │
                      ├─── Teacher (1)               │
                      └─── Room (optional)           │
                                                    │
                       TimetableException (N) ───────┘
```

---

**Next Step**: TDR-007 — Data Architecture: Timetables Tables
