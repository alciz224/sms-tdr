# TDR-002 — Data Architecture: Master Tables Physical Design

## 1. Purpose and Scope

This document defines the physical data architecture for the Master/Reference tables (Étape 1). It specifies:

- Table structures with precise column definitions
- Primary keys, unique constraints, and foreign keys
- Database indexes for optimal query performance
- Denormalization strategy for offline-first operations
- Partitioning and archiving considerations for 10+ year data retention

**Constraint**: All designs MUST support offline-first operations with eventual consistency synchronization.

---

## 2. Geographic Hierarchy Tables

### 2.1 Region Table (region)

```sql
CREATE TABLE region (
    -- Primary Key
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Natural Business Key (immutable)
    code VARCHAR(2) NOT NULL UNIQUE,  -- ISO 3166-2 code

    -- Descriptive Fields
    name VARCHAR(100) NOT NULL,
    name_localized JSONB,  -- {"fr": "Dakar", "en": "Dakar", "wo": "Dakar"}

    -- Control Fields
    is_active BOOLEAN NOT NULL DEFAULT TRUE,

    -- Audit Trail (tracks all changes for 10+ years)
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by UUID NOT NULL REFERENCES customuser(id),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by UUID REFERENCES customuser(id),
    archived_at TIMESTAMPTZ,
    archived_by UUID REFERENCES customuser(id),

    -- Constraints
    CONSTRAINT region_code_check CHECK (code ~ '^[A-Z]{2}$'),
    CONSTRAINT region_name_not_empty CHECK (trim(name) != '')
);

-- Indexes
CREATE INDEX idx_region_code ON region(code);
CREATE INDEX idx_region_is_active ON region(is_active) WHERE is_active = TRUE;

-- Denormalized/Computed Fields: None needed for this small table
```

**Physical Design Notes**:
- Small table (~5-15 rows) - full table scans are acceptable
- `code` is immutable natural key; never changes even if name does
- `name_localized` supports multi-language regions

---

### 2.2 Prefecture Table (prefecture)

```sql
CREATE TABLE prefecture (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Natural Keys
    code VARCHAR(10) NOT NULL UNIQUE,  -- e.g., "DN-MED"
    name VARCHAR(100) NOT NULL,
    name_localized JSONB,

    -- Foreign Keys
    region_id UUID NOT NULL REFERENCES region(id) ON DELETE RESTRICT,

    -- Denormalized for query performance (no joins for common queries)
    region_code VARCHAR(2) NOT NULL,

    -- Control Fields
    has_schools BOOLEAN NOT NULL DEFAULT FALSE,  -- computed, updated by triggers
    is_active BOOLEAN NOT NULL DEFAULT TRUE,

    -- Audit Trail
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by UUID NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by UUID,
    archived_at TIMESTAMPTZ,
    archived_by UUID,

    -- Constraints
    CONSTRAINT prefecture_code_check CHECK (code ~ '^[A-Z]{2}-[A-Z0-9]{3}$'),
    CONSTRAINT prefecture_region_code_match CHECK (
        left(code, 2) = region_code
    ),

    -- Composite unique (name within region)
    UNIQUE (region_id, name)
);

-- Indexes
CREATE INDEX idx_prefecture_code ON prefecture(code);
CREATE INDEX idx_prefecture_region_id ON prefecture(region_id);
CREATE INDEX idx_prefecture_region_code ON prefecture(region_code);
CREATE INDEX idx_prefecture_is_active ON prefecture(is_active) WHERE is_active = TRUE;

-- Denormalization: region_code stored to avoid joins for common reports
-- Maintained by BEFORE INSERT/UPDATE trigger: NEW.region_code = (SELECT code FROM region WHERE id = NEW.region_id)
```

**Sync Considerations**:
- Prefecture changes cascade from Region changes (Region determines valid Prefecture codes)
- Offline devices cache full table; lookups are local only
- `has_schools` computed by counting School references (async batch job)

---

### 2.3 SousPrefecture Table (sousprefecture)

```sql
CREATE TABLE sousprefecture (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Natural Keys
    code VARCHAR(15) NOT NULL UNIQUE,  -- e.g., "DN-MED-YOFF"
    name VARCHAR(100) NOT NULL,
    name_localized JSONB,

    -- Foreign Keys
    prefecture_id UUID NOT NULL REFERENCES prefecture(id) ON DELETE RESTRICT,
    region_id UUID NOT NULL REFERENCES region(id) ON DELETE RESTRICT,  -- denormalized

    -- Denormalized for query performance
    prefecture_code VARCHAR(10) NOT NULL,
    region_code VARCHAR(2) NOT NULL,

    -- Control Fields
    is_active BOOLEAN NOT NULL DEFAULT TRUE,

    -- Audit Trail
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by UUID NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by UUID,
    archived_at TIMESTAMPTZ,
    archived_by UUID,

    -- Constraints
    CONSTRAINT sousprefecture_code_check CHECK (code ~ '^[A-Z]{2}-[A-Z0-9]{3}-[A-Z0-9]{3,5}$'),
    CONSTRAINT sousprefecture_prefecture_region_match CHECK (
        left(prefecture_code, 2) = region_code
    ),

    UNIQUE (prefecture_id, name)
);

-- Indexes
CREATE INDEX idx_sousprefecture_code ON sousprefecture(code);
CREATE INDEX idx_sousprefecture_prefecture_id ON sousprefecture(prefecture_id);
CREATE INDEX idx_sousprefecture_region_code ON sousprefecture(region_code);
CREATE INDEX idx_sousprefecture_is_active ON sousprefecture(is_active) WHERE is_active = TRUE;
```

**Denormalization Strategy**:
- `region_id` and `region_code` stored to enable Region-level queries without joining to Prefecture first
- This accommodates the 4-level hierarchy while keeping common queries performant

---

### 2.4 Quartier Table (quartier)

```sql
CREATE TABLE quartier (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Natural Keys
    code VARCHAR(20) NOT NULL UNIQUE,  -- e.g., "DN-MED-YOFF-BISCUITERIE"
    name VARCHAR(100) NOT NULL,
    name_localized JSONB,

    -- Foreign Keys (full hierarchy denormalized)
    sousprefecture_id UUID NOT NULL REFERENCES sousprefecture(id) ON DELETE RESTRICT,
    prefecture_id UUID NOT NULL REFERENCES prefecture(id) ON DELETE RESTRICT,
    region_id UUID NOT NULL REFERENCES region(id) ON DELETE RESTRICT,

    -- Denormalized codes at all levels
    sousprefecture_code VARCHAR(15) NOT NULL,
    prefecture_code VARCHAR(10) NOT NULL,
    region_code VARCHAR(2) NOT NULL,

    -- Control Fields
    is_active BOOLEAN NOT NULL DEFAULT TRUE,

    -- Audit Trail
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by UUID NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by UUID,
    archived_at TIMESTAMPTZ,
    archived_by UUID,

    -- Constraints
    CONSTRAINT quartier_code_check CHECK (code ~ '^[A-Z]{2}-[A-Z0-9]{3}-[A-Z0-9]{3,5}-[A-Z0-9-]{3,10}$'),

    UNIQUE (sousprefecture_id, name)
);

-- Indexes
CREATE INDEX idx_quartier_code ON quartier(code);
CREATE INDEX idx_quartier_sousprefecture_id ON quartier(sousprefecture_id);
CREATE INDEX idx_quartier_region_code ON quartier(region_code);
CREATE INDEX idx_quartier_is_active ON quartier(is_active) WHERE is_active = TRUE;

-- Composite index for student address lookups
CREATE INDEX idx_quartier_hierarchy ON quartier(region_code, prefecture_code, sousprefecture_code);
```

**Offline Considerations**:
- Quartier is the most granular geographic unit used for student addresses
- Full table must be cached on all devices (~1000-10000 rows depending on country size)
- Read-heavy pattern (select by code during student enrollment)
- Write operations (add new quartier) are admin-only and rare
- Denormalized parent codes enable reporting without joins

---

## 3. Academic Structure Tables

### 3.1 AcademicYearType Table (academic_year_type)

```sql
CREATE TABLE academic_year_type (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Natural Key
    code VARCHAR(20) NOT NULL UNIQUE,  -- "SEMESTER", "TRIMESTER", "QUARTER"
    name VARCHAR(100) NOT NULL,
    description TEXT,

    -- Configuration
    max_terms_per_year INTEGER NOT NULL CHECK (max_terms_per_year BETWEEN 2 AND 4),
    default_term_duration_days INTEGER,  -- guideline for term planning

    -- System State
    is_default BOOLEAN NOT NULL DEFAULT FALSE,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,

    -- Audit Trail
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by UUID NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by UUID,
    archived_at TIMESTAMPTZ,
    archived_by UUID,

    -- Constraints
    CONSTRAINT ay_type_default_single CHECK (
        (is_default = TRUE)::int + (SELECT COUNT(*) FROM academic_year_type WHERE is_default = TRUE AND is_active = TRUE) <= 1
    )
);

-- Indexes
CREATE INDEX idx_ay_type_code ON academic_year_type(code);
CREATE INDEX idx_ay_type_is_default ON academic_year_type(is_default) WHERE is_default = TRUE AND is_active = TRUE;
CREATE INDEX idx_ay_type_is_active ON academic_year_type(is_active) WHERE is_active = TRUE;
```

**Configuration Note**:
- `is_default` constraint ensures exactly one default active type via partial unique index
- System operation requires one default AcademicYearType at all times

---

### 3.2 TermType Table (term_type)

```sql
CREATE TABLE term_type (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Natural Key (scoped to AcademicYearType)
    code VARCHAR(10) NOT NULL,  -- e.g., "S1", "T1", "Q1"
    name VARCHAR(100) NOT NULL,

    -- Foreign Key
    academic_year_type_id UUID NOT NULL REFERENCES academic_year_type(id) ON DELETE RESTRICT,

    -- Denormalized for reporting
    academic_year_type_code VARCHAR(20) NOT NULL,

    -- Position within the academic year
    sequence INTEGER NOT NULL,  -- 1, 2, 3, 4...

    -- Display configuration
    short_label VARCHAR(20),  -- "1st Sem", "2nd Sem"
    display_color CHAR(7),  -- hex color "#FF5733" for UI

    -- Control Fields
    is_active BOOLEAN NOT NULL DEFAULT TRUE,

    -- Audit Trail
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by UUID NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by UUID,
    archived_at TIMESTAMPTZ,
    archived_by UUID,

    -- Constraints
    CONSTRAINT term_type_code_format CHECK (
        code ~ '^(S|T|Q)[1-4]$'
    ),

    -- Uniqueness: Code unique within AcademicYearType
    UNIQUE (academic_year_type_id, code),
    UNIQUE (academic_year_type_id, sequence)
);

-- Indexes
CREATE INDEX idx_term_type_code ON term_type(code);
CREATE INDEX idx_term_type_ayt_id ON term_type(academic_year_type_id);
CREATE INDEX idx_term_type_sequence ON term_type(academic_year_type_id, sequence);
CREATE INDEX idx_term_type_is_active ON term_type(is_active) WHERE is_active = TRUE;

-- Trigger to denormalize academic_year_type_code
-- AFTER INSERT/UPDATE: NEW.academic_year_type_code = (SELECT code FROM academic_year_type WHERE id = NEW.academic_year_type_id)
```

---

### 3.3 Term Table (term) - Instance Table per SchoolYear

```sql
CREATE TABLE term (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Foreign Keys
    school_year_id UUID NOT NULL REFERENCES school_year(id) ON DELETE CASCADE,
    term_type_id UUID NOT NULL REFERENCES term_type(id) ON DELETE RESTRICT,

    -- Denormalized snapshots (immutable once created)
    school_year_code VARCHAR(20) NOT NULL,
    term_type_code VARCHAR(10) NOT NULL,
    academic_year_type_code VARCHAR(20) NOT NULL,

    -- Descriptive Fields (copied from TermType at creation time)
    name VARCHAR(100) NOT NULL,
    short_label VARCHAR(20),

    -- Position (from TermType)
    sequence INTEGER NOT NULL,

    -- Date Range (set when SchoolYear is configured)
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,

    -- Computed Cached Values
    duration_days INTEGER GENERATED ALWAYS AS (
        (end_date - start_date) + 1
    ) STORED,

    -- Control Fields
    is_active BOOLEAN NOT NULL DEFAULT TRUE,

    -- Audit Trail (minimal - dates should not change after first evaluation)
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Constraints
    CONSTRAINT term_date_range_valid CHECK (end_date >= start_date),
    CONSTRAINT term_no_overlap EXCLUDE USING gist (
        school_year_id WITH =,
        daterange(start_date, end_date, '[]') WITH &&
    ) WHERE (is_active = TRUE),

    -- Uniqueness
    UNIQUE (school_year_id, term_type_id),
    UNIQUE (school_year_id, sequence)
);

-- Indexes
CREATE INDEX idx_term_school_year_id ON term(school_year_id);
CREATE INDEX idx_term_term_type_id ON term(term_type_id);
CREATE INDEX idx_term_school_year_sequence ON term(school_year_id, sequence);
CREATE INDEX idx_term_date_range ON term USING btree(daterange(start_date, end_date, '[]'));
CREATE INDEX idx_term_school_year_code ON term(school_year_code);
CREATE INDEX idx_term_is_active ON term(is_active) WHERE is_active = TRUE;

-- Partitioning Strategy: By school_year_code for long-term storage
-- Quarters 10+ years old can be moved to archive partitions
CREATE TABLE term_y2023 PARTITION OF term FOR VALUES IN ('2023-2024');
CREATE TABLE term_y2024 PARTITION OF term FOR VALUES IN ('2024-2025');
-- Future partitions created by automation

-- Trigger to denormalize all codes and prevent updates after evaluations exist
CREATE OR REPLACE FUNCTION term_prevent_date_update()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'UPDATE' AND
       (OLD.start_date != NEW.start_date OR OLD.end_date != NEW.end_date) THEN
        IF EXISTS (
            SELECT 1 FROM evaluation WHERE term_id = OLD.id LIMIT 1
        ) THEN
            RAISE EXCEPTION 'Cannot modify term dates after evaluations exist';
        END IF;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_term_prevent_date_update
    BEFORE UPDATE ON term
    FOR EACH ROW EXECUTE FUNCTION term_prevent_date_update();
```

**Partitioning Rationale**:
- Query pattern: almost always filter by `school_year_id` or `school_year_code`
- Partitioning allows easy archival of old school years
- Partition key uses denormalized `school_year_code` for efficient pruning

---

### 3.4 Category Table (category)

```sql
CREATE TABLE category (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Natural Key
    code VARCHAR(10) NOT NULL UNIQUE,  -- "PRIM", "COLL", "LYC", "UNIV", "PRO"
    name VARCHAR(100) NOT NULL,
    name_localized JSONB,
    description TEXT,

    -- Control Fields
    is_active BOOLEAN NOT NULL DEFAULT TRUE,

    -- Audit Trail
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by UUID NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by UUID,
    archived_at TIMESTAMPTZ,
    archived_by UUID,

    -- Constraints
    CONSTRAINT category_code_check CHECK (code ~ '^[A-Z0-9]{2,6}$')
);

CREATE INDEX idx_category_code ON category(code);
CREATE INDEX idx_category_is_active ON category(is_active) WHERE is_active = TRUE;
```

---

### 3.5 Level Table (level)

```sql
CREATE TABLE level (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Natural Keys
    code VARCHAR(20) NOT NULL,  -- e.g., "CP", "6E", "TERM"
    name VARCHAR(100) NOT NULL,
    name_localized JSONB,

    -- Foreign Keys
    category_id UUID NOT NULL REFERENCES category(id) ON DELETE RESTRICT,

    -- Denormalized for reporting
    category_code VARCHAR(10) NOT NULL,

    -- Position within Category
    sequence INTEGER NOT NULL,  -- order within category

    -- Demographics
    min_age INTEGER NOT NULL CHECK (min_age >= 3),
    max_age INTEGER NOT NULL CHECK (max_age >= min_age),

    -- Grading Configuration
    grading_scale_type VARCHAR(20) NOT NULL CHECK (
        grading_scale_type IN ('TWENTY', 'TEN', 'FIVE', 'PERCENTAGE', 'LETTER', 'CUSTOM')
    ),
    min_passing_grade DECIMAL(5,2),  -- scale-dependent
    max_grade DECIMAL(5,2) NOT NULL DEFAULT 20.00,

    -- Control Fields
    is_active BOOLEAN NOT NULL DEFAULT TRUE,

    -- Audit Trail
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by UUID NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by UUID,
    archived_at TIMESTAMPTZ,
    archived_by UUID,

    -- Constraints
    CONSTRAINT level_code_category_unique UNIQUE (category_id, code),
    CONSTRAINT level_sequence_unique UNIQUE (category_id, sequence),
    CONSTRAINT level_age_valid CHECK (max_age - min_age <= 5),  -- levels shouldn't span > 5 years
    CONSTRAINT level_grade_min_max CHECK (min_passing_grade > 0 AND min_passing_grade <= max_grade)
);

CREATE INDEX idx_level_code ON level(code);
CREATE INDEX idx_level_category_id ON level(category_id);
CREATE INDEX idx_level_category_code ON level(category_code);
CREATE INDEX idx_level_sequence ON level(category_id, sequence);
CREATE INDEX idx_level_is_active ON level(is_active) WHERE is_active = TRUE;

-- Trigger to denormalize category_code
```

---

### 3.6 Track Table (track)

```sql
CREATE TABLE track (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Natural Keys
    code VARCHAR(20) NOT NULL,
    name VARCHAR(100) NOT NULL,
    name_localized JSONB,
    description TEXT,

    -- Foreign Keys
    level_id UUID NOT NULL REFERENCES level(id) ON DELETE RESTRICT,

    -- Denormalized
    level_code VARCHAR(20) NOT NULL,
    category_id UUID NOT NULL REFERENCES category(id) ON DELETE RESTRICT,
    category_code VARCHAR(10) NOT NULL,

    -- Configuration
    is_mandatory BOOLEAN NOT NULL DEFAULT FALSE,
    is_default BOOLEAN NOT NULL DEFAULT FALSE,  -- default track for this level

    -- Display
    display_order INTEGER NOT NULL DEFAULT 0,

    -- Control Fields
    is_active BOOLEAN NOT NULL DEFAULT TRUE,

    -- Audit Trail
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by UUID NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by UUID,
    archived_at TIMESTAMPTZ,
    archived_by UUID,

    -- Constraints
    CONSTRAINT track_code_level_unique UNIQUE (level_id, code),
    CONSTRAINT track_default_unique UNIQUE (level_id) WHERE is_default = TRUE,

    -- Only one default track per level
    CONSTRAINT track_one_default_per_level CHECK (
        (SELECT COUNT(*) FROM track WHERE level_id = NEW.level_id AND is_default = TRUE AND is_active = TRUE) <= 1
    )
);

CREATE INDEX idx_track_code ON track(code);
CREATE INDEX idx_track_level_id ON track(level_id);
CREATE INDEX idx_track_level_code ON track(level_code);
CREATE INDEX idx_track_is_default ON track(is_default) WHERE is_default = TRUE;
CREATE INDEX idx_track_is_active ON track(is_active) WHERE is_active = TRUE;

-- Multiple indexes to support different query patterns:
-- - By level (for track selection during enrollment)
-- - By active state (lookup by students)
-- - By default (find default track for a level)
```

---

### 3.7 Subject Table (subject)

```sql
CREATE TABLE subject (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Natural Key
    code VARCHAR(20) NOT NULL UNIQUE,  -- "MAT", "PHY", "CHE", "FRA", "ENG", "HIS"
    name VARCHAR(100) NOT NULL,
    name_localized JSONB,

    -- Classification
    category VARCHAR(20) NOT NULL CHECK (
        category IN ('SCIENCE', 'HUMANITIES', 'ARTS', 'TECHNICAL', 'PHYSICAL', 'LANGUAGE', 'CORE')
    ),

    -- Default Configuration
    default_weekly_hours INTEGER NOT NULL DEFAULT 0,
    min_coefficient DECIMAL(4,2) NOT NULL DEFAULT 1.00 CHECK (min_coefficient >= 0.5),
    max_coefficient DECIMAL(4,2) NOT NULL DEFAULT 10.00 CHECK (max_coefficient >= min_coefficient),

    -- Practical/Lab requirement
    has_practical BOOLEAN NOT NULL DEFAULT FALSE,
    practical_hours_per_week INTEGER DEFAULT 0,

    -- Grading (can be overridden at Level/Track via SubjectChoice)
    default_min_passing_grade DECIMAL(5,2),

    -- Curriculum metadata
    curriculum_code VARCHAR(50),  -- reference to official curriculum document
    is_core BOOLEAN NOT NULL DEFAULT FALSE,  -- Mathematics, Language, Sciences are usually core

    -- Control Fields
    is_active BOOLEAN NOT NULL DEFAULT TRUE,

    -- Audit Trail
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by UUID NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by UUID,
    archived_at TIMESTAMPTZ,
    archived_by UUID,

    -- Constraints
    CONSTRAINT subject_practical_hours_valid CHECK (
        has_practical = FALSE OR practical_hours_per_week > 0
    )
);

CREATE INDEX idx_subject_code ON subject(code);
CREATE INDEX idx_subject_category ON subject(category);
CREATE INDEX idx_subject_is_core ON subject(is_core) WHERE is_core = TRUE;
CREATE INDEX idx_subject_is_active ON subject(is_active) WHERE is_active = TRUE;
```

---

### 3.8 SubjectChoice Table (subject_choice)

```sql
CREATE TABLE subject_choice (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Composite Foreign Keys (forming the template)
    level_id UUID NOT NULL REFERENCES level(id) ON DELETE CASCADE,
    track_id UUID REFERENCES track(id) ON DELETE CASCADE,  -- NULL = Level-wide default

    -- Dependent on Track/Level configuration
    subject_id UUID NOT NULL REFERENCES subject(id) ON DELETE RESTRICT,

    -- Configuration overrides (from Subject defaults)
    is_required BOOLEAN NOT NULL DEFAULT TRUE,
    is_elective BOOLEAN NOT NULL DEFAULT FALSE,  -- can be swapped with another elective
    min_coefficient DECIMAL(4,2) NOT NULL,
    max_coefficient DECIMAL(4,2) NOT NULL,
    weekly_hours INTEGER NOT NULL,

    -- Display order (for report cards, timetables)
    display_order INTEGER NOT NULL DEFAULT 0,

    -- Control Fields
    is_active BOOLEAN NOT NULL DEFAULT TRUE,

    -- Audit Trail
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by UUID NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by UUID,
    archived_at TIMESTAMPTZ,
    archived_by UUID,

    -- Constraints
    CONSTRAINT subject_choice_coefficient_range CHECK (
        min_coefficient <= max_coefficient AND
        min_coefficient >= (SELECT min_coefficient FROM subject WHERE id = subject_choice.subject_id) AND
        max_coefficient <= (SELECT max_coefficient FROM subject WHERE id = subject_choice.subject_id)
    ),
    CONSTRAINT subject_choice_weekly_hours_positive CHECK (weekly_hours > 0),

    -- Uniqueness: a subject appears at most once per (level, track) combination
    -- Note: track_id can be NULL, so (level_id, NULL, subject_id) is valid
    UNIQUE (level_id, track_id, subject_id)
);

CREATE INDEX idx_subject_choice_level_id ON subject_choice(level_id);
CREATE INDEX idx_subject_choice_track_id ON subject_choice(track_id);
CREATE INDEX idx_subject_choice_subject_id ON subject_choice(subject_id);
CREATE INDEX idx_subject_choice_is_required ON subject_choice(is_required) WHERE is_required = TRUE;
CREATE INDEX idx_subject_choice_active ON subject_choice(is_active) WHERE is_active = TRUE;

-- Composite index for enrollment queries: given a (level, track), find all required subjects
CREATE INDEX idx_subject_choice_level_track_active ON subject_choice(level_id, track_id, is_active);

-- Trigger: validate weekly_hours doesn't exceed category's weekly limit (via level)
-- Trigger: copy subject defaults on insert if not specified
```

**Business Rule Enforcement**:
SubjectChoice represents the curriculum template. When a student enrolls, the system:
1. Looks up the student's Track for that SchoolYearCategory
2. Fetches all SubjectChoice records for (Level, Track)
3. Creates ClassroomSubject records with those coefficients and hours

---

## 4. Cross-Cutting Audit Pattern

All referential tables inherit this audit pattern:

```sql
-- Common audit columns explanation:
created_at     -- When the record was first created (immutable)
created_by     -- Admin/system that created it (immutable)
updated_at     -- Last modification timestamp (updated on every change)
updated_by     -- Who made the last change
archived_at    -- Soft delete timestamp (set instead of DELETE)
archived_by    -- Who archived it
is_active       -- Current active state; false = archived
```

**Audit Requirement**: Every change to reference data must be auditable for 10+ years. Implement through triggers that write to an audit log table:

```sql
CREATE TABLE referential_audit_log (
    id BIGSERIAL PRIMARY KEY,
    table_name VARCHAR(50) NOT NULL,
    record_id UUID NOT NULL,
    operation VARCHAR(10) NOT NULL,  -- INSERT, UPDATE, ARCHIVE
    old_values JSONB,
    new_values JSONB,
    changed_by UUID NOT NULL,
    changed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    change_reason TEXT  -- mandatory reason for updates/archives
);

CREATE INDEX idx_referential_audit_log_record ON referential_audit_log(table_name, record_id);
CREATE INDEX idx_referential_audit_log_changed_at ON referential_audit_log(changed_at);
```

---

## 5. Sync Strategy for Master Tables

### 5.1 Offline-First Sync Pattern

```
┌─────────────────┐                            ┌─────────────────┐
│   Offline Device│                            │   Central Server│
└─────────────────┘                            └─────────────────┘
           │                                            │
           │ 1. Request: "sync_reference_data_since(version)" │
           │─────────────────────────────────────────────►│
           │                                            │
           │                            │ 2. Query changes since version
           │                            │    (incremental changes only)
           │                            │
           │                            │ 3. Return: [changed_records], new_version
           │◄─────────────────────────────────────────────│
           │                                            │
           │ 4. Upsert records (insert/update)         │
           │    - New records: INSERT                 │
           │    - Modified: UPDATE (preserve local edits to unrelated fields)
           │    - Archived: soft-delete locally      │
           │                                            │
           │ 5. Update local reference_data_version  │
```

### 5.2 Reference Data Version Tracking

```sql
-- Device stores locally:
CREATE TABLE device_sync_state (
    device_id UUID PRIMARY KEY,
    reference_data_version BIGINT NOT NULL DEFAULT 0,
    last_sync_at TIMESTAMPTZ,
    sync_in_progress BOOLEAN DEFAULT FALSE
);

-- Server tracks all reference data changes:
CREATE TABLE reference_data_version (
    id BIGSERIAL PRIMARY KEY,
    version BIGINT NOT NULL UNIQUE,
    changed_tables TEXT[] NOT NULL,  -- which tables had changes
    changed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    changed_by UUID,
    change_summary TEXT
);

-- Incremental sync query (server-side):
-- SELECT * FROM region WHERE updated_at > (SELECT reference_data_version FROM device_sync_state WHERE device_id = $1)
-- UNION ALL
-- SELECT * FROM prefecture WHERE updated_at > ...
```

### 5.3 Offline Constraints

1. **No Local Creation**: Offline devices CANNOT create new reference records. All reads only.
2. **Cache Required**: All transactional operations verify reference data is cached.
3. **Version Mismatch**: If device's reference_data_version < server's, block non-reference operations until sync.
4. **Conflict Resolution**: Server always wins for reference data (authoritative source).
5. **Compression**: Reference data compressed for mobile sync (typical size: <5MB total).

---

## 6. Denormalization Summary

| Table | Denormalized Columns | Purpose | Maintenance |
|-------|---------------------|---------|-------------|
| prefecture | region_code | Region queries without join | Trigger on change |
| sousprefecture | prefecture_code, region_id, region_code | Hierarchy traversal | Trigger on change |
| quartier | prefecture_id, prefecture_code, region_id, region_code | Multi-level reporting | Trigger on change |
| term | school_year_code, term_type_code, academic_year_type_code | Reporting/display | Generation script |
| level | category_code | Category queries | Trigger |
| track | level_code, category_id, category_code | Enrollment lookups | Trigger |

**Trade-off**: Denormalization adds write complexity (triggers) but enables:
- Offline queries without joins (reduced data transfer)
- Partition pruning on denormalized keys
- Faster reporting queries (no cascade joins)

---

## 7. Partitioning Strategy for Long-Term Storage

### 7.1 Time-Based Partitioning

For tables with heavy time-based queries and historical retention:

```sql
-- Term table: partitioned by school_year_code
-- Pattern: term_yYYYY where YYYY = start year of school year (e.g., "2024" for "2024-2025")

-- Retention policy: Keep 10 years online, archive older to cold storage
-- Implementation: Create partitions annually via cron job:
--   CREATE TABLE term_2025 PARTITION OF term FOR VALUES IN ('2025-2026');

-- Archival procedure (run annually):
--   CREATE TABLE term_archive_y2014 PARTITION OF term_archive FOR VALUES IN ('2013-2014');
--   DETACH PARTITION term_y2013 FROM term;
--   ATTACH PARTITION term_y2013 TO term_archive;
```

### 7.2 Archive Tables (Cold Storage)

```sql
-- Separate schema for 10+ year old data
CREATE SCHEMA archive;

CREATE TABLE archive.term_y2013 (
    -- identical structure to term
    -- including all indexes
    -- but in tablespace 'archive_tsd'
) PARTITION OF term FOR VALUES IN ('2013-2014');
```

---

## 8. Migration Considerations

When schema changes are required after system deployment:

1. **Additive migrations only** (new columns with NULL/d defaults)
2. **No column type changes** (add new column, copy data, switch application, drop old)
3. **Backfill scripts** must be offline-safe (batch updates, checkpoints)
4. **Downgrade path** must exist (reversible migrations)
5. **Version tracking** in code: migrate_YYYYMMDD_XXX.sql naming convention

---

## 9. Index Selection Rationale

| Index Type | Use Case | Tables |
|------------|----------|--------|
| Single column | Equality lookups by code | All tables |
| Composite (level, track) | SubjectChoice lookup during enrollment | subject_choice |
| Partial (is_active) | Active record lookups (most queries) | All |
| GIST (daterange) | Term overlap prevention | term |
| Denormalized column | Reporting without joins | prefecture, sousprefecture, quartier |

**Guideline**: Every foreign key should have an index on the referencing column.

---

## 10. Performance Targets

- Reference lookup by code: < 10ms (indexed)
- Geographic hierarchy traversal: < 50ms (denormalized)
- SubjectChoice retrieval for enrollment: < 25ms (composite index)
- Full reference sync to device: < 5MB compressed, < 30s over 3G

---

## 11. Relationships Diagram (Foreign Key Map)

```
region (1) ────< prefecture (1) ────< sousprefecture (1) ────< quartier
   │                │                     │
   │                │                     │
   └────────────────┴─────────────────────┘ (denormalized paths)

academic_year_type (1) ────< term_type (N)
                         │
                         │ (instantiated per school_year)
                         ▼
                       term (N per school year)

category (1) ────< level (N) ────< track (N)
   │                │                │
   └────────────────┴────────────────┘ (academic hierarchy)
                         │
                         ▼
                 subject_choice (N)
                         │
                         ▼
                         subject (1)

level ────────────── grading_scale_type ──> (determines Grade table config)
```

---

**Next Step**: TDR-003 — Domain Rules: Core School Structure (SchoolYear, SchoolYearCategory, SchoolYearCategoryTerm, Classroom, Enrollment)
