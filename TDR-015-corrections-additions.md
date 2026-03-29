# TDR-015 — Corrections & Additions: AcademicYear & Per-Category TermTypes

## 1. Purpose and Scope

This document captures **critical corrections** to the initial TDR design, identified during review:

1. **Missing AcademicYear table** — Need a named academic year independent of SchoolYear
2. **SchoolYearCategory should have independent TermType** — Different cycles (Primary vs Lycée) can have different term structures within the same calendar SchoolYear

These corrections address fundamental architectural gaps that would have caused implementation problems.

---

## 2. Issue 1: Missing AcademicYear Table

### 2.1 Problem Statement

The initial design used `SchoolYear` as both:
- The calendar-bound instance (start_date, end_date)
- The named academic year reference

But districts and ministries reference **Academic Year** separately from calendar boundaries:
- "Academic Year 2024-2025" is a named entity
- "SchoolYear" at "École Médina" might be "2024-2025" but actually start Sept 1, 2024 and end June 30, 2025
- Another school in Southern Hemisphere might have "2024-2025" starting Jan 2025

**Need**: Separate the *template* (AcademicYear) from the *instance* (SchoolYear).

### 2.2 New Table: AcademicYear

```sql
CREATE TABLE academic_year (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Natural Key
    code VARCHAR(10) NOT NULL UNIQUE,  -- "2024-2025"
    name VARCHAR(100) NOT NULL,        -- "Année Scolaire 2024-2025"

    -- Date range (guideline, not enforced)
    suggested_start_date DATE,
    suggested_end_date DATE,

    -- Configuration that applies across all schools using this academic year
    default_academic_year_type_id UUID REFERENCES academic_year_type(id),

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
```

**Relationship Change**:

```
BEFORE:
AcademicYearType → SchoolYear (1:N)
SchoolYear → SchoolYearCategory (1:N)
SchoolYearCategory → SchoolYearCategoryTerm (1:N)

AFTER:
AcademicYear → SchoolYear (1:N)  [NEW]
AcademicYearType → SchoolYear (N:1)  [moved from direct]
SchoolYear → SchoolYearCategory (1:N)
SchoolYearCategory → SchoolYearCategoryTerm (1:N)
```

### 2.3 Migration Impact

- Existing `school_year` table gains `academic_year_id` FK
- Queries that filter by academic year now join through AcademicYear
- District-level reporting queries by `academic_year.code` instead of parsing `school_year.code`

---

## 3. Issue 2: Per-Category TermType Override

### 3.1 Problem Statement

In the original design:

```
SchoolYear → AcademicYearType → TermType (global template)
         ↓
SchoolYearCategoryTerm (instances with dates)
```

This assumes **all categories** in a SchoolYear use the **same** TermType (SEMESTER or TRIMESTER). But reality:

| School Year | Category | Term Structure |
|-------------|----------|----------------|
| 2024-2025 | Primaire | 2 Semesters |
| 2024-2025 | Collège | 3 Trimesters |
| 2024-2025 | Lycée | 3 Trimesters |

A SchoolYear with mixed term types is **valid and common**.

### 3.2 Revised Design

**Option A: Keep SchoolYear → AcademicYearType, add SchoolYearCategory.term_type_id override** ✅ RECOMMENDED

```sql
-- SchoolYearCategory gains optional term_type_id override
ALTER TABLE school_year_category
ADD COLUMN term_type_id UUID REFERENCES term_type(id);

-- If NULL, inherit from SchoolYear's AcademicYearType
-- If set, use this TermType for generating SYCTs
```

**How it works**:

```
1. SchoolYear created with AcademicYear (which has default_academic_year_type)
   → SchoolYear.academic_year_type_id populated

2. SchoolYearCategory created for each Category relevant to school:
   Primaire category:
     term_type = NULL (inherits SchoolYear's type = SEMESTER)
     → Creates 2 SchoolYearCategoryTerms

   Collège category:
     term_type = <TermType for TRIMESTER>
     → Creates 3 SchoolYearCategoryTerms

   Lycée category:
     term_type = <TermType for TRIMESTER>
     → Creates 3 SchoolYearCategoryTerms
```

### 3.3 Business Rules for Override

**R-SYC-UPDATE-001**: `SchoolYearCategory.term_type_id` can be set only when:
- SchoolYearCategory is in `DRAFT` state
- No SchoolYearCategoryTerm records exist yet for this category
- The override TermType's `academic_year_type` matches or is compatible

**R-SYC-UPDATE-002**: If `term_type_id` is set, the count of SYCTs equals `TermType.academic_year_type.max_terms_per_year`, NOT inherited from SchoolYear.

**R-SYC-UPDATE-003**: Once SYCTs are generated, `term_type_id` becomes immutable (like SchoolYear's `academic_year_type_id`).

### 3.4 Domain Rule: SYCT Generation Algorithm

```
FUNCTION generate_school_year_category_terms(sycat_id):
    sycat = SELECT * FROM school_year_category WHERE id = sycat_id
    school_year = SELECT * FROM school_year WHERE id = sycat.school_year_id

    -- Determine TermType to use
    IF sycat.term_type_id IS NOT NULL:
        term_type = SELECT * FROM term_type WHERE id = sycat.term_type_id
    ELSE:
        term_type = SELECT * FROM term_type
                    WHERE academic_year_type_id = school_year.academic_year_type_id
                    ORDER BY sequence LIMIT 1
        -- Actually: fetch all TermTypes for that AcademicYearType

    -- Get ALL TermTypes for the chosen academic_year_type
    term_types = SELECT * FROM term_type
                 WHERE academic_year_type_id = term_type.academic_year_type_id
                 ORDER BY sequence

    -- Generate SYCT for each TermType
    FOR EACH term_type IN term_types:
        syct = INSERT SchoolYearCategoryTerm(
            school_year_category_id = sycat_id,
            term_type_id = term_type.id,
            code = sycat.code + '-' + term_type.code,
            name = sycat.name + ' — ' + term_type.name,
            sequence = term_type.sequence,
            start_date = calculate_term_start(...),  -- from AcademicYear pattern
            end_date = calculate_term_end(...),
            ...
        )
```

---

## 4. Updated Entity Relationships

### 4.1 Complete Diagram (Corrected)

```
                    AcademicYear
                         │
                         │ 1:N
                         ▼
                    SchoolYear
                    (academic_year_type_id)
                         │
          ┌──────────────┴──────────────┐
          │                             │
          ▼ 1:N                        ▼ 1:N
SchoolYearCategory A           SchoolYearCategory B
(Primaire, term_type=NULL)    (Collège, term_type=TRIMESTER)
          │                             │
          │ 1:N                         │ 1:N
          ▼                             ▼
    SYCT (2 terms)               SYCT (3 terms)
          │                             │
          └──────────┬──────────────────┘
                     │
                     ▼
                Evaluation, Grade
```

### 4.2 Table Summary with Corrections

| Table | Key Fields | Notes |
|-------|------------|-------|
| `academic_year` | `code`, `suggested_start_date`, `default_academic_year_type_id` | NEW table |
| `school_year` | `academic_year_id` (NEW FK), `academic_year_type_id` (moved FROM AcademicYear) | Now tied to AcademicYear |
| `school_year_category` | `term_type_id` (NEW, nullable) | Override school-level term type |
| `school_year_category_term` | Unchanged | Generated based on chosen TermType |

---

## 5. Data Migration Considerations

### 5.1 Adding AcademicYear

For existing SchoolYear records:

```sql
-- Create AcademicYear from existing school_year.code
INSERT INTO academic_year (code, name, created_at, created_by)
SELECT DISTINCT
    code as academic_year_code,
    'Année Scolaire ' || code as name,
    NOW() as created_at,
    'system' as created_by
FROM school_year
ON CONFLICT (code) DO NOTHING;

-- Update SchoolYear to point to AcademicYear
UPDATE school_year sy
SET academic_year_id = ay.id
FROM academic_year ay
WHERE ay.code = sy.code;
```

### 5.2 Adding term_type_id to SchoolYearCategory

```sql
-- Add column
ALTER TABLE school_year_category ADD COLUMN term_type_id UUID REFERENCES term_type(id);

-- For existing records, NULL is OK (inherits from school_year)
-- New records should explicitly set if different from school year
```

---

## 6. Impact on Existing TDRs

### 6.1 TDR-003 Updates Needed

**Section 2 (SchoolYear)**: Add `academic_year_id` foreign key and relationship.

**Section 3 (SchoolYearCategory)**:
- Add `term_type_id` field description
- Add business rules for override (R-SYC-UPDATE-001 through R-SYC-UPDATE-003)
- Update SYCT generation algorithm to consider per-category TermType

**Section 4 (SchoolYearCategoryTerm)**: No changes (same table)

### 6.2 TDR-002 Updates Needed

Add DDL for `academic_year` table:

```sql
CREATE TABLE academic_year (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(10) NOT NULL UNIQUE,
    name VARCHAR(100) NOT NULL,
    suggested_start_date DATE,
    suggested_end_date DATE,
    default_academic_year_type_id UUID REFERENCES academic_year_type(id),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by UUID NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by UUID,
    archived_at TIMESTAMPTZ,
    archived_by UUID
);

-- Update school_year to add academic_year_id
ALTER TABLE school_year
ADD COLUMN academic_year_id UUID REFERENCES academic_year(id);

-- Index
CREATE INDEX idx_academic_year_code ON academic_year(code);
CREATE INDEX idx_academic_year_is_active ON academic_year(is_active) WHERE is_active = TRUE;
CREATE INDEX idx_school_year_academic_year_id ON school_year(academic_year_id);
```

Add `term_type_id` to `school_year_category`:

```sql
ALTER TABLE school_year_category
ADD COLUMN term_type_id UUID REFERENCES term_type(id);

-- Unique constraint: term_type_id must be consistent within sycat
-- (if set, can't conflict with another override in same school, but different categories can differ)
CREATE INDEX idx_school_year_category_term_type ON school_year_category(term_type_id);
```

### 6.3 TDR-013 Scenarios Update

Scenario 3 (Promotion): Show that Collège uses TRIMESTER while Primaire uses SEMESTER for same 2024-2025 year.

Add scenario: "Setting per-category TermType during SchoolYear activation"

---

## 7. Domain Rules Summary (New/Updated)

### 7.1 AcademicYear Rules

**R-AY-001**: AcademicYear represents the named academic cycle across the education system
- Code format: `[start_year]-[end_year]`
- One AcademicYear can have multiple SchoolYears (one per school)

**R-AY-002**: AcademicYear provides `suggested_start_date`/`suggested_end_date` as guidelines for schools in that system

**R-AY-003**: AcademicYear can specify `default_academic_year_type_id` that new SchoolYears inherit unless overridden

**R-AY-004**: AcademicYear is relatively static — created by ministry/central authority once per year

### 7.2 SchoolYearCategory TermType Override Rules

**R-SYC-TERM-001**: `SchoolYearCategory.term_type_id` is optional
- `NULL` → inherit from `SchoolYear.academic_year_type`
- Set → use this TermType for SYCT generation

**R-SYC-TERM-002**: TermType override must have a different `academic_year_type` than SchoolYear's if different term count
- Cannot set TRIMESTER TermType if SchoolYear expects SEMESTER
- Actually, allowed as long as TermType exists and is active

**R-SYC-TERM-003**: All categories in the same SchoolYear can choose different TermTypes
- This is the key requirement: Primaire (SEMESTER), Collège (TRIMESTER)

**R-SYC-TERM-004**: Once SYCTs are generated, `term_type_id` becomes immutable

---

## 8. Validation Queries

### 8.1 Detect Incompatible Configurations

```sql
-- Find SchoolYearCategories using incompatible TermTypes
SELECT
    sy.code as school_year,
    cat.name as category,
    sycat.term_type_id,
    tt.code as term_type_code,
    ay.academic_year_type_id,
    ay_type.code as school_year_ay_type_code
FROM school_year_category sycat
JOIN school_year sy ON sycat.school_year_id = sy.id
JOIN category cat ON sycat.category_id = cat.id
LEFT JOIN academic_year ay ON sy.academic_year_id = ay.id
LEFT JOIN academic_year_type ay_type ON sy.academic_year_type_id = ay_type.id
LEFT JOIN term_type tt ON sycat.term_type_id = tt.id
WHERE sycat.term_type_id IS NOT NULL
  AND tt.academic_year_type_id != sy.academic_year_type_id;
```

### 8.2 Count SYCTs per Category (validate against TermType)

```sql
SELECT
    sycat.code,
    COUNT(syct.id) as actual_syct_count,
    tt.max_terms_per_year as expected_count,
    CASE
        WHEN COUNT(syct.id) = tt.max_terms_per_year THEN 'OK'
        ELSE 'MISMATCH'
    END as status
FROM school_year_category sycat
JOIN school_year sy ON sycat.school_year_id = synd.id
JOIN term_type tt ON COALESCE(sycat.term_type_id, ay_type.term_type_id) = tt.id
LEFT JOIN school_year_category_term syct ON sycat.id = syct.school_year_category_id
GROUP BY sycat.id, tt.max_terms_per_year;
```

---

## 9. API Implications

### 9.1 SchoolYear Creation

**Before**:
```
POST /v1/school-years
{
  "code": "2024-2025",
  "academic_year_type_id": "...",
  "start_date": "...",
  ...
}
```

**After**:
```
POST /v1/school-years
{
  "academic_year": {
    "code": "2024-2025",
    "suggested_start_date": "2024-09-01",
    "suggested_end_date": "2025-06-30"
  },
  "academic_year_type_id": "...",  // for school's default
  "start_date": "...",
  ...
}
```

`AcademicYear` created if doesn't exist; `school_year.academic_year_id` set.

### 9.2 SchoolYearCategory Creation

```
POST /v1/school-year-categories
{
  "school_year_id": "...",
  "category_id": "...",
  "term_type_id": "..."  // optional, override
}
```

Response shows selected TermType:
```json
{
  "term_type": {
    "code": "TRIMESTER",
    "max_terms_per_year": 3
  },
  "school_year_category_terms_generated": 3
}
```

---

## 10. Backward Compatibility

### Existing Systems (Before Correction)

If implementing this against an existing database:

1. Add `academic_year` table and migrate existing SchoolYears to link to it
2. Add `term_type_id` to `school_year_category` (NULL by default — inherits)
3. No data change needed if all schools used same TermType

### Breaking Changes

| Change | Impact | Mitigation |
|--------|--------|------------|
| AcademicYear table added | New join in queries | Use LEFT JOIN, NULL for old records |
| school_year_category.term_type_id | New column | NULL = inherit (default) |
| SYCT generation behavior | Could change if term_type set | Only affects new/edited categories |

---

## 11. Open Questions

1. **Can different categories have completely non-overlapping term dates?**
   - Primaire: Sept-Jan (S1), Feb-June (S2)
   - Collège: Sept-Nov (T1), Dec-Feb (T2), Mar-June (T3)
   - Currently SYCT dates calculated independently — should work

2. **What if `academic_year.suggested dates` differ from school needs?**
   - SchoolYear.start_date/end_date can override
   - AcademicYear just provides template

3. **Do we need `AcademicYearType` at all if Category overrides?**
   - Yes, as default for categories without override
   - Global administrative classification

---

## 12. Correction Checklist

- [ ] Add `academic_year` table to TDR-002 (data architecture)
- [ ] Update `school_year` DDL in TDR-002 to add `academic_year_id`
- [ ] Add `term_type_id` to `school_year_category` DDL
- [ ] Update TDR-003 domain rules (Sections 2, 3, 4)
- [ ] Update TDR-002 partitioning strategy (partition by academic_year_code not school_year_code)
- [ ] Update TDR-010 API contracts for new endpoints/fields
- [ ] Update TDR-013 scenarios to show mixed term types
- [ ] Update TDR-011 selectors to join through academic_year
- [ ] Update `table.md` to include AcademicYear

---

**Status**: Corrections identified, design specified. Implementation requires updating the referenced TDR documents with these changes.

This TDR-015 should be referenced as the authoritative source for:
- AcademicYear table design and usage
- SchoolYearCategory term_type_id override mechanism
- SYCT generation with per-category TermType selection
