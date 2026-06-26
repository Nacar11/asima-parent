# Implementation Plan: DDD Revamp — `leave-allocations`

> Snapshot frozen 2026-06-26. Working copy lives in
> `asima-backend/tasks/plan.md`; checklist in `asima-backend/tasks/todo.md`.
> Backend-only feature; this parent-repo copy is the audit trail.

## Overview

Bring `src/leave-allocations/` up to the Domain-Driven Design blueprint
(`docs/universal-guidelines/module-architecture.md`), the same revamp just
landed on `leave-requests`. Today the module is hexagonal-with-an-anemic-domain:
the `LeaveAllocation` domain class is a bag of data that **carries
`@ApiProperty`** (a blueprint anti-pattern §12) and all rules live at the
edges (DTO `@Min(1)`, DB `CHECK (amount > 0)`). After the revamp the invariants
live in a pure aggregate + value objects, the domain is framework-free, and a
creation **domain event** establishes the side-effect seam.

`leave-allocations` is the right next target: it already sits on the hexagonal
persistence ports the DDD work builds on, it is the smallest such module, and
it is the *write side* of the leave-balance story whose *read side*
(`leave-requests`) was just revamped — so the patterns and value objects carry
straight over.

## Scope boundary

The module is an **append-only grant ledger**: one row GRANTS `amount` days of a
leave type to an employee; the allowance is `SUM(amount)`; revocation = soft-
delete a row (no endpoint exists today). Surfaces:

- `POST /admin/users/:id/leave-allocations` — grant (write path).
- `GET  /admin/users/:id/leave-allocations` — grant history (read path).
- `GET  /admin/users/:id/leave-balances` — balances; already delegates to
  `leave-requests`' `LeaveBalanceService` + `LeaveBalanceResponseDto`. **Out of
  scope** beyond an import check.

Cross-module consumers of the allocation **port** (not the domain class) that
must keep compiling: `UsersService.create` (grants the 2 defaults),
`LeaveBalanceService` + `LeaveRequestsService` (`sumsByEmployee` /
`sumForUpdate`), and the user seeder (uses the raw `LeaveAllocationEntity`, not
the port).

## Architecture decisions

1. **Split the data bag.** Rename the current `LeaveAllocation` (domain data
   class) → `LeaveAllocationRecord` and strip its `@ApiProperty`. The name
   `LeaveAllocation` is reused for the **(thin) aggregate root** (see #2).
   Mirrors `LeaveRequest` / `LeaveRequestRecord` (blueprint §3.1, hard rule).
2. **Thin, creation-only aggregate; minimal port (review I2).** The ledger is
   append-only with **no mutation** (no update/revoke endpoint). So the
   aggregate carries only the creation invariant — value objects + a pure
   static guard + the event class — and we omit the entire load-mutate-save
   apparatus: **no** `findAggregateById`, `update`, `softDelete` on the port;
   **no** `mapper.toAggregate`; **no** aggregate `reconstitute`; **no**
   `pullEvents()` drain. In `leave-requests` those exist only because
   approve/reject/cancel load-mutate-save (`leave-requests.service.ts:289/357/397`);
   here they would be dead code. Because nothing is buffered, the class need
   **not** extend `AggregateRoot`. This is a deliberate, documented deviation
   from the §2 file template and the §11 checklist, justified by the boundary —
   the module sits near the §6 "some tables are not aggregates" line, and the
   chosen treatment is the honest minimum, not a manufactured mirror of
   `leave-requests`. The port keeps its current methods; only `create` /
   `listForEmployee` **return types** change to `LeaveAllocationRecord`.
3. **Value objects.** `AllocationAmount` (positive integer — the column is
   `integer NOT NULL … CHECK (amount > 0)`). `leave_type` stays the existing
   `LeaveType` enum (validated at the DTO) and `source` is server-set to
   `admin_grant` in the use-case — neither needs a VO. (An `AllocationSource`
   VO was planned but removed post-implementation; see the I1 amendment.) The
   `[1, 365]` upper bound stays a **DTO/UI guardrail**, not a domain invariant.
4. **Insert-id-bound creation — published by the use-case (review I1, blueprint
   §3.1).** The row id is DB-generated, so the aggregate does **not** record the
   creation event — exactly how `leave-requests` does it:
   `leave-requests.service.ts:214` builds `new LeaveRequestSubmitted(created.id,
   …)` in the use-case and publishes it. Here: the aggregate exposes the pure
   guard `LeaveAllocation.assertGrantable(amount)` (constructs `AllocationAmount`,
   throws the domain error); the use-case validates via the guard → persists via
   the port → constructs `new LeaveAllocationGranted(created.id, …)` →
   `publisher.publish([...])`. No id-less buffered event, no post-hoc id stamping.
5. **`LeaveBalance` is a read-model, not an aggregate** (blueprint §6) —
   untouched; it stays owned by `leave-requests` and is projected, never
   written.
6. **Wire compatibility is a hard requirement.** A new
   `LeaveAllocationResponseDto` mirrors the current `LeaveAllocation`
   `@ApiProperty` fields **verbatim** (same names, order, nullability), and a
   `LeaveAllocationAssembler` does a structural copy (`Object.assign(new Dto(),
   record)`). JSON must be byte-identical to today. **Note (review S2):** the
   actual JSON key order is driven by `mapper.toDomain`'s field-assignment
   order (the assembler copies record keys), not the DTO declaration — so keep
   `toDomain`'s order == the legacy field order. The DTO order is for Swagger.
7. **Event seam without a subscriber.** As with the `leave-requests` revamp, no
   subscriber reacts to `LeaveAllocationGranted` yet — the event establishes the
   seam; behavior is unchanged.
8. **Add e2e coverage (resolved 2026-06-26).** The module has no
   `test/leave-allocations.e2e-spec.ts` today, so the byte-parity guard is
   missing. The revamp adds a thin one (grant happy-path, history, 422 on a bad
   amount). The `AllocationAmount` `[1, 365]` cap stays DTO-only (decision #3),
   confirmed in the same review.
9. **Two creation paths; the event is admin-grant-only for now (review I3).**
   The 2 default grants on user creation bypass the use-case entirely —
   `UsersService.create` (`users.service.ts:67-68`) and the user seeder
   (`user-seed.service.ts:140-147`) call the port / raw entity repo directly,
   so they neither run `assertGrantable` nor emit `LeaveAllocationGranted`.
   This is **intentionally out of scope**: routing them through the guard/event
   would make `UsersService` depend on `LeaveAllocationsService` and fire events
   mid-user-creation. We name the asymmetry explicitly so a future subscriber
   author knows `LeaveAllocationGranted` covers admin grants only, not the
   default 10+10. (Both paths still hit the DB `CHECK (amount > 0)`.)

## Dependency graph

```
value-object (AllocationAmount) + domain error                          [Task 1]
   └─> LeaveAllocationRecord + LeaveAllocation (guard) + Granted event   [Task 2]
         ├─> mapper (toDomain→record only) + port + repo return types    [Task 3]
         │      └─> LeaveAllocationsService (guard→persist→publish event) [Task 4]
         └─> LeaveAllocationResponseDto + assembler                       [Task 5]
                └─> AdminLeaveAllocationsController + module (publisher)   [Task 5]
                       └─> e2e + reconcile callers + wire-parity           [Task 6]
```

Built bottom-up, but each task lands its full layered slice so the module is
never half-converted (blueprint TL;DR §3: "a half-landed slice is worse than
none").

## Task List

### Phase 1 — Domain core (the consistency boundary)

#### Task 1: Value objects + domain errors
**Description:** Introduce the self-validating primitives the aggregate is built
from, plus the plain domain errors the use-case will map to HTTP.

**Acceptance criteria:**
- [ ] `AllocationAmount` throws on non-positive / non-integer; exposes `value`.
- [ ] `AllocationSource` throws on a value outside `default | admin_grant`.
- [ ] Errors are plain `Error` subclasses (`InvalidAllocationAmountError`,
      `InvalidAllocationSourceError`); zero `@nestjs/*` / `typeorm` /
      `class-validator` imports.

**Verification:**
- [ ] `npm test -- allocation-amount allocation-source` green (valid construct,
      invalid throws).
- [ ] `grep -rE "@nestjs|typeorm|class-validator" src/leave-allocations/domain` → no matches.

**Dependencies:** None.
**Files likely touched:** `domain/value-objects/allocation-amount.ts` (+`.spec`),
`domain/value-objects/allocation-source.ts` (+`.spec`),
`domain/leave-allocation-errors.ts`.
**Estimated scope:** Small.

#### Task 2: Data record rename + creation guard + event class
**Description:** Split the anemic class: `LeaveAllocationRecord` becomes the pure
persisted shape; `LeaveAllocation` becomes the thin aggregate root — a
value-object-backed creation guard (decision #2/#4) — plus the
`LeaveAllocationGranted` event class. No reconstitute, no event buffer.

**Acceptance criteria:**
- [ ] `domain/leave-allocation.ts` defines `LeaveAllocationRecord` — pure data,
      **no `@ApiProperty`**, snake_case, definite-assignment.
- [ ] `domain/leave-allocation.aggregate.ts` defines `LeaveAllocation` with the
      pure static guard `assertGrantable({ leave_type, amount, source })` that
      constructs `AllocationAmount` + `AllocationSource` (throwing the domain
      errors). **Does not** extend `AggregateRoot`, **no** `reconstitute`,
      **no** `pullEvents` (decision #2 — nothing is buffered).
- [ ] `domain/events/leave-allocation-events.ts` defines `LeaveAllocationGranted
      extends DomainEvent` (past tense; carries `allocation_id`, `employee_id`,
      `leave_type`, `amount`, `source`, `granted_by` — ids, not objects).
- [ ] (S4) The aggregate guard takes a small `GrantAllocationInput` shape;
      `domain/leave-allocation-inputs.ts`'s `CreateAllocationInput` stays as the
      **persistence** create input (the port's arg) — keep the two distinct.

**Verification:**
- [ ] `npm test -- leave-allocation.aggregate` green: `assertGrantable` passes
      on valid input and throws `InvalidAllocation*Error` on a bad amount /
      bad source (no DI, no DB).
- [ ] Domain folder still framework-free (grep as Task 1).

**Dependencies:** Task 1.
**Files likely touched:** `domain/leave-allocation.ts`,
`domain/leave-allocation.aggregate.ts`, `domain/events/leave-allocation-events.ts`,
`domain/leave-allocation.aggregate.spec.ts` (and a glance at
`domain/leave-allocation-inputs.ts`).
**Estimated scope:** Small–Medium.

### Checkpoint A — Domain core
- [ ] Aggregate + VO unit tests pass with **no DI and no DB**.
- [ ] `grep` confirms zero framework imports under `domain/`.
- [ ] Review before wiring the slice through.

### Phase 2 — Wire the slice (persistence → application → interface)

#### Task 3: Persistence layer
**Description:** Retype the mapper, port, and repo to the `LeaveAllocationRecord`
read shape. No `toAggregate` (decision #2 — no write-load path).

**Acceptance criteria:**
- [ ] `mapper.toDomain(entity): LeaveAllocationRecord` (explicit field copy) —
      **keep the existing field-assignment order** (S2: it drives JSON parity).
      **No** `mapper.toAggregate` (nothing reconstitutes the aggregate).
- [ ] `BaseLeaveAllocationRepository` + `LeaveAllocationRepository` return
      `LeaveAllocationRecord` from `create` / `listForEmployee`; sum methods
      unchanged; repo still returns records (never entities), never throws 404.
- [ ] (S1) `create` still stamps audit columns from the input
      (`created_by`/`updated_by` ← `created_by ?? granted_by`) — unchanged
      behavior; verify the retype didn't drop it.
- [ ] No new port methods added (decision #2 holds).

**Verification:**
- [ ] `npm test -- leave-allocation.mapper` green (type rename reflected).
- [ ] `npx tsc --noEmit` clean for the module.

**Dependencies:** Task 2.
**Files likely touched:** `persistence/mappers/leave-allocation.mapper.ts`
(+`.spec`), `persistence/base-leave-allocation.repository.ts`,
`persistence/repositories/leave-allocation.repository.ts`.
**Estimated scope:** Small–Medium.

#### Task 4: Application layer (thin use-case)
**Description:** Rewrite `grant` to the canonical shape — guard, persist,
publish — mapping domain errors to HTTP. `history` / `balancesFor` keep their
behavior.

**Acceptance criteria:**
- [ ] `grant`: `LeaveAllocation.assertGrantable({ leave_type, amount, source:
      admin_grant })` inside try/catch → map `InvalidAllocation*Error` to
      `unprocessable(field, msg)` (422); persist via the port; then construct
      `new LeaveAllocationGranted(created.id, …)` and `publisher.publish([...])`
      (I1 — event built post-commit with the real id, like
      `leave-requests.service.ts:214`).
- [ ] (S1) The persisted grant still stamps `granted_by` / `created_by =
      actor.id` (preserve the current behavior through the rewrite).
- [ ] Service depends on the **port** + `DomainEventPublisher`, owns the
      existing employee-not-found `NotFoundException`, has **no `if (admin)`
      branch**.
- [ ] `history` / `balancesFor` unchanged.

**Verification:**
- [ ] `npm test -- leave-allocations.service` green: mocks the **port** +
      publisher; asserts (a) the event is published with the persisted id on
      success, (b) `granted_by`/`created_by` stamping, (c) the existing "rejects
      grant to non-existent employee" 404 case still passes.
- [ ] (S5) The domain-error → 422 mapping is covered by calling the service with
      an invalid amount **directly** (over HTTP the DTO's `@Min(1)@Max(365)`
      rejects first — and this app maps validation errors to **422**, not 400,
      via `validation-options` `errorHttpStatusCode`, so both paths yield 422;
      the guard branch is only reachable in a unit test — cover it there so it
      isn't dead).

**Dependencies:** Task 3.
**Files likely touched:** `leave-allocations.service.ts`,
`leave-allocations.service.spec.ts`.
**Estimated scope:** Medium.

#### Task 5: Interface layer + module wiring
**Description:** Add the response DTO + assembler, return DTOs from the
controller, and provide the publisher in the module.

**Acceptance criteria:**
- [ ] `dto/response/leave-allocation-response.dto.ts` mirrors the **old**
      `LeaveAllocation` `@ApiProperty` fields verbatim (names, order,
      nullability).
- [ ] `leave-allocation.assembler.ts` maps `LeaveAllocationRecord` → DTO (single
      + list).
- [ ] `AdminLeaveAllocationsController` returns DTOs via the assembler;
      `@ApiResponse({ type: LeaveAllocationResponseDto })`; `/balances`
      untouched.
- [ ] (S3) The refactor preserves the route's gates/metadata verbatim:
      `@Permissions({ LEAVE_ALLOCATION: 'Create' | 'View' })`,
      `@ApiTags('Admin - Leave Allocations')`, `@ApiBearerAuth`, and the
      `@Controller({ path: 'admin/users/:id', version: API_VERSION })` mount —
      a controller rewrite is where an auth decorator silently disappears.
- [ ] `LeaveAllocationsModule` provides `DomainEventPublisher` (EventEmitter2 is
      global via `app.module`).

**Verification:**
- [ ] App boots (`npm run build`); Swagger renders the new response DTO.
- [ ] Manual: `POST` then `GET` history returns the same JSON shape as before.

**Dependencies:** Task 4.
**Files likely touched:** `dto/response/leave-allocation-response.dto.ts`,
`leave-allocation.assembler.ts`, `controllers/admin-leave-allocations.controller.ts`,
`leave-allocations.module.ts`.
**Estimated scope:** Medium.

### Checkpoint B — Slice wired
- [ ] `npm run lint && npm run test && npm run build` green.
- [ ] Swagger shows the response DTO; the domain no longer carries `@ApiProperty`.

### Phase 3 — Integration & wire-parity

#### Task 6: e2e coverage + reconcile callers + verify byte-parity
**Description:** Add the missing e2e (the byte-parity guard), confirm the port
return-type change ripples cleanly to external callers, then prove the wire is
unchanged end-to-end.

**Acceptance criteria:**
- [ ] New `test/leave-allocations.e2e-spec.ts`: admin grant happy-path (201 +
      expected JSON shape), grant history (newest-first list), and 422 on a bad
      amount. Seeds an employee + an admin with `LEAVE_ALLOCATION:*`.
- [ ] `UsersService.create`, `LeaveBalanceService`, `LeaveRequestsService`, and
      the user seeder all compile against the retyped port (none bind the
      `LeaveAllocation` class name; the seeder uses the raw entity). (I3) Confirm
      the 2 default grants still work and — by design — emit **no**
      `LeaveAllocationGranted`.
- [ ] Full backend gates green:
      `npm run lint && npm run test && npm run test:e2e && npm run build`
      (e2e with `SEED_DEFAULT_PASSWORD=Asima@1234` + `STORAGE_*` + MinIO).
- [ ] The blueprint §11 checklist is satisfied **except the load-path items
      consciously omitted in decision #2** (`findAggregateById` / `toAggregate`
      / `reconstitute` — no mutation use case). That deviation is recorded here,
      per §11's "if this guide and the code diverge, open a PR updating the doc."

**Verification:**
- [ ] The new e2e + existing e2e suite pass (byte-parity guard).

**Dependencies:** Task 5.
**Files likely touched:** `test/leave-allocations.e2e-spec.ts` (new);
otherwise verification only.
**Estimated scope:** Medium.

### Checkpoint C — Complete
- [ ] All acceptance criteria met; §11 checklist green.
- [ ] All gates pass; ready to commit straight to `main` (per CLAUDE.md git
      workflow) in `asima-backend`.

## Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| JSON drifts from today's shape (field order/nullability) | Med | Keep `mapper.toDomain` assignment order == legacy (S2); DTO mirrors `@ApiProperty`; e2e parity guard. |
| No `test/leave-allocations.e2e-spec.ts` exists → weaker parity net | Med | Add a thin e2e (grant + history + 422) — Task 6. |
| Audit stamping (`granted_by`/`created_by`) dropped in the rewrite | Med | S1: explicit AC in Tasks 3+4; service spec asserts it. |
| Auth/Swagger decorator dropped in the controller rewrite | Med | S3: explicit AC in Task 5 to preserve `@Permissions`/`@ApiTags`/mount. |
| Default-grant path diverges (no guard, no event) | Low | I3/decision #9: intentional + documented; both paths hit DB `CHECK`. |
| Over-engineering an append-only module | Low | I2/decision #2: thin guard-only aggregate; no reconstitute/toAggregate/buffer; lean port. |

## Resolved (2026-06-26 review)

- **Q1 — e2e coverage:** YES, add a thin `test/leave-allocations.e2e-spec.ts`
  (grant / history / 422). Folded into Task 6.
- **Q2 — amount cap:** keep `[1, 365]` DTO-only; not a domain invariant
  (decision #3).

### Five-axis review findings applied (2026-06-26)

- **I1 (Important, Correctness/Arch):** creation event was specified as both
  aggregate-recorded *and* "published with the persisted id" — contradictory
  (no id at factory time). Fixed → aggregate exposes a pure `assertGrantable`
  guard; the use-case builds + publishes `LeaveAllocationGranted(created.id,…)`
  post-commit, mirroring `leave-requests.service.ts:214`. (decision #4, Tasks 2/4)
- **I2 (Important, Arch):** `reconstitute` / `mapper.toAggregate` would be dead
  code (no mutation path), contradicting decision #2's own rationale. Fixed →
  dropped them and the `pullEvents` buffer; the aggregate is a thin guard and
  need not extend `AggregateRoot`. (decision #2, Tasks 2/3)
- **I3 (Important, Arch):** default grants bypass the guard/event; previously
  unstated. Fixed → named as decision #9 + Task 6 AC (event is admin-grant-only
  for now, by design).
- **S1:** preserve `granted_by`/`created_by` audit stamping through the rewrite
  (Tasks 3/4).
- **S2:** JSON key order comes from `mapper.toDomain`, not the DTO (decision #6,
  Tasks 3/5).
- **S3:** preserve `@Permissions`/`@ApiTags`/mount in the controller rewrite
  (Task 5).
- **S4:** ~~keep `CreateAllocationInput` distinct from the guard's
  `GrantAllocationInput`~~ — superseded by the post-implementation trim below;
  `GrantAllocationInput` was removed (the guard takes a bare `amount`).
- **S5:** unit-test the domain-error → 422 mapping directly (over HTTP the DTO
  rejects first — also 422, since this app maps validation errors to 422 via
  `validation-options`, not 400; verified in the e2e) (Task 4).
- **P1 (FYI, no action):** `listForEmployee` history is unpaginated, but
  pre-existing and per-employee counts are tiny.

### Post-implementation review — trim applied (2026-06-26)

A second five-axis review of the implemented code found the **`source`
validation machinery was dead on the only live path**: the use-case hardcodes
`source = admin_grant` (a constant valid value) and decision #9 keeps the
default-grant path off the guard, so `AllocationSource` never rejected anything
and the service's `InvalidAllocationSourceError → 422` branch was unreachable
and untested. It also created a name collision (`AllocationSource` the type vs
the VO class). **Trimmed:**

- Deleted the `AllocationSource` value object + spec and `InvalidAllocationSourceError`.
- Dropped `GrantAllocationInput`; the guard is now `assertGrantable(amount: number)`.
- Removed the `source` branch from the service's `rethrowDomainError`.

The amount invariant (the one rule with a domain home) stays. Gates re-run
green: typecheck, 418 unit tests, lint:ci, build, + the leave-allocations e2e.

### Third review — refinements applied (2026-06-26)

A five-axis review against the `leave-requests` exemplar surfaced three
follow-ups (all applied):

- **R1 (Arch) — the value object was ceremonial.** `assertGrantable` constructed
  `AllocationAmount` only to throw, then the use-case persisted the raw
  `input.amount`, so `AllocationAmount.value` was dead on every live path
  (unlike `leave-requests`, where VOs are held and read). Fixed →
  `assertGrantable(amount): AllocationAmount` now **returns** the validated VO
  and the service persists `.value`. Behavior identical (integer in = integer
  out); the VO is now load-bearing. Aggregate spec asserts the returned value.
- **R2 (Doc / missing detail) — blueprint not reconciled with the deviation.**
  The module deliberately omits `AggregateRoot`/`reconstitute`/`toAggregate`/
  `pullEvents`, but `module-architecture.md` §6 didn't name this shape, so it
  read as non-conformant. Added **§6.1 "The creation-only (append-only ledger)
  aggregate"** describing the variant and citing `leave-allocations`, per §11's
  "if the guide and the code diverge, open a PR updating the doc."
- **R3 (Dead code) — `sumByEmployeeAndType` had no callers.** Pre-existing on
  the port (the balance read uses `sumsByEmployee`; reserve uses
  `sumForUpdate`). Removed the abstract + impl (~12 lines).

The nits (guard-before-I/O ordering, `Object.assign` assembler) were left as-is
— immaterial over HTTP and guarded by the mapper-order spec respectively.

## Open questions

1. **Commit timing:** commit this snapshot now, or after the plan is approved?
