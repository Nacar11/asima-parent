# Implementation Plan: DDD Revamp — `work-schedules`

> Snapshot frozen 2026-06-27. Working copy lives in
> `asima-backend/tasks/plan.md`; checklist in `asima-backend/tasks/todo.md`.
> Backend-only feature; this parent-repo copy is the audit trail.

## Overview

Bring `src/work-schedules/` up to the Domain-Driven Design blueprint
(`docs/universal-guidelines/module-architecture.md`), mirroring the
`leave-requests` / `time-correction-requests` / `time-entries` revamps. Today
the module is hexagonal-with-an-anemic-domain: the `WorkSchedule` domain class
**carries `@ApiProperty`** (blueprint §12 anti-pattern) and the schedule
invariants live as **free functions** in the service —
`assertWindowOk` (`expected_out` strictly after `expected_in`) and
`assertBreakOk` (break ≥ 0, `break_start` required when positive, break fits
inside the window). Those two rules are **duplicated across `create` and
`update`** and **re-imported cross-file** by `schedule-change.service.ts`.

After the revamp those rules live on a `WorkWindow` value object + a `Break`
value object + a `WorkSchedule` aggregate; the domain is framework-free; the
effective-dating lifecycle (`create` / `endLogically`) raises **domain events**
that establish the side-effect seam.

`work-schedules` is a **full state-machine aggregate** — `endLogically` (set
`effective_to`) is a load-mutate-save transition — so it follows the §3.1
template: `extends AggregateRoot`, private constructor + static `reconstitute`,
behavior methods that `recordEvent(...)`. Reconstitution happens **use-case-side
from the record each mutate path already holds** (the §3.2 rule 3a
reconstitute-from-held-record variant, exactly as time-entries — there is no
`findAggregateById`).

**Why this is heavier than time-entries (read before scoping):**

1. **Two value objects, not one.** `WorkWindow` (the punch window) **and**
   `Break` (length + start), plus a cross-VO rule "the break fits inside the
   window" that lives on the aggregate (it needs both).
2. **The rename is NOT internal-only.** `leave-requests/leave-day-count
   .service.ts` **imports the `WorkSchedule` domain type** and reads
   `.expected_in` / `.expected_out` / `.break_minutes` / `.break_start` /
   `.day_of_week` off it (verified by grep). Renaming `WorkSchedule` →
   `WorkScheduleRecord` therefore edits a **leave-requests** file too. (Contrast
   time-entries, where no cross-module code imported the domain type.)
3. **A second concern shares the module — the schedule-change cascade.** The
   `ScheduleChangeService` (`preview`/`apply`) does a deliberate **multi-aggregate
   transaction** (versions the schedule **and** system-cancels affected leave +
   TC requests). Its decision logic is **already pure** in `cascade-policy.ts`.
   The cascade's behavior is **frozen**; this revamp only (a) retypes its
   `WorkSchedule` references to `WorkScheduleRecord`, (b) rewires its
   `assertWindowOk`/`assertBreakOk` imports to the new aggregate guards, and
   (c) relocates its `@ApiProperty` response shapes (`AffectedRequest` /
   `ScheduleChangeImpact` / `ScheduleChangeResult`) out of `domain/` into
   `dto/response/`.

## Scope boundary

The module owns the **`work_schedules` table + the weekly-schedule lifecycle**
**and** the **admin schedule-change cascade**. Surfaces (unchanged by this
revamp), verified in `src/work-schedules/controllers/`:

- **Self-service** (`me-work-schedules.controller.ts`, JWT-only): `GET` my
  active week.
- **Admin schedules** (`admin-work-schedules.controller.ts`,
  `SCHEDULE:View|Create|Update|Delete` gated): `GET` list, `GET :id`, `POST`
  create, `PATCH :id`, `DELETE :id` (logical-end, **not** physical delete).
- **Admin schedule-change** (`admin-schedule-changes.controller.ts`): `POST
  preview` (dry-run), `POST apply` (transactional cascade).

**Cross-module seams that MUST keep compiling + behaving (HARD requirement —
verified by grep):**

- `leave-requests/leave-day-count.service.ts` — imports the `WorkSchedule`
  domain type and calls **`BaseWorkScheduleRepository.findActiveForEmployee`**,
  reading `.day_of_week` / `.expected_in` / `.expected_out` / `.break_minutes` /
  `.break_start`. The repo method **signature + behavior are frozen**; it now
  returns `WorkScheduleRecord[]` (same fields). **This file (+ its spec) is
  edited** to import `WorkScheduleRecord` — a type-name change only, no logic
  change. Re-run its unit spec.
- `ScheduleChangeService` → **`BaseLeaveRequestRepository.systemCancel` /
  `findActiveCandidatesForScheduleChange`** and the **time-correction**
  equivalents. These are the leave/TC **ports** (one-way: schedule →
  leave/TC). **Frozen — this revamp does not touch leave or TC.**
- `ScheduleChangeService` → `BaseWorkScheduleRepository` (`create` / `update` /
  `softDelete` / `findActiveForEmployeeDay`, each **with an optional
  `manager?: EntityManager`** for the in-transaction path). The
  `EntityManager` parameters are **preserved verbatim** on the retyped port.

**Out of scope:** any schema/migration change (code-only; DB + wire identical);
the cascade decision matrix / `cascade-policy.ts` logic (already pure — only its
`WorkSchedule` **type** references change); retrofitting domain events into the
cascade versioning path (decision #7); converting leave/TC/users/etc.

## Architecture decisions

1. **Split the data bag.** Rename the current `WorkSchedule` (domain data class,
   `@ApiProperty`-decorated) → `WorkScheduleRecord` and strip its `@ApiProperty`.
   The name `WorkSchedule` is reused for the **aggregate root**. **No list-item**
   (the module has no joined read-model). Mirrors `TimeEntry` / `TimeEntryRecord`
   (§3.1). **Cross-module:** update the `leave-day-count` import to
   `WorkScheduleRecord` (decision — the rename is not internal-only).

2. **Full aggregate (load-mutate-save).** `WorkSchedule extends AggregateRoot`:
   `private constructor(props)` building the `WorkWindow` + `Break` VOs and
   running the break-fits-window cross-check, static `reconstitute(props)`,
   behavior methods that mutate in-memory and `recordEvent(...)`, drained by the
   use-case via `pullEvents()`. Reconstitution is **use-case-side** from
   already-held records (§3.2 rule 3a) — the mapper keeps only `toDomain`
   (record); there is no `toAggregate` / `findAggregateById`.

3. **Two value objects + one cross-VO guard:**
   - **`WorkWindow`** (`domain/value-objects/work-window.ts`): holds
     `expected_in` / `expected_out` (HH:MM:SS strings); invariant
     `expected_out` is **strictly after** `expected_in` (lexicographic compare
     on zero-padded times — homes `assertWindowOk`). Throws
     `InvalidWorkWindowError('expected_out')`.
   - **`Break`** (`domain/value-objects/break.ts`): holds `break_minutes` +
     nullable `break_start`; invariants `break_minutes >= 0` and `break_start`
     **required when `break_minutes > 0`** (homes the standalone half of
     `assertBreakOk`). Throws `InvalidBreakError(field)` with `field` ∈
     `{'break_minutes','break_start'}`.
   - **Cross-VO rule on the aggregate** — `WorkSchedule.assertBreakWithinWindow(
     window, brk)`: when a break is present, `break_start >= expected_in` and
     `break_start + break_minutes ≤ expected_out` (homes the window-relative
     half of `assertBreakOk`, using `toSeconds`). It lives on the aggregate, not
     a VO, because it spans two VOs. Returns void; throws
     `InvalidBreakError('break_start')` with the verbatim legacy messages.
   - **No `AuditStamp` / `EntrySource`-style VOs** — only build VOs that
     validate **and** are read (the time-correction `AuditStamp` lesson). Both
     `WorkWindow` and `Break` validate and are read by the persist patch.

4. **No actor / no authz on the aggregate.** Admin schedule routes are gated by
   `@Permissions({ SCHEDULE: '…' })` at the edge; `/me` is JWT identity. The
   aggregate never imports `User` or `hasPermission` and takes **no actor
   argument** (a deliberate simplification, as in time-entries). NOTE: the
   cascade's body-dependent `SCHEDULE:Delete` check for a `remove` intent
   (`ScheduleChangeService.validate`, C2) is **use-case authz, stays in the
   service** — it is not an aggregate concern.

5. **Behavior on the aggregate** (lifts the duplicated rules):
   - **Static creation guard** (insert-id-bound creation — no full factory):
     `assertSchedule(expected_in, expected_out, break_minutes, break_start?)`
     builds `WorkWindow` + `Break`, runs `assertBreakWithinWindow`, and
     **returns `{ window, brk }`** so the use-case persists without rebuilding.
     Maps each VO error to its 422 field.
   - `endLogically(effective_to: Date|string)` — the logical-end transition: set
     `effective_to`, guard "already ended" stays **use-case-side** (it is a
     read of current state → 409, like the one-open guard); records
     `WorkScheduleEnded`. Load-mutate-save.
   - Derived/read accessors the use-case needs for the persist patch
     (`id`, `employee_id`, `day_of_week`, `expected_in`, `expected_out`,
     `break_minutes`, `break_start`, `effective_from`, `effective_to`).
   - **Admin `update` is NOT an aggregate transition (mirror time-entries
     decision #5).** It stays a thin **use-case patch**: `findById` (404) →
     merge the patch over the existing row (`break_start` via `!== undefined`,
     **not** `??`, so an explicit `null` clears it) → `assertSchedule` on the
     merged values (422) → `repository.update(id, patch)`. Records **no event**
     (an admin edit of an unstarted/active row is not a downstream fact). The
     break/window rules are centralized in `assertSchedule`, which is all
     `update` needs.
   - **What stays in the use-case:** the **one-active-per-(employee,weekday)**
     guard (`findActiveForEmployeeDay` + the `23505` → 409 mapping; the partial
     unique index stays the race-safe source of truth, blueprint §7), the
     `endLogically` **already-ended** 409, and the 404s.

6. **Domain events** (`domain/events/work-schedule-events.ts`, past-tense, ids +
   scalars, `extends DomainEvent`): **`WorkScheduleCreated`** (admin create),
   **`WorkScheduleEnded`** (logical end). Published by the use-case post-commit
   (creation event with the insert id; end via `publisher.publish(pullEvents())`).
   **No subscriber reacts yet** — establishes the seam; behavior unchanged.

   **Decision #7 — the cascade versioning path stays event-free.** The
   `ScheduleChangeService.applyVersioning` calls the repo **directly** (with a
   transaction `manager`) for its `end_and_create` / `replace` / `end_only` /
   `delete_only` actions. It is **not** rerouted through the aggregate and emits
   **no** new events in this revamp: it is an already-designed multi-aggregate
   transaction (`2026-06-20-admin-schedule-change-cascade.md`) whose behavior is
   frozen, and it already publishes its outcome via the `systemCancel` calls. A
   future `ScheduleVersioned` event, if needed, is a separate change. (This keeps
   the revamp behavior-preserving and the diff small.)

7. **Cascade contract is frozen (cross-cutting).** Preserve the exact
   `preview` / `apply` signatures, the drift-guard 409, the `planVersioning`
   actions, the `evaluateLeave` / `evaluateCorrection` matrix, the
   transaction boundary, and the `systemCancel` calls. Internally: (a)
   `cascade-policy.ts` retypes its `WorkSchedule` references → `WorkScheduleRecord`
   (it only reads `.effective_from` / `.expected_in` / `.expected_out` /
   `.break_minutes` / `.break_start`); (b) `ScheduleChangeService.validate`
   swaps `assertWindowOk`/`assertBreakOk` for `WorkSchedule.assertSchedule`
   (same 422s); (c) the `created_row` it returns is a `WorkScheduleRecord`.

8. **Domain errors → identical HTTP codes.** Plain `Error` subclasses in
   `domain/work-schedule-errors.ts`, mapped by the use-case to the **same
   envelopes the current service emits**:

   | Domain error | HTTP (today) | Helper |
   |---|---|---|
   | `InvalidWorkWindowError` (carries `field`) | 422 | `unprocessable(err.field, …)` |
   | `InvalidBreakError` (carries `field`) | 422 | `unprocessable(err.field, …)` |

   A `rethrowWorkScheduleDomainError(err)` helper does the mapping (mirrors
   `rethrowTimeEntryDomainError`). The one-active 409, already-ended 409,
   `23505` 409, and 404 stay **direct use-case throws** (I/O-bound). Preserve
   message strings verbatim: `'expected_out must be strictly after expected_in'`,
   `'break_minutes must be >= 0'`, `'break_start is required when break_minutes
   > 0'`, `'break_start must be on or after expected_in'`, `'the break must end
   on or before expected_out'`.

9. **Wire compatibility is a hard requirement.** Add
   `dto/response/work-schedule-response.dto.ts` mirroring the current
   `WorkSchedule` `@ApiProperty` fields **verbatim** (names, order, nullability;
   `day_of_week` min 0 / max 6). **Relocate the cascade response shapes**
   `AffectedRequest` / `ScheduleChangeImpact` / `ScheduleChangeResult` from
   `domain/schedule-change.ts` into `dto/response/` (they currently carry
   `@ApiProperty` in `domain/` — the anti-pattern). Keep the pure working types
   (`ScheduleChangeIntent`, `VersioningAction`, `TemporalClass`,
   `CascadeDecision`, and the plain-data `AffectedRequest`/`Impact`/`Result`
   the policy/service compute) in `domain/`, **stripped of `@ApiProperty`**; the
   response DTOs mirror them and a `ScheduleChangeAssembler` maps the structural
   copy. `ScheduleChangeResultResponseDto.created_row` is a
   `WorkScheduleResponseDto | null`. A `WorkScheduleAssembler`
   (`toResponse` / `toPaginatedResponse` — **no list-item**) serves the schedule
   controllers. **JSON byte-identical** — `mapper.toDomain`'s assignment order
   drives key order. The existing `test/work-schedules.e2e-spec.ts` +
   `test/schedule-changes.e2e-spec.ts` are the parity guards.

10. **Port surface + load-path strategy (§3.2 rule 3a).** The port keeps every
    current method and **every `manager?: EntityManager` parameter**; the
    `WorkSchedule` return types rename → `WorkScheduleRecord` (reads **and**
    `create`/`update`). **No `findAggregateById`, no `mapper.toAggregate`** —
    every mutate path already holds the record (`update`/`endLogically` ←
    `findById`), so reconstitution is use-case-side via
    `WorkSchedule.reconstitute(record)`. Repos never return entities, never
    throw 404. The §11 load-path rows are intentionally N/A (rule 3a).

## Dependency graph

```
WorkWindow VO + Break VO + errors                                       [Task 1]
   └─> WorkScheduleRecord + aggregate (assertSchedule/endLogically) + events [Task 2]
         ├─> mapper (toDomain → Record) + port (retype; keep EntityManager)  [Task 3]
         │      └─> WorkSchedulesService (create/update/endLogically/publish) [Task 4]
         │             └─> ScheduleChangeService rewire (assertSchedule) +
         │                  cascade-policy retype + leave-day-count retype    [Task 5]
         └─> response DTOs (schedule + relocated cascade) + assemblers        [Task 6]
                └─> controllers return DTOs + module (publisher)              [Task 6]
                       └─> e2e parity (schedules + schedule-changes) +
                            verify leave-day-count + cascade seams + gates    [Task 7]
```

**Build order within ONE commit-set**, not independently-shippable slices
(behavior-preserving migration; the rename breaks intra-module **and the
leave-day-count** caller at once). Project `tsc`/build is **red mid-flight,
green only at Checkpoint B**; Checkpoint A is domain-unit-green only.
**Don't commit until Checkpoint C is green.**

## Task List

### Phase 1 — Domain core (the consistency boundary)

#### Task 1: `WorkWindow` + `Break` value objects + domain errors
**Acceptance criteria:**
- [ ] `domain/value-objects/work-window.ts` — `expected_in` / `expected_out`
      strings; throws `InvalidWorkWindowError` when `expected_out <= expected_in`
      (lexicographic). Exposes both + the strings. Unit-tested (after, equal,
      before).
- [ ] `domain/value-objects/break.ts` — `break_minutes` / `break_start`; throws
      `InvalidBreakError('break_minutes')` when negative, `InvalidBreakError(
      'break_start')` when minutes > 0 and start null. Exposes both +
      `hasBreak()`. Unit-tested (zero break/null start ok, positive+null throws,
      negative throws).
- [ ] `domain/work-schedule-errors.ts` — `InvalidWorkWindowError` /
      `InvalidBreakError` extend `Error`, each carrying a `field: string`.
      Messages derived/verbatim per decision #8.
- [ ] Zero `@nestjs/*` / `typeorm` / `class-validator` imports under `domain/`.

**Verification:** `npm test -- work-window break` green; framework-import grep
clean. **Dependencies:** None. **Scope:** Small.

#### Task 2: Record split + aggregate + events
**Acceptance criteria:**
- [ ] `domain/work-schedule.ts` → `WorkScheduleRecord`: pure data, **no
      `@ApiProperty`**, snake_case, definite-assignment, all current fields, same
      field order as `mapper.toDomain`.
- [ ] `domain/work-schedule.aggregate.ts` → `WorkSchedule extends AggregateRoot`:
      private ctor (builds `WorkWindow` + `Break`, runs `assertBreakWithinWindow`),
      `reconstitute`, static `assertSchedule(...)` (returns `{ window, brk }`),
      static `assertBreakWithinWindow(window, brk)`, `endLogically(effective_to)`,
      read accessors. Records `WorkScheduleEnded` on the end transition. **No
      `revise`** (admin `update` is a use-case patch — decision #5).
- [ ] `reconstitute` builds the VOs + cross-check, so a corrupt row throws on
      load — fail-fast (mirrors the exemplars).
- [ ] `domain/events/work-schedule-events.ts` — `WorkScheduleCreated(id,
      employee_id, day_of_week, effective_from)`, `WorkScheduleEnded(id,
      employee_id, effective_to)`.

**Verification:** `npm test -- work-schedule.aggregate` green (reconstitute →
endLogically → assert state + `pullEvents()`; `assertSchedule` valid/invalid per
field; break-within-window throws; corrupt-row throw). Domain framework-free.
**Dependencies:** Task 1. **Scope:** Medium.

### Checkpoint A — Domain core
- [ ] Aggregate + VO unit tests pass with no DI and no DB.
- [ ] `grep` confirms zero framework imports under `domain/` (incl. the
      retyped `schedule-change.ts` working types).
- [ ] Review before wiring the slice.

### Phase 2 — Wire the slice

#### Task 3: Persistence layer
**Acceptance criteria:**
- [ ] `mapper.toDomain` → `WorkScheduleRecord` (keep field-assignment order for
      JSON parity); `toPersistence` retyped to `Partial<WorkScheduleRecord>`.
      **No `toAggregate`** (rule 3a).
- [ ] Port retyped to `WorkScheduleRecord` on all reads **and**
      `create`/`update`; **every `manager?: EntityManager` parameter preserved
      verbatim**. **No `findAggregateById`.** Repo returns records, never
      entities, never throws 404.
- [ ] At this task, `tsc` may be **red project-wide** until Task 4–6 — confirm
      the persistence files themselves typecheck.

**Dependencies:** Task 2. **Scope:** Medium.

#### Task 4: Application layer — `WorkSchedulesService` (thin use-case)
**Acceptance criteria:**
- [ ] `create` — `assertSchedule` (422 per field) → one-active guard (409, only
      when `effective_to == null`) → `repository.create` with `23505` → 409 →
      publish `WorkScheduleCreated`.
- [ ] `update` — **use-case patch, no aggregate / no event** (decision #5):
      `findById` (404) → merge patch (`break_start` via `!== undefined`) →
      `assertSchedule` on merged values (422) → `repository.update(id, patch)`.
- [ ] `endLogically` — `findById` (404) → already-ended 409 →
      `WorkSchedule.reconstitute(existing)` → `aggregate.endLogically(effective_to)`
      → `repository.update(id, { effective_to, updated_by })` → publish
      `WorkScheduleEnded`.
- [ ] `findAll` / `findById` / `findActiveForEmployee` / `softDelete` preserved.
- [ ] Domain errors mapped via `rethrowWorkScheduleDomainError`; all I/O throws
      (one-active/already-ended/`23505`/404) unchanged.

**Verification:** `npm test -- work-schedules.service` green: mocks the port +
publisher; asserts create (incl. one-active 409 + `23505`), update derived
validation + `break_start` clear-to-null, `endLogically` (event + already-ended
409). **Dependencies:** Task 3. **Scope:** Large.

#### Task 5: Cascade rewire + cross-module retypes (frozen behavior)
**Acceptance criteria:**
- [ ] `cascade-policy.ts` — retype `WorkSchedule` → `WorkScheduleRecord`
      everywhere (`planVersioning`, `windowChanged`, `breakChanged`,
      `evaluateLeave`, `evaluateCorrection`). **No logic change.** Its spec stays
      green unchanged (or only the type import updates).
- [ ] `schedule-change.service.ts` — `validate` swaps `assertWindowOk` /
      `assertBreakOk` for `WorkSchedule.assertSchedule` (same 422 fields +
      messages); retype `WorkSchedule` → `WorkScheduleRecord` (incl.
      `applyVersioning`'s `live` param and `created_row`). **Versioning path
      unchanged** (decision #7 — direct repo calls, no events). Remove the now-
      unused `assertWindowOk`/`assertBreakOk` exports from
      `work-schedules.service.ts` **only after** confirming no other importer
      (grep) — else keep as thin re-exports delegating to the aggregate guard.
- [ ] `leave-requests/leave-day-count.service.ts` (+ its `.spec.ts`) — import
      `WorkScheduleRecord` instead of `WorkSchedule`; retype the two annotations.
      **No logic change.** Re-run its unit spec (day-count math frozen).

**Verification:** `npm test -- cascade-policy schedule-change leave-day-count`
green. **Dependencies:** Task 4. **Scope:** Medium — the riskiest task (the
cascade + the leave-day-count seam). Review the cascade rewire separately.

#### Task 6: Interface layer + module wiring
**Acceptance criteria:**
- [ ] `dto/response/work-schedule-response.dto.ts` mirrors the **old**
      `WorkSchedule` `@ApiProperty` fields verbatim.
- [ ] Relocate `AffectedRequest` / `ScheduleChangeImpact` /
      `ScheduleChangeResult` to `dto/response/schedule-change-response.dto.ts`
      (carry `@ApiProperty`); strip `@ApiProperty` from the `domain/
      schedule-change.ts` working types. `created_row` →
      `WorkScheduleResponseDto | null`.
- [ ] `work-schedule.assembler.ts` (`toResponse` / `toPaginatedResponse`) +
      `schedule-change.assembler.ts` (`toImpactResponse` / `toResultResponse`).
- [ ] All three controllers return DTOs via the assemblers; preserve **verbatim**
      every `@Permissions({ SCHEDULE: … })`, `@ApiTags`, `@ApiBearerAuth`, the
      `version: API_VERSION` mount, the `/me` vs `/admin/:id` split, the
      preview/apply mounts, and the DELETE→logical-end 200.
- [ ] `WorkSchedulesModule` provides `DomainEventPublisher`.
- [ ] Update `main.ts` Swagger groups: `WorkSchedule` →
      `WorkScheduleResponseDto`; add the relocated cascade DTOs to the group.

**Verification:** `npm run build`; Swagger renders the response DTOs; the domain
carries no `@ApiProperty`. **Dependencies:** Tasks 4–5. **Scope:** Medium.

### Checkpoint B — Slice wired
- [ ] `npm run lint && npm run test && npm run build` green.
- [ ] Swagger shows the response DTOs; the domain (incl. `schedule-change.ts`)
      is pure.

### Phase 3 — Integration & wire-parity

#### Task 7: e2e parity + verify the seams + gates
**Acceptance criteria:**
- [ ] `test/work-schedules.e2e-spec.ts` + `test/schedule-changes.e2e-spec.ts`
      stay green (byte-parity guards). Add a response key-set assertion if
      absent.
- [ ] **Verify the cross-module seams:** `leave-requests` compiles + its e2e
      (`leave-requests.e2e-spec.ts`) and `leave-day-count.service.spec.ts` stay
      green; the cascade's `systemCancel` into leave/TC still fires
      (`schedule-changes.e2e-spec.ts` exercises it).
- [ ] Migrate `work-schedules.service.spec.ts` + `schedule-change.service.spec.ts`
      + `admin-schedule-changes.controller.spec.ts` (mock the publisher;
      aggregate-driven validation; **no `findAggregateById`**).
- [ ] Full gates: `npm run lint && npm run test && npm run test:e2e && npm run
      build` (e2e with `SEED_DEFAULT_PASSWORD=Asima@1234` + `STORAGE_*` + MinIO).
- [ ] Blueprint §11 checklist satisfied — load-path rows N/A per §3.2 rule 3a.

**Dependencies:** Task 6. **Scope:** Medium.

### Checkpoint C — Complete
- [ ] All acceptance criteria met; §11 checklist green.
- [ ] All gates pass; ready to commit straight to `main` in `asima-backend`.

## Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| **Rename breaks `leave-day-count` (cross-module type import)** | **High** | Decision #1/Task 5: edit the import to `WorkScheduleRecord` (fields identical); re-run `leave-day-count.service.spec.ts` + `leave-requests` e2e. |
| **Cascade contract drifts → breaks the schedule-change apply** | **High** | Freeze `preview`/`apply`/`planVersioning`/matrix (decision #7); Task 7 re-runs `schedule-changes.e2e`. Only types + the `assertSchedule` rewire change. |
| `assertWindowOk`/`assertBreakOk` removed while still imported | Med | Task 5: grep importers first; keep thin re-exports delegating to the aggregate guard if any remain. |
| Break-within-window rule diverges from legacy | **High** | Centralize in `assertBreakWithinWindow`; aggregate + service specs assert the four legacy messages verbatim. |
| Dropped `EntityManager` params → cascade transaction breaks | **High** | Decision #10/Task 3: preserve every `manager?: EntityManager` parameter on the retyped port. |
| JSON drift (field order / cascade DTO relocation) | Med | `mapper.toDomain` order == legacy; response DTOs mirror `@ApiProperty` verbatim; e2e parity guards. |
| One-active invariant weakened | **High** | Keep the partial unique index + the `23505` → 409 mapping in the use-case (blueprint §7); don't move it onto the aggregate. |
| Over-building (actor / 3rd VO / findAggregateById / cascade events) | Low | Decisions #2/#4/#5/#7/#10: two VOs only; no actor, no `findAggregateById`, no cascade events (YAGNI). |

## Open questions

1. **`Break` as one VO vs. folding into `WorkWindow`?** Plan splits them (a
   break is conceptually distinct and independently nullable). Revisit if the
   cross-VO `assertBreakWithinWindow` feels awkward in implementation.
2. **Cascade `ScheduleVersioned` event** — deferred (decision #7). Add when a
   subscriber needs it (DTR recompute on schedule change).
3. **Commit timing:** commit this snapshot now, or after review?
```
