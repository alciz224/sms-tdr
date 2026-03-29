# TDR-012 — Offline Sync Strategy

## 1. Purpose and Scope

This document defines the **offline-first synchronization architecture**, the most critical piece for ensuring the system works during connectivity loss. It covers:

- **Sync architecture** (device ↔ server communication patterns)
- **Conflict resolution** strategies for concurrent modifications
- **Offline queue** management and retry logic
- **Data consistency** guarantees (eventual consistency model)
- **Bandwidth optimization** for slow/poor networks
- **Device state management** and recovery
- **Schema evolution** during sync
- **Security** of offline data

**Key Principle**: All operations must work offline with eventual consistency. Users should never see "cannot save" due to connectivity. Conflicts resolved predictably.

---

## 2. Sync Architecture Overview

### 2.1 System Context

```
┌─────────────────┐                    ┌─────────────────┐
│   Offline Device │                    │   Central Server│
│  (Mobile/Tablet) │                    │   (PostgreSQL)  │
└─────────────────┘                    └─────────────────┘
         │                                       │
         │ 1. User performs operation            │
         │    → stored in local queue            │
         │                                       │
         │ 2. Connectivity restored              │
         │    → POST /sync                       │
         │                                       │
         │ 3. Server processes batch             │
         │    → applies changes                  │
         │    → detects conflicts                │
         │    → returns server changes           │
         │                                       │
         │ 4. Device merges response             │
         │    → updates local DB                 │
         │    → clears processed queue           │
         │                                       │
         │ 5. Server emits events                │
         │    → other devices notified           │
```

### 2.2 Sync Types

| Type | Trigger | Purpose |
|------|---------|---------|
| **Full Sync** | First install, major version change | Download entire dataset (~5MB compressed) |
| **Incremental Sync** | Periodic (every 5 min) or on reconnection | Exchange changes since last sync |
| **Forced Sync** | Manual user action | Ensure latest data before critical operation |
| **Background Sync** | App in background, Wi-Fi available | Maintain cache freshness |

---

## 3. Data Versioning & Change Tracking

### 3.1 Global Sync Version

```sql
-- Server tracks global version counter
CREATE TABLE sync_metadata (
    id SERIAL PRIMARY KEY,
    current_version BIGINT NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Increment version on any data change (trigger-based)
CREATE OR REPLACE FUNCTION increment_sync_version()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE sync_metadata SET current_version = current_version + 1, updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Attach to all mutable tables
CREATE TRIGGER trg_increment_version
    AFTER INSERT OR UPDATE OR DELETE ON enrollment
    FOR EACH STATEMENT EXECUTE FUNCTION increment_sync_version();
```

### 3.2 Row-Level Versioning

Every table has:
- `updated_at` (TIMESTAMPTZ) — last modification time
- `updated_by` (UUID) — who modified
- `sync_version` (BIGINT) — version at time of this change (for incremental sync)

```sql
-- Device stores last sync version
CREATE TABLE device_sync_state (
    device_id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    tenant_id UUID NOT NULL,
    last_sync_version BIGINT NOT NULL DEFAULT 0,
    last_sync_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    is_syncing BOOLEAN DEFAULT FALSE,
    pending_changes_count INTEGER DEFAULT 0
);
```

### 3.3 Change Detection Query

Server responds to sync request with all changes since client's version:

```sql
-- Combined changes query (union all approach)
WITH enrollment_changes AS (
  SELECT 'enrollment' as table_name, id, operation, updated_at, sync_version,
         row_to_json(e) as record
  FROM enrollment e
  WHERE e.updated_at > (SELECT last_sync_version FROM device_sync_state WHERE device_id = $1)
    AND e.is_active = TRUE
),
grade_changes AS (
  SELECT 'grade' as table_name, id, operation, updated_at, sync_version,
         row_to_json(g) as record
  FROM grade g
  WHERE g.updated_at > (SELECT last_sync_version FROM device_sync_state WHERE device_id = $1)
    AND g.is_active = TRUE
)
-- Add all other tables...
SELECT * FROM enrollment_changes
UNION ALL
SELECT * FROM grade_changes
-- ...
ORDER BY sync_version ASC;
```

---

## 4. Offline Queue Architecture

### 4.1 Local Storage Schema

Offline device stores pending changes in a queue table:

```sql
-- SQLite schema (mobile) or local Postgres (desktop)
CREATE TABLE offline_queue (
    id TEXT PRIMARY KEY,              -- UUID generated client-side
    table_name TEXT NOT NULL,        -- 'grade', 'enrollment', etc.
    operation TEXT NOT NULL,         -- 'INSERT', 'UPDATE', 'DELETE'
    record_id TEXT,                  -- server ID if exists, NULL for new
    record_data JSONB NOT NULL,      -- full record data
    idempotency_key TEXT UNIQUE,     -- for deduplication
    created_at TIMESTAMPTZ NOT NULL,
    retry_count INTEGER DEFAULT 0,
    last_error TEXT,
    synced_at TIMESTAMPTZ
);

CREATE INDEX idx_offline_queue_pending ON offline_queue(synced_at) WHERE synced_at IS NULL;
CREATE INDEX idx_offline_queue_created ON offline_queue(created_at DESC);
```

### 4.2 Queue Operations

**Enqueue** (when user performs action offline):
```typescript
function enqueueChange(change: OfflineChange): void {
  const idempotencyKey = generateIdempotencyKey(change);

  // Check if already queued (user hit button twice)
  const existing = db.offline_queue.find({ idempotency_key: idempotencyKey });
  if (existing) {
    return existing; // idempotent, don't duplicate
  }

  db.offline_queue.insert({
    id: uuidv4(),
    table_name: change.table,
    operation: change.operation,
    record_id: change.recordId,  // may be null for new records
    record_data: change.record,
    idempotency_key: idempotencyKey,
    created_at: new Date(),
    retry_count: 0,
    synced_at: null
  });

  // Update pending count for UI indicator
  updatePendingIndicator();
}
```

**Dequeue** (during sync):
```typescript
async function getPendingChanges(limit: number = 50): Promise<OfflineChange[]> {
  return db.offline_queue
    .where({ synced_at: null })
    .orderBy('created_at')
    .limit(limit)
    .select();
}
```

**Mark Synced** (after successful sync):
```typescript
function markChangesSynced(changeIds: string[]): void {
  db.offline_queue
    .whereIn('id', changeIds)
    .update({ synced_at: new Date() });
}

// Periodically clean old synced entries (older than 30 days)
function cleanOldSyncedEntries(): void {
  const cutoff = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000);
  db.offline_queue
    .where({ synced_at: notNull })
    .where('synced_at', '<', cutoff)
    .del();
}
```

---

## 5. Sync Protocol

### 5.1 Request Format

```
POST /v1/sync
Headers:
  Authorization: Bearer <token>
  X-Device-ID: <device_uuid>
  X-Client-Version: 2.1.25
  X-Sync-Version: 12345
  X-Idempotency-Key: <sync_session_uuid>

Body:
{
  "last_sync_version": 12345,
  "pending_changes": [
    {
      "local_id": "local-uuid-1",        // for response correlation
      "idempotency_key": "unique-key-1",
      "operation": "UPDATE",
      "table": "grade",
      "record": {
        "evaluation_id": "server-eval-uuid",
        "student_profile_id": "server-student-uuid",
        "value": 15.5,
        "justification": "Good work",
        "_server_record": {  // cached server version for conflict detection
          "value": 15.0,
          "sync_version": 12340
        }
      }
    }
  ],
  "device_info": {
    "platform": "android",
    "os_version": "14",
    "app_version": "2.1.25",
    "storage_available_mb": 512,
    "last_sync_at": "2025-03-29T10:30:00Z"
  }
}
```

### 5.2 Response Format

```json
{
  "success": true,
  "data": {
    "new_sync_version": 12352,
    "server_timestamp": "2025-03-29T10:35:00Z",
    "server_changes": [
      {
        "table": "timetable",
        "operation": "UPDATE",
        "record": {
          "id": "tt-uuid",
          "code": "2024-2025-PRIM-TT01",
          "status": "ACTIVE",
          "sync_version": 12352
        }
      }
    ],
    "conflicts": [
      {
        "local_id": "local-uuid-1",
        "table": "grade",
        "record_id": "grade-uuid",
        "conflict_type": "CONCURRENT_MODIFICATION",
        "local_value": { "value": 15.5 },
        "server_value": { "value": 16.0, "state": "FINALIZED" },
        "resolution": "SERVER_WINS",
        "reason": "Grade already finalized on server"
      }
    ],
    "rejected_changes": [
      {
        "local_id": "local-uuid-2",
        "reason": "VALIDATION_FAILED",
        "error_code": "GRADE_EVALUATION_NOT_FOUND",
        "message": "Evaluation was deleted",
        "suggestion": "Contact administrator"
      }
    ],
    "next_sync_utc": "2025-03-29T12:00:00Z",
    "sync_duration_ms": 1240,
    "changes_applied": 7,
    "conflicts_detected": 1,
    "changes_rejected": 2
  }
}
```

---

## 6. Conflict Resolution Strategies

### 6.1 Conflict Types

| Type | Scenario | Resolution |
|------|----------|------------|
| **Concurrent Modification** | Same record edited on device and server | Last-write-wins OR manual review |
| **Delete vs Update** | Record deleted on server, updated locally | Server wins (deletion is explicit) |
| **Referential Integrity** | Local FK points to deleted server record | Reject with FK violation |
| **Constraint Violation** | Local change violates server constraint | Reject, notify user |
| **Immutable Finalized** | Attempt to modify FINALIZED grade | Reject, suggest appeal process |

### 6.2 Resolution Policies

#### Policy 1: Last-Write-Wins (LWW)

```
IF local.sync_version < server.sync_version:
    SERVER_WINS → replace local with server
    notify user: "Data changed on another device"
ELSE:
    CLIENT_WINS → overwrite server
    (rare: client has newer version)
```

**Applied to**: Non-critical data (timetable exceptions, user preferences)

#### Policy 2: Server Authoritative

```
Always use server version, reject client change
with clear error explaining why.
```

**Applied to**:
- Finalized grades
- Deleted records
- System-generated fields (enrollment numbers)
- Reference data (Region, Subject, etc.)

#### Policy 3: Merge (Field-Level)

```
For each field:
  IF field modified only locally: keep local
  IF field modified only on server: keep server
  IF field modified both: apply merge rule (last-write, array merge, etc.)
```

**Applied to**: CustomUser profile (name, phone), partial updates.

#### Policy 4: Manual Review

```
Queue conflict for admin resolution.
Both versions preserved.
User notified: "Data conflict requires review"
```

**Applied to**:
- Enrollment capacity conflicts
- Critical grade changes
- Promotion decision conflicts

---

### 6.3 Conflict Resolution Implementation

```typescript
enum ConflictResolution {
  SERVER_WINS = 'SERVER_WINS',
  CLIENT_WINS = 'CLIENT_WINS',
  MANUAL_REVIEW = 'MANUAL_REVIEW',
  MERGE = 'MERGE'
}

interface Conflict {
  localId: string;
  table: string;
  recordId: string;
  conflictType: ConflictType;
  localValue: any;
  serverValue: any;
  resolution: ConflictResolution;
  reason?: string;
}

function resolveConflict(conflict: Conflict): ResolutionResult {
  switch (conflict.table) {
    case 'grade':
      // Grades: server authoritative if FINALIZED
      if (conflict.serverValue.state === 'FINALIZED') {
        return { resolution: ConflictResolution.SERVER_WINS, reason: 'Finalized grade cannot be modified' };
      }
      // Otherwise, last-write-wins based on timestamps
      return applyLastWriteWins(conflict);

    case 'enrollment':
      // Enrollment capacity conflicts → manual review
      if (conflict.conflictType === ConflictType.CAPACITY_EXCEEDED) {
        return { resolution: ConflictResolution.MANUAL_REVIEW, reason: 'Classroom capacity conflict' };
      }
      return applyLastWriteWins(conflict);

    case 'timetable_entry':
      // Teacher scheduling conflicts → manual review
      if (conflict.conflictType === ConflictType.TEACHER_CONFLICT) {
        return { resolution: ConflictResolution.MANUAL_REVIEW };
      }
      return applyLastWriteWins(conflict);

    case 'customuser':
      // Profile: merge (fields don't conflict)
      return { resolution: ConflictResolution.MERGE, mergeStrategy: 'field_level' };

    default:
      return { resolution: ConflictResolution.SERVER_WINS };
  }
}
```

---

## 7. Sync Flow Detailed

### 7.1 Full Sync (Initial Download)

```
1. Device requests: GET /v1/sync/full
   Headers: X-Device-ID, X-Client-Version

2. Server validates:
   - Device registered
   - Version compatible
   - User has access to tenant

3. Server streams reference data in order:
   a. Geographic referentials (Region, Prefecture, SousPrefecture, Quartier)
   b. Academic referentials (AcademicYearType, TermType, Category, Level, Track, Subject)
   c. Current SchoolYear + Terms + SchoolYearCategories
   d. Timetable (if active)
   e. User profile + roles
   f. Student roster (if teacher/parent)

4. Device receives data in chunks:
   - Each chunk has checksum
   - Device verifies integrity
   - Stores to local DB transactionally

5. After full sync:
   - Set device_sync_state.last_sync_version = server.current_version
   - Mark full sync complete
   - Enable offline operations
```

**Compression**: All data compressed (gzip) or as protobuf for efficiency.

**Estimated sizes**:
- Reference data: ~500KB
- SchoolYear structure: ~200KB
- Full roster (1000 students): ~2MB
- **Total: < 5MB compressed** for typical school

---

### 7.2 Incremental Sync Flow

```
┌─────────────┐
│ Start Sync  │
└──────┬──────┘
       │
       ▼
┌─────────────────────┐
│ 1. Fetch pending    │  Read local offline_queue
│    local changes    │  (filter synced_at IS NULL)
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ 2. POST /sync       │  Send: last_sync_version +
│    with changes     │         pending_changes[]
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ 3. Server process   │  For each change:
│    batch            │   - Validate permissions
│                     │   - Run business rules
│                     │   - Check conflicts
│                     │   - Apply or reject
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ 4. Server response  │  Return:
│    includes:        │   - server_changes (for device)
│    - New version    │   - conflicts (to resolve)
│    - Conflicts      │   - rejected_changes (with reason)
│    - Rejected       │   - new_sync_version
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ 5. Device merges    │  Apply server changes to local
│    response         │  For conflicts: store in
│                     │   conflict_resolution_queue
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ 6. Update state     │  - Clear processed queue items
│                     │  - Set last_sync_version
│                     │  - Invalidate relevant caches
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ 7. Retry failed     │  Items in conflict or rejected
│    items            │  queues stay pending for manual
│                     │  resolution or retry
└─────────────────────┘
```

---

## 8. Conflict Resolution Workflow

### 8.1 Automatic Resolution

```typescript
async function processSyncResponse(response: SyncResponse): Promise<void> {
  // 1. Apply server changes
  for (const change of response.server_changes) {
    const localRecord = await localDb[change.table].find(change.record.id);
    if (localRecord) {
      // Upsert: server version overwrites local
      await localDb[change.table].update(
        { id: change.record.id },
        change.record
      );
    } else {
      await localDb[change.table].insert(change.record);
    }
  }

  // 2. Process conflicts
  for (const conflict of response.conflicts) {
    const resolution = resolveConflict(conflict);

    if (resolution.resolution === ConflictResolution.SERVER_WINS) {
      // Overwrite local with server
      await localDb[conflict.table].update(
        { local_id: conflict.local_id },
        conflict.server_value
      );
      markQueueItemSynced(conflict.local_id);

    } else if (resolution.resolution === ConflictResolution.CLIENT_WINS) {
      // Re-send on next sync (keep in queue but with updated retry count)
      await localDb.offline_queue.update(
        { id: conflict.local_id },
        { retry_count: db.raw('retry_count + 1') }
      );

    } else if (resolution.resolution === ConflictResolution.MANUAL_REVIEW) {
      // Move to manual review queue
      await localDb.conflict_resolution_queue.insert({
        conflict: conflict,
        created_at: new Date(),
        status: 'PENDING_REVIEW'
      });
      markQueueItemSynced(conflict.local_id);
    }
  }

  // 3. Handle rejected changes
  for (const rejected of response.rejected_changes) {
    await localDb.offline_queue.update(
      { local_id: rejected.local_id },
      {
        last_error: rejected.reason,
        retry_count: db.raw('retry_count + 1')
      }
    );
    notifyUserOfRejection(rejected);
  }

  // 4. Update version
  await localDb.device_sync_state.update(
    { device_id: deviceId },
    {
      last_sync_version: response.new_sync_version,
      last_sync_at: new Date()
    }
  );
}
```

---

### 8.2 Manual Conflict Resolution UI

```
Conflict detected:
─────────────────────────────────────────
A grade was modified on another device.

Your value:    15.5 (entered offline on Mar 28)
Server value: 16.0 (finalized by teacher on Mar 29)
Status:        Server record is FINALIZED

Resolution options:
[ ] Accept server value (16.0) ← Recommended
[ ] Request grade amendment (contact teacher)
[ ] Keep my value and request review (principal approval required)

─────────────────────────────────────────
[Resolve] [Dismiss]
```

---

## 9. Offline Capability Matrix

### 9.1 What Works Offline

| Operation | Offline Support | Notes |
|-----------|-----------------|-------|
| View own profile | ✅ | Cached on login |
| View classroom schedule | ✅ | Timetable cached |
| View student roster | ✅ | Roster cached, refreshed daily |
| View report cards | ✅ | Already finalized |
| Enter new grade | ✅ | Queued for submission |
| Edit draft grade | ✅ | Local only until submit |
| Submit grade | ⚠️ | Queued, actually submits on sync |
| Create evaluation | ❌ | Requires server validation |
| Publish timetable | ❌ | Admin-only, too risky |
| Register new student | ⚠️ | Initiate offline, complete online |
| Upload document | ⚠️ | Upload queued, URL signed later |

### 9.2 Offline Operation States

```
Operation Status:
─────────────────────────────────────
PENDING_QUEUED    → Saved locally, waiting for sync
SYNCING           → Being sent to server
SERVER_ACCEPTED   → On server, may need sync back
SERVER_REJECTED   → Error, needs attention
CONFLICT_DETECTED → Requires resolution
COMPLETED         → Fully synced and acknowledged
```

---

## 10. Sync Scheduling Strategies

### 10.1 Triggers

| Trigger | Conditions | Action |
|---------|------------|--------|
| **App Launch** | Always | Full sync if first launch, else incremental |
| **Network Reconnect** | Wi-Fi/Cellular available | Incremental sync |
| **Foreground** | App comes to foreground | Quick sync (only urgent changes) |
| **Periodic** | Every 5 minutes (foreground) | Incremental sync |
| **Background** | Device idle + charging + Wi-Fi | Full referential sync if version mismatch |
| **Manual** | User taps "Sync Now" | Full incremental with progress UI |
| **Operation Blocking** | Offline operation requires server data | Trigger sync before operation |

### 10.2 Exponential Backoff

```typescript
const BACKOFF_SCHEDULE = [0, 1000, 5000, 30000, 120000, 600000]; // milliseconds

function scheduleNextSync(retryCount: number): number {
  if (retryCount >= BACKOFF_SCHEDULE.length) {
    return BACKOFF_SCHEDULE[BACKOFF_SCHEDULE.length - 1];
  }
  return BACKOFF_SCHEDULE[retryCount];
}
```

On sync failure:
- Log error
- Increment retry count
- Schedule next attempt after backoff delay
- After 5 failures, notify user: "Sync issues, check connection"

---

## 11. Data Consistency Model

### 11.1 Eventual Consistency Guarantees

```
Device A                Server                Device B
    │                      │                      │
    │── grade.enter ──────>│                      │
    │                      │── store grade ──────>│
    │                      │                      │
    │◄── grade.enter OK ──│                      │
    │                      │                      │
    │                      │                      │─── grade.read (sees grade)
    │                      │                      │
    │◄── sync push ───────┤                      │
    │                      │                      │◄── pull sync ──────┘
```

**Sync window**: Maximum 5 minutes (aggressive) to 24 hours (passive).

### 11.2 Read-Your-Writes Guarantee

Device ensures:
- User's own changes appear in their queries immediately (optimistic UI)
- Changes marked as "synced" only when server acknowledges
- Local cache updated before showing success to user

---

## 12. Security Considerations

### 12.1 Data at Rest (Offline)

- Local database encrypted (SQLCipher, Realm encryption)
- Sensitive PII field-level encryption (medical, financial)
- Key derived from user password or device keystore
- Remote wipe capability on device lost

### 12.2 Data in Transit

- All sync over HTTPS (TLS 1.2+)
- Certificate pinning for device trust
- Request signing (HMAC) for integrity verification

### 12.3 Access Control

- Each sync request validated: user can only sync their tenant's data
- Row-level security enforced during data fetch
- Device blacklist for lost/stolen devices

---

## 13. Schema Evolution

### 13.1 Version Detection

```
Request: X-Client-Version: 2.1.25
Response: 409 CONFLICT
{
  "error": "CLIENT_VERSION_INCOMPATIBLE",
  "required_version": "2.2.0",
  "migration_url": "https://schoolsync.io/migrate/2.1.x-to-2.2.0"
}
```

### 13.2 Backward Compatibility

Server supports:
- Old client schema (missing new fields → use defaults)
- New client with old server (graceful degradation)
- Field addition (safe, old clients ignore new fields)
- Field removal (deprecated for 1 year, returns NULL)

### 13.3 Migration on Sync

When client version incompatible:
1. Server returns migration instructions
2. Client downloads migration package
3. Local data transformed to new schema
4. Full sync required post-migration

---

## 14. Performance Optimization

### 14.1 Delta Encoding

Instead of full rows, send only changed fields:

```json
{
  "table": "grade",
  "id": "grade-uuid",
  "operation": "UPDATE",
  "changed_fields": {
    "value": 15.5,
    "justification": "Updated after review"
  },
  "sync_version": 12352
}
```

### 14.2 Compression

- gzip HTTP compression (automatic)
- Protobuf encoding for large batches (optional)
- Binary diff for BLOBs (documents, images)

### 14.3 Selective Sync

User role determines sync scope:
- Teacher: Sync only their classrooms
- Parent: Sync only children's data
- Admin: Sync full school (Wi-Fi required option)

---

## 15. Monitoring & Alerting

### 15.1 Sync Health Metrics

| Metric | Target | Alert |
|--------|--------|-------|
| Sync success rate | > 99.5% | < 99% |
| Average sync duration | < 10 seconds | > 30 seconds |
| Conflict rate | < 0.1% | > 1% |
| Queue depth (per device) | < 10 | > 100 |
| Offline changes age | < 1 hour | > 24 hours |

### 15.2 Logging

```
[SYNC] device=abc123 user=456 tenant=789 version=12345→12352
  uploaded: 7 changes (grade=5, enrollment=2)
  downloaded: 12 changes (timetable=1, evaluation=3, grade=8)
  conflicts: 1 (grade LWW)
  duration: 1240ms
  status: SUCCESS
```

---

## 16. Offline Queue Priority

Queue items prioritized:
1. **High**: Grade submission (deadline-driven)
2. **Medium**: Enrollment activation (time-sensitive)
3. **Low**: Profile updates, preferences

Priority affects:
- Sync batch ordering
- Retry backoff (higher priority = more aggressive)
- Background sync scheduling

---

## 17. Emergency Recovery

### 17.1 Device Data Loss

If local database corrupted/lost:
1. User authenticates
2. Server detects device never synced or very old version
3. Server triggers **forced full sync**
4. User warned: "This may take several minutes"

### 17.2 Server Rollback

If server data corrupted:
1. Enable maintenance mode
2. Restore from backup (with PITR)
3. Devices sync automatically, conflicts arise
4. Conflict resolution: server version from backup is authoritative

---

## 18. Implementation Checklist

### Device (Client)

- [ ] Offline queue table with indexing
- [ ] Sync state tracking (last version, pending count)
- [ ] Idempotency key generation and deduplication
- [ ] Conflict resolution UI
- [ ] Exponential backoff retry
- [ ] Background sync scheduling
- [ ] Progress indicator during sync
- [ ] Network connectivity detection
- [ ] Data compression before upload
- [ ] Local encryption at rest

### Server

- [ ] Global sync version counter
- [ ] Change tracking triggers on all mutable tables
- [ ] Incremental change query optimization
- [ ] Conflict detection logic per table
- [ ] Idempotency key storage (prevent duplicate processing)
- [ ] Rate limiting per device
- [ ] Sync logging and audit
- [ ] Conflict queue for manual review
- [ ] Push notification for urgent server changes
- [ ] API endpoint with streaming response for large datasets

---

## 19. Testing Strategy

### 19.1 Offline Simulation Tests

```typescript
describe('Offline Sync', () => {
  beforeEach(() => {
    // Simulate offline: block network, use local DB
    mockNetwork({ offline: true });
  });

  it('should queue grade entry when offline', async () => {
    await gradeService.enterGrade(evaluationId, studentId, 15);
    const queued = await offlineQueue.getPending();
    expect(queued).toHaveLength(1);
    expect(queued[0].operation).toBe('INSERT');
  });

  it('should sync queued changes when online', async () => {
    // Queue changes offline
    await gradeService.enterGrade(evaluationId, studentId, 15);

    // Simulate reconnect
    mockNetwork({ offline: false });
    const result = await syncService.sync();

    expect(result.success).toBe(true);
    expect(result.conflicts).toHaveLength(0);
  });

  it('should handle concurrent modification conflicts', async () => {
    // Device A enters grade 15
    mockNetwork({ offline: true });
    await gradeService.enterGrade(evalId, studentId, 15);

    // Meanwhile, server updated grade to 16 (teacher)
    await gradeService.amendGrade(serverGradeId, 16, 'Correction');

    // Sync A's queued change
    mockNetwork({ offline: false });
    const result = await syncService.sync();

    expect(result.conflicts).toHaveLength(1);
    expect(result.conflicts[0].resolution).toBe('SERVER_WINS');
  });
});
```

---

## 20. Data Sync Compliance

### 20.1 GDPR Right to Erasure

When user requests data deletion:
1. Mark user as `gdpr_deleted = true`
2. Cron job excludes from future syncs
3. Existing devices receive delete operations:
   ```
   { table: "enrollment", operation: "DELETE", record_id: "..." }
   ```
4. Devices must delete local copies

### 20.2 Audit Trail

All sync operations logged:
- Device ID, user ID, IP
- Changes sent and received
- Conflicts detected and resolved
- Retained for 10 years (legal requirement)

---

**Next Step**: TDR-013 — Service Scenarios (use case flows, narrative examples)
