# Phase 5 — Bulk Student Registration: Technical Design v10

**Status:** READY FOR M1 DESIGN APPROVAL | **Date:** 2026-11-24 | **Supersedes:** v9 (2026-11-24)

Design-only. No code, migrations, or database changes are authorized until this document is explicitly approved for the milestone being implemented.

### Table of Contents

1. [Architecture Overview](#s1)
2. [Scope and Non-Goals](#s2)
3. [Database Model and Constraints](#s3)
4. [State Machine](#s4)
5. [T1 — Row Initialization](#s5)
6. [T2 — Registration Transaction](#s6)
7. [T3 — Result Persistence](#s7)
8. [Import Loop](#s8)
9. [After-Loop Finalization](#s9)
10. [Reconciliation Transaction](#s10)
11. [Complete Lock-Order Table](#s11)
12. [Child-State Consistency Matrix](#s12)
13. [Counter Definitions and Invariants](#s13)
14. [Audit Evidence and Rules](#s14)
15. [Failure and Recovery Behavior](#s15)
16. [API Contracts](#s16)
17. [CSV Preview and Confirmation](#s17)
18. [Expiry and Retention](#s18)
19. [Security and RBAC](#s19)
20. [Email Behavior](#s20)
21. [Logging and Privacy](#s21)
22. [Testing Strategy](#s22)
23. [Implementation Milestones](#s23)
24. [v8 Improvements — Revalidation](#s24)
25. [Appendix A: v9 Corrections](#appA)
26. [Appendix B: v10 Corrections](#appB)
27. [Appendix C: Decisions Applied](#appC)
28. [Appendix D: Remaining Limitations](#appD)
29. [Appendix E: Implementation Readiness](#appE)

## §1. Architecture Overview

Phase 5 adds CSV-based bulk student registration for Main Admin, processing up to 100 rows synchronously in a single HTTP request. It reuses the existing `register_student()` method per row. No background workers, no queues, no new infrastructure.

### New files in `app/modules/students/`

- `bulk_models.py` — SQLAlchemy ORM models for the two new tables
- `bulk_service.py` — upload, confirm/process, reconcile, results
- `bulk_schemas.py` — Pydantic request/response models
- `bulk_constants.py` — enums, `BulkImportOrigin`, `BulkImportSessionFencedError`, `ImporterState`
- `bulk_router.py` — FastAPI routes

### Inherited Phase 4 constraints (all preserved)

- FastAPI modular monolith: Router → Service → Repository
- SQLAlchemy 2.0 sync, `autoflush=False`, `expire_on_commit=False`
- One `Session` per request via `get_db()`
- `register_student()` manages its own transaction (commits at service.py:309)
- `AuditService.record()` only flushes — joins caller's transaction; never commits independently
- Email via `BackgroundTasks` (at-most-once, deferred, never raises)
- PostgreSQL READ COMMITTED isolation throughout
- `pool_pre_ping=True`

### Central design principle — parent-first lock order

Every database transaction in the bulk import subsystem acquires an appropriate lock on the `bulk_upload_sessions` parent row **before** accessing any child row in `bulk_upload_row_results`. This makes the lock dependency graph acyclic and structurally eliminates deadlock.

## §2. Scope and Non-Goals

### In scope

- CSV upload (≤5 MB, ≤100 rows), preview, confirmation
- Synchronous row-by-row registration using `register_student()`
- Durable per-row progress via `bulk_upload_row_results` child table
- Partial success (each row is independent)
- After-loop finalization (terminal write or interrupted audit)
- Manual reconciliation for sessions interrupted by crashes
- CSV downloadable report

### Not in scope (explicitly deferred)

- Section placement movement, promotion, freeze (Phase 5A)
- Student profile pictures; Instructor department assignment
- Asynchronous background processing, Redis, Celery, RQ
- Multi-file batches or pagination of import
- HOD delegated bulk import (permission not delegable)
- Any change to Phase 4 `register_student()` behavior or its approved transaction model
- Any purge mechanism (retention is indefinite pending future phase — D8 deferred)

## §3. Database Model and Constraints

### §3.1. `bulk_upload_sessions`

```
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
    uncertain_count           INTEGER NOT NULL DEFAULT 0,   -- OUTCOME_UNCERTAIN only

    -- Progress heartbeat
    last_started_valid_index  INTEGER NOT NULL DEFAULT 0,

    -- Lifecycle
    status                    VARCHAR(30) NOT NULL DEFAULT 'PENDING_CONFIRMATION',
    expires_at                TIMESTAMPTZ NOT NULL,
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
    CONSTRAINT ck_bulk_sessions_total_rows    CHECK (total_rows BETWEEN 1 AND 100),
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
    CONSTRAINT ck_bulk_sessions_progress  CHECK (
        last_started_valid_index >= 0
        AND last_started_valid_index <= valid_count
    )
);
```

**RECONCILING in CHECK constraint:** Required so the in-transaction `SET status='RECONCILING'` passes CHECK at statement time. RECONCILING is never the committed value — the reconciliation transaction always commits with a terminal status or rolls back entirely.

**Immutable columns (not in UPDATE privilege grant):** `id`, `uploaded_by_user_id`, `department_id`, `file_name`, `file_size_bytes`, `file_sha256`, `total_rows`, `valid_count`, `invalid_count`, `preview_invalid_results`, `expires_at`, `created_at`

### §3.2. `bulk_upload_row_results`

```
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

    -- B8: No separate ix_row_results_session index.
    -- uq_row_results_session_index UNIQUE (upload_session_id, valid_row_index) already
    -- creates a B-tree index with upload_session_id as its leading column, covering all
    -- query patterns on this table. A redundant single-column index wastes write overhead.
);
```

**B8 applied:** The redundant `CREATE INDEX ix_row_results_session ON bulk_upload_row_results (upload_session_id)` present in v9 has been removed. PostgreSQL UNIQUE constraints create an implicit B-tree index; the leading column `upload_session_id` satisfies all lookup patterns.

### §3.3. `validated_rows` JSONB structure

Each element in `validated_rows` MUST contain at minimum:

```
{
  "valid_row_index": 3,       // required integer ≥ 1 (D9 APPROVED)
  "csv_record_number": 5,     // required integer ≥ 1
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

**D9 APPROVED:** `valid_row_index` is required as an integer in every element. Reconciliation (§10.3) reads this array to build the authoritative set of valid indexes. The upload service (M1) must enforce this structure when writing `validated_rows`.

### §3.4. Privilege additions to `provision_db_roles.py`

Add to `NO_DELETE_TABLES`:

```
"bulk_upload_sessions",
"bulk_upload_row_results",
```

Add to `COLUMN_UPDATE_TABLES`:

```
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

**RECONCILING** is a transient in-transaction state. It is SET inside the reconciliation transaction but the transaction always commits with a terminal status or rolls back. Under READ COMMITTED, concurrent readers never observe RECONCILING as a committed value. The API GET endpoint never returns RECONCILING.

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

```
BEGIN;
SET LOCAL lock_timeout = '5s';  -- D2 applied

-- Step 1: Lock parent exclusively; verify PROCESSING
SELECT id
FROM bulk_upload_sessions
WHERE id = :session_id
  AND status = 'PROCESSING'
FOR UPDATE;
-- Use .one_or_none(): row present → locked; None → FENCED
-- (B5: rowcount on SELECT is unreliable in SQLAlchemy 2.0 — returns -1)
-- None → ROLLBACK → T1Result.FENCED
-- row  → parent exclusively locked; proceed

-- Step 2: Insert child row (idempotent)
INSERT INTO bulk_upload_row_results
    (upload_session_id, valid_row_index, csv_record_number,
     status, was_attempted)
VALUES
    (:session_id, :index, :csv_record_number, 'STARTED', NULL)
ON CONFLICT (upload_session_id, valid_row_index) DO NOTHING
RETURNING id;
-- RETURNING row present → new row; proceed to step 3
-- RETURNING empty      → conflict: row already exists from prior committed T1
--                        ROLLBACK → T1Result.SKIP

-- Step 3: Update parent heartbeat and progress (parent lock still held)
UPDATE bulk_upload_sessions
SET last_started_valid_index = :index,
    processing_heartbeat_at  = now(),
    updated_at               = now()
WHERE id = :session_id;
-- Guaranteed 1 row (we hold the exclusive lock)

COMMIT;
-- Result: T1Result.PROCEED
```

### §5.4. ON CONFLICT behavior with parent lock held

We hold the parent exclusive lock before the INSERT. No other T1 for the same `(session_id, valid_row_index)` can be executing concurrently — it would be blocked at step 1. Therefore, a conflict means the child row was inserted by a **previously committed T1**. The correct action is ROLLBACK + SKIP.

### §5.5. Conflict behavior by existing child status

| Existing status | T1 action | T2 called? | Rationale |
| --- | --- | --- | --- |
| STARTED | SKIP | No | Row initialized; may be in reconciliation |
| SUCCEEDED | SKIP | No | Already registered |
| FAILED | SKIP | No | Already classified |
| NOT_PROCESSED | SKIP | No | Already classified |
| OUTCOME_UNCERTAIN | SKIP | No | Awaiting reconciliation |

### §5.6. Outcomes and loop actions

| Result | Description | Loop action |
| --- | --- | --- |
| T1Result.PROCEED | New row inserted; parent updated | Call T2 for this row |
| T1Result.SKIP | Conflict; row already exists | Continue to next row |
| T1Result.FENCED | Parent not PROCESSING | Break loop — FENCED |
| Exception (lock_timeout, connection) | ROLLBACK | Break loop — INTERRUPTED |

## §6. T2 — Registration Transaction

### §6.1. Overview

T2 is `register_student()` from service.py:212–335. It owns its own transaction boundary: commits at service.py:309 (student + placement + audit all committed atomically); rolls back at service.py:248, 311, 322. The import loop does not control T2's transaction. T2 runs after T1 commits and before T3 starts.

### §6.2. BulkImportOrigin and FOR SHARE fencing

```
@dataclass(frozen=True)   # D4 applied: immutable, hashable
class BulkImportOrigin:
    upload_session_id: uuid.UUID
    row_index: int  # valid_row_index
```

When `bulk_origin` is provided, `register_student()` acquires FOR SHARE on the parent session before `_lock_and_validate()`:

```
if bulk_origin is not None:
    session_status = db.scalar(
        select(BulkUploadSession.status)
        .where(BulkUploadSession.id == bulk_origin.upload_session_id)
        .with_for_update(read=True)   # FOR SHARE
    )
    if session_status != BulkUploadStatus.PROCESSING.value:
        raise BulkImportSessionFencedError(...)  # D5: extends StudentServiceError directly
```

### §6.3. T2 lock order (parent-first)

1. `bulk_upload_sessions` FOR SHARE
2. `academic_terms` FOR SHARE (via `_lock_and_validate()`)
3. `departments` FOR SHARE
4. `programs` FOR SHARE
5. `sections` FOR SHARE (if applicable)
6. `users`/`students`/`student_placements` FOR UPDATE (correction path only)

### §6.4. Audit in T2

`AuditService.record()` is called inside `register_student()`. When `bulk_origin` is provided:

```
metadata["bulk_upload_session_id"]     = str(bulk_origin.upload_session_id)
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

## §7. T3 — Result Persistence Transaction

### §7.1. Purpose

Persist the definitive outcome (SUCCEEDED, FAILED, or OUTCOME_UNCERTAIN) to the child row and increment the matching parent counter. The child row transitions from STARTED to a terminal-or-uncertain status.

### §7.2. Lock acquisition order

1. `SELECT ... FOR UPDATE` on `bulk_upload_sessions` — exclusive parent lock
2. `UPDATE bulk_upload_row_results WHERE status='STARTED'` — child row lock (implicit)
3. `UPDATE bulk_upload_sessions SET counter += 1` — parent update (lock already held)
4. `COMMIT`

### §7.3. Complete SQL sequence

```
BEGIN;
SET LOCAL lock_timeout = '5s';  -- D2 applied

-- Step 1: Lock parent exclusively; verify PROCESSING
SELECT id
FROM bulk_upload_sessions
WHERE id = :session_id
  AND status = 'PROCESSING'
FOR UPDATE;
-- Use .one_or_none(): row present → locked; None → FENCED
-- (B5: rowcount on SELECT unreliable in SQLAlchemy 2.0 — returns -1)

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

COMMIT;  -- T3Result.OK
```

`<outcome>_count` mapping: SUCCEEDED → `imported_count`; FAILED → `failed_count`; OUTCOME_UNCERTAIN → `uncertain_count`. NOT_PROCESSED is never written by T3.

### §7.4. Zero-row child update (step 2 rowcount = 0)

```
-- Check existing status and student_id for consistency
SELECT status, student_id
FROM bulk_upload_row_results
WHERE upload_session_id = :session_id
  AND valid_row_index   = :index;
ROLLBACK;
```

| Existing | Intended outcome | Existing student_id | Result |
| --- | --- | --- | --- |
| NULL (no row) | any | — | T3Result.INCONSISTENT (T1 never committed) |
| SUCCEEDED | SUCCEEDED | = :student_id | T3Result.ALREADY_WRITTEN (idempotent) |
| SUCCEEDED | SUCCEEDED | ≠ :student_id | T3Result.INCONSISTENT (student_id mismatch) |
| other terminal/uncertain | same status | n/a | T3Result.ALREADY_WRITTEN (idempotent) |
| any other combination | different | — | T3Result.INCONSISTENT (status mismatch) |

**B6 applied:** For SUCCEEDED, ALREADY_WRITTEN requires both status AND `student_id` to match. A `student_id` mismatch on a SUCCEEDED row is INCONSISTENT even if the status matches — this indicates two different students were committed for the same row index, which cannot be resolved automatically.

Since we held the parent exclusive lock, no concurrent T3 could have changed the row. INCONSISTENT means an application-level bug; break loop INTERRUPTED.

### §7.5. Retry behavior

If T3 times out and rolls back: child row is still STARTED. Retry: step 2 succeeds normally. If T3 committed but the caller crashed: retry finds existing status = `:outcome` → T3Result.ALREADY_WRITTEN. No duplicate counter increment.

## §8. Import Loop: Exception Boundaries and State Machine

### §8.1. ImporterState enum

```
class ImporterState(StrEnum):
    NORMAL      = "NORMAL"       # Loop completed without interruption
    FENCED      = "FENCED"       # Session claimed by reconciliation
    INTERRUPTED = "INTERRUPTED"  # Unexpected failure; some rows unresolved
```

### §8.2. Full loop with separated exception boundaries

```
importer_state = ImporterState.NORMAL

for vr in valid_rows:
    index   = vr["valid_row_index"]
    csv_rec = vr["csv_record_number"]

    # ── T1: Row initialization ──────────────────────────────────────────
    try:
        t1 = persist_started(db, session_id, index, csv_rec)
    except Exception:
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
        try:
            t3 = persist_outcome(db, session_id, index,
                outcome     = RowOutcome.SUCCEEDED,
                student_id  = reg_result.student.id,
                first_name  = reg_result.user.first_name,
                last_name   = reg_result.user.last_name,
                roll_number = reg_result.student.roll_number)
        except Exception:
            importer_state = ImporterState.INTERRUPTED
            break

        if t3 == T3Result.FENCED:
            importer_state = ImporterState.FENCED
            break
        if t3 == T3Result.INCONSISTENT:
            importer_state = ImporterState.INTERRUPTED
            break

        # ── Email (separate try — failure is non-fatal) ─────────────────
        try:
            schedule_email(background_tasks, email_sender,
                build_student_onboarding_email(
                    to              = reg_result.user.email,
                    first_name      = reg_result.user.first_name,
                    program_name    = reg_result.program.name,
                    department_name = reg_result.department.name))
        except Exception:
            logger.warning("Email scheduling failed for row %d (session %s)",
                           index, session_id, exc_info=True)

    elif reg_error is not None:
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

```
if importer_state == ImporterState.FENCED:
    return   # Reconciliation owns this session; do nothing

try:
    db.execute(text("SET LOCAL lock_timeout = '5s'"))  # D2 applied

    # ── Step 0: Acquire ownership (exclusive lock on parent) ──
    ownership_rows = db.execute(
        update(BulkUploadSession)
        .where(BulkUploadSession.id == session_id)
        .where(BulkUploadSession.status == 'PROCESSING')
        .values(processing_heartbeat_at=func.now(), updated_at=func.now())
    ).rowcount

    if ownership_rows == 0:
        db.rollback()
        return  # Session no longer PROCESSING

    # ── Step 1: Find valid indexes with no child row ──
    existing_indexes = set(db.scalars(
        select(BulkUploadRowResult.valid_row_index)
        .where(BulkUploadRowResult.upload_session_id == session_id)
    ).all())
    all_indexes   = set(range(1, valid_count + 1))
    never_started = all_indexes - existing_indexes

    # ── Step 2: Insert NOT_PROCESSED for never-started ──
    for mi in sorted(never_started):
        csv_rec = index_to_csv_record[mi]
        db.execute(
            pg_insert(BulkUploadRowResult).values(
                upload_session_id=session_id,
                valid_row_index  =mi,
                csv_record_number=csv_rec,
                status           ='NOT_PROCESSED',
                was_attempted    =False
            ).on_conflict_do_nothing()
        )

    # ── Step 3: Recalculate ALL counters from child table ──
    counts = {
        status: cnt
        for status, cnt in db.execute(
            select(BulkUploadRowResult.status, func.count())
            .where(BulkUploadRowResult.upload_session_id == session_id)
            .group_by(BulkUploadRowResult.status)
        ).all()
    }
    succeeded     = counts.get('SUCCEEDED', 0)
    failed        = counts.get('FAILED', 0)
    not_processed = counts.get('NOT_PROCESSED', 0)
    uncertain     = counts.get('OUTCOME_UNCERTAIN', 0)
    started       = counts.get('STARTED', 0)
    unresolved    = uncertain + started

    # ── Step 4a: Unresolved rows remain ──
    if unresolved > 0:
        db.execute(
            update(BulkUploadSession)
            .where(BulkUploadSession.id == session_id)
            .where(BulkUploadSession.status == 'PROCESSING')
            .values(
                imported_count      =succeeded,
                failed_count        =failed,
                not_processed_count =not_processed,
                uncertain_count     =uncertain,
                updated_at          =func.now())
        )
        audit_service.record(
            action       =AuditAction.BULK_IMPORT_INTERRUPTED,
            entity_type  =AuditEntityType.BULK_UPLOAD_SESSION,
            entity_id    =session_id,
            actor_user_id=actor.id,
            metadata     ={"unresolved_count": unresolved,
                           "started_count": started,
                           "uncertain_count": uncertain,
                           "imported_count": succeeded,
                           "failed_count": failed},
            ip_address=ip_address)
        db.commit()
        return

    # ── Step 4b: All rows definitive → verify totals → write terminal ──
    total = succeeded + failed + not_processed
    if total != valid_count:
        db.rollback()
        return  # Inconsistent count; leave for reconciliation

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
        metadata     ={"status": terminal, "imported_count": succeeded,
                       "failed_count": failed, "not_processed_count": not_processed},
        ip_address=ip_address)
    db.commit()

except Exception:
    try: db.rollback()
    except Exception: pass
    # Session stays PROCESSING with stale heartbeat → reconciliation.
```

### §9.4. Ownership-conflict proof

After-loop acquires the parent exclusive lock via the heartbeat UPDATE. Reconciliation also needs the parent exclusive lock. They cannot both hold it simultaneously: whichever acquires it first commits terminal; the other sees `ownership_rows=0` (after-loop) or 409 not eligible (reconciliation).

## §10. Reconciliation Transaction

### §10.1. Single-transaction model

All reconciliation work — claim, read, classify, write, recalculate counters, terminal write, audit — runs in one transaction. The exclusive row lock on the parent session is held from the claim UPDATE through COMMIT or ROLLBACK.

### §10.2. Claim (parent-first)

```
BEGIN;
SET LOCAL lock_timeout = '10s';  -- D3 applied

UPDATE bulk_upload_sessions
SET status                    = 'RECONCILING',
    reconciliation_claimed_by = :actor_id,
    reconciliation_started_at = now(),
    reconciliation_claim_token= :new_token_uuid,
    updated_at                = now()
WHERE id                        = :upload_id
  AND status                    = 'PROCESSING'
  AND processing_heartbeat_at   < now() - INTERVAL '5 minutes'  -- D1 applied
RETURNING id, valid_count, validated_rows;
-- rowcount = 0:
--   if lock_timeout: ROLLBACK → 503
--   otherwise:       ROLLBACK → 409 (not eligible)
-- rowcount = 1: parent exclusively locked; proceed
```

### §10.3. Read phase (within reconciliation transaction)

#### Step 1 — Load and structurally validate validated_rows (B2)

```
validated_rows_data = session_row["validated_rows"]

# B2: Structural validation before building valid_index_set
if not isinstance(validated_rows_data, list):
    db.rollback()
    raise ReconciliationError("validated_rows is not a list")
if len(validated_rows_data) == 0:
    db.rollback()
    raise ReconciliationError("validated_rows is empty; cannot safely reconcile")
if len(validated_rows_data) != valid_count:
    db.rollback()
    raise ReconciliationError(
        f"validated_rows length {len(validated_rows_data)} != valid_count {valid_count}")

valid_index_set = set()
index_to_csv_map = {}
for i, element in enumerate(validated_rows_data):
    if not isinstance(element, dict):
        db.rollback()
        raise ReconciliationError(f"validated_rows[{i}] is not an object")
    raw_vri = element.get("valid_row_index")
    # bool is subclass of int in Python — MUST check bool first
    if isinstance(raw_vri, bool) or not isinstance(raw_vri, int) or raw_vri < 1:
        db.rollback()
        raise ReconciliationError(
            f"validated_rows[{i}].valid_row_index invalid: {raw_vri!r}")
    if raw_vri in valid_index_set:
        db.rollback()
        raise ReconciliationError(f"duplicate valid_row_index {raw_vri} in validated_rows")
    valid_index_set.add(raw_vri)
    raw_crn = element.get("csv_record_number")
    if isinstance(raw_crn, bool) or not isinstance(raw_crn, int) or raw_crn < 1:
        db.rollback()
        raise ReconciliationError(
            f"validated_rows[{i}].csv_record_number invalid: {raw_crn!r}")
    index_to_csv_map[raw_vri] = raw_crn
if valid_index_set != set(range(1, valid_count + 1)):
    db.rollback()
    raise ReconciliationError("valid_index_set does not match expected range 1..valid_count")
```

#### Step 2 — Load all existing child rows

```
SELECT valid_row_index, status, student_id, was_attempted,
       error_code, is_retryable
FROM bulk_upload_row_results
WHERE upload_session_id = :upload_id;
```

#### Step 3 — Load audit records for this session (B1)

```
SELECT
    entity_id,
    event_metadata->'bulk_upload_valid_row_index'  AS raw_index_jsonb,
    event_metadata->>'bulk_upload_session_id'       AS raw_session_id
FROM audit_logs
WHERE action = 'student.registered'
  AND event_metadata->>'bulk_upload_session_id' = :upload_id_str;
```

**B1 applied:** `->` (not `->>`) on `bulk_upload_valid_row_index` preserves the native JSONB type, allowing Python to distinguish JSON integer 3 from JSON string "3". `->>` in the WHERE clause is correct (text comparison for UUID).

**B1 limitation:** The WHERE filter `event_metadata->>'bulk_upload_session_id' = :upload_id_str` only returns records whose session_id matches. Records with a missing or mismatched `bulk_upload_session_id` are excluded before Python sees them. SESSION_ID_MISSING and SESSION_ID_MISMATCH contradictions **cannot be detected** by this query. This is an inherent limitation of a filter-based retrieval. The fail-closed guarantee is preserved for all detectable contradictions.

#### Step 4 — Validate every audit record (B1)

For each audit record retrieved:

```
raw_index = row.raw_index_jsonb   # native Python type (JSONB via ->)
if isinstance(raw_index, bool):       # bool is subclass of int — MUST check first
    raise Contradiction(INDEX_NOT_INTEGER)
if raw_index is None:
    raise Contradiction(INDEX_MISSING)
if not isinstance(raw_index, int):    # catches str "3", float 3.5, etc.
    raise Contradiction(INDEX_NOT_INTEGER)
if raw_index < 1:
    raise Contradiction(INDEX_NOT_POSITIVE)
if raw_index > valid_count:
    raise Contradiction(INDEX_OUT_OF_RANGE)
if raw_index not in valid_index_set:
    raise Contradiction(INDEX_NOT_IN_VALIDATED_ROWS)
if row.entity_id is None:
    raise Contradiction(ENTITY_ID_MISSING)
if raw_index in audit_by_index:
    raise Contradiction(DUPLICATE_AUDIT_INDEX)
```

| Check | Failure condition | Contradiction type |
| --- | --- | --- |
| Index present | `raw_index_jsonb` is JSON null | INDEX_MISSING |
| Index is integer, not boolean | Python bool, or non-int type | INDEX_NOT_INTEGER |
| Index is positive | parsed_index \< 1 | INDEX_NOT_POSITIVE |
| Index within valid_count | parsed_index > valid_count | INDEX_OUT_OF_RANGE |
| Index in validated_rows set | parsed_index not in valid_index_set | INDEX_NOT_IN_VALIDATED_ROWS |
| Entity ID present | entity_id is NULL | ENTITY_ID_MISSING |
| No duplicate index | Another record has same parsed_index | DUPLICATE_AUDIT_INDEX |

If any check fails → collect contradiction. After all records, if any contradictions → §10.4. SESSION_ID_MISSING and SESSION_ID_MISMATCH are undetectable (see B1 limitation note above).

#### Step 5 — Build audit and child maps

```
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

```
if contradictions:
    db.rollback()
    raise ContradictionError(contradictions=[...])
```

HTTP 409 with body:

```
{
  "error": "RECONCILIATION_CONTRADICTION",
  "contradictions": [
    {"valid_row_index": 5,
     "type": "SUCCEEDED_NO_AUDIT",
     "details": "Child is SUCCEEDED with student_id=<uuid> but no audit exists"}
  ]
}
```

Session remains PROCESSING with stale heartbeat (RECONCILING was never committed). After admin corrects the data issue, reconciliation can be retried.

### §10.5. Classification (apply §12 matrix to every index in valid_index_set)

For each `index` in `valid_index_set`: get child row (may be None) and audit entry (may be None). Apply §12 decision matrix. Collect contradictions without short-circuiting. After all indexes processed: if any contradiction → §10.4.

### §10.6. Write phase (all contradictions absent)

**Step 6** — Apply row-level changes per §12 (INSERT or UPDATE for each index needing action). No separate child locking needed — the parent exclusive lock prevents any T1/T3 from accessing children.

**Step 7** — Recalculate all counters:

```
SELECT status, count(*) FROM bulk_upload_row_results
WHERE upload_session_id = :upload_id GROUP BY status;
```

Verify: `succeeded + failed + not_processed = valid_count` AND `uncertain = 0` AND `started = 0`. If not → contradiction in write logic → ROLLBACK.

**Step 8** — Terminal write (D6 applied):

```
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
  AND reconciliation_claim_token= :my_token;  -- D6: defense-in-depth
-- rowcount = 0 → should not happen (we hold lock and set RECONCILING)
--              → ROLLBACK as safety
```

**Step 9** — `audit_service.record(BULK_IMPORT_RECONCILED, ...)` → flush. **Step 10** — COMMIT.

## §11. Complete Lock-Order Table

| Transaction | Parent operation | Child operation | Parent update |
| --- | --- | --- | --- |
| T1 | SELECT FOR UPDATE (X) WHERE PROCESSING | INSERT child ON CONFLICT DO NOTHING | UPDATE session SET heartbeat |
| T2 | SELECT FOR SHARE (S) WHERE PROCESSING | (no child bulk_upload access) | (none) |
| T3 | SELECT FOR UPDATE (X) WHERE PROCESSING | UPDATE child WHERE status='STARTED' | UPDATE session SET counter += 1 |
| After-loop | UPDATE session SET heartbeat (implicit X) WHERE PROCESSING | SELECT + INSERT NOT_PROCESSED + counter recalculate | Included in claim UPDATE |
| Reconciliation | UPDATE session SET status='RECONCILING' (implicit X) WHERE PROCESSING | SELECT + INSERT/UPDATE rows | Terminal UPDATE WHERE RECONCILING |

### Lock compatibility matrix (parent row)

| Holder \\ Requester | T1 (X) | T2 (S) | T3 (X) | After-loop (X) | Recon (X) |
| --- | --- | --- | --- | --- | --- |
| **T1 (X)** | waits | T2 waits | waits | waits | waits |
| **T2 (S)** | T1 waits | allowed | T3 waits | waits | waits |
| **T3 (X)** | waits | T2 waits | waits | waits | waits |
| **After-loop (X)** | waits | T2 waits | waits | N/A | waits |
| **Recon (X)** | waits | T2 waits | waits | waits | 409 |

**Deadlock proof:** A deadlock requires a cycle. With parent-first ordering, every transaction acquires the parent lock before any child lock. No transaction can be "holding a child lock while waiting for the parent lock." The dependency graph is acyclic. No deadlock detection is needed as a mitigation strategy.

## §12. Complete Child-State Consistency Matrix

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

### §12.3. Extended field validation per status (B3)

**B3 applied:** Extended to cover `was_attempted`, `error_code`, `is_retryable` columns.

| Status | student_id | was_attempted | error_code | is_retryable | Audit |
| --- | --- | --- | --- | --- | --- |
| SUCCEEDED | NOT NULL | TRUE | NULL | NULL | Present; entity_id = student_id |
| FAILED | NULL | TRUE | NOT NULL | NOT NULL | Absent |
| STARTED | NULL | NULL *(any other → CONTRADICTION)* | NULL | NULL | n/a (in-flight) |
| OUTCOME_UNCERTAIN | NULL | TRUE | NULL | NULL | Absent (expected) |
| NOT_PROCESSED | NULL | FALSE, NULL, or TRUE | NULL | NULL | Absent |

**STARTED with `was_attempted ≠ NULL` is a CONTRADICTION.** Any deviation from this table is a contradiction that must fail closed.

## §13. Counter Definitions and Invariants

### §13.1. Parent counter definitions

| Column | Tracks | Child status |
| --- | --- | --- |
| `imported_count` | Successfully registered students | SUCCEEDED |
| `failed_count` | Rows with known business errors | FAILED |
| `not_processed_count` | Rows with no committed registration | NOT_PROCESSED |
| `uncertain_count` | Rows where T2 was invoked but outcome unknown | OUTCOME_UNCERTAIN **only** |

**STARTED rows are NOT counted in any parent counter.** `uncertain_count` does NOT include STARTED rows.

### §13.2. Non-terminal invariant

```
imported_count + failed_count + not_processed_count + uncertain_count ≤ valid_count
```

Gap = STARTED rows + rows with no child record.

### §13.3. Terminal invariant (enforced by application and CHECK constraint)

```
imported_count + failed_count + not_processed_count = valid_count
uncertain_count = 0
-- No STARTED or OUTCOME_UNCERTAIN rows in child table
```

### §13.4. T3 counter updates

T3 increments exactly one counter in the same transaction as the child UPDATE (within parent X lock): SUCCEEDED → `imported_count += 1`; FAILED → `failed_count += 1`; OUTCOME_UNCERTAIN → `uncertain_count += 1`. NOT_PROCESSED is never written by T3.

### §13.5. After-loop and reconciliation: full recalculation

Both after-loop and reconciliation derive all counters by querying the child table (`GROUP BY status`). This eliminates any incremental drift from T3 counter failures.

### §13.6. `was_attempted` semantics (three-state)

| Value | Meaning | Who sets it |
| --- | --- | --- |
| TRUE | Registration was definitely invoked | T3 (all outcomes); reconciliation (STARTED/UNCERTAIN+audit) |
| FALSE | Registration was definitely never invoked | After-loop NOT_PROCESSED for never-started rows; reconciliation: no child + no audit |
| NULL | Whether invocation occurred cannot be determined | T1 (initial); reconciliation: STARTED + no audit → NOT_PROCESSED |

## §14. Audit Evidence and Reconciliation Rules

### §14.1. Audit action constants

| Constant | Value | Trigger | Transaction |
| --- | --- | --- | --- |
| BULK_IMPORT_UPLOADED | `bulk_import.uploaded` | Session INSERT | Same as upload INSERT |
| BULK_IMPORT_CONFIRMED | `bulk_import.confirmed` | Confirm UPDATE | Same as confirm UPDATE |
| STUDENT_REGISTERED | `student.registered` (Phase 4) | Per-row registration | T2's internal transaction |
| BULK_IMPORT_COMPLETED | `bulk_import.completed` | Terminal write | Same as after-loop terminal UPDATE |
| BULK_IMPORT_INTERRUPTED | `bulk_import.interrupted` | Unresolved rows remain | Same as after-loop counter UPDATE |
| BULK_IMPORT_RECONCILED | `bulk_import.reconciled` | Reconciliation terminal | Same as reconciliation terminal UPDATE |

New entity type: `BULK_UPLOAD_SESSION = "bulk_upload_session"`. `AuditService.record()` only flushes (never commits).

### §14.2. Per-registration audit extension

```
metadata["bulk_upload_session_id"]     = str(bulk_origin.upload_session_id)
metadata["bulk_upload_valid_row_index"] = bulk_origin.row_index
```

Committed atomically with the student at service.py:309. No committed student lacks an audit record; no audit record exists without a committed student.

### §14.3. Reconciliation audit query (B1)

```
SELECT
    entity_id,
    event_metadata->'bulk_upload_valid_row_index'  AS raw_index_jsonb,
    event_metadata->>'bulk_upload_session_id'       AS raw_session_id
FROM audit_logs
WHERE action = 'student.registered'
  AND event_metadata->>'bulk_upload_session_id' = :upload_id_str;
```

Uses existing `ix_audit_logs_action` index on `action`.

### §14.4. Partial audit index — DEFERRED (D7)

The proposed partial index on `(event_metadata->>'bulk_upload_session_id')` is **DEFERRED** and excluded from the M1 migration. It is not required for correctness; the existing `ix_audit_logs_action` index is sufficient. It can be introduced in a future migration if query performance justifies it as the audit table grows.

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
| CW7 | After-loop timeout | Per-row | Per-row | Reconciliation (heartbeat stale after 5 min) |
| CW8 | After-loop committed terminal | All final | completed audit | Done |
| CW9 | After-loop committed interrupted | Remaining STARTED/UNCERTAIN | interrupted audit | Reconciliation |
| CW10 | Reconciliation ROLLBACK (contradiction) | Pre-reconciliation state (unchanged) | Unchanged | Admin investigates; retry |
| CW11 | Reconciliation committed | All final | reconciled audit | Done |

### §15.2. Session recovery guarantee

Every PROCESSING session with a stale heartbeat (> 5 minutes) is eligible for reconciliation. After-loop rollbacks leave the session in PROCESSING. Reconciliation can always claim and resolve such a session.

### §15.3. Unrecoverable states

A PROCESSING session with contradictions in its audit or child table cannot be automatically resolved. The session remains PROCESSING, accessible for admin review. The admin resolves the data issue with migration-role DB access and retries reconciliation.

## §16. API Contracts

### §16.1. Upload — `POST /admin/bulk-uploads`

Permission: `student.bulk_import` (MAIN_ADMIN only). Request: `multipart/form-data` — `file` (≤5 MB), `department_id`.

Response 201:

```
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

#### preview_invalid_results structure (B7)

```
[
  {
    "csv_record_number": 3,
    "errors": [
      {
        "field": "semester_level",
        "error_code": "INVALID_SEMESTER_LEVEL",
        "message": "Must be a positive integer not exceeding program duration"
      }
    ]
  }
]
```

**B7 applied:** `preview_invalid_results` stores only `csv_record_number`, machine-readable `error_code`, and rule-description `message`. No PII fields (`first_name`, `last_name`, `email`, `phone`, `address`, `roll_number`). Error messages describe the rule, not the submitted value. Indefinite retention is acceptable because this structure contains no personal data.

Errors: 400 (zero valid rows, column mismatch, blank file, >100 rows, unknown/duplicate/missing headers), 413 (>5 MB), 422 (UTF-8 failure, CSV parse error).

### §16.2. Confirm — `POST /admin/bulk-uploads/{id}/confirm`

```
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

0 rows: check status (expired → lazy expire + 409; not found → 404; other → 409). Triggers synchronous import loop. Response 200 with terminal results if all rows settled, or PROCESSING status with `needs_reconciliation: true` if unresolved rows remain.

### §16.3. Get Status — `GET /admin/bulk-uploads/{id}`

Never returns RECONCILING (concurrent reads see last committed state under READ COMMITTED — PROCESSING). Applies lazy expiry. For terminal states: includes child-table results (omits email/phone/address), `preview_invalid_results`. For PROCESSING: live counters, `is_stale`, `needs_reconciliation`.

### §16.4. Reconcile — `POST /admin/bulk-uploads/{id}/reconcile`

Requirements: `status='PROCESSING'`, `processing_heartbeat_at < now() - '5 minutes'` (D1). Responses: 200 (terminal results), 409 RECONCILIATION_CONTRADICTION, 409 (not eligible), 503 (lock timeout).

### §16.5. Report — `GET /admin/bulk-uploads/{id}/report`

Terminal sessions only (409 otherwise). `text/csv; charset=utf-8`. Columns: `csv_record_number`, `valid_row_index`, `first_name`, `last_name`, `roll_number`, `status`, `was_attempted`, `error_code`, `error_message`, `corrective_action`, `is_retryable`. Formula injection prevention: prefix cells starting with `= + - @ TAB CR LF`.

### §16.6. List — `GET /admin/bulk-uploads`

Paginated. Filters: `department_id`, `status`. Summary counters only.

## §17. CSV Preview and Confirmation

### §17.1. File acceptance

Read up to 5,000,001 bytes. If more data remains → 413.

1. UTF-8 decode. Strip BOM. Failure → 422.
2. `csv.field_size_limit` is set **once at application startup**, not per-request (B4). See note below.
3. `csv.reader(strict=True)`. Parse failure → 422.
4. Header validation (§17.2).
5. Column-count consistency: every non-blank data record must have exactly the header's field count. Mismatch → 400 citing record number.
6. Count non-blank data records. Zero → 400. >100 → 400.

**B4 applied:** `csv.field_size_limit(8192)` is set once at application startup (e.g., in the FastAPI lifespan event or module-level in `bulk_service.py`). The per-request save/restore pattern from v9 was a data race: under concurrent upload requests in the same process, one request could restore the old limit while another request's CSV parse is in progress. The 8192-byte limit is preserved.

```
# Application startup (lifespan event or module-level in bulk_service.py):
import csv
csv.field_size_limit(8192)   # set once; process-wide; never changed per-request

# upload_csv() calls csv.reader() normally — no save/restore needed
```

### §17.2. Headers

Required (case-insensitive, trimmed): `first_name`, `last_name`, `email`, `roll_number`, `program_code`, `semester_level`, `admission_term`. Optional: `section_name`, `phone`, `address`. Missing required → 400. Duplicate → 400. Unknown → 400.

### §17.3. Record numbering

`csv_record_number`: 1-indexed ordinal of non-blank data records after the header. Multi-line quoted records count as one. `valid_row_index`: 1..`valid_count`, assigned in `csv_record_number` order to records passing all preview validations.

### §17.4. Formula-cell validation

All columns except `phone` are checked. Rule: reject if first character is in `{= + - @ \t \r \n}` or first non-whitespace is in `{= + - @}`.

### §17.5. Semester-level validation

Parse as integer ≥ 1. After resolving `program_code`, verify ≤ `program.duration_semesters`. No arbitrary cap in CSV code — the Phase 4 rule at service.py:734 is authoritative. DB backstop is `ck_student_placements_level BETWEEN 1 AND 16`.

### §17.6. Lazy expiry

```
UPDATE bulk_upload_sessions
SET status       = 'EXPIRED',
    validated_rows = '[]'::jsonb,
    updated_at   = now()
WHERE id     = :upload_id
  AND status = 'PENDING_CONFIRMATION'
  AND expires_at <= now();
```

## §18. Expiry and Retention

### §18.1. Expiry timing

`expires_at = created_at + INTERVAL '30 minutes'`. Immutable after upload. Lazy expiry applied on reads.

### §18.2. Data classification

| Location | PII | Scrubbing |
| --- | --- | --- |
| `validated_rows` JSONB (parent) | Yes: all parsed fields | Scrubbed to `[]` atomically with terminal write |
| `preview_invalid_results` JSONB (parent) | None: record numbers + error codes | Never scrubbed (no PII) |
| Child `first_name`, `last_name`, `roll_number` | Yes | Retained until purge |
| Child other columns | None | Retained until purge |
| Audit logs | Roll number, program, term (no email/phone/address) | Append-only; never scrubbed |

### §18.3. Current implementation: indefinite retention

No purge mechanism exists. All records retained indefinitely. This is explicitly accepted as temporary (D8 deferred).

### §18.4. Proposed retention (D8 — DEFERRED)

When a future purge mechanism is implemented: Terminal sessions — 90 days from `completed_at`; EXPIRED sessions — 30 days from `expires_at`. No purge mechanism is part of Phase 5.

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

## §20. Email Behavior

Email is at-most-once via FastAPI `BackgroundTasks`, registered AFTER `register_student()` returns (after T2 commit), in a separate try block from T3.

```
try:
    schedule_email(background_tasks, email_sender,
        build_student_onboarding_email(...))
except Exception:
    logger.warning("Email scheduling failed for row %d", index, exc_info=True)
# Always continue to next row.
```

`schedule_email` → `background_tasks.add_task(send_email_safely, ...)`. Registers a callable only; does not send anything synchronously. `send_email_safely` catches all exceptions internally. Email failure after committed SUCCEEDED: row remains SUCCEEDED; loop continues. No email failure can change a committed outcome. Email can be resent via `student.onboarding_email.resent` endpoint.

## §21. Logging and Privacy

| Event | Level | PII logged? |
| --- | --- | --- |
| Upload received | INFO | No |
| Preview complete | INFO | No |
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

## §22. Testing Strategy

### §22.1. Lock-order and concurrency tests

Tests use real PostgreSQL transactions with controlled barriers (`threading.Barrier`). No `time.sleep()`.

| Test | Scenario | Expected |
| --- | --- | --- |
| CT-LO-1 | T3 vs reconciliation (original v8 deadlock scenario) — T3 holds parent X; reconciliation tries X → blocks → T3 commits → reconciliation sees fresh heartbeat → 409 | No deadlock; sequential |
| CT-LO-2 | T1 vs reconciliation — T1 holds parent X; reconciliation blocks; T1 commits with fresh heartbeat → reconciliation 409 | No deadlock |
| CT-LO-3 | T3 vs after-loop — after-loop holds X; T3 blocks; after-loop commits terminal; T3 sees terminal → FENCED | Child not modified twice |
| CT-LO-4 | T1 vs after-loop — analogous to CT-LO-3 | T1 FENCED |
| CT-LO-5 | T2 registration vs reconciliation — T2 holds FOR SHARE; reconciliation waits; T2 commits; reconciliation sees committed audit | SUCCEEDED resolved |
| CT-LO-6 | Two simultaneous T1s for same row — first commits; second ON CONFLICT → SKIP | Exactly one child row |
| CT-LO-7 | Retry after T1 rollback — T1 connection lost → ROLLBACK; child not exist; retry inserts cleanly | Child created on retry |
| CT-LO-8 | T3 with terminal parent — after-loop committed terminal; T3 SELECT FOR UPDATE WHERE PROCESSING → None → FENCED | Child not modified |

### §22.2. FK and schema tests

| Test | Scenario | Expected |
| --- | --- | --- |
| CT-FK-1 | Insert child row with student_id = valid users.id that is NOT a student | FK violation |
| CT-FK-2 | Insert child row with student_id = valid students.id | Succeeds |
| CT-FK-3 | Attempt to delete student row referenced by child result | ON DELETE RESTRICT error |

### §22.3. Audit index validation tests (B1)

| Test | Setup | Expected |
| --- | --- | --- |
| CT-AI-1 | Audit with raw_index_jsonb = JSON string "3" (extracted via ->) | Contradiction: INDEX_NOT_INTEGER |
| CT-AI-2 | Audit with raw_index_jsonb = JSON boolean true | Contradiction: INDEX_NOT_INTEGER |
| CT-AI-3 | Audit with raw_index_jsonb = JSON number 3.5 (decimal) | Contradiction: INDEX_NOT_INTEGER |
| CT-AI-4 | Audit with raw_index_jsonb = JSON null | Contradiction: INDEX_MISSING |
| CT-AI-5 | Audit with raw_index = 999 for 20-row session | Contradiction: INDEX_OUT_OF_RANGE |
| CT-AI-6 | Audit with raw_index = 3 but index 3 not in validated_rows | Contradiction: INDEX_NOT_IN_VALIDATED_ROWS |
| CT-AI-7 | Two audit records with same raw_index = 2 | Contradiction: DUPLICATE_AUDIT_INDEX |
| CT-AI-8 | Audit record with entity_id = NULL | Contradiction: ENTITY_ID_MISSING |
| CT-AI-9 | Valid audit records for all indexes | No contradiction; proceeds |
| CT-AI-10 | validated_rows = \[\] while PROCESSING | ReconciliationError (fail closed) |
| CT-AI-11 | Concurrent upload requests with field_size_limit=8192 set at startup | No race; both requests parse correctly |

### §22.4. validated_rows structural validation tests (B2)

| Test | Setup | Expected |
| --- | --- | --- |
| CT-VR-1 | validated_rows is not a list (e.g., null or dict) | ReconciliationError; ROLLBACK |
| CT-VR-2 | validated_rows length ≠ valid_count | ReconciliationError; ROLLBACK |
| CT-VR-3 | An element is not a dict | ReconciliationError; ROLLBACK |
| CT-VR-4 | valid_row_index = true (Python bool) | ReconciliationError; ROLLBACK |
| CT-VR-5 | valid_row_index = "3" (string) | ReconciliationError; ROLLBACK |
| CT-VR-6 | Duplicate valid_row_index values | ReconciliationError; ROLLBACK |
| CT-VR-7 | valid_index_set ≠ set(range(1, valid_count+1)) | ReconciliationError; ROLLBACK |

### §22.5. Child-state consistency matrix tests (B3)

| Test | Child state | student_id | was_attempted | Audit | Expected |
| --- | --- | --- | --- | --- | --- |
| CT-CS-1 | SUCCEEDED | NULL | * | * | Contradiction |
| CT-CS-2 | SUCCEEDED | Present | * | Absent | Contradiction |
| CT-CS-3 | SUCCEEDED | Present | * | entity mismatch | Contradiction |
| CT-CS-4 | FAILED | Present | * | * | Contradiction |
| CT-CS-5 | FAILED | NULL | * | Present | Contradiction |
| CT-CS-6 | STARTED | NULL | TRUE or FALSE | * | Contradiction (was_attempted ≠ NULL) |
| CT-CS-7 | STARTED | NULL | NULL | Absent | → NOT_PROCESSED (was_attempted=NULL) |
| CT-CS-8 | STARTED | NULL | NULL | Present | → SUCCEEDED (was_attempted=TRUE) |
| CT-CS-9 | OUTCOME_UNCERTAIN | NULL | TRUE | Absent | → NOT_PROCESSED (was_attempted=TRUE kept) |
| CT-CS-10 | OUTCOME_UNCERTAIN | NULL | TRUE | Present | → SUCCEEDED |
| CT-CS-11 | OUTCOME_UNCERTAIN | Present | * | * | Contradiction |
| CT-CS-12 | No child | — | — | Absent | INSERT NOT_PROCESSED (was_attempted=FALSE) |
| CT-CS-13 | No child | — | — | Present | INSERT SUCCEEDED |

### §22.6. T3 idempotency tests (B6)

| Test | Setup | Expected |
| --- | --- | --- |
| CT-T3-1 | Existing SUCCEEDED with same student_id as intended | ALREADY_WRITTEN (idempotent) |
| CT-T3-2 | Existing SUCCEEDED with different student_id | INCONSISTENT |
| CT-T3-3 | Existing FAILED intended FAILED (same error_code) | ALREADY_WRITTEN (idempotent) |

### §22.7. Counter invariant tests

| Test | Scenario | Expected |
| --- | --- | --- |
| CT-CNT-1 | N T3 SUCCEEDED commits | parent.imported_count = N |
| CT-CNT-2 | 3 STARTED rows remain | None counted in any parent counter |
| CT-CNT-3 | OUTCOME_UNCERTAIN rows | uncertain_count = count; STARTED not included |
| CT-CNT-4 | After-loop recalculates | Full GROUP BY; not incremental |
| CT-CNT-5 | Terminal write | uncertain_count = 0; sum = valid_count; CHECK passes |

### §22.8. STARTED-row safety tests

| Test | Scenario | Expected |
| --- | --- | --- |
| S1 | T1→T2 success→crash before T3 | STARTED; audit exists; reconciliation → SUCCEEDED |
| S2 | T1→T2 rollback→crash before T3 | STARTED; no audit; reconciliation → NOT_PROCESSED |
| S3 | Row 1 SUCCEEDED, row 2 STARTED, row 3 never started | After-loop inserts NOT_PROCESSED for row 3; row 2 stays STARTED; reconciliation resolves row 2 |

### §22.9. Reconciliation state tests

| Test | Setup | Expected |
| --- | --- | --- |
| CT-RC-1 | Stale PROCESSING session (>5 min) | Claim succeeds |
| CT-RC-2 | Fresh heartbeat (\< 5 min) | Claim fails: 409 |
| CT-RC-3 | Terminal session | Claim fails: 409 |
| CT-RC-4 | Contradiction detected | ROLLBACK; session stays PROCESSING; 409 with contradictions |
| CT-RC-5 | RECONCILING visible from concurrent reader | Never; reader sees PROCESSING |
| CT-RC-6 | After contradiction rollback, retry with fixed data | Succeeds; terminal write |

## §23. Implementation Milestones

### M1 — Schema, Upload, Preview (3–4 days)

**Gate:** D2 ✓, D3 ✓, D9 ✓. D7 excluded from migration (deferred). All satisfied.

1. `bulk_models.py`: `BulkUploadSession`, `BulkUploadRowResult` with `students(id)` FK
2. `bulk_constants.py`: `BulkImportOrigin`, `BulkImportSessionFencedError`, `ImporterState`, `RowOutcome`, `T1Result`, `T3Result`
3. Alembic migration: both tables, all constraints, UNIQUE index. No partial audit index (D7 deferred).
4. `provision_db_roles.py`: add entries per §3.4
5. `bulk_service.py`: `upload_csv()` — strict parsing with `csv.field_size_limit(8192)` at startup (B4), formula check, semester-level check, `validated_rows` JSONB with `valid_row_index` per element (D9)
6. `bulk_schemas.py`, `bulk_router.py`: `POST /admin/bulk-uploads`
7. Audit constants: `BULK_IMPORT_UPLOADED`, `BULK_UPLOAD_SESSION` entity type
8. Tests: CT-FK-1, CT-FK-2, CT-AI-11, upload headers, formula-phone exemption, semester-level, >100 rows, >5 MB, zero valid, BOM, permissions, CT-VR-1 through CT-VR-7

### M2 — Confirm, Import Loop with Parent-First Locking (3–4 days)

**Gate:** D1 ✓, D4 ✓, D5 ✓. All satisfied.

1. `bulk_origin` parameter wiring on `register_student()`: FOR SHARE fencing; `BulkImportSessionFencedError` extends `StudentServiceError` (D5)
2. `persist_started()` (T1): parent FOR UPDATE → INSERT ON CONFLICT DO NOTHING → update heartbeat; uses `.one_or_none()` (B5)
3. `persist_outcome()` (T3): parent FOR UPDATE → UPDATE child WHERE STARTED → increment counter; uses `.one_or_none()` (B5); checks student_id on ALREADY_WRITTEN for SUCCEEDED (B6)
4. `confirm_and_process()`: confirmation claim, import loop with separated try blocks (§8.2), after-loop (§9.3)
5. `POST /admin/bulk-uploads/{id}/confirm`
6. Lazy expiry
7. Tests: CT-LO-1 through CT-LO-8, exception boundary table (§8.3), CT-CNT-1 through CT-CNT-5, S1 through S3, CT-T3-1 through CT-T3-3

### M3 — Reconciliation, Results, Report (2–3 days)

**Gate:** D6 ✓. All satisfied.

1. `reconcile()`: single-transaction model (§10), B2 structural validation, B1 audit query fix (`->`), Python bool-before-int check, child-state consistency matrix (§12) with B3 extended fields, fail-closed contradiction handling, D6 token check
2. GET status, GET report, POST reconcile, GET list
3. Response shaping: RECONCILING never returned; `was_attempted` three-state display; `needs_reconciliation` flag
4. Tests: CT-AI-1 through CT-AI-10, CT-VR-1 through CT-VR-7, CT-CS-1 through CT-CS-13, CT-RC-1 through CT-RC-6

### M4 — Integration, Edge Cases, Concurrency (2 days)

1. End-to-end: upload → confirm → results → report
2. All CT-LO concurrency tests under real database
3. Failure injection: DB disconnect at CW2/CW4/CW6; reconciliation after each crash window
4. Count invariant verification under concurrent T3s
5. Contradiction scenarios (CT-AI + CT-CS) with real audit log entries

## §24. v8 Improvements — Explicit Revalidation

| # | v8 point | v9/v10 status |
| --- | --- | --- |
| 1 | RECONCILING transient, never externally committed | §4.1, §10.1: SET RECONCILING within single transaction; always commits to terminal or ROLLBACK ✓ |
| 2 | Reconciliation holds parent lock for complete transaction | §10.1–§10.6: single transaction; claim UPDATE holds exclusive lock until COMMIT/ROLLBACK ✓ |
| 3 | After-loop and reconciliation use parent-first locking | §9.2, §10.2: both acquire parent lock first ✓ |
| 4 | Parent ownership rowcounts checked | §9.3 step 0: ownership_rows == 0 → return; §10.6 step 8: rowcount = 0 → ROLLBACK ✓ |
| 5 | Separated exception boundaries | §8.2: separate try blocks for T1, T2, T3, email ✓ |
| 6 | Registration and audit atomically consistent | §6.4: AuditService.record() joins T2's transaction ✓ |
| 7 | register_student() called once per valid row | §8.2: T1 SKIP prevents second T2 invocation ✓ |
| 8 | register_student() retains internal commit/rollback | §6.1: unchanged ✓ |
| 9 | AuditService.record() only flushes | §6.4, §14.1: confirmed ✓ |
| 10 | was_attempted semantics precise | §13.6: three-state table defined precisely ✓ |
| 11 | NOT_PROCESSED meaning | §13.6: "no committed registration found" ✓ |
| 12 | Counters recalculated from child table during recovery | §13.5: after-loop and reconciliation both use full GROUP BY ✓ |
| 13 | validated_rows scrubbed at correct terminal point | §10.6 step 8: validated_rows='\[\]' in terminal UPDATE atomically ✓ |
| 14 | CSV validation, preview, confirmation, expiry unchanged | §17, §18 ✓ |
| 15 | Email at-most-once via BackgroundTasks after DB commit | §20: schedule_email → background_tasks.add_task(send_email_safely, ...) ✓ |
| 16 | No durable background queue | §20: confirmed; no Redis/Celery/RQ ✓ |

## Appendix A: Resolved Issues (v9 Corrections)

### A1. CRITICAL: Consistent parent-first lock order

**Problem (v8):** T1 inserted the child row before updating the parent; T3 updated the child row before updating the parent. After-loop and reconciliation locked the parent first. This asymmetry created a deadlock: T3 holding child lock + reconciliation holding parent lock + each waiting for the other.

**v9 resolution:** T1 (§5) and T3 (§7) both begin with `SELECT ... FOR UPDATE` on the parent session `WHERE status='PROCESSING'` before any child mutation. All five paths now use parent-first ordering. Deadlock is structurally impossible.

### A2. HIGH: Student FK target corrected

**Problem (v8):** `bulk_upload_row_results.student_id` referenced `users(id)`. A user is not necessarily a student.

**v9 resolution (§3.2):** References `students(id) ON DELETE RESTRICT`. `students.id` is a shared PK/FK with `users.id`. Referencing `students(id)` guarantees the UUID is an actual student account.

### A3. HIGH: Audit index validation against validated_rows

**Problem (v8):** Reconciliation only checked that the audit row index was a positive integer. An index of 999 in a 20-row session would pass.

**v9 resolution (§10.3 step 4):** Multiple validation checks per audit record, including `INDEX_NOT_IN_VALIDATED_ROWS`: the index must be present in `valid_index_set`, derived from the `validated_rows` JSONB array stored at upload time.

### A4. MEDIUM: Complete child-row consistency matrix

**Problem (v8):** Reconciliation decision table only checked child status vs. audit presence. Field-level validation was absent.

**v9 resolution (§12):** Full matrix covering all combinations of (status × student_id presence × audit presence × entity_id match). Every impossible combination is a CONTRADICTION.

### A5. MEDIUM: Counter consistency formalized

**Problem (v8):** `uncertain_count` conflated STARTED and OUTCOME_UNCERTAIN. T3 counter updates and after-loop recalculation could race.

**v9 resolution (§13):** `uncertain_count` tracks ONLY OUTCOME_UNCERTAIN rows. T3 increments are within the parent exclusive lock. After-loop and reconciliation both use full GROUP BY recalculation.

### A6. Preservation of v8 improvements

All 16 v8 revalidation points explicitly confirmed in §24. No regressions.

## Appendix B: v10 Corrections Applied

### B1. Audit JSONB extraction fix

**Problem (v9):** The audit query used `->>` (text extraction) on `bulk_upload_valid_row_index`, making it impossible to distinguish JSON integer 3 from JSON string "3". Additionally, v9 claimed to detect SESSION_ID_MISSING and SESSION_ID_MISMATCH contradictions, but the WHERE filter excludes those records entirely.

**v10 fix:** Use `->` (native JSONB extraction) on `bulk_upload_valid_row_index`, preserving Python type information. Python validation checks `isinstance(raw, bool)` before `isinstance(raw, int)` (bool is a subclass of int in Python). SESSION_ID_MISSING and SESSION_ID_MISMATCH checks are removed and their undetectability is explicitly documented.

### B2. Structural validation of validated_rows

**Problem (v9):** Reconciliation attempted to iterate `validated_rows` without first verifying it was a well-formed list of the expected size with correctly-typed elements.

**v10 fix:** Full structural validation of `validated_rows` before building `valid_index_set`: type check (must be list), length check (must equal `valid_count`), per-element type check (must be dict), `valid_row_index` type and range check (bool-before-int), duplicate check, and final set equality check. Any failure → ROLLBACK + ReconciliationError.

### B3. Extended consistency matrix

**Problem (v9):** The §12.2 table covered only status, student_id, audit presence, and entity_id. The `was_attempted` and error fields were not formally specified.

**v10 fix:** §12.3 added with explicit per-status requirements for `was_attempted`, `error_code`, and `is_retryable`. STARTED with `was_attempted ≠ NULL` is now a formal CONTRADICTION.

### B4. csv.field_size_limit at startup

**Problem (v9):** The save/set/restore pattern was a process-wide data race under concurrent requests.

**v10 fix:** `csv.field_size_limit(8192)` set once at application startup. Per-request save/restore removed.

### B5. T1/T3 SELECT uses one_or_none()

**Problem (v9):** Used `rowcount` on `SELECT ... FOR UPDATE`. SQLAlchemy 2.0 returns -1 for `rowcount` on SELECT statements — the check was unreliable.

**v10 fix:** Use `.one_or_none()`: returns the row if found (locked), or None if not found (FENCED). Explicitly documented in SQL comments.

### B6. T3 ALREADY_WRITTEN checks student_id

**Problem (v9):** T3's idempotency check compared only the existing status to the intended status. For SUCCEEDED, two different students committed to the same row would appear idempotent.

**v10 fix:** For SUCCEEDED, ALREADY_WRITTEN requires both `status = SUCCEEDED` AND `student_id = :student_id`. A student_id mismatch → INCONSISTENT.

### B7. preview_invalid_results structure defined

**Problem (v9):** The `preview_invalid_results` JSONB structure was undefined. No explicit statement about PII was made.

**v10 fix:** Explicit structure defined in §16.1: `csv_record_number`, `errors[]` with `field`, `error_code`, `message`. No PII fields. Error messages describe the rule, never echo the submitted value. Indefinite retention is acceptable because this structure contains no personal data.

### B8. Redundant index removed

**Problem (v9):** Included `CREATE INDEX ix_row_results_session ON bulk_upload_row_results (upload_session_id)`. The UNIQUE constraint on `(upload_session_id, valid_row_index)` already creates a B-tree index with `upload_session_id` as its leading column, making the separate index redundant.

**v10 fix:** Index removed from §3.2 DDL. Comment added explaining the reasoning.

## Appendix C: Decisions Applied (D1–D9)

| Decision | Question | Resolution |
| --- | --- | --- |
| D1 | Stale heartbeat threshold for reconciliation eligibility | **5 minutes** — `processing_heartbeat_at < now() - INTERVAL '5 minutes'` |
| D2 | Lock timeout for T1 and T3 | **5s** — `SET LOCAL lock_timeout = '5s'` |
| D3 | Lock timeout for reconciliation claim | **10s** — `SET LOCAL lock_timeout = '10s'` |
| D4 | BulkImportOrigin mutability | **@dataclass(frozen=True)** — immutable, hashable |
| D5 | BulkImportSessionFencedError inheritance | **Extends StudentServiceError directly** — not StudentConflictError; fencing is a session lifecycle condition |
| D6 | reconciliation_claim_token in terminal UPDATE | **Included** — AND reconciliation_claim_token = :my_token; defense-in-depth |
| D7 | Partial audit index for reconciliation | **DEFERRED** — excluded from M1 migration; can be added later if query performance justifies it |
| D8 | Future purge retention periods | **DEFERRED** — no purge mechanism in Phase 5; retention is indefinite |
| D9 | valid_row_index required in validated_rows elements | **APPROVED** — required integer in every element; enforced at upload time and validated at reconciliation time |

## Appendix D: Remaining Limitations

1. **SESSION_ID_MISSING / SESSION_ID_MISMATCH undetectable (B1):** The reconciliation audit query filters by `bulk_upload_session_id`. Records with a missing or mismatched session_id are excluded before Python sees them. These anomalies cannot be detected. The fail-closed guarantee covers all anomalies that can be observed.
2. **No partial audit index in M1 (D7 deferred):** Reconciliation uses a full table scan on `action = 'student.registered'` filtered by existing `ix_audit_logs_action`. With a large audit table, reconciliation may be slow. This is acceptable for Phase 5 scale.
3. **No purge mechanism (D8 deferred):** PII in child rows (`first_name`, `last_name`, `roll_number`) is retained indefinitely. `validated_rows` PII is scrubbed at terminal write. This is accepted as temporary.
4. **Email is at-most-once:** Students whose onboarding email was not sent (email failure, crash before send) must be resent via the `student.onboarding_email.resent` endpoint.
5. **Synchronous import only:** Up to 100 rows in a single HTTP request. No background processing, no progress streaming.

## Appendix E: Implementation Readiness

READY FOR M1 DESIGN APPROVAL

All 9 decisions (D1–D9) have been resolved. All 8 technical corrections (B1–B8) have been applied. No unresolved design decisions block implementation.

### Milestone gates

| Milestone | Required decisions | Status |
| --- | --- | --- |
| M1 (Schema, Upload, Preview) | D2 ✓, D3 ✓, D9 ✓; D7 excluded | ALL CLEAR |
| M2 (Confirm, Import Loop) | D1 ✓, D4 ✓, D5 ✓ | ALL CLEAR |
| M3 (Reconciliation, Results, Report) | D6 ✓ | ALL CLEAR |

### No previously approved decisions were changed

- Phase 4 `register_student()` transaction ownership: unchanged
- `AuditService.record()` flush-only behavior: unchanged
- PostgreSQL READ COMMITTED isolation: unchanged
- Parent-first locking: unchanged (confirmed from v9)
- Main Admin-only `student.bulk_import` permission: unchanged
- Synchronous processing, no Redis/Celery/RQ: unchanged
- Partial success, five row outcomes: unchanged
- Email at-most-once behavior: unchanged

### Corrected issues in this version

- B1–B8 applied (documented in Appendix B)
- D1–D9 all resolved (documented in Appendix C)
- Milestone gates corrected: D1 is a M2 gate, not M1
- SESSION_ID undetectability documented rather than falsely claimed as detectable

Phase 5 Technical Design v10 — 2026-11-24 — Supersedes v9