# Implementation Plan: DDD Revamp — `compensation`

> Snapshot frozen 2026-06-28. Working copy lives in
> `asima-backend/tasks/plan.md`; checklist in `asima-backend/tasks/todo.md`.
> Backend-only feature; this parent-repo copy is the audit trail.

## Overview

Bring `src/compensation/` up to the Domain-Driven Design blueprint
(`docs/universal-guidelines/module-architecture.md`), mirroring the
`leave-requests` / `leave-allocations` / `time-correction-requests` /
`time-entries` / `work-schedules` revamps — the **last module with a genuinely
rich domain still sitting anemic**. Today the `Compensation` domain class
**carries `@ApiProperty`** (blueprint §12 anti-pattern) and the pay rules live
in `CompensationService`:

- **monthly→hourly derivation + override** — `deriveHourlyRate(monthly_salary)`
  unless an explicit `hourly_rate` is given (flags `hourly_rate_is_overridden`).
  Duplicated across `insertWithin` (create) and `update` (correction).
- **future-date invariant** — `assertNotFutureDated(effective_from)`.
- **in-place correction re-derivation** — a `monthly_salary` change re-derives
  the hourly rate and **clears** the override; an explicit `hourly_rate` wins.

After the revamp those rules live on a `PayRate` value object + a `Compensation`
aggregate; the domain is framework-free; the grant/correct/delete transitions
raise **domain events** establishing the side-effect seam.

`compensation` is a **full state-machine aggregate** (`update`/`softDelete` are
load-mutate-save), so it follows the §3.1 template: `extends AggregateRoot`,
private constructor + static `reconstitute`, behavior that `recordEvent(...)`.
Reconstitution is **use-case-side from the record each mutate path already
holds** (§3.2 rule 3a — `update`/`softDelete` ← `findById`, `insertWithin` ←
`findActiveForEmployee`), so there is **no `findAggregateById`**.

**Why this differs from work-schedules (key differences):**

1. **Self-contained.** **No cross-module consumer** (grep: only the migration +
   its own e2e reference `Compensation`). The `Compensation` → `CompensationRecord`
   rename is **internal-only** — no leave-day-count-style seam to retype.
2. **A first-class transactional audit log.** Every write records a
   `compensation_audits` row (CREATED / UPDATED / DELETED, before→after values)
   **inside the same transaction** ("the trail can't drift"). This is a hard
   atomicity requirement, so the audit **stays a use-case-side transactional
   write — NOT a domain-event subscriber** (post-commit subscribers run after
   the tx, breaking atomicity). Decision #6.
3. **A derived value object.** `PayRate` (monthly + derived/overridden hourly) is
   the money rule — richer than work-schedules' pure window/break VOs.
4. **A non-column display field.** `currency` is a single-tenant constant the
   mapper fills from `COMPENSATION_CURRENCY` (never persisted). It rides the
   **record + response DTO**, but is **not** aggregate state.

## Scope boundary

The module owns the **`employee_compensations` table + `compensation_audits`
ledger + the pay lifecycle**. Surfaces (unchanged), verified in
`src/compensation/controllers/`:

- **Self-service** (`me-compensation.controller.ts`, `COMPENSATION:ViewOwn`
  gated): `GET` my current pay (the active row; 200 + null for a new hire).
- **Admin** (`admin-compensation.controller.ts`, `COMPENSATION:ViewAll|Create|
  Update|Delete` gated): `GET` list, `GET employees/:id` history, `GET :id`,
  `GET :id/audit`, `POST` set-pay, `POST bulk`, `PATCH :id` correct,
  `DELETE :id` (soft-delete active + reactivate prior, 204).

**Internal read seams (preserve signatures — a FUTURE OT/DTR module will
consume them; no consumer exists today, verified by grep):**
`findRateOnDate`, `findRatesOnDate` (batched, **no N+1** — one query → `Map`),
`findCurrentForEmployee`, `findHistoryForEmployee`. Their `Compensation` returns
rename to `CompensationRecord`; behavior frozen.

**Out of scope:** any schema/migration change (code-only; DB + wire identical);
the OT/DTR module that will consume the rate seams; the `compensation_audits`
becoming an aggregate (it's an append-only ledger the use-case writes —
decision #6); converting `users`/`approval-chains`/etc.

## Architecture decisions

1. **Split the data bag.** Rename the current `Compensation` (domain data class,
   `@ApiProperty`-decorated) → `CompensationRecord` and strip its `@ApiProperty`
   (keep `currency` — it's the read/display shape). The name `Compensation` is
   reused for the **aggregate root**. `CompensationAudit` likewise → strip
   `@ApiProperty` (stays a **record**, not an aggregate — decision #6). Mirrors
   `WorkSchedule`/`WorkScheduleRecord`.

2. **Full aggregate (load-mutate-save).** `Compensation extends AggregateRoot`:
   `private constructor(props)` building the `PayRate` VO, static
   `reconstitute(props)`, behavior (`correct`), read accessors, events. The
   mapper keeps only `toDomain` (record); reconstitution is use-case-side (§3.2
   rule 3a) — **no `toAggregate`/`findAggregateById`**.

3. **One value object — `PayRate`** (`domain/value-objects/pay-rate.ts`,
   self-validating, unit-tested, no DI):
   - holds `monthly_salary`, `hourly_rate`, `hourly_rate_is_overridden`;
   - invariant: `monthly_salary >= 0` and `hourly_rate >= 0` (fail-fast on a
     corrupt row at `reconstitute`, like the other VOs);
   - **static factory `PayRate.fromInput(monthly_salary, hourly_rate?)`** homes
     the derivation+override rule duplicated today in `insertWithin` + `update`:
     an explicit `hourly_rate` ⇒ overridden; else derive via
     `deriveHourlyRate(monthly_salary)`, not overridden. (`deriveHourlyRate`
     stays in `compensation.constants` — a pure policy fn the VO calls.)
   - **No `EffectiveDate` VO** — the not-future rule is a single comparison
     needing the clock; it lives as a static aggregate guard taking `today`
     (below), not a stored legal-values VO (the AuditStamp lesson: only build
     VOs that validate **and** are read).

4. **No actor / no authz on the aggregate.** Admin routes are gated by
   `@Permissions({ COMPENSATION: '…' })` at the edge; `/me` is JWT identity. The
   aggregate never imports `User` and takes no actor argument (mirrors
   time-entries / work-schedules).

5. **Behavior on the aggregate** (lifts the duplicated rules):
   - **Two separate static guards — NOT one combined `assertGrantable` (I-1).**
     The not-future check and the rate derivation fire at **different times**
     today and must stay split to preserve behavior:
     - **`assertNotFuture(effective_from, today)`** — the future-date rule
       (lexicographic on `YYYY-MM-DD`); `today` fed by the use-case
       (`businessDateString()`). The use-case calls it **up-front, before any DB
       work** — single create before the transaction (`service.ts:43`), and
       **bulk `forEach` before the transaction opens** (`:72`: every batch item
       is validated before the first write). Bundling it into a creation factory
       would push the bulk check inside the per-item loop/transaction (partial
       write + rollback instead of pre-write reject) — a behavioral drift.
     - **`PayRate.fromInput(monthly_salary, hourly_rate?)`** — the derivation,
       called **inside** `insertWithin` (in the transaction), exactly where
       `service.ts:100` derives today. Returns the validated `PayRate` the
       use-case persists.
     There is **no combined `assertGrantable`** (the leave-allocations §6.1
     analogy doesn't map — compensation splits validation across the transaction
     boundary).
   - **`correct(patch, today)`** — the in-place correction transition (load-
     mutate-save): re-derive via `PayRate.fromInput` when `monthly_salary` or
     `hourly_rate` is patched (salary change re-derives + **clears** override;
     explicit hourly wins; neither ⇒ rate unchanged), run `assertNotFuture` when
     `effective_from` is patched; records `CompensationCorrected`. Returns the
     persist patch the use-case writes (`{ monthly_salary?, hourly_rate?,
     hourly_rate_is_overridden?, effective_from?, updated_by }`) — matching
     today's `next` object exactly.
   - Read accessors (`id`, `employee_id`, `monthly_salary`, `hourly_rate`,
     `hourly_rate_is_overridden`, `effective_from`, `effective_to`) the use-case
     needs for the persist patch + the audit before→after values.
   - **What stays in the use-case (I/O the aggregate can't do):** the
     effective-dating orchestration (`findActiveForEmployee` → end-date the prior
     row at `dayBefore` → insert the new row), the **`effective_from > prior
     .effective_from`** guard (needs the prior row), the **one-active 409**
     (`23505` mapping; the partial unique index stays the race-safe source of
     truth, §7), the **softDelete reactivation** (soft-delete active +
     reactivate prior — spans two rows + the index), the **bulk** duplicate-id
     pre-check, and **every `auditRepo.record(...)`** write (decision #6).

6. **The audit log stays a transactional use-case write — NOT an event
   subscriber.** Every grant/correct/delete writes a `compensation_audits` row
   via `auditRepo.record(...)` **inside the same `dataSource.transaction`** as
   the compensation write — atomicity is a hard requirement ("the trail can't
   drift"). A post-commit domain-event subscriber would run *after* the tx and
   could leave the trail inconsistent on a late failure, so the audit is **not**
   moved onto the events seam. `CompensationAudit` stays a plain append-only
   **record** + its existing repo (no audit aggregate). This mirrors
   work-schedules keeping its cascade writes inline (that plan's decision #7).

7. **Domain events** (`domain/events/compensation-events.ts`, past-tense, ids +
   scalars, `extends DomainEvent`): **`CompensationGranted`** (set-pay /
   bulk item), **`CompensationCorrected`** (in-place update),
   **`CompensationDeleted`** (soft-delete). Published by the use-case post-commit
   (creation events with the insert id; correct/delete via `pullEvents()`). **No
   subscriber reacts yet** — establishes the seam (the roadmap's OT-recompute /
   pay-change-notification consumers). **Distinct from the audit** (decision #6):
   the audit is the synchronous transactional trail HR queries; events are the
   async integration seam. **See open question #1** — compensation is the one
   module where skipping events is defensible (the audit already records every
   write), so confirm before building.

8. **Domain errors → identical HTTP codes.** Plain `Error` subclass(es) in
   `domain/compensation-errors.ts`, mapped by the use-case to the **same
   envelopes the current service emits**:

   | Domain error | HTTP (today) | Helper |
   |---|---|---|
   | `FutureEffectiveDateError` | 422 (`effective_from`) | `unprocessable('effective_from', …)` |
   | `InvalidPayRateError` (carries `field`) | 422 | `unprocessable(err.field, …)` |

   A `rethrowCompensationDomainError(err)` helper does the mapping (mirrors
   `rethrowWorkScheduleDomainError`). The one-active 409, the `effective_from >
   prior` 422, the bulk duplicate-id 422, the historical-row-delete 409, the
   `23505` 409, and the 404 stay **direct use-case throws** (I/O-bound).
   Preserve message strings verbatim: `'effective_from cannot be in the future'`,
   `` `effective_from must be after the current rate's effective_from (${…})` ``,
   etc.

9. **Wire compatibility is a hard requirement.** Add
   `dto/response/compensation-response.dto.ts` **and**
   `dto/response/compensation-audit-response.dto.ts` mirroring the current
   `@ApiProperty` fields **verbatim** (names, order, nullability, incl.
   `currency` on the compensation DTO). Assemblers use the **shared
   `toDto`/`toPaginatedDto` helpers** (`@/utils/helpers/assemble`, from the
   reuse work) — `CompensationAssembler` + `CompensationAuditAssembler`. All
   controllers return DTOs (history / bulk / audit return **plain arrays** →
   `rows.map((r) => CompensationAssembler.toResponse(r))`, **not** paginated).
   **Update the inline `$ref` schema names** in the admin controller's array
   responses: `#/components/schemas/Compensation` → `CompensationResponseDto`,
   `#/components/schemas/CompensationAudit` → `CompensationAuditResponseDto`, and
   the `type: Compensation` `@ApiResponse`s → `CompensationResponseDto`.
   **JSON byte-identical** — `mapper.toDomain`'s assignment order drives key
   order. `test/compensation.e2e-spec.ts` is the parity guard.

10. **Port surface + load-path (§3.2 rule 3a).** The port keeps every current
    method and **every `manager?: EntityManager` parameter**; `Compensation`
    returns rename → `CompensationRecord` (reads **and** `create`/`update`,
    incl. the OT read seams). **No `findAggregateById`, no `mapper.toAggregate`**
    — every mutate path already holds the record. The audit port
    (`record`/`findByCompensationId`) keeps its signatures; its mapper retypes
    to `CompensationAuditRecord`. Repos never return entities, never throw 404.

## Dependency graph

```
PayRate VO + errors                                                     [Task 1]
   └─> CompensationRecord + CompensationAudit(Record) + aggregate
        (PayRate.fromInput / assertNotFuture / correct) + events         [Task 2]
         ├─> mappers (toDomain → records) + ports (retype; keep manager;
         │     NO findAggregateById)                                      [Task 3]
         │      └─> CompensationService (create/bulk/insertWithin/update/
         │            softDelete via aggregate; audit inline; publish)    [Task 4]
         └─> response DTOs (compensation + audit) + assemblers (shared
              toDto/toPaginatedDto)                                       [Task 5]
                └─> controllers return DTOs + $ref fixes + module publisher [Task 5]
                       └─> e2e parity + gates                            [Task 6]
```

**Build order within ONE commit-set** (behavior-preserving migration; the rename
breaks intra-module callers at once). Project `tsc`/build is **red mid-flight,
green only at Checkpoint B**; Checkpoint A is domain-unit-green only.
**Don't commit until Checkpoint C is green.**

## Task List

### Phase 1 — Domain core

#### Task 1: `PayRate` value object + domain errors
**Acceptance criteria:**
- [ ] `domain/value-objects/pay-rate.ts` — holds `monthly_salary` /
      `hourly_rate` / `hourly_rate_is_overridden`; throws `InvalidPayRateError`
      when `monthly_salary < 0` or `hourly_rate < 0`; static
      `fromInput(monthly_salary, hourly_rate?)` derives (override when hourly
      given, else `deriveHourlyRate`). Unit-tested (derive, override, negative).
- [ ] `domain/compensation-errors.ts` — `FutureEffectiveDateError` (message
      `'effective_from cannot be in the future'`) + `InvalidPayRateError`
      (carries `field`), both `extends Error`.
- [ ] Zero `@nestjs/*` / `typeorm` / `class-validator` imports under `domain/`.

**Verification:** `npm test -- pay-rate` green; framework-import grep clean.
**Dependencies:** None. **Scope:** Small.

#### Task 2: Record split + aggregate + events
**Acceptance criteria:**
- [ ] `domain/compensation.ts` → `CompensationRecord`: pure data, **no
      `@ApiProperty`**, all current fields (incl. `currency`), same field order
      as `mapper.toDomain`.
- [ ] `domain/compensation-audit.ts` → `CompensationAuditRecord`: strip
      `@ApiProperty`; stays a plain append-only record (decision #6).
- [ ] `domain/compensation.aggregate.ts` → `Compensation extends AggregateRoot`:
      private ctor (builds `PayRate`), `reconstitute`, static
      `assertNotFuture(date, today)`, `correct(patch, today)`, read accessors
      (derivation lives on `PayRate.fromInput`, not a combined `assertGrantable`
      — I-1). Records `CompensationCorrected` on correction. Corrupt row
      (`monthly_salary < 0`) throws on `reconstitute`.
- [ ] `domain/events/compensation-events.ts` — `CompensationGranted(id,
      employee_id, effective_from)`, `CompensationCorrected(id, employee_id)`,
      `CompensationDeleted(id, employee_id)`.

**Verification:** `npm test -- compensation.aggregate pay-rate` green
(reconstitute → correct → assert state + `pullEvents()`; `PayRate.fromInput`
derive/override; `assertNotFuture`; corrupt-row throw). Domain framework-free.
**Dependencies:** Task 1. **Scope:** Medium.

### Checkpoint A — Domain core
- [ ] Aggregate + VO unit tests pass with no DI/DB.
- [ ] `grep` confirms zero framework imports under `domain/`.
- [ ] Review before wiring the slice.

### Phase 2 — Wire the slice

#### Task 3: Persistence layer
**Acceptance criteria:**
- [ ] `compensation.mapper.toDomain` → `CompensationRecord` (keep field-
      assignment order incl. the `currency` fill; JSON parity); `toPersistence`
      retyped to `Partial<CompensationRecord>`. `compensation-audit.mapper`
      retypes to `CompensationAuditRecord`. **No `toAggregate`.**
- [ ] Both ports retyped to the records on all reads **and** create/update;
      **every `manager?: EntityManager` preserved**. **No `findAggregateById`.**
      Repos return records, never entities, never throw 404. The audit port's
      **`RecordCompensationAuditInput` interface stays frozen** (S-2 — the
      use-case builds it verbatim); only its return type retypes to
      `CompensationAuditRecord`.
- [ ] At this task, `tsc` may be red project-wide until Task 4/5.

**Dependencies:** Task 2. **Scope:** Medium.

#### Task 4: Application layer (thin use-case)
**Acceptance criteria:**
- [ ] `create` / `createBulk`: **`assertNotFuture` up-front** (single before the
      tx; **bulk `forEach` before the tx** — all items validated pre-write, I-1)
      + bulk duplicate-id 422 → then `insertWithin` (in the tx): `PayRate
      .fromInput` derivation → `findActiveForEmployee` → `effective_from > prior`
      guard (422) → end-date prior (`dayBefore`) → `create` (derived PayRate)
      with `23505` → 409 → `auditRepo.record(CREATED)` **inside the tx** →
      publish `CompensationGranted` post-commit.
- [ ] `update` — `findById` (404) → `reconstitute` → `aggregate.correct(patch,
      today)` → `repository.update(id, patch)` + `auditRepo.record(UPDATED,
      before→after)` **in one tx** → publish `CompensationCorrected`. Re-derive
      rule + clear-override preserved exactly, **including the `effective_from`-
      only patch ⇒ no rate-field change** branch (S-1 — add a test; the current
      spec doesn't cover it).
- [ ] `softDelete` — `findById` (404) → historical-row 409 → tx: soft-delete
      active → reactivate prior (`findPreviousForEmployee` → `update
      effective_to=null`) → `auditRepo.record(DELETED)` → publish
      `CompensationDeleted`.
- [ ] OT read seams (`findRateOnDate` / `findRatesOnDate` / `findCurrentForEmployee`
      / `findHistoryForEmployee` / `findAuditTrail`) preserved (retyped to records).
- [ ] Domain errors mapped via `rethrowCompensationDomainError`; all I/O throws
      (one-active/prior-date/bulk-dup/historical-delete/`23505`/404) unchanged.

**Verification:** `npm test -- compensation.service` green: mocks both ports +
publisher + a fake `EntityManager`/`DataSource` (keep the existing
sentinel-`'MGR'` `transaction` mock); asserts set-pay (end prior + insert +
audit + event), the one-active 409 (incl. `23505`), the `effective_from > prior`
422, the not-future 422, **bulk rejects a future-dated item before any write**
(I-1 ordering), update re-derivation + clear-override + audit before→after, **the
`effective_from`-only patch leaves the rate untouched** (S-1), softDelete
reactivation + historical 409.
**Dependencies:** Task 3. **Scope:** Large — the transactional audit + the
effective-dating orchestration are the risk; review `insertWithin` and
`softDelete` separately.

#### Task 5: Interface layer + module wiring
**Acceptance criteria:**
- [ ] `dto/response/compensation-response.dto.ts` +
      `dto/response/compensation-audit-response.dto.ts` mirror the **old**
      `@ApiProperty` fields verbatim (incl. `currency`).
- [ ] `compensation.assembler.ts` + `compensation-audit.assembler.ts` using
      `toDto` / `toPaginatedDto` (`@/utils/helpers/assemble`). History / bulk /
      audit return plain arrays mapped per-item.
- [ ] Both controllers return DTOs; preserve **verbatim** every
      `@Permissions({ COMPENSATION: … })`, `@ApiTags`, mounts, the `/me` vs
      `/admin/:id` split, the 201/204 codes. **Fix the `$ref` schema names** +
      `type:` refs (decision #9).
- [ ] `CompensationModule` provides `DomainEventPublisher`.
- [ ] `main.ts` Swagger group: `Compensation` → `CompensationResponseDto`; add
      `CompensationAuditResponseDto`.

**Verification:** `npm run build`; Swagger renders the response DTOs; the domain
carries no `@ApiProperty`. **Dependencies:** Task 4. **Scope:** Medium.

### Checkpoint B — Slice wired
- [ ] `npm run lint && npm run test && npm run build` green.
- [ ] Swagger shows the response DTOs; the domain is pure.

### Phase 3 — Integration & wire-parity

#### Task 6: e2e parity + gates
**Acceptance criteria:**
- [ ] `test/compensation.e2e-spec.ts` stays green (byte-parity guard); confirm it
      exercises set-pay (end-date prior), bulk all-or-nothing, update
      re-derivation, soft-delete reactivation, the audit trail, and the 404/409/422s.
      Add a response key-set assertion if absent.
- [ ] Migrate `compensation.service.spec.ts` (mock the publisher + a fake
      `DataSource.transaction`; aggregate-driven derivation; **no
      `findAggregateById`**).
- [ ] Full gates: `npm run lint && npm run test && npm run test:e2e && npm run
      build` (e2e with `SEED_DEFAULT_PASSWORD=Asima@1234` + `STORAGE_*` + MinIO).
- [ ] Blueprint §11 checklist — load-path rows **N/A** per §3.2 rule 3a; the
      audit-as-transactional-ledger noted as a sanctioned deviation (decision #6).

**Dependencies:** Task 5. **Scope:** Medium.

### Checkpoint C — Complete
- [ ] All acceptance criteria met; §11 checklist green.
- [ ] All gates pass; ready to commit straight to `main` in `asima-backend`.

## Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| **Audit trail drifts from the write** (atomicity) | **High** | Decision #6: audit stays an inline `auditRepo.record(...)` in the same `dataSource.transaction`, NOT an event subscriber; service spec asserts the audit row + the compensation write commit together; e2e checks the trail. |
| Re-derivation / clear-override rule diverges | **High** | Centralize in `PayRate.fromInput` + `aggregate.correct`; aggregate + service specs assert: salary change re-derives & clears override, explicit hourly wins, neither leaves rate unchanged. |
| Effective-dating orchestration breaks (end prior / reactivate prior) | **High** | Keep it in the use-case (it spans rows + the index); preserve the `effective_from > prior` 422 + the `dayBefore` end-date + the softDelete reactivation; e2e guards. |
| One-active invariant weakened | **High** | Keep the partial unique index + the `23505` → 409 mapping in the use-case (§7); don't move it onto the aggregate. |
| Dropped `EntityManager` params → transactions break | **High** | Decision #10/Task 3: preserve every `manager?: EntityManager` on both retyped ports. |
| JSON drift (`currency` non-column field / field order / audit shape) | Med | `mapper.toDomain` order == legacy incl. the `currency` fill; response DTOs mirror `@ApiProperty` verbatim; fix the `$ref` names; e2e parity guard. |
| Over-building (events that duplicate the audit / EffectiveDate VO / findAggregateById) | Low | Decisions #3/#6/#7/#10: one VO (`PayRate`); audit stays a record; no `EffectiveDate` VO; no `findAggregateById`; events flagged as open question #1. |

## Plan-review pass (2026-06-28)

A five-axis review against the live code tightened the plan before
implementation:

- **I-1 — split the creation guard.** The original `assertGrantable` bundled the
  not-future check with the rate derivation, but they fire at **different times**
  today: not-future is **up-front, pre-transaction** (single `:43`; bulk
  `forEach` `:72` — all items before any write), while derivation is **in the
  transaction** (`:100`). Bundling would push the bulk check inside the per-item
  loop (partial-write-then-rollback instead of pre-write reject). Fixed:
  use-case calls `assertNotFuture` up-front; `PayRate.fromInput` derives inside
  `insertWithin`; no combined `assertGrantable` (decision #5, Tasks 2/4).
- **S-1 — `effective_from`-only correction.** The existing spec covers the three
  rate-change branches but not "no rate field patched ⇒ rate unchanged". `correct`
  must preserve it; Task 4/6 adds the test.
- **S-2 — frozen audit input.** `RecordCompensationAuditInput` (the interface the
  use-case hands `auditRepo.record`) stays verbatim; only its return retypes
  (Task 3).
- **Affirmed:** decision #6 (audit inline-transactional, not an event subscriber)
  is correct and grounded in the audit port's `manager?` design; the batched OT
  seam stays N/A for N+1; pay routes' `COMPENSATION:*`/`ViewOwn` gating is right.
- **FYI:** bulk's per-item `findActiveForEmployee` is pre-existing (import path),
  not a regression — don't "optimize" it in this behavior-preserving change.

## Open questions

1. **Domain events vs. the existing audit (the key call).** Compensation is the
   one module where the events seam **overlaps** a first-class transactional
   audit that already records every write with before→after values. **Lean:
   include the minimal `Granted`/`Corrected`/`Deleted` set** for consistency with
   the 5 prior revamps + a plausible OT-recompute / pay-notification consumer,
   keeping the audit separate (decision #6/#7). **But skipping events here is
   defensible** (the audit already records; the time-correction "don't build
   what nothing reads" lesson) — confirm before Task 2 builds them.
2. **`PayRate` invariant scope.** Plan validates `>= 0` only (the DTO owns the
   real `@Min`/positivity at the edge). Tighten if the aggregate should re-assert
   a stricter bound at reconstitute.
3. **Commit timing:** commit this snapshot now, or after review?
```
