# Timesheet: status, approver & deficit columns + merged Time in/out

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Date:** 2026-06-13
**Status:** Planned
**Scope:** `asima-frontend` (most of the work) + one small `asima-backend`
read-model addition. **No migration.**

**Goal:** Extend the employee timesheet (`/employee/timesheet`) with a derived
**Status** column, an **Approver (L1/L2)** column showing names + per-level
state, and a **Deficit (h)** decimal-hours column — and merge the separate
**In** / **Out** columns into one **Time in/out** column that shows the
`original → proposed` diff when a correction has been requested for that day.

**Architecture:** Everything except approver *names* is derived on the frontend
from data already in hand (the time entry, the matching `work_schedules` row,
and the matching `time_correction_requests` record). The L1/L2 approval
lifecycle already exists on `time_correction_requests`; we only *surface* it —
no new workflow, no new table. The single backend change exposes
`l1_approver_name` / `l2_approver_name` on the correction **list read-model** by
joining `users` (metadata-only relations on the existing
`l1_approver_id` / `l2_approver_id` FK columns — no schema change).

**Tech Stack:** NestJS + TypeORM (backend read-model), Next.js + React +
TanStack Query + Zod + Tailwind (frontend), Jest / Testing Library.

---

## Problem

The current timesheet (`asima-frontend/src/features/time-entries/components/entries-table.tsx`)
shows `Date · In · Out · Tardiness · Undertime · Work hours · Action`. All
derived metrics are computed client-side in
`asima-frontend/src/features/time-entries/metrics.ts` against the matching
`WorkSchedule`. We want, per the screenshot review:

1. **Status** — one of `Ongoing` (still clocked in), `Applied` (a correction is
   pending L1/L2), `Approved` (correction approved), `Logged` (normal confirmed
   punch, no active correction).
2. **Approver** — L1 and L2 approver **names** plus a per-level state badge
   (`pending` / `approved` / `n/a`), shown only when a correction exists for the
   row.
3. **Deficit (h)** — total scheduled-time deficit `(tardiness + early-out)` in
   decimal hours, `toFixed(2)`. Tardiness/Undertime (minutes) columns stay.
4. **Time in/out** — merge `In` + `Out` into one column. When a correction has
   been requested for the day, render the diff:
   ```
   in:  9:02 → 9:00
   out: 6:30 → 6:30
   ```

## Key finding — only approver *names* need the backend

The timesheet page already fetches the visible window's corrections via
`useMyActiveCorrections` (`asima-frontend/src/features/time-correction/hooks/use-my-active-corrections.ts`),
which calls `timeCorrectionApi.me.list({ from, to, status, limit })`. That
record already carries `status`, `l1_approver_id`, `l2_approver_id`,
`proposed_time_in`, `proposed_time_out`, `work_date`. So **Status**, the merged
**Time in/out** diff, the per-level **state**, and **Deficit (h)** are all
frontend-only.

The *one* gap: the list read-model exposes `employee_name` but **not** approver
names. The fix mirrors the existing `employee` join exactly — and the entity
**already** uses the "scalar `*_id` column + `@ManyToOne` on the same column"
pattern for `employee` / `employee_id`, so this is a proven, in-file pattern.
Because `l1_approver_id` / `l2_approver_id` columns already exist, **no
migration is required** (per `docs/universal-guidelines/database-migration-conventions.md`
— we add no columns and no DB-level FK).

## File structure

**Backend (`asima-backend/`):**
- `src/time-correction-requests/persistence/entities/time-correction-request.entity.ts` — add `l1_approver` / `l2_approver` `@ManyToOne` relations on the existing FK columns.
- `src/time-correction-requests/domain/time-correction-request-list-item.ts` — add `l1_approver_name` / `l2_approver_name`.
- `src/time-correction-requests/persistence/mappers/time-correction-request.mapper.ts` — set the two names in `toListItem`.
- `src/time-correction-requests/persistence/repositories/time-correction-request.repository.ts` — two `leftJoinAndSelect` in `findAll`.
- `src/time-correction-requests/persistence/mappers/time-correction-request.mapper.spec.ts` — extend coverage.

**Frontend (`asima-frontend/`):**
- `src/features/time-correction/schemas.ts` — add `l1_approver_name` / `l2_approver_name`.
- `src/features/time-entries/metrics.ts` — add `timesheetStatus`, `deficitHours`, `approverStates`.
- `src/features/time-correction/hooks/use-my-active-corrections.ts` — return `Map<work_date, TimeCorrectionRequest>` (rename to `useMyCorrectionsByDate`).
- `src/features/time-entries/components/entries-table.tsx` — merged Time in/out cell, new columns.
- `src/app/(app)/employee/timesheet/page.tsx` — pass the map through.
- Tests: `tests/unit/features/time-entries/*.spec.ts(x)`, correction hook test if present.

---

## Task 1: Backend — approver names on the list read-model (domain + entity + mapper)

**Files:**
- Modify: `asima-backend/src/time-correction-requests/domain/time-correction-request-list-item.ts`
- Modify: `asima-backend/src/time-correction-requests/persistence/entities/time-correction-request.entity.ts`
- Modify: `asima-backend/src/time-correction-requests/persistence/mappers/time-correction-request.mapper.ts`
- Test: `asima-backend/src/time-correction-requests/persistence/mappers/time-correction-request.mapper.spec.ts`

- [ ] **Step 1: Write the failing mapper test**

Add to `time-correction-request.mapper.spec.ts` inside the existing
`describe('TimeCorrectionRequestMapper.toListItem', ...)`:

```ts
it('derives l1/l2 approver names from the joined approver relations', () => {
  const item = TimeCorrectionRequestMapper.toListItem(
    rawEntity({
      l1_approver: { first_name: 'Jane', last_name: 'Cruz' } as UserEntity,
      l2_approver: { first_name: 'Bob', last_name: 'Lim' } as UserEntity,
    }),
  );
  expect(item.l1_approver_name).toBe('Jane Cruz');
  expect(item.l2_approver_name).toBe('Bob Lim');
});

it('leaves approver names null when those relations are not loaded', () => {
  const item = TimeCorrectionRequestMapper.toListItem(rawEntity());
  expect(item.l1_approver_name).toBeNull();
  expect(item.l2_approver_name).toBeNull();
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd asima-backend && npx jest time-correction-request.mapper.spec -t "approver names"`
Expected: FAIL — `l1_approver_name` is `undefined` (property not set yet).

- [ ] **Step 3: Add the list-item fields**

In `time-correction-request-list-item.ts`, after the `employee_name` property:

```ts
  @ApiPropertyOptional({ example: 'Jane Cruz', nullable: true })
  l1_approver_name!: string | null;

  @ApiPropertyOptional({ example: 'Bob Lim', nullable: true })
  l2_approver_name!: string | null;
```

- [ ] **Step 4: Add the relations on the entity**

In `time-correction-request.entity.ts`, directly after the existing
`l2_approver_id` column (keep the scalar `*_id` columns — same dual pattern the
entity already uses for `employee` / `employee_id`):

```ts
  @ManyToOne(() => UserEntity, { eager: false, onDelete: 'RESTRICT' })
  @JoinColumn({ name: 'l1_approver_id' })
  l1_approver!: UserEntity;

  @ManyToOne(() => UserEntity, { eager: false, onDelete: 'RESTRICT' })
  @JoinColumn({ name: 'l2_approver_id' })
  l2_approver!: UserEntity | null;
```

`UserEntity`, `ManyToOne`, and `JoinColumn` are already imported in this file.
No migration: the FK columns already exist and no DB-level constraint is added.

- [ ] **Step 5: Set the names in the mapper**

In `time-correction-request.mapper.ts`, in `toListItem`, after the
`item.employee_name = ...` line:

```ts
    item.l1_approver_name = raw.l1_approver
      ? `${raw.l1_approver.first_name} ${raw.l1_approver.last_name}`
      : null;
    item.l2_approver_name = raw.l2_approver
      ? `${raw.l2_approver.first_name} ${raw.l2_approver.last_name}`
      : null;
```

- [ ] **Step 6: Run the test to verify it passes**

Run: `cd asima-backend && npx jest time-correction-request.mapper.spec`
Expected: PASS (all toListItem cases, old and new).

- [ ] **Step 7: Commit**

```bash
cd asima-backend
git add src/time-correction-requests/domain/time-correction-request-list-item.ts \
        src/time-correction-requests/persistence/entities/time-correction-request.entity.ts \
        src/time-correction-requests/persistence/mappers/time-correction-request.mapper.ts \
        src/time-correction-requests/persistence/mappers/time-correction-request.mapper.spec.ts
git commit -m "feat(time-correction): expose l1/l2 approver names on list read-model"
```

---

## Task 2: Backend — join the approvers in `findAll`

**Files:**
- Modify: `asima-backend/src/time-correction-requests/persistence/repositories/time-correction-request.repository.ts:38-42` (the `findAll` query builder)

- [ ] **Step 1: Add the joins**

In `findAll`, after the existing `.leftJoinAndSelect('tc.employee', 'employee')`
line, add:

```ts
      // 1:1 ManyToOne joins — resolve approver display names in one trip,
      // no row multiplication, pagination stays correct.
      .leftJoinAndSelect('tc.l1_approver', 'l1_approver')
      .leftJoinAndSelect('tc.l2_approver', 'l2_approver')
```

- [ ] **Step 2: Build to confirm the query compiles**

Run: `cd asima-backend && npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 3: Verify the names come through end-to-end**

Run the existing time-correction e2e/integration suite (whichever covers the
list endpoint), e.g.:
Run: `cd asima-backend && npm run test:e2e -- time-correction`
Expected: PASS, and a listed request now carries non-null `l1_approver_name`
for any request whose L1 approver exists. If the e2e suite needs the CI seed
password, run with `SEED_DEFAULT_PASSWORD=Asima@1234` (see project memory).

- [ ] **Step 4: Commit**

```bash
cd asima-backend
git add src/time-correction-requests/persistence/repositories/time-correction-request.repository.ts
git commit -m "feat(time-correction): join l1/l2 approvers in findAll read-model"
```

---

## Task 3: Frontend — extend the correction schema with approver names

**Files:**
- Modify: `asima-frontend/src/features/time-correction/schemas.ts` (the `TimeCorrectionSchema` object)

- [ ] **Step 1: Add the two optional, nullable fields**

In `TimeCorrectionSchema`, right after `l2_approver_id: z.number().int().nullable(),`:

```ts
  // Present on list responses (joined read-model); absent on single GET.
  l1_approver_name: z.string().nullable().optional(),
  l2_approver_name: z.string().nullable().optional(),
```

- [ ] **Step 2: Type-check**

Run: `cd asima-frontend && npx tsc --noEmit`
Expected: no errors.

- [ ] **Step 3: Commit**

```bash
cd asima-frontend
git add src/features/time-correction/schemas.ts
git commit -m "feat(time-correction): parse l1/l2 approver names in schema"
```

---

## Task 4: Frontend — derive status, deficit, and approver state in `metrics.ts`

**Files:**
- Modify: `asima-frontend/src/features/time-entries/metrics.ts`
- Test: `asima-frontend/tests/unit/features/time-entries/metrics.spec.ts` (create if absent)

- [ ] **Step 1: Write the failing tests**

Create/extend `tests/unit/features/time-entries/metrics.spec.ts`:

```ts
import { describe, it, expect } from 'vitest';
import { deficitHours, timesheetStatus, approverStates } from '@/features/time-entries/metrics';
import type { TimeEntry } from '@/features/time-entries/schemas';
import type { WorkSchedule } from '@/features/schedule/schemas';
import type { TimeCorrectionRequest } from '@/features/time-correction/schemas';

const schedule = { expected_in: '09:00:00', expected_out: '18:30:00', break_minutes: 60 } as WorkSchedule;

// Build wall-clock times as local-naive (NO trailing Z) then ISO — mirrors
// entries-table.spec.tsx. In test, resolveDisplayTz() is runtime-local, so a
// local-naive time formats back to the same wall-clock on any machine (UTC CI
// included). Hardcoding a Z/UTC offset would only pass on an Asia/Manila box.
const iso = (t: string) => new Date(`2026-06-10T${t}`).toISOString();

function entry(over: Partial<TimeEntry> = {}): TimeEntry {
  return {
    id: 1, employee_id: 1, work_date: '2026-06-10',
    time_in: iso('09:00:00'),
    time_out: iso('18:30:00'),
    source: 'manual', status: 'confirmed', notes: null,
    created_by: null, updated_by: null, deleted_by: null,
    created_at: '', updated_at: '', deleted_at: null, ...over,
  };
}

function correction(over: Partial<TimeCorrectionRequest> = {}): TimeCorrectionRequest {
  return {
    id: 9, employee_id: 1, target_entry_id: 1, work_date: '2026-06-10',
    proposed_time_in: iso('09:00:00'), proposed_time_out: iso('18:30:00'),
    reason: 'x', status: 'pending_l1', submitted_at: '', decided_at: null, decided_by: null,
    decision_note: null, decision_path: null, cancelled_at: null, cancelled_by: null,
    l1_approver_id: 5, l2_approver_id: 7,
    l1_approver_name: 'Jane Cruz', l2_approver_name: 'Bob Lim',
    created_at: '', updated_at: '', ...over,
  };
}

describe('deficitHours', () => {
  it('counts a one-hour-late arrival with on-time out as 1.00', () => {
    const e = entry({ time_in: iso('10:00:00') });
    expect(deficitHours(e, schedule)).toBeCloseTo(1, 5);
  });
  it('is 0 for an exactly-on-schedule day', () => {
    expect(deficitHours(entry(), schedule)).toBe(0);
  });
  it('is null for an open entry', () => {
    expect(deficitHours(entry({ time_out: null }), schedule)).toBeNull();
  });
  it('is null when there is no schedule', () => {
    expect(deficitHours(entry(), undefined)).toBeNull();
  });
});

describe('timesheetStatus', () => {
  it('is ongoing when still clocked in', () => {
    expect(timesheetStatus(entry({ time_out: null }), undefined)).toBe('ongoing');
  });
  it('is applied for a pending correction', () => {
    expect(timesheetStatus(entry(), correction({ status: 'pending_l2' }))).toBe('applied');
  });
  it('is approved for an approved correction', () => {
    expect(timesheetStatus(entry(), correction({ status: 'approved' }))).toBe('approved');
  });
  it('is logged for a normal confirmed punch', () => {
    expect(timesheetStatus(entry(), undefined)).toBe('logged');
  });
});

describe('approverStates', () => {
  it('pending_l1 → both pending (two-level chain)', () => {
    expect(approverStates(correction({ status: 'pending_l1' }))).toEqual({ l1: 'pending', l2: 'pending' });
  });
  it('pending_l2 → L1 approved, L2 pending', () => {
    expect(approverStates(correction({ status: 'pending_l2' }))).toEqual({ l1: 'approved', l2: 'pending' });
  });
  it('approved → both approved', () => {
    expect(approverStates(correction({ status: 'approved' }))).toEqual({ l1: 'approved', l2: 'approved' });
  });
  it('single-level chain marks L2 n/a', () => {
    expect(approverStates(correction({ status: 'approved', l2_approver_id: null })).l2).toBe('na');
  });
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `cd asima-frontend && npx vitest run tests/unit/features/time-entries/metrics.spec.ts`
Expected: FAIL — `deficitHours`/`timesheetStatus`/`approverStates` not exported.

- [ ] **Step 3: Implement the helpers**

At the top of `metrics.ts`, add to the type imports:

```ts
import type { TimeCorrectionRequest } from '@/features/time-correction/schemas';
```

Append to `metrics.ts`:

```ts
export type TimesheetStatus = 'ongoing' | 'applied' | 'approved' | 'logged';

/**
 * Derived row status for the timesheet. "Ongoing" = still clocked in;
 * "Applied"/"Approved" reflect a matching correction's lifecycle; otherwise a
 * normal confirmed punch is "Logged". Corrections that were rejected/cancelled
 * aren't fetched for this view, so those rows correctly fall back to "Logged".
 */
export function timesheetStatus(
  entry: TimeEntry,
  correction: TimeCorrectionRequest | undefined,
): TimesheetStatus {
  if (!entry.time_out) return 'ongoing';
  if (correction) {
    if (correction.status === 'pending_l1' || correction.status === 'pending_l2') return 'applied';
    if (correction.status === 'approved') return 'approved';
  }
  return 'logged';
}

/**
 * Total scheduled-time deficit in decimal hours: tardiness (late in) plus
 * undertime (early out), each already floored at 0. Null for open entries or
 * days with no schedule row (same "don't fabricate a baseline" rule as the
 * minute metrics).
 */
export function deficitHours(entry: TimeEntry, schedule: WorkSchedule | undefined): number | null {
  const late = tardinessMinutes(entry, schedule);
  const under = undertimeMinutes(entry, schedule);
  if (late === null || under === null) return null;
  return (late + under) / 60;
}

export type ApproverLevelState = 'pending' | 'approved' | 'rejected' | 'na';

/**
 * Per-level approval state derived from the single correction status enum.
 * `na` = no such level (single-level chain) or a terminal state where the
 * level never acted.
 */
export function approverStates(correction: TimeCorrectionRequest): {
  l1: ApproverLevelState;
  l2: ApproverLevelState;
} {
  const hasL2 = correction.l2_approver_id !== null;
  switch (correction.status) {
    case 'pending_l1':
      return { l1: 'pending', l2: hasL2 ? 'pending' : 'na' };
    case 'pending_l2':
      return { l1: 'approved', l2: 'pending' };
    case 'approved':
      return { l1: 'approved', l2: hasL2 ? 'approved' : 'na' };
    case 'rejected':
      return { l1: 'rejected', l2: 'na' };
    default: // cancelled
      return { l1: 'na', l2: 'na' };
  }
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `cd asima-frontend && npx vitest run tests/unit/features/time-entries/metrics.spec.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd asima-frontend
git add src/features/time-entries/metrics.ts tests/unit/features/time-entries/metrics.spec.ts
git commit -m "feat(timesheet): derive status, deficit hours, and approver state"
```

---

## Task 5: Frontend — return corrections keyed by date from the hook

**Files:**
- Modify: `asima-frontend/src/features/time-correction/hooks/use-my-active-corrections.ts`

- [ ] **Step 1: Change the hook to return a map**

Rewrite the return so the memo builds a `Map<work_date, TimeCorrectionRequest>`
and rename the hook to `useMyCorrectionsByDate` (active statuses still drive the
fetch — they're exactly the ones the timesheet renders: pending/approved):

```ts
'use client';

import { useMemo } from 'react';
import { useQuery } from '@tanstack/react-query';
import { timeCorrectionApi } from '@/features/time-correction/api';
import type { TcStatus, TimeCorrectionRequest } from '@/features/time-correction/schemas';

const ACTIVE_STATUSES: TcStatus[] = ['pending_l1', 'pending_l2', 'approved'];

/**
 * Map of `work_date` → the active correction for that day, scoped to the date
 * span of the passed entries. The timesheet uses it to (a) disable "Request
 * correction" on days that already have one, and (b) render the Status /
 * Approver / Time-in-out-diff columns. The backend allows at most one active
 * correction per (employee, day), so keying by date is unambiguous.
 */
export function useMyCorrectionsByDate(
  entries: { work_date: string }[],
): Map<string, TimeCorrectionRequest> {
  const dates = entries.map((e) => e.work_date).sort();
  const from = dates[0];
  const to = dates[dates.length - 1];

  const query = useQuery({
    queryKey: ['time-correction', 'me', 'active', from, to],
    queryFn: () => timeCorrectionApi.me.list({ from, to, status: ACTIVE_STATUSES, limit: 100 }),
    enabled: dates.length > 0,
  });

  return useMemo(
    () => new Map((query.data?.data ?? []).map((r) => [r.work_date, r] as const)),
    [query.data],
  );
}
```

- [ ] **Step 2: Type-check (page will fail until Task 6 — that's expected)**

Run: `cd asima-frontend && npx tsc --noEmit`
Expected: error only in `timesheet/page.tsx` (old hook name / Set usage),
fixed in Task 6. No other files reference this hook.

- [ ] **Step 3: Commit**

```bash
cd asima-frontend
git add src/features/time-correction/hooks/use-my-active-corrections.ts
git commit -m "refactor(timesheet): key active corrections by work_date"
```

---

## Task 6: Frontend — merged Time in/out, Status, Approver, Deficit columns

**Files:**
- Modify: `asima-frontend/src/features/time-entries/components/entries-table.tsx`
- Modify: `asima-frontend/src/app/(app)/employee/timesheet/page.tsx`
- Test: `asima-frontend/tests/unit/features/time-entries/entries-table.spec.tsx`

- [ ] **Step 1: Write the failing component tests**

Extend `entries-table.spec.tsx` with cases for the new behavior (reuse the
file's existing render helper / fixtures; adapt prop name to `correctionsByDate`):

```ts
it('renders a merged Time in/out cell with original → proposed when a correction exists', () => {
  const rows = [/* confirmed entry on 2026-06-10, in 09:02 / out 18:30 */];
  const corrections = new Map([['2026-06-10', /* correction proposing in 09:00 / out 18:30 */]]);
  render(<EntriesTable rows={rows} schedules={schedules} correctionsByDate={corrections} />);
  expect(screen.getByText(/in:/i).textContent).toMatch(/→/);
  expect(screen.getByText(/Applied/i)).toBeInTheDocument();
});

it('shows Deficit (h) to two decimals', () => {
  // entry one hour late, schedule 09:00–18:30 → 1.00
  render(<EntriesTable rows={[lateRow]} schedules={schedules} correctionsByDate={new Map()} />);
  expect(screen.getByText('1.00')).toBeInTheDocument();
});

it('shows L1/L2 approver names and per-level state', () => {
  render(<EntriesTable rows={rows} schedules={schedules} correctionsByDate={corrections} />);
  expect(screen.getByText(/Jane Cruz/)).toBeInTheDocument();
  expect(screen.getByText(/Bob Lim/)).toBeInTheDocument();
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `cd asima-frontend && npx vitest run tests/unit/features/time-entries/entries-table.spec.tsx`
Expected: FAIL — `correctionsByDate` prop not supported; new columns absent.

- [ ] **Step 3: Update the table component**

In `entries-table.tsx`:

(a) Imports — add the new metrics + correction type:

```ts
import {
  findScheduleForDate,
  scheduledRegularHours,
  tardinessMinutes,
  undertimeMinutes,
  deficitHours,
  timesheetStatus,
  approverStates,
  type ApproverLevelState,
  type TimesheetStatus,
} from '@/features/time-entries/metrics';
import type { TimeCorrectionRequest } from '@/features/time-correction/schemas';
```

(b) Props — replace `correctedDates?: Set<string>` with:

```ts
  /**
   * `work_date` → active correction for that day. Drives the merged Time
   * in/out diff, Status and Approver columns, and disables "Request
   * correction" on days that already have one.
   */
  correctionsByDate?: Map<string, TimeCorrectionRequest>;
```

(c) Header — replace the `In` / `Out` `<Th>`s with one, and add the three new
columns. Final header row:

```tsx
            <Th>Date</Th>
            <Th>Time in/out</Th>
            <Th className="text-right">Tardiness</Th>
            <Th className="text-right">Undertime</Th>
            <Th className="text-right">Deficit (h)</Th>
            <Th className="text-right">Work hours</Th>
            <Th>Status</Th>
            <Th>Approver</Th>
            <Th className="text-right">Action</Th>
```

(d) Row body — inside `rows.map`, resolve the correction and metrics:

```tsx
            const correction = correctionsByDate?.get(row.work_date);
            const schedule = findScheduleForDate(schedules, row.work_date);
            const late = tardinessMinutes(row, schedule);
            const under = undertimeMinutes(row, schedule);
            const deficit = deficitHours(row, schedule);
            const worked = durationMinutes(row);
            const regular = scheduledRegularHours(schedule);
            const status = timesheetStatus(row, correction);
            const guarded = correctionsByDate?.has(row.work_date) ?? false;
```

Replace the old `In` / `Out` `<Td>`s with one merged cell, and add the new
cells (Deficit after Undertime; Status + Approver before Action):

```tsx
                <Td className="font-medium">{formatDateInTz(row.time_in)}</Td>
                <Td><TimeInOutCell entry={row} correction={correction} /></Td>
                <Td className="text-right tabular-nums"><MinutesCell minutes={late} kind="late" /></Td>
                <Td className="text-right tabular-nums"><MinutesCell minutes={under} kind="under" /></Td>
                <Td className="text-right tabular-nums">
                  {deficit === null ? <span className="text-neutral-400">—</span> : deficit.toFixed(2)}
                </Td>
                <Td className="text-right">
                  <div className="tabular-nums">{formatDuration(worked)}</div>
                  <div className="text-xs text-neutral-500">
                    {regular === null ? '—' : `${regular.toFixed(2)}h regular`}
                  </div>
                </Td>
                <Td><StatusBadge status={status} /></Td>
                <Td><ApproverCell correction={correction} /></Td>
                <Td className="text-right">
                  <div className="flex flex-col items-end gap-1">
                    <button
                      type="button"
                      onClick={() => onRequestCorrection?.(row)}
                      disabled={guarded}
                      className="rounded-md border border-neutral-300 px-2.5 py-1 text-xs font-medium text-neutral-700 hover:bg-neutral-50 focus:outline-none focus:ring-2 focus:ring-neutral-900 disabled:cursor-not-allowed disabled:opacity-50"
                    >
                      Request correction
                    </button>
                    {guarded && <span className="text-xs text-neutral-500">Correction requested</span>}
                  </div>
                </Td>
```

(e) Add the three presentational sub-components (near `MinutesCell`):

```tsx
function TimeInOutCell({
  entry,
  correction,
}: {
  entry: TimeEntry;
  correction?: TimeCorrectionRequest;
}) {
  const inOrig = formatTimeInTz(entry.time_in);
  const outOrig = entry.time_out ? formatTimeInTz(entry.time_out) : '—';
  const inProposed = correction ? formatTimeInTz(correction.proposed_time_in) : null;
  const outProposed = correction?.proposed_time_out
    ? formatTimeInTz(correction.proposed_time_out)
    : null;
  return (
    <div className="font-mono text-xs leading-5">
      <div>
        in:&nbsp;{inOrig}
        {inProposed && <span className="text-neutral-500"> → {inProposed}</span>}
      </div>
      <div>
        out:&nbsp;{outOrig}
        {outProposed && <span className="text-neutral-500"> → {outProposed}</span>}
      </div>
    </div>
  );
}

const STATUS_LABEL: Record<TimesheetStatus, string> = {
  ongoing: 'Ongoing',
  applied: 'Applied',
  approved: 'Approved',
  logged: 'Logged',
};
const STATUS_CLASS: Record<TimesheetStatus, string> = {
  ongoing: 'bg-blue-50 text-blue-700',
  applied: 'bg-amber-50 text-amber-700',
  approved: 'bg-emerald-50 text-emerald-700',
  logged: 'bg-neutral-100 text-neutral-600',
};

function StatusBadge({ status }: { status: TimesheetStatus }) {
  return (
    <span
      className={cn(
        'inline-flex rounded-full px-2 py-0.5 text-xs font-medium',
        STATUS_CLASS[status],
      )}
    >
      {STATUS_LABEL[status]}
    </span>
  );
}

function ApproverCell({ correction }: { correction?: TimeCorrectionRequest }) {
  if (!correction) return <span className="text-neutral-400">—</span>;
  const { l1, l2 } = approverStates(correction);
  return (
    <div className="space-y-0.5 text-xs">
      <ApproverLine level="L1" name={correction.l1_approver_name ?? null} state={l1} />
      <ApproverLine level="L2" name={correction.l2_approver_name ?? null} state={l2} />
    </div>
  );
}

const STATE_MARK: Record<ApproverLevelState, string> = {
  pending: '⏳ pending',
  approved: '✓ approved',
  rejected: '✕ rejected',
  na: 'n/a',
};
const STATE_CLASS: Record<ApproverLevelState, string> = {
  pending: 'text-amber-700',
  approved: 'text-emerald-700',
  rejected: 'text-rose-700',
  na: 'text-neutral-400',
};

function ApproverLine({
  level,
  name,
  state,
}: {
  level: string;
  name: string | null;
  state: ApproverLevelState;
}) {
  return (
    <div>
      <span className="font-medium text-neutral-700">{level}:</span>{' '}
      {state === 'na' ? (
        <span className="text-neutral-400">n/a</span>
      ) : (
        <>
          <span className="text-neutral-800">{name ?? '—'}</span>{' '}
          <span className={STATE_CLASS[state]}>{STATE_MARK[state]}</span>
        </>
      )}
    </div>
  );
}
```

`TimeEntry`, `formatTimeInTz`, `formatDateInTz`, `cn`, `durationMinutes`,
`formatDuration` are already imported (the metrics/types above are added in
step 3a).

- [ ] **Step 4: Wire the page to the renamed hook + map prop**

In `timesheet/page.tsx`:
- Change the import `useMyActiveCorrections` → `useMyCorrectionsByDate`.
- Change `const correctedDates = useMyActiveCorrections(...)` →
  `const correctionsByDate = useMyCorrectionsByDate(listQuery.data?.data ?? []);`
- Change the `<EntriesTable ... correctedDates={correctedDates} />` prop to
  `correctionsByDate={correctionsByDate}`.

- [ ] **Step 5: Run the table tests + type-check**

Run: `cd asima-frontend && npx vitest run tests/unit/features/time-entries/entries-table.spec.tsx && npx tsc --noEmit`
Expected: PASS, no type errors anywhere (page now compiles).

- [ ] **Step 6: Commit**

```bash
cd asima-frontend
git add src/features/time-entries/components/entries-table.tsx \
        src/app/\(app\)/employee/timesheet/page.tsx \
        tests/unit/features/time-entries/entries-table.spec.tsx
git commit -m "feat(timesheet): merged Time in/out diff + Status/Approver/Deficit columns"
```

---

## Task 7: Full verification + push

- [ ] **Step 1: Backend gate**

Run: `cd asima-backend && npm run lint && npm test && npm run build`
Expected: all PASS.

- [ ] **Step 2: Frontend gate**

Run: `cd asima-frontend && npm run lint && npx vitest run && npm run build`
Expected: all PASS.

- [ ] **Step 3: Manual smoke (optional but recommended)**

Run the app, open `/employee/timesheet` as an employee with at least one
day that has a pending correction. Confirm: merged Time in/out shows
`09:02 → 09:00`, Status reads `Applied`, Approver shows L1/L2 names + state,
Deficit (h) matches the minutes columns ÷ 60.

- [ ] **Step 4: Push each repo whose files changed (straight to main)**

```bash
cd asima-backend && git push origin main
cd ../asima-frontend && git push origin main
```

---

## Out of scope (YAGNI)

- No new daily-DTR approval workflow — Status is *derived* from existing
  corrections, not a new sign-off lifecycle.
- No stored `status`/`deficit` columns on `time_entries` — all derived at read
  time, matching the existing metrics approach.
- No admin-timesheet changes — this is the employee self-service surface only.
- No migration — approver-name joins reuse existing FK columns.
