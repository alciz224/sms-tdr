# TDR-008 — Domain Rules: User & Role System

## 1. Purpose and Scope

This document defines the domain rules for user management and role-based access control (Étape 2). The user system is the **security backbone** of the School Management System, controlling:

- Identity verification and authentication
- Role assignments with granular permissions
- Profile data specific to each user type
- Access control to sensitive student/grade data
- Offline authentication and capability restrictions

**Key Principle**: A user can have multiple roles, and each role has specific permissions scoped to the schools/departments they're associated with.

---

## 2. CustomUser Domain Rules

### 2.1 Definition

**CustomUser** is the authentication and identity base table, extending standard user management with:

- Multi-tenant support (user may belong to multiple schools)
- Federated identity (local + SSO options)
- Password policies and MFA requirements
- Session management for offline devices

### 2.2 Domain Rules

1. **R-USER-001**: Unique identity constraints:
   - `email` must be globally unique (case-insensitive)
   - `phone_number` must be unique if provided
   - `username` may be optional if using SSO; otherwise unique per tenant

2. **R-USER-002**: Email as primary identifier:
   - All notifications sent to `email`
   - Email change requires:
     - Verification of new email (send confirmation link)
     - 30-day grace period to use old email for login
     - Cascade update to all profile records

3. **R-USER-003**: Password policy (for password-based auth):
   - Minimum 12 characters
   - Must contain uppercase, lowercase, digit, special character
   - Cannot reuse last 12 passwords
   - Expires after 180 days (configurable per school)
   - Failed login lockout: 5 attempts → 30-minute lockout

4. **R-USER-004**: Multi-Factor Authentication (MFA):
   - Required for: Admin roles, Grade access, Financial data
   - Optional for: Teachers, Parents (recommended)
   - Methods: TOTP app, SMS OTP, Email OTP
   - Device trust: 30-day grace for trusted devices

5. **R-USER-005**: User status lifecycle:
   ```
   INVITED → PENDING_ACTIVATION → ACTIVE → SUSPENDED → DEACTIVATED
   ```
   - INVITED: Admin created, invitation sent
   - PENDING_ACTIVATION: User clicked link, profile setup pending
   - ACTIVE: Normal operation
   - SUSPENDED: Temporarily disabled (security, leave)
   - DEACTIVATED: Permanently disabled (attrition, violation)

6. **R-USER-006**: Tenant association (multi-school):
   - User can be associated with multiple School entities
   - Current active school selected at login
   - Role assignments Scope: Global (system admin) or per-school
   - Data isolation: User sees only data from current active school

7. **R-USER-007**: Last login tracking:
   - `last_login_at` updated on successful auth
   - `last_login_ip` stored for audit
   - Devices table tracks device IDs for offline sync

8. **R-USER-008**: Privacy and compliance:
   - GDPR right to erasure: Mark as DEACTIVATED, retain audit data
   - Data minimization: PII field-by-field deletion policy
   - Consent tracking for marketing/notifications

**Fields**:
- `id` (UUID)
- `email` (VARCHAR[254], unique, normalized to lowercase)
- `email_verified` (BOOLEAN, default FALSE)
- `email_verified_at` (TIMESTAMPTZ)
- `phone_number` (VARCHAR[20], unique nullable)
- `phone_verified` (BOOLEAN)
- `username` (VARCHAR[150], unique nullable)
- `password_hash` (VARCHAR[255], nullable for SSO-only)
- `mfa_enabled` (BOOLEAN, default FALSE)
- `mfa_method` (VARCHAR[20]) — "TOTP", "SMS", "EMAIL"
- `mfa_secret` (VARCHAR[255], encrypted)

- `first_name` (VARCHAR[150])
- `last_name` (VARCHAR[150])
- `full_name` (VARCHAR[300], computed/denormalized)
- `avatar_url` (VARCHAR[500])
- `date_of_birth` (DATE, nullable)
- `gender` (VARCHAR[10]) — "MALE", "FEMALE", "OTHER", "PREFER_NOT_TO_SAY"
- `nationality` (VARCHAR[100])
- `language` (VARCHAR[10]) — "fr", "en", "wo" — UI localization

- `status` (VARCHAR[20]) — "INVITED", "PENDING_ACTIVATION", "ACTIVE", "SUSPENDED", "DEACTIVATED"
- `status_changed_at` (TIMESTAMPTZ)
- `status_changed_by` (UUID)

- `last_login_at` (TIMESTAMPTZ)
- `last_login_ip` (INET)
- `last_login_device_id` (UUID)

- `password_changed_at` (TIMESTAMPTZ)
- `failed_login_attempts` (INTEGER, default 0)
- `locked_until` (TIMESTAMPTZ)
- `must_change_password` (BOOLEAN, default FALSE)

- `offline_sync_enabled` (BOOLEAN, default TRUE)
- `offline_sync_device_ids` (JSONB) — array of registered device IDs

- `tenant_id` (UUID) — for multi-tenancy (nullable for system admin)
- `is_system_admin` (BOOLEAN, default FALSE) — global admin flag

- `terms_accepted_at` (TIMESTAMPTZ)
- `privacy_policy_version` (VARCHAR[20])
- `gdpr_consent` (JSONB)

- Audit trail (created_at, created_by, updated_at, updated_by, archived_at, archived_by)

**Unique Constraints**:
- `UNIQUE(email)` — globally unique
- `UNIQUE(username, tenant_id)` — per-tenant unique (when username used)

**Indexes**:
- `idx_customuser_email` (email)
- `idx_customuser_username` (username, tenant_id)
- `idx_customuser_status` (status)
- `idx_customuser_tenant` (tenant_id, is_system_admin)
- `idx_customuser_active` (status) WHERE status = 'ACTIVE'

---

### 2.3 Authentication Flow Rules

```
1. Login Request
   ↓
2. Check user exists and status = ACTIVE
   ↓ [if invalid] → error
   ↓
3. Verify password / SSO token
   ↓ [if invalid] → increment failed_login_attempts
                   → lock if threshold reached
   ↓
4. Check MFA if required for user's roles
   ↓ [if MFA required but not provided/enabled] → MFA challenge
   ↓ [if MFA fails] → deny
   ↓
5. Generate session token (JWT or opaque)
   ↓
6. Return:
   - user_id
   - roles (for current tenant)
   - permissions (computed for roles)
   - tenant context (current school)
   - profile data
```

---

## 3. Role System Domain Rules

### 3.1 Role Definition

A **Role** defines a set of permissions and capabilities. Roles are **tenant-scoped** (per school) but can have system-wide variants.

**Standard Roles** (defined at system initialization):
| Role Code | Display Name | Description | Scope |
|-----------|-------------|-------------|-------|
| SYSTEM_ADMIN | System Administrator | Full system access, tenant management | Global |
| SCHOOL_ADMIN | School Administrator | All school operations | Per School |
| PRINCIPAL | Principal | School oversight, staff management | Per School |
| VICE_PRINCIPAL | Vice Principal | Academic supervision | Per School |
| ACADEMIC_INSPECTOR | Academic Inspector | Grade review, promotion decisions | District/Region |
| TEACHER | Teacher | Grade entry, attendance, own classes | Per School/Class |
| HEAD_TEACHER | Head of Department | Department oversight | Per School/Department |
| REGISTRAR | Registrar | Enrollment, student records | Per School |
| ACCOUNTANT | Accountant | Financial records, invoices | Per School |
| PARENT | Parent | View children's data | Per Student |
| STUDENT | Student | View own data, assignments | Self |
| LIBRARIAN | Librarian | Library management | Per School |
| NURSE | School Nurse | Health records | Per School |

### 3.2 Domain Rules

1. **R-ROLE-001**: Role hierarchy (optional simplification via role inheritance):
   ```
   SYSTEM_ADMIN > SCHOOL_ADMIN > PRINCIPAL > TEACHER
                          └── REGISTRAR ───────┘
   ```
   - Higher roles inherit all permissions of lower roles
   - Can be overridden by explicit DENY

2. **R-ROLE-002**: Permission granularity:
   - CRUD permissions per resource (Student, Enrollment, Grade, etc.)
   - Field-level: some roles can see but not edit certain fields
   - Record-level: teacher sees only own classroom's students

3. **R-ROLE-003**: Role assignment:
   - User can have multiple roles
   - Roles can be assigned globally (across all tenant schools) or per-school
   - Student/Parent roles automatically created via enrollment linkage
   - Role changes require approval (except self-assignment invite)

4. **R-ROLE-004**: Permission evaluation:
   ```
   user_permission = ANY(
       role.permissions
       WHERE role.tenant_id IN (user.current_tenant, GLOBAL)
         AND role.is_active = true
   )
   ```

5. **R-ROLE-005**: Self-service role capabilities:
   - Student: can view own data, submit assignments (if feature enabled)
   - Parent: can view linked children's data, submit absence notes
   - Teacher: can enter grades, take attendance, view own classes

---

## 4. RolePermission Junction Table

Defines fine-grained permissions per role.

```sql
-- Domain Rules documents referential structure, not full DDL
RolePermission:
- role_id (FK to Role)
- resource (e.g., "grade", "enrollment", "evaluation")
- action ("CREATE", "READ", "UPDATE", "DELETE", "APPROVE")
- scope ("GLOBAL", "SCHOOL", "DEPARTMENT", "CLASSROOM", "SELF")
- conditions (JSONB) — e.g., {"only_own_classes": true}
```

**Example Permissions**:
- TEACHER: READ grade (scope: classroom), UPDATE grade (own evaluations only)
- REGISTRAR: CREATE enrollment, READ student (all)
- PRINCIPAL: DELETE enrollment (only inactive), APPROVE promotion
- PARENT: READ grade (children only), READ attendance (children only)

---

## 5. Profile Tables Domain Rules

### 5.1 TeacherProfile

**Definition**: Extended profile for teaching staff, including qualifications, subject competence, and employment details.

**Domain Rules**:

1. **R-TP-001**: Teacher must have at least one teaching subject:
   - `taught_subjects` stores array of Subject IDs they're qualified to teach
   - Subject competence verified during assignment to ClassroomSubject
   - Subject list derived from TeacherQualification table (preferred)

2. **R-TP-002**: Employment status lifecycle:
   ```
   APPLIED → INTERVIEWED → HIRED → ACTIVE → ON_LEAVE → TERMINATED
   ```
   - Only ACTIVE teachers can be assigned to classes
   - ON_LEAVE: scheduled but not teaching, cannot enter grades

3. **R-TP-003**: Weekly teaching hours:
   - `weekly_teaching_hours` maximum (contractual)
   - Actual scheduled hours computed from TimetableEntry
   - Cannot exceed max unless special approval (overload pay)

4. **R-TP-004**: Teacher-school association:
   - Teacher can teach at multiple schools (visiting lecturer pattern)
   - Assigned SchoolProfile references via `schools_taught_at`
   - Current active school determines current timetable

5. **R-TP-005**: Substitute capability:
   - `is_substitute_qualified` indicates can serve as substitute
   - `substitute_subjects` lists subjects they can cover
   - Used for absence coverage

6. **R-TP-006**: Timetable optimization flags:
   - `preferred_time_slots` — teacher's availability constraints
   - `unavailable_days` — regular unavailability
   - `max_consecutive_periods` — for scheduling algorithm

7. **R-TP-007**: Documentation and compliance:
   - `teaching_certificate_number` + expiry
   - `background_check_date` + expiry
   - `contract_type` ("PERMANENT", "CONTRACT", "VOLUNTEER", "SUBSTITUTE")
   - Renewal alerts 90 days before expiry

**Fields**:
- `id` (UUID)
- `user_id` (UUID, foreign key to CustomUser, unique)
- `employee_number` (VARCHAR[20], unique per school)
- `hire_date` (DATE)
- `employment_status` (VARCHAR[20]) — "APPLIED", "HIRED", "ACTIVE", "ON_LEAVE", "TERMINATED"
- `contract_type` (VARCHAR[20])
- `contract_end_date` (DATE)

- `department` (VARCHAR[100]) — "Mathematics", "Sciences", "Languages"
- `taught_subjects` (JSONB array of Subject IDs)
- `qualifications` (JSONB) — [{"degree": "B.Sc Math", "institution": "...", "year": 2010}]

- `weekly_teaching_hours` (INTEGER, default 35)
- `max_class_size_preference` (INTEGER, default 30)
- `is_substitute_qualified` (BOOLEAN)
- `substitute_subjects` (JSONB)

- Scheduling preferences
  - `preferred_time_slots` (JSONB) — ["MON_1", "MON_2", "TUE_3", ...]
  - `unavailable_days` (JSONB) — ["SAT"]
  - `max_consecutive_periods` (INTEGER, default 4)

- Compliance
  - `teaching_certificate_number` (VARCHAR[50])
  - `teaching_certificate_expiry` (DATE)
  - `background_check_date` (DATE)
  - `background_check_expiry` (DATE)

- Payroll (optional integration)
  - `salary_grade` (VARCHAR[20])
  - `bank_account_number` (VARCHAR[50], encrypted)
  - `tax_identification_number` (VARCHAR[50], encrypted)

- Audit trail
- `is_active`

**Indexes**:
- `(employee_number, school_id)` unique per school
- `(user_id)` foreign key
- `(department)` for reporting

---

### 5.2 StudentProfile

**Definition**: Extended profile for enrolled students, including demographic, academic history, and parent/guardian information.

**Domain Rules**:

1. **R-STU-001**: Student registration:
   - Must have parent/guardian linked (ParentProfile)
   - `date_of_birth` determines level eligibility (age check)
   - Previous school transcript required for transfers
   - Medical information required (immunizations, allergies)

2. **R-STU-002**: Student identification:
   - `student_number` unique per school
   - `qr_code_url` for ID card generation
   - `national_student_id` (government-issued, optional but recommended)

3. **R-STU-003**: Guardian/parent association:
   - Must have at least one emergency contact (ParentProfile)
   - `legal_guardians` array of parent user IDs
   - Financial responsibility assigned to guardian(s)
   - Both parents can have access if both give consent

4. **R-STU-004**: Academic history:
   - `previous_schools` array with:
     - school name, dates attended
     - final grades/transcript
     - reason for leaving
   - `entrance_exam_results` if applicable
   - `special_needs` documentation linked

5. **R-STU-005**: Health and medical:
   - `medical_conditions` (JSONB array)
   - `allergies` (JSONB array with severity)
   - `medications` (JSONB with dosage instructions)
   - `doctor_contact` (reference to contact)
   - `immunization_records` (JSONB array)

6. **R-STU-006**: Transport and logistics:
   - `transportation_method` ("BUS", "WALK", "PRIVATE", "PUBLIC")
   - `bus_route_number` if applicable
   - `pickup_location` / `dropoff_location`
   - `dismissal_time` — early dismissal instructions

7. **R-STU-007**: Photo and media consent:
   - `photo_consent` — "PARENTAL_CONSENT", "NO_PUBLIC", "SCHOOL_ONLY"
   - `media_release` — can student image be used in marketing?
   - `name_publication` — can name appear in honor roll?

8. **R-STU-008**: Academic tracking fields (updated annually):
   - `current_level_id` — current grade level
   - `current_track_id` — current track/specialization
   - `academic_standing` — "GOOD", "PROBATION", "WARNING", "HONOR"
   - `promotion_year` — expected graduation year

9. **R-STU-009**: Withdrawal/graduation:
   - `withdrawal_date` if student leaves before completion
   - `withdrawal_reason` (DROP_OUT, TRANSFER, EXPULSION, GRADUATION)
   - `final_transcript_sent` (boolean + date)
   - `exit_interview_conducted` (boolean)

**Fields**:
- `id` (UUID)
- `user_id` (UUID, foreign key to CustomUser, unique)
- `student_number` (VARCHAR[20], unique per school)
- `qr_code_url` (VARCHAR[500])
- `national_student_id` (VARCHAR[50], unique, nullable)

- `date_of_birth` (DATE, not null)
- `gender` (VARCHAR[10])
- `nationality` (VARCHAR[100])
- `ethnicity` (VARCHAR[100], nullable — privacy sensitive)
- `language_spoken_at_home` (VARCHAR[100])

- `entry_date` (DATE) — first day at this school
- `entry_level_id` (UUID, FK to Level) — initial placement
- `entry_school_year_id` (UUID, FK to SchoolYear)
- `enrollment_type` ("NEW", "TRANSFER", "RETURNING")

- Academic
  - `current_level_id` (UUID, FK to Level)
  - `current_track_id` (UUID, FK to Track, nullable)
  - `academic_standing` (VARCHAR[20])
  - `promotion_year` (INTEGER) — expected graduation

- Guardian information
  - `legal_guardians` (JSONB array of user_ids)
  - `emergency_contact_phone` (VARCHAR[20])
  - `emergency_contact_name` (VARCHAR[200])

- Health
  - `medical_conditions` (JSONB)
  - `allergies` (JSONB)
  - `medications` (JSONB)
  - `doctor_contact` (JSONB)
  - `immunization_records` (JSONB)
  - `blood_type` (VARCHAR[10], nullable)

- Transport
  - `transportation_method` (VARCHAR[20])
  - `bus_route_number` (VARCHAR[20])
  - `pickup_location` (VARCHAR[200])
  - `dropoff_location` (VARCHAR[200])
  - `dismissal_time` (TIME) — early dismissal

- Photo/media
  - `photo_consent` (VARCHAR[30])
  - `media_release` (BOOLEAN)
  - `name_publication` (BOOLEAN)

- Previous schools
  - `previous_schools` (JSONB array)

- Withdrawal/graduation
  - `withdrawal_date` (DATE)
  - `withdrawal_reason` (VARCHAR[50])
  - `final_transcript_sent` (BOOLEAN)
  - `final_transcript_sent_date` (DATE)
  - `exit_interview_conducted` (BOOLEAN)

- `is_active` (BOOLEAN)
- Audit trail

**Indexes**:
- `(student_number, school_id)` — lookup by student number
- `(user_id)` — FK to CustomUser
- `(current_level_id, current_track_id)` — classroom assignment

---

### 5.3 ParentProfile

**Definition**: Extended profile for parents/guardians of students.

**Domain Rules**:

1. **R-PAR-001**: Parent-student linkage:
   - `children_student_ids` array of StudentProfile IDs
   - Relationship type per child: "MOTHER", "FATHER", "GUARDIAN", "GRANDPARENT"
   - Legal guardianship status tracked
   - Financial responsibility flag

2. **R-PAR-002**: Notification preferences:
   - `notification_channels` — ["EMAIL", "SMS", "APP"]
   - `notification_frequency` — "REAL_TIME", "DAILY_DIGEST", "WEEKLY_SUMMARY"
   - `notification_language` — "en", "fr", "wo"

3. **R-PAR-003**: Access restrictions:
   - Parent can only see own children's data
   - Cannot view other students even if same school
   - Cannot access teacher-only data (grade entry, attendance modify)

4. **R-PAR-004**: Parent-teacher communication:
   - `communication_preferences` — topics allowed (GRADES, ATTENDANCE, CONDUCT)
   - `allowed_contact_hours` — restrict messaging times
   - `blocked_teachers` (rare, requires admin override)

5. **R-PAR-005**: Financial responsibility:
   - `financial_guardian_for` array of student IDs
   - Receives billing notifications
   - Can view payment history, make online payments

6. **R-PAR-006**: Pick-up authorization:
   - `authorized_pickup_persons` — who can pick up child
   - Photo ID required for unauthorized persons
   - Changes require 24-hour notice (security)

**Fields**:
- `id` (UUID)
- `user_id` (UUID, FK to CustomUser, unique)
- `parent_type` — "MOTHER", "FATHER", "GUARDIAN", "OTHER"

- Children linkage
  - `children_student_ids` (JSONB array of student_profile IDs)
  - `relationship_to_child` (JSONB) — {student_id: "MOTHER", ...}

- Work/business info (for communication)
  - `employer` (VARCHAR[200])
  - `work_phone` (VARCHAR[20])
  - `work_hours` (VARCHAR[50])

- Communication preferences
  - `notification_channels` (JSONB)
  - `notification_frequency` (VARCHAR[20])
  - `preferred_contact_method` (VARCHAR[20])
  - `language_preference` (VARCHAR[10])

- Pick-up authorization
  - `authorized_pickup_persons` (JSONB array of names)
  - `requires_id_verification` (BOOLEAN, default TRUE)

- Financial
  - `financial_guardian_for` (JSONB array of student_ids)
  - `billing_agreement_accepted` (BOOLEAN)
  - `auto_pay_enabled` (BOOLEAN)

- `is_active` (BOOLEAN)
- Audit trail

**Indexes**:
- `(user_id)` unique
- GIN index on `children_student_ids` for parent lookup by student

---

### 5.4 AdminProfile

**Definition**: Extended profile for school administrators and system staff.

**Domain Rules**:

1. **R-ADM-001**: Admin role scope:
   - `admin_scope` — "SYSTEM", " SCHOOL", "DEPARTMENT"
   - `school_ids` array of schools they administer (for SCHOOL scope)
   - Global admins see all schools; school admins see only their school

2. **R-ADM-002**: Administrative permissions matrix:
   - `can_manage_users` — create/modify CustomUser records
   - `can_manage_enrollments` — override enrollment rules
   - `can_modify_grades` — emergency grade changes
   - `can_archive_data` — long-term retention management
   - `can_view_audit_logs` — full audit access

3. **R-ADM-003**: Emergency override capabilities:
   - `emergency_override_enabled` — can bypass certain constraints
   - Emergency actions logged with justification
   - Requires MFA for overrides

4. **R-ADM-004**: Reporting and analytics:
   - `report_access_level` — "BASIC", "SCHOOL", "DISTRICT", "SYSTEM"
   - Can export aggregated data (anonymized options)
   - PII access logged for compliance

**Fields**:
- `id` (UUID)
- `user_id` (UUID, FK to CustomUser, unique)
- `admin_scope` (VARCHAR[20])
- `school_ids` (JSONB array of school UUIDs)

- Permissions
  - `can_manage_users` (BOOLEAN)
  - `can_manage_enrollments` (BOOLEAN)
  - `can_approve_promotions` (BOOLEAN)
  - `can_modify_grades` (BOOLEAN)
  - `can_manage_timetable` (BOOLEAN)
  - `can_view_financials` (BOOLEAN)
  - `can_export_data` (BOOLEAN)
  - `can_view_audit_logs` (BOOLEAN)
  - `emergency_override_enabled` (BOOLEAN)

- Reporting
  - `report_access_level` (VARCHAR[20])
  - `can_export_pii` (BOOLEAN) — personally identifiable information

- `is_active` (BOOLEAN)
- Audit trail

**Indexes**:
- `(user_id)` unique
- `(admin_scope, is_active)`

---

## 6. Permission Evaluation Algorithm

### 6.1 Permission Check

```
Given: user_id, action, resource, school_context (optional)

1. Get all Roles assigned to user (global + in current school)
2. For each Role, get RolePermissions where:
   resource = requested resource
   action = requested action
3. Check Scope constraint:
   - GLOBAL: always allowed
   - SCHOOL: user's current school must be in user.school_ids
   - DEPARTMENT: user's department must match
   - CLASSROOM: user must be assigned to classroom
   - SELF: resource.user_id must equal user_id
4. Check Conditions from RolePermission.conditions JSONB
5. If any role grants permission without condition violation → ALLOWED
6. If explicit DENY found (role has deny flag) → DENIED
7. Default: DENIED
```

### 6.2 Permission Caching

Permissions cached per user session (24-hour TTL):
```json
{
  "user_id": "...",
  "computed_permissions": [
    {"resource": "grade", "action": "READ", "scope": "classroom"},
    {"resource": "enrollment", "action": "CREATE", "scope": "school"}
  ],
  "cached_at": "2025-03-29T10:30:00Z"
}
```

---

## 7. Offline-First User/Role Considerations

### 7.1 Offline Authentication

- User credentials (password hash or SSO token) cached locally
- Offline login: validate against cached credentials
- Session token refreshed upon reconnection with server-side validation

### 7.2 Offline Permission Enforcement

- Permissions computed at login and cached
- Offline operations checked against cached permissions
- Server validates on sync; rejects operations user shouldn't perform

### 7.3 Role Change Propagation

Role changes (admin modifies user's roles):
- Pushed to all user's registered devices immediately
- Device invalidates cached permissions
- User forced to re-login for new permissions to take effect

---

## 8. Data Integrity Constraints

| Invariant | Enforcement |
|-----------|--------------|
| One StudentProfile per CustomUser | Unique constraint on student_profile.user_id |
| At least one legal guardian per student | Application validation on StudentProfile.create |
| Teacher taught_subjects includes all assigned subjects | Trigger validation on ClassroomSubject.teacher_id assignment |
| Cannot delete CustomUser with active enrollments | Foreign key CASCADE (deactivate profile only) |
| User's roles must be from tenant's available roles | FK constraint with tenant_id filtering |

---

## 9. Security & Privacy Rules

1. **PII Protection**:
   - Full name, DOB, address stored encrypted at rest
   - Only visible to users with explicit needs (registrar, admin, assigned teacher)
   - Parents see PII of own children only
   - Teachers see limited PII (name, photo, emergency contact)

2. **Student Data Access Logging**:
   - Every access to student profile logged
   - Alert on unusual access patterns (teacher viewing other class's students)
   - Retain access logs for 10 years

3. **GDPR Compliance**:
   - Right to erasure: Mark as inactive, retain audit trail
   - Data export: Generate JSON/CSV of all personal data
   - Consent management: Track purposes for data use

4. **Role Assignment Approval**:
   - SYSTEM_ADMIN role requires multi-admin approval
   - CHANGE requests logged with justification
   - Emergency role grant limited to 24 hours

---

## 10. Performance Considerations

- User lookup by email: indexed, < 10ms
- Role/permission retrieval: cached in session
- Guardian lookup (by student): GIN index on JSONB array
- Multi-school user's active school flag: indexed

---

## 11. Data Model Summary

```
CustomUser (1) ───< UserRole (N) >── Role (1)
       │                        │
       ├── TeacherProfile (1)   └── RolePermission (N) >── Permission
       ├── StudentProfile (1)
       ├── ParentProfile (1)
       └── AdminProfile (1)

ParentProfile.children_student_ids ──< StudentProfile.id
```

---

**Next Step**: TDR-009 — Domain Rules: Service Layer (Enrollment Service, Grade Service, Sync Service, etc.)
