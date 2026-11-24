Markdown · phase5_v9_design.md

# Phase 5 — Bulk Student Registration: Technical Design v9

**Status:** PENDING USER APPROVAL — design only. No code, migrations, or database changes.\
**Date:** 2026-11-24\
**Supersedes:** v8 (2026-11-24)

---

## Table of Contents

1. Architecture Overview
2. Scope and Non-Goals
3. Database Model and Constraints
4. State Machine
5. T1 — Row Initialization Transaction
6. T2 — Registration Transaction
7. T3 — Result Persistence Transaction
8. Import Loop: Exception Boundaries and State Machine
9. After-Loop Finalization Transaction
10. Reconciliation Transaction
11. Complete Lock-Order Table
12. Complete Child-State Consistency Matrix
13. Counter Definitions and Invariants
14. Audit Evidence and Reconciliation Rules
15. Failure and Recovery Behavior
16. API Contracts
17. CSV Preview and Confirmation
18. Expiry and Retention
19. Security and RBAC
20. Email Behavior
21. Logging and Privacy
22. Testing Strategy
23. Implementation Milestones
24. v8 Improvements — Explicit Revalidation\
    A. Resolved Issues (v9 corrections)\
    B. Pending User Decisions\
    C. Implementation Readiness

---

## §1. Architecture Overview

Phase 5 adds CSV-based bulk student registration for Main Admin, processing up to 100 rows synchronously in a single HTTP request. It reuses the existing `register_student()` method per row. No background workers, no queues, no new infrastructure.

**Modular structure (new files in `app/modules/students/`):**

- `bulk_models.py` — SQLAlchemy ORM models for the two new tables
- `bulk_service.py` — upload, confirm/process, reconcile, results
- `bulk_schemas.py` — Pydantic request/response models
- `bulk_constants.py` — enums, `BulkImportOrigin`, `BulkImportSessionFencedError`, `ImporterState`
- `bulk_router.py` — FastAPI routes

**Inherited constraints from Phase 4 (all preserved, none changed):**

- FastAPI modular monolith: Router → Service → Repository
- SQLAlchemy 2.0 sync, `autoflush=False`, `expire_on_commit=False`
- One `Session` per request via `get_db()`
- `register_student()` manages its own transaction (commits at service.py:309)
- `AuditService.record()` only flushes — joins caller's transaction; never commits independently
- Email via `BackgroundTasks` (at-most-once, deferred, never raises)
- PostgreSQL READ COMMITTED isolation throughout
- `pool_pre_ping=True`

**v9 central design principle — parent-first lock order:**\
Every database transaction in the bulk import subsystem acquires an appropriate lock on the `bulk_upload_sessions` parent row **before** accessing any child row in `bulk_upload_row_results`. This makes the lock dependency graph acyclic and structurally eliminates deadlock.

---

## §2. Scope and Non-Goals

**In scope:**

- CSV upload (≤5 MB, ≤100 rows), preview, confirmation
- Synchronous row-by-row registration using `register_student()`
- Durable per-row progress via `bulk_upload_row_results` child table
- Partial success (each row is independent)
- After-loop finalization (terminal write or interrupted audit)
- Manual reconciliation for sessions interrupted by crashes
- CSV downloadable report

**Not in scope (explicitly deferred):**

- Section placement movement, promotion, freeze (Phase 5A)
- Student profile pictures
- Instructor department assignment
- Asynchronous background processing, Redis, Celery, RQ
- Multi-file batches or pagination of import
- HOD delegated bulk import (permission not delegable)
- Any change to Phase 4 `register_student()` behavior or its approved transaction model
- Any purge mechanism (retention is indefinite pending future phase)

---

## §3. Database Model and Constraints

### §3.1. `bulk_upload_sessions`

```sql
CREATE TABLE bulk_upload_sessions (
    id                        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    uploaded_by_user_id       UUID NOT NULL REFERENCES users(id),
    department_id             UUID NOT NULL REFERENCES departments(id),
    file_name                 VARCHAR(255) NOT NULL,
    file_size_bytes           INTEGER NOT NULL,
    file_sha256               VARCHAR(64) NOT NULL,

    -- Preview (immutable after upload)
    total_rows                INTEGER NOT NULL,
    valid_count               INTEGER NOT NULL,
    invalid_count             INTEGER NOT NULL,
    validated_rows            JSONB NOT NULL DEFAULT '[]',
    preview_invalid_results   JSONB NOT NULL DEFAULT '[]',

    -- Parent counters (authoritative source: child table)
    -- STARTED rows are NOT tracked by any parent counter.
    imported_count            INTEGER NOT NULL DEFAULT 0,   -- SUCCEEDED rows
    failed_count              INTEGER NOT NULL DEFAULT 0,   -- FAILED rows
    not_processed_count       INTEGER NOT NULL DEFAULT 0,   -- NOT_PROCESSED rows
    uncertain_count           INTEGER NOT NULL DEFAULT 0,   -- OUTCOME_UNCERTAIN rows only

    -- Progress heartbeat
    last_started_valid_index  INTEGER NOT NULL DEFAULT 0,

    -- Lifecycle
    status                    VARCHAR(30) NOT NULL DEFAULT 'PENDING_CONFIRMATION',
    expires_at                TIMESTAMPTZ NOT NULL,                -- immutable
    confirmed_by_user_id      UUID REFERENCES users(id),
    confirmed_at              TIMESTAMPTZ,
    processing_started_at     TIMESTAMPTZ,
    processing_heartbeat_at   TIMESTAMPTZ,
    completed_at              TIMESTAMPTZ,

    -- Reconciliation
    reconciliation_claimed_by  UUID REFERENCES users(id),
    reconciliation_started_at  TIMESTAMPTZ,
    reconciliation_claim_token UUID,
    was_reconciled             BOOLEAN NOT NULL DEFAULT FALSE,

    created_at                TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at                TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT ck_bulk_sessions_status CHECK (
        status IN (
            'PENDING_CONFIRMATION', 'EXPIRED',
            'PROCESSING', 'RECONCILING',
            'COMPLETED', 'PARTIALLY_COMPLETED', 'FAILED'
        )
    ),
    CONSTRAINT ck_bulk_sessions_total_rows CHECK (total_rows BETWEEN 1 AND 100),
    CONSTRAINT ck_bulk_sessions_preview_counts CHECK (
        valid_count >= 0 AND invalid_count >= 0
        AND total_rows = valid_count + invalid_count
    ),
    CONSTRAINT ck_bulk_sessions_import_counts CHECK (
        imported_count >= 0 AND failed_count >= 0
        AND not_processed_count >= 0 AND uncertain_count >= 0
        AND (CASE
            WHEN status IN ('COMPLETED','PARTIALLY_COMPLETED','FAILED')
                THEN imported_count + failed_count + not_processed_count = valid_count
                     AND uncertain_count = 0
            ELSE imported_count + failed_count + not_processed_count
                 + uncertain_count <= valid_count
        END)
    ),
    CONSTRAINT ck_bulk_sessions_file_size CHECK (file_size_bytes BETWEEN 1 AND 5000000),
    CONSTRAINT ck_bulk_sessions_progress CHECK (
        last_started_valid_index >= 0
        AND last_started_valid_index <= valid_count
    )
);
```

**RECONCILING in CHECK constraint:** Required so the in-transaction `SET status='RECONCILING'` passes CHECK at statement time. RECONCILING is never the committed value — the reconciliation transaction always commits with a terminal status or rolls back entirely.

**Immutable columns (not in UPDATE privilege grant):** `id`, `uploaded_by_user_id`, `department_id`, `file_name`, `file_size_bytes`, `file_sha256`, `total_rows`, `valid_count`, `invalid_count`, `preview_invalid_results`, `expires_at`, `created_at`

### §3.2. `bulk_upload_row_results`

```sql
CREATE TABLE bulk_upload_row_results (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    upload_session_id     UUID NOT NULL
        REFERENCES bulk_upload_sessions(id) ON DELETE RESTRICT,
    valid_row_index       INTEGER NOT NULL,
    csv_record_number     INTEGER NOT NULL,

    status                VARCHAR(30) NOT NULL DEFAULT 'STARTED',

    -- References students(id), NOT users(id).
    -- students.id is a shared PK/FK with users.id (Phase 4 schema).
    -- A user is not necessarily a student; this FK guarantees the
    -- referenced record is an actual student account.
    -- Students are never hard-deleted → ON DELETE RESTRICT is correct.
    student_id            UUID REFERENCES students(id) ON DELETE RESTRICT,

    error_code            VARCHAR(50),
    error_message         VARCHAR(500),
    corrective_action     VARCHAR(500),
    is_retryable          BOOLEAN,

    -- Nullable three-state: TRUE=invoked, FALSE=not invoked, NULL=unknown
    was_attempted         BOOLEAN,

    -- Retained for the report and admin review
    first_name            VARCHAR(100),
    last_name             VARCHAR(100),
    roll_number           VARCHAR(30),

    created_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT uq_row_results_session_index
        UNIQUE (upload_session_id, valid_row_index),
    CONSTRAINT ck_row_results_status CHECK (
        status IN (
            'STARTED', 'SUCCEEDED', 'FAILED',
            'NOT_PROCESSED', 'OUTCOME_UNCERTAIN'
        )
    ),
    CONSTRAINT ck_row_results_valid_row_index   CHECK (valid_row_index >= 1),
    CONSTRAINT ck_row_results_csv_record_number CHECK (csv_record_number >= 1)
);

CREATE INDEX ix_row_results_session
    ON bulk_upload_row_results (upload_session_id);
```

**Why `students(id)` and not `users(id)` (v9 correction 2):**\
Phase 4 uses a shared-PK pattern: `students.id` is simultaneously the student's PK and an FK to `users.id` (see `fk_students_user_id` in models.py:44). A user record exists for all account types; a student record exists only for students. Referencing `users(id)` would allow pointing at a staff member's or HOD's user record. Referencing `students(id)` enforces that the UUID corresponds to an actual registered student. No existing Phase 4 relationship is affected: this is a new column on a new table.

### §3.3. `validated_rows` JSONB structure

Each element in `validated_rows` MUST contain at minimum:

```json
{
  "valid_row_index": 3,
  "csv_record_number": 5,
  "first_name": "...",
  "last_name": "...",
  "email": "...",
  "roll_number": "...",
  "program_code": "...",
  "semester_level": 2,
  "admission_term": "...",
  "section_name": null,
  "phone": null,
  "address": null
}
```

`valid_row_index` MUST be present as an integer in every element. Reconciliation (§10.3) reads this array to build the authoritative set of valid indexes. **PENDING USER APPROVAL — D9** (confirm this structure before M1 upload implementation).

### §3.4. Privilege additions to `provision_db_roles.py`

Add to `NO_DELETE_TABLES`:

```python
"bulk_upload_sessions",
"bulk_upload_row_results",
```

Add to `COLUMN_UPDATE_TABLES`:

```python
"bulk_upload_sessions": (
    "status", "validated_rows",
    "imported_count", "failed_count",
    "not_processed_count", "uncertain_count",
    "last_started_valid_index",
    "confirmed_by_user_id", "confirmed_at",
    "processing_started_at", "processing_heartbeat_at", "completed_at",
    "reconciliation_claimed_by", "reconciliation_started_at",
    "reconciliation_claim_token", "was_reconciled",
    "updated_at",
),
"bulk_upload_row_results": (
    "status", "student_id",
    "error_code", "error_message",
    "corrective_action", "is_retryable",
    "was_attempted",
    "first_name", "last_name", "roll_number",
    "updated_at",
),
```

---

## §4. State Machine

### §4.1. Externally observable states

| State | API-visible | Meaning |
| --- | --- | --- |
| PENDING_CONFIRMATION | Yes | Uploaded; awaiting admin confirm within 30 min |
| EXPIRED | Yes | 30-min window elapsed without confirm |
| PROCESSING | Yes | Confirmed; import running, interrupted, or awaiting reconciliation |
| COMPLETED | Yes | All valid rows SUCCEEDED |
| PARTIALLY_COMPLETED | Yes | Some SUCCEEDED, rest FAILED/NOT_PROCESSED |
| FAILED | Yes | Zero rows SUCCEEDED |

**RECONCILING** is a transient in-transaction state. It is SET inside the reconciliation transaction but the transaction always commits with a terminal status or rolls back. Under READ COMMITTED, concurrent readers never observe RECONCILING as a committed value. The API GET endpoint never returns RECONCILING. The stale-RECONCILING reclaim logic from v7 is removed — it was meaningless under the single-transaction model.

### §4.2. Allowed transitions

| From | Trigger | To | Condition |
| --- | --- | --- | --- |
| PENDING_CONFIRMATION | Confirm endpoint | PROCESSING | `expires_at > now()` |
| PENDING_CONFIRMATION | Any read (lazy) | EXPIRED | `expires_at ≤ now()` |
| PROCESSING | After-loop: all rows definitive | COMPLETED | `imported = valid_count` |
| PROCESSING | After-loop: all rows definitive | PARTIALLY_COMPLETED | `0 < imported < valid_count` |
| PROCESSING | After-loop: all rows definitive | FAILED | `imported = 0` |
| PROCESSING | Crash/interrupt (no after-loop commit) | PROCESSING (stays) | Heartbeat goes stale |
| PROCESSING | Reconciliation: success | COMPLETED / PARTIALLY_COMPLETED / FAILED | All rows resolved, no contradictions |
| PROCESSING | Reconciliation: contradiction found | PROCESSING (stays) | Rolls back |

Disallowed: any terminal → anything; PROCESSING → EXPIRED.

---

## §5. T1 — Row Initialization Transaction

### §5.1. Purpose

Reserve a child-table slot for a specific `valid_row_index` before calling `register_student()`. Provides a durable STARTED marker for crash recovery and prevents duplicate T2 invocation on retry.

### §5.2. Lock acquisition order

1. `SELECT ... FOR UPDATE` on `bulk_upload_sessions` — acquires **exclusive** row lock on parent
2. `INSERT INTO bulk_upload_row_results ... ON CONFLICT DO NOTHING` — locks child row (new insert)
3. `UPDATE bulk_upload_sessions SET heartbeat ...` — updates parent (lock already held)
4. `COMMIT`

Parent is locked exclusively **before** any child mutation. No transaction can hold a child lock while acquiring the parent lock.

### §5.3. Complete SQL sequence

```sql
BEGIN;
SET LOCAL lock_timeout = '[D2: 5s recommended]';

-- Step 1: Lock parent exclusively; verify PROCESSING
SELECT id
FROM bulk_upload_sessions
WHERE id = :session_id
  AND status = 'PROCESSING'
FOR UPDATE;
-- rowcount = 0 → ROLLBACK → T1Result.FENCED
-- rowcount = 1 → parent exclusively locked; proceed

-- Step 2: Insert child row (idempotent)
INSERT INTO bulk_upload_row_results
    (upload_session_id, valid_row_index, csv_record_number,
     status, was_attempted)
VALUES
    (:session_id, :index, :csv_record_number, 'STARTED', NULL)
ON CONFLICT (upload_session_id, valid_row_index) DO NOTHING
RETURNING id;
-- rowcount = 1 → new row; proceed to step 3
-- rowcount = 0 → conflict: row already exists from prior committed T1
--               ROLLBACK → T1Result.SKIP

-- Step 3: Update parent heartbeat and progress (parent lock still held)
UPDATE bulk_upload_sessions
SET last_started_valid_index = :index,
    processing_heartbeat_at  = now(),
    updated_at               = now()
WHERE id = :session_id;
-- rowcount = 1 (guaranteed; we hold the exclusive lock)

COMMIT;
-- Result: T1Result.PROCEED
```

### §5.4. ON CONFLICT behavior with parent lock held

We hold the parent exclusive lock before the INSERT. No other T1 for the same `(session_id, valid_row_index)` can be executing concurrently — it would be blocked at step 1. Therefore, a conflict means the child row was inserted by a **previously committed T1** (from a prior loop execution or prior request). The correct action is ROLLBACK + SKIP.

### §5.5. Conflict behavior by existing child status

| Existing status | T1 conflict action | T2 called? | Rationale |
| --- | --- | --- | --- |
| STARTED | SKIP | No | Row initialized; may be in reconciliation |
| SUCCEEDED | SKIP | No | Already registered |
| FAILED | SKIP | No | Already classified |
| NOT_PROCESSED | SKIP | No | Already classified |
| OUTCOME_UNCERTAIN | SKIP | No | Awaiting reconciliation |

We do not inspect the existing status during T1. Existence alone is sufficient to skip.

### §5.6. Outcomes and loop actions

| Result | Description | Loop action |
| --- | --- | --- |
| T1Result.PROCEED | New row inserted; parent updated | Call T2 for this row |
| T1Result.SKIP | Conflict; row already exists | Continue to next row |
| T1Result.FENCED | Parent not PROCESSING | Break loop — FENCED |
| Exception (lock_timeout, connection) | ROLLBACK | Break loop — INTERRUPTED |

### §5.7. Idempotency

If T1 times out and the entire transaction rolled back, the child row does not exist. The next T1 attempt inserts cleanly (no conflict). If T1 committed but the application crashed before reading the result, the next iteration detects the conflict (rowcount=0) and returns SKIP.

---

## §6. T2 — Registration Transaction

### §6.1. Overview

T2 is `register_student()` from service.py:212–335. It owns its own transaction boundary:

- Commits at service.py:309 (student + placement + audit all committed atomically)
- Rolls back at service.py:248 (validation failure), 311 (IntegrityError), 322 (unexpected error)

The import loop does not control T2's transaction. T2 runs after T1 commits and before T3 starts.

### §6.2. BulkImportOrigin and FOR SHARE fencing

```python
@dataclass(frozen=True)
class BulkImportOrigin:
    upload_session_id: uuid.UUID
    row_index: int  # valid_row_index
```

When `bulk_origin` is provided, `register_student()` acquires FOR SHARE on the parent session before `_lock_and_validate()`:

```python
if bulk_origin is not None:
    session_status = db.scalar(
        select(BulkUploadSession.status)
        .where(BulkUploadSession.id == bulk_origin.upload_session_id)
        .with_for_update(read=True)   # FOR SHARE
    )
    if session_status != BulkUploadStatus.PROCESSING.value:
        raise BulkImportSessionFencedError(...)
```

T2's lock order is strictly parent-first:

1. `bulk_upload_sessions` FOR SHARE
2. `academic_terms` FOR SHARE (via `_lock_and_validate()`)
3. `departments` FOR SHARE
4. `programs` FOR SHARE
5. `sections` FOR SHARE (if applicable)
6. `users`/`students`/`student_placements` FOR UPDATE (correction path only)

### §6.3. Interaction with T1 and T3

T1 (which held the parent FOR UPDATE) commits before T2 starts. T2 acquires only FOR SHARE on the parent. If after-loop or reconciliation holds the parent FOR UPDATE when T2 tries FOR SHARE, T2 blocks until the holder releases, then reads the current status.

### §6.4. Audit in T2

`AuditService.record()` is called inside `register_student()`. When `bulk_origin` is provided, the audit metadata includes:

```python
metadata["bulk_upload_session_id"] = str(bulk_origin.upload_session_id)
metadata["bulk_upload_valid_row_index"] = bulk_origin.row_index
```

The audit record is committed atomically with the student at T2's commit (service.py:309). No committed student lacks a matching audit record; no audit record for a valid row index exists without a committed student.

### §6.5. T2 outcomes

| Outcome | Description |
| --- | --- |
| `RegistrationResult` returned | Student committed; audit committed in same transaction |
| `StudentConflictError` | Known conflict (duplicate roll number, email); T2 rolled back |
| `StudentValidationError` | Input invalid; T2 rolled back |
| `StudentReferenceNotFoundError` | Referenced entity not found; T2 rolled back |
| `BulkImportSessionFencedError` | Session no longer PROCESSING; T2 never proceeded past fencing check |
| Unexpected `Exception` | Unknown error; T2 may or may not have committed |

---

## §7. T3 — Result Persistence Transaction

### §7.1. Purpose

Persist the definitive outcome (SUCCEEDED, FAILED, or OUTCOME_UNCERTAIN) to the child row and increment the matching parent counter. The child row transitions from STARTED to a terminal-or-uncertain status.

### §7.2. Lock acquisition order

1. `SELECT ... FOR UPDATE` on `bulk_upload_sessions` — exclusive parent lock
2. `UPDATE bulk_upload_row_results WHERE status='STARTED'` — child row lock (implicit)
3. `UPDATE bulk_upload_sessions SET counter += 1` — parent update (lock already held)
4. `COMMIT`

Parent locked exclusively before child is touched. The v8 deadlock scenario (T3 holds child lock → reconciliation holds parent lock → deadlock) is eliminated.

### §7.3. Complete SQL sequence

```sql
BEGIN;
SET LOCAL lock_timeout = '[D2: 5s recommended]';

-- Step 1: Lock parent exclusively; verify PROCESSING
SELECT id
FROM bulk_upload_sessions
WHERE id = :session_id
  AND status = 'PROCESSING'
FOR UPDATE;
-- rowcount = 0 → ROLLBACK → T3Result.FENCED

-- Step 2: Update child (guard: only if currently STARTED)
UPDATE bulk_upload_row_results
SET status            = :outcome,
    student_id        = :student_id,    -- NULL for FAILED and OUTCOME_UNCERTAIN
    was_attempted     = TRUE,
    error_code        = :ec,
    error_message     = :em,
    corrective_action = :ca,
    is_retryable      = :ir,
    first_name        = :fn,
    last_name         = :ln,
    roll_number       = :rn,
    updated_at        = now()
WHERE upload_session_id = :session_id
  AND valid_row_index   = :index
  AND status            = 'STARTED';
-- rowcount = 1 → updated; proceed to step 3
-- rowcount = 0 → see §7.4

-- Step 3: Increment parent counter (parent lock still held)
UPDATE bulk_upload_sessions
SET <outcome>_count      = <outcome>_count + 1,
    processing_heartbeat_at = now(),
    updated_at           = now()
WHERE id = :session_id;
-- rowcount = 1 (guaranteed; we hold the exclusive lock)

COMMIT;
-- Result: T3Result.OK
```

`<outcome>_count` mapping:

- SUCCEEDED → `imported_count`
- FAILED → `failed_count`
- OUTCOME_UNCERTAIN → `uncertain_count`

NOT_PROCESSED is never written by T3. NOT_PROCESSED is inserted by after-loop (never-started rows) and reconciliation.

### §7.4. Zero-row child update (step 2 rowcount = 0)

```sql
-- Check existing status for consistency
SELECT status, student_id
FROM bulk_upload_row_results
WHERE upload_session_id = :session_id
  AND valid_row_index   = :index;
ROLLBACK;
```

| Existing | Intended | Result |
| --- | --- | --- |
| NULL (no row) | any | T3Result.INCONSISTENT (T1 never committed or rolled back) |
| `:outcome` | `:outcome` | T3Result.ALREADY_WRITTEN (idempotent) |
| other terminal | `:outcome` | T3Result.INCONSISTENT (status mismatch) |

Since we held the parent exclusive lock, no concurrent T3 could have changed the row. INCONSISTENT means an application-level inconsistency; break loop INTERRUPTED.

### §7.5. Retry behavior

If T3 times out and rolls back: child row is still STARTED (the rolled-back T3 made no changes). Retry: step 2 succeeds normally.

If T3 committed but the caller crashed before reading the result: on retry, step 2 returns rowcount=0. Step 2 check finds existing status = `:outcome` → T3Result.ALREADY_WRITTEN. No duplicate counter increment.

---

## §8. Import Loop: Exception Boundaries and State Machine

### §8.1. ImporterState enum

```python
class ImporterState(StrEnum):
    NORMAL      = "NORMAL"       # Loop completed without interruption
    FENCED      = "FENCED"       # Session claimed by reconciliation
    INTERRUPTED = "INTERRUPTED"  # Unexpected failure; some rows unresolved
```

### §8.2. Full loop with separated exception boundaries

```python
importer_state = ImporterState.NORMAL

for vr in valid_rows:
    index   = vr["valid_row_index"]
    csv_rec = vr["csv_record_number"]

    # ── T1: Row initialization ──────────────────────────────────────────
    try:
        t1 = persist_started(db, session_id, index, csv_rec)
    except Exception:
        # lock_timeout or connection failure
        importer_state = ImporterState.INTERRUPTED
        break

    if t1 == T1Result.FENCED:
        importer_state = ImporterState.FENCED
        break
    if t1 == T1Result.SKIP:
        continue

    # ── T2: Registration (separate try) ─────────────────────────────────
    reg_result = None
    reg_error  = None
    try:
        reg_result = student_service.register_student(
            actor         = actor,
            department_id = department_id,
            data          = make_registration_data(vr),
            ip_address    = ip_address,
            bulk_origin   = BulkImportOrigin(session_id, index),
        )
    except BulkImportSessionFencedError:
        importer_state = ImporterState.FENCED
        break
    except (StudentConflictError, StudentValidationError,
            StudentReferenceNotFoundError) as exc:
        reg_error = exc
    except Exception:
        # Unexpected T2 failure. Student existence is unknown.
        try:
            persist_outcome(db, session_id, index,
                outcome    = RowOutcome.OUTCOME_UNCERTAIN,
                student_id = None,
                first_name = vr.get("first_name"),
                last_name  = vr.get("last_name"),
                roll_number= vr.get("roll_number"))
        except Exception:
            pass  # child stays STARTED; reconciliation handles it
        importer_state = ImporterState.INTERRUPTED
        break

    # ── T3: Result persistence (separate try) ───────────────────────────
    if reg_result is not None:
        # T2 succeeded — persist SUCCEEDED
        try:
            t3 = persist_outcome(db, session_id, index,
                outcome     = RowOutcome.SUCCEEDED,
                student_id  = reg_result.student.id,
                first_name  = reg_result.user.first_name,
                last_name   = reg_result.user.last_name,
                roll_number = reg_result.student.roll_number)
        except Exception:
            # T3 raised. Student IS registered (T2 committed + audit).
            # Child stays STARTED. Reconciliation: audit → SUCCEEDED.
            importer_state = ImporterState.INTERRUPTED
            break

        if t3 == T3Result.FENCED:
            # Parent no longer PROCESSING. Student IS registered.
            # Reconciliation will find the audit and resolve this row.
            importer_state = ImporterState.FENCED
            break
        if t3 == T3Result.INCONSISTENT:
            importer_state = ImporterState.INTERRUPTED
            break

        # ── Email (separate try — failure is non-fatal) ─────────────────
        try:
            schedule_email(background_tasks, email_sender,
                build_student_onboarding_email(
                    to           = reg_result.user.email,
                    first_name   = reg_result.user.first_name,
                    program_name = reg_result.program.name,
                    department_name = reg_result.department.name))
        except Exception:
            logger.warning("Email scheduling failed for row %d (session %s)",
                           index, session_id, exc_info=True)
        # Always continue to next row regardless of email outcome.

    elif reg_error is not None:
        # T2 raised a business error — persist FAILED
        try:
            t3 = persist_outcome(db, session_id, index,
                outcome          = RowOutcome.FAILED,
                student_id       = None,
                error_code       = classify_error(reg_error),
                error_message    = str(reg_error),
                corrective_action= suggest_correction(reg_error),
                is_retryable     = is_retryable(reg_error),
                first_name       = vr.get("first_name"),
                last_name        = vr.get("last_name"),
                roll_number      = vr.get("roll_number"))
        except Exception:
            # T3 raised. T2 rolled back; no student. Child stays STARTED.
            # Reconciliation: no audit → NOT_PROCESSED (was_attempted=NULL).
            importer_state = ImporterState.INTERRUPTED
            break

        if t3 == T3Result.FENCED:
            importer_state = ImporterState.FENCED
            break
        if t3 == T3Result.INCONSISTENT:
            importer_state = ImporterState.INTERRUPTED
            break
```

### §8.3. Exception boundary summary

| Failure scenario | Child state | Student exists? | Loop action |
| --- | --- | --- | --- |
| T1 lock_timeout | No child | No | INTERRUPTED, break |
| T1 FENCED | No child | No | FENCED, break |
| T2 business error | STARTED | No | T3→FAILED, continue |
| T2 unexpected error | STARTED→OUTCOME_UNCERTAIN (or STARTED if T3 also fails) | Unknown | INTERRUPTED, break |
| T2 FENCED | STARTED | No | FENCED, break |
| T3 exception (after T2 success) | STARTED | **Yes** | INTERRUPTED, break |
| T3 FENCED (after T2 success) | STARTED | **Yes** | FENCED, break |
| T3 exception (after T2 error) | STARTED | No | INTERRUPTED, break |
| Email failure (after SUCCEEDED) | SUCCEEDED (unchanged) | Yes | Warn + continue |

---

## §9. After-Loop Finalization Transaction

### §9.1. When after-loop runs

After the import loop exits, based on `importer_state`:

- **FENCED**: do nothing; reconciliation owns the session.
- **NORMAL or INTERRUPTED**: attempt ownership acquisition and finalization.

### §9.2. Lock acquisition order

1. `UPDATE bulk_upload_sessions SET processing_heartbeat_at=now() WHERE id=:id AND status='PROCESSING'` — acquires exclusive row lock implicitly; also verifies PROCESSING.
2. `SELECT valid_row_index FROM bulk_upload_row_results WHERE upload_session_id=:id` — read children (parent lock held).
3. `INSERT INTO bulk_upload_row_results ... ON CONFLICT DO NOTHING` — for never-started rows.
4. `SELECT status, count(*) FROM bulk_upload_row_results WHERE upload_session_id=:id GROUP BY status` — recalculate counters.
5. `UPDATE bulk_upload_sessions SET status=terminal | counters ...` — write outcome.
6. `AuditService.record()` → flush.
7. `COMMIT`.

### §9.3. Full algorithm

```python
if importer_state == ImporterState.FENCED:
    return   # Reconciliation owns this session; do nothing

# importer_state is NORMAL or INTERRUPTED
try:
    db.execute(text("SET LOCAL lock_timeout = '[D2: 5s recommended]'"))

    # ── Step 0: Acquire ownership (exclusive lock on parent) ──
    ownership_rows = db.execute(
        update(BulkUploadSession)
        .where(BulkUploadSession.id == session_id)
        .where(BulkUploadSession.status == 'PROCESSING')
        .values(processing_heartbeat_at=func.now(),
                updated_at=func.now())
    ).rowcount

    if ownership_rows == 0:
        # Session no longer PROCESSING (reconciliation committed terminal,
        # or state changed). Do not proceed.
        db.rollback()
        return

    # ── Step 1: Find valid indexes with no child row ──
    existing_indexes = set(db.scalars(
        select(BulkUploadRowResult.valid_row_index)
        .where(BulkUploadRowResult.upload_session_id == session_id)
    ).all())
    all_indexes   = set(range(1, valid_count + 1))
    never_started = all_indexes - existing_indexes

    # ── Step 2: Insert NOT_PROCESSED for never-started ──
    for mi in sorted(never_started):
        csv_rec = index_to_csv_record[mi]  # from validated_rows mapping
        db.execute(
            pg_insert(BulkUploadRowResult).values(
                upload_session_id=session_id,
                valid_row_index  =mi,
                csv_record_number=csv_rec,
                status           ='NOT_PROCESSED',
                was_attempted    =False
            ).on_conflict_do_nothing()   # safety: idempotent
        )

    # ── Step 3: Recalculate ALL counters from child table ──
    # This is a full recalculation, not an increment.
    counts = {
        status: cnt
        for status, cnt in db.execute(
            select(BulkUploadRowResult.status, func.count())
            .where(BulkUploadRowResult.upload_session_id == session_id)
            .group_by(BulkUploadRowResult.status)
        ).all()
    }
    succeeded    = counts.get('SUCCEEDED', 0)
    failed       = counts.get('FAILED', 0)
    not_processed= counts.get('NOT_PROCESSED', 0)
    uncertain    = counts.get('OUTCOME_UNCERTAIN', 0)
    started      = counts.get('STARTED', 0)
    # uncertain_count tracks ONLY OUTCOME_UNCERTAIN; STARTED is not counted.
    unresolved = uncertain + started

    # ── Step 4a: Unresolved rows remain → no terminal; write interrupted ──
    if unresolved > 0:
        db.execute(
            update(BulkUploadSession)
            .where(BulkUploadSession.id == session_id)
            .where(BulkUploadSession.status == 'PROCESSING')
            .values(
                imported_count      =succeeded,
                failed_count        =failed,
                not_processed_count =not_processed,
                uncertain_count     =uncertain,   # STARTED not included
                updated_at          =func.now())
        )
        audit_service.record(
            action       =AuditAction.BULK_IMPORT_INTERRUPTED,
            entity_type  =AuditEntityType.BULK_UPLOAD_SESSION,
            entity_id    =session_id,
            actor_user_id=actor.id,
            metadata     ={
                "unresolved_count": unresolved,
                "started_count"   : started,
                "uncertain_count" : uncertain,
                "imported_count"  : succeeded,
                "failed_count"    : failed,
            },
            ip_address=ip_address)
        db.commit()
        return

    # ── Step 4b: All rows definitive → verify totals → write terminal ──
    total = succeeded + failed + not_processed
    if total != valid_count:
        # Inconsistent count — do not terminal-write; leave for reconciliation
        db.rollback()
        return

    terminal = (
        'COMPLETED'           if succeeded == valid_count
        else 'FAILED'         if succeeded == 0
        else 'PARTIALLY_COMPLETED'
    )

    db.execute(
        update(BulkUploadSession)
        .where(BulkUploadSession.id == session_id)
        .where(BulkUploadSession.status == 'PROCESSING')
        .values(
            status              =terminal,
            imported_count      =succeeded,
            failed_count        =failed,
            not_processed_count =not_processed,
            uncertain_count     =0,
            validated_rows      =cast('[]', JSONB),
            completed_at        =func.now(),
            updated_at          =func.now())
    )
    audit_service.record(
        action       =AuditAction.BULK_IMPORT_COMPLETED,
        entity_type  =AuditEntityType.BULK_UPLOAD_SESSION,
        entity_id    =session_id,
        actor_user_id=actor.id,
        metadata     ={
            "status"           : terminal,
            "imported_count"   : succeeded,
            "failed_count"     : failed,
            "not_processed_count": not_processed,
        },
        ip_address=ip_address)
    db.commit()

except Exception:
    try:
        db.rollback()
    except Exception:
        pass
    # Session stays PROCESSING with stale heartbeat → reconciliation.
```

### §9.4. STARTED rows are not reclassified by after-loop

After-loop only inserts NOT_PROCESSED for rows with **no child record at all** (never-started). Rows already in STARTED, OUTCOME_UNCERTAIN, or any terminal status are left as-is. STARTED rows remain for reconciliation to resolve.

### §9.5. Ownership-conflict proof

After-loop acquires the parent exclusive lock via the heartbeat UPDATE. Reconciliation also needs the parent exclusive lock (via RECONCILING UPDATE). They cannot both hold it simultaneously:

- If after-loop holds lock first → reconciliation blocks → after-loop commits terminal → reconciliation sees terminal → 409 (not PROCESSING).
- If reconciliation holds lock first → after-loop blocks → reconciliation commits terminal → after-loop `ownership_rows = 0` → returns.
- If reconciliation rolls back (contradiction) → after-loop unblocks, sees PROCESSING → proceeds.

---

## §10. Reconciliation Transaction

### §10.1. Single-transaction model

All reconciliation work — claim, read, classify, write, recalculate counters, terminal write, audit — runs in one transaction. The exclusive row lock on the parent session is held from the claim UPDATE through COMMIT or ROLLBACK. This is not a multi-step protocol; it is a single atomic operation.

### §10.2. Claim (parent-first)

```sql
BEGIN;
SET LOCAL lock_timeout = '[D3: 10s recommended]';

UPDATE bulk_upload_sessions
SET status                    = 'RECONCILING',
    reconciliation_claimed_by = :actor_id,
    reconciliation_started_at = now(),
    reconciliation_claim_token= :new_token_uuid,
    updated_at                = now()
WHERE id                        = :upload_id
  AND status                    = 'PROCESSING'
  AND processing_heartbeat_at   < now() - INTERVAL '[D1: 5 min recommended]'
RETURNING id, valid_count, validated_rows;
-- rowcount = 0:
--   if lock_timeout: ROLLBACK → 503
--   otherwise:       ROLLBACK → 409 (not eligible)
-- rowcount = 1: parent exclusively locked; proceed
```

No stale-RECONCILING reclaim. The WHERE clause checks only `status='PROCESSING'`. If a previous reconciliation transaction rolled back (e.g., due to contradiction), the session reverts to PROCESSING with a stale heartbeat, and this same claim succeeds on retry.

### §10.3. Read phase (within reconciliation transaction)

**Step 1 — Load validated_rows and extract authoritative valid index set:**

```python
validated_rows_data = session_row["validated_rows"]   # list of dicts
if not validated_rows_data:
    # validated_rows should never be [] while status is PROCESSING.
    # If it is, we cannot safely validate audit indexes.
    db.rollback()
    raise ReconciliationError("validated_rows is empty; cannot safely reconcile")

valid_index_set: set[int] = {
    vr["valid_row_index"]
    for vr in validated_rows_data
}
# valid_index_set is the authoritative set, derived from the upload preview.
# It is strictly stronger than relying on valid_count alone.
```

**Step 2 — Load all existing child rows:**

```sql
SELECT valid_row_index, status, student_id, was_attempted
FROM bulk_upload_row_results
WHERE upload_session_id = :upload_id;
```

**Step 3 — Load audit records for this session:**

```sql
SELECT
    entity_id,
    event_metadata->>'bulk_upload_valid_row_index' AS raw_index,
    event_metadata->>'bulk_upload_session_id'       AS raw_session_id
FROM audit_logs
WHERE action = 'student.registered'
  AND event_metadata->>'bulk_upload_session_id' = :upload_id_str;
```

Uses the existing `ix_audit_logs_action` index on `action`.

**Step 4 — Validate every audit record (all failures → contradiction):**

For each audit record, perform ALL of the following checks:

| Check | Failure condition | Contradiction type |
| --- | --- | --- |
| Session ID present | `raw_session_id` is NULL | `SESSION_ID_MISSING` |
| Session ID matches | `raw_session_id != upload_id_str` | `SESSION_ID_MISMATCH` |
| Index present | `raw_index` is NULL | `INDEX_MISSING` |
| Index is integer, not boolean | `raw_index` is JSON boolean, or cannot be parsed as int | `INDEX_NOT_INTEGER` |
| Index is positive | `parsed_index < 1` | `INDEX_NOT_POSITIVE` |
| Index within valid_count | `parsed_index > valid_count` | `INDEX_OUT_OF_RANGE` |
| Index in validated_rows set | `parsed_index not in valid_index_set` | `INDEX_NOT_IN_VALIDATED_ROWS` |
| Entity ID present | `entity_id is NULL` | `ENTITY_ID_MISSING` |
| No duplicate index | Another record has same `parsed_index` | `DUPLICATE_AUDIT_INDEX` |

If any audit record fails any check → collect the contradiction. After iterating all records, if contradictions exist → §10.4.

**Step 5 — Build audit map:**

```python
audit_by_index: dict[int, uuid.UUID] = {
    record.parsed_index: record.entity_id
    for record in validated_audit_records
}
child_by_index: dict[int, ChildRow] = {
    row.valid_row_index: row
    for row in child_rows
}
```

### §10.4. Fail-closed on any contradiction

```python
if contradictions:
    db.rollback()
    # Session reverts to PROCESSING (RECONCILING was never committed).
    # All child mutations are rolled back.
    raise ContradictionError(contradictions=[...])
# HTTP response: 409 with body:
# {
#   "error": "RECONCILIATION_CONTRADICTION",
#   "contradictions": [
#     {"valid_row_index": 5,
#      "type": "SUCCEEDED_NO_AUDIT",
#      "details": "Child is SUCCEEDED with student_id=<uuid> but no audit exists"}
#   ]
# }
```

The session remains recoverable: it is PROCESSING with a stale heartbeat. After admin investigates and corrects the data issue (using migration-role DB access), reconciliation can be retried.

### §10.5. Classification (apply §12 matrix to every index in valid_index_set)

For each `index` in `valid_index_set`:

- `child = child_by_index.get(index)` — may be None
- `audit_entity_id = audit_by_index.get(index)` — may be None
- Apply §12 decision matrix.
- Collect contradictions (do not short-circuit; collect all first).

After all indexes processed: if any contradiction → §10.4.

### §10.6. Write phase (all contradictions absent)

**Step 6 — Apply row-level changes:**

For each index needing an action from §12:

- INSERT (child = None): `INSERT INTO bulk_upload_row_results (...)`
- UPDATE (child exists): `UPDATE bulk_upload_row_results SET status=:new_status, ... WHERE upload_session_id=:id AND valid_row_index=:index`

No separate locking needed for child rows — the parent exclusive lock (held since the claim UPDATE) prevents any T1/T3 from accessing children.

**Step 7 — Recalculate all counters from child table:**

```sql
SELECT status, count(*)
FROM bulk_upload_row_results
WHERE upload_session_id = :upload_id
GROUP BY status;
```

After the write phase: verify `succeeded + failed + not_processed = valid_count` and `uncertain = 0` and `started = 0`. If not → contradiction in write logic → ROLLBACK.

**Step 8 — Terminal write (defense-in-depth token):**

```sql
UPDATE bulk_upload_sessions
SET status              = :terminal,
    imported_count      = :succeeded,
    failed_count        = :failed,
    not_processed_count = :not_processed,
    uncertain_count     = 0,
    validated_rows      = '[]'::jsonb,
    completed_at        = now(),
    was_reconciled      = TRUE,
    updated_at          = now()
WHERE id                        = :upload_id
  AND status                    = 'RECONCILING'
  AND reconciliation_claim_token= :my_token;
-- rowcount = 0: should not happen (we hold the lock and set RECONCILING)
--              → ROLLBACK as safety
```

**Step 9 — Audit:**\
`audit_service.record(BULK_IMPORT_RECONCILED, ...)` → flush (joins this transaction).

**Step 10 — COMMIT.**

---

## §11. Complete Lock-Order Table

| Transaction | Parent operation | Child operation | Parent update |
| --- | --- | --- | --- |
| T1 | `SELECT FOR UPDATE` (X) on session WHERE PROCESSING | `INSERT child ON CONFLICT DO NOTHING` | `UPDATE session SET heartbeat` |
| T2 | `SELECT FOR SHARE` (S) on session WHERE PROCESSING | (no child bulk_upload access) | (none) |
| T3 | `SELECT FOR UPDATE` (X) on session WHERE PROCESSING | `UPDATE child WHERE status='STARTED'` | `UPDATE session SET counter += 1` |
| After-loop | `UPDATE session SET heartbeat` (implicit X) WHERE PROCESSING | `SELECT` + `INSERT NOT_PROCESSED` + counter recalculate | Included in claim UPDATE |
| Reconciliation | `UPDATE session SET status='RECONCILING'` (implicit X) WHERE PROCESSING | `SELECT` + `INSERT/UPDATE` rows | Terminal UPDATE WHERE RECONCILING |

**Lock compatibility matrix (parent row):**

| Holder \\ Requester | T1 (X) | T2 (S) | T3 (X) | After-loop (X) | Recon (X) |
| --- | --- | --- | --- | --- | --- |
| **T1 (X)** | waits | T2 waits | waits | waits | waits |
| **T2 (S)** | T1 waits | allowed | T3 waits | waits | waits |
| **T3 (X)** | waits | T2 waits | waits | waits | waits |
| **After-loop (X)** | waits | T2 waits | waits | N/A | waits |
| **Recon (X)** | waits | T2 waits | waits | waits | 409 |

**Deadlock proof:**\
A deadlock requires a cycle: Tx A holds lock L1 and waits for L2; Tx B holds L2 and waits for L1. With parent-first ordering, every transaction acquires the parent lock before any child lock. Therefore no transaction can be "holding a child lock while waiting for the parent lock." The dependency graph is acyclic. No PostgreSQL deadlock detection is needed as a mitigation strategy.

---

## §12. Complete Child-State Consistency Matrix

This matrix is used by reconciliation to classify rows or flag contradictions.

### §12.1. Rows with no child record (child = None)

| Audit present | student_id | Action | Contradiction? |
| --- | --- | --- | --- |
| No | — | INSERT NOT_PROCESSED (was_attempted=FALSE) | No |
| Yes | — | INSERT SUCCEEDED (student_id=audit.entity_id, was_attempted=TRUE) | No |

### §12.2. Existing child rows

| Status | student_id | Audit | entity_id match | Action | Contradiction? |
| --- | --- | --- | --- | --- | --- |
| STARTED | NULL | Absent | n/a | Update → NOT_PROCESSED (was_attempted=NULL) | No |
| STARTED | NULL | Present | n/a | Update → SUCCEEDED (student_id=audit.entity_id, was_attempted=TRUE) | No |
| STARTED | Present | * | * | STARTED must not have student_id | **YES** |
| SUCCEEDED | NULL | * | * | SUCCEEDED must have student_id | **YES** |
| SUCCEEDED | Present | Absent | n/a | Committed student must have audit | **YES** |
| SUCCEEDED | Present | Present | Matches | Keep SUCCEEDED | No |
| SUCCEEDED | Present | Present | Mismatch | entity_id ≠ student_id | **YES** |
| SUCCEEDED | Present | Multiple | * | Duplicate audits for one row | **YES** |
| FAILED | NULL | Absent | n/a | Keep FAILED | No |
| FAILED | Present | * | * | FAILED must not have student_id | **YES** |
| FAILED | NULL | Present | n/a | No student created but audit exists | **YES** |
| NOT_PROCESSED | NULL | Absent | n/a | Keep NOT_PROCESSED | No |
| NOT_PROCESSED | Present | * | * | NOT_PROCESSED must not have student_id | **YES** |
| NOT_PROCESSED | NULL | Present | n/a | No student but audit exists | **YES** |
| OUTCOME_UNCERTAIN | NULL | Absent | n/a | Update → NOT_PROCESSED (was_attempted=TRUE, kept) | No |
| OUTCOME_UNCERTAIN | NULL | Present | n/a | Update → SUCCEEDED (student_id=audit.entity_id) | No |
| OUTCOME_UNCERTAIN | Present | * | * | T3 for OUTCOME_UNCERTAIN never sets student_id | **YES** |

**Valid field combinations per status:**

- **SUCCEEDED**: `student_id` NOT NULL, exactly one audit record, `audit.entity_id = student_id`
- **FAILED**: `student_id` NULL, no audit record, `error_code` non-null (application-enforced)
- **NOT_PROCESSED**: `student_id` NULL, no audit record
- **STARTED**: `student_id` NULL, `was_attempted` NULL
- **OUTCOME_UNCERTAIN**: `student_id` NULL, `was_attempted` TRUE

---

## §13. Counter Definitions and Invariants

### §13.1. Parent counter definitions

| Column | Tracks | Child status |
| --- | --- | --- |
| `imported_count` | Successfully registered students | SUCCEEDED |
| `failed_count` | Rows with known business errors | FAILED |
| `not_processed_count` | Rows with no committed registration | NOT_PROCESSED |
| `uncertain_count` | Rows where T2 was invoked but outcome unknown | OUTCOME_UNCERTAIN **only** |

**STARTED rows are NOT counted in any parent counter.** STARTED is an in-flight transient state. The gap `valid_count - (imported + failed + not_processed + uncertain)` equals the number of STARTED rows plus rows with no child record.

**uncertain_count does NOT include STARTED rows.** STARTED and OUTCOME_UNCERTAIN are distinct:

- STARTED: T1 ran; T3 has not classified the row yet (T2 may or may not have been attempted)
- OUTCOME_UNCERTAIN: T3 ran; T2 raised an unexpected exception; invocation is known

### §13.2. Non-terminal invariant

```
imported_count + failed_count + not_processed_count + uncertain_count ≤ valid_count
```

The gap = STARTED rows + rows with no child record.

### §13.3. Terminal invariant (enforced by application and CHECK constraint)

```
imported_count + failed_count + not_processed_count = valid_count
uncertain_count = 0
-- No STARTED or OUTCOME_UNCERTAIN rows in child table
```

### §13.4. T3 counter updates (incremental, within parent X lock)

T3 increments exactly one counter in the same transaction as the child UPDATE:

- SUCCEEDED → `imported_count += 1`
- FAILED → `failed_count += 1`
- OUTCOME_UNCERTAIN → `uncertain_count += 1`

NOT_PROCESSED is never written by T3. The counter `not_processed_count` is updated only by after-loop and reconciliation (both of which do full recalculation).

### §13.5. After-loop and reconciliation: full recalculation

Both after-loop (§9.3 step 3) and reconciliation (§10.6 step 7) derive all counters by querying the child table:

```sql
SELECT status, count(*)
FROM bulk_upload_row_results
WHERE upload_session_id = :id
GROUP BY status;
```

This eliminates any incremental drift from T3 counter failures or partial updates. The child table is always the authoritative source.

### §13.6. Post-transaction consistency check

After every successful T3 commit, the following invariant should hold:

```
child table count WHERE status='SUCCEEDED'      = parent.imported_count
child table count WHERE status='FAILED'         = parent.failed_count
child table count WHERE status='NOT_PROCESSED'  = parent.not_processed_count
child table count WHERE status='OUTCOME_UNCERTAIN' = parent.uncertain_count
```

Reconciliation verifies this invariant at step 7 before terminal write. Failure → ROLLBACK.

### §13.7. `was_attempted` semantics (three-state)

| Value | Meaning | Who sets it |
| --- | --- | --- |
| `TRUE` | Registration was definitely invoked | T3 (all outcomes); reconciliation (STARTED/UNCERTAIN+audit) |
| `FALSE` | Registration was definitely never invoked | After-loop NOT_PROCESSED for never-started rows; reconciliation: no child + no audit |
| `NULL` | Whether invocation occurred cannot be determined | T1 (initial); reconciliation: STARTED + no audit → NOT_PROCESSED |

---

## §14. Audit Evidence and Reconciliation Rules

### §14.1. Audit action constants

| Constant | Value | Trigger | Transaction |
| --- | --- | --- | --- |
| `BULK_IMPORT_UPLOADED` | `"bulk_import.uploaded"` | Session INSERT | Same as upload INSERT |
| `BULK_IMPORT_CONFIRMED` | `"bulk_import.confirmed"` | Confirm UPDATE | Same as confirm UPDATE |
| `STUDENT_REGISTERED` | `"student.registered"` (Phase 4) | Per-row registration | T2's internal transaction |
| `BULK_IMPORT_COMPLETED` | `"bulk_import.completed"` | Terminal write | Same as after-loop terminal UPDATE |
| `BULK_IMPORT_INTERRUPTED` | `"bulk_import.interrupted"` | Unresolved rows remain | Same as after-loop counter UPDATE |
| `BULK_IMPORT_RECONCILED` | `"bulk_import.reconciled"` | Reconciliation terminal | Same as reconciliation terminal UPDATE |

New entity type: `BULK_UPLOAD_SESSION = "bulk_upload_session"`.

`AuditService.record()` only flushes (never commits). Each audit event is committed by its caller's transaction.

### §14.2. Per-registration audit extension

Inside `register_student()` when `bulk_origin` is provided:

```python
metadata["bulk_upload_session_id"]     = str(bulk_origin.upload_session_id)
metadata["bulk_upload_valid_row_index"] = bulk_origin.row_index
```

Key renamed from v7's `bulk_upload_row_number` to `bulk_upload_valid_row_index`.

The audit record is committed atomically with the student at service.py:309. There is no way to have:

- A committed student without a matching audit record
- An audit record for a valid row without a committed student

### §14.3. Reconciliation audit query

```sql
SELECT
    entity_id,
    event_metadata->>'bulk_upload_valid_row_index' AS raw_index,
    event_metadata->>'bulk_upload_session_id'       AS raw_session_id
FROM audit_logs
WHERE action = 'student.registered'
  AND event_metadata->>'bulk_upload_session_id' = :upload_id_str;
```

Uses existing `ix_audit_logs_action` index on `action`. Post-filter on JSONB is applied after the index scan.

### §14.4. Proposed: partial audit index (pending approval — D7)

```sql
CREATE INDEX ix_audit_logs_bulk_session_id
    ON audit_logs ((event_metadata->>'bulk_upload_session_id'))
   WHERE action = 'student.registered'
     AND event_metadata ? 'bulk_upload_session_id';
```

Improves reconciliation query performance. Currently not required for correctness. PENDING USER APPROVAL (D7).

---

## §15. Failure and Recovery Behavior

### §15.1. Crash-window analysis

| # | Crash point | Child state | Audit? | Recovery |
| --- | --- | --- | --- | --- |
| CW1 | Before T1 commits | No child | No | Loop retries T1; inserts cleanly |
| CW2 | T1 committed; before T2 | STARTED (NULL) | No | After-loop or reconciliation: no audit → NOT_PROCESSED |
| CW3 | T2 running; before commit | STARTED (NULL) | No | Same as CW2 (T2 rolled back) |
| CW4 | T2 committed; before T3 | STARTED (NULL) | Yes | Reconciliation: audit exists → SUCCEEDED |
| CW5 | T3 committed | Terminal status | Matches | Done |
| CW6 | T3 exception after T2 success | STARTED | Yes | Reconciliation: audit exists → SUCCEEDED |
| CW7 | After-loop timeout | Per-row | Per-row | Reconciliation (heartbeat stale after D1 threshold) |
| CW8 | After-loop committed terminal | All final | completed audit | Done |
| CW9 | After-loop committed interrupted | Remaining STARTED/UNCERTAIN | interrupted audit | Reconciliation |
| CW10 | Reconciliation ROLLBACK (contradiction) | Pre-reconciliation state (unchanged) | Unchanged | Admin investigates; retry reconciliation |
| CW11 | Reconciliation committed | All final | reconciled audit | Done |

### §15.2. Session recovery guarantee

Every PROCESSING session with a stale heartbeat (> D1 threshold) is eligible for reconciliation. After-loop rollbacks leave the session in PROCESSING. Reconciliation can always claim and resolve such a session.

### §15.3. Unrecoverable states

A PROCESSING session with contradictions in its audit or child table cannot be automatically resolved. The session remains PROCESSING, accessible for admin review. The admin resolves the data issue with migration-role DB access and retries reconciliation.

---

## §16. API Contracts

### §16.1. Upload — `POST /admin/bulk-uploads`

Permission: `student.bulk_import` (MAIN_ADMIN only).

Request: `multipart/form-data` — `file` (≤5 MB), `department_id`.

Response 201:

```json
{
  "upload_id":               "uuid",
  "status":                  "PENDING_CONFIRMATION",
  "valid_count":             18,
  "invalid_count":           2,
  "total_rows":              20,
  "expires_at":              "2026-11-24T...",
  "valid_rows":              [...],
  "preview_invalid_results": [...]
}
```

Errors: 400 (zero valid rows, column count mismatch, blank file, >100 rows, unknown headers, duplicate headers, missing required headers), 413 (>5 MB), 422 (UTF-8 failure, CSV parse error).

### §16.2. Confirm — `POST /admin/bulk-uploads/{id}/confirm`

```sql
UPDATE bulk_upload_sessions
SET status                  = 'PROCESSING',
    confirmed_by_user_id    = :actor_id,
    confirmed_at            = now(),
    processing_started_at   = now(),
    processing_heartbeat_at = now(),
    updated_at              = now()
WHERE id     = :upload_id
  AND status = 'PENDING_CONFIRMATION'
  AND expires_at > now()
RETURNING id;
```

0 rows: check status (expired → lazy expire + 409; not found → 404; other → 409).

Triggers synchronous import loop. Response 200 with terminal results if all rows settled, or PROCESSING status with `needs_reconciliation: true` if unresolved rows remain.

### §16.3. Get Status — `GET /admin/bulk-uploads/{id}`

Never returns RECONCILING (concurrent reads see last committed state under READ COMMITTED — PROCESSING).

Applies lazy expiry: if `status='PENDING_CONFIRMATION' AND expires_at ≤ now()`, update to EXPIRED.

For terminal states: includes child-table results (omits email/phone/address), `preview_invalid_results`.\
For PROCESSING: live counters, `is_stale`, `needs_reconciliation`.

### §16.4. Reconcile — `POST /admin/bulk-uploads/{id}/reconcile`

Requirements: `status='PROCESSING'`, `processing_heartbeat_at < now() - D1`.

Responses:

- 200: Terminal results
- 409 `RECONCILIATION_CONTRADICTION`: session reverts to PROCESSING; `contradictions` array in body
- 409 (not eligible): session not stale PROCESSING
- 503: lock timeout (another operation holds the session lock)

### §16.5. Report — `GET /admin/bulk-uploads/{id}/report`

Terminal sessions only (409 otherwise). `text/csv; charset=utf-8`.

Columns: `csv_record_number`, `valid_row_index`, `first_name`, `last_name`, `roll_number`, `status`, `was_attempted`, `error_code`, `error_message`, `corrective_action`, `is_retryable`.

Includes child-table rows + preview-invalid rows (SKIPPED_PREVIEW_INVALID). Formula injection prevention: prefix cells starting with `{= + - @ TAB CR LF}`.

### §16.6. List — `GET /admin/bulk-uploads`

Paginated. Filters: `department_id`, `status`. Summary counters only.

---

## §17. CSV Preview and Confirmation

### §17.1. File acceptance

Read up to 5,000,001 bytes. If more data remains → 413.

1. UTF-8 decode. Strip BOM. Failure → 422.
2. Save and restore `csv.field_size_limit(8192)`:

   ```python
   old_limit = csv.field_size_limit()
   try:
       csv.field_size_limit(8192)
       # ... parse ...
   finally:
       csv.field_size_limit(old_limit)
   ```

   This is a process-wide setting; save/restore prevents affecting other CSV consumers.
3. `csv.reader(strict=True)`. Parse failure → 422.
4. Header validation (§17.2).
5. Column-count consistency: every non-blank data record must have exactly the header's field count. Mismatch → 400 citing record number.
6. Count non-blank data records. Zero → 400. >100 → 400.

### §17.2. Headers

Required (case-insensitive, trimmed): `first_name`, `last_name`, `email`, `roll_number`, `program_code`, `semester_level`, `admission_term`.\
Optional: `section_name`, `phone`, `address`.\
Missing required → 400. Duplicate → 400. Unknown → 400.

### §17.3. Record numbering

`csv_record_number`: 1-indexed ordinal of non-blank data records after the header. Multi-line quoted records count as one. Blank records (all fields empty after strip) are skipped.

`valid_row_index`: 1..`valid_count`, assigned in `csv_record_number` order to records passing all preview validations.

### §17.4. Formula-cell validation

All columns except `phone` are checked. Phone is exempt — `normalize_phone()` already rejects non-phone strings, and E.164 numbers with `+` prefix would trigger false positives.

Rule: reject if first character is in `{= + - @ \t \r \n}` or first non-whitespace character is in `{= + - @}`.

### §17.5. Semester-level validation

Parse as integer ≥ 1. After resolving `program_code`, verify `≤ program.duration_semesters`. No arbitrary cap in CSV code — the Phase 4 rule at service.py:734 is authoritative. DB backstop is `ck_student_placements_level BETWEEN 1 AND 16`.

### §17.6. Lazy expiry

```sql
UPDATE bulk_upload_sessions
SET status      = 'EXPIRED',
    validated_rows = '[]'::jsonb,
    updated_at  = now()
WHERE id     = :upload_id
  AND status = 'PENDING_CONFIRMATION'
  AND expires_at <= now();
```

Applied on any read of a PENDING_CONFIRMATION session.

---

## §18. Expiry and Retention

### §18.1. Expiry timing

`expires_at = created_at + INTERVAL '30 minutes'`. Immutable after upload (not in UPDATE privilege grant). Lazy expiry applied on reads.

### §18.2. Data classification

| Location | PII | Scrubbing |
| --- | --- | --- |
| `validated_rows` JSONB (parent) | Yes: all parsed fields | Scrubbed to `[]` atomically with terminal write |
| `preview_invalid_results` JSONB (parent) | None: record numbers + error codes | Never scrubbed |
| Child `first_name`, `last_name`, `roll_number` | Yes | Retained until purge |
| Child other columns | None | Retained until purge |
| Audit logs | Roll number, program, term (no email/phone/address) | Append-only; never scrubbed |

### §18.3. Current implementation: indefinite retention

No purge mechanism exists. All records retained indefinitely. This is explicitly accepted as temporary.

### §18.4. Proposed retention (pending approval — D8)

| State | Proposed | Measured from |
| --- | --- | --- |
| Terminal | 90 days | `completed_at` |
| EXPIRED | 30 days | `expires_at` |

### §18.5. Future purge specification

When implemented:

1. Delete child rows first (`ON DELETE RESTRICT` prevents parent deletion with children present).
2. Delete parent session.
3. Audit records are **not deleted** (append-only; application role has no DELETE on `audit_logs`).
4. Purge runs with the migration role (not application role — no DELETE granted).
5. PROCESSING sessions with heartbeat far older than 90 days are included in purge scope (abandoned sessions).

---

## §19. Security and RBAC

| Concern | Rule |
| --- | --- |
| Permission | `student.bulk_import` required (confirmed at permissions.py:49) |
| Role | MAIN_ADMIN only (holds `set(PermissionCode)`) |
| Delegability | NOT in `DELEGABLE_STAFF_PERMISSIONS`; HOD cannot perform bulk import |
| Session retrieval | Any Main Admin (not only the uploader) may GET/reconcile/report |
| File content | Validated at application layer before any DB write |
| Formula injection | Checked in preview (§17.4) and CSV report output |
| Audit | Every lifecycle event audited with actor, IP, metadata |
| PII retention | `validated_rows` scrubbed at terminal; audit metadata never includes email/phone/address |
| DB privileges | `NO_DELETE_TABLES`, `COLUMN_UPDATE_TABLES` enforced (§3.4) |

No changes to Phase 4 RBAC, permission codes, authentication/session policies, or `register_student()` authorization.

---

## §20. Email Behavior

Email is at-most-once via FastAPI `BackgroundTasks`, registered AFTER `register_student()` returns (after T2 commit), in a separate try block from T3.

```python
# After T3 commits SUCCEEDED:
try:
    schedule_email(background_tasks, email_sender,
        build_student_onboarding_email(...))
except Exception:
    logger.warning("Email scheduling failed for row %d", index, exc_info=True)
# Always continue to next row.
```

`schedule_email` → `background_tasks.add_task(send_email_safely, ...)`. This registers a callable only; it does not send anything synchronously. `send_email_safely` catches all exceptions internally — delivery failures never propagate.

**Email failure after committed SUCCEEDED:** The row remains SUCCEEDED. The student account exists. The import loop continues to the next row. No email failure can change a committed outcome.

No durable background queue. Email can be resent via `student.onboarding_email.resent` endpoint.

---

## §21. Logging and Privacy

| Event | Level | PII logged? |
| --- | --- | --- |
| Upload received | INFO | No |
| Preview complete | INFO | No |
| Confirm started | INFO | No |
| T1 FENCED | WARNING | No |
| T2 business error | DEBUG | No |
| T2 unexpected error | ERROR (exc_info=True) | No |
| T3 exception | ERROR (exc_info=True) | No |
| Email scheduling failure | WARNING (exc_info=True) | No |
| After-loop ownership rowcount=0 | INFO | No |
| After-loop committed terminal | INFO | No |
| Reconciliation claim | INFO | No |
| Reconciliation contradiction | WARNING | Contradiction type + row index (no names/emails) |
| Reconciliation committed | INFO | No |

Audit metadata exclusions: no email, phone, address, password. Roll number and program appear only in `student.registered` metadata (consistent with Phase 4 policy).

---

## §22. Testing Strategy

### §22.1. Lock-order and concurrency tests (correction 1)

Tests use real PostgreSQL transactions with controlled barriers (`threading.Barrier`). No `time.sleep()`.

**CT-LO-1: T3 vs reconciliation (the original v8 deadlock scenario)**

- Setup: session PROCESSING; child row STARTED
- Thread A (T3): acquires parent FOR UPDATE; holds at barrier B1 (before child UPDATE)
- Thread B (Reconciliation): tries to acquire parent FOR UPDATE → blocks
- Release B1: T3 completes child UPDATE + counter UPDATE + COMMIT
- Verify: Thread B unblocks; sees parent PROCESSING with fresh heartbeat; claim check heartbeat stale → 409 (not eligible)
- Expected: No deadlock; sequential execution; exactly one child update

**CT-LO-2: T1 vs reconciliation**

- Thread A (T1): acquires parent FOR UPDATE at barrier B1 (before child INSERT)
- Thread B (Reconciliation): tries to claim parent → blocks
- Release B1: T1 inserts child + updates heartbeat + COMMIT
- Verify: Thread B unblocks; sees fresh heartbeat → 409 (not stale enough to claim)

**CT-LO-3: T3 vs after-loop (after-loop wins)**

- Thread A (After-loop): acquires parent X lock; holds at B1
- Thread B (T3): tries parent FOR UPDATE → blocks
- Release B1: After-loop commits terminal
- Verify: Thread B unblocks; sees `status ≠ 'PROCESSING'` → T3Result.FENCED; child not modified

**CT-LO-4: T1 vs after-loop**

- Analogous to CT-LO-3. T1 blocks; after-loop commits terminal; T1 sees terminal → FENCED.

**CT-LO-5: T2 registration vs reconciliation**

- Thread A (T2): acquires parent FOR SHARE; commits (student + audit)
- Thread B (Reconciliation): tries parent X lock while A holds S → blocks
- Thread A COMMIT
- Thread B unblocks; audits are now visible; proceeds (if heartbeat stale)
- Verify: Reconciliation sees committed audit record; classifies row as SUCCEEDED

**CT-LO-6: Two simultaneous T1s for same row**

- Thread A (T1-1): acquires parent X lock at B1
- Thread B (T1-2): tries parent X lock → blocks
- Release B1: T1-1 inserts child + COMMIT
- Thread B unblocks; INSERT ON CONFLICT DO NOTHING → rowcount=0 (conflict) → ROLLBACK → SKIP
- Verify: Exactly one child row; row processed exactly once

**CT-LO-7: Retry after T1 rollback**

- T1 attempt 1: acquires parent lock, inserts child, then connection lost → ROLLBACK
- Verify: child row does not exist
- T1 attempt 2: clean insert → COMMIT
- Verify: child row exists, was_attempted=NULL

**CT-LO-8: T3 with terminal parent**

- After-loop commits terminal while T3 is about to start
- T3 step 1: `SELECT FOR UPDATE WHERE status='PROCESSING'` → rowcount=0
- Verify: T3Result.FENCED; child not modified

### §22.2. FK and schema tests (correction 2)

**CT-FK-1:** Insert child row with `student_id = <valid users.id that is NOT a student>` → FK violation (referencing students(id) fails).

**CT-FK-2:** Insert child row with `student_id = <valid students.id>` → succeeds.

**CT-FK-3:** Attempt to delete a student row referenced by a child result row → ON DELETE RESTRICT error.

### §22.3. Audit index validation tests (correction 3)

One test per contradiction type in §10.3 step 4:

| Test | Setup | Expected |
| --- | --- | --- |
| CT-AI-1 | Audit with `raw_index = "abc"` | Contradiction: INDEX_NOT_INTEGER |
| CT-AI-2 | Audit with `raw_index = true` (JSON boolean) | Contradiction: INDEX_NOT_INTEGER |
| CT-AI-3 | Audit with `raw_index = 999` for 20-row session | Contradiction: INDEX_OUT_OF_RANGE |
| CT-AI-4 | Audit with `raw_index = 3` but index 3 not in `validated_rows` | Contradiction: INDEX_NOT_IN_VALIDATED_ROWS |
| CT-AI-5 | Two audit records with same `raw_index = 2` | Contradiction: DUPLICATE_AUDIT_INDEX |
| CT-AI-6 | Audit record with `entity_id = NULL` | Contradiction: ENTITY_ID_MISSING |
| CT-AI-7 | Valid audit records for all indexes | No contradiction; proceeds |
| CT-AI-8 | `validated_rows = []` while PROCESSING | Reconciliation fails closed with specific error |

### §22.4. Child-state consistency matrix tests (correction 4)

| Test | Child state | student_id | Audit | Expected |
| --- | --- | --- | --- | --- |
| CT-CS-1 | SUCCEEDED | NULL | any | Contradiction |
| CT-CS-2 | SUCCEEDED | Present | Absent | Contradiction |
| CT-CS-3 | SUCCEEDED | Present | entity mismatch | Contradiction |
| CT-CS-4 | FAILED | Present | any | Contradiction |
| CT-CS-5 | FAILED | NULL | Present | Contradiction |
| CT-CS-6 | NOT_PROCESSED | Present | any | Contradiction |
| CT-CS-7 | NOT_PROCESSED | NULL | Present | Contradiction |
| CT-CS-8 | STARTED | NULL | Absent | → NOT_PROCESSED (was_attempted=NULL) |
| CT-CS-9 | STARTED | NULL | Present | → SUCCEEDED (was_attempted=TRUE) |
| CT-CS-10 | OUTCOME_UNCERTAIN | NULL | Absent | → NOT_PROCESSED (was_attempted=TRUE kept) |
| CT-CS-11 | OUTCOME_UNCERTAIN | NULL | Present | → SUCCEEDED |
| CT-CS-12 | OUTCOME_UNCERTAIN | Present | any | Contradiction |
| CT-CS-13 | No child | Absent | — | INSERT NOT_PROCESSED (was_attempted=FALSE) |
| CT-CS-14 | No child | Present audit | — | INSERT SUCCEEDED |

### §22.5. Counter invariant tests (correction 5)

| Test | Scenario | Expected |
| --- | --- | --- |
| CT-CNT-1 | N T3 SUCCEEDED commits | `parent.imported_count = N` |
| CT-CNT-2 | 3 STARTED rows remain | None counted in any parent counter |
| CT-CNT-3 | OUTCOME_UNCERTAIN rows | `uncertain_count = count`; STARTED not included |
| CT-CNT-4 | After-loop recalculates | Full GROUP BY query; not incremental |
| CT-CNT-5 | Terminal write | `uncertain_count = 0`; `sum = valid_count`; CHECK passes |

### §22.6. STARTED-row safety tests (from v7, preserved)

| Test | Scenario | Expected |
| --- | --- | --- |
| S1 | T1→T2 success→crash before T3 | STARTED; audit exists; reconciliation → SUCCEEDED |
| S2 | T1→T2 rollback→crash before T3 | STARTED; no audit; reconciliation → NOT_PROCESSED (was_attempted=NULL) |
| S3 | 3 rows: row 1 SUCCEEDED, row 2 STARTED, row 3 never started | After-loop inserts NOT_PROCESSED for row 3; leaves row 2 STARTED; session stays PROCESSING; reconciliation resolves row 2 |

### §22.7. Reconciliation state tests

| Test | Setup | Expected |
| --- | --- | --- |
| CT-RC-1 | Stale PROCESSING session | Claim succeeds |
| CT-RC-2 | Fresh heartbeat (\< D1 threshold) | Claim fails: 409 |
| CT-RC-3 | Terminal session | Claim fails: 409 |
| CT-RC-4 | Contradiction detected | ROLLBACK; session stays PROCESSING; 409 with contradictions |
| CT-RC-5 | RECONCILING visible from concurrent reader | Never; reader sees PROCESSING |
| CT-RC-6 | After contradiction rollback, retry with fixed data | Succeeds; terminal write |

---

## §23. Implementation Milestones

### M1 — Schema, Upload, Preview (3–4 days)

1. `bulk_models.py`: `BulkUploadSession`, `BulkUploadRowResult` with `students(id)` FK
2. `bulk_constants.py`: `BulkImportOrigin`, `BulkImportSessionFencedError`, `ImporterState`, `RowOutcome`, `T1Result`, `T3Result`
3. Alembic migration: both tables, all constraints, all indexes. Optionally partial audit index (D7).
4. `provision_db_roles.py`: add entries per §3.4
5. `bulk_service.py`: `upload_csv()` — strict parsing with field_size_limit save/restore, formula check, semester-level check, `validated_rows` JSONB with `valid_row_index` per element (D9 must be confirmed)
6. `bulk_schemas.py`, `bulk_router.py`: `POST /admin/bulk-uploads`
7. Audit constants: `BULK_IMPORT_UPLOADED`, `BULK_UPLOAD_SESSION` entity type
8. Tests: CT-FK-1, CT-FK-2, upload headers, formula-phone exemption, semester-level, >100 rows, >5 MB, zero valid, BOM, permissions

### M2 — Confirm, Import Loop with Parent-First Locking (3–4 days)

1. `bulk_origin` parameter wiring on `register_student()`: FOR SHARE fencing
2. `persist_started()` (T1): parent FOR UPDATE → INSERT ON CONFLICT DO NOTHING → update heartbeat
3. `persist_outcome()` (T3): parent FOR UPDATE → UPDATE child WHERE STARTED → increment counter
4. `confirm_and_process()`: confirmation claim, import loop with separated try blocks (§8.2), after-loop (§9.3)
5. `POST /admin/bulk-uploads/{id}/confirm`
6. Lazy expiry
7. Tests: CT-LO-1 through CT-LO-8, exception boundary table (§8.3), CT-CNT-1 through CT-CNT-5, S1 through S3

### M3 — Reconciliation, Results, Report (2–3 days)

1. `reconcile()`: single-transaction model (§10), audit index validation against `validated_rows` (§10.3 step 4), child-state consistency matrix (§12), fail-closed contradiction handling
2. GET status, GET report, POST reconcile, GET list
3. Response shaping: RECONCILING never returned; `was_attempted` three-state display; `needs_reconciliation` flag
4. Tests: CT-AI-1 through CT-AI-8, CT-CS-1 through CT-CS-14, CT-RC-1 through CT-RC-6

### M4 — Integration, Edge Cases, Concurrency (2 days)

1. End-to-end: upload → confirm → results → report
2. All CT-LO concurrency tests under real database
3. Failure injection: DB disconnect at CW2/CW4/CW6; reconciliation after each crash window
4. Count invariant verification under concurrent T3s
5. Contradiction scenarios (CT-AI + CT-CS) with real audit log entries

---

## §24. v8 Improvements — Explicit Revalidation

| # | v8 point | v9 revalidation status |
| --- | --- | --- |
| 1 | RECONCILING transient, never externally committed | §4.1, §10.1: SET RECONCILING within single transaction; always commits to terminal or ROLLBACK; API never returns RECONCILING ✓ |
| 2 | Reconciliation holds parent lock for complete transaction | §10.1–§10.6: single transaction; claim UPDATE holds exclusive lock until COMMIT/ROLLBACK ✓ |
| 3 | After-loop and reconciliation use parent-first locking | §9.2: after-loop acquires via UPDATE heartbeat; §10.2: reconciliation via UPDATE SET RECONCILING; both parent-first ✓ |
| 4 | Parent ownership rowcounts checked | §9.3 step 0: `ownership_rows == 0 → return`; §10.5 step 8: `rowcount = 0 → ROLLBACK` ✓ |
| 5 | Separated exception boundaries | §8.2: separate try blocks for T1, T2, T3, email ✓ |
| 6 | Registration and audit atomically consistent | §6.4: `AuditService.record()` joins T2's transaction; committed at service.py:309 ✓ |
| 7 | `register_student()` called once per valid row | §8.2: T1 SKIP prevents second T2 invocation ✓ |
| 8 | `register_student()` retains internal commit/rollback | §6.1: unchanged; import loop does not control T2 boundary ✓ |
| 9 | `AuditService.record()` only flushes | §6.4, §14.1: confirmed via audit/service.py:96–125 ✓ |
| 10 | `was_attempted` semantics precise | §13.7: three-state table; NULL/FALSE/TRUE each defined precisely ✓ |
| 11 | NOT_PROCESSED meaning | §13.7: "no committed registration found"; not necessarily "never invoked" ✓ |
| 12 | Counters recalculated from child table during recovery | §13.5: after-loop and reconciliation both use full GROUP BY query ✓ |
| 13 | `validated_rows` scrubbed at correct terminal point | §10.6 step 8: `validated_rows='[]'` in terminal UPDATE atomically; immutable after upload ✓ |
| 14 | CSV validation, preview, confirmation, expiry unchanged | §17, §18 ✓ |
| 15 | Email at-most-once via BackgroundTasks after DB commit | §20: `schedule_email` → `background_tasks.add_task(send_email_safely, ...)`; deferred ✓ |
| 16 | No durable background queue | §20: confirmed; no Redis/Celery/RQ ✓ |

---

## §A. Resolved Issues (v9 Corrections)

### A1. CRITICAL: Consistent parent-first lock order

**Problem (v8):** T1 inserted the child row before updating the parent; T3 updated the child row before updating the parent. After-loop and reconciliation locked the parent first. This asymmetry created a deadlock: T3 holding child lock + reconciliation holding parent lock + each waiting for the other.

**v9 resolution:**\
T1 (§5) and T3 (§7) both begin with `SELECT ... FOR UPDATE` on the parent session `WHERE status='PROCESSING'` before any child mutation. The lock acquisition order for all five paths is now uniformly:

```
parent session (exclusive) → child rows → parent counter update
```

Deadlock is structurally impossible: no transaction can hold a child lock while waiting for the parent lock. The dependency graph is acyclic. No retry logic or deadlock detection is needed as a mitigation.

### A2. HIGH: Student FK target corrected

**Problem (v8):** `bulk_upload_row_results.student_id` referenced `users(id)`. A user is not necessarily a student.

**v9 resolution (§3.2):** References `students(id) ON DELETE RESTRICT`. Confirmed from Phase 4 models.py:42–46: `students.id` is a shared PK/FK with `users.id`. Referencing `students(id)` guarantees the UUID is an actual student account. No existing Phase 4 relationship is affected.

### A3. HIGH: Audit index validation against validated_rows

**Problem (v8):** Reconciliation only checked that the audit row index was a positive integer. An index of 999 in a 20-row session would pass.

**v9 resolution (§10.3 step 4):** Nine validation checks per audit record, including `INDEX_NOT_IN_VALIDATED_ROWS`: the index must be present in `valid_index_set`, derived from the `validated_rows` JSONB array stored at upload time. This is strictly stronger than checking against `valid_count`. Any failure → CONTRADICTION → ROLLBACK.

### A4. MEDIUM: Complete child-row consistency matrix

**Problem (v8):** Reconciliation decision table only checked child status vs. audit presence. Field-level validation was absent.

**v9 resolution (§12):** Full matrix covering all combinations of (status × student_id presence × audit presence × entity_id match). Every impossible combination is a CONTRADICTION. Fail-closed for all contradictory combinations.

### A5. MEDIUM: Counter consistency formalized

**Problem (v8):** `uncertain_count` conflated STARTED and OUTCOME_UNCERTAIN. T3 counter updates and after-loop recalculation could race.

**v9 resolution (§13):**

- `uncertain_count` tracks ONLY OUTCOME_UNCERTAIN rows. STARTED rows are not counted in any parent counter.
- T3 increments are within the parent exclusive lock, eliminating the race condition.
- After-loop and reconciliation both use full `GROUP BY` recalculation from child table (never incremental).
- Counter invariant formally stated and verified by reconciliation before terminal write.

### A6. Preservation of v8 improvements

All 16 v8 revalidation points explicitly confirmed in §24. No regressions.

---

## §B. Pending User Decisions

### D1 — Stale heartbeat threshold for reconciliation eligibility

**Question:** What threshold should determine that a PROCESSING session's heartbeat is "stale" and eligible for reconciliation?

**Used in:** `processing_heartbeat_at < now() - INTERVAL '[threshold]'` in the reconciliation claim SQL.

| Option | Value | Consequence |
| --- | --- | --- |
| 2 minutes | Aggressive | May claim sessions that are still processing under load |
| **5 minutes** | **Recommended** | Import of 100 rows should complete well within 5 min; heartbeat refreshed per row |
| 10 minutes | Conservative | Longer wait before crash recovery |
| 30 minutes | Very conservative | Long delay for admin |

**Recommended:** 5 minutes.

---

### D2 — Lock timeout for T1 and T3

**Question:** What `SET LOCAL lock_timeout` value should T1 and T3 use when acquiring the parent exclusive lock?

**Used in:** `SET LOCAL lock_timeout = '[value]'` at the start of T1 and T3 transactions.

| Option | Value | Consequence |
| --- | --- | --- |
| 1 second | Aggressive | T1/T3 timeout under momentary lock contention; loop interrupts unnecessarily |
| **5 seconds** | **Recommended** | Reasonable balance for a synchronous import loop |
| 10 seconds | Conservative | Per-row latency grows; overall HTTP timeout risk |

**Recommended:** 5 seconds.

---

### D3 — Lock timeout for reconciliation claim

**Question:** What `SET LOCAL lock_timeout` value should reconciliation use when claiming the session?

**Used in:** `SET LOCAL lock_timeout = '[value]'` in the reconciliation BEGIN.

| Option | Value | Consequence |
| --- | --- | --- |
| 5 seconds | Same as T1/T3 | May fail during active import loop |
| **10 seconds** | **Recommended** | Admin-initiated operation; slightly more patient |
| 30 seconds | Long | Admin waits on 503 for 30 seconds |

**Recommended:** 10 seconds.

---

### D4 — `BulkImportOrigin` as frozen dataclass

**Question:** Should `BulkImportOrigin` be a `@dataclass(frozen=True)`?

**Recommended:** Yes. Immutable; safely shared without risk of mutation across the call chain.

| Option | Consequence |
| --- | --- |
| `@dataclass(frozen=True)` (**recommended**) | Immutable; hashable |
| Plain `@dataclass` | Mutable; risk of accidental mutation |

---

### D5 — `BulkImportSessionFencedError` inheritance

**Question:** Should `BulkImportSessionFencedError` extend `StudentServiceError` directly (not `StudentConflictError`)?

**Recommended:** Yes. Fencing is a session lifecycle condition, not a data conflict.

| Option | Consequence |
| --- | --- |
| Extend `StudentServiceError` directly (**recommended**) | Flat hierarchy; clearly distinct from data conflicts |
| Extend `StudentConflictError` | Misleading; suggests a data uniqueness issue |

---

### D6 — `reconciliation_claim_token` defense-in-depth

**Question:** Should the terminal UPDATE include `AND reconciliation_claim_token = :my_token` as a defense-in-depth check?

**Recommended:** Yes. The token was set by this exact transaction (which holds the exclusive lock), so a 0-rowcount on the terminal UPDATE is an internal bug signal. Belt-and-suspenders.

| Option | Consequence |
| --- | --- |
| Include token check (**recommended**) | Defense-in-depth; zero extra cost |
| Omit token check | Relies solely on `status='RECONCILING'` guard |

---

### D7 — Partial audit index for reconciliation

**Question:** Should the Alembic migration include the following partial index?

```sql
CREATE INDEX ix_audit_logs_bulk_session_id
    ON audit_logs ((event_metadata->>'bulk_upload_session_id'))
   WHERE action = 'student.registered'
     AND event_metadata ? 'bulk_upload_session_id';
```

| Option | Consequence |
| --- | --- |
| Include in M1 migration (**recommended**) | Improves reconciliation query as audit table grows; small additional index maintenance |
| Defer to later migration | Correct now; may need a later migration as scale grows |
| Omit entirely | Acceptable for small scale; full table scan on `action` for reconciliation |

**Recommended:** Include in M1 migration.

---

### D8 — Proposed purge retention periods

**Question:** When a future purge mechanism is implemented, should it use 90-day terminal / 30-day expired retention?

| Option | Terminal | EXPIRED | Consequence |
| --- | --- | --- | --- |
| **Recommended** | 90 days | 30 days | Sufficient for investigation; PII cleared promptly |
| Extended | 180 days | 90 days | Longer PII retention; more time for audits |
| Short | 30 days | 7 days | Faster PII removal; less time for investigation |

**Note:** No purge mechanism is implemented in Phase 5. This decision affects only future implementation.

---

### D9 — `validated_rows` JSONB element structure

**Question:** Must every element in `validated_rows` include `valid_row_index` as a top-level integer field?

**Used in:** Reconciliation's `valid_index_set` extraction (§10.3 step 1).

| Option | Consequence |
| --- | --- |
| **Yes, required** (**recommended**) | Reconciliation can validate audit indexes against the actual validated rows |
| Omit `valid_row_index` from elements | Reconciliation falls back to `valid_count` range check only (weaker) |

**Recommended:** Yes. The `validated_rows` elements MUST include `valid_row_index`. This must be enforced in the upload service implementation in M1.

**This is a cross-milestone dependency:** M1 (upload) and M3 (reconciliation) must both agree on this structure. D9 must be confirmed before M1 coding begins.

---

## Unconfirmed Items

No design elements are UNCONFIRMED. All sections are based on:

- Verified Phase 4 source files (service.py, models.py, validators.py, database.py, email.py, router.py, audit files, provision_db_roles.py)
- Verified Phase 4 constraints and lock behavior
- v8 design (which was derived from the same sources)
- v9 corrections as specified

The nine items above (D1–D9) are design decisions pending user approval, not unconfirmed technical facts.

## Inconsistencies Between v8 and v9

| # | Location | v8 | v9 | Change type |
| --- | --- | --- | --- | --- |
| 1 | T1 SQL | Child INSERT before parent lock | Parent FOR UPDATE before child INSERT | **v9 correction 1** |
| 2 | T3 SQL | Child UPDATE before parent lock | Parent FOR UPDATE before child UPDATE | **v9 correction 1** |
| 3 | `student_id` FK | `REFERENCES users(id)` | `REFERENCES students(id)` | **v9 correction 2** |
| 4 | Audit index validation | Checks positive integer only | 9-check validation including `INDEX_NOT_IN_VALIDATED_ROWS` | **v9 correction 3** |
| 5 | Consistency matrix | Status vs audit only | Full status × student_id × audit × entity_id matrix | **v9 correction 4** |
| 6 | `uncertain_count` | Included STARTED rows | OUTCOME_UNCERTAIN only | **v9 correction 5** |
| 7 | Counter update | Incremental in T3; recalculate in after-loop/reconciliation | Same pattern; now within parent X lock (correction 1 prevents the race) | Clarified |

---

## §C. Implementation Readiness

**NOT READY — MORE DESIGN WORK REQUIRED**

Decisions **D2**, **D3**, and **D9** must be approved before M1 coding begins:

- **D2** (T1/T3 lock timeout): embedded as a literal in `SET LOCAL lock_timeout` SQL. Must be agreed before `persist_started()` and `persist_outcome()` are written.
- **D3** (reconciliation lock timeout): embedded in the reconciliation claim SQL. Must be agreed before `reconcile()` is written.
- **D9** (`validated_rows` structure): M1's `upload_csv()` and M3's `reconcile()` must agree on whether `validated_rows` elements include `valid_row_index`. If M1 is built without this field and M3 expects it, a migration and data fix are needed.

**D1** (stale heartbeat threshold) can be agreed at M2 (reconciliation claim is in `confirm_and_process()` and the reconcile endpoint).

**D4–D6** are low-risk structural decisions (dataclass, exception inheritance, token) that can be confirmed as part of M1 approval or later.

**D7** (partial audit index): affects the M1 migration. Should be confirmed before M1 migration is written.

**D8** (future purge) has no impact on Phase 5 implementation.

**No critical correctness issues remain** in the concurrency model, FK targets, audit validation, consistency matrix, or counter semantics. The design is internally consistent and all v9 corrections are addressed.

Once D1–D3, D7, and D9 are confirmed, v9 is ready for M1 design approval.