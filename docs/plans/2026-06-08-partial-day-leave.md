# Implementation Plan: Partial-day leave (first-half / second-half)

> Cross-cutting feature (backend + frontend). Audit snapshot, 2026-06-08
> (revised after review — adds `work_schedules.break_start`, admin-edit
> recompute, and the birthday whole-day rule). This `docs/plans/` file is the
> committed record; the execution copy + todo live in the gitignored
> `asima-parent/tasks/`.

## Overview

Revamp "Apply for leave" so a **single-day** request can be a **half day**:
the employee chooses **Full day**, **First half**, or **Second half**. A half
day charges **0.5** working days against the balance instead of 1, and records
the real clock window of the half (e.g. `09:00–14:00`) derived from the
employee's work schedule — including *when* their break is.

Multi-day requests stay full-day only. Birthday leave is whole-day only.

## The spec — work-hours / half-day model

Driven by the employee's active `work_schedules` row for the requested
weekday: `expected_in`, `expected_out`, `break_minutes`, and the **new
`break_start`** (time the break begins).

```
W (work minutes) = (expected_out − expected_in) − break_minutes
half             = W / 2                       → each half charges 0.5 day
pre              = break_start − expected_in    (work minutes before the break)

split instant S  = expected_in + half                 if half ≤ pre
                 = expected_in + half + break_minutes  otherwise   (split is after the break)

First half  window = [expected_in, S]   → 0.5 day
Second half window = [S, expected_out]  → 0.5 day
```

Worked example — `09:00–18:00`, `break_start 12:00`, `break_minutes 60`:
`W = 8h`, `half = 4h`, `pre = 3h`. `half > pre` → `S = 09:00 + 4h + 1h = 14:00`.
→ **First half `09:00–14:00`**, **Second half `14:00–18:00`**, each 0.5 day.
The lunch (12:00–13:00) falls inside the first-half window because the split
is computed against the real break — no "attribute the break to AM" fudge.

Each half always contains exactly `W/2` work minutes; whichever side the break
falls on contains the break. Deterministic for any break position.

Rules:
- `day_portion ∈ { full, first_half, second_half }`, default `full`.
- A partial portion is valid **only** when `start_date == end_date`, that day
  is a scheduled workday, and the leave type allows half-days. Enforced in the
  DTO/service AND a DB CHECK (`day_portion = 'full' OR start_date = end_date`).
- **Half-day-eligible types:** `vacation`, `sick`, `bereavement`, `emergency`.
  **`birthday` is whole-day only** — a partial birthday request → 422, and the
  UI hides the portion control when birthday is selected.
- `working_days` becomes `numeric(4,1)`: `0.5` for a half day, else the
  existing schedule-aware whole-day count.
- `start_time` / `end_time` are **snapshotted** on the request at submit, so
  the record doesn't drift when the schedule is later edited.

### Schedule break display — one combined column

The break stays **two stored values** — `break_start` (time) + `break_minutes`
(int). The break **end is derived** (`break_start + break_minutes`); no extra
column. Wherever the schedule is shown, the BREAK column renders them as a
single combined string:

```
break_start–break_end (break_minutes min)     e.g.  13:00–14:00 (60 min)
—                                              when break_minutes = 0 / no break
```

Match the table's existing time style (the START/END columns use 24h via
`trimSeconds`, e.g. `09:00`), so the break renders `13:00–14:00 (60 min)`.
A `formatBreak(break_start, break_minutes)` helper owns this; `weekly-schedule.tsx`
(and any admin schedule table) uses it instead of the bare `{break_minutes} min`.

## Architecture decisions (CONFIRMED 2026-06-08)

- **D1 — break position from `break_start`.** The split is computed against the
  real break interval, not a heuristic. (Superseded the earlier "break in first
  half" assumption once `break_start` was added to the schedule.)
- **D2 — one request per date.** A date holds at most one pending/approved
  request. The existing `findOverlapping` already enforces this — **no
  overlap-query change**; a single-day partial is blocked by any prior same-day
  request automatically.
- **D3 — snapshot the window.** Store `start_time`/`end_time` on the request,
  mirroring the `working_days` snapshot.
- **D4 — partial is single-day only.** Multi-day ranges force `full`.
- **D5 — UI control → standard `Select`** (`@/components/select`).
- **D6 — birthday is whole-day only.** Per-type allow-list
  `HALF_DAY_LEAVE_TYPES = { vacation, sick, bereavement, emergency }`.

## Dependency graph

```
Phase 1  work_schedules.break_start  (edit CreateWorkSchedulesTable — unreleased)
   • + break_start (time, nullable) + CHECKs: required when break_minutes>0,
     break_start >= expected_in, break_start + break_minutes <= expected_out
   • entity/domain/mapper + create/update DTO (within-window validation) + seed
        │
        ▼
Phase 2  leave_requests schema  (edit CreateLeaveRequestsTable — unreleased)
   • CREATE TYPE leave_day_portion; working_days int → numeric(4,1) (CHECK >=0.5)
   • + day_portion (default 'full') + start_time/end_time + CHECK partial⇒single-day
   • entity (numeric transformer) + domain + mapper + constants (DAY_PORTIONS,
     HALF_DAY_LEAVE_TYPES); relax LeaveBalance numeric fields
        │
        ▼
Phase 3  LeaveDayCountService  (compute window from break_start; reuse the ONE
   schedule fetch; partial⇒single-day + type-eligibility; returns 0.5 + window)
        │
        ├─► Phase 4  day-count endpoint + DTO accept day_portion
        ▼
Phase 5  submit (reserve 0.5, snapshot, type guard)  +  admin update recompute
        │
        ▼
Phase 6  frontend schemas/api/format (relax z.int; portion labels; HALF_DAY set)
        ▼
Phase 7  ApplyLeaveDrawer (portion control — hidden for multi-day & birthday)
        ▼
Phase 8  Admin work-schedule form gains break_start input
        ▼
Phase 9  Display portion + window + 0.5 on detail + list rows
```

## Task list

### Phase 0 — Decisions
- [x] D1–D6 confirmed 2026-06-08.

### Phase 1 — Work-schedule break_start

## Task 1: Add `break_start` to `work_schedules`

**Description:** Per `database-migration-conventions.md` (unreleased), edit
`1778200100000-CreateWorkSchedulesTable.ts`: add `"break_start" time` (nullable)
and three CHECKs — required when `break_minutes > 0`, `break_start >=
expected_in`, and `break_start + (break_minutes || ' min')::interval <=
expected_out`. Add `break_start` to `WorkScheduleEntity`, `WorkSchedule`
domain, mapper, and `CreateWorkScheduleDto` / `UpdateWorkScheduleDto` (reuse
`TIME_REGEX`; service-level cross-field check that the break fits the window).
Set `break_start: '12:00:00'` in `WorkScheduleSeedService`.

**Acceptance criteria:**
- [ ] `npm run db:fresh` clean; seeded Mon–Fri rows carry `break_start 12:00:00`.
- [ ] Creating a schedule with a break outside the window → 422; `break_minutes>0`
      without `break_start` → 422 (and DB CHECK as the backstop).
- [ ] `break_minutes = 0` may omit `break_start`.

**Verification:** `npm run db:fresh`; `npm test -- work-schedules`; e2e create.
**Dependencies:** none · **Files:** migration, work-schedule entity/domain/mapper, create+update DTO, service, seed · **Scope:** M

### Phase 2 — Leave-request schema foundation (full-day unchanged)

## Task 2: Fold partial-day columns into `CreateLeaveRequestsTable`

**Description:** Edit `1778400000000-CreateLeaveRequestsTable.ts`: add
`leave_day_portion` enum (before the table); `working_days` `integer` →
`numeric(4,1)`; relax `CHK_leave_requests_working_days` to `>= 0.5`; add
`day_portion` (`NOT NULL DEFAULT 'full'`), nullable `start_time`/`end_time`
(`time`), and CHECK `day_portion = 'full' OR start_date = end_date`. Reverse
all in `down()`.

**Acceptance criteria:**
- [ ] `db:fresh` clean; `working_days` is `numeric(4,1)`; `day_portion` defaults `full`.
- [ ] `down()` drops `leave_day_portion` and reverses every addition.

**Verification:** `npm run db:fresh`; schema inspect.
**Dependencies:** Task 0 · **Files:** `1778400000000-CreateLeaveRequestsTable.ts` · **Scope:** S

## Task 3: Entity + domain + mapper + constants

**Description:** Add `DAY_PORTIONS`/`DayPortion` and
`HALF_DAY_LEAVE_TYPES` to `leave-requests.constants.ts`. On the entity:
`working_days` → `@Column('numeric', { precision:4, scale:1, transformer })`
(pg returns numeric as a string — a `ColumnNumericTransformer` keeps the domain
a `number`); add `day_portion`, `start_time`, `end_time`. Extend `LeaveRequest`
domain + mapper; relax `LeaveBalance` numeric fields (they now carry `.5`).

**Acceptance criteria:**
- [ ] `working_days` round-trips as a JS `number` (`0.5`, not `"0.5"`) — assert in a mapper spec.
- [ ] New fields map both directions; `toListItem` unaffected.
- [ ] `npm run build` + mapper specs pass.

**Verification:** `npm test -- leave-request.mapper`; `npm run build`.
**Dependencies:** Task 2 · **Files:** entity, `leave-request.ts`, `leave-balance.ts`, mapper, constants · **Scope:** M

### Checkpoint: Foundations
- [ ] `db:fresh` + full unit suite + leave/work-schedule e2e green, with **no
      behavior change** to existing full-day flow. Update any e2e/seed fixtures
      that assert integer `working_days`.

### Phase 3 — Day-count compute

## Task 4: Portion-aware day count + real break window

**Description:** In `LeaveDayCountService`, fetch the active schedule rows
**once** and reuse them for both the weekday set and the window row (don't add
a second query). Add a window computer using `break_start` per the spec
formula. Extend `assertSubmittableRange` to take `day_portion` + `leave_type`:
for a partial portion require `start_date == end_date` (422 on a range),
require the type is in `HALF_DAY_LEAVE_TYPES` (422 for birthday), require a
workday, and return `{ working_days: 0.5, start_time, end_time }`; `full`
unchanged.

**Acceptance criteria:**
- [ ] `first_half` on `09:00/18:00/12:00/60` → `0.5`, `09:00:00`–`14:00:00`;
      `second_half` → `0.5`, `14:00:00`–`18:00:00`.
- [ ] Partial + multi-day → 422; partial + `birthday` → 422.
- [ ] `full` path returns today's counts exactly.

**Verification:** `npm test -- leave-day-count.service`.
**Dependencies:** Task 3 · **Files:** `leave-day-count.service.ts` (+ spec) · **Scope:** M

## Task 5: Day-count endpoint + DTO accept `day_portion`

**Description:** Add optional `day_portion` to `DayCountQueryDto` (default
`full`) and `leave_type` if needed for the eligibility check; the endpoint +
`previewWorkingDays` return `{ working_days, start_time?, end_time? }`.

**Acceptance criteria:**
- [ ] `GET .../day-count?...&day_portion=first_half` → 0.5 + window.
- [ ] Omitting `day_portion` behaves as before.

**Verification:** e2e on the day-count route.
**Dependencies:** Task 4 · **Files:** `day-count-query.dto.ts`, `me-leave-requests.controller.ts`, `leave-requests.service.ts` · **Scope:** S

### Phase 4 — Submit + admin edit

## Task 6: Submit persists `day_portion` (reserve 0.5)

**Description:** Add `day_portion` to `SubmitLeaveRequestDto` (IsIn, default
`full`) + `SubmitLeaveInput`. In `submit`, pass `day_portion` + `leave_type`
to `assertSubmittableRange`, persist `day_portion` + snapshotted window, reserve
`working_days` (0.5 for partial). Extend the repo `create` input + entity
write. D2: existing same-day overlap already blocks a second request — add a
test confirming it.

**Acceptance criteria:**
- [ ] `first_half` row has `working_days 0.5` + window; balance `reserved` +0.5.
- [ ] Birthday + partial → 422; partial + multi-day → 422.
- [ ] Approve → `0.5` `used`; balance available shows `.5` (e.g. `9.5`).

**Verification:** `npm test -- leave-requests.service`; leave e2e (submit + balances).
**Dependencies:** Task 5 · **Files:** `submit-leave-request.dto.ts`, `leave-request-inputs.ts`, `leave-requests.service.ts`, repo `create`, entity · **Scope:** M

## Task 7: Admin pending-edit recompute (closes a pre-existing gap)

**Description:** `service.update` currently saves date patches **without
recomputing `working_days`** — stale today, worse with partial. Add
`day_portion` to `UpdateLeaveRequestDto`; on any edit of dates / portion /
type, re-run `assertSubmittableRange` to recompute `working_days` +
re-snapshot `start_time`/`end_time`, and re-validate the type rule. Persist the
recomputed fields.

**Acceptance criteria:**
- [ ] Editing a pending request's dates updates `working_days` (no stale value).
- [ ] Editing to/from a half-day updates portion, window, and reserved balance.
- [ ] Editing a birthday request to a partial → 422.

**Verification:** `npm test -- leave-requests.service`; admin-edit e2e.
**Dependencies:** Task 6 · **Files:** `update-leave-request.dto.ts`, `leave-request-inputs.ts`, `leave-requests.service.ts`, repo `update` + entity write · **Scope:** M

### Checkpoint: Backend complete
- [ ] API end-to-end half-day (submit → reserve 0.5 → approve → use 0.5);
      admin edit recomputes; decimal math correct; full suite green.

### Phase 5 — Frontend

## Task 8: Frontend schemas / api / format

**Description:** Add `DAY_PORTIONS` + `LeaveDayPortionSchema` and a
`HALF_DAY_LEAVE_TYPES` set. On `LeaveRequestSchema` add `day_portion` +
nullable `start_time`/`end_time`, relax `working_days` `z.number().int()` →
`z.number()`. Relax `LeaveBalanceSchema` + `DayCountSchema.working_days` to
`z.number()`; add optional `start_time`/`end_time` to `DayCountSchema`. Add
`day_portion` (default `full`) to `SubmitLeaveSchema` with refines: partial ⇒
`start_date == end_date` AND type ∈ HALF_DAY. `api.dayCountPreview` takes
`day_portion`. Add `LEAVE_PORTION_LABELS` + a `date-fns` time formatter. Add
`break_start` (`z.string().nullable()`) to `features/schedule/schemas.ts`
`WorkScheduleSchema`, and a `formatBreak(break_start, break_minutes)` helper
(returns `13:00–14:00 (60 min)` or `—`).

**Acceptance criteria:**
- [ ] `0.5` parses through every `working_days`/balance schema.
- [ ] `SubmitLeaveSchema` rejects partial+multi-day and partial+birthday client-side.

**Verification:** `npm run lint && npm run test && npm run build`.
**Dependencies:** Task 7 (wire contract) · **Files:** `features/leave/{schemas,api,format}.ts`, work-schedule schema · **Scope:** M

## Task 9: ApplyLeaveDrawer — portion control + window preview

**Description:** Add a **Day portion** `Select` that renders only when
`start_date == end_date` **and** the chosen `leave_type` is half-day-eligible
(hidden for birthday). Reset to `full` on multi-day / cleared range / switch to
birthday. Pass `day_portion` into the day-count preview (query key includes it)
and submit. `DayCountBanner` shows, for a partial day, `"0.5 working day · 9:00
AM – 2:00 PM"`; full day unchanged.

**Acceptance criteria:**
- [ ] Single eligible date → control appears; First/Second half → banner `0.5` + window.
- [ ] Multi-day or birthday → control hidden, portion reset to full.
- [ ] Submit sends `day_portion`; standard components only.

**Verification:** lint/test/build; manual on a `09:00–18:00 / 12:00 / 60` schedule.
**Dependencies:** Task 8 · **Files:** `features/leave/components/apply-leave-drawer.tsx` · **Scope:** M

## Task 10: Work-schedule UI — `break_start` input + combined BREAK display

**Description:** Two parts: (a) the create/edit schedule form gets a
`break_start` time input (shown/required when `break_minutes > 0`), wired to the
relaxed schema and the admin work-schedules API; (b) `weekly-schedule.tsx`
(the "My schedule" table, `break_minutes` rendered at line ~41) and any admin
schedule table render the BREAK column via `formatBreak` as a single combined
value (`13:00–14:00 (60 min)`), not the bare `{break_minutes} min`.

**Acceptance criteria:**
- [ ] Saving a schedule persists `break_start`; client validates it sits within the window.
- [ ] "My schedule" BREAK column shows `13:00–14:00 (60 min)`; non-working / no-break days show `—`.

**Verification:** lint/test/build; manual against the seeded Mon–Fri schedule.
**Dependencies:** Task 8 · **Files:** admin work-schedule form component(s), `features/schedule/components/weekly-schedule.tsx` · **Scope:** S

## Task 11: Display portion + window on detail + list rows

**Description:** `leave-detail-drawer.tsx`, `employee-leaves-page.tsx`,
`admin-leave-page.tsx`, and `leave-balance-summary.tsx` render the portion, the
window, and `0.5 day` (and decimal balances) wherever `working_days` shows today.

**Acceptance criteria:**
- [ ] A partial request shows `First half · 0.5 day · 9:00 AM–2:00 PM`.
- [ ] Balance cards render decimals (`9.5`) cleanly; full-day rows unchanged.

**Verification:** lint/test/build; manual.
**Dependencies:** Task 9 · **Files:** the four leave components · **Scope:** S

### Checkpoint: Complete
- [ ] All acceptance criteria met; backend + frontend suites green; manual
      walkthrough of a half-day request end to end. Ready for review.

## Risks and mitigations
| Risk | Impact | Mitigation |
|---|---|---|
| pg returns `numeric` as a string → balances string-concatenate | High | `ColumnNumericTransformer`; mapper spec asserts `typeof === 'number'` |
| Frontend `z.number().int()` rejects `0.5` → 500s the page | High | Relax every `working_days`/balance int (Task 8); `0.5` fixture test |
| `break_start` outside the shift / missing when break>0 | Med | DTO cross-field check + DB CHECKs (Task 1) |
| Stale `working_days` on admin date edits (pre-existing) | Med | Task 7 recompute closes it |
| Editing unreleased CREATE migrations vs. ALTER | Low | Follow `database-migration-conventions.md`; `db:fresh` replays |

## Open questions
- None. D1–D6 confirmed 2026-06-08.
