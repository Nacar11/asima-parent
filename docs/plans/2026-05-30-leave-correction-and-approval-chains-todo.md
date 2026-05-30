# Todo — Leave, Time-Correction, Approval Chains

Companion checklist for [`2026-05-30-leave-correction-and-approval-chains.md`](./2026-05-30-leave-correction-and-approval-chains.md).
Tick boxes as work lands. Each phase ends in a **checkpoint** — STOP for sign-off before continuing.

**Status:** post-review (2026-05-30). All 7 design questions resolved
(Q1, Q2, Q3, Q4, Q5, Q6, Q7). Reflects review findings C1, C2, C7,
C8, C9, A4, A5, S1, S2, S3, P1, P3. No remaining blockers — ready for
Phase 0 implementation pending your final go-ahead.

---

## Phase 0 — Permissions + role grants (backend + frontend catalog)

- [ ] **0.1** Extend `PERMISSION_RESOURCES` in
      `src/permissions/permissions.constants.ts` with `LEAVE`,
      `TIME_CORRECTION`, `APPROVAL_CHAIN`. Extend `PERMISSION_ACTIONS`
      with `Approve`, `ViewOwn`, `ViewAll`.
- [ ] **0.2** Append **16** new rows to
      `src/database/seeds/data/permissions.json` (the table in plan
      §2.2; split View into ViewOwn + ViewAll per Q2).
- [ ] **0.3** Update `src/database/seeds/data/roles.json` per the diff
      in plan §2.3. OPERATIONS_MANAGER gets NO new grants (Q7).
- [ ] **0.4** Seed unit test asserts the exact `(role, permission)`
      grant matrix. Include OPS_MGR has no LEAVE / CORRECTION / CHAIN
      codes.
- [ ] **0.5** `npm run db:fresh && npm run seed` — no errors,
      idempotent on re-run.
- [ ] **0.6** (FRONTEND) Update
      `asima-frontend/src/features/auth/permission-codes.ts` with all
      16 new codes in the static catalog (C9 — TypeScript catches
      route-gate typos against this list).
- [ ] **0.7** (FRONTEND) `npm run test && npm run build` confirms.

> 🛑 **CHECKPOINT A** — sign off on the grant matrix before any new
> module reads these codes.

---

## Phase 1 — `approval-chains` backend module

### 1.1 Schema (one operation per migration — A5)
- [ ] **1.1.1** Migration `CreateApprovalChainsTable` per plan §3.1.
- [ ] **1.1.2** Migration `AlterApprovalChainsTableAddActiveStepUniqueIndex`
      (partial unique on `(employee_id, step) WHERE ended_at IS NULL`).
- [ ] **1.1.3** Verify migration reverts cleanly.

### 1.2 Hexagonal module
- [ ] **1.2.1** `domain/approval-chain.ts` (data class).
- [ ] **1.2.2** `domain/approval-chain-search-criteria.ts`.
- [ ] **1.2.3** `domain/approval-chain-inputs.ts` (`SetChainPatch`,
      `BulkReassignInput`, `RepairPendingInput`).
- [ ] **1.2.4** `domain/find-all-approval-chain.ts`.
- [ ] **1.2.5** `persistence/entities/approval-chain.entity.ts`.
- [ ] **1.2.6** `persistence/mappers/approval-chain.mapper.ts`.
- [ ] **1.2.7** `persistence/base-approval-chain.repository.ts` (abstract).
- [ ] **1.2.8** `persistence/repositories/approval-chain.repository.ts`.
- [ ] **1.2.9** `persistence/persistence.module.ts`.
- [ ] **1.2.10** `dto/admin/{set-chain,bulk-reassign,repair-pending,query-approval-chain}.dto.ts`.
- [ ] **1.2.11** `controllers/admin-approval-chains.controller.ts`
      with 6 routes per plan §6 (GET list, GET one, **PATCH** set
      (A4), POST bulk-reassign, POST repair-pending (C2), DELETE step).
- [ ] **1.2.12** `approval-chains.service.ts`:
      - `setChain` validates approver `is_active`, blocks
        self-assignment.
      - `bulkReassign` skips `employee_id == to_user_id` rows and
        returns `{ reassigned, skipped_self: [...] }` (S1).
      - `repairPending` rewrites `l1_approver_id` / `l2_approver_id`
        on every `pending_*` row in leave + correction tables for the
        given `from_approver_id`. Atomic across both tables.
      - Transactions via `dataSource.manager.transaction()` (P2).
- [ ] **1.2.13** `approval-chains.module.ts` + register in
      `app.module.ts`.
- [ ] **1.2.14** `approval-chains.service.spec.ts` mocks
      `BaseApprovalChainRepository`.
- [ ] **1.2.15** Add `Approval Chains` schema group to `main.ts`
      Swagger.
- [ ] **1.2.16** e2e: `test/approval-chains.e2e-spec.ts` covering
      set → get → reassign → bulk-reassign (with self-skip)
      → repair-pending → self-assign rejection.

### 1.3 Users module — `missing_chain` filter (Q1 fix)
- [ ] **1.3.1** Extend `QueryUserDto` with `filter?: 'missing_chain'`.
- [ ] **1.3.2** `UserRepository.findAll` joins `approval_chains` and
      filters to users with zero active chain rows when
      `filter='missing_chain'`.
- [ ] **1.3.3** Service-level cross-feature DI: `UsersModule` imports
      `ApprovalChainPersistenceModule`. Verify no circular dependency.
- [ ] **1.3.4** Unit test: seeded employee with chain → not in
      missing_chain; employee with no chain → in missing_chain.

### 1.4 Edit-User drawer endpoint wiring
- [ ] **1.4.1** Confirm `PATCH /admin/users/:id` rejects unknown
      `level_1_approver_id` field (forbidNonWhitelisted catches it).
- [ ] **1.4.2** `GET /admin/approvers/:employee_id` returns
      `{ l1: User | null, l2: User | null }` so the drawer can pre-fill.

> 🛑 **CHECKPOINT B** — sign off on chain semantics (versioned rows,
> active-step partial unique index, snapshot policy, repair-pending
> path) before any request type consumes them.

---

## Phase 2 — Admin frontend for chain assignment

### 2.1 Edit drawer L1 / L2 fields
- [ ] **2.1.1** Install shadcn Command (combobox with type-ahead) per
      P1: `npx shadcn@latest add command popover`. NOT `Select` —
      Select truncates silently at >100 active employees.
- [ ] **2.1.2** Add `src/features/admin-approvers/api.ts` (list, get,
      set, bulk-reassign, repair-pending).
- [ ] **2.1.3** Add `src/features/admin-approvers/schemas.ts` (Zod).
- [ ] **2.1.4** Add `src/features/admin-approvers/hooks/use-employee-chain.ts`.
- [ ] **2.1.5** `ApproverCombobox` component — uses shadcn Command +
      Popover; server-side search via `GET /admin/users?search=...&is_active=true`;
      excludes the current employee from results.
- [ ] **2.1.6** Patch `edit-user-drawer.tsx`: insert two
      `ApproverCombobox` fields under Role.
- [ ] **2.1.7** Save handler chains two mutations (profile then chain
      via PATCH); surfaces both errors independently per plan §6.3.
- [ ] **2.1.8** Unit test: re-opening drawer shows previously saved
      L1 / L2 values.

### 2.2 `/admin/approvers` bulk page
- [ ] **2.2.1** `src/app/(app)/admin/approvers/page.tsx` with
      `RequirePermission code="APPROVAL_CHAIN:View"`.
- [ ] **2.2.2** `src/features/admin-approvers/components/approvers-page.tsx`.
- [ ] **2.2.3** `approvers-table.tsx` with `InlineApproverCell` per row.
- [ ] **2.2.4** `bulk-reassign-dialog.tsx`. Shows the
      `skipped_self: [...]` list returned from the backend so HR
      knows which rows still need manual assignment.
- [ ] **2.2.5** `missing-chain-banner.tsx` — shown at top of the page
      when `?filter=missing_chain` hits return non-empty.
- [ ] **2.2.6** Sidebar entry "Approvers" under Administration
      (`src/components/layout/app-shell.tsx`), gated on
      `APPROVAL_CHAIN:View`.
- [ ] **2.2.7** Tests: inline edit fires PATCH once; bulk-reassign
      hits the right endpoint; empty state renders;
      skipped-self list renders.

> 🛑 **CHECKPOINT C** — sign off on chain admin UX before building
> request flows.

---

## Phase 3 — `leave-requests` backend

### 3.1 Domain + persistence (5 migrations per A5)
- [ ] **3.1.1** Migration `CreateLeaveRequestsTable` per plan §3.2.
      Includes `decision_path` enum, `cancelled_at`/`cancelled_by`.
      `l1_approver_id` is NOT NULL (Q1 hard-block guarantees it).
- [ ] **3.1.2** Migration `AlterLeaveRequestsTableAddEmployeeStatusIndex`.
- [ ] **3.1.3** Migration `AlterLeaveRequestsTableAddL1PendingPartialIndex`
      (`(l1_approver_id) WHERE status='pending_l1' AND deleted_at IS NULL`).
- [ ] **3.1.4** Migration `AlterLeaveRequestsTableAddL2PendingPartialIndex`.
- [ ] **3.1.5** Migration `AlterLeaveRequestsTableAddStatusSubmittedIndex`.
- [ ] **3.1.6** Domain files (`leave-request.ts`, `-search-criteria`,
      `-inputs`, `find-all-`).
- [ ] **3.1.7** Entity + mapper + base repo + concrete repo +
      `persistence.module.ts`.
- [ ] **3.1.8** Const-object enums in `leave-requests.constants.ts`:
      `LEAVE_TYPES`, `LEAVE_REQUEST_STATUSES`, `DECISION_PATHS`.

### 3.2 Service
- [ ] **3.2.1** `submit(input)`: snapshot chain; **throw 422 if no L1**
      (Q1); set initial status `pending_l1`.
- [ ] **3.2.2** `submit` Q4 overlap check: reject 422
      `errors.dates` if any pending or approved request for the
      same employee overlaps the proposed date range.
- [ ] **3.2.3** `list(criteria)`: callers with `LEAVE:ViewOwn` only
      see `employee_id = caller.id`; callers with `LEAVE:ViewAll`
      see everything subject to filters.
- [ ] **3.2.4** `cancel(id, actor)`: requester or HR; only `pending_*`;
      sets `cancelled_at`/`cancelled_by` (not `decided_*`).
- [ ] **3.2.5** `approve(id, actor)`: `canActOn` check (with `===`
      and null guards per S3); is_active check on snapshotted
      approver per C2; state advance; set `decision_path`.
- [ ] **3.2.6** `reject(id, actor, note)`: non-empty note (422 else).
- [ ] **3.2.7** `update(id, patch, actor)` Q3: HR-only
      (`LEAVE:Update`); reject 409 if status not in
      `pending_l1`/`pending_l2`. Approved/rejected/cancelled
      rows are immutable.

### 3.3 Controllers + DTOs (URL convention per Q5)
- [ ] **3.3.1** DTOs split: `dto/admin/`, `dto/me/`, `dto/approval/`.
- [ ] **3.3.2** `controllers/admin-leave-requests.controller.ts`
      (`@Permissions(LEAVE: 'ViewAll' | 'Update' | 'Delete')`),
      mounted at `/admin/leave-requests`.
- [ ] **3.3.3** `controllers/me-leave-requests.controller.ts`
      (`@Permissions(LEAVE: 'Create' | 'ViewOwn')`),
      mounted at `/users/me/leave-requests` (existing me-route
      convention).
- [ ] **3.3.4** `controllers/leave-request-approvals.controller.ts`
      with `POST /leave-requests/:id/approve` and `:id/reject`
      (`LEAVE:Approve`). Top-level path — neither admin nor self.
- [ ] **3.3.5** `controllers/leave-requests.controller.ts` with
      `GET /leave-requests/:id` (LEAVE:ViewOwn route gate; service
      grants requester | l1 | l2 | LEAVE:ViewAll holders).
- [ ] **3.3.6** `leave-requests.module.ts` + register in
      `app.module.ts`.

### 3.4 Approvals module integration
- [ ] **3.4.1** Update `approvals.service.ts`: replace placeholder
      with real query over `leave_requests`. Branch on `canSeeAll`.
- [ ] **3.4.2** Update `approvals` DTOs (`PendingApproval` shape).
- [ ] **3.4.3** **New endpoint** `GET /approvals/pending/count` →
      `{ count: number }`. Same scoping logic as the list (P3). Powers
      sidebar badge.
- [ ] **3.4.4** e2e: L1 sees pending_l1 only; L2 sees pending_l2
      only; HR sees both; count endpoint matches list length.

### 3.5 Testing
- [ ] **3.5.1** Service spec covers state machine.
- [ ] **3.5.2** Service spec covers the `canActOn` matrix (chain
      member, non-member, ApproveAny, system_admin,
      deactivated approver).
- [ ] **3.5.3** Service spec covers the cancel-after-L1 case
      (cancelled_at + decided_at both populated).
- [ ] **3.5.4** e2e covers full multi-step flow.

---

## Phase 4 — `leave-requests` frontend

### 4.1 `/employee/leaves` self-service (URL per Q5)
- [ ] **4.1.1** `src/features/leave/{api,schemas,hooks/use-my-leave-requests,hooks/use-submit-leave}.ts`.
- [ ] **4.1.2** `src/features/leave/components/{leave-request-form,my-leave-list,leave-status-badge}.tsx`.
- [ ] **4.1.3** `src/app/(app)/employee/leaves/page.tsx` (plural,
      matches `/employee/timesheet` pattern).
- [ ] **4.1.4** Sidebar entry "Leaves" in the Me cluster.
- [ ] **4.1.5** Surface 422 `errors.approval_chain` "No approver
      assigned" on submit failure with a clear empty-state-style
      message + a "Contact HR" affordance (Q1 UX).
- [ ] **4.1.6** Unit tests on the form + the cancel mutation.

### 4.2 `/admin/leave-requests`
- [ ] **4.2.1** API hooks for admin scope.
- [ ] **4.2.2** Table with filters; row click opens detail drawer.
- [ ] **4.2.3** Drawer HR actions: force-approve (badge "Override"
      visible per S2 `decision_path`), edit (per Q3), cancel.
- [ ] **4.2.4** Sidebar entry under Administration, gated on
      `LEAVE:ViewAll`.

### 4.3 `/approvals` real data + actions
- [ ] **4.3.1** Replace mock query with
      `GET /approvals?type=leave`.
- [ ] **4.3.2** Approve button: fires `POST /leave-requests/:id/approve`.
- [ ] **4.3.3** Reject dialog asks for note; fires
      `POST /leave-requests/:id/reject`.
- [ ] **4.3.4** Optimistic update; refetch on success; rollback
      toast on error.
- [ ] **4.3.5** Empty state copy.
- [ ] **4.3.6** Sidebar badge: poll `GET /approvals/pending/count`
      every 30s via React Query staleTime (P3).

> 🛑 **CHECKPOINT D** — full leave end-to-end demo.

---

## Phase 5 — `time-correction-requests` backend

- [ ] **5.1** Migrations (5 files): `CreateTimeCorrectionRequestsTable`
      + 4 index migrations. Include
      `CHECK (proposed_time_out IS NULL OR proposed_time_out > proposed_time_in)`
      (C8). `decision_path`, `cancelled_at`/`cancelled_by`,
      `l1_approver_id NOT NULL`.
- [ ] **5.2** Domain + persistence + mapper + repo files.
- [ ] **5.3** Q6: add a new method `applyCorrection(request)` to
      `TimeEntriesService` (in the time-entries module). Contract:
      if `target_entry_id` non-null update that row, else create
      a fresh row; `source='correction'`; sets created/updated_by
      from `request.decided_by`. Pre-checks the open-entry partial
      unique index to return a clean 409 before the transaction
      (C7), not after a DB constraint trip.
- [ ] **5.4** Time-correction service `approve` calls
      `applyCorrection` from inside the same transaction as the
      request-status flip. `TimeCorrectionRequestsModule` imports
      `TimeEntriesModule` so the service is injectable.
- [ ] **5.5** DTOs split (admin / me / approval).
- [ ] **5.6** Controllers: admin (`/admin/time-correction-requests`),
      me (`/users/me/time-correction-requests`), approval
      (`/time-correction-requests/:id/{approve,reject}`).
- [ ] **5.7** Update `approvals.service.ts` to UNION corrections
      with leave; update `/approvals/pending/count` accordingly.
- [ ] **5.8** Permission gates use `TIME_CORRECTION:*` codes
      (split ViewOwn / ViewAll).
- [ ] **5.9** Service rules per Q3 / Q4 (mirror leave):
      `update` is HR-only on pending_* only; `submit` rejects 422 if
      a pending or approved correction already exists for the same
      `(employee_id, work_date)`.
- [ ] **5.10** Tests: state machine; chain check; conflict
      (open-entry duplicate) returns 409 at approve time with a
      clear message (C7); overlap rejection.

---

## Phase 6 — `time-correction-requests` frontend

- [ ] **6.1** "Request correction" button on each row of
      `/employee/timesheet`; opens drawer with pre-filled times +
      reason; pre-fills `target_entry_id`.
- [ ] **6.2** `/approvals` row renderer branches on `type` (leave
      vs correction); both actions identical.
- [ ] **6.3** `/admin/time-corrections` HR view (mirror of
      `/admin/leave-requests`). `decision_path='override'` shows
      Override badge.

> 🛑 **CHECKPOINT E** — correction end-to-end demo. Verify
> approving a correction mutates the underlying `time_entries`
> row with `source='correction'`.

---

## Final gate

- [ ] **F1** Full backend test suite green
      (`npm run test && npm run test:e2e`).
- [ ] **F2** Full frontend test suite green (`npm run test`).
- [ ] **F3** `npm run build` green on both repos.
- [ ] **F4** Grant matrix lock — `psql` query confirms the Phase 0
      matrix is unchanged from what the seed test asserted.
- [ ] **F5** Spot-check: every new module's docstring references
      ADR 0001 where relevant.
- [ ] **F6** Sidebar audit: every new sidebar entry is gated on
      the correct permission code.
- [ ] **F7** **Audit surface check**: `decision_path='override'`
      rows are visibly distinguished in admin views from
      `decision_path='chain'`.

> 🛑 **CHECKPOINT F** — final sign-off before rollout.
