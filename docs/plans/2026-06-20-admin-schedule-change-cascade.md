# Admin Schedule Management + Schedule-Change Cascade

**Date:** 2026-06-20
**Status:** Planned (approved design; not yet implemented)
**Repos touched:** `asima-backend` (cascade engine + endpoints), `asima-frontend` (admin schedule page)

## Overview

Build the admin **Schedule** page where an admin can change or remove **any**
employee's work schedule, and make those changes **safe** with respect to the
two entities that reference schedules — **time-correction requests** and
**leave requests**. When an admin modifies or removes a schedule, dependent
requests that the change invalidates are auto-cancelled according to a precise
matrix, the admin sees the blast radius before committing, and leave balances
self-correct.

This document is both the **formalized business logic** (the "what" and the
"why") and the **implementation plan** (the task breakdown). It is the audit
snapshot; the live working copy is `asima-parent/tasks/plan.md`.

---

## Part A — Formalized business logic

### A.1 The mental model (why the rest follows)

A `work_schedules` row is **not** a per-date record. It is a **versioned weekly
template**:

> "Employee `E`, on weekday `W`, is expected to work `expected_in..expected_out`
> with `break_minutes` (starting `break_start`), valid from `effective_from`
> until `effective_to` (NULL = still active)."

A partial unique index enforces **at most one active row per `(employee,
weekday)`**. Time-corrections (a single `work_date`) and leave requests
(`start_date..end_date`) reference **calendar dates**. A request is *connected*
to a schedule row when its date(s) land on that row's `weekday` and within its
effective range.

Everything below is a consequence of keeping that template **versioned and
forward-only**.

### A.2 Edit model — versioned, forward-only (Decision 1)

Every change to an **existing** schedule is forward-only. The admin chooses an
effective date `X` with `X >= today`:

- **Modify** weekday `W` effective `X`: stamp `effective_to = X − 1` on the live
  row, then insert a new row with `effective_from = X`, new window/break,
  `effective_to = NULL`.
- **Remove** weekday `W` effective `X`: stamp `effective_to = X − 1` on the live
  row; insert **no** replacement.

> **Same-day edge (C1):** if the live row's own `effective_from >= X` (e.g. a
> schedule created today and edited today with `X = today`), stamping
> `effective_to = X − 1` would make `effective_to < effective_from` and violate
> the `effective_to >= effective_from` CHECK. In that case the live row never
> took effect, so **replace it**: soft-delete the un-started row and insert the
> new row with the same `effective_from` (modify), or just soft-delete it
> (remove). The versioning step branches on `old.effective_from >= X`.

The past is **never** rewritten — this preserves historical DTRs and payroll,
the documented `work_schedules` invariant (see `asima-backend/CLAUDE.md` →
"Work schedules — at most one ACTIVE row per (employee, weekday)").

**Why this matters for the cascade:** every *governed date* of a change is
`>= X >= today`. Therefore a **past** request can never be "affected" — "the
past is never touched" is not a special case, it falls out of the model for
free.

> The existing low-level `POST/PATCH/DELETE /admin/work-schedules` endpoints
> remain for creating a brand-new schedule (no prior active row → no cascade
> possible). The admin UI routes **all edits/removals of an existing row**
> through the new cascade-aware "changes" flow (Part B). The destructive
> in-place `PATCH` of a live window is a footgun and is not used by the UI.

### A.3 "Affected" predicate — only if the window actually changes (Decision 3)

Define a change's **governed dates** = all calendar dates `D` where `D >= X`,
`weekday(D) == W`, and `D` was previously governed by the edited row.

A request is **affected** iff it has ≥1 governed date **and** the change
actually alters what that request depends on:

| Request | Affected when… |
|---|---|
| **Time-correction** (`work_date`) | `expected_in` / `expected_out` changed, **or** schedule removed. (A break-only change does not affect a correction — break is not part of punch-in/out correctness.) |
| **Leave — half-day** (`first_half` / `second_half`, single day) | window **or** `break_start` / `break_minutes` changed, **or** removed. Its `start_time` / `end_time` were **snapshotted from the schedule** at submit time, so a window/break change makes the snapshot stale. |
| **Leave — full-day** | schedule **removed only** (the day stops being a working day → `working_days` snapshot is wrong). A pure in/out time shift does **not** affect a full-day leave — the whole day is off regardless of the clock window. |

**Why precision:** the alternative ("any change to the same schedule lineage")
would cancel a Monday correction because Tuesday's break moved. Least surprise,
least collateral.

### A.4 Cancellation matrix (Decision 2)

Let `today` = the server's single-timezone current date. For an *affected*
request, its **temporal class** is:

- **PRESENT / in-progress** — its date range includes today, or `work_date == today`.
- **FUTURE** — its date range is entirely after today.
- (**PAST** cannot occur for an affected request — see A.2.)

Decision, applied **only to affected requests**:

| | Affected + PENDING (`pending_l1`/`pending_l2`) | Affected + APPROVED |
|---|---|---|
| **FUTURE** | **cancel** | **cancel** |
| **PRESENT / in-progress** | **cancel** | **KEEP** |

**No partial cancellation.** A multi-day approved leave that is in-progress is
protected as a whole; a wholly-future one is cancelled as a whole.

> **Accepted inconsistencies (I3).** A *protected* (KEEP) request keeps its
> **submit-time snapshot** — `working_days`, and a half-day's `start_time`/
> `end_time` — even though the live schedule changed underneath it. Because
> balance derives from the snapshot, an employee mid-leave keeps the day they
> "spent" even if that day is no longer scheduled. This is deliberate: a
> forward-only change must not disrupt an in-flight request. The cascade makes
> *future* requests consistent, not in-flight ones.

> **Collateral on multi-day leave (I2).** With no partial cancellation,
> removing or changing a *single* weekday cancels an entire multi-day leave that
> merely touches it (e.g. removing Wednesday cancels a full-day Mon–Fri leave).
> The preview MUST therefore show, per affected leave, **which governed date(s)
> triggered the cancel** (`AffectedRequest.trigger_dates`) so the admin
> understands why a week-long leave is in the list.

**Why:**
- **Pending always cancels** — nothing is committed; it is cheaper and cleaner
  to force a resubmit against the new schedule than to let an approver later
  approve a request built on a stale template.
- **Approved + future cancels** — the request has not started; the schedule
  change is forward-looking and the request no longer reflects reality.
- **Approved + in-progress is protected** — the employee has already organized
  around an in-flight leave (or a correction for today). Yanking it mid-stream
  is disruptive, and a forward-only schedule change is about the future anyway.

### A.5 Balance handling — automatic, no ledger writes (Decision 4, refined)

Leave balance is **derived, not stored**:
`balance = allowance(ledger) − used(approved working-days) − reserved(pending
working-days)` (`LeaveBalanceService`). The code comment is explicit: flipping a
request to `cancelled` "frees the days."

Therefore the chosen "refund to the ledger" intent is satisfied **for free**:
cancelling an approved future leave stops it counting as `used`; cancelling a
pending leave stops it counting as `reserved`. **No compensating allocation
entry is written** — that would double-count. Time-corrections carry no balance.

### A.6 Authorization & audit (Decision 6)

The cascade is a **side effect of a schedule write**, so it is gated by the
schedule permission the admin already needs — **not** by leave/correction
permissions:

- **Modify** → `SCHEDULE:Update`. **Remove** → `SCHEDULE:Delete`.
- Each auto-cancelled request is stamped `status = cancelled`,
  `cancelled_by = admin.id`, `cancelled_at = now`,
  `decision_note = "auto-cancelled: schedule changed (weekday W, effective X)"`.

This requires a **system-cancel** service path that skips the per-user
ownership check the normal `cancel()` enforces — justified by the upstream
`SCHEDULE:*` gate. It does **not** couple the two modules' permission sets (an
admin who can fix a schedule is never blocked because they lack `LEAVE:Update`).

> **Deliberate cross-module read (S2).** Because preview is gated by
> `SCHEDULE:*` alone, a schedule admin can see an employee's leave/correction
> dates via the impact report without holding `LEAVE:View`. This is an accepted
> consequence of treating the cascade as a schedule-write side effect — the same
> admin is already authorized to reshape that employee's schedule.

> **Conditional gating (C2).** A single endpoint cannot express "needs
> `SCHEDULE:Delete` only when `mode='remove'`" via the static `@Permissions`
> decorator. The controller is gated `SCHEDULE:Update`; when `mode='remove'`
> the service performs an explicit runtime check that the caller holds
> `SCHEDULE:Delete` (or `system_admin`) and throws `ForbiddenException`
> otherwise.

### A.7 Two-phase API — preview then confirm (Decision 5)

- `POST /api/v1/admin/work-schedules/changes/preview` — **read-only**, writes
  nothing. Returns a `ScheduleChangeImpact`: the row that will be ended, the row
  that will be created (modify), and the lists of leaves & corrections that
  **will** cancel (employee, dates, status, temporal class), plus the leave-days
  that will free up.
- `POST /api/v1/admin/work-schedules/changes` — **commits atomically** in one DB
  transaction: end old row → insert new row (modify) → system-cancel each
  affected request. It **re-computes the affected set inside the transaction**;
  if it differs from the previewed snapshot the caller submits (a request
  slipped in or out concurrently), it returns **409** so the admin re-previews.
  Returns a `ScheduleChangeResult` with what was actually cancelled.

**Why preview-first:** one click cascading into N cancellations across two
modules is exactly the blast radius an admin should see before it happens.

---

## Part B — Implementation plan

### Architecture decisions

- **Cascade engine lives in `work-schedules`.** It is a schedule-write concern.
  New `ScheduleChangeService` orchestrates; the affected/decision logic is a
  **pure domain policy** (`cascade-policy.ts`) so the matrix is unit-tested
  without a DB.
- **Cross-module access via the abstract ports.** `ScheduleChangeService`
  depends on `BaseLeaveRequestRepository` and `BaseTimeCorrectionRequestRepository`
  (new finder + system-cancel methods), never on concrete repos.
- **Atomicity via a persistence-layer unit of work (I1).** `apply` runs all
  three modules' writes in one transaction. To keep `typeorm` out of the service
  layer (per `asima-backend/CLAUDE.md`), the transaction is owned by a thin
  persistence-layer `runInTransaction(work)` port (backed by
  `DataSource.transaction`); the repository write methods used inside accept an
  injected `EntityManager`, but the **service never imports `EntityManager`** —
  it passes a callback to the port. This is the main architectural risk (see
  Risks).
- **One-way module dependency (S4).** `work-schedules.module` imports the leave
  + time-correction persistence modules; the dependency only ever points
  schedule → leave/TC, never the reverse. Do not let a later change introduce a
  cycle.
- **Weekday-in-range matching in memory.** Coarse DB query (employee + active
  status + date window) then compute weekday membership in JS — volumes are
  per-employee and tiny; SQL weekday-in-range is not worth the complexity.
- **Frontend:** feature-sliced + shadcn per `frontend-architecture.md`;
  data-access with `keys.ts`; page gated on `SCHEDULE:*` from
  `/users/me/permissions` (no client-side `role.permissions` parsing).

### Dependency graph

```
cascade-policy (pure)            ← Task 1
   │
   ├── leave-requests: finder + systemCancel        ← Task 2
   ├── time-corrections: finder + systemCancel       ← Task 3
   │        │
   │        └── ScheduleChangeService.preview         ← Task 4
   │                 └── preview endpoint + DTO        ← Task 5   [CHECKPOINT 1]
   │                          │
   │                          └── txn plumbing + apply ← Task 6
   │                                   └── apply endpoint + e2e ← Task 7  [CHECKPOINT 2]
   │
frontend data-access + read page  ← Task 8
   └── edit/remove + preview→confirm flow             ← Task 9   [CHECKPOINT 3]
```

### Phase 1 — Cascade brain (pure domain)

#### Task 1: Schedule-change domain types + cascade policy (pure)
**Description:** Add the domain types and the pure decision logic for the
cascade, with no NestJS/TypeORM imports. This is the highest-risk logic and is
tested in isolation first.

**Acceptance criteria:**
- [ ] `work-schedules/domain/schedule-change.ts` defines `ScheduleChangeIntent`
  (`employee_id`, `day_of_week`, `effective_from`, `mode: 'modify'|'remove'`,
  optional window/break), `AffectedRequest` (kind, id, employee_id, dates,
  `trigger_dates` (I2 — the governed date(s) that caused the cancel),
  status, temporal class, decision), and `ScheduleChangeImpact`
  (`ended_row`, `created_row?`, `affected_leaves[]`, `affected_corrections[]`,
  `freed_leave_days`).
- [ ] `work-schedules/domain/cascade-policy.ts` exports pure functions:
  `windowChanged(old, intent)`, `isLeaveAffected(leave, intent, …)`,
  `isCorrectionAffected(tc, intent)`, and `classify(temporal, status) →
  'cancel' | 'keep'` implementing the A.4 matrix.
- [ ] Cancellation reason/`decision_note` template constant added to
  `work-schedules.constants.ts`.

**Verification:**
- [ ] Unit spec covers every matrix cell: pending-future, approved-future,
  approved-in-progress (keep), pending-in-progress, half-day vs full-day vs
  removal, break-only change (correction not affected; half-day affected).
- [ ] **C1 same-day case:** versioning helper returns "replace" (not "end at
  X−1") when `old.effective_from >= X`.
- [ ] `npm test -- cascade-policy` green; no `@nestjs/*`/`typeorm` import in
  either domain file.

**Dependencies:** None. **Scope:** M (3 files + spec).

### Phase 2 — Cross-module read + system-cancel

#### Task 2: Leave-requests — candidate finder + system-cancel path
**Description:** Add a repository finder for active leave candidates and a
service/repository system-cancel that bypasses the ownership check and stamps
the schedule-change note. Both write methods accept an optional `EntityManager`.

**Acceptance criteria:**
- [ ] `BaseLeaveRequestRepository.findActiveCandidatesForScheduleChange(
  employee_id, from_date)` returns non-terminal (`pending_l1`/`pending_l2`/
  `approved`) leaves whose `end_date >= from_date`.
- [ ] `systemCancel(ids, actor_id, note, manager?)` flips status to `cancelled`
  with `cancelled_by`/`cancelled_at`/`decision_note`, honoring an injected
  `EntityManager`.
- [ ] `LeaveRequestsService.systemCancelForScheduleChange(...)` wraps it without
  the per-user `ForbiddenException` ownership guard.

**Verification:**
- [ ] Service spec: cancels target ids, leaves others untouched, sets the note.
- [ ] `npm test -- leave-requests` green.

**Dependencies:** Task 1. **Scope:** M.

#### Task 3: Time-correction-requests — candidate finder + system-cancel path
**Description:** Mirror Task 2 for corrections (single `work_date`).

**Acceptance criteria:**
- [ ] `BaseTimeCorrectionRequestRepository.findActiveCandidatesForScheduleChange(
  employee_id, from_date)` returns non-terminal corrections with
  `work_date >= from_date`.
- [ ] `systemCancel(ids, actor_id, note, manager?)` + service
  `systemCancelForScheduleChange(...)` parallel to leave.

**Verification:**
- [ ] Service spec parallels Task 2. `npm test -- time-correction` green.

**Dependencies:** Task 1. **Scope:** M.

### Phase 3 — Preview (read-only vertical slice)

#### Task 4: `ScheduleChangeService.preview()`
**Description:** Resolve the current active row for `(employee, weekday)`, pull
candidates from both modules, run `cascade-policy` over them, and assemble a
`ScheduleChangeImpact`. No writes.

**Acceptance criteria:**
- [ ] Loads the live row; if none and `mode='modify'`, returns an empty-impact
  result flagged as "create, no cascade".
- [ ] Filters candidates to governed dates (weekday `W`, `>= effective_from`) in
  memory, applies the affected predicate per type, classifies temporal+status,
  and includes only `decision === 'cancel'` requests in the impact lists.
- [ ] Each `AffectedRequest` carries `trigger_dates` (I2) — the governed
  date(s) that caused its cancel — so the UI can explain a multi-day leave.
- [ ] Computes `freed_leave_days` from cancelled approved/pending leaves.

**Verification:**
- [ ] Service spec with mocked ports covers: removal cancels full-day future
  leave; window shift cancels half-day but not full-day; in-progress approved
  leave excluded; unaffected weekday excluded.
- [ ] `npm test -- schedule-change` green.

**Dependencies:** Tasks 1–3. **Scope:** M.

#### Task 5: Preview endpoint + DTO
**Description:** Expose `POST /admin/work-schedules/changes/preview`.

**Acceptance criteria:**
- [ ] `ScheduleChangePreviewDto` validates the intent; `mode='remove'` forbids
  window fields, `mode='modify'` requires them (reuse the window/break
  validation already in `WorkSchedulesService`).
- [ ] Controller gated `@Permissions({ SCHEDULE: 'Update' })`; when
  `mode='remove'`, the **service** does a runtime check that the caller holds
  `SCHEDULE:Delete` or `system_admin` and throws `ForbiddenException` otherwise
  (C2 — static decorators can't gate on the body). Swagger
  `@ApiTags('Admin - Work Schedules')`, `@ApiOperation`, `@ApiResponse`. Returns
  `ScheduleChangeImpact`, writes nothing.

**Verification:**
- [ ] Controller/e2e: preview returns the affected lists and persists nothing
  (DB unchanged afterward). `npm run test:e2e` for the new spec green.

**Dependencies:** Task 4. **Scope:** S–M.

#### Checkpoint 1 — Preview slice
- [ ] `npm run lint && npm test` clean in `asima-backend`.
- [ ] Preview endpoint returns a correct impact for a hand-built scenario; DB
  untouched. Review before building apply.

### Phase 4 — Apply (transactional vertical slice)

#### Task 6: Transaction plumbing + `ScheduleChangeService.apply()`
**Description:** Make the schedule write + both modules' cancels run in one
`DataSource.transaction`. Re-compute the affected set inside the transaction;
compare against the caller's previewed id snapshot and abort with 409 on drift.

**Acceptance criteria:**
- [ ] A persistence-layer `runInTransaction(work)` port wraps
  `DataSource.transaction`; the service passes a callback and never imports
  `EntityManager` (I1). Work-schedule `endLogically`/`create` and both
  `systemCancel` methods accept an injected `EntityManager` and use it.
- [ ] `apply()` opens the transaction: **version the row** — branch on the C1
  same-day case (end at X−1 + create, or replace) — then cancel the
  freshly-recomputed affected set → return `ScheduleChangeResult`.
- [ ] If the recomputed cancel-set drifts from the caller's previewed snapshot,
  throw `ConflictException` (409) and roll back. Compare on
  `(id, status)` per request, not ids alone (S1), so a status flip between
  preview and apply is caught.

**Verification:**
- [ ] Service spec: a thrown error inside the txn rolls back the schedule change
  (no orphaned ended row); drift triggers 409.
- [ ] `npm test -- schedule-change` green.

**Dependencies:** Task 5. **Scope:** M (touches 3 repos + service).

#### Task 7: Apply endpoint + end-to-end matrix
**Description:** Expose `POST /admin/work-schedules/changes` and prove the whole
matrix end-to-end against a real DB.

**Acceptance criteria:**
- [ ] `ScheduleChangeApplyDto` = intent + previewed `(id, status)` snapshot.
  Same permission gating as preview (controller `SCHEDULE:Update` + runtime
  `Delete` check on remove). Returns `ScheduleChangeResult`.
- [ ] On success the old row is ended, the new row exists (modify), and exactly
  the affected requests are `cancelled` with the system note.

**Verification:**
- [ ] e2e seeds: pending-future leave → cancelled; approved-future leave →
  cancelled and the employee's derived balance increases by its working-days;
  approved-in-progress leave → still `approved`; correction on an unaffected
  weekday → untouched; removal → full-day leave cancelled.
- [ ] `npm run test:e2e` green (run with the CI seed password — see project
  memory on `SEED_DEFAULT_PASSWORD`).

**Dependencies:** Task 6. **Scope:** M.

#### Checkpoint 2 — Backend complete
- [ ] `npm run lint && npm test && npm run test:e2e && npm run build` clean.
- [ ] Matrix verified end-to-end. Commit + push `asima-backend` to `main`.

### Phase 5 — Frontend admin Schedule page

#### Task 8: Data-access + read-only schedule view
**Description:** Admin Schedule route: employee picker + weekly grid of the 7
active rows for the selected employee.

**Acceptance criteria:**
- [ ] Data-access slice with `keys.ts` and a query hook over
  `GET /admin/work-schedules?employee_id=…`; types are snake_case end-to-end.
- [ ] Searchable employee picker → weekly grid showing window + break per
  weekday. Page gated on `SCHEDULE:View`/`Update` from `/users/me/permissions`.

**Verification:**
- [ ] `npm run lint && npm run build` (frontend) clean; grid renders a seeded
  employee's schedule; non-permitted user does not see the page.

**Dependencies:** Task 7 (contract stable). **Scope:** M.

#### Task 9: Edit / remove + preview→confirm flow
**Description:** The cascade UX: edit or remove a weekday, see the impact, confirm.

**Acceptance criteria:**
- [ ] Editor for a weekday: new window/break + `effective_from` (default today,
  min today); plus a Remove action.
- [ ] Save → calls `changes/preview` → impact dialog lists the leaves &
  corrections that will cancel and the days freed → Confirm → calls `changes`
  with the previewed ids → success toast and grid refresh.
- [ ] `409` from apply → toast "schedule changed since preview" and re-run
  preview inline.

**Verification:**
- [ ] Manual: modify a weekday with a future approved leave → dialog lists it →
  confirm → leave shows cancelled and the change persisted; a `409` path is
  reachable by racing a second tab.
- [ ] `npm run lint && npm run build` clean.

**Dependencies:** Task 8. **Scope:** M.

#### Checkpoint 3 — Complete
- [ ] All acceptance criteria met across both repos; lint/test/build/e2e green.
- [ ] Commit + push each repo to its own `main`.

## Risks and mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Cross-module transaction across hexagonal ports | High | Persistence-layer `runInTransaction` port keeps `typeorm` out of the service (I1); inject `EntityManager` into write methods; spec proves rollback leaves no orphaned ended row. |
| Same-day schedule edit violates `effective_to >= effective_from` CHECK (C1) | High | Versioning helper branches on `old.effective_from >= X` → replace the un-started row instead of ending it at X−1; unit + e2e cover it. |
| Conditional `SCHEDULE:Delete` gating not expressible in `@Permissions` (C2) | Med | Controller gated `Update`; service runtime-checks `Delete`/`system_admin` when `mode='remove'`. |
| Preview/apply race (request slips in/out) | Med | `apply` recomputes inside the txn and 409s on `(id, status)` drift vs the submitted snapshot (S1). |
| One-day change cancels a whole multi-day leave (I2) | Low | Preview surfaces `trigger_dates` per affected leave so the admin sees why. |
| `today` / timezone ambiguity in temporal class | Med | Single server timezone (use the existing date helper); document the boundary; e2e pins "today". |
| Full-day-leave rule surprises users | Low | Documented in A.3 and surfaced in the preview list; pure time shift never cancels a full-day leave. |
| Destructive in-place `PATCH` bypasses cascade | Low | UI never calls it; A.2 notes it as a footgun; hardening deferred (out of scope). |

## Open questions

- None blocking. The destructive `PATCH` hardening and any notification to
  affected employees on auto-cancel are explicitly out of scope for this plan.
