# Employee Compensation foundation

**Date:** 2026-06-20 (Phase 4 appended 2026-06-21)
**Scope:** cross-cutting (asima-backend + asima-frontend)
**Status:** **Phases 1–2 SHIPPED** (commits `64c083c` permission resource, `0f28a45`
module + set-pay, `76e8088` read/correct/remove + `/me` + e2e; frontend
`features/admin-compensation/*` + `features/profile/components/compensation-card.tsx`).
Phase 3 seeder folded into **Phase 4 — gap closure** (below), in progress.
Original five-axis review (I1–I4 + S1/S2/S4) all folded in and verified present in code.

## Why this, why now

The product lifecycle (`identity → leave/approvals → schedules → time entries
→ workforce`) has reached its final stage. Everything before *workforce* is
built. The first workforce feature we chose is **Overtime (OT)** — and
brainstorming established that OT, as the user wants it (**post-hoc claim,
per-entry, real monetary pay, auto-derived categories**), is not one feature
but a feature sitting on three foundations the codebase lacks:

| Foundation | Needed because |
|---|---|
| **A. Employee compensation** | "Actual monetary amount" = `hours × multiplier × base_rate`; no wage field exists on `users`. |
| B. Holiday calendar | Auto-derived OT category needs to know which dates are holidays. |
| C. OT policy / multipliers | The category → multiplier map. |
| D. Overnight-window support in `metrics.ts` | OT classification breaks for night shifts (flagged as a prerequisite in the 2026-06-14 punch-revamp plan). |

Each foundation gets its own spec → plan → implementation cycle; the OT claim
workflow ties them together; then DTR reporting sums approved OT into periods.
**This plan covers only Foundation A — Employee compensation.** It is the
deepest dependency (no pay math without it) and independently valuable.

## Decisions locked during brainstorming

1. **Post-hoc OT, anchored to real punch logs** (context for why A exists; not
   built here).
2. **Store both** a canonical `monthly_salary` AND a derived-but-overridable
   `hourly_rate`. Hourly is derived from monthly via a company **divisor**
   convention; HR may override the result.
3. **Effective-dated history**, not a single mutable value — mirrors the
   `work_schedules` pattern (`effective_from`/`effective_to`, partial unique
   index on the active row). OT pay for a past date uses the rate in effect
   **then**.
4. **Sensitive, audience-split surfaces:** a new HR-gated `COMPENSATION`
   permission resource (separate from `USER:*`) for admin writes/reads, plus a
   **read-only `/me/compensation`** so an employee can see their own pay.

## Architecture decisions

- **New standalone hexagonal module `src/compensation/`**, table
  `employee_compensations`. Rejected alternatives: columns on `users` (can't
  hold history, pollutes the most sensitive field across every user read) and a
  generic "effective-dated attribute" table (over-engineered; the codebase
  keeps one table per concern). Reuses the `work-schedules` effective-dating
  **shape** (columns, indexes, audit), with two deliberate divergences (I1, I2).
- **Effective-dating via the `work_schedules` template.** "Change pay" =
  end-date the active row (`effective_to = new effective_from − 1 day`) and
  insert a new active row, inside one transaction. Never a destructive UPDATE
  of an active row except for an in-place *correction* of an erroneous entry.
- **(I1) One-step auto-end, a deliberate divergence from `work_schedules`.**
  `work_schedules` *rejects* a `POST` when an active row exists and requires a
  separate end step (`work-schedules.service.ts:55-62`, `endLogically` at
  `:111`). Compensation instead **auto-end-dates the prior active row inside the
  same `POST` transaction** — chosen because a pay change is a single HR action
  and the two-step adds friction with no safety benefit (the partial unique
  index still guarantees one active row). This is the one place the modules
  intentionally differ; reviewers should expect it.
- **(I2) No future-dating in this foundation.** `effective_from <= today` is
  enforced (service validation). Consequence: **the active row
  (`effective_to IS NULL`) IS the current rate**, so reads MAY use it directly.
  Scheduling a raise ahead of time is deferred; if it ever lands, "current"
  must move to `findRateOnDate(today)` everywhere. Until then
  `findCurrentForEmployee` (active row) and `findRateOnDate(today)` return the
  same row by construction.
- **The OT-facing seam is `findRateOnDate(employeeId, date)`** on the
  repository/service — "what was this employee's hourly rate on date X." This
  is the single method Foundation E (OT pay computation) will consume. Defining
  it now keeps the later OT slice from reaching into compensation internals.
- **Derivation lives behind one config knob.** `deriveHourly(monthly) =
  round(monthly / DIVISOR, 4)`; `DIVISOR` is a single constant
  (env-overridable) in `compensation.constants.ts`. Storing `hourly_rate`
  concretely (not computing on read) keeps OT math stable when the divisor
  policy later changes.
- **Audience split is physical (DTO + controller), per the backend CLAUDE
  contract.** `dto/admin/*` (wide) vs a read-only `/me` surface; one domain,
  one repository, one service shared by both controllers.

## Data model

Table **`employee_compensations`** (migration `CreateEmployeeCompensationsTable`):

| Column | Type | Notes |
|---|---|---|
| `id` | PK | |
| `employee_id` | `int` + FK→`users` | `onDelete: 'RESTRICT'`, eager false (as `work_schedules`) |
| `monthly_salary` | `numeric(12,2)` | Canonical HR figure. **`select: false`** (S1) — opt-in via finders, like `users.password_hash`, so a careless join/serialization can't leak pay |
| `hourly_rate` | `numeric(12,4)` | Concrete; defaults to derived value, HR may override. **`select: false`** (S1) |
| `hourly_rate_is_overridden` | `boolean` default `false` | Whether `hourly_rate` diverged from derivation — so a later salary edit knows whether to recompute or preserve HR's manual value |
| `effective_from` | `date` NOT NULL | |
| `effective_to` | `date` nullable | **NULL = active** |
| `created_by`/`updated_by`/`deleted_by` | `int` null | audit |
| `created_at`/`updated_at`/`deleted_at` | `timestamptz` | via `EntityHelper` |

**Indexes / DB-level guards** (the `work_schedules` template):
- `@Index(['employee_id', 'effective_to'])`.
- **Partial unique index** on `(employee_id) WHERE effective_to IS NULL AND
  deleted_at IS NULL` → at most one active comp row per employee. The DB is the
  source of truth for the concurrent-write race, not just the service.
- CHECK `monthly_salary >= 0`, CHECK `hourly_rate >= 0`,
  CHECK `effective_to IS NULL OR effective_to >= effective_from`.

**Config (`compensation.constants.ts`):**
- `COMPENSATION_MONTHLY_HOURS_DIVISOR` — default **208.67**
  (`= 313 working days × 8h ÷ 12 months`), env-overridable. **Policy number —
  see Open Questions.**
- `HOURLY_RATE_SCALE = 4`. Currency is a single company constant (`PHP`), not a
  per-row column (single-tenant; YAGNI).

## Permission / access model

New resource `COMPENSATION` in `PERMISSION_RESOURCES` + `permissions.json`,
using the dual-audience `ViewOwn`/`ViewAll` split (as `LEAVE`/`TIME_CORRECTION`):

| Code | Audience |
|---|---|
| `COMPENSATION:ViewAll` | HR reads anyone's compensation |
| `COMPENSATION:ViewOwn` | employee reads own (`/me/compensation`) |
| `COMPENSATION:Create` | set/change pay (new effective-dated row) |
| `COMPENSATION:Update` | correct an erroneous row in place |
| `COMPENSATION:Delete` | soft-delete an erroneous row |

**Role mapping (`roles.json`):** all `COMPENSATION:*` → `HR_ADMIN` (and
`SUPER_ADMIN`, which also bypasses via `system_admin`). `COMPENSATION:ViewOwn` →
every role (`EMPLOYEE`, `PROJECT_MANAGER`, `TECHNICAL_DIRECTOR`, `HR_ADMIN`) so
all employees can read their own pay. Ordinary admins get **no**
`COMPENSATION:ViewAll` — salaries stay HR-only.

## API surface

**Admin (`admin/compensation`, HR-gated):**
- `POST /api/v1/admin/compensation` — set/change pay. Body: `employee_id`,
  `monthly_salary`, optional `hourly_rate` (override), `effective_from`. Service
  auto-end-dates the employee's prior active row. Gate `COMPENSATION:Create`.
- `GET /api/v1/admin/compensation` — paginated current rows (one per employee),
  filterable by `employee_id`. `{ data, total, page, limit, has_more }`. Gate
  `COMPENSATION:ViewAll`.
- `GET /api/v1/admin/compensation/:id` — one row by id (I3; backs the FE edit
  form, parity with `work_schedules`' `GET :id`). Gate `COMPENSATION:ViewAll`.
- `GET /api/v1/admin/compensation/employees/:employeeId` — current + full
  history for one employee. Gate `COMPENSATION:ViewAll`. (`employees/` is a
  literal segment, so no collision with the numeric `:id` route.)
- `PATCH /api/v1/admin/compensation/:id` — correct an erroneous row in place
  (recompute hourly if `monthly_salary` changes and not overridden). Gate
  `COMPENSATION:Update`.
- `DELETE /api/v1/admin/compensation/:id` — soft-delete an erroneous row.
  **Only the active row is deletable** (S2); deleting it reactivates the prior
  row (`effective_to = NULL`). Deleting a historical row would punch a gap in
  `findRateOnDate`, so it is rejected (400). Gate `COMPENSATION:Delete`.

Throttling: these routes keep the global `default` throttle — do **not**
`@SkipThrottle()` them (S1-nit); salary reads aren't a hot typeahead path.

**Self-service:**
- `GET /api/v1/users/me/compensation` — caller's current compensation
  (`monthly_salary`, `hourly_rate`, `effective_from`), read-only. Gate
  `COMPENSATION:ViewOwn`. No `:id`; identity from `req.user.id`. **When the
  employee has no comp row yet (new hire), returns `200` with `null`** (I4) —
  not 404 — so the FE renders a "not set" state.

Swagger: `@ApiTags('Admin - Compensation')` and `'Compensation'`; new DTO group
in `main.ts`.

## Service logic

`CompensationService` (depends on `BaseCompensationRepository`):
- `create(dto, actorId)` — validate employee exists; validate
  **`effective_from <= today`** (I2) and not before the prior row's
  `effective_from`; compute hourly (`deriveHourly` or honor override → set
  `is_overridden`); in a transaction end-date the prior active row and insert
  the new one.
- `findAllCurrent(query)` / `findHistoryForEmployee(id)` / `findOne(id)`. All
  money-returning finders explicitly `addSelect` the `select:false` columns
  (S1); `findAllCurrent` joins `users` in a single query selecting only
  `id`/name (no N+1, S4) and rides `@Index(['employee_id','effective_to'])`.
- `update(id, dto, actorId)` — in-place correction; recompute hourly when
  `monthly_salary` changes and `is_overridden = false`.
- `softDelete(id, actorId)` — soft-delete + reactivate prior row when needed.
- `findCurrentForEmployee(employeeId)` — backs `/me`.
- **`findRateOnDate(employeeId, date)`** — the OT-facing port: the row where
  `effective_from <= date AND (effective_to IS NULL OR effective_to >= date)`.

## Frontend surfaces

- **New feature `admin-compensation`** (mirrors `admin-schedule`): HR-gated
  page (driven by `COMPENSATION:ViewAll` from `/users/me/permissions`) listing
  employees' current pay; a drawer/form to set/change pay (POST) and view
  history; data-access via the `keys.ts` + queries/mutations convention;
  shadcn-only components. Permission-gated sidebar entry (role-based-sidebar
  pattern).
- **Employee self-view:** read-only current-pay card in the `profile` feature,
  gated by `COMPENSATION:ViewOwn`.

## Task list

### Phase 1 — Backend foundation — ✅ SHIPPED (Tasks 1–5, commits `64c083c`/`0f28a45`/`76e8088`)

#### Task 1: COMPENSATION permission resource + role mappings
**Description:** Add the `COMPENSATION` resource and its five permission codes,
and map them to roles, so subsequent admin routes can be gated.
**Acceptance criteria:**
- [ ] `COMPENSATION` added to `PERMISSION_RESOURCES`; five codes in
  `permissions.json` with descriptions.
- [ ] `roles.json` maps all `COMPENSATION:*` to `HR_ADMIN`/`SUPER_ADMIN` and
  `COMPENSATION:ViewOwn` to every role.
- [ ] `npm run seed` is idempotent (re-run is a no-op).
**Verification:** `npm run seed` twice → no duplicate rows; codes queryable.
**Dependencies:** None. **Files:** `permissions.constants.ts`,
`seeds/data/permissions.json`, `seeds/data/roles.json`. **Scope:** S.

#### Task 2: Set compensation (write path + table)
**Description:** The foundational vertical slice — entity, migration, domain,
mapper, repository (`create`, `endLogically`, `findActiveForEmployee`),
`service.create` + `deriveHourly`, `CreateCompensationDto`, and
`POST /admin/compensation`. Lands the table together with its first capability
(the backend CLAUDE forbids a persistence-only half-slice).
**Acceptance criteria:**
- [ ] HR can `POST /admin/compensation`; a prior active row is auto-end-dated
  (I1); the partial unique index prevents two active rows.
- [ ] `hourly_rate` is derived when omitted (`is_overridden=false`) and honored
  when supplied (`is_overridden=true`); rounding to 4 dp.
- [ ] `effective_from` before the prior row's `effective_from` → 400; **`effective_from > today` → 400 (I2).**
**Verification:** `npm run test` (service spec, mocked repo) + a focused e2e
exercising the auto-end-date and the unique-index race; `npm run build`.
**Dependencies:** Task 1. **Files:** `compensation/` module set + one migration
(timestamp must sort after the latest existing migration). **Scope:** L —
acknowledged: kept whole because the backend CLAUDE forbids a persistence-only
half-slice, but bounded to entity+migration+`create`+`POST`; *all* list/lookup
finders move to Task 3 (only `findActiveForEmployee`, needed by `create`, lands
here).

#### Task 3: View compensation (admin read)
**Description:** Repository finders + service + `QueryCompensationDto` +
`GET /admin/compensation`, `GET /admin/compensation/:id` (I3),
`GET /admin/compensation/employees/:employeeId`. Includes `findRateOnDate`
(the OT seam) with a boundary unit test. Money finders `addSelect` the
`select:false` columns (S1).
**Acceptance criteria:**
- [ ] List returns one current row per employee in `{ data, total, page, limit,
  has_more }`, filterable by `employee_id`, joining `users` once (no N+1, S4).
- [ ] `GET /:id` returns a single row incl. money columns; `employees/:id`
  history is ordered by `effective_from` desc.
- [ ] `findRateOnDate` returns the correct row on boundary dates (== from,
  == to) and `null` in a gap / before first row.
**Verification:** service spec + e2e for the queries; `npm run build`.
**Dependencies:** Task 2. **Files:** repository, service, query DTO, admin
controller. **Scope:** M.

#### Task 4: Correct & remove rows
**Description:** `PATCH /admin/compensation/:id` (in-place correction, recompute
hourly when `monthly_salary` changes and not overridden) and
`DELETE /admin/compensation/:id` (soft-delete + reactivate prior row).
**Acceptance criteria:**
- [ ] PATCH on the active row updates fields without creating history.
- [ ] DELETE soft-deletes **only the active row**; deleting it restores the
  prior row's `effective_to = NULL`; deleting a historical row → 400 (S2).
**Verification:** service spec covering recompute + reactivation + the
historical-delete rejection; e2e for soft-delete reactivation; `npm run build`.
**Dependencies:** Task 3. **Files:** update DTO, service, controller. **Scope:** M.

#### Task 5: Employee self-view
**Description:** `GET /users/me/compensation` (read-only) gated
`COMPENSATION:ViewOwn`; me-controller keyed on `req.user.id`.
**Acceptance criteria:**
- [ ] Returns only the caller's current compensation; **no comp row → 200 with
  `null`, not 404 (I4).**
- [ ] Employee cannot reach admin routes (403); no write surface on `/me`.
**Verification:** e2e: employee reads own; new-hire null case; employee → admin
route = 403; `npm run build`. **Dependencies:** Task 3. **Files:** me-controller,
module wiring. **Scope:** S.

### Checkpoint: Backend complete
- [ ] `npm run test`, `npm run test:e2e`, `npm run build`, `npm run lint` green
  (run with CI seed password per known e2e gotcha).
- [ ] Swagger shows the new admin + me routes and grouped DTOs.
- [ ] Review with human before frontend.

### Phase 2 — Frontend — ✅ SHIPPED (Tasks 6–7)

#### Task 6: Admin compensation page
**Description:** `admin-compensation` feature — list of current pay, set/change
drawer (POST), history view; permission-gated nav. Data-access via `keys.ts` +
queries/mutations; shadcn-only.
**Acceptance criteria:**
- [ ] Without `COMPENSATION:ViewAll`, the page + nav entry are hidden/blocked.
- [ ] HR can set/change pay and see it reflected; history visible.
**Verification:** vitest (permission-gated render, form submit) + `npm run
build`/`lint`. **Dependencies:** Tasks 2–3. **Files:** `features/admin-compensation/*`,
route, sidebar. **Scope:** M.

#### Task 7: Employee self-view card
**Description:** Read-only current-pay card in `profile`, gated
`COMPENSATION:ViewOwn`.
**Acceptance criteria:**
- [ ] Shows current `monthly_salary` + `hourly_rate`; hidden without ViewOwn.
**Verification:** vitest render test + `npm run build`. **Dependencies:** Task 5.
**Files:** `features/profile/*`. **Scope:** S.

### Checkpoint: Complete
- [ ] Frontend lint/test/build green; regenerate lockfile on Linux if changed.
- [ ] All acceptance criteria met; ready for review.

### Phase 3 — Optional — folded into Phase 4 (Task 14)

#### Task 8: Compensation seed data (optional)
**Description:** Seed a base rate for seeded users (idempotent by
`employee_id` + `effective_from`) so downstream OT dev has rates to compute on.
**Acceptance criteria:** [ ] `npm run seed` idempotent; each seeded employee has
one active comp row. **Verification:** re-run seed = no-op. **Dependencies:**
Task 2. **Files:** `seeds/compensation/*`, `seeds/data/compensation.json`.
**Scope:** S.

### Phase 4 — Gap closure + seeder (added 2026-06-21)

Brainstorming (2026-06-21) found Phases 1–2 already shipped, then surfaced the
gaps below. Decisions locked with the user: divisor `208.67` confirmed; currency
surfaced on reads; salary-correction always recomputes; **build** the salary
audit table and the bulk-import endpoint (not deferred). Verified during review
that the create-race **already** maps to a clean 409 (`isUniqueViolation` →
`conflict` in `compensation.service.ts`), so that earlier-suspected gap is a
non-issue and no work is needed for it.

#### Task 9: Salary-correction always recomputes (behavior change)
**Description:** In `CompensationService.update`, a `monthly_salary` correction
**without** an explicit `hourly_rate` must always re-derive hourly via
`deriveHourlyRate` and set `hourly_rate_is_overridden = false` — removing the
`&& !existing.hourly_rate_is_overridden` guard at `compensation.service.ts:136`.
An explicit `hourly_rate` in the same PATCH still wins (sets overridden=true).
**Acceptance criteria:**
- [ ] PATCH `{monthly_salary}` on a previously-overridden row re-derives hourly
  and flips `hourly_rate_is_overridden` to `false`.
- [ ] PATCH `{monthly_salary, hourly_rate}` keeps the explicit value (overridden=true).
- [ ] The existing service spec's "preserve override" case is rewritten to assert
  the new behavior.
**Verification:** `npm run test` (service spec); `npm run build`.
**Dependencies:** none. **Files:** `compensation.service.ts`, its spec. **Scope:** S.

#### Task 10: `compensation_audit` table (value-level write trail)
**Description:** New table `compensation_audits` recording every write to a comp
row (create / correct / delete) with actor, action, and **before→after** of
`monthly_salary` + `hourly_rate` + `effective_from`. Written inside the same
transaction as the originating `create`/`update`/`softDelete` so the trail can't
drift from the row. Money columns are `select: false` (S1 parity); a read
endpoint `GET /admin/compensation/:id/audit` gated `COMPENSATION:ViewAll`.
**Acceptance criteria:**
- [ ] Migration `CreateCompensationAuditsTable` (timestamp sorts after the comp
  table migration); entity + mapper + repo following the hexagonal set.
- [ ] A create writes one `created` audit row; a correction writes one `updated`
  row with non-null before-values; a soft-delete writes one `deleted` row.
- [ ] `GET /admin/compensation/:id/audit` returns the trail newest-first, money
  columns included via `addSelect`; gated `COMPENSATION:ViewAll`.
- [ ] No salary value reaches application logs (S1 holds for audit too).
**Verification:** service spec (audit written in each path) + e2e for the read;
`npm run build`. **Dependencies:** Task 9 (shares the update write path).
**Files:** `compensation/` audit entity/mapper/repo/service hook, one migration,
admin controller route. **Scope:** L.

#### Task 11: Currency on read payloads (gap #1)
**Description:** Attach `currency` (from `COMPENSATION_CURRENCY`) to every comp
read payload — `/me/compensation`, `GET /admin/compensation`, `:id`,
`employees/:employeeId` — via a thin presenter at the controller boundary
(domain stays pure; no per-row column — single-tenant YAGNI).
**Acceptance criteria:**
- [ ] All four read payloads include `currency: "PHP"`; the `null` new-hire `/me`
  response still returns `200` (currency may be omitted or sit alongside `null`).
- [ ] No new DB column; value sourced from the constant.
**Verification:** e2e asserts `currency` on each read; `npm run build`.
**Dependencies:** none. **Files:** both controllers (or a small presenter util).
**Scope:** S.

#### Task 12: OT-readiness batch seam — `findRatesOnDate` (gap #4)
**Description:** Add `findRatesOnDate(employee_ids: number[], date)` to the repo +
`BaseCompensationRepository` port + service — a batched form of `findRateOnDate`
(the per-day, whole-team access pattern OT pay classification will need),
returning a map/array keyed by `employee_id`. A full date-range matrix is
**deferred** to the OT/DTR slice that defines its real query shape (YAGNI — no
consumer yet); documented as a known limitation.
**Acceptance criteria:**
- [ ] Returns the correct in-effect row per employee for the given date,
  including boundary dates (== from, == to); omits/`null`s employees with a gap.
- [ ] Single query (no per-employee N+1), rides `@Index(['employee_id','effective_to'])`.
**Verification:** repository/service boundary unit test; `npm run build`.
**Dependencies:** none. **Files:** repository, base repo, service, spec. **Scope:** S.

#### Task 13: Bulk-set-pay endpoint (gap #3)
**Description:** `POST /api/v1/admin/compensation/bulk` accepting an array of
set-pay items; reuses the single-create invariants per item (auto-end-date the
prior active row) inside **one all-or-nothing transaction**. Rejects duplicate
`employee_id`s within the payload (400). Gate `COMPENSATION:Create`.
**Acceptance criteria:**
- [ ] N items → N current rows in one transaction; any item failing rolls back all.
- [ ] Duplicate `employee_id` in the body → 400 before any write.
- [ ] Same per-row rules as single create (future-date → 400, before-prior → 422,
  active-row auto-end-dated).
**Verification:** service spec (rollback + dup rejection) + e2e (happy path +
one bad item rolls back); `npm run build`. **Dependencies:** Task 9 (shared
create/derive logic). **Files:** bulk DTO, service method, admin controller route.
**Scope:** M.

#### Task 14: Compensation seeder (was Task 8)
**Description:** `CompensationSeedService` mirroring `WorkScheduleSeedService`,
idempotent on `(employee_id, effective_from)`, fed by
`seeds/data/compensation.json` keyed by email (resolve to `employee_id` at run
time). Wire into `run-seed.ts` (after `UserSeedService`) and `seed.module.ts`.
Concrete bands + edge cases for ready testing:
- TECHNICAL_DIRECTOR ₱130,000 — **Karen Taylor = override example**
  (explicit `hourly_rate`, `is_overridden=true`).
- OPERATIONS_MANAGER (Mary Brown) ₱95,000; PROJECT_MANAGER (James Wilson, Robert
  Johnson, Linda Miller) ₱85,000; HR_ADMIN (Jane Smith, John Doe) ₱75,000.
- **Liam Garcia = history example**: prior row ₱60,000 eff. `2025-01-01`
  end-dated `2025-12-31`, current ₱70,000 eff. `2026-01-01` →
  `findRateOnDate('2025-06-01')` returns the ₱60k row.
- EMPLOYEE software engineers ₱48,000 (incl. **Daniel Aguilar**, the named
  example → ₱230.03/hr at /208.67); other employees ₱42,000–₱55,000.
- **Harper Vargas = new-hire null example**: deliberately no comp row → `/me`
  returns `200` + `null`.
- `admin@asima.inc` (`system_admin`) skipped — not an employee.
**Acceptance criteria:**
- [ ] `npm run seed` twice → no duplicate comp rows (idempotent).
- [ ] Each non-skipped seeded employee (except the deliberate null) has exactly
  one active row; Liam has one ended + one active; Karen's row is overridden.
**Verification:** re-run seed = no-op; spot-check via `/admin/compensation`.
**Dependencies:** Task 9 (uses `deriveHourlyRate`). **Files:**
`seeds/compensation/compensation-seed.service.ts`, `seeds/data/compensation.json`,
`run-seed.ts`, `seed.module.ts`. **Scope:** M.

#### Task 15: Frontend — correct-in-place + currency (gaps #2, #1-FE)
**Description:** Wire the already-existing `adminCompensationApi.update()` into a
`correctRate` mutation in `use-compensation-mutations.ts` and add an "edit current
rate" affordance on the active row in `compensation-manager.tsx`. Consume the new
`currency` field from API reads instead of hardcoding ₱ (admin page + profile card).
**Acceptance criteria:**
- [ ] HR can correct the active row from the UI (PATCH), with the list refetching.
- [ ] Rendered amounts use the API-provided `currency`; no hardcoded symbol.
**Verification:** vitest (mutation wiring + render uses currency) + `npm run
build`/`lint`; regenerate lockfile on Linux if changed. **Dependencies:** Tasks
11, 9. **Files:** `features/admin-compensation/*`, `features/profile/*`. **Scope:** M.

### Checkpoint: Phase 4 complete
- [ ] `npm run test`, `npm run test:e2e`, `npm run build`, `npm run lint` green
  (CI seed password per the known e2e gotcha).
- [ ] `npm run seed` idempotent; seeded rows visible incl. the override/history/null examples.
- [ ] Swagger shows the bulk route + the audit read route; grouped DTOs updated.
- [ ] Plan doc reflects shipped reality; commit backend + frontend to their own `main`.

## Risks and mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Divisor is a policy number; wrong value scales every OT peso | Med | One env-overridable constant; flagged as an Open Question to confirm before Task 2 |
| Sensitive data leaks via a wide read | High | Separate `COMPENSATION` resource; `ViewAll` HR-only; `/me` is read-only; no comp field on any `/users` payload |
| Effective-dating edge cases (gaps, overlaps, delete-of-active) | Med | DB partial unique index + CHECKs as backstop; explicit reactivation logic in Task 4 with e2e |
| `numeric` returned as string by pg | Low | Map to number in the mapper; assert types in service spec |
| Half-landed hexagonal slice | Med | Task 2 lands entity→controller together per backend CLAUDE |

## Review findings folded in (2026-06-20, agent-skills:code-review)

- **I1** one-step auto-end — chosen + documented as a deliberate divergence.
- **I2** no future-dating (`effective_from <= today`) — active row == current.
- **I3** added `GET /admin/compensation/:id`.
- **I4** `/me` returns `200` + `null` for a new hire (no comp row).
- **S1** `select:false` on money columns + finders `addSelect`; list selects
  only `id`/name; no salary in logs; no `@SkipThrottle()`.
- **S2** only the active row is deletable (historical delete → 400).
- **S4** list joins `users` once; rides the `(employee_id, effective_to)` index.
- Resolved: `/me` gating = `COMPENSATION:ViewOwn` (matches `LEAVE`/`TIME_CORRECTION`
  me-routes; work-schedule's JWT-only `/me` is the outlier).

## Open questions — RESOLVED 2026-06-21

1. **Divisor value** — ✅ **confirmed `208.67`** (313×8÷12). Kept as the coded
   default in `compensation.constants.ts`; no change.
2. **Currency** — ✅ single config constant `PHP`, **now surfaced on read
   payloads** (Phase 4 Task 11) — the constant existed but reached zero
   payloads; the FE was rendering ₱ with no currency from the API.
3. **Salary-edit recompute** — ✅ **REVERSED from the original default.** A
   `monthly_salary` correction now **always re-derives** hourly and resets
   `hourly_rate_is_overridden = false` (Phase 4 Task 9), discarding any prior
   manual override. (The shipped code preserved the override; that behavior is
   being changed.)

## Out of scope (later foundations / features)

- OT policy multipliers (Foundation C), holiday calendar (B), overnight metrics
  (D), the OT claim workflow (E), DTR period reporting.
- Real payroll run / payslips / tax. This stores comp; it does not run payroll.
