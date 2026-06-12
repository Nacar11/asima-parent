# Bulk-assign approvers

**Date:** 2026-06-12
**Status:** Planned
**Surface:** `/admin/approvers` (Approvers admin page)
**Scope:** Frontend (primary) + one small backend filter param.

## Problem

The approvers admin page can today reassign an *existing* approver to another
(`Bulk reassign…`), but there is no way to assign an approver to **many
employees who currently have none**. That is the common day-one case: most
employees start with no approval chain, so there is nothing to reassign
*from*. An admin has to set each employee's L1/L2 one row at a time via the
inline selects.

## Key finding — the backend already supports this

`POST /admin/approvers/bulk-assign` already exists and is fully implemented
(`AdminApprovalChainsController.bulkAssign` →
`ApprovalChainsService.bulkAssign`):

- **Request:** `{ employee_ids: number[], l1_approver_id: number (required),
  l2_approver_id?: number }`
- **Response:** `{ assigned: number, skipped: { employee_id, reason }[] }`
- Overwrites any existing approver at the assigned step(s); L1 is required so
  the "L2 without L1" state cannot arise.
- Employees who ARE the chosen approver are skipped (`reason: 'self_approval'`,
  matching the DB `CHECK (approver_id <> employee_id)`) and reported back.
- All changes commit in one `applyStepChanges` transaction.

The frontend simply never wired it up: `api.ts`, `schemas.ts`, and
`approvers-page.tsx` only know about `bulkReassign`, and there is no
multi-select affordance on the table.

## Design

### 1. Backend — add an `unassigned` list filter (small)

The list endpoint (`GET /admin/approvers`) supports only `search`/`page`/
`limit` today. To make "assign to everyone without an approver" work across
the paginated list (not just the 20 visible rows), add a server-side filter:

- `QueryApprovalChainDto`: add optional `unassigned?: boolean`
  (`@Transform` `'true'`/`'1'` → `true`, default `undefined`).
- `ApprovalChainSearchCriteria`: add the same optional `unassigned?: boolean`.
- `ApprovalChainRepository.listEmployeesWithChains`: when set, add
  `base.andWhere('c1.approver_id IS NULL')` **before** the `getCount()` so
  both `total` and the page slice reflect the filter. The query already
  left-joins the active L1 row as `c1` (`step = 1 AND ended_at IS NULL`), so
  "no active L1" == unassigned. **Add a comment at this `andWhere` citing the
  invariant**: L2-without-L1 is structurally impossible (enforced in
  `setChain`/`bulkAssign`), so the L1 null-check fully defines "unassigned".
  If a future change ever allows a standalone L2, this filter must be
  revisited.

**No criteria-mapping step exists** — resolved from the code:
`AdminApprovalChainsController.list(@Query() query)` passes the DTO straight
to `service.list(query)`, which passes it straight to
`repository.listEmployeesWithChains(criteria)`. `ApprovalChainSearchCriteria`
is a pure structural type and the DTO is duck-typed into it. So adding
`unassigned` to **both** the DTO and the criteria type is sufficient; the repo
reads `criteria.unassigned` with no intermediate wiring. `bulk-assign` itself
is untouched.

### 2. Frontend — schema + api

`src/features/admin-approvers/schemas.ts`:

- `BulkAssignSchema`: `{ employee_ids: z.array(z.number().int().positive()).min(1),
  l1_approver_id: z.number().int().positive(),
  l2_approver_id: z.number().int().positive().optional() }`.
- `BulkAssignResultSchema`: `{ assigned: z.number().int().nonnegative(),
  skipped: z.array(z.object({ employee_id: z.number().int(),
  reason: z.string() })) }`.
- Add `unassigned?: boolean` to `EmployeeChainQuery`.

`src/features/admin-approvers/api.ts`:

- `bulkAssign(input, client?)` → `POST /admin/approvers/bulk-assign`, parse
  with `BulkAssignResultSchema`.
- `allMatchingIds(query, client?)`: helper that pages the list endpoint and
  returns the full set of matching `employee_id`s. Backs the "Select all N
  unassigned" action so it spans every page, not just the loaded one.
  **Correctness — it must genuinely loop, not fire one `limit: 100` call.**
  Start at `page: 1, limit: 100`; after each response, collect
  `data.map(r => r.employee_id)`, and **while `has_more` is true, increment
  `page` and refetch**, accumulating ids until `has_more` is false. A single
  capped call silently truncates any org over 100 employees and breaks
  acceptance criterion #3.

  **Performance — this is the one spot the design can over-pay.** Each list
  call runs 4 LEFT JOINs plus a separate COUNT, and "Select all" fans that out
  to `ceil(N/100)` round-trips. For the id-collection path none of that is
  needed — we only want ids for the filter. Preferred mitigation: a lean
  server affordance that returns just the matching `employee_id`s for the
  filter (no approver joins, no count, no pagination), e.g. a small
  `GET /admin/approvers/ids?unassigned=true&search=…` returning
  `{ employee_ids: number[] }`. If that endpoint is deemed out of scope,
  fall back to the paging loop above and **document the ceiling**: selecting
  all unassigned costs N/100 round-trips of the heavy join query, acceptable
  only because the deployment is single-tenant with a small headcount.

### 3. Frontend — selection UX (checkboxes + unassigned filter)

- **Filter control** above the table: an `All / Only unassigned` toggle bound
  to the `unassigned` query param. Toggling resets `page` to 1 (mirrors the
  existing `debouncedSearch` → page-reset effect).
- **Checkbox column** added to `ApproversTable`. A header checkbox selects /
  deselects every row on the current page. Selection state is a
  `Set<number>` of `employee_id`, **lifted to `ApproversPage`** so it persists
  across pagination and across filter toggles (we hold ids, not row objects).
  Toggling a row or the header is O(page size) on a copied Set — cheap and
  intentional; holding ids (not row snapshots) is what lets selection survive
  paging without stale row data. The header checkbox reflects three states:
  none / some / all of the current page's rows selected.
- **"Select all N unassigned"** affordance, shown only when the unassigned
  filter is active: calls `allMatchingIds({ unassigned: true, search })` and
  unions the result into the selection Set — lets the admin grab everyone
  unassigned in one move.
- **Action bar**, rendered when `selected.size > 0`:
  `"N selected — [Assign approver…] [Clear]"`. `Clear` empties the Set.
- **Dialog** — new `bulk-assign-dialog.tsx`, structured like
  `bulk-reassign-dialog.tsx`:
  - L1 approver select (**required**), L2 approver select (optional, includes
    a `— None —` choice that omits `l2_approver_id`).
  - Submit → `bulkAssign({ employee_ids: [...selected], l1_approver_id,
    l2_approver_id })`.
  - `onSuccess`: toast `"Assigned N employees"` (+ ` (M skipped)` when
    `skipped.length > 0`), clear the selection, and
    `invalidateQueries(['admin-approvers'])`. With the unassigned filter on,
    the just-assigned rows drop out of the view — immediate visual feedback.
  - **The toast count comes from the server response (`result.assigned` /
    `result.skipped.length`), never from the client's `selected.size`.**
    Between "Select all N" resolving and Submit, the selection can go stale
    (another admin assigns someone, an id is no longer unassigned). The
    backend handles this gracefully — overwrite semantics + self-approval
    skip — so it is not a data-integrity bug, but `selected.size` can
    legitimately differ from what actually changed. Reporting the server's
    numbers keeps the toast honest; do not "fix" it to echo the selection.
  - Reuses the same `candidates` list already fetched on the page for the
    inline selects and the reassign dialog.

### Entry point

Both bulk actions sit in the existing header row next to the search box:
`[Bulk reassign…] [Bulk assign…]`. Both gated by `canUpdate`
(`APPROVAL_CHAIN:Update`), consistent with the current page. The server
remains the security boundary (`PermissionsGuard` on the controller).

## Out of scope (YAGNI)

- Per-step bulk *clear* (removing approvers in bulk).
- Bulk-assign by role / department / team.
- Undo after assign.
- Changing the `bulk-assign` request contract (already shipped and correct).

## Touch list

**Backend (`asima-backend`):**
- `src/approval-chains/dto/admin/query-approval-chain.dto.ts` — `unassigned`.
- `src/approval-chains/domain/approval-chain-search-criteria.ts` — `unassigned`.
- `src/approval-chains/persistence/repositories/approval-chain.repository.ts`
  — `andWhere('c1.approver_id IS NULL')` (with the invariant comment).
- `src/approval-chains/approval-chains.service.ts` — **no change needed**;
  `list` already forwards the criteria object straight through (confirmed). If
  the lean ids endpoint is adopted, add a `listIds(criteria)` method + repo
  query and a controller route here instead.
- *(Optional, perf)* lean `GET /admin/approvers/ids` returning
  `{ employee_ids }` for the filter — controller + repo + DTO reuse. Skip if
  the paging-loop fallback is accepted.
- Tests: repository/service coverage for the `unassigned` filter (and the ids
  endpoint if added).

**Frontend (`asima-frontend`):**
- `src/features/admin-approvers/schemas.ts` — bulk-assign schemas + query flag.
- `src/features/admin-approvers/api.ts` — `bulkAssign`, `allMatchingIds`.
- `src/features/admin-approvers/components/approvers-page.tsx` — filter toggle,
  lifted selection state, action bar, second button + dialog.
- `src/features/admin-approvers/components/approvers-table.tsx` — checkbox
  column + header checkbox.
- `src/features/admin-approvers/components/bulk-assign-dialog.tsx` — new.
- Unit tests for the dialog + selection behavior.

## Acceptance criteria

1. Admin can filter the approvers list to only employees with no approver,
   and the filter spans all pages (server-side).
2. Admin can select employees via row checkboxes; selection survives paging
   and filter changes.
3. "Select all N unassigned" selects every unassigned employee across pages.
4. Admin can assign an L1 (required) and optional L2 to all selected
   employees in one action; the result toast reports assigned + skipped.
5. Self-approval rows are skipped server-side and surfaced, not errored.
6. After assigning, the assigned employees disappear from the unassigned view.
7. The whole surface is gated by `APPROVAL_CHAIN:Update`; the server enforces.
