# Implementation Plan: Leave request status column — approvers + relative decision time

## Overview

On `/employee/leaves`, the **My requests** table's `STATUS` column currently
shows only a status badge (e.g. "Approved"). We want it to also show **who
the approvers are** (L1, and L2 when the chain has one) and, for decided
requests, **when** the decision happened in relative form ("2 minutes ago").

The blocker: the self-service list endpoint (`GET /users/me/leave-requests`,
read-model `LeaveRequestListItem`) only joins the **requester's** name
(`employee_name`). Approvers and the decider are present only as integer IDs
(`l1_approver_id`, `l2_approver_id`, `decided_by`). So the work is one
vertical feature with a backend half (surface approver/decider names on the
read-model) and a frontend half (render the enriched cell + relative time).

**Ships as two independent PRs.** Phases 1–2 (Tasks 1–3, status-display
enrichment) and Phase 3 (Tasks 4–5, the cancellation-rule change) share only
the `/employee/leaves` page — they have no code dependency on each other.
Land them as **two separate changes** so each is reviewable and revertable on
its own (change-sizing discipline: one change = one thing).

## Architecture decisions

- **Enrich the existing list read-model, not a new endpoint.** Add
  `l1_approver_name`, `l2_approver_name`, `decided_by_name` to
  `LeaveRequestListItem`, resolved by the same `leftJoinAndSelect` pattern
  already used for `employee_name`. One join-based read-model keeps the HR
  admin list and the employee list on the same contract (no second
  round-trip, no per-row fetch). Rationale mirrors the existing
  `employee_name` decision and the approval-chains `EmployeeChainView`.
- **Reuse existing entity-relation pattern.** The entity already has a
  `@ManyToOne employee` reusing the `employee_id` column. Add three more
  read-only `@ManyToOne` relations (`l1_approver`, `l2_approver`,
  `decided_by_user`) that reuse the existing `l1_approver_id` /
  `l2_approver_id` / `decided_by` columns. No migration — the columns
  already exist; relations are metadata only.
- **Relative time lives in `src/lib/format.ts`.** That file is the single
  render path for user-facing timestamps. `formatDistanceToNow` from
  `date-fns` (already a dep, `^4.1.0`) is timezone-agnostic, so it doesn't
  violate the tz policy — but it still belongs in the one format module so
  callers never reach for `date-fns` directly (frontend-stack §"One date
  library: date-fns").
- **Static render, no live ticking (v0).** "2 minutes ago" is computed at
  render time and refreshes naturally on TanStack Query refetch. No
  `setInterval` re-render loop — flag for v1 if product wants it live.
- **New Layer-3 composition, not a Layer-2 edit.** Build
  `LeaveRequestStatusCell` in `src/features/leave/components/`. It composes
  the existing `LeaveStatusBadge` (unchanged) plus approver/decider lines.
  No new shadcn primitive is required (plain markup + existing badge);
  per the component blueprint, we do NOT install anything.
- **Scope the UI change to the employee page** (what was asked). The
  read-model enrichment is shared, so the same cell can later drop into the
  admin table — but that swap is out of scope here to avoid scope drift.
- **Cancellation broadens from "pending-only" to "active-and-not-past."**
  Today both the service and the table button gate cancel on `isPending`
  (pending_l1 / pending_l2 only). The new rule: a request is cancellable
  while it is in an **active** state (pending *or* approved) **and** the
  leave is **not in the past**. `rejected` and `cancelled` stay terminal.
- **No balance ledger work needed for the broader cancel.** Balances are
  *derived* live from request status — `workingDaySumsByEmployee` sums only
  `approved` (→ used) and pending (→ reserved); `cancelled` is excluded. So
  cancelling an approved leave automatically returns its days to the
  balance, with zero restoration code. (Verified against the balance repo
  query; this is the same reason cancelling a pending request frees its
  reservation today.)

## Status-cell content matrix (the design)

| Status | Badge | Supporting lines |
|---|---|---|
| `pending_l1` | Pending L1 | `Awaiting L1 · {l1_name}` then, if L2 exists, `then L2 · {l2_name}` (muted) |
| `pending_l2` | Pending L2 | `L1 ✓ {l1_name}` (muted) then `Awaiting L2 · {l2_name}` |
| `approved` | Approved | `by {decided_by_name} · {relative}` from `decided_at` (e.g. "2 minutes ago") |
| `rejected` | Rejected | `by {decided_by_name} · {relative}` from `decided_at` |
| `cancelled` | Cancelled | `{relative}` from `cancelled_at` (no approver line) |

Notes:
- "L2 approver (if it has L2)" = render the L2 line only when
  `l2_approver_id !== null`.
- The relative timestamp source is **`decided_at`** for approved/rejected and
  **`cancelled_at`** for cancelled. Pending rows show no relative time.
- `decided_by_name` covers the chain approver **or** an HR override approver
  (the request records whichever decided it); pairing it with relative time
  answers "who approved/rejected and when". Showing an override admin's name
  to the employee is an intentional disclosure (the page exists to show the
  approval chain), not a leak.
- All names degrade gracefully to `—` if the join resolves nothing — the
  real path is a **soft-deleted / deactivated** approver (users use
  `@DeleteDateColumn`; TypeORM filters joined soft-deletable relations by
  `deleted_at`). The `RESTRICT` FK means a referenced user can't be
  hard-deleted, so "missing name" ≈ "approver since deactivated".

## Task list

### Phase 1 — Backend: surface approver + decider names on the list read-model

#### Task 1: Add approver/decider names to `LeaveRequestListItem`

**Description:** Resolve L1 approver, L2 approver, and decider display names
in the paginated list query and expose them on the list read-model, mirroring
the existing `employee_name` join.

**Acceptance criteria:**
- [ ] `LeaveRequestEntity` gains three read-only `@ManyToOne(() => UserEntity)`
      relations — `l1_approver`, `l2_approver`, `decided_by_user` — each with
      `@JoinColumn` reusing the existing `l1_approver_id` / `l2_approver_id` /
      `decided_by` column. `eager: false`, no new DB column, no migration.
- [ ] `LeaveRequestRepository.findAll` `leftJoinAndSelect`s the three
      relations alongside `employee`. Pagination still uses `getManyAndCount`
      and remains correct (all joins are ManyToOne → no row multiplication).
- [ ] `LeaveRequestListItem` declares `l1_approver_name: string | null`,
      `l2_approver_name: string | null`, `decided_by_name: string | null`
      with `@ApiPropertyOptional`.
- [ ] `LeaveRequestMapper.toListItem` populates the three names as
      `${first_name} ${last_name}` (or `null` when the relation is absent),
      same shape as `employee_name`.

**Verification:**
- [ ] Unit: `leave-request.mapper.spec.ts` covers all three names present,
      and `null` when the relation is missing / `l2_approver` is null.
- [ ] `npm run test -- leave-request` passes (mapper + service specs).
- [ ] `npm run build` succeeds.
- [ ] Manual: `GET /api/v1/users/me/leave-requests` returns the three new
      fields on each row (check via `/docs` or curl with a JWT).

**Dependencies:** None.

**Files likely touched:**
- `asima-backend/src/leave-requests/persistence/entities/leave-request.entity.ts`
- `asima-backend/src/leave-requests/persistence/repositories/leave-request.repository.ts`
- `asima-backend/src/leave-requests/domain/leave-request-list-item.ts`
- `asima-backend/src/leave-requests/persistence/mappers/leave-request.mapper.ts`
- `asima-backend/src/leave-requests/persistence/mappers/leave-request.mapper.spec.ts`

**Estimated scope:** Medium (4–5 files).

### Checkpoint: Backend contract ready
- [ ] Mapper + service unit tests pass; build clean.
- [ ] List endpoint returns `l1_approver_name`, `l2_approver_name`,
      `decided_by_name` (manually verified against a seeded request).
- [ ] Review the read-model diff before frontend work depends on it.

### Phase 2 — Frontend: render the enriched status cell

#### Task 2: Add relative-time helper + extend the leave Zod schema

**Description:** Add a single `formatRelative` helper to the format module and
teach the leave schema about the three new wire fields, so the UI has typed,
parsed data to render.

**Acceptance criteria:**
- [ ] `src/lib/format.ts` exports `formatRelative(value: Date | string):
      string` returning e.g. "2 minutes ago" via `date-fns`
      `formatDistanceToNow(..., { addSuffix: true })`. No direct `date-fns`
      import anywhere else.
- [ ] `LeaveRequestSchema` (schemas.ts) adds
      `l1_approver_name: z.string().nullable().optional()`,
      `l2_approver_name: ...`, `decided_by_name: ...` (optional because the
      single-GET response doesn't include them — list-only, same as
      `employee_name`).

**Verification:**
- [ ] Unit: a small spec asserts `formatRelative` produces a suffixed
      relative string for a recent timestamp.
- [ ] `npm run test -- format` passes.
- [ ] `tsc` / `npm run build` clean — schema change type-checks against the
      page.

**Dependencies:** Task 1 (wire contract) for the schema fields to be real.

**Files likely touched:**
- `asima-frontend/src/lib/format.ts`
- `asima-frontend/tests/unit/lib/format.spec.ts` (or existing format spec)
- `asima-frontend/src/features/leave/schemas.ts`

**Estimated scope:** Small (2–3 files).

#### Task 3: Build `LeaveRequestStatusCell` and wire it into the employee table

**Description:** Create the Layer-3 status cell that composes the existing
badge with approver/decider lines and relative time per the content matrix,
and use it in the `STATUS` column of the My requests table.

**Acceptance criteria:**
- [ ] New `src/features/leave/components/leave-request-status-cell.tsx`
      renders, for a given `LeaveRequest` row: the `LeaveStatusBadge` plus the
      supporting lines from the matrix above (L1/L2 for pending, decider +
      relative time for approved/rejected, relative time for cancelled).
- [ ] L2 line renders only when `l2_approver_id !== null`.
- [ ] Missing names render `—`; no crash when `decided_at`/`decided_by_name`
      is null.
- [ ] `employee-leaves-page.tsx` `STATUS` `<Td>` uses the new cell instead of
      the bare `<LeaveStatusBadge>`. No other column changes.
- [ ] Imports obey the blueprint (lucide-react icons only, `cn`, no new UI
      source, no new dependency).

**Verification:**
- [ ] Unit: `leave-request-status-cell.spec.tsx` covers each status branch
      (pending_l1 with/without L2, pending_l2, approved, rejected, cancelled)
      and the `—` fallback.
- [ ] `npm run lint && npm run test && npm run build` all pass.
- [ ] Manual: load `/employee/leaves`; an approved row shows
      "by {name} · N minutes ago"; a pending row shows the L1 (+L2) approver.

**Dependencies:** Task 1, Task 2.

**Files likely touched:**
- `asima-frontend/src/features/leave/components/leave-request-status-cell.tsx`
- `asima-frontend/src/features/leave/components/employee-leaves-page.tsx`
- `asima-frontend/tests/unit/features/leave/leave-request-status-cell.spec.tsx`

**Estimated scope:** Medium (3 files).

### Phase 3 — Cancellation rule: cancel anytime unless the leave is in the past

This phase is independent of the status-display work (Tasks 1–3) and can land
before, after, or in parallel — it touches the cancel path, not the read-model.

#### Task 4: Broaden the backend cancel rule to active-and-not-past

**Description:** Replace the `isPending`-only guard in
`LeaveRequestsService.cancel` with a rule that allows cancelling any
**active** request (pending *or* approved) as long as the leave has **not
fully elapsed**. Terminal states (`rejected`, `cancelled`) stay
non-cancellable. Ownership/permission checks are unchanged.

**Acceptance criteria:**
- [ ] A new predicate (e.g. `isCancellable(row, today)`) returns true when
      `status ∈ {pending_l1, pending_l2, approved}` **and** the leave is not
      fully in the past — i.e. `end_date >= today` (CONFIRMED 2026-06-08: a
      leave in progress is still cancellable; only a wholly-elapsed leave
      locks). `today` is the current date in the configured display tz.
- [ ] `cancel()` throws `409` with a clear message for terminal states
      ("Cannot cancel a request in state …") **and** a distinct `409`
      for already-elapsed leave ("Cannot cancel a leave that has already
      ended"). Message says **ended/elapsed**, not "started".
- [ ] Cancelling an **approved** future request flips it to `cancelled` and
      its days return to the balance automatically (no ledger code) — proven
      by a balance assertion.
- [ ] Ownership + `LEAVE:Delete`/`system_admin` checks unchanged.

**Verification:**
- [ ] **Revise the existing cancel test** at `leave-requests.service.spec.ts`
      (currently "409 when cancelling a terminal request" mocks
      `status: 'approved'` with default dates → asserts 409). Under the new
      rule an approved + future request is **cancellable**, so this test's
      premise is stale: rewrite it to assert (a) approved+future → cancels,
      (b) a truly terminal `rejected`/`cancelled` → 409, (c) approved+elapsed
      → 409. Do not merely add cases — the old assertion must change.
- [ ] Unit: approved-future cancellable; approved-elapsed rejected;
      pending-elapsed rejected; rejected/cancelled rejected; non-owner
      without permission still 403.
- [ ] e2e: cancel an approved future request → 200, balance restored;
      cancel an elapsed request → blocked. **Audit existing cancel e2e for
      any approved-cancel-rejected assertion and revise it too.**
- [ ] `npm run test -- leave-request` + `npm run build` green.

**Dependencies:** None.

**Files likely touched:**
- `asima-backend/src/leave-requests/leave-requests.service.ts`
- `asima-backend/src/leave-requests/leave-requests.constants.ts` (helper, if
  the predicate lives alongside `isPending`)
- `asima-backend/src/leave-requests/leave-requests.service.spec.ts`
- `asima-backend/test/**` (cancel e2e, if present)

**Estimated scope:** Small–Medium (2–4 files).

#### Task 5: Update the table's Cancel-button gating to match

**Description:** The employee table's `ACTION` column shows **Cancel** only
when `isPending(row.status)`. Mirror the backend rule on the client so the
button appears for active, not-yet-past requests (including approved ones).

**Acceptance criteria:**
- [ ] A `canCancel(row)` helper in `features/leave/format.ts` returns true for
      active statuses whose leave is not in the past (same boundary as the
      backend); `employee-leaves-page.tsx` uses it in place of `isPending`.
- [ ] An approved future request shows **Cancel**; a past request shows `—`.
- [ ] The backend remains the source of truth — a slipped-through cancel of a
      past request still surfaces the server error via the existing
      `onError` toast.
- [ ] **Admin drawer kept consistent.** `leave-detail-drawer.tsx` gates its
      "Cancel request" button on `isPending(request.status)` and calls the
      *same* shared `cancel` service. Swap that gate to the same `canCancel`
      helper so an admin can also cancel an approved (not-yet-elapsed)
      request — otherwise employee and admin UIs diverge against one backend
      rule. (Small add to this task; the helper is shared.)

**Verification:**
- [ ] Unit: `canCancel` truth table (approved-future true, approved-past
      false, pending-past false, rejected/cancelled false).
- [ ] `npm run lint && npm run test && npm run build` green.
- [ ] Manual: an approved future request on `/employee/leaves` is cancellable.

**Dependencies:** Task 4 (contract). Independent of Tasks 1–3.

**Files likely touched:**
- `asima-frontend/src/features/leave/format.ts`
- `asima-frontend/src/features/leave/components/employee-leaves-page.tsx`
- `asima-frontend/src/features/leave/components/leave-detail-drawer.tsx` (admin gate)
- `asima-frontend/tests/unit/features/leave/*` (helper spec)

**Estimated scope:** Small–Medium (3–4 files).

### Checkpoint: Cancellation rule shipped
- [ ] Backend + frontend agree on the cancellable predicate and date boundary.
- [ ] Approved-future cancellation frees balance; past cancellation blocked.

### Checkpoint: Feature complete
- [ ] All acceptance criteria met across Tasks 1–5.
- [ ] `npm run lint && npm run test && npm run build` green on both repos.
- [ ] Manual walkthrough of `/employee/leaves` matches the content matrix.
- [ ] Ready for review.

## Risks and mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Three extra joins slow the list query | Low | All ManyToOne; each join resolves against the `users` **primary key** (driven from an already-paginated ≤100-row set), so the unindexed `l1_approver_id`/`l2_approver_id`/`decided_by` columns don't matter here. Negligible; measure only if a slow-query log flags it. |
| Adding entity relations accidentally makes them eager / breaks other queries | Med | `eager: false`; only `findAll` opts in via `leftJoinAndSelect`. Existing `findById`/overlap queries untouched — covered by existing service specs. |
| `decided_by_name` shows an HR override approver, surprising users who expect L1/L2 | Low | Intended — label is "by {name}", not "L1/L2". Chain approvers are shown separately on the approver lines. |
| Relative time goes stale without a refetch | Low | Acceptable for v0; refreshes on query refetch. Live-tick deferred to v1. |

## Open questions (non-blocking — defaults chosen)

- **Live-updating relative time?** Default: no (static, refresh on refetch).
- **Show approver lines on `cancelled` rows?** Default: no — only relative
  cancel time, since no approval occurred.
- **Reuse this cell in the admin table too?** Default: not in this change
  (kept to the requested employee page); the cell is built reusable so a
  follow-up can adopt it.
- **What counts as "in the past" for cancellation (Tasks 4–5)?**
  **RESOLVED 2026-06-08:** a leave is blocked from cancellation only once it
  is **fully elapsed** — `end_date < today`. A leave in progress (started but
  not yet ended) can still be cancelled. The "today" boundary uses the
  configured display tz (Asia/Manila in prod).
