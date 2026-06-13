# Split approvals into two pages + per-row Details drawer

**Date:** 2026-06-13
**Status:** Planned
**Scope:** `asima-frontend` only — no backend changes.

## Problem

Today there is a single combined approvals inbox at `/approvals` that lists
both leave and time-correction requests (a `Kind` column distinguishes them).
A reviewer can Approve/Reject from the row but cannot see the full request —
dates, reason, duration, decision history, or (for leave) the attached
file/image they are being asked to approve.

We want:

1. The combined inbox split into **two dedicated pages**, one per request kind.
2. Each row to gain a **Details** button that opens a drawer showing the full
   request — including the leave attachment when present, and rendering cleanly
   when there is none.
3. The reviewer to be able to **Approve/Reject from inside the drawer**, not
   only from the table row.

| Page                              | URL                          | Sidebar label                          |
|-----------------------------------|------------------------------|----------------------------------------|
| Leave approvals                   | `/leave-approvals`           | `Pending Approvals (Leaves)`           |
| Time-correction approvals         | `/time-correction-approvals` | `Pending Approvals (Time Corrections)` |

The old `/approvals` route is **deleted** (not redirected).

## Key finding — this is frontend-only

The backend already supports every server-side need:

- **`GET /approvals/pending?type=leave|time_correction`** — the inbox endpoint
  already accepts a `type` filter (`QueryPendingApprovalsDto.type`,
  `ApprovalsService.findPending`). Each page just passes its `type`.
- **`GET /leave-requests/:id`** and **`GET /time-correction-requests/:id`** —
  both top-level approver-facing routes use `findByIdForViewer`, which
  authorizes **the requester, the snapshotted L1/L2 approver, OR
  `LEAVE`/`TIME_CORRECTION:ViewAll` / `system_admin`**. So the chain approver
  in this inbox is authorized to fetch the full request for a row they can see.
- **`GET /leave-requests/:id/attachment`** — same `findByIdForViewer` gating;
  the existing `LeaveAttachment` component already calls this top-level route,
  so it is reusable unchanged inside the approver drawer.

**No backend work. No new permissions.** Both pages remain gated by
`APPROVAL:View` (server is the boundary; route gating is UX).

## Architecture

One generic inbox component, parametrized by kind, rendered by two thin pages —
so the table / query / mutation / reject-dialog logic is written once.

```
/leave-approvals            → LeaveApprovalsPage → <ApprovalsInbox
                                                      type="leave"
                                                      title="Pending Approvals (Leaves)"
                                                      DetailDrawer={LeaveApprovalDetailDrawer} />

/time-correction-approvals  → TimeCorrectionApprovalsPage → <ApprovalsInbox
                                                      type="time_correction"
                                                      title="Pending Approvals (Time Corrections)"
                                                      DetailDrawer={TimeCorrectionApprovalDetailDrawer} />
```

`ApprovalsInbox` owns everything the current `ApprovalsPage` does today (it is
the generic replacement, parametrized by kind) — these behaviors **must carry
over**, not be dropped in the rewrite:

- `usePendingApprovals({ type, page, limit })` query.
- **Loading, error, and empty states** matching `approvals-page.tsx` today:
  the `query.isLoading` text, the `query.error` retry `EmptyState` via the
  existing `describeError` helper, and the `ApprovalsEmptyState` (reused as-is;
  it already accepts `canSeeAll`).
- The **`canSeeAll`-driven subtitle** (`APPROVAL:ApproveAny` / `system_admin`
  → org-wide copy vs. "where you are the current approver"). `canSeeAll` is
  computed the same way as today.
- Approve/Reject mutations, reusing the existing `APPROVAL_ACTIONS[kind]`
  registry (`features/approvals/actions.ts`) — no new dispatch logic.
- The `RejectApprovalDialog` (note-capture modal) — **kept here in the shared
  inbox, not duplicated per page** — selected-row state, the `ApprovalsTable`,
  the heading, and the `Paginator`.
- A `busy` flag covering **both** approve and reject pending (today's
  `pendingId` only tracks approve) so the table row and drawer footer disable
  correctly during either action.
- A single `selected: PendingApproval | null` that drives the detail drawer,
  plus a `rejecting: PendingApproval | null` that drives the dialog.

### Reject ↔ drawer coordination

The drawer's Reject and the table's Reject both route through the inbox's
`onReject(row)` → opens `RejectApprovalDialog`. To avoid stacking a dialog over
an open sheet, **clear `selected` (close the drawer) when opening the reject
dialog**. On a **successful approve or reject**, clear `selected` (and
`rejecting`) so the drawer/dialog don't linger over a row that has left the
inbox on refetch. The detail `getOne` should also surface its error state
gracefully (e.g. a 403/404 if a row went terminal between list and open) rather
than crash.

### The Details drawer (approver-facing, new)

Two thin drawers — `LeaveApprovalDetailDrawer` and
`TimeCorrectionApprovalDetailDrawer` — **distinct from** the existing HR
`LeaveDetailDrawer` / `TcDetailDrawer`. The HR drawers are admin-coupled (Edit
and Cancel via `admin.*`, plus an "Approve (override)" path) which an approver
in this inbox cannot use; reusing them would force a `role`-branch, which the
backend/admin conventions explicitly forbid. The codebase already duplicates the
tiny `Detail` presentational helper across both HR drawers, so a fresh, lean
detail body here follows existing precedent rather than introducing a refactor
of the HR drawers.

Each new drawer:

1. Takes the thin `PendingApproval` row, `open` / `onClose`, and
   `onApprove(row)` / `onReject(row)` / `busy` callbacks.
2. Fetches the **full** request by id via a new top-level `getOne`
   (`useQuery`, `enabled` only while open), with a small loading state and an
   error state.
3. Renders read-only detail fields:
   - **Leave:** status, type, dates, duration (full vs. half-day window),
     reason, submitted/decided timestamps + decision note; and
     `<LeaveAttachment requestId={id} />` **only when `attachment_id != null`**
     — absent attachment renders nothing extra (works "regardless of
     attachment").
   - **Time correction:** status, work date, proposed in/out, reason, target
     entry, submitted/decided timestamps + decision note.
4. Footer **Approve / Reject** buttons that call back to the inbox's mutations.
   The drawer does **not** own mutations. Reject reuses the same
   `RejectApprovalDialog` the table uses (the inbox's `onReject(row)` sets the
   rejecting row) — no duplicated note-capture UI.

### Table change

`ApprovalsTable` becomes single-kind:

- **Drop the `Kind` column** (each page is one kind now) and the `KIND_LABEL`
  map.
- Add a **Details** button in the Actions column (always shown, placed before
  Reject/Approve) via a new `onDetails(row)` prop.
- Actions column is always present (both pages provide details + approve/reject).

## API + schema changes (frontend)

- `leaveApi.getOne(id)` → `GET /leave-requests/:id`, parsed by
  `LeaveRequestSchema`. New **top-level** method beside `approve` / `reject` /
  `downloadAttachment` (the leave api already has `me.getOne` / `admin.getOne`;
  this is the third, approver-facing, audience).
- `timeCorrectionApi.getOne(id)` → `GET /time-correction-requests/:id`, parsed
  by `TimeCorrectionSchema` (the TC request schema is exported under that name,
  not `TimeCorrectionRequestSchema`). New top-level method beside `approve` /
  `reject`. Note the TC api currently has **no** `getOne` on either `me` or
  `admin`, so this is the first and only `getOne` on that client.
- `PendingApprovalsQuery` gains `type?: PendingApprovalKind`.
  `approvalsApi.listPending` already spreads params, so it forwards `type`
  automatically.

Detail fetching lives inline in each drawer via `useQuery` (the same inline
pattern `LeaveAttachment` already uses) — no new `hooks/` directories. Use
**distinct detail query keys** (e.g. `['leave', 'request', id]` and
`['time-correction', 'request', id]`) so they don't collide with the broader
`['approvals']` / `['leave']` / `['time-correction']` keys the approve/reject
mutations already invalidate.

## Routing + navigation

- **New routes:**
  - `src/app/(app)/leave-approvals/page.tsx`
  - `src/app/(app)/time-correction-approvals/page.tsx`
  Each wrapped in `<RequirePermission code="APPROVAL:View">`.
- **Delete:** `src/app/(app)/approvals/` (the whole route dir).
- **`app-shell.tsx`:** replace the single `/approvals` "Pending approvals" nav
  item with two items in the existing `approvals` section, both gated by
  `APPROVAL:View`:
  - `Pending Approvals (Leaves)` → `/leave-approvals` (icon `Plane`).
  - `Pending Approvals (Time Corrections)` → `/time-correction-approvals`
    (icon `Clock`).
  - Note `isActive` uses `pathname.startsWith(href)`; the two hrefs share no
    prefix, so no false-active highlighting.
  - `Pending Approvals (Time Corrections)` is a long label — verify it doesn't
    wrap awkwardly in the sidebar at the current width; shorten if it does.

## File inventory

**New**

- `src/features/approvals/components/approvals-inbox.tsx`
- `src/features/approvals/components/leave-approvals-page.tsx`
- `src/features/approvals/components/time-correction-approvals-page.tsx`
- `src/features/approvals/components/leave-approval-detail-drawer.tsx`
- `src/features/approvals/components/time-correction-approval-detail-drawer.tsx`
- `src/app/(app)/leave-approvals/page.tsx`
- `src/app/(app)/time-correction-approvals/page.tsx`

**Edit**

- `src/features/approvals/components/approvals-table.tsx` (drop Kind col, add Details)
- `src/features/approvals/schemas.ts` (`type?` on `PendingApprovalsQuery`)
- `src/features/leave/api.ts` (top-level `getOne`)
- `src/features/time-correction/api.ts` (top-level `getOne`)
- `src/components/layout/app-shell.tsx` (two nav items)

**Delete**

- `src/app/(app)/approvals/` (route dir)
- `src/features/approvals/components/approvals-page.tsx` (replaced by the two pages + inbox)

## Out of scope

- No backend changes (controllers, services, DTOs, permissions all unchanged).
- No changes to the HR `LeaveDetailDrawer` / `TcDetailDrawer`.
- No redirect from `/approvals` — it is deleted.
- No new permission codes.

## Verification

- `npm run lint` and `npm run build` (frontend) pass.
- If the frontend has a test setup, add unit tests for the two new `getOne`
  methods (URL + schema parse) and for the table rendering its Details button.
- Manual: as a chain approver (non-HR), open `/leave-approvals`, click Details
  on a sick-leave row → drawer shows fields + attachment image; Approve/Reject
  from the drawer work and the row leaves the inbox on refetch. Repeat on
  `/time-correction-approvals`. Confirm `/approvals` 404s and the sidebar shows
  the two new entries gated by `APPROVAL:View`.
