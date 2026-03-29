# TDR-010 — API Contracts

## 1. Purpose and Scope

This document defines the **Application Programming Interface (API) layer** contracts for the School Management System. It specifies:

- **API design principles** and conventions
- **Authentication & authorization** mechanisms
- **RESTful endpoint specifications** for all domains
- **GraphQL schema** (optional alternative)
- **Request/response formats**, error handling, pagination
- **Webhook/event subscriptions** for integrations
- **Versioning strategy** for 10+ year evolution
- **SDK generation guidelines** for client libraries

**Key Principle**: APIs are the primary integration surface. They must be stable, well-documented, and backward-compatible across major versions.

---

## 2. API Design Principles

### 2.1 Architectural Style

**Primary: RESTful HTTP API** with JSON payloads
- Resources identified by nouns (not verbs)
- HTTP methods for operations (GET, POST, PUT, DELETE, PATCH)
- Stateless requests with authentication tokens
- Standard status codes for responses

**Secondary: GraphQL** (optional for flexible queries)
- Single endpoint `/graphql`
- Schema-first design
- Introspection enabled for SDK generation
- Used primarily by dashboards and admin consoles

**Tertiary: WebSockets** (real-time updates)
- Event subscriptions: grade updates, attendance, notifications
- Connection authenticated via JWT

### 2.2 Base URL Structure

```
Production:  https://api.schoolsync.io/v1/
Staging:     https://api-staging.schoolsync.io/v1/
```

### 2.3 Versioning Strategy

**URL Path Versioning**: `/v1/`, `/v2/`

- Major version increment = breaking changes (retire old version after 2 years)
- Minor changes (new fields, endpoints) add to current version
- Deprecated endpoints return `Deprecation` header + 1 year notice

**Supported Versions**:
- Latest version (v1 at launch)
- Previous version (v0 if exists, retired after 1 year)
- No more than 2 versions active simultaneously

---

## 3. Authentication & Authorization

### 3.1 Authentication Methods

#### 3.1.1 Bearer Token (Primary)

```
Authorization: Bearer <jwt_token>
```

JWT (JSON Web Token) with claims:
```json
{
  "sub": "user-uuid",
  "email": "teacher@school.edu",
  "roles": ["TEACHER"],
  "tenant_id": "school-uuid",
  "permissions": [
    "grade:read:classroom",
    "grade:create:own_evaluations"
  ],
  "exp": 1743264000,
  "iat": 1740657600
}
```

**Token Exchange Flow**:
```
1. Client sends: {email, password, tenant_id}
2. /auth/login validates credentials + MFA if required
3. Returns: {access_token, refresh_token, expires_in}
4. Client uses access_token for API calls
5. On expiry: use refresh_token at /auth/refresh
```

#### 3.1.2 API Key (For Integrations)

```
X-API-Key: <api_key>
X-Tenant-ID: <tenant_uuid>
```

For system-to-system integrations (SIS sync, payment gateway).

#### 3.1.3 Offline Device Token

Long-lived token for offline devices:
```
X-Device-ID: <device_uuid>
X-Device-Token: <signed_device_token>
X-User-ID: <user_uuid>
```

Offline sync uses this header combination.

---

### 3.2 Authorization Model

**Role-Based Access Control (RBAC)** with scoping:

```
Permission Format: <resource>:<action>:<scope>
Examples:
  - grade:read:classroom (read grades in assigned classroom)
  - grade:create:own_evaluations (create grades for own evaluations)
  - enrollment:read:school (read all enrollments in school)
  - student:read:self (read own profile)
```

**Evaluation**:
1. Get user's roles for current tenant
2. Aggregate all permissions from roles
3. Check if requested (resource, action, scope) matches any permission
4. Apply scope filters (row-level) based on user context

---

## 4. Common Response Format

### 4.1 Success Response

```json
{
  "success": true,
  "data": { ... },           // requested resource(s)
  "meta": {                  // pagination, rate limit info
    "page": 1,
    "per_page": 20,
    "total": 150,
    "has_more": true,
    "rate_limit": {
      "remaining": 95,
      "reset_at": "2025-03-29T11:00:00Z"
    }
  },
  "included": [              // related resources (JSON API style)
    { "type": "user", "id": "...", "attributes": {...} }
  ]
}
```

### 4.2 Error Response

```json
{
  "success": false,
  "error": {
    "code": "ENROLLMENT_CAPACITY_EXCEEDED",
    "message": "Classroom has reached maximum capacity",
    "field": "classroom_id",
    "suggestion": "Select a different classroom or create a new section",
    "details": {
      "classroom_id": "abc-123",
      "current_enrollment": 30,
      "max_capacity": 30
    },
    "request_id": "req-uuid-for-support"
  }
}
```

---

## 5. REST API Endpoints

### 5.1 Enrollment Endpoints

#### List Enrollments

```
GET /v1/enrollments
```

**Query Parameters**:
- `school_year_category_id` (required): Filter by SYC
- `classroom_id`: Filter by classroom
- `state`: Filter by state (ACTIVE, PENDING, WITHDRAWN, COMPLETED)
- `student_id`: Filter by student
- `page` (default: 1)
- `per_page` (default: 20, max: 100)
- `sort` (default: `-created_at`)

**Response** (200):
```json
{
  "data": [
    {
      "id": "enrollment-uuid",
      "type": "enrollment",
      "attributes": {
        "enrollment_number": "2024-2025-000123",
        "student": {
          "id": "student-uuid",
          "full_name": "Amina Diallo",
          "student_number": "STU001"
        },
        "classroom": {
          "id": "class-uuid",
          "code": "5EME-S-A",
          "level": "5ème",
          "track": "Scientifique"
        },
        "state": "ACTIVE",
        "admission_date": "2024-09-02",
        "level": { "code": "5E", "name": "Cinquième" },
        "track": { "code": "S", "name": "Scientifique" },
        "promotion_decision": null,
        "created_at": "2025-03-29T10:30:00Z"
      }
    }
  ],
  "meta": { "page": 1, "total": 150, "has_more": true }
}
```

**Permissions**:
- `enrollment:read:school` — all enrollments
- `enrollment:read:classroom` — only assigned classroom
- `enrollment:read:self` (parent) — own children only

---

#### Create Enrollment

```
POST /v1/enrollments
Content-Type: application/json
```

**Request Body**:
```json
{
  "student_profile_id": "student-uuid",
  "school_year_category_id": "syc-uuid",
  "level_id": "level-uuid",           // optional, auto-determined if omitted
  "track_id": "track-uuid",           // optional
  "enrollment_type": "NEW",
  "preferred_classroom_ids": ["class-uuid-1"],  // optional preference
  "scholarship_percentage": 0,
  "payment_plan_id": "plan-uuid"      // optional
}
```

**Response** (201):
```json
{
  "data": {
    "id": "new-enrollment-uuid",
    "enrollment_number": "2024-2025-000124",
    "state": "PENDING",
    "estimated_fee": 1500000,
    "required_documents": [
      "BIRTH_CERTIFICATE",
      "IMMUNIZATION_RECORD",
      "PREVIOUS_TRANSCRIPT"
    ],
    "classroom": null,  // assigned upon activation
    "links": {
      "upload_documents": "/v1/enrollments/abc123/documents"
    }
  }
}
```

**Error Codes**:
- `ENROLLMENT_DUPLICATE`: Student already enrolled this year
- `CLASSROOM_CAPACITY_EXCEEDED`: No seats available
- `LEVEL_AGE_INELIGIBLE`: Age doesn't match level
- `PREREQUISITE_MISSING`: Required documents or conditions not met

---

#### Activate Enrollment

```
POST /v1/enrollments/{id}/activate
```

**Request Body**:
```json
{
  "operator_notes": "Student completed registration, fees paid",
  "override_capacity": false  // only for admins
}
```

**Response** (200):
```json
{
  "data": {
    "id": "...",
    "state": "ACTIVE",
    "classroom": {
      "id": "class-uuid",
      "code": "5EME-S-A",
      "homeroom_teacher": { "name": "Mme. Diop" }
    },
    "student_identifier": {
      "qr_code_url": "https://schoolsync.io/qr/STU001",
      "student_card_pdf_url": "/v1/students/abc/student-card"
    },
    "assigned_subjects": [
      { "code": "MAT", "name": "Mathématiques", "teacher": "M. Sow" },
      { "code": "PHY", "name": "Physique", "teacher": "Mme. Ba" }
    ]
  }
}
```

---

#### Withdraw Enrollment

```
DELETE /v1/enrollments/{id}
```

**Query Parameters**:
- `reason`: WITHDRAW_REASON (FAMILY_MOVED, TRANSFER, WITHDRAWN, EXPULSION)
- `withdrawal_date`: Date (defaults to today)

**Response** (200):
```json
{
  "data": {
    "id": "...",
    "state": "WITHDRAWN",
    "withdrawal_reason": "FAMILY_MOVED",
    "withdrawal_date": "2025-03-29",
    "refund_calculated": 1250000,
    "refund_status": "PROCESSING"
  }
}
```

---

### 5.2 Grade Endpoints

#### Enter Grade

```
POST /v1/grades
Content-Type: application/json
```

**Request Body**:
```json
{
  "evaluation_id": "eval-uuid",
  "student_profile_id": "student-uuid",
  "value": 15.5,
  "justification": "Excellent analysis of quadratic equations",
  "is_exempt": false,
  "exemption_reason": null
}
```

**Response** (201):
```json
{
  "data": {
    "id": "grade-uuid",
    "evaluation": {
      "code": "2024-2025-MATH5A-001",
      "name": "Test 3 - Algebra",
      "max_score": 20
    },
    "student": {
      "full_name": "Amina Diallo",
      "student_number": "STU001"
    },
    "value": 15.5,
    "state": "ENTERED",
    "relative_position": "above average",
    "class_average": 12.8,
    "links": {
      "submit": "/v1/evaluations/abc/submit"
    }
  }
}
```

**Error Codes**:
- `GRADE_EVALUATION_NOT_FOUND`: Invalid evaluation ID
- `GRADE_EVALUATION_LOCKED`: Evaluation already finalized
- `GRADE_VALUE_OUT_OF_RANGE`: Value exceeds max_score
- `GRADE_STUDENT_NOT_ENROLLED`: Student not in this class

---

#### Submit Evaluation

```
POST /v1/evaluations/{id}/submit
```

**Response** (200):
```json
{
  "data": {
    "id": "...",
    "state": "SUBMITTED",
    "submitted_at": "2025-03-29T14:30:00Z",
    "completion_percentage": 100,
    "total_students_expected": 30,
    "total_students_completed": 30,
    "next_steps": [
      "Await academic council review",
      "Review expected within 7 days"
    ]
  }
}
```

---

#### Get Student Report Card

```
GET /v1/students/{student_id}/report-cards
```

**Query Parameters**:
- `school_year_category_id`: Filter by term
- `term_code`: Specific term
- `include_class_ranking`: true/false

**Response** (200):
```json
{
  "data": {
    "student": {
      "full_name": "Amina Diallo",
      "student_number": "STU001",
      "classroom": "5EME-S-A"
    },
    "school_year": "2024-2025",
    "term": "Trimestre 2",
    "generated_at": "2025-03-29T10:00:00Z",
    "subjects": [
      {
        "code": "MAT",
        "name": "Mathématiques",
        "teacher": "M. Sow",
        "evaluations": [
          { "name": "Test 1", "score": 14.0, "max": 20, "coefficient": 1 },
          { "name": "Test 2", "score": 16.0, "max": 20, "coefficient": 1 },
          { "name": "Composition", "score": 15.5, "max": 20, "coefficient": 2 }
        ],
        "term_average": 15.25,
        "class_average": 12.8,
        "class_rank": 5,
        "class_size": 30,
        "grade_letter": "B+"
      }
    ],
    "attendance": {
      "present": 45,
      "absent_unexcused": 2,
      "absent_excused": 1,
      "late": 3
    },
    "conduct": "BON",
    "promotion_recommendation": "PASS"
  }
}
```

---

### 5.3 Timetable Endpoints

#### Get Teacher Schedule

```
GET /v1/teachers/{teacher_id}/schedule
```

**Query Parameters**:
- `date`: Specific date (defaults to today)
- `timetable_id`: Specific timetable (defaults to active)
- `expand_exceptions`: Include exceptions

**Response** (200):
```json
{
  "data": {
    "teacher": {
      "name": "M. Sow",
      "employee_number": "TCH001"
    },
    "date": "2025-03-31",
    "day_of_week": "MONDAY",
    "schedule": [
      {
        "period": 1,
        "start_time": "08:00",
        "end_time": "08:55",
        "classroom": "5EME-S-A",
        "subject": "Mathématiques",
        "room": "R102",
        "type": "REGULAR",
        "is_exception": false
      },
      {
        "period": 2,
        "start_time": "09:00",
        "end_time": "09:55",
        "classroom": "4EME-B",
        "subject": "Mathématiques",
        "room": "R102",
        "type": "REGULAR"
      },
      {
        "period": 3,
        "start_time": "10:15",
        "end_time": "11:10",
        "is_free_period": true,
        "notes": "Planning period"
      }
    ],
    "total_periods": 6,
    "total_teaching_periods": 4
  }
}
```

---

#### Get Student Daily Timetable

```
GET /v1/students/{student_id}/timetable
```

**Query Parameters**:
- `date`: Date (defaults to today)
- `include_room`: true/false

**Response** (200):
```json
{
  "data": {
    "student": "Amina Diallo",
    "classroom": "5EME-S-A",
    "date": "2025-03-31",
    "schedule": [
      {
        "period": 1,
        "start_time": "08:00",
        "subject": "Mathématiques",
        "teacher": "M. Sow",
        "room": "R102",
        "is_cancelled": false
      },
      {
        "period": 2,
        "start_time": "09:00",
        "subject": "Physique",
        "teacher": "Mme. Ba",
        "room": "LAB1",
        "is_cancelled": false
      },
      {
        "period": 3,
        "start_time": "10:15",
        "subject": "Français",
        "teacher": "M. Gueye",
        "room": "R205",
        "is_cancelled": true,
        "cancellation_reason": "Teacher absence - substitute scheduled for tomorrow"
      }
    ]
  }
}
```

---

### 5.4 Sync Endpoints

#### Sync Request (Offline Devices)

```
POST /v1/sync
Content-Type: application/json
X-Device-ID: <device_uuid>
X-Sync-Version: <last_sync_version>
```

**Request Body**:
```json
{
  "last_sync_version": 12345,
  "pending_changes": [
    {
      "idempotency_key": "local-operation-uuid",
      "operation": "INSERT",
      "table": "grade",
      "record": {
        "evaluation_id": "eval-uuid",
        "student_profile_id": "student-uuid",
        "value": 14.5,
        "justification": "Good work"
      }
    }
  ],
  "device_info": {
    "platform": "android",
    "app_version": "2.1.25",
    "storage_available_mb": 512
  }
}
```

**Response** (200):
```json
{
  "success": true,
  "data": {
    "new_sync_version": 12346,
    "timestamp": "2025-03-29T10:35:00Z",
    "server_changes": [
      {
        "operation": "UPDATE",
        "table": "evaluation",
        "record": { ... }
      }
    ],
    "conflicts": [
      {
        "local_change_id": "local-uuid",
        "table": "grade",
        "record_id": "grade-uuid",
        "conflict_type": "CONCURRENT_MODIFICATION",
        "server_value": { "value": 16.0 },
        "local_value": { "value": 15.5 },
        "resolution": "SERVER_WINS"  // or CLIENT_WINS, MANUAL_REVIEW
      }
    ],
    "rejected_changes": [
      {
        "idempotency_key": "...",
        "reason": "GRADE_EVALUATION_FINALIZED",
        "suggestion": "Use grade appeal process"
      }
    ],
    "next_sync_utc": "2025-03-29T12:00:00Z"
  }
}
```

---

### 5.5 User & Profile Endpoints

#### Get Current User

```
GET /v1/users/me
```

**Response** (200):
```json
{
  "data": {
    "id": "user-uuid",
    "email": "teacher@school.edu",
    "first_name": "Moussa",
    "last_name": "Sow",
    "full_name": "Moussa Sow",
    "avatar_url": "https://..."
  },
  "included": [
    {
      "type": "profile",
      "id": "teacher-profile-uuid",
      "attributes": {
        "profile_type": "teacher",
        "employee_number": "TCH001",
        "department": "Mathematics",
        "taught_subjects": ["MAT", "PHY", "CHE"]
      }
    }
  ]
}
```

---

### 5.6 Reporting Endpoints

#### Generate Report Card (Async)

```
POST /v1/report-cards
Content-Type: application/json
```

**Request Body**:
```json
{
  "enrollment_id": "enrollment-uuid",
  "term_code": "2024-2025-PRIM-T1",
  "include_class_ranking": true,
  "notify_parent": true,
  "template_id": "official_french"  // optional, school default if omitted
}
```

**Response** (202 Accepted):
```json
{
  "data": {
    "job_id": "report-job-uuid",
    "status": "PENDING",
    "estimated_completion_seconds": 30,
    "check_status_url": "/v1/report-cards/jobs/abc123"
  }
}
```

**Status Check**:
```
GET /v1/report-cards/jobs/{job_id}
```

**Response** (200 when complete):
```json
{
  "data": {
    "job_id": "...",
    "status": "COMPLETED",
    "report_card": {
      "id": "report-uuid",
      "enrollment_id": "...",
      "pdf_url": "/v1/report-cards/abc/download?token=...",
      "expires_at": "2025-03-30T10:00:00Z",
      "generated_at": "2025-03-29T10:32:00Z"
    }
  }
}
```

---

## 6. GraphQL Schema (Optional)

### 6.1 Schema Overview

```
type Query {
  # Current user
  me: User!

  # Enrollments
  enrollments(
    schoolYearCategoryId: UUID!
    classroomId: UUID
    state: EnrollmentState
  ): [Enrollment!]!

  # Grades
  studentGrades(
    studentId: UUID!
    termId: UUID
  ): [Grade!]!

  # Timetable
  timetable(
    timetableId: UUID!
  ): Timetable
  teacherSchedule(
    teacherId: UUID!
    date: Date
  ): DailySchedule!

  # Reports
  reportCard(
    enrollmentId: UUID!
    termCode: String!
  ): ReportCard!

  # Sync
  sync(
    input: SyncInput!
  ): SyncResponse!
}

type Mutation {
  # Enrollment
  createEnrollment(input: CreateEnrollmentInput!): Enrollment!
  activateEnrollment(id: UUID!, notes: String): Enrollment!
  withdrawEnrollment(id: UUID!, reason: WithdrawalReason!): Enrollment!

  # Grades
  enterGrade(input: EnterGradeInput!): Grade!
  submitEvaluation(id: UUID!): Evaluation!
  finalizeEvaluation(id: UUID!): Evaluation!
  createGradeAppeal(gradeId: UUID!, reason: String!): GradeAppeal!

  # Timetable
  generateTimetable(sycatId: UUID!, options: GenerationOptions!): Timetable!
  publishTimetable(id: UUID!): Timetable!

  # Sync
  pushChanges(changes: [LocalChange!]!): SyncResponse!
}

type Subscription {
  gradeUpdated(evaluationId: UUID!): Grade!
  enrollmentStateChanged(studentId: UUID!): Enrollment!
  timetablePublished(schoolYearCategoryId: UUID!): Timetable!
}
```

---

## 7. Webhooks & Event Subscriptions

### 7.1 Webhook Configuration

Integrations can register webhooks to receive real-time notifications:

```
POST /v1/webhooks
{
  "url": "https://integrator.example.com/webhooks/sms",
  "events": ["grade.finalized", "enrollment.activated"],
  "secret": "webhook-secret-for-signature-verification"
}
```

### 7.2 Event Payloads

**grade.finalized**:
```json
{
  "event": "grade.finalized",
  "timestamp": "2025-03-29T14:30:00Z",
  "signature": "sha256=...",
  "payload": {
    "grade_id": "...",
    "evaluation_id": "...",
    "student_id": "...",
    "value": 15.5,
    "evaluation_name": "Test 3 - Algebra",
    "teacher_id": "...",
    "finalized_by": "..."
  }
}
```

---

## 8. Rate Limiting

**Per-User Token Bucket**:
- Default: 100 requests per minute
- Background/job tokens: 1000 requests per minute
- Burst allowance: 20 additional requests

**Response Headers**:
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 85
X-RateLimit-Reset: 2025-03-29T11:00:00Z
Retry-After: 30  // on 429 response
```

---

## 9. Pagination

**Cursor-based pagination** (recommended for large datasets):
```
GET /v1/enrollments?school_year_category_id=...&cursor=eyJpZCI6IjEwMCJ9&limit=20
```

**Response**:
```json
{
  "data": [...],
  "meta": {
    "cursor": "eyJpZCI6IjIyMCJ9",  // base64 encoded last ID
    "has_more": true
  }
}
```

**Alternative**: Offset-based for simpler use cases:
```
GET /v1/enrollments?page=2&per_page=20
```

---

## 10. Filtering & Sorting

**Filtering** (query parameters):
```
GET /v1/enrollments?
  school_year_category_id=...
  &state=ACTIVE
  &level_id=...
  &created_after=2025-01-01
  &created_before=2025-03-31
```

**Sorting**:
```
GET /v1/enrollments?sort=created_at,-updated_at
  // ascending: created_at
  // descending: -updated_at
```

Allowed sort fields documented per endpoint.

---

## 11. File Uploads

### 11.1 Document Upload

```
POST /v1/enrollments/{id}/documents
Content-Type: multipart/form-data
```

**Request**:
```
------boundary
Content-Disposition: form-data; name="file"; filename="birth_certificate.pdf"
Content-Type: application/pdf

<binary data>
------boundary
Content-Disposition: form-data; name="document_type"

BIRTH_CERTIFICATE
------boundary--
```

**Response** (201):
```json
{
  "data": {
    "id": "doc-uuid",
    "filename": "birth_certificate.pdf",
    "document_type": "BIRTH_CERTIFICATE",
    "url": "/v1/documents/abc/download?token=...",
    "uploaded_at": "2025-03-29T10:30:00Z",
    "verified": false
  }
}
```

---

### 11.2 S3-Style Upload (Large Files)

```
1. Request upload URL:
POST /v1/uploads/request-url
{
  "filename": "large-document.pdf",
  "content_type": "application/pdf",
  "size_bytes": 5242880
}

Response: { "upload_url": "https://storage.googleapis.com/...", "access_token": "..." }

2. Upload directly to storage using returned URL (PUT with token)

3. Confirm upload:
POST /v1/uploads/confirm
{
  "upload_token": "...",
  "document_type": "TRANSCRIPT",
  "linked_record_id": "enrollment-uuid"
}
```

---

## 12. Error Handling Reference

### 12.1 HTTP Status Codes

| Code | Meaning | Usage |
|------|---------|-------|
| 200 OK | Success | GET/PUT/PATCH successful |
| 201 Created | Resource created | POST successful |
| 204 No Content | Success, no body | DELETE successful |
| 400 Bad Request | Validation error | Request body malformed |
| 401 Unauthorized | No/invalid auth | Missing/invalid token |
| 403 Forbidden | Insufficient permission | Valid auth, no permission |
| 404 Not Found | Resource missing | Invalid ID |
| 409 Conflict | Business conflict | Duplicate, capacity exceeded |
| 422 Unprocessable Entity | Business rule violation | Age ineligible, etc. |
| 429 Too Many Requests | Rate limit exceeded | Retry after header |
| 500 Internal Server Error | Server error | Unexpected condition |
| 503 Service Unavailable | Maintenance/backoff | Retry later |

### 12.2 Error Codes Catalog

| Code | HTTP | Description | Retryable |
|------|------|-------------|-----------|
| VALIDATION_ERROR | 400 | Input validation failed | No |
| AUTH_INVALID_CREDENTIALS | 401 | Wrong password/email | No |
| AUTH_MFA_REQUIRED | 401 | MFA challenge needed | No |
| PERMISSION_DENIED | 403 | Lacks required role | No |
| RESOURCE_NOT_FOUND | 404 | Record doesn't exist | No |
| ENROLLMENT_DUPLICATE | 409 | Already enrolled | No |
| CLASSROOM_CAPACITY_EXCEEDED | 409 | No seats | No (admin may increase) |
| CONCURRENT_MODIFICATION | 409 | Optimistic lock failed | Yes (with backoff) |
| GRADE_FINALIZED_IMMUTABLE | 409 | Cannot modify | No (use appeal) |
| SYNC_CONFLICT | 409 | Server version differs | Yes (notify user) |
| RATE_LIMIT_EXCEEDED | 429 | Too many requests | Yes |
| SERVICE_UNAVAILABLE | 503 | Temporarily down | Yes |

---

## 13. API Version History

### Version 1.0 (Current)

**Released**: 2025-03-29

**Endpoints**:
- Enrollment: CRUD + activate, withdraw
- Grades: enter, submit, finalize, appeals
- Timetable: get schedule, generate, publish
- Users: profile, permissions
- Reports: report cards, transcripts
- Sync: full offline sync
- Search: basic student/teacher lookup

**Breaking Changes from v0**:
- Enrollment ID renamed from `id` to `enrollment_id` in responses (for clarity)
- Grade submission renamed from `/grades/submit` to `/evaluations/{id}/submit`
- Removed deprecated `school` entity (using `tenant` instead)

---

## 14. Contracts for Offline-First

### 14.1 Idempotency Keys

All write operations support `X-Idempotency-Key` header:
```
X-Idempotency-Key: abc123-uuid
```

Server stores processed keys for 24 hours. Duplicate request with same key returns cached response.

### 14.2 ETag & Conditional Requests

GET responses include ETag:
```
ETag: "W/\"abc123hash\""
```

Client can use `If-None-Match` to get 304 Not Modified.

### 14.3 Sync Checkpointing

```
GET /v1/sync/status?device_id=...
```

Returns:
```json
{
  "last_sync_version": 12345,
  "last_sync_at": "2025-03-29T10:30:00Z",
  "pending_server_changes": 5,
  "recommended_sync_interval_seconds": 300
}
```

---

## 15. Performance Expectations

| Endpoint | Target Latency (p95) |
|----------|---------------------|
| GET /users/me | < 50ms |
| GET /enrollments (list, 20 items) | < 200ms |
| POST /grades (enter) | < 150ms |
| GET /teachers/{id}/schedule | < 100ms |
| POST /sync (incremental) | < 500ms |
| POST /report-cards (async start) | < 50ms |

---

## 16. API Security

### 16.1 HTTPS Only
All endpoints require TLS 1.2+.

### 16.2 CORS Policy
```
Access-Control-Allow-Origins: https://app.schoolsync.io, https://admin.schoolsync.io
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, PATCH
Access-Control-Allow-Headers: Authorization, Content-Type, X-Idempotency-Key
```

### 16.3 Input Validation
- All inputs validated server-side
- SQL injection prevention via parameterized queries
- JSON schema validation for request bodies

---

## 17. SDK Generation Guidelines

**Auto-generation** from OpenAPI spec:
- Generate client libraries for: JavaScript/TypeScript, Python, Java, Kotlin, Swift, Dart
- Authentication helper included
- Retry logic with exponential backoff
- Offline queue support (for mobile SDKs)

**OpenAPI Spec**: Stored at `/api-specs/v1/openapi.yaml`

---

## 18. Testing & Validation

### 18.1 Contract Testing

Use **Postman Collections** and **Pact** for consumer-driven contract tests.

### 18.2 Mock Server

```
GET /v1/_health → 200 {status: "healthy"}
GET /v1/_mock/enrollments → sample data for UI development
```

---

**Next Step**: TDR-011 — Selectors & Queries (data access patterns, repository layer)
