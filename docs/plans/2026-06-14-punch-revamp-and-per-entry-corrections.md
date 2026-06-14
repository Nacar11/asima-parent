# Punch revamp, per-entry corrections, and home dashboard

**Date:** 2026-06-14
**Scope:** cross-cutting (asima-backend + asima-frontend)
**Status:** planned

## Objective

Make the employee punch + timesheet behave the way a normal employee
time-management system does:

1. A **5-minute cooldown** after any punch, so rapid taps can't create
   junk zero-length segments.
2. **Per-entry** time-correction status (fix the "Applied" bleed where one
   correction lights up every row of the same day).
3. A **home dashboard** that shows today at a glance and tells the employee
   whether they punched in **late** against their work schedule.

It also records — **as design only, deferred to the future OT feature** — how
overtime entries are kept out of the tardiness calculation.

## Problem (current behavior)

- **No punch cooldown.** `TimeEntriesService.punch()` is a pure toggle. Rapid
  punches create many tiny same-day segments (the `13:15 → 13:15` rows).
- **Correction status is day-scoped.** Corrections already carry a
  `target_entry_id`, but:
  - the backend enforces *one active correction per `(employee, work_date)`*
    (`findActiveForEmployeeDate`), and
  - the frontend keys the timesheet by `work_date` (`useMyCorrectionsByDate`).

  So one correction marks **every** entry on that day as "Applied" /
  "Correction requested", and a second same-day entry can't be corrected at
  all — which breaks the legitimate **regular-shift + evening-OT** case.
- **Home page is bare.** Just a clock and a toggle button. No cooldown
  feedback, no "today at a glance", no late indication.

## Decisions (locked during brainstorming)

- A day can legitimately hold **multiple entries** (a regular shift plus a
  separate OT session). The model is **per-entry**, not one-row-per-day.
- The "5-minute delay" is a **cooldown after a punch**: the button locks for
  5 minutes after any punch event.
- OT is a **future feature**. This plan makes the model *ready* for it but
  does **not** build OT.

## In scope (build now)

### 1. Punch cooldown — backend-authoritative

**Backend (`asima-backend/src/time-entries/`)**
- Add `PUNCH_COOLDOWN_MINUTES = 5` to `time-entries.constants.ts`.
- New repository method `findLatestForEmployee(employee_id)` → the most recent
  non-deleted entry (by `time_out` if present, else `time_in`).
- In `TimeEntriesService.punch()`: before toggling, compute the **last punch
  event** = the open entry's `time_in` (if clocked in) or the latest entry's
  `time_out` (if clocked out). If `now − lastEvent < 5 min`, reject with
  **HTTP 429** and body `{ status: 429, errors: { cooldown: "<msg>" },
  retry_after_seconds: <n> }`.
- Admin manual create (`create`) and `applyCorrection` are **exempt** — only
  the self-service toggle is rate-limited this way.

**Frontend (`asima-frontend/src/app/(app)/employee/home`)**
- Derive the cooldown countdown client-side from today's entries (last event
  time + 5 min); disable the button and show "Punch out in 4:32".
- On a 429, read `retry_after_seconds` to set the countdown authoritatively
  (tolerates clock drift); toast the cooldown message.

### 2. Per-entry corrections — fix the bleed

**Backend (`asima-backend/src/time-correction-requests/`)**
- Replace the per-day uniqueness check in submit with **per-entry**:
  `findActiveForEntry(target_entry_id)` blocks a second active correction for
  the **same entry**. Two different entries on one day are each correctable.
- Keep the existing null-target (new-log / missed-punch) guard: at most one
  new-log request per date, and only when no entry exists for that date
  (`hasEntryOnDate`).

**Frontend (`asima-frontend/src/features/time-correction` + `time-entries`)**
- Replace `useMyCorrectionsByDate` with **`useMyCorrectionsByEntry`** — a
  `Map<target_entry_id, TimeCorrectionRequest>` over the visible window.
- `EntriesTable` keys Status / Approver / in-out-diff / the "Correction
  requested" guard by **`row.id`**, not `row.work_date`.
- Null-target (new-log) corrections render as their **own pending row** for
  the date, not smeared across existing rows.

### 3. Home dashboard + late detection

**Frontend (`asima-frontend/src/app/(app)/employee/home`)**

A full "today at a glance" dashboard, composed of small sections:
- **Punch control** — the button + live cooldown countdown + disabled state +
  last-punch time.
- **Today's sessions** — each in → out with duration.
- **Total worked today** + **this week's total**.
- **Today's schedule** — expected in/out (from `scheduleApi.mySchedule`).
- **Pending corrections** — a count/link.
- **Late detection** — on punch-in, compare the punch time to today's
  schedule via the existing `findScheduleForDate` + `tardinessMinutes`
  (`metrics.ts`); surface "You're 12 min late" as a toast and a tardiness
  chip on the dashboard. **Reuses `metrics.ts` — no new math.**

## Out of scope — designed, deferred to the OT feature

The following is **recorded for when OT ships**; it is **not built** in this
plan. Rationale: the 5-min cooldown keeps a normal day at one entry, so
tardiness math stays correct today without it.

### OT-ready metric model

- **Entry type `regular | overtime`.** Only `regular` entries are ever
  compared to the work schedule. An `overtime` entry is outside scheduled
  hours by definition, so its tardiness/undertime are `—` and its hours
  accrue as OT (separate column/feature).
- **Classification is by scheduled-window overlap, not chronological order.**
  For the day's window `[expected_in → expected_out]` (which **crosses
  midnight** when `expected_out ≤ expected_in`, i.e. a night shift), the entry
  whose session overlaps the window is `regular`; entries outside it are
  `overtime`. This is what keeps a **morning OT punch from being flagged late
  when the schedule is at night** — the morning punch is never the
  schedule-matched session, so it never receives a tardiness value.
- **How `type` is set (future):** an approved OT request tags its punch
  `overtime`; absent that, classification falls back to window-matching at
  read time.
- **Overnight-window support in `metrics.ts` (prerequisite).** The metric math
  currently compares wall-clock minute-of-day and assumes same-day
  (`expected_out < expected_in` would yield negative undertime). Night-shift
  tardiness/undertime — OT or not — needs windows that cross midnight. This
  lands **with** the OT/night-shift work, not now.

## Data flow & edge cases

- Cooldown countdown is **mirrored** on the frontend but **enforced** on the
  backend; the 429 `retry_after_seconds` is the source of truth on drift.
- Cross-midnight cooldown is negligible (only matters within 5 min); the
  backend uses the true latest entry regardless of date boundaries.
- Re-punch after closing the day (e.g. forgot something) is gated by the
  cooldown like any other punch; genuine mistakes are fixed via a correction.

## Testing

**Backend (jest)**
- Cooldown: clocked-in and clocked-out, just-under vs just-over 5 min, admin
  create exempt, `applyCorrection` exempt.
- Per-entry uniqueness: two entries same day each correctable; same entry
  double-correct blocked; null-target new-log guard unchanged.

**Frontend (vitest)**
- `useMyCorrectionsByEntry` keys by `target_entry_id`; a correction on one
  entry does **not** mark sibling same-day rows.
- Home: cooldown countdown + disabled state; punch-in late detection against a
  schedule (late, on-time, no-schedule → no chip).

## Phasing

1. **Backend cooldown** (constant + repo method + service guard + tests).
2. **Backend per-entry uniqueness** (submit check + tests).
3. **Frontend per-entry correction keying** (hook + `EntriesTable` + tests).
4. **Frontend home dashboard + late detection** (sections + cooldown UI +
   tests).

Each phase is independently shippable; 1–2 are backend-only, 3–4 frontend-only.

## Acceptance criteria

- Punching twice within 5 minutes is rejected (429) and the home button shows
  a live countdown.
- Requesting a correction on one same-day entry marks **only that row**
  Applied; a second same-day entry remains independently correctable.
- The home page shows today's sessions, totals, schedule, and a late chip when
  the employee punches in after `expected_in`.
- The OT-ready design section above is captured for the future OT feature; no
  OT code ships in this plan.
