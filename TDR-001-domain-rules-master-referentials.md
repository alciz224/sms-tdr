# TDR-001 — Domain Rules: Master/Reference Tables (Referentials)

## 1. Purpose and Scope

This document defines the immutable domain rules governing the Master/Reference data layer (Étape 1). These tables constitute the foundation of the entire School Management System and establish the permanent structural elements that other domains reference.

**Invariant Principle**: Reference data is authoritative, relatively static, and MUST maintain referential integrity across all 10+ years of system operation.

---

## 2. Geographic Hierarchies

### 2.1 Region (Région Administrative)

**Definition**: First-level administrative division of the country.

**Domain Rules**:
1. **R-REG-001**: A Region code MUST be a 2-character ISO 3166-2 code (e.g., "DN" for Dakar, "TH" for Thiès)
2. **R-REG-002**: Region name is unique within the country
3. **R-REG-003**: A Region CANNOT be deleted if it has associated Prefectures
4. **R-REG-004**: Region creation is restricted to system administrators only
5. **R-REG-005**: Region modifications (name changes only) require administrative approval and audit logging

**Fields**:
- `id` (UUID, primary key)
- `code` (string[2], unique, not null)
- `name` (string[100], not null)
- `is_active` (boolean, default true)
- Created audit fields

### 2.2 Prefecture (Préfecture/Département)

**Definition**: Second-level administrative division within a Region.

**Domain Rules**:
1. **R-PREF-001**: A Prefecture MUST belong to exactly one Region
2. **R-PREF-002**: Prefecture code format: `[REGION_CODE]-[3CHAR]` (e.g., "DN-MED" for Médina)
3. **R-PREF-003**: Prefecture name MUST be unique within its parent Region
4. **R-PREF-004**: Deleting a Prefecture cascades or blocks based on `has_schools` flag:
   - If `has_schools = true`: Prevent deletion, require archival
   - If `has_schools = false`: Allow deletion with audit trail

**Fields**:
- `id` (UUID)
- `code` (string[10], unique)
- `name` (string[100])
- `region_id` (UUID, foreign key to Region)
- `has_schools` (boolean, computed/denormalized)
- `is_active`

### 2.3 SousPrefecture (Sous-préfecture/Arrondissement)

**Definition**: Third-level administrative division within a Prefecture.

**Domain Rules**:
1. **R-SP-001**: A SousPrefecture MUST belong to exactly one Prefecture
2. **R-SP-002**: SousPrefecture code format: `[PREF_CODE]-[3CHAR]`
3. **R-SP-003**: Name uniqueness within parent Prefecture
4. **R-SP-004**: Must validate that the Region hierarchy is consistent (through Prefecture)

**Fields**:
- `id` (UUID)
- `code` (string[15], unique)
- `name` (string[100])
- `prefecture_id` (UUID, foreign key to Prefecture)
- `is_active`

### 2.4 Quartier (Neighborhood/District)

**Definition**: Local subdivision within a SousPrefecture, used for precise school location and student residence mapping.

**Domain Rules**:
1. **R-Q-001**: A Quartier belongs to exactly one SousPrefecture
2. **R-Q-002**: Quartier code: `[SP_CODE]-[3-5CHAR]`
3. **R-Q-003**: Name may repeat across different SousPrefectures but MUST be unique within a SousPrefecture
4. **R-Q-004**: Quartier is the finest geographic granularity needed for:
   - Student registration address
   - School catchment area definitions
   - Demographics reporting

**Fields**:
- `id` (UUID)
- `code` (string[20], unique)
- `name` (string[100])
- `sousprefecture_id` (UUID, foreign key to SousPrefecture)
- `is_active`

**Offline Consideration**: Quartier codes MUST be pre-downloaded to devices. No offline creation allowed (only referencing existing Quartiers).

---

## 3. Academic Structure Referentials

### 3.1 AcademicYearType

**Definition**: Classification of academic year systems used by the country or educational system.

**Domain Rules**:
1. **R-AYT-001**: At minimum, the system MUST support:
   - "SEMESTER" (2 semesters per year)
   - "TRIMESTER" (3 trimesters per year)
   - "QUARTER" (4 quarters per year)
2. **R-AYT-002**: The AcademicYearType defines the maximum number of SchoolYearCategoryTerm entries per SchoolYear
3. **R-AYT-003**: Changes to AcademicYearType require global system freeze (affects all evaluations)
4. **R-AYT-004**: AcademicYearType is system-wide; all schools in the system use the same type
5. **R-AYT-005**: Custom AcademicYearType creation is forbidden

**Fields**:
- `id` (UUID)
- `code` (string[20], unique) — e.g., "SEMESTER", "TRIMESTER"
- `name` (string[100])
- `max_terms_per_year` (integer, 2-4)
- `is_default` (boolean)
- `is_active`

### 3.2 TermType

**Definition**: Standard term names or labels used within an AcademicYearType.

**Domain Rules**:
1. **R-TT-001**: TermType codes MUST be standardized:
   - For SEMESTER: "S1", "S2"
   - For TRIMESTER: "T1", "T2", "T3"
   - For QUARTER: "Q1", "Q2", "Q3", "Q4"
2. **R-TT-002**: TermType `sequence` determines evaluation order (1, 2, 3...)
3. **R-TT-003**: TermType is independent of specific SchoolYear; it's a template
4. **R-TT-004**: Each AcademicYearType has its own set of TermTypes (1:N relationship)

**Fields**:
- `id` (UUID)
- `code` (string[10], unique)
- `name` (string[100])
- `academic_year_type_id` (UUID, foreign key to AcademicYearType)
- `sequence` (integer, unique within AcademicYearType)
- `is_active`

### 3.3 Term

**Definition**: Concrete instance of a Term within a specific SchoolYear.

**Critical Distinction**: Term is NOT a referential table in the traditional sense — it's specific to a SchoolYear but MUST be generated from TermType templates.

**Domain Rules (applied during SchoolYear creation)**:
1. **R-T-001**: When a SchoolYear is created, Term records are automatically generated for each TermType of the chosen AcademicYearType
2. **R-T-002**: Term `start_date` and `end_date` are specific to the SchoolYear instance
3. **R-T-003**: Term dates from consecutive SchoolYears MUST NOT overlap
4. **R-T-004**: Each Term within a SchoolYear MUST have a unique `sequence` number (inherited from TermType)
5. **R-T-005**: Term dates are immutable once any evaluation is created for that Term

**Fields**:
- `id` (UUID)
- `school_year_id` (UUID, foreign key to SchoolYear)
- `term_type_id` (UUID, foreign key to TermType)
- `name` (string[100]) — copy of TermType.name for this SchoolYear
- `code` (string[10]) — copy of TermType.code
- `sequence` (integer)
- `start_date` (date, not null)
- `end_date` (date, not null)
- `is_active`

**Business Rationale**: Separating Term (instance) from TermType (template) allows:
- Different date ranges each year while maintaining consistent naming
- Historical grade retention if SchoolYear dates shift
- Term-specific reporting across multiple years

### 3.4 Category (Educational Cycle)

**Definition**: Broad educational stage (Primaire, Collège, Lycée, etc.)

**Domain Rules**:
1. **R-CAT-001**: Category represents government-defined educational cycles
2. **R-CAT-002**: Standard categories (must exist):
   - "Maternelle" / "Preschool"
   - "Primaire" / "Primary"
   - "Collège" / "Middle School"
   - "Lycée" / "High School"
   - "Université" / "University"
   - "Formation Professionnelle" / "Vocational"
3. **R-CAT-003**: Category codes MUST be unique and follow format: `[2CHAR_CODE]`
4. **R-CAT-004**: Categories can be country-specific but must be created during system initialization
5. **R-CAT-005**: Category deletion is forbidden if any Level or SchoolYearCategory references it

**Fields**:
- `id` (UUID)
- `code` (string[10], unique)
- `name` (string[100])
- `description` (text, optional)
- `is_active`

### 3.5 Level (Grade/Class within Category)

**Definition**: Specific grade or class level within a Category.

**Domain Rules**:
1. **R-LEVEL-001**: A Level MUST belong to exactly one Category
2. **R-LEVEL-002**: Level sequence within a Category is defined by `order` field (1, 2, 3...)
3. **R-LEVEL-003**: Common Level codes and naming:
   - Primary: "CP", "CE1", "CE2", "CM1", "CM2" (FR) OR "Grade 1-5" (US)
   - Collège: "6e", "5e", "4e", "3e"
   - Lycée: "2nde", "1ère", "Terminale" OR "9-12"
4. **R-LEVEL-004**: Level codes MUST be unique within the Category but CAN repeat across Categories (e.g., "Seconde" in both Lycée and different country systems)
5. **R-LEVEL-005**: Level defines the minimum and maximum age range for student enrollment
6. **R-LEVEL-006**: Level determines:
   - Available Tracks
   - Mandatory Subjects
   - Evaluation scale (some levels use 0-20, others use A-F, etc.)

**Fields**:
- `id` (UUID)
- `code` (string[20])
- `name` (string[100])
- `category_id` (UUID, foreign key to Category)
- `order` (integer, unique within Category)
- `min_age` (integer)
- `max_age` (integer)
- `grading_scale_type` (string) — "TWENTY", "LETTER", "PERCENTAGE"
- `is_active`

### 3.6 Track (Option/Série/Branch)

**Definition**: Specialization pathway within a Level (e.g., Scientifique, Littéraire, Techniques).

**Domain Rules**:
1. **R-TRACK-001**: A Track belongs to exactly one Level
2. **R-TRACK-002**: Track `is_mandatory` flag indicates if all students in the Level must follow it
3. **R-TRACK-003**: Track determines:
   - Which subjects are required (via SubjectChoice)
   - Coefficient weights for each subject
   - Career pathways (for counseling)
4. **R-TRACK-004**: A student enrolls in exactly one Track per SchoolYearCategory
5. **R-TRACK-005**: NULL Track (no specialization) is valid for levels without tracking

**Fields**:
- `id` (UUID)
- `code` (string[20])
- `name` (string[100])
- `level_id` (UUID, foreign key to Level)
- `description` (text)
- `is_mandatory` (boolean)
- `order` (integer, for display sequence)
- `is_active`

### 3.7 Subject

**Definition**: Academic discipline or course subject taught in the school.

**Domain Rules**:
1. **R-SUBJ-001**: Subject represents curriculum-aligned academic disciplines:
   - Core: Mathematics, Physics, Chemistry, Biology, French/English/Language Arts, History, Geography, Civic Education, PE
   - Electives: Arts, Music, Computer Science, Foreign Languages
   - Technical: Workshop, Typing, Agricultural Science
2. **R-SUBJ-002**: Subject codes follow national curriculum standards (e.g., "MAT" for Mathematics)
3. **R-SUBJ-003**: Subject defines:
   - Default weekly hours per Level
   - Maximum coefficient range
   - Minimum passing grade (varies by Category)
   - Whether practical/lab sessions apply
4. **R-SUBJ-004**: A Subject CANNOT be deleted if referenced by:
   - ClassroomSubject
   - EvaluationSubject
   - GradeType
5. **R-SUBJ-005**: Subject categorization (for reporting):
   - Required vs Optional (determined per Level)
   - Major vs Minor (based on coefficient)
   - Field grouping (Sciences, Humanities, Arts, etc.)

**Fields**:
- `id` (UUID)
- `code` (string[20], unique)
- `name` (string[100])
- `category` (string) — "SCIENCE", "HUMANITIES", "ARTS", "TECHNICAL", "PHYSICAL"
- `default_hours_per_week` (integer, varies by Level)
- `min_coefficient` (decimal)
- `max_coefficient` (decimal)
- `has_practical` (boolean)
- `min_passing_grade` (decimal, system-dependent)
- `is_active`

### 3.8 SubjectChoice

**Definition**: Junction table defining which Subjects are available/required for a specific Track at a specific Level.

**Critical Purpose**: SubjectChoice is the configuration that determines the curriculum for each enrollment.

**Domain Rules**:
1. **R-SC-001**: SubjectChoice defines the School-agnostic Subject availability pattern:
   - All "2nde S" (Scientific Track) students take Mathematics, Physics, Chemistry, Biology as required
   - All "Terminale L" (Literary) students take Philosophy, Literature as required
2. **R-SC-002**: A (Level, Track, Subject) combination MUST be unique
3. **R-SC-003**: `is_required` flag determines:
   - If `true`: Student MUST be enrolled in this Subject (part of compulsory curriculum)
   - If `false`: Student MAY optionally choose this Subject
4. **R-SC-004**: `is_elective` allows students to replace one elective with another
5. **R-SC-005**: A Track with `is_mandatory = true` inherits all Subjects from:
   - Its parent Level's "default track" subjects
   - Plus any Track-specific subjects
6. **R-SC-006**: Default coefficient range comes from Subject, but can be constrained by Category here
7. **R-SC-007**: SubjectChoice is the template; when a student enrolls, ClassroomSubject records are created based on this

**Fields**:
- `id` (UUID)
- `level_id` (UUID, foreign key to Level)
- `track_id` (UUID, foreign key to Track, nullable for Level-default)
- `subject_id` (UUID, foreign key to Subject)
- `is_required` (boolean)
- `is_elective` (boolean)
- `min_coefficient` (decimal)
- `max_coefficient` (decimal)
- `weekly_hours` (integer)
- `order` (integer, for timetable and report card sequence)

**Unique Constraints**:
- `(level_id, track_id, subject_id)` must be unique
- Track can be NULL representing Level-default subjects

---

## 4. Referential Data Constraints Summary

### 4.1 Invariants That Must NEVER Be Violated

1. **I-REF-001 (Geographic Integrity)**: The geographic hierarchy (Region → Prefecture → SousPrefecture → Quartier) must always form a valid tree structure with no cycles

2. **I-REF-002 (Academic Tree Integrity)**: AcademicYearType → TermType → Term chain must maintain consistency:
   - Term count per SchoolYear == count of TermTypes for the AcademicYearType
   - Term dates must not overlap within a SchoolYear
   - Term.end_date of term N must be < Term.start_date of term N+1

3. **I-REF-003 (Subject Hierarchy Stability)**: Once a SubjectChoice is published for a Track/Level, changing SubjectChoice affects future enrollments only; historical ClassroomSubject records retain their configuration

4. **I-REF-004 (No Orphaned References)**: Every foreign key reference to a referential table must point to an `is_active = true` record

5. **I-REF-005 (Offline Cache Coherence)**: All offline devices must cache the complete referential data set before any transactional operations. Missing referentials = offline operation blocked.

### 4.2 Operations That Modify Referentials

These operations require elevated permissions and audit trails:

- **Create**: Only system administrators with country-level education authority credentials
- **Update**: Restricted changes only (name spelling fixes). Code, structure changes require approval workflow.
- **Archive/Soft-Delete**: Set `is_active = false` and prevent deletion if foreign key dependencies exist
- **Reactive**: Allowed only for archival reversals within 90 days

### 4.3 Sync Strategy for Reference Data

```
Reference Data Sync Rules:
- Push-only from Central Server to Devices
- Devices NEVER create or modify reference data
- Conflict resolution: Server always wins
- Versioning: Every change increments global reference_data_version
- Devices cache reference_data_version and must download full dataset if version mismatch
```

---

## 5. Cross-Referential Business Rules

1. **Level-to-Subjects Mapping**:
   - For any (Level, Track) combination, there must exist at least one required SubjectChoice
   - SubjectChoice coefficient ranges must align with Subject's min/max

2. **Geographic-School Relationship**:
   - A School must belong to exactly one Quartier
   - Quartier → SousPrefecture → Prefecture → Region must form a valid path
   - All geographic lookups denormalize Region/Prefecture codes to School for performance

3. **AcademicYearTerm Consistency**:
   - AcademicYearType determines valid TermType codes
   - When TermType is added, all existing SchoolYears remain valid (retroactive compatibility required)
   - TermType deletion is forbidden if any Term references it

---

## 6. Data Validation Rules

These rules MUST be enforced at data entry (both server and offline with pre-cached referentials):

1. Code format validation (regex per entity type)
2. Uniqueness validation across parent scopes
3. Active record requirement for foreign keys
4. Sequence validation (Level.order, TermType.sequence)
5. Date range validation for Terms (start < end, non-overlapping)

---

## 7. Performance Considerations (10+ Year Lifespan)

1. Add `code` fields as immutable identifiers (never change even if name changes)
2. Use `is_active` soft-deletion instead of DELETE to maintain historical integrity
3. Index all foreign key columns used in joins
4. Denormalize commonly accessed parent data (e.g., store Region.code on Prefecture for reporting queries)
5. Plan for schema evolution: use versioned referential data tables if structural changes are anticipated

---

## 8. Dependencies on Master/Reference Tables

All subsequent tables in the TDR will depend on these referentials:

- **School** → Quartier + Code + Active
- **SchoolYear** → AcademicYearType
- **SchoolYearCategory** → Category + Level
- **Classroom** → SchoolYearCategory + Track
- **Enrollment** → Classroom + StudentProfile
- **ClassroomSubject** → Classroom (implicitly through SubjectChoice template)
- **Evaluation** → Term
- **Grade** → Evaluation + GradeType (derived from Level's grading scale)

---

**Next Step**: Proceed to TDR-002 — Data Architecture: Master Tables Physical Design (indexes, constraints, denormalization strategy)
