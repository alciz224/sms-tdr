# TDR-013 — Service Scenarios

## 1. Purpose and Scope

This document presents **narrative use cases** that illustrate how the system's services, domain rules, and offline capabilities work together in real-world scenarios.

These scenarios serve as:
- **Acceptance criteria** for implementation
- **Training material** for users and administrators
- **Integration test specifications**
- **Communication tool** for stakeholders to understand system behavior

**Key Scenarios Covered**:
1. New student registration (online + offline)
2. Teacher entering grades during connectivity loss
3. Year-end promotion process
4. Timetable conflict resolution
5. Mid-year student transfer
6. Grade appeal workflow
7. Offline-first daily routine
8. Emergency school closure

---

## 2. Scenario 1: New Student Registration (Aminata's Enrollment)

### 2.1 Context

**Characters**:
- Aminata Diallo, age 7 (potential student for CP/1st grade)
- Mama Diallo (parent, registered user with ParentProfile)
- Registrar (school administrator)

**Situation**: Aminata's family moves to Dakar. Mama Diallo wants to enroll her at École Primaire de Médina for the 2024-2025 school year. She has internet access at home but the school office is closed today.

### 2.2 Happy Path (Online Registration)

```
[DAY 1 - At home, with internet]

1. Mama Diallo logs into the parent portal (mobile app)
   → Authentication successful
   → Loads her ParentProfile with linked children (none yet)

2. She clicks "Enroll New Student"
   → System checks: User has PARENT role, SCHOOL_PARENT permission
   → Shows available SchoolYears: "2024-2025" (ACTIVE enrollment period)

3. She selects "2024-2025" and Category "Primaire"
   → System validates: SchoolYearCategory exists, is in DRAFT/ACTIVE state
   → Displays available Levels for age 7: CP, CE1

4. She selects "CP" (based on age)
   → System confirms age eligibility: DOB 2017-03-15 → age 7.5 → OK for CP
   → Shows available Tracks: (none for Primaire, track not applicable)

5. She uploads documents:
   - Birth certificate (PDF)
   - Immunization record (PDF)
   - Previous school transcript (from old city)

6. System validates document types, file size
   → Documents stored in secure storage
   → Enrollment state = PENDING

7. System checks classroom availability:
   CP classrooms: 3 sections (A, B, C)
   - CP A: 28/30 students → available
   - CP B: 30/30 → full
   - CP C: 25/30 → available

   → Assigns CP A (first available)
   → Enrollment.state = PENDING, classroom_id = CP A

8. System calculates tuition:
   Base: 300,000 XOF
   Less: 0% scholarship
   Total: 300,000 XOF

9. Mama Diallo pays via integrated payment:
   → Payment successful
   → Enrollment.fee_status = PAID

10. She clicks "Activate Enrollment"
    → RegistrarService.activateEnrollment() executes:
      a. Validates prerequisites: documents ✓, fees ✓
      b. Checks capacity: CP A 28/30 → OK
      c. Transaction:
         - Enrollment.state = ACTIVE
         - classroom.current_enrollment_count = 29
         - Create ClassroomSubject assignments for CP A (Math, French, etc.)
         - Generate StudentIdentifier (QR code)
      d. Commit

11. Confirmation screen:
    Enrollment Number: 2024-2025-000156
    Student ID: STU00256
    QR Code: [displayed]
    Classroom: CP A
    Teacher: Mme. Diop
    First day: September 2, 2024

12. Notifications sent:
    - Email to Mama: "Aminata enrolled successfully"
    - SMS: "École: Aminate inscrite en CP A. Voir QR code."
    - Teacher Mme. Diop: "New student added to CP A roster"

Result: ✅ Complete online enrollment in 15 minutes
```

---

### 2.3 Offline Registration (Network Outage Scenario)

```
[DAY 2 - School office, internet down]

Registrar needs to process enrollment in person.
Internet provider outage. School's offline app is ready.

1. Registrar logs into offline app
   → Last sync: today 8:00 AM (before outage)
   → All reference data cached
   → All active classrooms cached

2. Mama Diallo arrives with documents
   Registrar creates enrollment offline:

   a. Select student (create CustomUser + StudentProfile)
   b. Select SchoolYear: 2024-2025
   c. Select Level: CP
   d. System checks: Age OK, no duplicate enrollment ✓
   e. System finds available classrooms (from cached data):
      CP A: 28/30 → assign
   f. Upload documents (stored locally, queued for sync)
   g. Process payment (local queue, will sync)

3. Registrar clicks "Activate"
   → Enrollment saved locally with state = ACTIVE
   → Classroom count incremented locally (29)
   → Operation queued for sync: {idempotency_key: ..., table: enrollment, operation: INSERT}
   → UI shows: "Saved. Will sync when internet available."

4. Mama leaves with enrollment certificate (printed from local data)

5. [Later that day, internet restored]
   → Sync automatically triggered
   → POST /sync sends all pending changes

   Server processes:
   a. Enrollment INSERT → valid, classroom capacity still OK
   b. Documents uploaded → linked to enrollment
   c. Payment record created

   Response: SUCCESS, new_sync_version = 12350

6. Local device:
   → Marks changes as synced
   → Downloads any server changes (none)
   → Sends notification to Mama: "Enrollment confirmed"

Result: ✅ Offline enrollment seamless, user unaware of connectivity issue
```

---

## 3. Scenario 2: Grade Entry During Connectivity Loss (Teacher Workflow)

### 3.1 Context

**Characters**:
- Moussa Sow (Mathematics teacher, 5ème S class)
- Student: Ibrahima Diop

**Situation**: Mid-term test (Test #3) conducted on March 27. Moussa grades papers at home that evening. His home internet is spotty (frequent outages).

### 3.2 Offline Grade Entry

```
[March 27, 8:00 PM - At home, internet intermittently available]

1. Moussa opens teacher app
   → Authenticates with cached credentials
   → Loads his schedule (cached timetable)

2. Navigates to: Class 5ème S → Test 3 → Enter Grades

3. App shows grade entry grid:
   Students: [Ibrahima, Aminata, Oumar, ...]

4. Moussa enters grades:
   - Ibrahima: 14.5
   - Aminata: 16.0
   - Oumar: 12.0 (absent with note: "malaria")
     → Marks as EXEMPT, reason: "Medical - malaria"
   - 27 other students...

   Total: 30 students, 29 with grades, 1 exempt

5. Moussa hits "Submit"
   → App checks: all required grades entered
   → Network check: OFF ⚠

6. App shows notification:
   "Cannot submit while offline. Grades saved locally.
   They will submit automatically when you reconnect."

   Action: Grades saved in local grade table with state = ENTERED
           Operation queued: {table: grade, operation: INSERT, ...} x 30
           Evaluation submission queued: {table: evaluation, operation: UPDATE, state: SUBMITTED}

7. [March 28, 6:00 AM - Internet restored, device on Wi-Fi]
   → Sync triggered automatically

8. Sync request sent:
   Upload: 31 changes (30 grades + 1 evaluation state update)
   Device last_sync_version: 12340

9. Server processes:
   a. Validates each grade:
      - Evaluation exists: ✓
      - Student enrolled: ✓
      - Values 0-20: ✓
      - Exempt has reason: ✓
   b. Updates evaluation:
      total_students_completed = 29 (1 exempt)
      state = SUBMITTED
      submitted_at = NOW()
   c. All 30 grades: state = ENTERED
   d. Increment sync version to 12341

10. Server response:
    SUCCESS, 31 changes applied
    No conflicts

11. Device:
    → Marks all 31 operations as synced
    → Updates local state: evaluation.state = SUBMITTED
    → Clears offline queue
    → Sends notification: "Test 3 grades submitted successfully"

Result: ✅ Teacher grades offline, submits seamlessly when connectivity returns
```

---

### 3.3 Conflict Scenario: Teacher vs Academic Council

```
[March 30 - Academic Council reviewing submitted evaluations]

Registrar notices: Ibrahima's grade was entered as 14.5, but his test showed copying.
Registrar (with can_modify_grades permission) amends grade on server:

1. Registrar finds evaluation in admin portal
2. Selects Ibrahima's grade
3. Changes 14.5 → 10.0 (with note: "Academic integrity violation")
4. Clicks "Amend Grade"
   → Grade amendment created (original 14.5 preserved)
   → Grade record updated: value = 10.0, is_amendment = true
   → Sync version incremented to 12345

5. [Later that day, teacher's device syncs]
   Teacher's device had old cached grade value (14.5) before amendment

6. Sync detects conflict:
   Local: grade.value = 14.5, grade.sync_version = 12340
   Server: grade.value = 10.0, grade.sync_version = 12345
   Conflict type: CONCURRENT_MODIFICATION

7. Resolution (server wins for amendments):
   Conflict resolution: SERVER_WINS
   Reason: "Grade amendment by academic council"

8. Device:
   → Overwrites local grade with server value (10.0)
   → Marks conflict resolved
   → Teacher notified: "Ibrahima's grade was modified by registrar: 14.5 → 10.0. Reason: Academic integrity violation."

9. Teacher opens app, sees corrected grade
   Grade amendment visible in audit trail

Result: ✅ Conflict resolved server-authoritative, teacher informed
```

---

## 4. Scenario 3: Year-End Promotion Process

### 4.1 Context

**Academic Year**: 2024-2025
**SchoolYearCategory**: Collège (6ème, 5ème, 4ème, 3ème)
**Time**: May 2025 (final term closing)

**Characters**:
- Principal (reviews promotion decisions)
- Academic Council ( approves/reviews borderline cases)
- Teachers (finalized grades)
- Students & Parents (awaiting results)

### 4.2 Promotion Workflow

```
PHASE 1: Grade Finalization (April 15 - May 20)

[April 15]
All teachers submit term 3 (final) evaluations
→ EvaluationService.submitEvaluation() → state = SUBMITTED
→ Academic Council notified: "Evaluations pending review"

[April 20 - May 10]
Academic Council reviews all SUBMITTED evaluations:
  1. Checks for completeness (all students graded)
  2. Reviews any low grades with justification
  3. Approves or returns for correction

For 5ème S:
   - 30 students
   - All evaluations SUBMITTED
   - Council reviews: no issues found
   → All evaluations → FINALIZED
   → Report cards generated (async batch)

[May 10] All evaluations for Collège 2024-2025 are FINALIZED

────────────────────────────────────────────────────────

PHASE 2: Promotion Calculation (May 15)

[May 15, 9:00 AM]
Principal runs: "Compute Promotion Decisions" for Collège 2024-2025

1. PromotionService triggered:
   FOR EACH ACTIVE enrollment in Collège 2024-2025:
     ┌────────────────────────────────────────┐
     │ Student: Ibrahima Diop (5ème S)       │
     │                                        │
     │ Fetch term averages:                   │
     │   T1: 12.8                            │
     │   T2: 14.2                            │
     │   T3: 13.5                            │
     │                                        │
     │ Year average (weighted):              │
     │   (12.8×1 + 14.2×1 + 13.5×1) / 3     │
     │   = 13.50                            │
     │                                        │
     │ Absences: 3 unexcused (limit: 5) ✓    │
     │                                        │
     │ Promotion rules for 5ème → 4ème:      │
     │   Pass threshold: 10.00               │
     │   Retention threshold: 8.00           │
     │                                        │
     │ Decision: PASS ✓                      │
     │ Next level: 4ème S                    │
     └────────────────────────────────────────┘

     ┌────────────────────────────────────────┐
     │ Student: Aminata Diallo (5ème S)      │
     │                                        │
     │ Year average: 8.75                    │
     │ Absences: 8 unexcused ✗               │
     │                                        │
     │ 8.75 < 10.00 → Below pass threshold  │
     │ 8 > 8.00 → Not auto-retain           │
     │ → Requires council review             │
     │                                        │
     │ Decision: REVIEW (manual decision)   │
     └────────────────────────────────────────┘

   Results:
   - PASS: 25 students → promotion_to_4ème S
   - RETAIN: 3 students (below 8.0)
   - REVIEW: 2 students (8.0-10.0, consider absences)

2. Council meets to review REVIEW cases:
   Aminata: 8.75 average, 8 absences (some excused medical)
   → Council votes: PASS (medical circumstances considered)

   Oumar: 9.2 average but 2 failing critical evaluations (Physics)
   → Council votes: RETAIN (needs to repeat Physics)

3. Council updates enrollment.promotion_decision:
   Aminata: REVIEW → PASS
   Oumar: REVIEW → RETAIN

4. System generates promotion report:
   - 27 students promoted to 4ème S
   - 2 students retained in 5ème S
   - 1 student transferring to another school

────────────────────────────────────────────────────────

PHASE 3: Communication (May 20)

[Automated notifications sent]
- Parents of promoted students:
  "Congratulations! {student} promoted to 4ème S. Re-enroll for 2025-2026."

- Parents of retained students:
  "{student} will repeat 5ème S. Please schedule meeting with academic counselor."

- Students transferring out:
  "Transcript and report card prepared. Good luck at your new school."

────────────────────────────────────────────────────────

PHASE 4: Next Year Enrollment (June 1)

[Automated process]
For each promoted student:
  1. Create next year's enrollment (2025-2026):
     enrollment.school_year_category = next year's Collège
     enrollment.level_id = promotion_to_level_id (4ème S)
     enrollment.state = PENDING (needs parent confirmation)
  2. Send re-enrollment invitation to parent

For retained students:
  1. Enrollment for 2025-2026:
     enrollment.level_id = same level (5ème S again)

Result: ✅ Promotion process automated, council-reviewed, communicated
```

---

## 5. Scenario 4: Timetable Conflict Resolution (Teacher Scheduling)

### 5.1 Context

**SchoolYear**: 2024-2025
**SchoolYearCategory**: Collège
**Character**: Moussa Sow (Math teacher)
**Problem**: Moussa teaches 5ème S Math (4 periods/week) and 4ème S Math (3 periods/week). Automatic timetable generation creates conflict.

### 5.2 Automatic Generation with Conflicts

```
[May 1 - Admin generates timetable]

1. Admin clicks "Generate Timetable" for Collège 2024-2025
   → TimetableService.generateTimetable() called

2. Algorithm loads requirements:
   ClassroomSubjects needed:
   - 5ème S Math: 4 hours/week
   - 4ème S Math: 3 hours/week
   - Teacher: Moussa Sow (max 20 hours/week)
   - Available slots: Mon-Fri periods 1-6

3. Assignment attempts:
   a. 5ème S Math periods 1-2 assigned (Monday)
   b. 4ème S Math periods 1-2 attempted Monday → CONFLICT (teacher busy)
   c. Tries Tuesday periods 1-2 → CONFLICT (teacher's physics lab)
   d. Tries Wednesday periods 1-3 → OK (available)

4. Continue for all subjects...
   Some subjects fail to schedule:
   - Group 3ème History: needs 3 hours, no slots
   - 2nde Practical Science: needs lab, lab booked

5. Result: Timetable with 85% coverage, 5 unscheduled subjects

6. Admin reviews conflict report:
   Unscheduled:
   - 3ème Histoire-Géographie (3 hrs) - no teacher availability
   - 2nde Physique Pratique (2 hrs) - lab conflict
   - 4ème S Anglais (2 hrs) - teacher leave conflict

7. Admin resolves manually:
   a. History: switch teacher (Mme. Ba available)
   b. Practical Science: reschedule lab session to Friday afternoon
   c. English: adjust teacher schedule (trade with colleague)

8. After adjustments, re-validate:
   ✓ All subjects scheduled
   ✓ No teacher conflicts
   ✓ No room conflicts
   ✓ Weekly hours per subject match requirements

9. Publish timetable:
   Timetable.status = ACTIVE
   Invalidate previous timetable
   Push to all devices

Result: ✅ Timetable generated, conflicts identified and resolved manually
```

---

## 6. Scenario 5: Mid-Year Student Transfer (Incoming)

### 6.1 Context

**Student**: Oumar Diop (currently in 5ème S at另一所学校)
**Transfer reason**: Family relocation
**Sending school**: Lycée de Biscuiterie
**Receiving school**: École Primaire de Médina (our school)
**Timeline**: October 15 (mid-first term)

### 6.2 Transfer-In Workflow

```
[October 10 - Receiving school registrar receives transfer request]

1. Sending school submits transfer request via secure API:
   {
     "student_id": "STU-OLDSCH-789",
     "from_school": "Lycée Biscuiterie",
     "to_school": "École Médina",
     "requested_school_year": "2024-2025",
     "requested_level": "5ème",
     "requested_track": "S",
     "transfer_date": "2024-10-15",
     "reason": "Family relocation",
     "documents": [transcript, attendance, health records]
   }

2. Receiving registrar reviews:
   - Check: 5ème S has capacity? Yes (29/30)
   - Check: Student's transcript OK? Yes (passing grades)
   - Check: No disciplinary holds? Yes

3. Registrar initiates enrollment:
   a. Create StudentProfile (or find existing if student was previously enrolled)
      → New student record created
      → Link to parent (Mama Diallo exists in system from Aminata!)

   b. Enrollment:
      school_year_category: Collège 2024-2025
      level: 5ème (based on transcript)
      track: S (confirmed by transcript)
      classroom: 5ème S A (current class)
      state: BLOCKED (pending transfer validation)

4. Registrar requests sending school to release records:
   → System sends secure message to sending school
   → Sending school approves, uploads transcript

5. [October 14] Records received and validated:
   - Transcript: 5ème grades (13.2 average)
   - All evaluations transferred with grades
   - Student meets promotion criteria

6. Registrar clicks "Complete Transfer":
   Enrollment.state → ACTIVE
   classroom.current_enrollment_count: 29 → 30
   StudentIdentifier created
   Student added to class roster
   Notification sent to teacher (Moussa Sow): "New transfer student"

7. Notifications:
   - Parent: "Oumar enrolled in 5ème S A. Classes start Oct 16."
   - Student (if has device): "Welcome! Your schedule is ready."

8. Grade transfer:
   Previous grades (from sending school) loaded with source marking:
   - "5ème S at Lycée Biscuiterie" as separate semester
   - Applied to year average calculation
   - Transcript shows both schools

Result: ✅ Transfer processed in 5 days, student integrated mid-term

```

---

## 7. Scenario 6: Grade Appeal Process (Parent Concern)

### 7.1 Context

**Student**: Aminata Diallo (5ème S)
**Evaluation**: Test 4 - Algebra (coefficient 2)
**Grade received**: 8.0 (below passing)
**Parent concern**: "My child studied, deserves better"

### 7.2 Appeal Workflow

```
[May 5 - Report card distributed]

1. Mama Diallo reviews report card:
   Mathematics: 8.0 (Test 4)
   Comment: "Incomplete solutions, need more practice"

   Mama believes grade unfair.

2. Mama logs into parent portal (within 30-day appeal window)
   → Navigates to Aminata's grades
   → Clicks "Appeal Grade" next to Test 4

3. Appeal form:
   - Grade to appeal: Test 4 - Mathematics (grade: 8.0)
   - Reason: "Student completed all problems. Teacher may have miscounted."
   - Attachments: [Aminata's test paper with solutions]
   - Requested action: "Regrade"

4. System validates:
   ✓ Grade is FINALIZED
   ✓ Within 30 days of report card
   ✓ Max appeals per year not exceeded

5. Create GradeAppeal:
   state = PENDING
   grade_id = original grade
   student_id = Aminata
   appeal_date = 2025-05-05
   requested_action = REGRADE
   status = PENDING

6. Notifications:
   - Academic Council chair: "New grade appeal: Aminata Diallo, Math Test 4"
   - Teacher (Moussa Sow): "Your grade has been appealed"
   - Parent: "Appeal submitted, review within 14 days"

────────────────────────────────────────────────────────

[May 6 - Teacher reviews]

7. Moussa Saw appeal:
   Reviews Aminata's test paper
   Realizes grading error: missed 2 problems → actual grade should be 11.5

8. Moussa submits review:
   - Approval: GRANTED
   - Notes: "Grading error discovered. Original grade incorrect."
   - New grade proposed: 11.5

────────────────────────────────────────────────────────

[May 7 - Academic Council finalizes]

9. Council reviews teacher's proposed amendment:
   ✓ Teacher acknowledges error
   ✓ Proposed grade (11.5) supported by evidence
   → Approves grade amendment

10. Amendment executed:
    Original grade: value = 8.0, state = FINALIZED, is_amendment = false
    New grade: value = 11.5, state = FINALIZED, is_amendment = true,
              amends_grade_id = original grade id

    GradeAppeal.state = APPROVED
    grade_appeal.amendment_grade_id = new grade id

11. Update cascades:
    - Student term average recalculated: 8.0 → 9.8
    - Report card cache invalidated
    - New report card generated

12. Notifications:
    - Parent: "Appeal approved. Test 4 grade corrected: 8.0 → 11.5"
    - Teacher: "Grade amendment recorded"
    - Registrar: "Grade change requires transcript update"

Result: ✅ Appeal resolved fairly, original preserved, corrected grade applied
```

---

## 8. Scenario 7: Offline-First Daily Routine (Teacher Perspective)

### 8.1 Context

**Teacher**: Moussa Sow
**Device**: Android tablet with offline app
**Connectivity**: Unreliable at home, good at school, no cellular in classrooms

### 8.2 Daily Workflow

```
[6:30 AM - At home, no internet]

1. Moussa wakes up, checks tablet
   → App shows: "Offline - 100% charge"
   → Last sync: yesterday 5:30 PM

2. Reviews today's schedule (cached):
   Period 1 (8:00-8:55): 5ème S - Mathematics (R102)
   Period 2 (9:00-9:55): 4ème S - Mathematics (R103)
   Period 3: Free period
   Period 4: 5ème S - Mathematics Lab (LAB1)
   Afternoon: Planning

3. Lesson plan:
   - 5ème S: Quadratic equations review
   - 4ème S: Chapter 3 quiz

   Notes are cached locally, accessible offline.

────────────────────────────────────────────────────────

[8:00 AM - At school, classroom R102]

4. Takes attendance using tablet:
   → Attendance app loads class roster (cached)
   → Marks absent students (network not needed)
   → Saves locally: attendance records queued for sync

   When connectivity available at break:
   → Sync uploads attendance
   → Parents notified of absences

────────────────────────────────────────────────────────

[9:00 AM - 4ème S, giving quiz]

5. Midway through quiz, realizes:
   Question 3 has typo. Need to adjust grading.

   Opens evaluation on tablet:
   → Evaluation data cached (questions, max_score)
   → Makes note: "Q3: ignore, max_score reduced by 1 point"

────────────────────────────────────────────────────────

[10:00 AM - Break room, Wi-Fi available]

6. Connects to school Wi-Fi, sync triggered:
   → Attendance uploaded ✓
   → Note saved to evaluation (requires server update)
     Problem: evaluation already PUBLISHED → modification rejected
   → Conflict detected: "Cannot modify published evaluation"
   → Moussa notified: "Evaluation locked. Create new version instead."

────────────────────────────────────────────────────────

[1:00 PM - After classes, planning period]

7. Enters quiz grades for 4ème S (28 students):
   → Grade grid loads cached student roster
   → Enters scores (0-20) offline
   → Scores saved locally

   Developer note: Grade entry doesn't require server validation while offline.
   State = ENTERED locally.

8. After entering all grades:
   App shows: "28 grades entered. Submit when online."

────────────────────────────────────────────────────────

[2:30 PM - Back in classroom with Wi-Fi]

9. Wi-Fi restored, sync runs automatically:
   → 28 grade INSERT operations uploaded
   → Server validates:
      • All students enrolled ✓
      • Values 0-20 ✓
      • Quiz has 28 expecte ✓
   → All accepted
   → Evaluation.completion_percentage = 100%

10. Clicks "Submit Evaluation" in app:
    → POST /evaluations/{id}/submit
    → Server: evaluation.state = SUBMITTED
    → Academic Council notified

────────────────────────────────────────────────────────

[5:00 PM - At home, checking status]

11. Opens tablet:
    → Quiz grades: SUBMITTED (awaiting council review)
    → Report: "7 students below 10, 5 students above 15"

12. Receives notification:
    "Your submission passed review. Grades finalized."
    → App updates local state to FINALIZED

Result: ✅ Complete teaching day offline-capable, seamless sync when available
```

---

## 9. Scenario 8: Emergency School Closure (Snow Day / Strike)

### 9.1 Context

**Event**: Unexpected teacher strike starting at 10:00 AM
**Students**: Already in class, need to be sent home
**Communication**: Urgent notification to parents

### 9.2 Emergency Response Workflow

```
[9:30 AM - Announcement received]

1. Principal logs into admin portal
   → "Emergency Closure" button available

2. Principal clicks "Emergency Closure":
   - Select: Today (March 29)
   - Reason: Teacher strike (predefined option)
   - Message: "Students dismissed at 10:00 AM. Parents please pick up."

3. System creates TimetableException for today:
   - All lessons scheduled periods 2-6 marked CANCELLED
   - Original entries preserved for record

4. Emergency notification broadcast:
   → SMS to all parents: "École fermée à 10h. Votre enfant sera renvoyé à 10h."
   → Push notification to all app users
   → Email to all contacts
   → Website banner

5. System updates student dismissal:
   For each classroom:
   • Generate early dismissal list
   • Bus routes: adjust departure time
   • After-school programs: notify cancellation

6. Attendance adjustment:
   Students present before 10:00: marked PRESENT for morning
   Students absent after 10:00: marked DISMISSED (not absent)

7. Sync to all devices:
   TimetableException pushed to teacher tablets
   → Teachers see: "Periods 2-6 cancelled"
   → No attendance taken for cancelled periods

────────────────────────────────────────────────────────

[March 30 - Reopening]

8. School reopens, system restores normal schedule
   → TimetableException for March 29 archived
   → Attendance system allows normal reporting

9. Report generated for strike day:
   - Days missed: 0.5 (affects required instructional days)
   - Make-up schedule planned for later

Result: ✅ Emergency handled efficiently, parents notified, records accurate
```

---

## 10. Scenario 9: Real-Time Classroom Dashboard (Administrator View)

### 10.1 Context

**Role**: School Principal
**Objective**: Monitor school-wide academic progress during active term

### 10.2 Dashboard Query Patterns

```
[Principal opens "Academic Dashboard" in admin portal]

1. Dashboard loads key metrics (cached, real-time where needed):
   ┌─────────────────────────────────────────────┐
   │ School Year: 2024-2025 (ACTIVE)            │
   │ Current Term: Trimestre 2 (OPEN)           │
   │                                             │
   │ Total Students: 420                        │
   │ Average Attendance: 94.2%                  │
   │ Evaluations Finalized: 65/120 (54%)       │
   └─────────────────────────────────────────────┘

2. Principal clicks "See details" → Drill down:

   Query 1: Submissions by evaluation status
   SELECT state, COUNT(*) FROM evaluation
   WHERE school_year_category_term in current term
   GROUP BY state
   → {DRAFT:5, PUBLISHED:15, SUBMITTED:40, REVIEW:10, FINALIZED:50}

   Shows: 40 evaluations ready for council review → attention needed

3. Click "See pending reviews"

   Query 2: Pending evaluations needing attention
   SELECT e.name, c.code as classroom, COUNT(g.id) as grades_in
   FROM evaluation e
   JOIN classroom_subject cs ON e.classroom_subject_id = cs.id
   JOIN classroom c ON cs.classroom_id = c.id
   WHERE e.state IN ('SUBMITTED', 'REVIEW')
     AND e.school_year_category_term_id = current_term
   GROUP BY e.id, c.code
   ORDER BY e.submitted_at ASC

   Shows list: Principal prioritizes oldest first

4. Principal reviews "5ème S - Test 3 - Statistics" (submitted 3 days ago)
   → Opens evaluation
   → Sees grade distribution:
       Average: 11.2, Min: 4.0, Max: 18.5
       15 students below 10 → below expectations
   → Approves (no issues)

5. Navigates to "At-risk students" tab:

   Query 3: Students with term average < 8.0
   SELECT s.name, c.code as class,
          ROUND(AVG(g.value), 1) as term_avg
   FROM enrollment e
   JOIN student_profile s ON e.student_profile_id = s.id
   JOIN classroom c ON e.classroom_id = c.id
   JOIN grade g ON e.id = g.enrollment_id
   WHERE e.school_year_category_id = current_syc
     AND g.state = 'FINALIZED'
   GROUP BY s.id, c.code
   HAVING AVG(g.value) < 8.0
   ORDER BY term_avg ASC

   Results: 12 students flagged
   → Principal assigns academic support to these students

6. Checks subject performance:

   Query 4: Subject averages across all classes
   SELECT sub.code, sub.name,
          ROUND(AVG(g.value), 1) as school_avg,
          COUNT(*) as grades_count
   FROM subject sub
   JOIN classroom_subject cs ON sub.id = cs.subject_id
   JOIN evaluation e ON cs.id = e.classroom_subject_id
   JOIN grade g ON e.id = g.evaluation_id
   WHERE e.school_year_category_term_id = current_term
     AND g.state = 'FINALIZED'
   GROUP BY sub.id
   ORDER BY school_avg ASC

   Finds: Physics average = 9.8 (lowest)
   → Reviews physics class schedules, teacher assignments

7. Exports report:
   → "Academic Progress Report - Term 2"
   → PDF generated with charts
   → Sent to school board

Result: ✅ Data-driven administrative decisions enabled by performant queries
```

---

## 11. Scenario 10: Device Recovery After Data Loss

### 11.1 Context

**Teacher**: Moussa Sow
**Problem**: Tablet factory reset, all local data lost
**Available**: Teacher account credentials, new device

### 11.2 Recovery Flow

```
[Device lost, teacher gets replacement]

1. Moussa installs new app
   → Login screen

2. Enters credentials:
   email: moussa.sow@ecole.sn
   password: [entered]

3. System authenticates:
   ✓ User exists
   ✓ Password correct
   ✓ Account active

4. First device detection:
   Server checks: No device registered for this user+this device_id

5. Registration flow:
   a. User: "This is my new device, recover my data"
   b. Server: Requires MFA (TOTP)
      → Moussa enters code from authenticator app
   c. Device registered:
      INSERT INTO device_registry (device_id, user_id, last_seen)

6. Sync initiated automatically:
   Request: GET /v1/sync/full?device_id=new-device

7. Server determines minimal dataset for teacher:
   - Reference data (Regions, Subjects, etc.)
   - Current SchoolYear + Terms
   - Active Timetable (teacher's schedule)
   - Classroom rosters (teacher's classes)
   - Students in those classes (names, basic info)
   - Evaluations (teacher's own)
   - Grades (teacher entered)

   Estimated size: ~2MB compressed

8. Streaming response:
   Chunk 1: Reference data
   Chunk 2: School structure
   Chunk 3: Timetable + schedule
   Chunk 4: Classroom rosters
   Chunk 5: Evaluations
   Chunk 6: Grades (if requested)

9. Device receives, decrypts, stores locally
   → Progress indicator: "Downloading your data... 75%"

10. Sync complete:
    last_sync_version set to current server version
    "Recovery complete. 3 classes, 90 students loaded."

11. Moussa opens app:
    → Sees his 5ème S and 4ème S classes
    → Can view rosters, enter grades, view schedule
    → All data restored

Result: ✅ Teacher recovered after device loss in under 5 minutes
```

---

## 12. Cross-System Integration Scenarios

### 12.1 Payment Gateway Integration

```
[Invoice generation for enrollment]

1. EnrollmentService.activateEnrollment() creates FeeInvoice:
   amount: 300,000 XOF
   due_date: 2024-09-15
   student: Aminata Diallo

2. Invoice payload sent to payment provider (Stripe/PayPal):
   POST /invoices
   {
     "external_id": "sms-invoice-abc123",
     "amount": 300000,
     "currency": "XOF",
     "due_date": "2024-09-15",
     "payee": {
       "name": "Aminata Diallo (Parent: Mama Diallo)",
       "email": "mama@example.com"
     },
     "callback_url": "https://api.schoolsync.io/webhooks/payment"
   }

3. Payment provider generates payment link:
   → SMS to parent: "Pay school fees: https://pay.schoolsync.io/inv/xyz"

4. Parent pays, provider calls webhook:
   POST /webhooks/payment
   {
     "event": "payment.completed",
     "invoice_id": "sms-invoice-abc123",
     "amount_paid": 300000,
     "payment_method": "card",
     "transaction_id": "pi_123456"
   }

5. System:
   a. Finds FeeInvoice by external_id
   b. Updates: status = PAID, paid_at = NOW()
   c. Updates Enrollment.fee_status = PAID
   d. Sends confirmation to parent
   e. Notifies registrar: "Payment received"

Result: ✅ External payment integrated seamlessly
```

---

### 12.2 Student Information System (SIS) Integration

```
[Annual import from Ministry of Education SIS]

1. Admin clicks "Import Students from National SIS"

2. Fetches from government API (authenticated):
   GET /api/v1/students?school_id=OUR_CODE&year=2024

3. Receives CSV/JSON with 500 student records:
   [
     {
       "national_id": "123456789012",
       "full_name": "Aminata Diallo",
       "dob": "2017-03-15",
       "gender": "F",
       "previous_school": "Ecole Maternelle Point E",
       "parent_contact": "778765432"
     },
     ...
   ]

4. For each student:
   a. Check if exists: SELECT FROM student_profile WHERE national_id = ?
   b. UPDATE existing OR CREATE new
   c. Link to parent (find by phone)
   d. Create pre-enrollment for upcoming year

5. Report generated:
   - 450 students matched (already in system)
   - 50 new students added
   - 0 errors

Result: ✅ Bulk import handles thousands of records with match logic
```

---

## 13. Scenario Templates (For Future Scenarios)

### Template: Multi-Step Workflow

```
[Title]: [ Descriptive title ]

Context:
- Actors: [...]
- Preconditions: [...]

Steps:
1. [Step description]
   → System action
   → Expected outcome

2. [Next step]
   ...

[Alternate paths]:
- [Error condition]: [Recovery action]
- [Edge case]: [Handling]

Result: ✅ [Summary of successful completion]
```

---

## 14. Acceptance Criteria Format

For test automation, each scenario translates to:

```gherkin
Feature: [Feature name]

  Scenario: [Specific scenario from this document]
    Given [initial state]
    When [action performed]
    Then [expected outcome]
    And [additional verification]

  Scenario Outline: Parameterized variant
    Given student with level <level> and age <age>
    When enrollment attempted
    Then result <result> with reason <reason>

    Examples:
      | level | age | result | reason |
      | CP    | 6   | FAIL   | "Age below minimum" |
      | CP    | 7   | PASS   | "Age eligible" |
```

---

## 15. Non-Functional Scenarios

### 15.1 Performance Under Load

**Scenario**: End-of-term grade submission surge

```
[Friday before deadline: 100 teachers submitting grades]

Given:
- 100 teachers online simultaneously
- Each has 30 evaluations to submit (3000 grades)

Expected:
- API rate limiting prevents system overload
- Queue system processes submissions in order
- All submissions complete within 2 hours
- No data loss

Metrics:
- 95th percentile sync time: < 5 seconds
- Server CPU: < 80%
- Database connections: < 80% of pool
```

### 15.2 Disaster Recovery

**Scenario**: Database failure during promotion process

```
[Database server crash at 2:00 PM]

1. Load balancer detects failure → routes to standby
2. Standby promoted to primary (RPO ~ 5 seconds via streaming replication)
3. System resumes:
   - In-flight transactions rolled back
   - Promotion service retries automatically
   - Notifications delayed, not lost

4. Post-mortem:
   - 5 minutes downtime
   - Data integrity verified
   - No grade loss (written within transactions)

Result: ✅ 5-minute RTO, near-zero RPO achieved
```

---

## 16. Implementation Priority

Based on scenario analysis, critical paths to implement first:

1. **Path A**: Enrollment flow (online + offline) → Core revenue impact
2. **Path B**: Grade entry + sync → Core teaching activity
3. **Path C**: Promotion process → Year-end critical
4. **Path D**: Timetable + schedule → Daily operational need
5. **Path E**: Reports (report cards, transcripts) → Compliance & parent communication

---

**Next Step**: TDR-014 — Architecture Decision Records (ADRs)
