# Backend architecture: Domain-Driven Design

**Date:** 2026-06-25
**Scope:** `asima-parent` (living docs) + `asima-backend` (all feature modules).
The frozen `docs/plans/*.md` snapshots are **out of scope** — they are the
audit trail of features built earlier and are left untouched.
**Status:** planned, brainstormed (4 architecture decisions locked); not started.

---

## Overview

Asima's backend is built "hexagonal + anemic": domain classes are data
containers decorated with `@ApiProperty` (so the domain doubles as the HTTP
response), and every business rule lives in `~4,200 LOC` of services across 13
feature modules. This plan moves the backend to **Domain-Driven Design**:
behavior moves onto rich aggregates, the domain becomes framework-free, validated
primitives become Value Objects, invalid objects become unconstructable, and
aggregates raise domain events.

DDD is the **standing architecture** from here on — the documentation describes
it as the way Asima is built, not as a migration from something else. There is
no "we switched on date X" framing anywhere in the living docs.

The work is sequenced **foundation → pilot → phased rollout**: rewrite the docs
first, build a shared DDD kit, migrate one pilot aggregate (`LeaveRequest`)
end-to-end to validate the blueprint, then migrate the remaining aggregates one
per phase, each a full vertical slice that leaves the system green before the
next begins.

---

## Architecture Decisions (locked during brainstorming)

1. **Plan shape: foundation + pilot, then phased rest.** Docs first, then the
   shared kit, then `LeaveRequest` as the proving aggregate, then the rest
   enumerated as ordered phases. Rationale: the blueprint is validated on one
   aggregate before mass application; lowest risk of a half-migrated codebase.

2. **Wire boundary: response DTO + assembler per aggregate.** The aggregate is
   pure (zero `@nestjs/*`). A dedicated `*-response.dto.ts` (carrying
   `@ApiProperty`) holds the snake_case wire shape; an assembler maps
   aggregate → response DTO. Swagger contract and snake_case end-to-end are
   preserved. This is principle #2 made concrete.

3. **Repositories: keep the abstract ports, scope them to the aggregate root.**
   One `Base<Root>Repository` port per aggregate root (still the DI/mock seam
   the service specs depend on). Child entities load/save through the root, not
   via their own public repo. The mapper reconstitutes a rich aggregate via a
   static `reconstitute(...)` factory instead of `Object.assign`.

4. **Domain events: build the base + raise events from the pilot; no behavior
   change yet.** Aggregates buffer events; the use-case publishes them
   post-commit via `@nestjs/event-emitter`. The pilot raises its events but adds
   no new subscribers. `compensation`/`audit` (the one real cross-aggregate
   write) is left exactly as-is this round; its child-vs-event modelling is
   decided when Compensation migrates.

### What is explicitly KEPT (not changed by this migration)

- Snake_case end-to-end (DB column, domain field, JSON payload).
- The admin vs. self-service split at the **edge** (audience-split request DTOs,
  `admin-*` / `me-*` controllers, permission gates, `req.user.id`-only `/me`).
- `@/` path alias; pagination shape `{ data, total, page, limit, has_more }`.
- Database-migration conventions (one table = one CREATE migration, etc.) — they
  are architecture-agnostic and only get a light terminology pass.
- Partial-unique-index DB invariants (one open punch, one active schedule, …)
  remain the source of truth for concurrency; aggregates do not replace them.

---

## The target blueprint (what every module becomes)

```
src/<feature>/
├── domain/
│   ├── <root>.aggregate.ts            # Rich aggregate ROOT. Pure TS. Behavior lives here.
│   ├── <child>.entity.ts              # Child entities — loaded/saved via the root only
│   ├── value-objects/*.ts             # Self-validating, immutable VOs
│   ├── events/*.ts                    # Past-tense domain events the root raises
│   └── <root>-search-criteria.ts      # (kept) filter shape for queries
├── application/
│   └── <feature>.service.ts           # Thin use-case: load → call aggregate → save → publish
├── persistence/
│   ├── base-<root>.repository.ts      # (kept) abstract port — one per aggregate ROOT
│   ├── entities/*.entity.ts           # TypeORM rows (root + children)
│   ├── mappers/<root>.mapper.ts       # toPersistence + reconstitute (rebuild rich aggregate)
│   └── persistence.module.ts
├── dto/
│   ├── admin/  me/                     # (kept) request DTOs, audience-split
│   └── response/<resource>-response.dto.ts   # @ApiProperty lives HERE, not the domain
├── <resource>.assembler.ts            # aggregate → response DTO
└── controllers/ (admin- / me-)        # (kept) edge split, permission gates
```

**The five hard rules** (these replace the current "anemic by design" rule):

1. **Domain is pure.** Zero `@nestjs/*` (no more `@ApiProperty` on the domain),
   zero `typeorm`, zero `class-validator`. The aggregate imports only other
   domain code.
2. **Behavior lives on the aggregate**, not the service. `leaveRequest.approve(caller)`
   performs the state transition and checks invariants.
3. **Invalid objects cannot be constructed.** A private constructor + static
   factory (`submit(...)`) enforces creation invariants; a static
   `reconstitute(...)` rebuilds from persisted state (already valid) — this
   replaces `Object.assign`.
4. **Validated primitives are Value Objects** — immutable, self-validating on
   construction.
5. **Aggregates raise events; the use-case publishes them** after the save
   commits.

---

## The aggregate map (the 80% — boundaries)

| Aggregate root | Children (via root) | Referenced by ID | Key Value Objects | Events |
|---|---|---|---|---|
| **LeaveRequest** ⭐pilot | — | employee_id, l1/l2_approver_id, attachment_id | `DateRange`, `LeaveDuration` (0.5 steps), `HalfDayWindow`, `LeaveRequestStatus` (legal transitions), `DecisionPath` | Submitted, Approved, AdvancedToL2, Rejected, Cancelled |
| **TimeEntry** | — | employee_id | `PunchWindow` (out>in), `TimeEntryStatus`, `TimeSource` | Punched, PunchClosed |
| **WorkSchedule** | — | employee_id | `SlotRange` (expected_in/out), `DayOfWeek`, `BreakMinutes`, `EffectivePeriod` | Scheduled, LogicallyEnded |
| **ApprovalChain** | chain steps | employee_id, approver_id | `ChainStep`, `EffectivePeriod` | Assigned, Reassigned, Ended |
| **TimeCorrectionRequest** | — | time_entry_id, approver ids | status VO, `PunchWindow` | mirrors LeaveRequest |
| **LeaveAllocation** | — | employee_id | `Days`, `LeaveType`, `Year` | Granted |
| **User** | — | role_id | `Email`, `PersonName`, `PasswordHash` | (later) |
| **Role** | permission grants | permission ids | `RoleCode` | (later) |
| **Compensation** | CompensationAudit (deferred) | employee_id | `Money`/`PayRate`, `PayType`, `EffectivePeriod` | (deferred) |

**Not aggregates** (documented as such, treated accordingly):

- **Permission** — seed-managed *reference data*. Stays a simple read catalog.
- **Approvals** — a cross-aggregate **query/read service** (the inbox over leave
  + time-correction). No aggregate; it composes other aggregates' read models.
- **Storage** — infrastructure adapter (MinIO). Attachments referenced by ID.
- **Auth** — application service over the `User` aggregate (login, token rotate).
- **LeaveBalance** — a **derived read-model** (computed from approved/pending
  working-day sums + the allocation ledger). Never written; never an aggregate.

---

## Task List

### Phase 0: Documentation rewrite (DDD-native) — pure docs, no code

> Living docs only. The `docs/plans/*.md` snapshots are the frozen audit trail and
> are NOT edited. No "moved from hexagonal" / dated-switch phrasing anywhere.

#### Task 0.1: Rewrite `module-architecture.md` as the DDD blueprint
**Description:** Replace the hexagonal/anemic blueprint with the DDD blueprint
above: the file tree, the five hard rules, aggregate/VO/event/reconstitution
templates, the response-DTO + assembler convention, the per-root repository port,
and a new-aggregate checklist. Remove the "anemic by design" rule and the
"rich domain methods" anti-pattern (it is now the *required* pattern). Keep the
admin/me edge split, snake_case, pagination, and migration cross-references.
**Acceptance criteria:**
- [ ] Document describes aggregates, VOs, events, reconstitution, response DTOs as the standard; no "anemic" guidance remains.
- [ ] No `hexagonal` token and no "port pattern as the architecture" framing; "abstract repository port" survives only as a DI/testing seam description.
- [ ] New-aggregate checklist reflects the five hard rules + the kept conventions.
**Verification:**
- [ ] `grep -ni 'hexagonal\|anemic' docs/universal-guidelines/module-architecture.md` → no matches.
- [ ] Manual read: a new contributor could build a fresh aggregate from this doc alone.
**Dependencies:** None. **Files:** `docs/universal-guidelines/module-architecture.md`. **Scope:** L (single doc, full rewrite).

#### Task 0.2: Light terminology pass on `database-migration-conventions.md`
**Description:** The DB-migration rules are architecture-agnostic and stay. Update
only the wording that ties audit columns / domain field names to the anemic
model — point the audit-column cluster at the new shared `AuditStamp` VO and the
reconstitution boundary.
**Acceptance criteria:**
- [ ] Audit-columns section references `AuditStamp` (mapped at the persistence boundary), not an anemic domain class.
- [ ] All migration rules (one CREATE per table, naming, FK actions, partial indexes) are unchanged in substance.
**Verification:** Manual read; `grep -ni 'anemic\|hexagonal'` → no matches.
**Dependencies:** None. **Files:** `docs/universal-guidelines/database-migration-conventions.md`. **Scope:** S.

#### Task 0.3: Vocabulary swap in `frontend-architecture.md`
**Description:** The frontend deliberately does NOT mirror the backend's layering.
Swap the "reject **hexagonal**" contrast to "do not mirror the backend's **DDD**
layering." FSD content and rationale unchanged — vocabulary only.
**Acceptance criteria:**
- [ ] No `hexagonal` token; the contrast now names "the backend's DDD layering."
- [ ] FSD decision, the "two ideas that do pay off" list, and the no-port rule read identically in substance.
**Verification:** `grep -ni 'hexagonal' docs/universal-guidelines/frontend-architecture.md` → no matches.
**Dependencies:** None. **Files:** `docs/universal-guidelines/frontend-architecture.md`. **Scope:** S.

#### Task 0.4: Parent `CLAUDE.md` + `README.md`
**Description:** Replace the three `hexagonal` references in `CLAUDE.md`
("hexagonal layering blueprint") and the two in `README.md` ("Hexagonal per
module", "hexagonal layout on the backend") with DDD phrasing. The cross-cutting
concepts (Role/Title/Approval-chain, permission codes, admin/me contract) are
unchanged.
**Acceptance criteria:**
- [ ] Both files describe the backend as DDD; no `hexagonal` token in either.
- [ ] Doc-pointer lines point at the rewritten `module-architecture.md` (now "the DDD blueprint").
**Verification:** `grep -ni 'hexagonal' CLAUDE.md README.md` → no matches.
**Dependencies:** 0.1 (so pointers describe it correctly). **Files:** `CLAUDE.md`, `README.md`. **Scope:** S.

#### Task 0.5: `asima-backend/CLAUDE.md` + `asima-backend/README.md`
**Description:** Heaviest of the sub-repo doc edits. In `CLAUDE.md`: line 3
tagline, the `## Hexagonal layout (non-negotiable)` section → `## DDD layout`,
the "Service depends on the abstract `Base*Repository`" rule reframed (port is a
DI/test seam, not the architecture's defining trait), the testing section, the
time-tracking "standard hexagonal layout" line, and the `module-architecture.md`
pointer. In `README.md`: 4 hexagonal references.
**Acceptance criteria:**
- [ ] No `hexagonal` token in either backend file; layout section describes the DDD building blocks.
- [ ] The kept conventions (guards pipeline, snake_case, migrations, throttler tiers, swagger groups) are unchanged.
- [ ] "Reference exemplar" pointer updates from `permissions` to the new pilot (`leave-requests`) once Phase 2 lands — leave a TODO note here until then.
**Verification:** `grep -ni 'hexagonal' asima-backend/CLAUDE.md asima-backend/README.md` → no matches.
**Dependencies:** 0.1. **Files:** `asima-backend/CLAUDE.md`, `asima-backend/README.md`. **Scope:** M.

#### Task 0.6: `asima-frontend/CLAUDE.md`
**Description:** Swap the two contrast references ("Feature-Sliced (NOT
hexagonal)" and "not mirror the backend's hexagonal domain/ports/adapters
layering") to name the backend's DDD layering. FSD rules unchanged.
**Acceptance criteria:**
- [ ] No `hexagonal` token; contrast names "the backend's DDD layering."
**Verification:** `grep -ni 'hexagonal' asima-frontend/CLAUDE.md` → no matches.
**Dependencies:** None. **Files:** `asima-frontend/CLAUDE.md`. **Scope:** S.

#### ✅ Checkpoint: Documentation
- [ ] `grep -rni 'hexagonal\|anemic' CLAUDE.md README.md docs/universal-guidelines asima-backend/CLAUDE.md asima-backend/README.md asima-frontend/CLAUDE.md` → no matches.
- [ ] `docs/plans/*.md` snapshots are byte-for-byte unchanged (frozen audit trail).
- [ ] Living docs are internally consistent: every cross-reference resolves and uses DDD vocabulary.
- [ ] Commit per repo (`asima-parent` for parent docs; `asima-frontend` for its CLAUDE.md; backend doc edits live in the parent under `docs/` + the backend's own `CLAUDE.md`/`README.md` in the backend repo). **Review with human before Phase 1.**

---

### Phase 1: Shared DDD kit (foundation) — no feature behavior changes

#### Task 1.1: Add `@nestjs/event-emitter`, register it, add the publisher helper
**Description:** `@nestjs/event-emitter` is not currently a dependency. Add it,
register `EventEmitterModule.forRoot()` in `app.module.ts`, and add a small
publisher helper the use-cases call to drain an aggregate's buffered events and
emit them **after** the persistence save resolves.
**Acceptance criteria:**
- [ ] `@nestjs/event-emitter` in `package.json` + lockfile; app boots with the module registered.
- [ ] A `DomainEventPublisher` (or equivalent) publishes a list of events post-save; covered by a unit test.
**Verification:**
- [ ] `npm ci && npm run build` clean; `npm run test` green.
- [ ] App starts (`npm run start:dev` boots without DI errors).
**Dependencies:** None. **Files:** `package.json`, `package-lock.json`, `app.module.ts`, `src/utils/domain/event-publisher.ts` (+ spec). **Scope:** S.

#### Task 1.2: `AggregateRoot` base + `DomainEvent` marker
**Description:** Base class giving every aggregate a private event buffer with
`protected recordEvent(e)` and `pullEvents(): DomainEvent[]` (clears on read).
`DomainEvent` is a marker interface/abstract with `occurred_at`.
**Acceptance criteria:**
- [ ] `recordEvent` buffers; `pullEvents` returns then clears; both unit-tested.
- [ ] Base class is pure TS (no `@nestjs/*`, no `typeorm`).
**Verification:** `npm run test` green for the base specs.
**Dependencies:** None. **Files:** `src/utils/domain/aggregate-root.ts`, `src/utils/domain/domain-event.ts` (+ specs). **Scope:** S.

#### Task 1.3: `ValueObject` base + `AuditStamp` VO
**Description:** `ValueObject` base providing structural `equals()` and
construction-time freeze. `AuditStamp` VO wraps the `created_by/at`,
`updated_by/at`, `deleted_by/at` cluster every aggregate carries, with an
`isDeleted()` helper.
**Acceptance criteria:**
- [ ] VO base freezes instances; `equals()` is structural; both unit-tested.
- [ ] `AuditStamp` constructs from persisted values and exposes the audit fields read-only.
**Verification:** `npm run test` green.
**Dependencies:** None. **Files:** `src/utils/domain/value-object.ts`, `src/utils/domain/audit-stamp.ts` (+ specs). **Scope:** S.

#### Task 1.4: Shared cross-aggregate VOs — `DateRange`, `Money`
**Description:** `DateRange` (inclusive `start`/`end`, `end >= start`, day-count
helper) used by leave/schedule/approval-chain/compensation; `Money`/`PayRate`
(non-negative, currency-aware) for compensation. Both immutable, self-validating,
throwing a domain error on invalid construction.
**Acceptance criteria:**
- [ ] Constructing an invalid `DateRange` (`end < start`) or negative `Money` throws; valid ones succeed; covered by unit tests.
- [ ] Pure TS; no framework imports.
**Verification:** `npm run test` green.
**Dependencies:** 1.3. **Files:** `src/utils/domain/value-objects/date-range.ts`, `.../money.ts` (+ specs). **Scope:** S.

#### Task 1.5: Reconstitution + response-DTO/assembler convention + domain-purity lint guard
**Description:** Establish the repeatable seams the pilot will follow: (a) the
`reconstitute(props)` static-factory pattern (documented + a tiny reference
example), (b) the `dto/response/*.dto.ts` + `*.assembler.ts` convention, and
(c) an ESLint `no-restricted-imports` rule scoped to `**/domain/**` forbidding
`@nestjs/*`, `typeorm`, and `class-validator`, so domain purity is CI-enforced.
**Acceptance criteria:**
- [ ] ESLint rule fails on a deliberately-dirty domain import and passes on clean domain code.
- [ ] Convention is documented in `module-architecture.md` (already rewritten in 0.1) and matches what the rule enforces.
**Verification:**
- [ ] `npm run lint` green on current (still-anemic) code OR the rule is scoped so it activates per-folder as aggregates migrate (decide in-task; prefer activating now on the `utils/domain` kit, broadening per phase).
- [ ] `npm run build` clean.
**Dependencies:** 0.1, 1.2, 1.3. **Files:** `.eslintrc`/eslint config, `src/utils/domain/*` reference example. **Scope:** S–M.

#### ✅ Checkpoint: Foundation
- [ ] `npm run lint && npm run test && npm run build` all green.
- [ ] App boots; no feature module changed; no endpoint behavior changed.
- [ ] The kit is documented in `module-architecture.md`. **Review with human before Phase 2.**

---

### Phase 2: Pilot — migrate `LeaveRequest` to a rich aggregate end-to-end

> Proves the whole blueprint on the richest state machine. **Behavior must be
> identical** — same routes, same JSON, same status transitions, same errors.
> Phases 3+ replicate this internal task pattern.

#### Task 2.1: LeaveRequest Value Objects
**Description:** `LeaveDuration` (0.5 increments, full=whole days, half=0.5,
single-day rule for halves), `HalfDayWindow` (SlotRange `start_time < end_time`,
null for full day), `LeaveRequestStatus` (enumerates legal transitions:
pending_l1→pending_l2/approved/rejected/cancelled, etc.), `DecisionPath`. Reuse
`DateRange` from 1.4 for `[start_date, end_date]`.
**Acceptance criteria:**
- [ ] Illegal status transitions and invalid half-day windows throw at construction; covered by unit tests.
- [ ] VOs are pure; values serialize back to the existing snake_case primitives.
**Verification:** `npm run test` green for the VO specs.
**Dependencies:** Phase 1. **Files:** `src/leave-requests/domain/value-objects/*` (+ specs). **Scope:** M.

#### Task 2.2: `LeaveRequest` aggregate (behavior + events)
**Description:** Convert `domain/leave-request.ts` to `LeaveRequest` extending
`AggregateRoot`: private constructor, static `submit(input)` factory enforcing
all creation invariants currently in `LeaveRequestsService.submit`, static
`reconstitute(props)`, and behavior methods `approve(caller)`, `reject(caller, note)`,
`cancel(caller)`, plus the L1→L2 advance and override logic — each performing its
state transition and recording the matching event (Submitted/Approved/
AdvancedToL2/Rejected/Cancelled). Remove `@ApiProperty`.
**Acceptance criteria:**
- [ ] All state-transition rules currently in the service (pending checks, approver checks, override behavior, single-step auto-approve) live on the aggregate and are unit-tested without any DI.
- [ ] `approve`/`reject`/`cancel` record the correct events; `pullEvents` returns them.
- [ ] Domain folder passes the purity lint rule (zero `@nestjs/*`/`typeorm`).
**Verification:** `npm run test` green; lint green.
**Dependencies:** 2.1. **Files:** `src/leave-requests/domain/leave-request.aggregate.ts` (+ spec). **Scope:** M.

#### Task 2.3: Persistence — reconstitute + save the aggregate through the port
**Description:** Update the mapper to `reconstitute` a `LeaveRequest` aggregate
(building VOs) and to `toPersistence` from it; the concrete repo (behind the kept
`BaseLeaveRequestRepository` port) returns aggregates and persists transitions.
No schema change.
**Acceptance criteria:**
- [ ] Repo returns `LeaveRequest` aggregates; mapper builds VOs and fails fast on missing required relations.
- [ ] No TypeORM types leak past the persistence boundary.
**Verification:** `npm run test` (mapper/repo specs) green.
**Dependencies:** 2.2. **Files:** `src/leave-requests/persistence/mappers/*`, `.../repositories/*`, `base-leave-request.repository.ts` (+ specs). **Scope:** M.

#### Task 2.4: Response DTO + assembler; controllers return the DTO
**Description:** Add `dto/response/leave-request-response.dto.ts` (the
`@ApiProperty` shape that used to live on the domain — identical fields/order)
and `leave-request.assembler.ts` (`toResponse(aggregate)`). Controllers
(`admin-`, `me-`, base, balances) return response DTOs. Update the Swagger
schema-group registration in `main.ts`.
**Acceptance criteria:**
- [ ] Every leave endpoint returns the response DTO; the emitted JSON is byte-identical to today's (snake_case, same fields).
- [ ] Swagger schema for leave resources is present and grouped; no orphaned `$ref`.
**Verification:**
- [ ] Diff a sample response against the pre-migration shape (manual or snapshot test).
- [ ] `npm run build` clean; `/docs` renders the leave schemas.
**Dependencies:** 2.2. **Files:** `src/leave-requests/dto/response/*`, `src/leave-requests/leave-request.assembler.ts`, controllers, `main.ts`. **Scope:** M.

#### Task 2.5: Thin use-case service + publish events
**Description:** Reduce `LeaveRequestsService` (and the balance/day-count
services as needed) to orchestration: load aggregate → call behavior → save →
`publisher.publish(aggregate.pullEvents())`. No new subscribers. Update
`leave-requests.service.spec.ts` to the new shape (still mocking the abstract
repo port).
**Acceptance criteria:**
- [ ] Service methods contain no business rules — only load/call/save/publish + 404 translation.
- [ ] Events are published post-save; no subscriber changes behavior.
- [ ] All existing leave unit + e2e tests pass unchanged in observable behavior.
**Verification:**
- [ ] `npm run test` green; `npm run test:e2e` green (run with the CI seed password, `STORAGE_*` env set, MinIO container up — leave attachments touch storage).
- [ ] `npm run lint && npm run build` green.
**Dependencies:** 2.3, 2.4. **Files:** `src/leave-requests/application/*.service.ts` (+ specs). **Scope:** M.

#### ✅ Checkpoint: Pilot validated
- [ ] Full backend suite green: `npm run lint && npm run test && npm run test:e2e && npm run build`.
- [ ] Leave endpoints' JSON + status transitions + error shapes are unchanged (verified by diff/snapshot).
- [ ] `leave-requests/domain` passes the purity lint rule.
- [ ] Update the "reference exemplar" pointer in `asima-backend/CLAUDE.md` from `permissions` to `leave-requests` (closes the Task 0.5 TODO).
- [ ] **Review with human. This blueprint is the template for Phases 3+.** Adjust the doc/blueprint if the pilot surfaced friction before replicating it 8×.

---

### Phases 3–N: Remaining aggregates (one aggregate per phase)

Each phase replicates the Phase 2 task pattern: **(a) Value Objects → (b)
aggregate (behavior + events) → (c) persistence reconstitution → (d) response
DTO + assembler → (e) thin use-case service**, each with the same acceptance
criteria (behavior identical, domain pure, tests green) and a closing checkpoint
that runs the full suite before the next phase starts. Ordered by risk/coupling:

- **Phase 3 — TimeEntry.** VOs: `PunchWindow` (out>in), `TimeEntryStatus`,
  `TimeSource`. Factory `punchIn()` / behavior `punchOut()`. The "one open punch
  per employee" partial unique index stays the concurrency source of truth; the
  service still maps `23505`→409. Events: Punched, PunchClosed. **Scope:** M.
- **Phase 4 — WorkSchedule.** VOs: `SlotRange` (expected_in/out), `DayOfWeek`,
  `BreakMinutes`, `EffectivePeriod`. Behavior: `endLogically()` + the
  schedule-change cascade. Active-row partial index kept. **Scope:** M.
- **Phase 5 — ApprovalChain.** Effective-dated assignment with logical-end
  (`ended_at`); behavior `reassign()` / `endLogically()`. **Scope:** M.
- **Phase 6 — TimeCorrectionRequest.** Mirrors the LeaveRequest chain/state
  machine; reuse the status/decision VOs pattern. **Scope:** M.
- **Phase 7 — LeaveAllocation.** Grant ledger aggregate. `LeaveBalance` stays a
  derived read-model (query service) — confirm it reads cleanly off the migrated
  aggregates and is NOT turned into an aggregate. **Scope:** S–M.
- **Phase 8 — User.** VOs: `Email`, `PersonName`, `PasswordHash`. Behavior:
  `deactivate()`, `changeEmail()`, `assignRole()`. Password rotation stays its
  own use-case/endpoint. `users_email_lower_uq` index kept. **Scope:** M.
- **Phase 9 — Role.** Permission grants as child collection on the root;
  built-in-role guard becomes an aggregate invariant. Permissions remain
  reference data. **Scope:** S–M.
- **Phase 10 — Compensation.** VOs: `Money`/`PayRate`, `PayType`,
  `EffectivePeriod`. **Decision point (deferred from brainstorming):** model
  `CompensationAudit` as an append-only **child of the Compensation aggregate**
  (one aggregate, one transaction — drop the second public repo) vs. an
  event-driven subscriber. Recommend the child route for an append-only trail
  that must never drift; record the choice in `module-architecture.md` when
  taken. **Scope:** M.
- **Phase 11 — Non-aggregate sweep.** Confirm and document that **Permission**
  (reference data), **Approvals** (cross-aggregate query service), **Storage**
  (infra adapter), and **Auth** (application service over `User`) need no
  aggregate. Ensure each consumes the migrated pure domain + response DTOs where
  it returns leave/time/etc. data. Remove any lingering anemic-domain `@ApiProperty`
  output classes. **Scope:** S–M.

#### ✅ Checkpoint: Migration complete
- [ ] Every `*/domain/**` folder passes the purity lint rule (rule now repo-wide).
- [ ] `npm run lint && npm run test && npm run test:e2e && npm run build` all green.
- [ ] No service contains business rules that belong on an aggregate (spot-check the largest services).
- [ ] All endpoints' JSON contracts unchanged vs. pre-migration.
- [ ] `module-architecture.md` matches the shipped code; the Compensation audit decision is recorded.

---

## Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Response-DTO migration silently changes JSON shape (field order, nulls) | High | Snapshot/diff a sample response per resource against pre-migration output in each phase's checkpoint. |
| Half-migrated codebase mixes anemic + rich aggregates mid-rollout | Med | One aggregate per phase, full vertical slice, full suite green before the next. Purity lint rule broadens per phase, not all at once. |
| e2e needs MinIO + `STORAGE_*` + CI seed password (leave attachments) | Med | Bake the env/container requirement into Phase 2's verification; reuse for later phases. |
| Domain-event publish timing (before vs. after commit) causes phantom side effects | Med | Publish strictly post-save; no behavior-changing subscribers this round (decision #4). |
| Pilot blueprint has friction discovered only at scale | Med | The Phase 2 checkpoint explicitly revisits the doc/blueprint before replicating 8×. |
| Lockfile drift when adding `@nestjs/event-emitter` (CI on Linux) | Low | `npm ci` after the dependency add; verify CI is green at the Phase 1 checkpoint. |

## Open Questions

- **Compensation audit modelling (Phase 10):** child entity inside the
  Compensation aggregate vs. event-driven subscriber. Deferred to that phase;
  recommendation is the child route. Confirm with human at Phase 10.
- **Purity lint activation cadence (Task 1.5):** activate the `no-restricted-imports`
  rule on `utils/domain` immediately and broaden the glob per migrated module, or
  ship it repo-wide once at the final checkpoint? Default: broaden per phase.

## Parallelization

- **Phase 0 (docs)** can run alongside **Phase 1 (kit)** — independent surfaces.
- Within Phase 0, tasks 0.2/0.3/0.6 are independent; 0.4/0.5 depend on 0.1.
- **Phases 3–10** are largely independent once Phase 1 + Phase 2 land (they share
  only the kit + cross-aggregate VOs). They can be parallelized across sessions,
  but each must still pass its own checkpoint before merge to keep `main` green.
- **Sequential:** anything sharing the eslint config / `main.ts` schema-groups
  (coordinate edits) and the dependency-add in 1.1.
