# Implementation Plan: DDD Revamp — `time-entries`

> Snapshot frozen 2026-06-27. Working copy lives in
> `asima-backend/tasks/plan.md`; checklist in `asima-backend/tasks/todo.md`.
> Backend-only feature; this parent-repo copy is the audit trail.

## Overview

Bring `src/time-entries/` up to the Domain-Driven Design blueprint
(`docs/universal-guidelines/module-architecture.md`), mirroring the
`leave-requests` / `time-correction-requests` revamps. Today the module is
hexagonal-with-an-anemic-domain: the `TimeEntry` domain class **carries
`@ApiProperty`** (blueprint §12 anti-pattern) and the lifecycle rules live in
`TimeEntriesService`. Two rules in particular are **duplicated three times**
(in `create`, `update`, and `applyCorrection`):

- `time_out` must be **strictly after** `time_in`;
- `status` is **derived** from the window (`time_out == null → open`, else
  `confirmed`).

After the revamp those rules live on a `TimeWindow` value object + a pure
`TimeEntry` aggregate; the domain is framework-free; the open→confirmed
transitions raise **domain events** that establish the side-effect seam.

`time-entries` is a **full state-machine aggregate** — `punch` (close) and
`applyCorrection` (target) are load-mutate-save — so it follows the §3.1
template: `extends AggregateRoot`, private constructor + static `reconstitute`,
behavior methods that `recordEvent(...)`. (Admin `update` stays a thin use-case
patch — decision #5 — and reconstitution happens **use-case-side from the
record each mutate path already holds**, so there is no `findAggregateById` —
decision #10.)

**Why this is simpler than leave / time-correction (key differences):**

1. **No approval chain, no actor, no approver.** `punch` acts on self (JWT
   identity); admin create/update/delete are gated by the route's
   `@Permissions({ TIME: '…' })`. The aggregate carries **zero** capability
   logic — there is **no `TimeEntryActor`** (decision #4). This is the main
   way the build is lighter than the two approval modules.
2. **`status` is derived, not an independently-gated state machine.** The only
   transition is open → confirmed (close); admin can revise the window. The
   aggregate computes `status` from the `TimeWindow`; there is no role-gated
   advance/approve.
3. **No joined-name read-model.** Unlike `LeaveRequestListItem` /
   `TimeCorrectionRequestListItem`, the list returns plain rows — **no
   `*ListItem`, no `*_name` fields, no list-item response DTO**.

## Scope boundary

The module owns the **`time_entries` table + punch lifecycle**. Surfaces
(unchanged by this revamp), verified in `src/time-entries/controllers/`:

- **Self-service** (`me-time-entries.controller.ts`, mount
  `users/me/time-entries`, JWT-only): `POST punch` (toggle in/out), `GET`
  (list mine), `GET today`.
- **Admin** (`admin-time-entries.controller.ts`, mount `admin/time-entries`,
  `TIME:View|Create|Update|Delete` gated): `GET` list, `GET :id`, `POST`
  create, `PATCH :id`, `DELETE :id` (soft-delete, 204).

**Cross-module consumers that MUST keep compiling + behaving (HARD
requirement — verified by grep):**

- `time-correction-requests/time-correction-requests.service.ts` — the module
  just revamped — calls **`TimeEntriesService.applyCorrection(...)`** (final
  approval write, decision #7 of that plan), **`findById(id)`** (submit
  ownership guard — reads `.employee_id`), and **`hasEntryOnDate(...)`**. These
  three method **signatures + behavior are frozen**. `findById` now returns
  `TimeEntryRecord`, which still exposes `.employee_id`; `applyCorrection`'s
  return is ignored by the caller. **No cross-module code imports the
  `TimeEntry` domain *type*** (grep confirms), so the rename is internal-only.
- `TimeEntrySeedService` (`database/seeds/…`) — uses the repo/service to seed;
  keep its call sites compiling.

**Out of scope:** any schema/migration change (code-only; DB + wire identical);
DTR aggregation / tardiness flagging / day-close (the named future
subscribers — see decision #6); fixing the pre-existing `source` `@ApiProperty`
enum that omits `'correction'` (decision #9 — mirror it verbatim, fix
separately); converting `users`/`roles`/etc.

## Architecture decisions

1. **Split the data bag.** Rename the current `TimeEntry` (domain data class,
   `@ApiProperty`-decorated) → `TimeEntryRecord` and strip its `@ApiProperty`.
   The name `TimeEntry` is reused for the **aggregate root**. **No list-item**
   (the module has no joined read-model). Mirrors `LeaveRequest` /
   `LeaveRequestRecord` (§3.1), minus the list-item.

2. **Full aggregate (load-mutate-save).** `TimeEntry extends AggregateRoot`:
   `private constructor(props)` building the `TimeWindow`, static
   `reconstitute(props)`, behavior methods that mutate in-memory and
   `recordEvent(...)`, drained by the use-case via `pullEvents()`. Reconstitution
   is **use-case-side** from already-held records (decision #10), so the mapper
   keeps only `toDomain` (record) — there is no `toAggregate`/`findAggregateById`.

3. **One value object — `TimeWindow`** (self-validating, throws on invalid
   construction, unit-tested with no DI):
   - holds `time_in: Date` + nullable `time_out: Date | null`;
   - invariant: when `time_out` is present it is **strictly after** `time_in`
     (homes the rule duplicated today in `create` / `update` /
     `applyCorrection`);
   - `isOpen()` (`time_out == null`) — the single source of the open/confirmed
     **status derivation** duplicated today in the same three places.
   - **No `AuditStamp` VO** — the `time-correction` revamp showed an unread
     `AuditStamp` is dead code (it has no invariant and nothing reads it); only
     build value objects that validate **and** are read. **No `EntrySource`
     VO** — `source` is assigned, never re-validated (YAGNI). **No
     `EntryStatus` VO** — status is derived from `TimeWindow.isOpen()`, not an
     independently-stored legal-values field.
   - **Single status-derivation site (S1):** the `open ↔ confirmed` mapping
     lives in exactly one place — a static `TimeEntry.deriveStatus(window:
     TimeWindow): TimeEntryStatus` (`window.isOpen() ? open : confirmed`). The
     aggregate's `status` getter delegates to it, and the use-case **creation**
     paths (which have no aggregate instance) call it too — so the `isOpen ?
     open : confirmed` mapping is never re-duplicated across the three create
     sites the way it is today.

4. **No actor / no authz on the aggregate.** `time-entries` has no chain and no
   approver. `punch` is self (JWT identity from `@CurrentUser`); admin routes
   are gated by `@Permissions({ TIME: '…' })` at the edge. The aggregate never
   imports `User` or `hasPermission` and takes **no actor argument** — a
   deliberate simplification from `LeaveActor` / `CorrectionActor`. (Defer any
   shared approval abstraction — still YAGNI on the 2nd→3rd workflow.)

5. **Behavior on the aggregate** (lifts the duplicated rules):
   - **Static creation guard** (insert-id-bound creation, like leave/TC submit
     — no full factory): `assertWindow(time_in, time_out?, field?)` constructs a
     `TimeWindow` to validate **and returns the validated window** (so callers
     derive status via `deriveStatus(window)` without rebuilding it). The
     `TimeWindow` VO throws `InvalidTimeWindowError` carrying the **default**
     field (`'time_out'`); `assertWindow`'s `field` arg (default `'time_out'`,
     `applyCorrection` passes `'proposed_time_out'`) re-labels the thrown error
     so the use-case maps it to the right 422 key. The error derives its message
     from `field` (`` `${field} must be strictly after ${in-field}` ``), which
     reproduces **both** verbatim strings (decision #8) from one rule-home.
   - `close(at: Date)` — punch-out / close: build the closed `TimeWindow`
     (`at` must be after `time_in`), `status → confirmed`; records
     `TimeEntryClosed`. Load-mutate-save. The use-case then persists the
     **narrow** patch `{ time_out, status, updated_by: actor.id }` — matching
     today's punch-close (`service.ts`) exactly, **not** the full aggregate
     field set (mirrors the `update` narrow-patch discipline below).
     **Behavior delta (S1, accepted):** `close(now)` runs the strictly-after
     invariant on punch-out, which today's `punch` does **not** check explicitly
     (it relies on the DB `CHECK (time_out > time_in)`). With `now` always after
     a past, cooldown-gated punch-in this is unreachable in practice; it only
     moves the clock-skew edge from a sanitized DB error to a domain 422 —
     cleaner, but flagged because the migration is otherwise behavior-preserving.
   - `applyCorrection(window: TimeWindow)` — mutate a **target** entry to the
     proposed window, `source = correction`, status derived; records
     `TimeEntryCorrected`. The use-case then persists the **narrow** patch
     `{ work_date: input.work_date, time_in, time_out, source, status,
     updated_by: decided_by }` — **`work_date` and `updated_by` are added
     use-case-side** (the request supplies the date; today's target update
     moves it — `service.ts` — and a target correction's `work_date` is **not**
     pinned to the target's own date, so dropping it would silently fail to move
     the date and **break the frozen contract**, decision #7). Audit fields
     aren't aggregate state, so the aggregate method takes no `decided_by`. The
     new-row (missed-punch) path stays **use-case creation** (decision #7).
   - Derived `status` getter (delegates to `deriveStatus`) + read accessors the
     use-case needs (below).
   - **Admin `update` is NOT an aggregate transition (I1).** Like leave/TC's
     `update`, it stays a thin **use-case patch**: `findById` (404) → merge the
     patch over the existing row (**`time_out` uses `patch.time_out !== undefined
     ? patch.time_out : existing.time_out`, NOT `??`** — a `time_out: null` patch
     must clear to `open`, exactly as today's `service.ts`; `??` would regress
     the clear-to-open case) → validate the merged window via the static
     `assertWindow` (422) → derive status via `deriveStatus` → `repository
     .update(id, { ...patch, status })`. It records **no event** (an admin edit
     is not a downstream timesheet fact — decision #6). Putting a `revise`
     mutation method on the aggregate with no event to record would be ceremony
     that doesn't earn its keep; the status-derivation rule is centralized in
     `deriveStatus` (decision #3), which is all `update` needs.
   - **What stays in the use-case (I/O the aggregate can't do):** the punch
     **cooldown** (`findLatestForEmployee` → 429), the **one-open-entry** guard
     (`findOpenForEmployee` + the `23505` → 409 mapping; the partial unique
     index stays the race-safe source of truth, blueprint §7), the
     **TOCTOU** existing-entry guard (`existsForEmployeeDate` → 409), and the
     correction **target lookup** (`findById` → 404).
   - **§11 "creation factory" (S2):** creation is insert-id-bound (punch-in /
     admin create / correction new-row), so — exactly as leave/TC submit — there
     is **no full static creation factory**; the aggregate contributes the pure
     `assertWindow` guard + the `TimeEntryOpened` event (published post-commit
     with the real id). This is the sanctioned §3.1 "I/O-bound creation"
     deviation, not a missing checklist item.

6. **Domain events** (`domain/events/time-entry-events.ts`, past-tense, ids +
   scalars, `extends DomainEvent`): **`TimeEntryOpened`** (punch-in / admin
   create / correction new-row), **`TimeEntryClosed`** (punch-out / close),
   **`TimeEntryCorrected`** (target updated by an approved correction).
   Published by the use-case post-commit (creation events with the insert id,
   like leave submit; close/correct via `publisher.publish(pullEvents())`).
   **No subscriber reacts yet** — establishes the seam; behavior unchanged. The
   roadmap names the eventual subscribers (DTR aggregation, tardiness).

   **Decision (was Open Q#1, S3): these three events only — no
   `TimeEntryRevised`.** This set is deliberate, not default ceremony: the
   three are genuine timesheet **facts** the roadmap consumes (a segment opened,
   closed, or was corrected). An admin edit is a back-office correction of an
   existing fact, not a new one, so it emits nothing — and `update` stays
   event-free, which is what lets it remain a thin use-case patch (decision #5).
   If a future feature needs to react to admin edits, add `TimeEntryRevised`
   then.

7. **`applyCorrection` ordering + contract are frozen (cross-module).** This is
   the seam the `time-correction` module's decision-#7 depends on. Preserve the
   exact method signature, the target-update-vs-new-row branch, the
   proposed-window 422, the TOCTOU 409, the one-open 409, and the `23505`
   mapping. Internally the pure window/status derivation moves onto the
   aggregate (`assertWindow` + `applyCorrection`/creation); the I/O guards stay.
   The return value is unchanged (a `TimeEntryRecord`).

8. **Domain errors → identical HTTP codes.** Plain `Error` subclass in
   `domain/time-entry-errors.ts`, mapped by the use-case to the **same
   envelopes the current service emits**:

   | Domain error | HTTP (today) | Helper |
   |---|---|---|
   | `InvalidTimeWindowError` (carries `field`) | 422 | `unprocessable(err.field, …)` |

   A `rethrowTimeEntryDomainError(err)` helper does the mapping (mirrors
   `rethrowCorrectionDomainError`). Everything else — cooldown 429, one-open /
   TOCTOU 409, `23505` 409, target 404 — stays a **direct use-case throw**
   (I/O-bound), unchanged. Preserve message strings verbatim:
   `'time_out must be strictly after time_in'` (create/update) and
   `'proposed_time_out must be strictly after proposed_time_in'`
   (applyCorrection) — note the **field + message differ by call site**, which
   is exactly why `InvalidTimeWindowError` carries `field` (+ message).

9. **Wire compatibility is a hard requirement.** Add
   `dto/response/time-entry-response.dto.ts` mirroring the current `TimeEntry`
   `@ApiProperty` fields **verbatim** (names, order, nullability — **including
   the `source` enum that lists only `['manual','biometric','admin']`**, which
   omits `'correction'`; this is a **pre-existing** Swagger quirk — mirror it
   exactly here and fix it as a separate change). A `TimeEntryAssembler` does
   the structural copy (`toResponse` / `toPaginatedResponse` — **no
   list-item**). Both controllers return DTOs. **JSON byte-identical** —
   `mapper.toDomain`'s assignment order drives key order (keep it == legacy).
   The existing `test/time-entries.e2e-spec.ts` is the parity guard.

10. **Port surface + load-path strategy (I2 — a deliberate divergence from
    leave/TC).** The port keeps every current method (`findAll`, `findById`,
    `findOpenForEmployee`, `findLatestForEmployee`, `existsForEmployeeDate`,
    `create`, `update`, `softDelete`); their `TimeEntry` return types rename →
    `TimeEntryRecord` (reads **and** `create`/`update`). Repos never return
    entities, never throw 404.

    **No `findAggregateById`, and no `mapper.toAggregate`.** Unlike leave/TC —
    whose `approve`/`reject`/`cancel` receive only an `id` and *must* load the
    aggregate — every `time-entries` mutate path **already holds the persisted
    record** for an I/O reason, so a separate aggregate load would be a
    redundant query (and the unused port/mapper methods would be dead code — the
    `AuditStamp` lesson). The mutate paths reconstitute **use-case-side** from
    the record they hold, via the public `TimeEntry.reconstitute(record)`:

    | Mutate path | Record already in hand because… | Then |
    |---|---|---|
    | **punch-close** | `findOpenForEmployee` was called to decide open-vs-create | `reconstitute(open)` → `close(now)` |
    | **applyCorrection (target)** | `findById` was called for the 404 **+ the `deleted_at` check** (the aggregate has no `deleted_at` — decision #3) | `reconstitute(target)` → `applyCorrection(window)` |
    | **admin `update`** | `findById` was called for the 404 | use-case patch (decision #5) — no aggregate, no event |

    So `mapper` provides `toDomain` (→ record) only; reconstitution is the
    aggregate's public static, called from the use-case. The §11 load-path rows
    (`findAggregateById`, `toAggregate`) are **intentionally N/A** here. Note the
    rationale is **distinct from** the leave-allocations §6.1 ledger: this is a
    full §3.1 load-mutate-save aggregate (it `extends AggregateRoot`,
    `reconstitute`s, `pullEvents`), so §6.1's "no mutation path" reason does
    **not** apply. The reason here is the **reconstitute-from-held-record**
    variant — every mutate path already holds the persisted record for an
    independent I/O reason, so a dedicated `findAggregateById`/`toAggregate`
    would be a redundant query and dead code. That variant is now written into
    the blueprint at **§3.2 rule 3a** (reconciling the "reconstitution lives at
    the mapper" rule it qualifies); time-entries is its reference module. Like
    §6.1, the deviation is noted in this plan so it reads as sanctioned, not an
    oversight.

## Dependency graph

```
TimeWindow VO + InvalidTimeWindowError                                  [Task 1]
   └─> TimeEntryRecord + aggregate (close/applyCorrection/deriveStatus) + events [Task 2]
         ├─> mapper (toDomain → Record) + port (retype; NO findAggregateById) [Task 3]
         │      └─> service (punch/create/update/applyCorrection/publish) [Task 4]
         └─> response DTO + assembler                                    [Task 5]
                └─> controllers return DTOs + module (publisher)         [Task 5]
                       └─> e2e parity + verify TC seam + gates           [Task 6]
```

**Build order within ONE commit-set**, not independently-shippable slices
(behavior-preserving migration; the rename breaks intra-module callers at once).
Project `tsc`/build is **red mid-flight, green only at Checkpoint B**; Checkpoint
A is domain-unit-green only (jest compiles per-file via `isolatedModules`).
**Don't commit until Checkpoint C is green.**

## Task List

### Phase 1 — Domain core (the consistency boundary)

#### Task 1: `TimeWindow` value object + domain error
**Acceptance criteria:**
- [ ] `domain/value-objects/time-window.ts` — `time_in: Date`, `time_out: Date |
      null`; throws `InvalidTimeWindowError` when `time_out <= time_in`; exposes
      `time_in` / `time_out` / `isOpen()`. Unit-tested (valid, equal, before,
      null-out/open).
- [ ] `domain/time-entry-errors.ts` — `InvalidTimeWindowError extends Error`,
      carrying a `field: string` (default `'time_out'`) so the use-case maps it
      to the correct 422 key. Message derives from `field`
      (`` `${field} must be strictly after ${in-field}` ``) so the **two**
      verbatim strings (`time_out…`/`proposed_time_out…`) come from one home.
- [ ] Zero `@nestjs/*` / `typeorm` / `class-validator` imports under `domain/`.

**Verification:** `npm test -- time-window` green; framework-import grep clean.
**Dependencies:** None. **Scope:** Small.

#### Task 2: Record split + aggregate + events
**Acceptance criteria:**
- [ ] `domain/time-entry.ts` → `TimeEntryRecord`: pure data, **no
      `@ApiProperty`**, snake_case, definite-assignment, all current fields, same
      field order as `mapper.toDomain`.
- [ ] `domain/time-entry.aggregate.ts` → `TimeEntry extends AggregateRoot`:
      private ctor (builds `TimeWindow`), `reconstitute`, static `assertWindow`
      (**returns the validated window**), static `deriveStatus(window)`, `close`,
      `applyCorrection(window)` (no `decided_by` — audit fields are use-case
      state, not aggregate state), a derived `status` getter (delegates to
      `deriveStatus`), and read accessors (`id`, `employee_id`, `work_date`,
      `time_in`, `time_out`, `source`, `notes`) the use-case needs for the
      persist patch + the `applyCorrection` branch.
      Records `TimeEntryClosed` / `TimeEntryCorrected` on transitions. **No
      `revise`** (admin `update` is a use-case patch — decision #5).
- [ ] `reconstitute` builds the `TimeWindow`, so a corrupt row (`time_out <=
      time_in`) **throws on load** — fail-fast (mirrors leave/TC).
- [ ] `domain/events/time-entry-events.ts` — `TimeEntryOpened(id, employee_id,
      work_date)`, `TimeEntryClosed(id, employee_id)`, `TimeEntryCorrected(id,
      employee_id)`.

**Verification:** `npm test -- time-entry.aggregate` green (reconstitute →
behavior → assert state + `pullEvents()`, no DI/DB): cover close (open→confirmed
+ Closed event), applyCorrection (source=correction + derived status + Corrected
event), `deriveStatus` (open vs confirmed), the corrupt-row throw. Domain folder
framework-free.
**Dependencies:** Task 1. **Scope:** Medium.

### Checkpoint A — Domain core
- [ ] Aggregate + VO unit tests pass with no DI and no DB.
- [ ] `grep` confirms zero framework imports under `domain/`.
- [ ] Review before wiring the slice.

### Phase 2 — Wire the slice

#### Task 3: Persistence layer
**Acceptance criteria:**
- [ ] `mapper.toDomain` → `TimeEntryRecord` (keep field-assignment order for
      JSON parity); `toPersistence` retyped to `Partial<TimeEntryRecord>`
      (behavior unchanged). **No `toAggregate`** — reconstitution is use-case-
      side (decision #10).
- [ ] Port retyped to `TimeEntryRecord` on all reads **and** `create`/`update`;
      `findOpenForEmployee` / `findLatestForEmployee` / `existsForEmployeeDate` /
      `softDelete` unchanged in behavior. **No `findAggregateById`** (decision
      #10). Repo returns records (never entities), never throws 404.
- [ ] **(Optional) enum-source DRY** on the entity `source` / `status` `@Column`
      enums via `Object.values(TIME_ENTRY_SOURCES)` / `Object.values(
      TIME_ENTRY_STATUSES)` — **only if** order is byte-identical to today's
      literals (`source`: manual, biometric, admin, **correction**; `status`:
      open, confirmed). Mirror the time-correction caveat; skip if unsure.
- [ ] At this task, `tsc` may be **red project-wide** until Task 4/5 — confirm
      the persistence files themselves typecheck against the new record/port.

**Dependencies:** Task 2. **Scope:** Medium.

#### Task 4: Application layer (thin use-case)
**Acceptance criteria:**
- [ ] `punch` — `assertNotInCooldown` (429) → `findOpenForEmployee`: **if open**
      → `TimeEntry.reconstitute(open)` (no 2nd query) → `aggregate.close(now)` →
      `update(open.id, { time_out, status, updated_by: actor.id })` (**narrow
      patch — exactly today's three fields**, not the full aggregate) → publish
      `TimeEntryClosed`; **else** → `create` (open, source manual) with the
      `23505` → 409 mapping → publish `TimeEntryOpened(created.id, …)` post-commit.
- [ ] `create` (admin) — one-open guard (422) → `assertWindow` (422, field
      `time_out`) → status via `deriveStatus` → `create` with `23505` → 409 →
      publish `TimeEntryOpened`.
- [ ] `update` (admin) — **use-case patch, no aggregate / no event** (decision
      #5): `findById` (404) → merge patch over existing (**`time_out` via
      `!== undefined`, NOT `??`** — a `null` patch clears to `open`) →
      `assertWindow` (422, field `time_out`) → status via `deriveStatus` →
      `repository.update(id, { ...patch, status })`. Persist the patched fields +
      derived status (matches today's `{ ...patch, status }` exactly — don't
      write back unchanged fields).
- [ ] `applyCorrection` — **frozen contract** (decision #7): proposed-window 422
      (field `proposed_time_out`) → branch: **target** → `findById` 404 **+
      `deleted_at` check on the record** → one-open 409 (when open) →
      `TimeEntry.reconstitute(target)` → `aggregate.applyCorrection(window)` →
      `update(target.id, { work_date: input.work_date, time_in, time_out,
      source, status, updated_by: decided_by })` (**`work_date` + `updated_by`
      added use-case-side** — see decision #5; dropping `work_date` would not
      move the date and break the seam) → publish `TimeEntryCorrected`;
      **new-row** → TOCTOU 409 → one-open 409 → `create` (source correction,
      status via `deriveStatus`) with `23505` → 409 → publish `TimeEntryOpened`.
      Return a record (the caller in `time-correction` ignores it).
- [ ] `softDelete` / `findById` / `findAll` / `hasEntryOnDate` preserved.
- [ ] Domain errors mapped via `rethrowTimeEntryDomainError` to the **same
      codes**; all I/O throws (cooldown/one-open/TOCTOU/23505/404) unchanged.

**Verification:** `npm test -- time-entries.service` green: mocks the port +
publisher; asserts punch toggle (close vs create), the cooldown 429, the
one-open 409 (incl. the `23505` path), the derived status on update,
`applyCorrection` parity (target update + new row + TOCTOU 409 + 422 field), and
events published on transitions.
**Dependencies:** Task 3. **Scope:** Large — the riskiest task (the
`applyCorrection` seam to the just-shipped time-correction module). Review the
write-path (`punch`/`create`) and the `applyCorrection` branch separately.

#### Task 5: Interface layer + module wiring
**Acceptance criteria:**
- [ ] `dto/response/time-entry-response.dto.ts` mirrors the **old** `TimeEntry`
      `@ApiProperty` fields verbatim (names, order, nullability — incl. the
      `source` enum's pre-existing omission of `'correction'`).
- [ ] `time-entry.assembler.ts` (`toResponse` / `toPaginatedResponse` — **no
      list-item**).
- [ ] Both controllers return DTOs via the assembler; preserve **verbatim**
      every `@Permissions({ TIME: … })`, `@ApiTags`, `@ApiBearerAuth`, the
      `version: API_VERSION` mount, the `/me` (punch/today, no `:id`) vs
      `/admin/:id` split, the punch 201/429 responses, and the delete 204.
- [ ] `TimeEntriesModule` provides `DomainEventPublisher`.

**Verification:** `npm run build`; Swagger renders the response DTO; the domain
carries no `@ApiProperty`.
**Dependencies:** Task 4. **Scope:** Medium.

### Checkpoint B — Slice wired
- [ ] `npm run lint && npm run test && npm run build` green.
- [ ] Swagger shows the response DTO; the domain is pure.

### Phase 3 — Integration & wire-parity

#### Task 6: e2e parity + verify the TC seam + gates
**Acceptance criteria:**
- [ ] `test/time-entries.e2e-spec.ts` stays green (byte-parity guard); confirm
      it (or the new unit specs) exercises punch in→out, the cooldown 429, admin
      create/update/delete, and the one-open 409. Add a response key-set
      assertion if absent.
- [ ] **Verify the cross-module seam:** `time-correction-requests` compiles
      against the retyped `findById`/`applyCorrection`/`hasEntryOnDate`, and its
      **e2e stays green** (the L2-approval-writes-the-entry path, decision #7 of
      that module). No change should be needed in the TC module — confirm it.
- [ ] Migrate the existing `time-entries.service.spec.ts` (mock the publisher;
      retype `fakeOpenEntry` to `TimeEntryRecord`; assert aggregate-driven status
      + events on transitions). **No `findAggregateById` to mock** (decision #10
      — mutate paths reconstitute from the record the mocked port already
      returns: `findOpenForEmployee` for punch, `findById` for `applyCorrection`).
- [ ] Full gates: `npm run lint && npm run test && npm run test:e2e && npm run
      build` (e2e with `SEED_DEFAULT_PASSWORD=Asima@1234` + `STORAGE_*` + MinIO).
- [ ] Blueprint §11 checklist satisfied — with the `findAggregateById` /
      `toAggregate` load-path rows **intentionally N/A** (decision #10:
      reconstitution is use-case-side from already-held records), per the
      **§3.2 rule 3a** reconstitute-from-held-record variant (distinct from the
      §6.1 ledger — this module does load-mutate-save).

**Dependencies:** Task 5. **Scope:** Medium.

### Checkpoint C — Complete
- [ ] All acceptance criteria met; §11 checklist green.
- [ ] All gates pass; ready to commit straight to `main` in `asima-backend`.

## Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| **`applyCorrection` contract drifts → breaks the just-shipped time-correction approval (decision #7)** | **High** | Freeze the signature + branch + codes (decision #7); Task 6 re-runs the TC e2e (L2 approval writes the entry). |
| Status-derivation diverges (open vs confirmed) | **High** | Centralize in the single `deriveStatus`; aggregate + service specs assert status after close / applyCorrection / create / update. |
| JSON drift (field order / the `source` enum omission) | Med | `mapper.toDomain` order == legacy; response DTO mirrors `@ApiProperty` verbatim incl. the omission; e2e parity guard. |
| One-open invariant weakened | **High** | Keep the partial unique index as source of truth + the `23505` → 409 mapping in the use-case (blueprint §7); don't move it onto the aggregate. |
| Redundant aggregate load on mutate paths | Low | Decision #10: every mutate path reconstitutes from the record it already holds; no `findAggregateById`, no second query. |
| Over-building (actor / AuditStamp / source VO / findAggregateById / revise) | Low | Decisions #3/#4/#5/#10: only `TimeWindow`; no actor, no AuditStamp, no source/status VO, no `findAggregateById`/`toAggregate`, no `revise` (YAGNI — the time-correction simplification lesson). |
| `update` patch writes back unchanged fields → drift | Low | Decision #5 / Task 4: persist `{ ...patch, status }`, not all aggregate fields — matches today's update exactly. |

## Plan-review pass (2026-06-27)

A five-axis review of this plan (against the live `time-entries` code + the
three revamped exemplars) tightened it before implementation:

- **I1 — `revise` vs. the deferred event.** Dropped the `revise` aggregate
  method; admin `update` stays a thin **use-case patch** (validate via
  `assertWindow`, derive status via `deriveStatus`, persist `{ ...patch, status
  }`), and `TimeEntryRevised` is dropped. A mutation method with no event to
  record didn't earn its keep (decisions #5, #6).
- **I2 — load path.** Surfaced that **no** `time-entries` mutate path needs
  `findAggregateById`: each already holds its record for an I/O reason
  (punch ← `findOpenForEmployee`; `applyCorrection` ← the `deleted_at` check),
  so they reconstitute use-case-side. Dropped `findAggregateById` +
  `mapper.toAggregate` as they'd be dead code; documented the §6.1-style
  sanctioned deviation (decision #10).
- **S1 — one status-derivation site:** added the static `TimeEntry.deriveStatus`
  reused by the aggregate getter and the creation paths (decision #3).
- **S2 — §11 note:** the absent creation factory is the sanctioned I/O-bound-
  creation deviation, not a miss (decision #5).
- **S3 — event set is a deliberate decision** (Opened/Closed/Corrected), not
  default ceremony (decision #6).

### Second plan-review pass (2026-06-27)

A second five-axis pass against the live code closed the gaps below before
implementation:

- **I-A — `applyCorrection` dropped `work_date`.** The aggregate method was
  `applyCorrection(window, decided_by)`, but today's target update persists
  `work_date: input.work_date` (`service.ts`), and a target correction's
  `work_date` is **not** pinned to the target entry's own date (verified: TC
  submit takes `work_date` from input, no `== target.work_date` guard). Building
  the persist patch from the reconstituted aggregate would silently fail to move
  the date — a regression on the **frozen** seam (decision #7). Fix: aggregate
  method is now `applyCorrection(window)`; the use-case persists the explicit
  narrow patch incl. `work_date` + `updated_by` (decisions #5, Task 4).
- **I-B — narrow persist patches for `close`/`applyCorrection`.** The `update`
  path already mandated `{ ...patch, status }`; `close` and the target
  correction now likewise persist only today's exact field sets, not the full
  aggregate (decision #5, Task 4) — avoids a widened UPDATE.
- **A-1 — `toAggregate` deviation reconciled in the blueprint.** Dropping
  `findAggregateById`/`toAggregate` contradicted §3.2 rule 3 / §11 ("reconstitution
  lives at the mapper"). Rather than override it by footnote, the
  reconstitute-from-held-record variant is now written into the blueprint at
  **§3.2 rule 3a**, with time-entries as its reference. The §6.1-ledger analogy
  was corrected: this is a full load-mutate-save aggregate (decision #10).
- **S1/S2/S3 (this pass)** — `close(now)` applying the strictly-after invariant
  to punch-out is flagged as an accepted behavior delta; `assertWindow` now
  **returns** the validated window and `InvalidTimeWindowError` derives its
  message from `field`; the admin-`update` merge keeps `!== undefined` (not
  `??`) so a `time_out: null` patch clears to `open`.

## Open questions

1. **Commit timing:** commit this snapshot now, or after review?
```
