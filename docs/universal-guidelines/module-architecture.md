# Module Architecture — Guide for Creating a New Backend Module

This is the **authoritative blueprint** for adding any feature module to
`asima-backend`. Asima's backend is **Domain-Driven Design**: business rules
live on rich aggregates, the domain is framework-free, validated primitives are
Value Objects, invalid objects cannot be constructed, and aggregates raise
domain events for side effects.

The **reference exemplar is `src/leave-requests/`** — a leave request is a rich
aggregate with a real state machine (submit → pending_l1 → pending_l2 →
approved / rejected / cancelled), value objects, domain events, a response DTO,
and a thin use-case service. When unsure how a piece fits, mirror what
`leave-requests` does.

> **TL;DR — when creating an aggregate module called `<feature>` (plural,
> kebab-case path; singular PascalCase aggregate name):**
> 1. Scaffold the file tree in §2.
> 2. Get the **aggregate boundary** right first (§6) — this is 80% of the work.
> 3. Build the slice together: value objects → aggregate (+events) → repository
>    port + mapper (reconstitution) → response DTO + assembler → thin use-case
>    service → controller(s) → module. A half-landed slice is worse than none.
> 4. Audience (admin vs. self-service) is enforced at the **edge** (request DTO
>    + controller), never in the aggregate or service.
> 5. Run the §11 checklist before committing.

---

## 1. The building blocks

Dependencies point inward. The domain knows nothing about NestJS, TypeORM, or
HTTP. Everything outside depends on the domain; the domain depends on nothing.

```
┌─────────────────────────────────────────────────────────────┐
│ Interface (HTTP)                                              │
│   controllers/*.controller.ts   — admin- / me- / approver    │
│   dto/response/*.dto.ts          — the @ApiProperty wire shape│
│   <resource>.assembler.ts        — aggregate/read-model → DTO │
├─────────────────────────────────────────────────────────────┤
│ Application (use cases)                                       │
│   <feature>.service.ts           — load → behave → save →     │
│                                     publish; maps domain errors│
│   dto/admin/*  dto/me/*          — class-validator request DTOs│
├─────────────────────────────────────────────────────────────┤
│ Domain (PURE TS — no @nestjs/*, no typeorm, no class-validator)│
│   <root>.aggregate.ts            — aggregate root + behavior  │
│   value-objects/*.ts             — self-validating VOs        │
│   events/*.ts                    — past-tense domain events   │
│   <root>-errors.ts               — pure domain errors         │
│   <root>.ts                      — persisted data record      │
│   <root>-search-criteria.ts      — read filter shape          │
├─────────────────────────────────────────────────────────────┤
│ Infrastructure (persistence)                                  │
│   persistence/base-<root>.repository.ts   — ABSTRACT port     │
│   persistence/repositories/<root>.repository.ts               │
│   persistence/entities/<root>.entity.ts   — TypeORM @Entity   │
│   persistence/mappers/<root>.mapper.ts    — toAggregate /     │
│                                             toDomain          │
│   persistence/persistence.module.ts       — binds port→impl   │
└─────────────────────────────────────────────────────────────┘
```

Two rules hold all of this together:

1. **The service depends on the abstract `Base<Root>Repository`** (the port),
   not the concrete repository. Tests mock the port — that's the DI seam.
2. **The aggregate is pure.** It imports only other domain code. This is what
   makes the state machine unit-testable with no DI graph and no database.

The shared DDD kit lives in `src/utils/domain/`: `AggregateRoot`,
`DomainEvent`, `DomainEventPublisher`, `ValueObject`, `AuditStamp`, and shared
value objects (`value-objects/date-range.ts`, …).

---

## 2. Required file layout

```
src/<feature>/
├── domain/                                   # PURE TS — no @nestjs/* runtime, no typeorm, no class-validator.
│   ├── <root>.aggregate.ts                   # Aggregate ROOT: private ctor + static factory + reconstitute + behavior.
│   ├── <root>.ts                             # Persisted data record `<Root>Record` (reconstitution input + read-path shape).
│   ├── value-objects/
│   │   └── <vo>.ts                           # Self-validating, immutable value objects.
│   ├── events/
│   │   └── <feature>-events.ts               # Past-tense domain events the aggregate raises.
│   ├── <root>-errors.ts                      # Pure domain errors the use-case maps to HTTP.
│   ├── <root>-actor.ts                       # Caller-capability VO (distilled from User), if behavior needs authz.
│   ├── <root>-search-criteria.ts             # Filter shape consumed by repo.findAll().
│   └── find-all-<root>.ts                    # PaginatedResponse<…> alias (over the read-model).
├── persistence/
│   ├── base-<root>.repository.ts             # ABSTRACT CLASS — the port the service depends on.
│   ├── entities/<root>.entity.ts             # TypeORM @Entity. Extends EntityHelper.
│   ├── mappers/<root>.mapper.ts              # toAggregate (write load) / toDomain (read) / toListItem.
│   ├── repositories/<root>.repository.ts     # Concrete impl. Extends Base*Repository.
│   └── persistence.module.ts                 # TypeOrmModule.forFeature + port→impl binding.
├── dto/
│   ├── admin/                                # Wide request field set. admin-* controllers only.
│   ├── me/                                   # Narrow request field set. me-* controllers only.
│   └── response/<resource>-response.dto.ts   # @ApiProperty wire shape. The ONLY place Swagger sees the domain.
├── controllers/
│   ├── admin-<feature>.controller.ts         # /admin/<feature>. @Permissions gated.
│   └── me-<feature>.controller.ts            # /<feature>/me OR /users/me/<sub>. JWT-only.
├── <resource>.assembler.ts                   # aggregate / read-model → response DTO.
├── <feature>.service.ts                      # Thin use-case orchestrator.
├── <feature>.module.ts                       # Composes everything.
├── <feature>.constants.ts                    # Stable enums, protected names, regexes (optional).
└── <root>.aggregate.spec.ts                  # Pure unit tests of the aggregate behavior.
```

Skip `dto/me/` + `me-*.controller.ts` only when the resource truly has **no
self-service surface** (rare — `roles` is the only current example).

Path alias: use `@/` (resolves to `src/`). Never write `../../../`.

---

## 3. Layer-by-layer rules and templates

### 3.1 Domain layer — the aggregate

**Hard rules:**

1. **Zero framework imports.** No `@nestjs/*` (not even `@ApiProperty`), no
   `typeorm`, no `class-validator`. The domain imports only other domain code
   and the shared kit in `@/utils/domain`. CI/grep this: a domain folder with a
   framework import is a bug.
2. **Behavior lives on the aggregate**, not the service. The state transitions,
   guards, and derived state are methods on the root.
3. **Invalid objects cannot be constructed.** A `private constructor` plus a
   static factory (`submit(...)`) that enforces creation invariants, and a
   static `reconstitute(...)` that rebuilds from persisted (already-valid)
   state. `Object.assign` into the aggregate is forbidden — it bypasses
   construction.
4. **The aggregate references other aggregates by ID**, never by object. Inside
   `LeaveRequest` you hold `employee_id: number` and `l1_approver_id: number`,
   not a `User`.
5. **The aggregate raises events; it never publishes them.** It records events
   into the `AggregateRoot` buffer; the use-case drains and publishes them
   post-commit.

**Aggregate root template** (see `leave-requests/domain/leave-request.aggregate.ts`):

```ts
import { AggregateRoot } from '@/utils/domain/aggregate-root';
// value objects, events, errors, the actor capability type …

// The reconstitution input IS the persisted record — alias it, don't duplicate the field list.
export type <Root>Props = <Root>Record;

export class <Root> extends AggregateRoot {
  // value-object view of the persisted state
  private readonly _range: DateRange;
  // mutable transition state …

  private constructor(private readonly p: <Root>Props) {
    super();
    // build + validate value objects from props (invalid state impossible)
    this._range = new DateRange(p.start_date, p.end_date);
  }

  /** Creation factory — enforces the invariants of a brand-new aggregate. */
  static create(input: Create<Root>Input): <Root> { /* validate → new <Root>(…) → recordEvent */ }

  /** Rebuild a valid aggregate from persisted state (already validated). */
  static reconstitute(props: <Root>Props): <Root> { return new <Root>(props); }

  // read accessors (unwrap value objects to primitives) …

  // behavior — each transition guards, mutates, and records an event:
  approve(actor: <Root>Actor): void { /* guard → mutate → this.recordEvent(new …Approved(…)) */ }
}
```

> **Creation factory vs. I/O-bound creation.** `static create(...)` is the
> ideal for an aggregate you can construct in memory and then persist. When
> creation is transaction- and insert-id-bound (the row's id is DB-generated,
> the write spans a balance check + an attachment row in one transaction — as
> `leave-requests` submit is), keep the orchestration in the use-case and let
> the aggregate contribute the **pure invariants** (a `static assert…` guard)
> plus the **creation event** (published post-commit with the real id). Don't
> force a full factory through the transaction.

**Value objects** (`domain/value-objects/`): immutable, self-validating,
compared by value. Extend `ValueObject<TProps>` from `@/utils/domain`. Validate
in the constructor and throw a **plain domain error** (never a Nest exception).
Examples in the codebase: `LeaveStatus` (legal states), `DateRange`
(`end >= start`), `LeaveDuration` (non-negative, 0.5 steps), `HalfDayWindow`
(full ⇒ no window; half ⇒ `start < end`). The shared `AuditStamp` VO groups the
`created_by/at`, `updated_by/at`, `deleted_by/at` cluster.

> **When does a primitive want to be a Value Object?** When it has rules you
> keep re-checking: a status with legal values, a money amount that can't go
> negative, a date span where `end >= start`, a time window where
> `start < end`. Push the rule into the constructor so an invalid one can't
> exist anywhere.

**Domain events** (`domain/events/`): past-tense facts, carrying IDs not
objects. Extend `DomainEvent` (gives `name` + `occurred_at`). The aggregate
records them; subscribers (if any) react via `@OnEvent('<name>')`.

**Domain errors** (`domain/<root>-errors.ts`): plain `Error` subclasses the
aggregate throws. The use-case maps each to the exact HTTP exception (§3.3) —
this keeps the domain free of `@nestjs/common` while preserving the wire
contract.

**The data record** (`domain/<root>.ts`): a pure class (no decorators) holding
the persisted columns. It is the mapper's `reconstitute` input and the
read-path shape the assembler serializes. Snake_case fields, definite
assignment (`!`). **Name it `<Root>Record`** (e.g. `LeaveRequestRecord`) so it
stays distinct from the aggregate `<Root>` — they share a feature but play two
roles: the record is data, the aggregate is behavior. No aliasing at call
sites.

### 3.2 Persistence layer

**Hard rules:**

1. The **port is an abstract class** (NestJS DI needs a runtime token; an
   interface compiles to nothing).
2. **One repository per aggregate ROOT.** Child entities load/save through the
   root, not via their own public repo.
3. **Reconstitution lives at the persistence seam:** `mapper.toAggregate(entity)`
   rebuilds the rich aggregate; `mapper.toDomain(entity)` builds the plain
   data record for read paths. The repo never returns entities.
4. The repo **never throws `NotFoundException`** — it returns `… | null`; the
   service owns the 404.
5. The repo **respects `deleted_at IS NULL`** unless the caller opts into
   `includeDeleted`.
6. DB-level invariants (partial unique indexes for "at most one active row")
   remain the source of truth for concurrency; the service maps `23505` to a
   friendly `ConflictException`.

**Port template** (`persistence/base-<root>.repository.ts`):

```ts
export abstract class Base<Root>Repository {
  abstract findAll(criteria: <Root>SearchCriteria): Promise<FindAll<Root>>;       // read-model
  abstract findById(id: number): Promise<<Root>Record | null>;                 // read path
  abstract findAggregateById(id: number): Promise<<Root> | null>;                  // write path (reconstituted)
  abstract create(input: Create<Root>Persistence, manager?: EntityManager): Promise<<Root>Record>;
  abstract update(id: number, patch: <Root>Patch): Promise<<Root>Record>;
  abstract softDelete(id: number, deleted_by: number | null): Promise<void>;
}
```

**Mapper template** (`persistence/mappers/<root>.mapper.ts`):

```ts
export class <Root>Mapper {
  /** Write-path load — reconstitute the rich aggregate (validates VOs). */
  static toAggregate(raw: <Root>Entity): <Root> {
    return <Root>.reconstitute(<Root>Mapper.toDomain(raw));
  }
  /** Read-path — the plain data record the assembler serializes. */
  static toDomain(raw: <Root>Entity): <Root>Record { /* explicit field copy */ }
}
```

`persistence.module.ts` binds `{ provide: Base<Root>Repository, useClass:
<Root>Repository }` and re-exports `TypeOrmModule`. Audit columns, partial
unique indexes, and migration shape follow
`database-migration-conventions.md`.

### 3.3 Application layer (the thin use-case service)

**Hard rules:**

1. The service **orchestrates, it does not decide.** The shape of every
   write use case is: **load the aggregate → call a behavior method → persist
   the result → publish the buffered events.**
2. The service **depends on the port** (`Base<Root>Repository`), owns the
   `NotFoundException` (repo returns null), and does the I/O the aggregate
   can't (cross-aggregate lookups, schedule math, file upload, transactions).
3. The service **maps domain errors to HTTP** at the edge of a behavior call,
   so the aggregate stays framework-free:

   ```ts
   try { aggregate.approve(actor); } catch (err) { rethrowDomainError(err); }
   ```

   where `rethrowDomainError` translates each domain-error type to the project
   error envelope (`conflict`/`forbidden`/`unprocessable` from
   `@/utils/helpers/http-errors`, or a plain Nest exception).
4. The service **builds a caller-capability value** (`<Root>Actor`) from `User`
   via `hasPermission(...)` and passes it to behavior, so the aggregate never
   imports `User` or the permission system.
5. **Publish events post-commit.** Drain `aggregate.pullEvents()` after the save
   resolves and hand them to `DomainEventPublisher`. For creation events whose
   id is insert-generated, publish from the use-case with the persisted id.
6. **No `if (audience === 'admin')` branches.** Self-service-specific rules get
   their own method, not a fork inside a shared one.

**The error-shape contract** (unchanged from before — the use-case reproduces it):

| Situation | Throw |
|---|---|
| Row not found by id | `NotFoundException` |
| Field validation not expressible on the DTO | `unprocessable(field, msg)` → 422 envelope |
| Hard uniqueness conflict / illegal state transition | `conflict(field, msg)` → 409 envelope |
| Forbidden action | `forbidden(field, msg)` or plain `ForbiddenException` |
| Authn-related | `UnauthorizedException` |

### 3.4 Interface layer — response DTO + assembler + controllers

**Hard rules:**

1. **The domain never carries `@ApiProperty`.** The wire/Swagger shape lives in
   `dto/response/<resource>-response.dto.ts`. Field names + order mirror the
   persisted record (snake_case end-to-end), so the JSON is identical to what a
   domain-as-response design would emit.
2. **An assembler maps aggregate / read-model → response DTO** (`<resource>.assembler.ts`).
   It is the seam where the wire shape may one day diverge from the domain;
   today it is a faithful structural copy.
3. Controllers return response DTOs (`await` the service, then assemble).
   `@ApiResponse({ type: <Resource>ResponseDto })` everywhere.
4. **Admin vs. self-service is enforced here and in the request DTOs**, never in
   the aggregate/service — see §5. `me-*` routes have no `:id`; identity is
   `@CurrentUser().id`.
5. `@Controller({ path, version: API_VERSION })`; `JwtAuthGuard` is global
   (never re-apply it); admin routes declare `@Permissions({ RESOURCE: 'Action' })`.

---

## 4. The shared DDD kit (`src/utils/domain/`)

| File | Purpose |
|---|---|
| `aggregate-root.ts` | `AggregateRoot` base — `recordEvent` / `pullEvents` (drain) event buffer. |
| `domain-event.ts` | `DomainEvent` base — `name` + `occurred_at`. |
| `domain-event-publisher.ts` | `DomainEventPublisher` — publishes events over `@nestjs/event-emitter` post-commit. The ONE place the domain meets the framework, by design (it is application/infra, not domain). |
| `value-object.ts` | `ValueObject<T>` base — structural `equals`, frozen props. |
| `audit-stamp.ts` | `AuditStamp` VO — the `created_by/at`, `updated_by/at`, `deleted_by/at` cluster + `isDeleted()`. |
| `value-objects/date-range.ts` | Shared `DateRange` (`end >= start`, `endsBefore`, `isSingleDay`). |

`EventEmitterModule.forRoot()` is registered globally in `app.module.ts`.

---

## 5. Admin vs self-service split

The audience is enforced at the **edge** (request DTO + controller). The
aggregate, persistence, and service are oblivious to who is calling.

| Layer | Audience-aware? | How |
|---|---|---|
| Controller | YES | Distinct files. `admin-*` carries `@Permissions(...)`; `me-*` has no `:id` and uses `req.user.id`. |
| Request DTO | YES | `dto/admin/*` (wide) vs `dto/me/*` (narrow allowlist). A field absent from a `/me` DTO is rejected 400 by `forbidNonWhitelisted` — that IS the boundary. |
| Response DTO | NO | One response DTO per resource. |
| Service / Aggregate / Persistence | NO | One of each. Self-service-specific logic is a distinct method, not an `if (audience)` branch. |

Privileged operations get their own endpoint, not an extra field (password
change is the canonical example). Reference: `src/users/`.

---

## 6. Aggregate boundaries — get this right first

An **aggregate is a consistency boundary**: one root, everything inside changes
together in one transaction, everything outside is referenced by ID.

- **One transaction = one aggregate.** If a use case must change two
  aggregates, do it via a **domain event**, not one giant transaction.
- Decide the boundary by asking *what must stay consistent together?* A
  `SalesOrder` and its `SalesOrderItems` are one aggregate (all-or-nothing). A
  `Seller`, `User`, `Voucher` are separate aggregates referenced by id.
- Inside the boundary you hold full child entities; outside you hold an id.
- Some tables are **not** aggregates: seed-managed reference data (e.g.
  `permissions`), cross-aggregate **query/read services** (e.g. an approvals
  inbox), infrastructure adapters (e.g. object storage), and **derived
  read-models** (e.g. a computed balance) which are projected, never written.

Mapping the modules to aggregates (root, children, referenced-by-id) before
writing code is the highest-leverage step in a new module.

---

## 7. Concurrency & DB-level invariants

For any "at most N rows where …" rule, **enforce it at the DB level with a
partial unique index** and let the service translate the violation into a
friendly error. The aggregate's guard is the nice-path check; the index is the
race-safe source of truth. Current examples: one open punch per employee, one
active schedule per (employee, weekday), case-insensitive soft-delete-aware
email uniqueness. See `database-migration-conventions.md`.

---

## 8. Testing

- **Aggregate behavior is unit-tested with no DI and no DB** — that's the
  payoff of a pure domain. Construct via `reconstitute(props)`, call a behavior
  method, assert the new state and `pullEvents()`. See
  `leave-request.aggregate.spec.ts`.
- **Value objects are unit-tested** for their construction invariants (valid
  inputs succeed; invalid inputs throw).
- **Service specs mock the abstract `Base<Root>Repository`** — never the
  concrete class, never `Repository<Entity>`.
- E2E (`npm run test:e2e`) is the byte-identical contract guard. It needs
  Postgres + MinIO + `STORAGE_*`; run with the CI seed password
  (`SEED_DEFAULT_PASSWORD=Asima@1234`) so seeded logins match.

---

## 9. Data flow walkthrough — `POST /leave-requests/:id/approve`

1. **Interface.** `LeaveRequestsController.approve` runs under the global
   `JwtAuthGuard`. Identity from `@CurrentUser()`.
2. **Application.** `LeaveRequestsService.approve(id, caller)`:
   - `repository.findAggregateById(id)` → reconstituted `LeaveRequest` (the
     mapper builds its value objects); `null` → `NotFoundException`.
   - builds a `LeaveActor` from `caller` (capabilities only).
   - `aggregate.assertApprovable(actor)` (pure guard) inside a try/catch that
     maps domain errors to the same 409/403 envelopes as before.
   - does the I/O the aggregate can't — the snapshotted step approver must
     still be active.
   - `aggregate.applyApproval(actor)` performs the transition and records
     `LeaveRequestApproved` / `LeaveRequestAdvancedToL2`.
   - persists via `repository.update(id, patch)`; then
     `publisher.publish(aggregate.pullEvents())`.
3. **Interface.** The controller assembles the result into a
   `LeaveRequestResponseDto`. JSON is identical to the pre-DDD response.

---

## 10. Module boundaries with other features

When your service needs another feature, inject the **abstract repository** of
that feature, never its concrete class. Avoid circular module imports — if A
and B both need each other, one side imports only the other's persistence
module. Cross-aggregate consistency that isn't a simple lookup is a **domain
event + subscriber**, not a shared transaction.

---

## 11. New-aggregate checklist

**Boundary**
- [ ] The aggregate root and its consistency boundary are decided; neighbors
      are referenced by id, not object.

**Domain**
- [ ] `domain/<root>.aggregate.ts` extends `AggregateRoot`, has a `private`
      constructor, a static creation factory, and `reconstitute`.
- [ ] Zero `@nestjs/*` / `typeorm` / `class-validator` imports anywhere in
      `domain/`.
- [ ] Validated primitives are value objects in `value-objects/`, each
      throwing on invalid construction, each unit-tested.
- [ ] Behavior methods guard, mutate, and `recordEvent(...)`.
- [ ] Domain errors are plain `Error` subclasses; events extend `DomainEvent`.

**Persistence**
- [ ] `base-<root>.repository.ts` is an `abstract class`; one repo per root.
- [ ] `mapper.toAggregate` reconstitutes; `mapper.toDomain` builds the read
      record; the repo returns neither entities nor throws 404.
- [ ] Audit columns + any partial unique index per
      `database-migration-conventions.md`.

**Application**
- [ ] Service = load → behave → persist → publish; depends on the port; owns
      the 404; maps domain errors to the HTTP envelope.
- [ ] A `<root>-actor` capability value is built from `User`; the aggregate
      never imports `User`.

**Interface**
- [ ] `dto/response/*` carries `@ApiProperty`; the domain does not.
- [ ] An assembler maps aggregate/read-model → response DTO; controllers return
      DTOs.
- [ ] Admin vs `/me` split at the controller + request DTOs; `/me` has no `:id`.

**Tests**
- [ ] Aggregate behavior + value objects unit-tested (no DI).
- [ ] Service spec mocks the port.
- [ ] `npm run lint && npm run test && npm run test:e2e && npm run build` green.

---

## 12. Anti-patterns (don't do these)

| Anti-pattern | Why it's wrong | Do this instead |
|---|---|---|
| **Anemic domain** — logic in the service, the domain class a bag of data | The whole point of DDD; invariants scatter and drift across services. | Put behavior + invariants on the aggregate. |
| `@ApiProperty` / `@Entity()` / `@Injectable()` on a domain class | Couples the core to HTTP/ORM/DI; breaks pure unit testing. | Decorators live on the response DTO / entity; the domain is pure. |
| `Object.assign(aggregate, row)` | Bypasses construction — an invalid aggregate becomes representable. | `static reconstitute(props)` via a private constructor that builds value objects. |
| Holding a full sibling aggregate (`order.seller: Seller`) | Blurs the consistency boundary; invites multi-aggregate transactions. | Hold `seller_id`; load the other aggregate in the use-case. |
| One giant transaction spanning two aggregates | Couples their lifecycles; lock contention; broken boundary. | Change one aggregate per transaction; cross-aggregate via a domain event. |
| Aggregate throwing a `NestException` | Couples the domain to `@nestjs/common`. | Throw a plain domain error; map it to HTTP in the use-case. |
| Aggregate publishing events itself | Mixes I/O into the domain; events fire before commit. | Aggregate records; the use-case publishes post-commit. |
| Raw primitive with rules re-checked everywhere | Validation drifts; invalid values slip through. | A self-validating value object. |
| Repository returning entities | Leaks TypeORM through the port. | Map to aggregate / data record at the seam. |
| Mocking the concrete repository in a service spec | Binds the test to TypeORM internals. | Mock `Base<Root>Repository`. |
| `synchronize: true` | Silently rewrites schema. | Always migrations. |

---

## 13. Authoritative cross-references

When a rule here conflicts with one of these, the **specific document wins**
for its area:

- `asima-backend/CLAUDE.md` — backend operational detail (permissions seed,
  throttler tiers, migration commands, guard pipeline, swagger groups).
- `asima-parent/CLAUDE.md` — system-level conventions (snake_case end-to-end,
  permission code shape, admin/me contract, terminology).
- `database-migration-conventions.md` — one table = one CREATE migration.
- `src/leave-requests/` — the reference exemplar. If this guide and the code
  diverge, the code is the source of truth; open a PR updating this document.
