# TDR-016 — Technology Stack Selection

## Executive Summary

After completing the technology-agnostic TDR (TDR-001 through TDR-015), the implementation team has selected the following technology stack:

| Layer | Technology | Rationale |
|-------|------------|-----------|
| **Backend API** | Python 3.11+ / Django 5.0+ / Django REST Framework | Mature, scalable, excellent admin interface, strong ORM for complex queries |
| **Web Frontend** | TypeScript / TanStack Start (React) | Modern React framework with SSR, type safety, excellent developer experience |
| **Mobile** | Flutter 3.x (Dart) | Single codebase for iOS/Android, offline-first packages, good sync capabilities |
| **Database** | PostgreSQL 15+ | ACID compliance, partitioning, JSONB, proven at scale |
| **Cache/Session** | Redis | Fast session storage, caching materialized view results |
| **Queue/Sync** | PostgreSQL-based queue initially, upgrade to RabbitMQ if needed | Simplicity first, can scale later |
| **File Storage** | MinIO (S3-compatible) or AWS S3 | Document storage, report cards, uploads |
| **Authentication** | JWT + Simple JWT (DRF) + MFA via TOTP | Industry standard, mobile-friendly |
| **Deployment** | Docker + Docker Compose (dev), Kubernetes (prod) | Consistent environments, scalable |

---

## Backend: Python/Django/DRF

### Why Django?

| Requirement | Django Feature |
|-------------|----------------|
| **Complex queries** | Powerful ORM with joins, aggregates, subqueries |
| **Admin interface** | Built-in Django Admin for CRUD operations (prototyping) |
| **REST API** | Django REST Framework (serializers, viewsets, permissions) |
| **Authentication** | Django auth system + Simple JWT for tokens |
| **Migrations** | Built-in migration system for schema evolution |
| **Validation** | Model validation + DRF serializer validation |
| **Multi-tenancy** | Tenant-based middleware, easy to add tenant_id filters |
| **Testing** | pytest-django, factory_boy for fixtures |
| **Maturity** | 18+ years, large ecosystem, excellent documentation |

### Key Django Apps Structure

```
sms_backend/
├── apps/
│   ├── referentials/        (Region, Prefecture, Category, Subject, etc.)
│   ├── users/              (CustomUser, profiles, authentication)
│   ├── core/               (AcademicYear, SchoolYear, Category, Term)
│   ├── enrollments/        (Enrollment, Classroom, ClassroomSubject)
│   ├── evaluations/        (Evaluation, Grade, GradeType, GradeAppeal)
│   ├── timetable/          (TimeSlot, Timetable, Entry, Exception)
│   ├── sync/               (Offline sync endpoint, conflict resolution)
│   ├── reporting/          (Report cards, transcripts, analytics)
│   ├── notifications/      (Email/SMS notifications)
│   └── api/                (Versioned API endpoints, throttling)
├── config/
│   └── settings/
│       ├── base.py
│       ├── development.py
│       └── production.py
└── manage.py
```

### DRF Patterns

**ViewSets for standard CRUD**:
```python
class EnrollmentViewSet(viewsets.ModelViewSet):
    queryset = Enrollment.objects.all()
    serializer_class = EnrollmentSerializer
    permission_classes = [IsAuthenticated, HasSchoolPermission]
    filterset_fields = ['school_year_category', 'classroom', 'state']
    ordering = ['-created_at']
```

**Custom actions for business operations**:
```python
class EnrollmentViewSet(viewsets.ModelViewSet):
    @action(detail=True, methods=['post'])
    def activate(self, request, pk=None):
        enrollment = self.get_object()
        enrollment_service.activate(enrollment, request.user)
        return Response(EnrollmentSerializer(enrollment).data)

    @action(detail=True, methods=['delete'])
    def withdraw(self, request, pk=None):
        enrollment = self.get_object()
        enrollment_service.withdraw(
            enrollment,
            reason=request.data.get('reason'),
            operator=request.user
        )
        return Response(status=204)
```

**Sync endpoint with batch processing**:
```python
@api_view(['POST'])
@permission_classes([IsAuthenticated])
def sync(request):
    device_id = request.headers.get('X-Device-ID')
    last_sync_version = request.headers.get('X-Sync-Version')

    # Process pending changes from device
    pending_changes = request.data.get('pending_changes', [])

    results = SyncService.process_sync(
        device_id=device_id,
        last_sync_version=int(last_sync_version),
        pending_changes=pending_changes,
        user=request.user
    )

    return Response(results)
```

---

## Frontend Web: TypeScript / TanStack Start

### Why TanStack Start?

| Requirement | TanStack Solution |
|-------------|-------------------|
| **Type safety** | TypeScript + TanStack Query typed hooks |
| **Server components** | TanStack Start SSR for initial page load |
| **Routing** | TanStack Router (file-based or route-based) |
| **Data fetching** | TanStack Query for server state |
| **Offline capability** | Service Workers + TanStack Query persistence |
| **Forms** | React Hook Form + Zod validation |
| **UI components** | Shadcn/ui or similar Radix-based component library |

### Frontend App Structure

```
sms-web/
├── src/
│   ├── app/                    # TanStack Start app router
│   │   ├── (auth)/            # Auth routes (login, logout)
│   │   ├── dashboard/         # Teacher/parent/student dashboard
│   │   ├── admin/             # Admin panels
│   │   └── layout.tsx
│   ├── components/            # Reusable UI components
│   │   ├── ui/                # Shadcn/ui primitives
│   │   ├── enrollment/        # Enrollment-specific components
│   │   ├── grades/            # Grade entry components
│   │   └── timetable/         # Schedule components
│   ├── lib/                   # Utilities, API clients
│   │   ├── api.ts             # Configured TanStack Query client
│   │   ├── auth.ts            # Authentication helpers
│   │   └── sync.ts            # Offline sync manager
│   ├── hooks/                 # Custom React hooks
│   ├── types/                 # TypeScript definitions
│   └── utils/                 # Helper functions
├── public/
└── package.json
```

### Key Frontend Patterns

**TanStack Query for API state**:
```typescript
// React Query hook for enrollment
export function useEnrollment(id: string) {
  return useQuery({
    queryKey: ['enrollment', id],
    queryFn: () => api.get<Enrollment>(`/enrollments/${id}`),
  });
}

// Mutation for activation
export function useActivateEnrollment() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (id: string) => api.post(`/enrollments/${id}/activate`, {}),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['enrollment'] });
    },
  });
}
```

**Offline sync manager**:
```typescript
class SyncManager {
  private queue: OfflineChange[] = [];
  private isOnline = navigator.onLine;

  constructor() {
    window.addEventListener('online', this.sync.bind(this));
    window.addEventListener('offline', this.setOffline.bind(this));
    this.loadQueue();
  }

  async enqueue(change: OfflineChange): Promise<void> {
    this.queue.push(change);
    await this.saveQueue();
    if (this.isOnline) {
      await this.sync();
    }
  }

  private async sync(): Promise<void> {
    const changes = this.queue.filter(c => !c.synced);
    const response = await api.sync(changes);
    response.applied.forEach(id => this.markSynced(id));
    // Handle conflicts...
  }
}
```

---

## Mobile: Flutter

### Why Flutter?

| Requirement | Flutter Advantage |
|-------------|-------------------|
| **Cross-platform** | Single codebase for iOS + Android |
| **Offline-first** | SQLite via sqflite/moor, Isar for fast local storage |
| **Sync capabilities** | Background fetch, WorkManager for periodic sync |
| **Performance** | Compiled to native ARM, smooth animations |
| **UI consistency** | Material Design & Cupertino widgets out of box |
| **Rapid development** | Hot reload, declarative UI like React |

### Flutter App Structure

```
sms_mobile/
├── lib/
│   ├── main.dart
│   ├── config/
│   │   ├── routes.dart      # Navigation routes
│   │   ├── theme.dart       # App theming
│   │   └── api_client.dart  # Dio HTTP client with interceptors
│   ├── data/
│   │   ├── local/           # SQLite/Isar database, models
│   │   │   ├── database.dart
│   │   │   ├── daos/
│   │   │   └── models/
│   │   ├── remote/          # API models, DTOs
│   │   └── repositories/    # Repository pattern implementation
│   ├── domain/
│   │   ├── entities/        # Business entities
│   │   ├── services/        # Business logic services
│   │   └── repositories.dart # Repository interfaces
│   ├── presentation/
│   │   ├── pages/
│   │   │   ├── auth/
│   │   │   ├── dashboard/
│   │   │   ├── enrollments/
│   │   │   ├── grades/
│   │   │   └── timetable/
│   │   ├── widgets/         # Reusable UI components
│   │   ├── providers/       # State management (Riverpod/Bloc)
│   │   └── view_models/     # View models for pages
│   └── utils/
│       ├── sync_manager.dart
│       ├── offline_queue.dart
│       └── notifications.dart
├── android/
├── ios/
├── pubspec.yaml
└── analysis_options.yaml
```

### Key Flutter Patterns

**Repository pattern with sync**:
```dart
abstract class GradeRepository {
  Future<List<Grade>> getStudentGrades(String studentId, String termId);
  Future<Grade> enterGrade(Grade grade, {bool offline = false});
  Stream<List<Grade>> watchPendingSync();
}

class GradeRepositoryImpl implements GradeRepository {
  final LocalDataSource local;
  final RemoteDataSource remote;
  final SyncQueue syncQueue;

  @override
  Future<Grade> enterGrade(Grade grade, {bool offline = false}) async {
    // Save locally first
    final savedGrade = await local.saveGrade(grade);

    // Queue for sync
    await syncQueue.enqueue(
      table: 'grade',
      operation: 'INSERT',
      record: grade.toJson(),
    );

    if (!offline) {
      await syncQueue.process();
    }

    return savedGrade;
  }
}
```

**State management with Riverpod**:
```dart
finalgradesProvider = FutureProvider.family<List<Grade>, String>((ref, studentId) async {
  final repo = ref.watch(gradeRepositoryProvider);
  return await repo.getStudentGrades(studentId);
});

final activateEnrollmentProvider = FutureProvider.family<void, String>((ref, enrollmentId) async {
  final repo = ref.watch(enrollmentRepositoryProvider);
  await repo.activateEnrollment(enrollmentId);
  ref.invalidate(enrollmentsProvider); // Refresh list
});
```

---

## Database: PostgreSQL

### Why PostgreSQL?

- **ACID compliance** — Critical for grade integrity
- **Partitioning** — Native declarative partitioning for 10+ year retention
- **JSONB** — Flexible storage for profile fields (medical, preferences)
- **GIN indexes** — Full-text search (optional)
- **Row-level security** — Multi-tenant security
- **Proven** — Battle-tested for educational systems

### Key Configuration (postgresql.conf)

```conf
# Memory (adjust based on server RAM)
shared_buffers = 4GB
effective_cache_size = 12GB
work_mem = 64MB

# Connection pooling (use PgBouncer separately)
max_connections = 200

# WAL for replication
wal_level = replica
archive_mode = on

# Parallel query
max_parallel_workers_per_gather = 4

# Partition pruning
enable_partition_pruning = on
```

### Recommended Extensions

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;      -- UUID generation
CREATE EXTENSION IF NOT EXISTS btree_gin;     -- GIN indexes on sorted data
CREATE EXTENSION IF NOT EXISTS intarray;      -- Array operations
```

---

## Sync Strategy Implementation

### Backend (Django)

```python
# apps/sync/views.py
class SyncView(APIView):
    permission_classes = [IsAuthenticated]

    def post(self, request):
        device_id = request.headers.get('X-Device-ID')
        last_sync_version = int(request.headers.get('X-Sync-Version', 0))
        pending_changes = request.data.get('pending_changes', [])

        sync_service = SyncService(
            user=request.user,
            tenant=request.user.current_tenant
        )

        result = sync_service.process_sync(
            device_id=device_id,
            last_sync_version=last_sync_version,
            pending_changes=pending_changes
        )

        return Response(result.data)

# apps/sync/services.py
class SyncService:
    def __init__(self, user, tenant):
        self.user = user
        self.tenant = tenant
        self.device, _ = Device.objects.get_or_create(
            device_id=device_id,
            user=user,
            defaults={'tenant': tenant}
        )

    def process_sync(self, device_id, last_sync_version, pending_changes):
        with transaction.atomic():
            # 1. Get server changes since last sync
            server_changes = self._get_server_changes(last_sync_version)

            # 2. Process pending changes from device
            conflicts = []
            for change in pending_changes:
                try:
                    self._apply_change(change)
                except ConflictError as e:
                    conflicts.append(e.to_dict())
                except ValidationError as e:
                    # Rejected change
                    pass

            # 3. Update device sync state
            new_version = self._increment_version()
            self.device.update_sync(new_version)

        return SyncResponse(
            new_sync_version=new_version,
            server_changes=server_changes,
            conflicts=conflicts
        )
```

### Frontend (React + Service Worker)

```typescript
// lib/sync/ServiceWorkerSync.ts
class ServiceWorkerSync {
  private readonly syncQueue = 'sync-queue';
  private readonly syncVersion = 'sync-version';

  async enqueue(change: OfflineChange): Promise<void> {
    const queue = await this.getQueue();
    queue.push({
      ...change,
      id: crypto.randomUUID(),
      timestamp: new Date().toISOString(),
      synced: false,
    });
    await this.saveQueue(queue);

    // Register for background sync if supported
    if ('serviceWorker' in navigator && 'sync' in window.ServiceWorkerRegistration.prototype) {
      const registration = await navigator.serviceWorker.ready;
      await registration.sync.register('sync-changes');
    }
  }

  async syncInBackground(): Promise<void> {
    const queue = await this.getQueue();
    const pending = queue.filter(c => !c.synced);

    if (pending.length === 0) return;

    try {
      const result = await api.sync(pending);
      await this.applyServerChanges(result.server_changes);
      await this.markSynced(result.applied_ids);
      await this.handleConflicts(result.conflicts);
    } catch (error) {
      console.error('Sync failed:', error);
      // Exponential backoff will retry
    }
  }
}
```

### Flutter (Mobile)

```dart
// lib/utils/sync_manager.dart
class SyncManager {
  final Isar _isar;
  final Dio _dio;
  final FlutterLocalNotificationsPlugin _notifications;

  Future<void> startPeriodicSync() async {
    // Background sync every 5 minutes when online
    await Workmanager().initialize(
      callbackDispatcher,
      isInDebugMode: kDebugMode,
    );

    await Workmanager().registerPeriodicTask(
      'syncTask',
      'syncTask',
      frequency: Duration(minutes: 5),
      constraints: Constraints(networkType: NetworkType.connected),
    );
  }

  Future<void> syncNow() async {
    final pending = await _isar.offlineChanges
        .where()
        .syncedEqualTo(false)
        .findAll();

    if (pending.isEmpty) return;

    try {
      final response = await _dio.post(
        '/v1/sync',
        data: {'pending_changes': pending.map((c) => c.toJson()).toList()},
        options: Options(headers: {
          'X-Device-ID': await _getDeviceId(),
          'X-Sync-Version': await _getLastSyncVersion(),
        }),
      );

      await _processSyncResponse(response.data);
    } on DioException catch (e) {
      if (e.response?.statusCode == 409) {
        await _handleConflicts(e.response?.data['conflicts']);
      } else {
        rethrow;
      }
    }
  }
}

// Background task callback
@pragma('vm:entry-point')
void callbackDispatcher() {
  Workmanager().executeTask((task, inputData) async {
    final syncManager = SyncManager();
    await syncManager.syncNow();
    return Future.value(true);
  });
}
```

---

## Deployment Architecture

### Development

```yaml
# docker-compose.yml
version: '3.8'
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: sms_dev
      POSTGRES_USER: sms
      POSTGRES_PASSWORD: sms
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  django:
    build: .
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgres://sms:sms@postgres/sms_dev
      REDIS_URL: redis://redis:6379/0
    depends_on:
      - postgres
      - redis

  web:
    build: ./sms-web
    ports:
      - "3000:3000"
    environment:
      VITE_API_URL: http://localhost:8000/api/v1

volumes:
  postgres_data:
```

### Production (Kubernetes)

```yaml
# k8s/deployment simplified
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: django-api
  template:
    metadata:
      labels:
        app: django-api
    spec:
      containers:
      - name: django
        image: sms-backend:prod
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: sms-secrets
              key: database-url
        - name: REDIS_URL
          value: "redis://redis-master:6379/0"
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
---
apiVersion: v1
kind: Service
metadata:
  name: django-api
spec:
  selector:
    app: django-api
  ports:
  - port: 8000
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sms-ingress
spec:
  rules:
  - host: api.schoolsync.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: django-api
            port:
              number: 8000
```

---

## Testing Strategy

### Backend (Django/pytest)

```python
# tests/test_enrollment_service.py
@pytest.mark.django_db
class TestEnrollmentService:
    def test_create_enrollment_valid(self, school_year_category, student):
        request = CreateEnrollmentRequest(
            student_profile_id=student.id,
            school_year_category_id=school_year_category.id,
        )
        enrollment = EnrollmentService().create_enrollment(request)

        assert enrollment.state == EnrollmentState.PENDING
        assert enrollment.enrollment_number.startswith(school_year_year)

    def test_activate_enrollment_success(self, pending_enrollment):
        enrollment = EnrollmentService().activate_enrollment(
            pending_enrollment.id,
            operator=admin_user
        )

        assert enrollment.state == EnrollmentState.ACTIVE
        assert enrollment.classroom is not None
```

### Frontend (React + Testing Library)

```typescript
// tests/enrollment/EnrollmentForm.test.tsx
describe('EnrollmentForm', () => {
  it('should submit enrollment and show success', async () => {
    render(<EnrollmentForm />);

    await userEvent.selectOptions(screen.getByLabelText('School Year'), '2024-2025');
    await userEvent.selectOptions(screen.getByLabelText('Category'), 'Primaire');

    await userEvent.click(screen.getByText('Submit'));

    await waitFor(() => {
      expect(screen.getByText('Enrollment submitted successfully')).toBeInTheDocument();
    });
  });
});
```

### Flutter (Widget + Integration Tests)

```dart
// test/widgets/grade_entry_test.dart
testWidgets('Teacher can enter grade offline', (tester) async {
  await tester.pumpWidget(TestSetup.withOfflineMode());

  await tester.tap(find.byIcon(Icons.assignment));
  await tester.pumpAndSettle();

  await tester.tap(find.text('Enter Grades'));
  await tester.pumpAndSettle();

  // Enter grade
  await tester.enterText(find.byKey(Key('grade-0')), '15.5');
  await tester.tap(find.byIcon(Icons.save));
  await tester.pumpAndSettle();

  // Verify queued
  final queue = await getOfflineQueue();
  expect(queue.length).toBe(1);
});
```

---

## CI/CD Pipeline

### GitHub Actions Workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test-backend:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7-alpine
    steps:
      - uses: actions/checkout@v3
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - name: Install dependencies
        run: |
          pip install -r requirements/dev.txt
      - name: Run tests
        run: |
          python manage.py test --keepdb
      - name: Lint
        run: |
          black --check .
          isort --check-only .
          flake8 .

  test-frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
      - name: Install
        run: npm ci
      - name: Lint
        run: npm run lint
      - name: Test
        run: npm run test:ci
      - name: Build
        run: npm run build

  test-mobile:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.19.0'
      - name: Install dependencies
        run: flutter pub get
      - name: Analyze
        run: flutter analyze
      - name: Test
        run: flutter test
```

---

## Package Dependencies Summary

### Backend (requirements/*.txt)

```
# requirements/base.txt
Django==5.0.3
djangorestframework==3.15.1
django-cors-headers==4.3.1
django-filter==23.5
psycopg[binary,pool]==3.1.18
redis==5.0.4
celery==5.3.6
drf-yasg==1.21.7  # OpenAPI schema generation

# requirements/dev.txt
pytest==7.4.4
pytest-django==4.7.0
pytest-cov==4.1.0
factory-boy==3.3.0
faker==22.6.0
black==24.2.0
isort==5.13.2
flake8==7.0.0
pre-commit==3.6.0

# requirements/prod.txt
gunicorn==21.2.0
whitenoise==6.7.0
sentry-sdk==1.40.4
```

### Frontend (package.json)

```json
{
  "dependencies": {
    "@tanstack/react-query": "^5.28.4",
    "@tanstack/react-router": "^1.28.1",
    "@tanstack/query-core": "^5.28.4",
    "@tanstack/react-query-devtools": "^5.28.4",
    "@hookform/resolvers": "^3.3.4",
    "react-hook-form": "^7.51.0",
    "zod": "^3.22.4",
    "lucide-react": "^0.363.0",
    "class-variance-authority": "^0.7.0",
    "tailwindcss": "^3.4.3",
    "axios": "^1.6.7"
  },
  "devDependencies": {
    "@tanstack/eslint-plugin-query": "^5.28.4",
    "@types/node": "^20.11.30",
    "@types/react": "^18.2.66",
    "@vitejs/plugin-react": "^4.2.1",
    "typescript": "^5.4.2",
    "vitest": "^1.3.1",
    "@testing-library/react": "^14.2.1",
    "autoprefixer": "^10.4.19",
    "postcss": "^8.4.38"
  }
}
```

### Flutter (pubspec.yaml)

```yaml
name: sms_mobile
description: School Management System Mobile App

environment:
  sdk: '>=3.3.0 <4.0.0'

dependencies:
  flutter:
    sdk: flutter

  # State Management
  flutter_riverpod: ^2.4.9
  riverpod_annotation: ^2.3.3

  # Navigation
  go_router: ^13.0.0

  # Networking
  dio: ^5.4.1
  retrofit: ^4.3.0

  # Local Storage
  isar: ^3.1.0+1
  isar_flutter_libs: ^3.1.0+1
  flutter_secure_storage: ^9.0.0

  # Offline Sync
  workmanager: ^0.5.1
  connectivity_plus: ^5.0.2

  # UI
  flutter_screenutil: ^5.9.3
  shimmer: ^3.0.0
  cached_network_image: ^3.3.1

  # Utils
  freezed: ^2.5.2
  json_annotation: ^4.8.1
  intl: ^0.18.1

dev_dependencies:
  flutter_test:
    sdk: flutter
  build_runner: ^2.4.8
  riverpod_generator: ^2.3.9
  retrofit_generator: ^8.0.6
  isar_generator: ^3.1.0+1
  freezed: ^2.5.2
```

---

## Known Gaps / Future Decisions

| Decision | Status | Notes |
|----------|--------|-------|
| **Search implementation** | Deferred | PostgreSQL full-text search initially, Elasticsearch later if needed |
| **Real-time notifications** | TBD | WebSockets vs Server-Sent Events vs Push notifications |
| **File upload strategy** | TBD | Direct to S3 vs through backend |
| **CDN for static assets** | TBD | CloudFront, Cloudflare, or self-hosted |
| **Monitoring stack** | TBD | Prometheus + Grafana vs Datadog vs custom |
| **Email service** | TBD | SendGrid, AWS SES, Postmark |
| **SMS service** | TBD | Twilio, MessageBird, local provider |

These can be decided during Phase 3 implementation.

---

## Conclusion

This stack provides:
- **Rapid development** — Django admin, TypeScript type safety, Flutter hot reload
- **Scalability** — PostgreSQL partitioning, Redis caching, container deployment
- **Offline capability** — Local DB on all clients, sync protocol in place
- **Maintainability** — Well-known frameworks with large communities
- **Cost-effective** — All components open-source (except optional cloud services)

**Next steps**: Start implementation with Phase 1 (Foundation) per IMPLEMENTATION-ROADMAP.md.
