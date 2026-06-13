# Plan — Leave, Time-Correction, and 2-Level Approval Chains

**Date:** 2026-05-30
**Owner:** backend → frontend
**Scope:** asima-backend (new modules + permission catalog) + asima-frontend (admin surfaces + self-service flows)
**Mode:** PLAN ONLY — no code in this PR. Implementation lands phase-by-phase per the checkpoints in §9.

> **Read first:**
> - `asima-parent/docs/universal-guidelines/module-architecture.md` —
>   backend hexagonal blueprint.
> - `asima-parent/docs/universal-guidelines/frontend-component-blueprint.md`
>   — prompt-context rules for any frontend work.
> - `asima-backend/docs/adr/0001-roles-and-approval-design.md` — the
>   already-accepted ADR that pins the role-vs-chain split. Everything
>   below honors that decision; nothing in this plan revises it.
> - `asima-backend/docs/schema.dbml` — `approval_chains` schema already
>   designed with versioned assignments. Same in this plan.
> - `asima-parent/CLAUDE.md` §"Admin vs. self-service" — the audience-split
>   contract every new resource must follow.

---

## 0. Headline summary

You asked five questions. Direct answers up front so the rest of the
plan is the **how**, not the **what**:

| Question | Answer |
|---|---|
| Does the admin Edit-User drawer need L1 / L2 approver fields? | **Yes.** Two new comboboxes ("Level 1 approver", "Level 2 approver") under Role. Backed by a new `approval_chains` table — not by columns on `users`. |
| Is an admin "Approvers" page (bulk reassignment) necessary? | **Yes — and it's cheap.** Same backend endpoint as the drawer; just a table-driven UI for org-wide reassignments (manager leaves, reporting line changes). Without it, every reassignment is a click-into-drawer-per-employee chore. Ship it. |
| Do leave and time-correction share the same chain? | **Yes.** One `approval_chains` table powers both. ADR 0001 mandates this. Adding OT, expenses, etc. later is the same table with no schema change. |
| Are L1 and L2 BOTH required? | **Recommended yes, with two carve-outs:** (1) if an employee has only L1 assigned, the request advances to "approved" after L1 — supports "employee reports directly to PM"; (2) any user with `LEAVE:ApproveAny` / `TIME_CORRECTION:ApproveAny` can override at any step. Documented in §3.3. |
| What if NO chain is assigned at all? | **Hard-block submission.** Return 422 `{ errors: { approval_chain: "No approver assigned. Contact HR." } }`. HR has a `/admin/users?filter=missing_chain` lookup to find unconfigured employees. No silent rubber-stamping. (Q1 decided 2026-05-30.) |
| Where do approve / reject endpoints live? | **On each request resource at the top level.** Backend: `POST /leave-requests/:id/approve`, `:id/reject`. Self-service routes use the existing `/users/me/<resource>` convention (`/users/me/leave-requests`). Frontend URLs follow the existing `/employee/<thing>` convention (`/employee/leaves`). (Q5 decided 2026-05-30.) |

### Resolved design questions (2026-05-30)

All seven open questions answered. No outstanding blockers before
implementation.

| # | Question | Decision |
|---|---|---|
| Q1 | Submission with no chain assigned | **Hard-block (422)** with `errors.approval_chain` |
| Q2 | Leave / correction view scoping | **Two codes per resource** — `ViewOwn` + `ViewAll` |
| Q3 | HR editing an approved request | **Pending-only edits.** `LEAVE:Update` / `TIME_CORRECTION:Update` only allow editing `pending_l1` / `pending_l2`. Approved/rejected/cancelled rows are immutable. Edits to terminal-state rows go through cancel + new submission. |
| Q4 | Overlapping leave requests | **Block overlap of pending/approved.** Submission rejected with 422 if any pending or approved request overlaps proposed dates. Cancelled/rejected don't count. HR can override via `LEAVE:ApproveAny` if a legitimate stacking case appears. |
| Q5 | URL convention | Backend `/users/me/<resource>` + frontend `/employee/<thing>` |
| Q6 | Time-correction approval mutation | **Via `TimeEntriesService.applyCorrection(request)`.** Time-correction module owns the request lifecycle; time-entries module owns the `time_entries` table write. Cross-module call via the `TimeEntriesService` (not the repo). |
| Q7 | OPERATIONS_MANAGER scope | No involvement in v1 — keep current empty permission set |

---

## 1. Goal and scope

**Goal.** Ship the leave + time-correction request flows with a 2-step
approval chain (TD → PM by default, but the chain is configurable per
employee). HR (and SUPER_ADMIN) can override at any step.

**In scope this plan:**
- New backend modules: `approval-chains`, `leave-requests`, `time-correction-requests`.
- New permission catalog entries: `LEAVE:*`, `TIME_CORRECTION:*`,
  `APPROVAL_CHAIN:*`.
- Updated role grants per ADR 0001's taxonomy.
- Existing `approvals` module: replace the empty-payload placeholder
  with real `findPending` queries over the new tables.
- Admin Edit-User drawer: add L1 / L2 approver dropdowns.
- New admin page `/admin/approvers`: bulk view + reassign.
- New employee self-service pages: `/me/leave` and "request correction"
  affordance on the existing timesheet.
- The `/approvals` page (already exists) wired to real data + approve / reject buttons.

**Out of scope (§11 spells these out):** leave-type catalog deeper than a
small enum, leave balances / accruals, overtime requests, expense
approvals, push notifications to approvers, email digests, approval
delegation, "acting approver" UI.

---

## 2. Authorization model — the STRICT contract

This is the most important section. Get this wrong and every later
slice is patched into shape.

### 2.1 Two orthogonal axes

ADR 0001 says it; we restate it for code review:

| Axis | What it answers | Stored where | Who edits it |
|---|---|---|---|
| **Role** | "Can this user ever approve leave at all?" | `users.role_id` → `role_permissions` → permission codes | HR via `/admin/users/:id` (role swap) and `/admin/roles/:id/permissions` (catalog edit) |
| **Approval chain** | "For THIS employee's request, who is the assigned approver at each step?" | `approval_chains(employee_id, step, approver_id, effective_at, ended_at)` | HR via `/admin/approvers` (bulk) or `/admin/users/:id` drawer (per-employee) |
| **System override** | "Skip the chain entirely" | `users.system_admin` boolean + `*:ApproveAny` codes | Seed-managed (`system_admin`); HR can grant `ApproveAny` via role-permission edits |

Two checks at request-approval time, in order:

```
1. PermissionsGuard      → does caller's role grant LEAVE:Approve?
                            (declarative @Permissions decorator)
2. Service-layer gate    → for THIS request, is caller the assigned approver
                            for the current step  OR  does caller have
                            LEAVE:ApproveAny (or system_admin)?
```

Both must pass. Neither replaces the other. **There is no third "auth-via-title" axis** — `users.title` is freeform display string and must NEVER be read by approval code (ADR 0001 §2).

### 2.2 Permission catalog — full diff

Adding to `permissions.constants.ts` and seeding via
`src/database/seeds/data/permissions.json`. **16 new codes** (Q2
resolved 2026-05-30: split `View` into `ViewOwn` + `ViewAll` for both
LEAVE and TIME_CORRECTION so the code itself encodes the scope —
eliminates the "did I scope this query?" failure mode):

| New code | Resource | Action | Description |
|---|---|---|---|
| `LEAVE:Create` | LEAVE | Create | Submit a leave request for myself |
| `LEAVE:ViewOwn` | LEAVE | ViewOwn | See my own leave requests |
| `LEAVE:ViewAll` | LEAVE | ViewAll | See every leave request across the org (HR) |
| `LEAVE:Update` | LEAVE | Update | Edit a leave request (HR only — see §2.4) |
| `LEAVE:Delete` | LEAVE | Delete | Cancel any leave request (HR override) |
| `LEAVE:Approve` | LEAVE | Approve | Capability to approve leave requests (chain-gated in service) |
| `LEAVE:ApproveAny` | LEAVE | ApproveAny | Override: approve any leave request regardless of chain placement |
| `TIME_CORRECTION:Create` | TIME_CORRECTION | Create | Submit a time-correction request for myself |
| `TIME_CORRECTION:ViewOwn` | TIME_CORRECTION | ViewOwn | See my own correction requests |
| `TIME_CORRECTION:ViewAll` | TIME_CORRECTION | ViewAll | See every correction request across the org (HR) |
| `TIME_CORRECTION:Update` | TIME_CORRECTION | Update | Edit a request (HR only) |
| `TIME_CORRECTION:Delete` | TIME_CORRECTION | Delete | Cancel a request |
| `TIME_CORRECTION:Approve` | TIME_CORRECTION | Approve | Capability to approve corrections (chain-gated) |
| `TIME_CORRECTION:ApproveAny` | TIME_CORRECTION | ApproveAny | Override |
| `APPROVAL_CHAIN:View` | APPROVAL_CHAIN | View | See who is whose approver |
| `APPROVAL_CHAIN:Update` | APPROVAL_CHAIN | Update | Assign / reassign approvers |

Three new resources added to `PERMISSION_RESOURCES`: `LEAVE`,
`TIME_CORRECTION`, `APPROVAL_CHAIN`. Three new actions to
`PERMISSION_ACTIONS`: `Approve`, `ViewOwn`, `ViewAll` (`ApproveAny`
already exists from the approvals module).

**Approver inbox access** uses the existing `APPROVAL:View` code (already
in the seeds) — that's how PM/TD see their pending items. `LEAVE:Approve`
is the gate on the action itself. `LEAVE:ViewOwn` lets the approver see
their own personal leave requests they submitted.

**Approver detail-view access** for a specific request they're about to
act on: `GET /leave-requests/:id` accepts callers who are (a) the
requester (`employee_id === caller.id`), (b) have `LEAVE:ViewAll`, OR
(c) appear as `l1_approver_id` or `l2_approver_id` on the snapshot.
Service-layer check, declared on the route as `LEAVE:ViewOwn`.

### 2.3 Role grants — full diff

Updating `src/database/seeds/data/roles.json`. (Q7 resolved 2026-05-30:
OPERATIONS_MANAGER gets no new grants in v1.)

| Role | Add codes |
|---|---|
| `SUPER_ADMIN` | `LEAVE:*` (all seven incl. both View flavors), `TIME_CORRECTION:*` (all seven), `APPROVAL_CHAIN:View`, `APPROVAL_CHAIN:Update` |
| `HR_ADMIN` | `LEAVE:ViewAll`, `LEAVE:ViewOwn`, `LEAVE:Update`, `LEAVE:Delete`, `LEAVE:ApproveAny`, `TIME_CORRECTION:ViewAll`, `TIME_CORRECTION:ViewOwn`, `TIME_CORRECTION:Update`, `TIME_CORRECTION:Delete`, `TIME_CORRECTION:ApproveAny`, `APPROVAL_CHAIN:View`, `APPROVAL_CHAIN:Update` |
| `OPERATIONS_MANAGER` | *(no new grants — keeps current empty set; revisit when concrete OPS use case appears)* |
| `PROJECT_MANAGER` | `LEAVE:Approve`, `LEAVE:ViewOwn`, `TIME_CORRECTION:Approve`, `TIME_CORRECTION:ViewOwn` *(approve capability — chain placement still required for a specific request; ViewOwn lets them see their own personal leave they submitted)* |
| `TECHNICAL_DIRECTOR` | `LEAVE:Approve`, `LEAVE:ViewOwn`, `TIME_CORRECTION:Approve`, `TIME_CORRECTION:ViewOwn` *(same as PM — PM/TD distinction lives in the chain, per ADR 0001)* |
| `EMPLOYEE` | `LEAVE:Create`, `LEAVE:ViewOwn`, `TIME_CORRECTION:Create`, `TIME_CORRECTION:ViewOwn` |

**Why `ViewOwn` + `ViewAll` instead of one `View` code:** if the service
forgets to scope `LEAVE:View` to own-rows for a non-HR caller, every
employee sees everyone's leave. With two codes, an EMPLOYEE simply
can't pass the `LEAVE:ViewAll` route guard at all — the failure mode
is 403, not data leak. Stronger boundary, slightly larger catalog.

### 2.4 Why `LEAVE:Update` is HR-only and not the requester's

A submitted leave request is an audit object. The requester should not
edit it after submission — that breaks the chain (an approver could
approve "5 days" and find it silently became "10 days" before payroll).
Instead the requester gets `cancel + resubmit`. HR can edit (e.g. fix a
date typo on behalf of an employee) under `LEAVE:Update`.

This is a deliberate departure from the "user owns their own data"
default. Document on the request entity that this is intentional.

---

## 3. Data model

### 3.1 `approval_chains` — already designed in `schema.dbml`

Reusing the exact shape from ADR 0001 + schema.dbml. **No deviation.**

```
id            serial PK
employee_id   int FK → users.id        (the user whose request is routed)
step          int NOT NULL              (1 = first approver, 2 = second, ...)
approver_id   int FK → users.id        (who can approve at this step)
effective_at  timestamptz DEFAULT now()
ended_at      timestamptz NULL          (NULL = currently active)
created_by    int FK → users.id
updated_by    int FK → users.id

UNIQUE (employee_id, step, effective_at)
INDEX  (employee_id, ended_at)
INDEX  (approver_id)
```

**No soft-delete on this table.** Reassignment is modeled as "end the
old row (set `ended_at = now`), insert a new row". Historical leave
requests then resolve their original approver by looking up the row
that was active at the request's `submitted_at` — even after the chain
is changed. This is the same logical-end pattern as `work_schedules`.

**Active-row constraint.** Add a partial unique index in the
migration:

```sql
CREATE UNIQUE INDEX approval_chains_active_step_uq
  ON approval_chains (employee_id, step)
  WHERE ended_at IS NULL;
```

Guarantees "one active approver per (employee, step)" at the DB level.
Service-layer pre-check returns 422; the index is the safety net.

### 3.2 `leave_requests`

```
id                serial PK
employee_id       int FK → users.id        (the requester)
leave_type        enum('annual','sick','bereavement','unpaid','other')
start_date        date NOT NULL
end_date          date NOT NULL  CHECK (end_date >= start_date)
reason            varchar(500) NULL
status            enum('pending_l1','pending_l2','approved','rejected','cancelled')
                                           DEFAULT 'pending_l1'
submitted_at      timestamptz DEFAULT now()
decided_at        timestamptz NULL         (set on approved/rejected)
decided_by        int FK → users.id NULL   (who closed the request)
decision_note     varchar(500) NULL        (rejection reason, etc.)
decision_path     enum('chain','override') NULL  (S2 fix — chain = normal
                                                  approver acted; override
                                                  = ApproveAny/system_admin
                                                  used the bypass path)
cancelled_at      timestamptz NULL         (separate from decided_* so a
                                            cancel-after-L1-approval keeps
                                            both events visible)
cancelled_by      int FK → users.id NULL

-- Snapshot of the chain at submission time. Lets historical requests
-- show "this was approved by X who was the L1 then" even after the
-- chain is reassigned. l1 is guaranteed non-null after Q1 hard-block;
-- l2 may be null (single-step chain → auto-approve after L1).
l1_approver_id    int FK → users.id NOT NULL
l2_approver_id    int FK → users.id NULL

-- audit
created_by …, updated_by …, deleted_by …, created_at, updated_at, deleted_at
```

**Status state machine:**

```
                  ┌──────────────┐
                  │ pending_l1   │ ←── submit (employee)
                  └─┬──────┬─────┘
       L1 approves  │      │ L1 rejects → 'rejected'
                    │      │ caller cancels → 'cancelled'
                    ▼      │
                ┌──────────────┐
                │ pending_l2   │     (skipped if no L2 assigned at submit)
                └─┬──────┬─────┘
     L2 approves  │      │ L2 rejects → 'rejected'
                  │      │ caller cancels → 'cancelled'
                  ▼      │
                ┌──────────────┐
                │ approved     │ ── terminal
                └──────────────┘
```

**Approve / reject authorization (service layer):**

```
function canActOn(request, caller):
  // 1. Permission gate already passed (LEAVE:Approve on the route)
  // 2. ApproveAny / system_admin bypass:
  if caller.system_admin: return true
  if caller has LEAVE:ApproveAny: return true
  // 3. Chain placement:
  if request.status == 'pending_l1': return caller.id == request.l1_approver_id
  if request.status == 'pending_l2': return caller.id == request.l2_approver_id
  return false  // terminal or unknown state
```

**Indexes:**
- `(employee_id, status)` — list mine.
- `(l1_approver_id) WHERE status='pending_l1' AND deleted_at IS NULL` — inbox query for L1 approvers.
- `(l2_approver_id) WHERE status='pending_l2' AND deleted_at IS NULL` — inbox query for L2 approvers.
- `(status, submitted_at)` — admin sort.

### 3.3 `time_correction_requests`

Similar shape, plus a pointer to the entity being corrected.

```
id                  serial PK
employee_id         int FK → users.id
target_entry_id     int FK → time_entries.id NULL  (NULL = missed-punch; no row to correct)
work_date           date NOT NULL                  (denormalized for query speed)
proposed_time_in    timestamptz NOT NULL
proposed_time_out   timestamptz NULL  (NULL = open segment)
reason              varchar(500) NOT NULL
status              enum('pending_l1','pending_l2','approved','rejected','cancelled')
submitted_at        timestamptz DEFAULT now()
decided_at          timestamptz NULL
decided_by          int FK → users.id NULL
decision_note       varchar(500) NULL
decision_path       enum('chain','override') NULL  (same audit shape as leave)
cancelled_at        timestamptz NULL
cancelled_by        int FK → users.id NULL
l1_approver_id      int FK → users.id NOT NULL
l2_approver_id      int FK → users.id NULL

CHECK (proposed_time_out IS NULL OR proposed_time_out > proposed_time_in)
                                    -- C8 fix; mirrors the existing
                                    -- time_entries CHECK constraint

-- audit columns identical to leave_requests
```

**On approve (terminal state):** the service creates or updates the
underlying `time_entries` row using `source = 'correction'` (the enum
value already exists in `schema.dbml`). The request entity becomes
the audit record; the `time_entries` row is the source of truth for
the timesheet.

**Conflict handling.** If approving would create a second open entry
for the employee on the same `work_date`, throw 409. The existing
`time_entries` partial unique index already enforces this — we just
translate the Postgres `23505` into a friendly message.

### 3.4 No `leave_types` table for v0

Five values (`annual`, `sick`, `bereavement`, `unpaid`, `other`) cover
the common cases. A separate `leave_types` table with quotas, accrual
rules, etc. is a v1 conversation (§11). Use a Postgres enum + const
object pattern (mirrors `TIME_ENTRY_STATUSES`).

---

## 4. Module list and dependency graph

```
┌─────────────────────────────────────────────────────────────────┐
│ Phase 0  Permissions catalog + role grants (seed-only changes)  │
└──────────────────────────┬──────────────────────────────────────┘
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
┌─────────────────────────┐  ┌─────────────────────────────────┐
│ Phase 1  approval-chains│  │ Phase 0.5  (parallel)           │
│   backend module        │  │   Leave enum constants + types  │
│   (depends only on      │  │   (no migration; just TS)       │
│    users)               │  └─────────────────────────────────┘
└─────────────┬───────────┘
              │
   ┌──────────┴──────────┐
   ▼                     ▼
┌─────────────────┐  ┌─────────────────────────────────────────┐
│ Phase 2         │  │ Phase 3   leave-requests backend module │
│  admin frontend │  │   (depends on approval-chains for chain │
│  for chains:    │  │   snapshot on submit; on users for FKs) │
│  - Edit drawer  │  └────────────────┬────────────────────────┘
│    L1/L2 fields │                   │
│  - /admin/      │                   ▼
│    approvers    │  ┌─────────────────────────────────────────┐
│    page         │  │ Phase 4   leave-requests frontend       │
└─────────────────┘  │   - /me/leave (submit + my list)        │
                     │   - /admin/leave-requests (HR view)     │
                     │   - /approvals: real data + actions     │
                     └────────────────┬────────────────────────┘
                                      │
                                      ▼
                  ┌──────────────────────────────────────────────┐
                  │ Phase 5   time-correction-requests backend   │
                  │   (depends on approval-chains, time-entries) │
                  └────────────────┬─────────────────────────────┘
                                   │
                                   ▼
                  ┌──────────────────────────────────────────────┐
                  │ Phase 6   time-correction-requests frontend  │
                  │   - "Request correction" affordance on the   │
                  │     existing /employee/timesheet page        │
                  │   - /approvals: handles correction items     │
                  │   - /admin/time-corrections                  │
                  └──────────────────────────────────────────────┘
```

Critical edges:
- Phase 0 unblocks everything else.
- Phase 1 unblocks Phases 2, 3, and 5 in parallel (Phase 2 needs the
  endpoints, Phase 3 needs the snapshot helper, Phase 5 same).
- Phase 3 must land before Phase 4 (frontend has nothing to call).
- Phases 5/6 can run after Phase 4 ships, OR in parallel by a second
  pair if you want to fan out.

---

## 5. Phase 0 — Permissions + role grants

### Slice 0.1 — Add the new permission codes

**Files:**
- `src/permissions/permissions.constants.ts` — extend
  `PERMISSION_RESOURCES` with `LEAVE`, `TIME_CORRECTION`,
  `APPROVAL_CHAIN`. Extend `PERMISSION_ACTIONS` with `Approve`.
- `src/database/seeds/data/permissions.json` — append the 14 rows from
  §2.2.

**Acceptance:**
- [ ] `npm run seed` is idempotent (re-running is a no-op).
- [ ] `SELECT code FROM permissions WHERE code LIKE 'LEAVE:%'` returns 6 rows.
- [ ] Same for `TIME_CORRECTION:%` (6 rows) and `APPROVAL_CHAIN:%` (2 rows).

**Verification:**
```bash
npm run seed && psql … -c "SELECT code FROM permissions ORDER BY code"
```

### Slice 0.2 — Update role grants

**Files:**
- `src/database/seeds/data/roles.json` — apply the diff in §2.3.

**Acceptance:**
- [ ] HR_ADMIN seed user can call any `LEAVE:Approve`-gated endpoint that lands later.
- [ ] EMPLOYEE seed user can NOT call any admin or approve route.
- [ ] PROJECT_MANAGER + TECHNICAL_DIRECTOR have identical permission
      sets (the chain alone differentiates them).
- [ ] Re-running `npm run seed` does not produce duplicate grants
      (uses the existing idempotent natural-key upsert).

**Verification:** unit test on the seed service that asserts the
expected `(role, permission)` pairs after seeding.

---

## 6. Phase 1 — `approval-chains` backend module

### Slice 1.1 — Schema

**Migration:** `CreateApprovalChainsTable` per the schema in §3.1, plus
the partial unique index `approval_chains_active_step_uq`.

**Acceptance:**
- [ ] Migration runs cleanly on a fresh DB via `npm run db:fresh`.
- [ ] Migration reverts cleanly (`migration:revert`).
- [ ] Concurrent inserts of two active rows for the same `(employee, step)` produce one success and one `23505`.

### Slice 1.2 — Hexagonal module per blueprint

**Files** (mirror `src/users/` structure):
```
src/approval-chains/
├── domain/
│   ├── approval-chain.ts                  (data class: id, employee_id, step,
│   │                                       approver_id, effective_at, ended_at, audit)
│   ├── approval-chain-search-criteria.ts  (filter shape)
│   ├── approval-chain-inputs.ts           (AssignInput, ReassignInput)
│   └── find-all-approval-chain.ts
├── persistence/
│   ├── base-approval-chain.repository.ts  (abstract class)
│   ├── entities/approval-chain.entity.ts
│   ├── mappers/approval-chain.mapper.ts
│   ├── repositories/approval-chain.repository.ts
│   └── persistence.module.ts
├── dto/admin/
│   ├── assign-approver.dto.ts             ({ step, approver_id })
│   ├── set-chain.dto.ts                   ({ l1_approver_id, l2_approver_id })
│   ├── query-approval-chain.dto.ts
│   └── bulk-reassign.dto.ts               ({ from_approver_id, to_approver_id })
├── controllers/admin-approval-chains.controller.ts
├── approval-chains.service.ts
└── approval-chains.module.ts
```

**Endpoints (all `@Permissions(APPROVAL_CHAIN: 'View' | 'Update')`):**

| Method | Path | Permission | Purpose |
|---|---|---|---|
| GET | `/admin/approvers` | View | Paginated list of all employees with their current L1/L2 approver |
| GET | `/admin/approvers/:employee_id` | View | Get a single employee's current chain (both steps) |
| PATCH | `/admin/approvers/:employee_id` | Update | Set/clear one or both steps; both fields optional + nullable (sending only L1 keeps L2 unchanged; sending L1=null clears L1). The Edit drawer endpoint. |
| POST | `/admin/approvers/bulk-reassign` | Update | "Where X was an approver, replace with Y." Skips rows where `employee_id == to_user_id` (S1 / self-assignment guard) and returns the skipped list in the response. |
| POST | `/admin/approvers/repair-pending` | Update | Take `{from_approver_id, to_approver_id}` — rewrite the snapshotted `l1_approver_id`/`l2_approver_id` on every `pending_*` row across leave + correction tables. Used when an approver is deactivated and HR needs to advance their queue. (C2 fix.) |
| DELETE | `/admin/approvers/:employee_id/:step` | Update | End an assignment without replacing (chain shrinks) |

Also add a `?filter=missing_chain` query option to `GET /admin/users`
(lives in the existing users module, not approval-chains) — returns
active users with no L1 row. Drives the new-hire onboarding workflow
HR needs after Q1's hard-block decision.

**Key service behaviors:**
- `setChain(employee_id, { l1_approver_id, l2_approver_id }, actor_id)`:
  end any currently-active rows for both steps, insert new rows in one
  transaction. The PUT endpoint maps to this.
- `getActive(employee_id)`: returns `{ l1, l2 }` lookup of active rows.
- Validate: approver `is_active = true`, approver `!= employee_id`
  (no self-approval), approver exists.
- `bulkReassign(from_id, to_id)`: end every active row where
  `approver_id = from_id`, insert mirror rows pointing to `to_id`.

**Acceptance:**
- [ ] Setting L1+L2 on a new employee creates 2 rows, both with
      `ended_at IS NULL`.
- [ ] Updating L1 ends the old row and inserts a new one — the old row
      stays in the table (so a leave request that snapshotted it can
      still resolve to the old approver).
- [ ] Self-assignment returns 422 with a clear `errors.approver_id`.
- [ ] Assigning a soft-deleted user returns 422.
- [ ] Bulk reassign of an approver who covers 50 employees creates 50
      end-row updates and 50 insert-row updates inside one transaction.
- [ ] Service unit tests mock `BaseApprovalChainRepository`.

### Slice 1.3 — Wire the Edit-User drawer endpoint

**Choice point.** Two options:

(a) **Extend `PATCH /admin/users/:id`** to accept
`level_1_approver_id` and `level_2_approver_id`. The service forwards
those fields to the approval-chain service.

(b) **Keep `PATCH /admin/users/:id` lean** (no approver fields) and have
the Edit drawer make a second call to `PUT /admin/approvers/:id`.

**Recommendation: (b).** Two reasons:
- Keeps the User module unaware of approval-chain semantics
  (otherwise the User update DTO needs a cross-table FK validator).
- The bulk page already needs the dedicated endpoint, so we're not
  building two paths to the same end.

**Acceptance:**
- [ ] Edit drawer issues TWO requests on save: `PATCH /admin/users/:id`
      (profile fields) then `PUT /admin/approvers/:id` (chain). Both
      use TanStack Query mutations chained with `await`.
- [ ] If the profile save succeeds but the chain save fails, surface
      both: "Profile saved, approvers update failed — retry." Don't
      silently roll back the profile.

---

## 7. Phase 2 — Admin frontend for chain assignment

### Slice 2.1 — Edit-User drawer: add approver dropdowns

**File:** `src/features/admin-users/components/edit-user-drawer.tsx`.

**UI placement:** under the Role select, above the Active checkbox
(matches the screenshot). Two `Select` (shadcn — install per
`docs/universal-guidelines/frontend-component-blueprint.md` since we don't
have it yet) labeled "Level 1 approver" and "Level 2 approver".

**Options source:** `GET /admin/users?is_active=true&limit=100`. Filter
the dropdown to exclude the employee being edited (no self-approval).

**Empty state:** "— None —" option clears the assignment.

**Acceptance:**
- [ ] L1 and L2 default to the employee's currently active assignments
      (fetched on drawer open via the new `GET /admin/approvers/:id`).
- [ ] Dropdown list excludes the current employee.
- [ ] Save button is disabled until something changed; on save, both
      mutations run in sequence.
- [ ] Test: editing an employee with no chain, then setting L1=Alice
      and L2=Bob, then re-opening the drawer shows Alice and Bob.

**Stack compliance check** (per
`docs/universal-guidelines/frontend-component-blueprint.md`):
- Need shadcn `Select` — run `npx shadcn@latest add select`. Adds
  `@radix-ui/react-select` and `src/components/ui/select.tsx`. No
  third-party Combobox library.

### Slice 2.2 — `/admin/approvers` bulk page

**Files:**
```
src/app/(app)/admin/approvers/page.tsx
src/features/admin-approvers/
├── api.ts                          (list, set, bulk-reassign)
├── schemas.ts                      (Zod)
├── hooks/use-approvers.ts
└── components/
    ├── approvers-page.tsx
    ├── approvers-table.tsx
    ├── inline-approver-cell.tsx    (per-row inline edit with the same Select)
    └── bulk-reassign-dialog.tsx    (Dialog primitive from /components/ui)
```

**UX:** table with columns Employee · L1 Approver · L2 Approver ·
Updated. Each L1/L2 cell is an inline `Select`. Top-bar action:
"Bulk reassign…" opens a dialog asking "From: <user>, To: <user>",
confirms, fires `POST /admin/approvers/bulk-reassign`.

**Gating:** `RequirePermission code="APPROVAL_CHAIN:View"` at the
route. The bulk-reassign action and inline edits are additionally
hidden when the caller lacks `APPROVAL_CHAIN:Update`.

**Acceptance:**
- [ ] Sidebar gains "Approvers" entry under "Administration", gated on
      `APPROVAL_CHAIN:View`.
- [ ] Inline edit fires a single `PUT /admin/approvers/:id` per row
      on commit; optimistic update reverts on error.
- [ ] Bulk reassign with `from=Alice, to=Bob` updates every row that
      had Alice and shows a toast "Reassigned N rows."
- [ ] Empty state: "No employees yet — add one via Employees."
- [ ] Loading skeleton matches the Employees page pattern.

**Bulk page — is it necessary?** YES. Without it, reassigning when
Alice goes on parental leave is N drawer-open / drawer-save round
trips. With it, one dialog covers it. Same backend endpoint; the cost
is one page + one dialog component. Ship it.

---

## 8. Phase 3 — `leave-requests` backend

### Slice 3.1 — Domain + persistence + migration

Files per blueprint, plus migration `CreateLeaveRequestsTable` with
the schema in §3.2 and its four indexes.

**Acceptance:**
- [ ] Migration applies and reverts cleanly.
- [ ] Mapper fails fast if requester is not joined (mirrors
      `UserMapper` pattern).
- [ ] Inserting a request with `end_date < start_date` is rejected at
      the DB level by the CHECK constraint.

### Slice 3.2 — Service: submit / list / cancel

**Service contract:**

```ts
submit(input: { employee_id, leave_type, start_date, end_date, reason })
  → reads active chain via BaseApprovalChainRepository.getActive(employee_id)
  → if NO l1 assigned → throw 422 errors.approval_chain
                        "No approver assigned. Contact HR."
                        (Q1 hard-block — no silent rubber-stamps.)
  → Q4 overlap check: throw 422 errors.dates "Overlaps existing
    request #N" if any leave_request for the same employee with
    status IN ('pending_l1','pending_l2','approved') overlaps
    [start_date, end_date]. SQL:
      WHERE employee_id = $1
        AND status IN ('pending_l1','pending_l2','approved')
        AND deleted_at IS NULL
        AND NOT (end_date < $start OR start_date > $end)
  → snapshots l1_approver_id, l2_approver_id onto the row
  → initial status: 'pending_l1' (always — l1 is now guaranteed present)
  → insert

list(criteria) — filters: employee_id, status[], from, to, leave_type
                          (HR with ViewAll sees all; ViewOwn callers
                          have employee_id forced to caller.id)
cancel(id, actor) — only the requester or HR; only if status is pending_*
                    (caller.id === request.employee_id
                     || hasPermission(caller, 'LEAVE:Update'))

update(id, patch, actor)  -- Q3: pending-only edits
  → load request
  → if status NOT IN ('pending_l1','pending_l2') →
       throw 409 errors.status "Cannot edit a request in state X.
       Use cancel + resubmit."
  → require LEAVE:Update on caller
  → apply patch (does not change l1/l2 snapshot; for that use
    /admin/approvers/repair-pending)
```

**Cancel-after-L1-approval (clarification):** if cancel hits during
`pending_l2` (L1 already approved), the row transitions to `cancelled`
but `decided_at` and `decided_by` from the L1 approval are preserved as
the historical record. Add a separate `cancelled_at` / `cancelled_by`
pair so both events are visible in audit.

### Slice 3.3 — Service: approve / reject

```ts
approve(id, actor)
  → load request
  → assert canActOn(request, actor) — chain check OR ApproveAny
  → if approver assigned but is_active = false at this moment, throw
    409 errors.approver "Assigned approver is deactivated. HR must
    repair the chain (POST /admin/approvers/repair-pending)."
  → transition: pending_l1 → pending_l2 (if l2_approver_id != null)
                pending_l1 → approved   (if l2_approver_id IS null)
                pending_l2 → approved
  → set decided_at, decided_by
  → set decision_path = caller used ApproveAny/system_admin ? 'override' : 'chain'

reject(id, actor, note)
  → assert canActOn (same rules as approve)
  → transition to 'rejected' from either pending state
  → require note (422 if empty)
  → set decided_at, decided_by, decision_path
```

`canActOn` (strict — use `===` and guard nulls):

```ts
function canActOn(request, caller): boolean {
  if (caller.system_admin === true) return true;
  if (hasPermission(caller, 'LEAVE:ApproveAny')) return true;
  if (request.status === 'pending_l1'
      && request.l1_approver_id !== null
      && caller.id === request.l1_approver_id) return true;
  if (request.status === 'pending_l2'
      && request.l2_approver_id !== null
      && caller.id === request.l2_approver_id) return true;
  return false;
}
```

### Slice 3.4 — Controllers: admin + me split

Backend routes follow the existing `/users/me/<sub-resource>` convention
(matches `/users/me/time-entries`, `/users/me/permissions`). Approve /
reject are top-level — neither admin nor self-service, they're the
*approver* acting on someone else's request. (Q5 resolved 2026-05-30.)

```
/admin/leave-requests                       (LEAVE:ViewAll / Update / Delete)
  GET / GET :id / PATCH :id / DELETE :id

/users/me/leave-requests                    (LEAVE:Create / ViewOwn)
  GET / POST / GET :id / POST :id/cancel

/leave-requests/:id                         (LEAVE:ViewOwn route gate;
                                             service grants access to
                                             requester | l1 | l2 | ViewAll)
  GET

/leave-requests/:id/approve                 (LEAVE:Approve)
/leave-requests/:id/reject                  (LEAVE:Approve)
```

Frontend pages live under `/employee/<thing>` (existing convention,
matches `/employee/home`, `/employee/timesheet`, `/employee/schedule`):

```
/employee/leaves                            self-service: submit + my list
/admin/leave-requests                       HR view (sidebar: Administration)
/approvals                                  approver inbox (existing route)
```

**Acceptance:**
- [ ] Employee A submits → row has `status='pending_l1'`,
      `l1_approver_id=their_l1`, `l2_approver_id=their_l2`.
- [ ] Their L1 calls approve → status moves to `pending_l2`.
- [ ] Some OTHER user with `LEAVE:Approve` (not on the chain) calls
      approve → 403 with `errors.approver` "Not the assigned approver
      for this step".
- [ ] HR_ADMIN (has `LEAVE:ApproveAny`) calls approve from
      `pending_l1` → status jumps to `approved`.
- [ ] Employee submits then immediately calls cancel → status
      `cancelled`; their L1 can no longer approve (404 from approve).
- [ ] e2e: 5 requests from 3 employees, mixed approvers, all transition
      correctly.

### Slice 3.5 — `approvals` module integration

Replace the placeholder `findPending` in `approvals.service.ts` with a
real query that UNIONs across leave + (future) time-correction.
Initially just leave (Phase 5 extends to corrections).

```sql
-- Pseudocode
WHERE (status='pending_l1' AND l1_approver_id = :user_id)
   OR (status='pending_l2' AND l2_approver_id = :user_id)
   -- OR caller has APPROVAL:ApproveAny (return everything pending)
```

**Acceptance:**
- [ ] L1 approver lists `/approvals?type=leave` and sees only requests
      where they're the current step approver.
- [ ] HR_ADMIN (with `APPROVAL:ApproveAny`) sees all pending leave
      requests.
- [ ] An employee with no `LEAVE:Approve` calling `/approvals` gets
      403 (already enforced by the existing route gate).

---

## 9. Phase 4 — `leave-requests` frontend

### Slice 4.1 — `/me/leave` self-service page

**Stack notes:** form with shadcn `Input` + `Select` + `Calendar`
(shadcn ships these — install if missing). Date range via two
`<input type="date">` initially; upgrade to the shadcn `Calendar`
popover once we have a real design need.

**Acceptance:**
- [ ] Submit form: leave_type, start_date, end_date, reason.
- [ ] My-list section below: paginated, status badge per row.
- [ ] Cancel button on pending rows; disabled on terminal rows.
- [ ] Sidebar entry: "Leave" under the "Me" cluster.

### Slice 4.2 — `/admin/leave-requests` HR view

- Paginated table: Employee · Type · Dates · Status · Decision · Submitted.
- Filters: employee, status, date range.
- Click row → drawer with details and HR-only actions (force-approve, edit, cancel).
- Sidebar entry: "Leave requests" under "Administration".

### Slice 4.3 — `/approvals` real data + actions

Existing page currently shows nothing useful. Now:
- Wire to `GET /approvals?type=leave`.
- Each row gets "Approve" + "Reject" buttons (the reject button opens a
  dialog asking for a note).
- Optimistic update on click; refetch the list on success.
- Empty state: "No pending approvals."

**Acceptance:**
- [ ] An L1 approver sees pending L1 leave requests and acts on them.
- [ ] After approving, the row disappears from their inbox (it's now
      `pending_l2`, owned by L2).
- [ ] After rejecting, requester sees status `rejected` with the note.

---

## 10. Phase 5 — `time-correction-requests` backend

Mirrors Phase 3 step for step. Three differences:

1. **On approve (terminal),** the time-correction service calls
   `TimeEntriesService.applyCorrection(request)` — a new method
   added to the existing time-entries module. The time-entries
   module owns its own table writes; the time-correction module
   owns the request lifecycle. Wrapped in one transaction so the
   request status flip and the `time_entries` mutation commit
   together. (Q6 decision 2026-05-30.)

2. **`applyCorrection(request)` contract** (new method on
   `TimeEntriesService`):
   - If `request.target_entry_id` is non-NULL → load that row;
     verify it's not soft-deleted; update `time_in` / `time_out`
     with the proposed values; set `source = 'correction'`;
     `updated_by = request.decided_by`.
   - If `target_entry_id` is NULL → create a new row with
     `source = 'correction'`, `created_by = request.decided_by`.
   - At approve time, pre-check the partial unique index would
     not be violated; return 409 `errors.conflict` with a clear
     "an open entry already exists" message before the
     transaction commits. (C7 fix — better UX than letting the
     DB throw 23505 from inside the approve flow.)

3. **`TimeCorrectionRequestsModule` imports `TimeEntriesModule`**
   so the cross-module DI works without leaking the time-entries
   repo into the correction service.

**Permission gates use `TIME_CORRECTION:*`** — different code family
from `LEAVE:*`. A user can have one without the other.

**Update / overlap rules** mirror leave's Q3 / Q4 decisions:
- HR can only edit `pending_*` correction requests.
- Submission rejected if a pending or approved correction request
  already exists for the same `(employee_id, work_date)`.

Acceptance + verification: same shape as Phase 3.

---

## 11. Phase 6 — `time-correction-requests` frontend

### Slice 6.1 — "Request correction" affordance on `/employee/timesheet`

Each time-entry row in the existing timesheet gets a "Request
correction" button. Clicking opens a drawer pre-filled with the
current `time_in` / `time_out`, lets the user adjust + add a reason,
submits.

### Slice 6.2 — `/approvals` handles correction items

Same page as leave; the row renderer branches on `type`. Action buttons
identical (`Approve` / `Reject`).

### Slice 6.3 — `/admin/time-corrections` HR view

Same shape as `/admin/leave-requests`.

---

## 12. Checkpoints

Stop and get sign-off before crossing each line.

### ☑ Checkpoint A — after Phase 0 (permissions + roles)

Verify: `npm run seed` then `psql -c "SELECT r.name, p.code FROM
roles r JOIN role_permissions rp ON rp.role_id=r.id JOIN permissions
p ON p.id=rp.permission_id WHERE p.code LIKE 'LEAVE:%' OR p.code LIKE
'TIME_CORRECTION:%' OR p.code LIKE 'APPROVAL_CHAIN:%' ORDER BY r.name,
p.code"` returns exactly the grant matrix in §2.3.

Decision needed: have we over-granted anything? (Common drift: giving
EMPLOYEE `LEAVE:View` and forgetting to scope-to-own in the service.)

### ☑ Checkpoint B — after Phase 1 (approval-chains backend)

Verify with curl:
1. `PUT /admin/approvers/12` with `{l1:5, l2:7}` → 200.
2. `GET /admin/approvers/12` → returns the same.
3. `PUT` again with `{l1:8, l2:7}` → 200; row history shows two L1
   rows (one ended, one active).
4. Self-assign `{l1:12}` for employee 12 → 422.

Decision needed: are the chain's audit semantics (versioned rows,
never UPDATE in place) right before any request snapshots them?

### ☑ Checkpoint C — after Phase 2 (admin frontend)

Demo: HR opens Edit drawer for an employee, sets L1/L2, saves. Then
goes to /admin/approvers and runs a bulk reassign. All visible in the
table. Get sign-off on the UX before building the request flows.

### ☑ Checkpoint D — after Phase 3 + 4 (leave end-to-end)

End-to-end demo: employee submits leave → L1 sees it in /approvals →
L1 approves → L2 sees it → L2 approves → employee sees `approved`. HR
also force-approves a separate request via the override path.

Decision needed: is the state machine right? Any policy gaps surfaced
during the demo (e.g. "what if a request straddles two chain
reassignments" — answer should be "the snapshot wins").

### ☑ Checkpoint E — after Phase 5 + 6 (corrections end-to-end)

Same shape as D, plus: verify that approving a correction request
actually mutates the underlying `time_entries` row, and that the row
ends up with `source='correction'` so HR can audit.

### ☑ Checkpoint F — before any rollout

Run the full backend test suite (unit + e2e) and the frontend test
suite. Confirm no orphan grants (every permission code is held by at
least one role, every role grants at least one code that's actually
checked).

---

## 13. Out of scope (will need separate plans)

- **Leave types catalog** with per-type accrual rules, balances, carryover.
- **Leave balances** (annual leave days remaining, etc.).
- **Overtime requests** (separate workflow, same chain).
- **Expense / reimbursement requests** (same chain).
- **Approval delegation** ("acting approver while X is out").
- **Push / email notifications** to approvers — Slack-style inbox only for v1.
- **Approval comments thread** (today we have a single `decision_note`).
- **3+ step chains** — the schema supports `step` of any int, but the
  UI and service explicitly assume L1/L2. Promoting to N levels later
  is a UI + DTO change, not a schema change.
- **Org-chart visualizer** for the chains. Bulk table is enough for v1.
- **Audit log surface** ("show me everything Alice approved in May") —
  the data exists, the UI doesn't.

---

## 14. Risks and mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| Permission code typos let an unauthorized role through | Medium | Use the existing const-object pattern; CI typecheck catches drift; Phase 0 seed test asserts exact grant matrix |
| Snapshot model gets out of sync with current chain → "this approver doesn't exist anymore" | Medium | Validate approver `is_active = true` at *approve time*, not just submission; if false, force HR override or chain repair |
| Two simultaneous chain edits race | Low | Single endpoint wraps end-old + insert-new in a transaction; partial unique index is the safety net |
| EMPLOYEE accidentally gets `LEAVE:Approve` via misconfigured role | Low | Phase 0 seed test pins the matrix; PR review catches manual `roles.json` edits |
| `LEAVE:ApproveAny` becomes the easy "skip the chain" path everyone uses | Medium | Restrict to HR_ADMIN + SUPER_ADMIN in seed; document in the role grant docstring; review usage in code review |
| Self-approval slips past (employee is their own L1) | Low | Both the service `setChain` and the seed validator reject `approver_id == employee_id`; covered by Phase 1 acceptance test |
| Frontend talks to the wrong endpoint shape after backend ships | Medium | Co-evolve Zod schemas; Phase 4 / 6 acceptance includes contract tests |
| `time_correction` approval creates a duplicate open `time_entry` | Low | DB partial unique index already enforces this; service maps 23505 to 409 |
| HR forgets to assign chains for new hires → requests auto-approve | High | Phase 3.2 logs a WARNING when a request lands with no L1 chain; Phase 4.2 admin view surfaces "Employees with no chain" filter |

---

## 15. Done definition

This plan is "done" when:

1. All six phases shipped with their acceptance criteria satisfied.
2. Phase 0 grant matrix is locked by a seed test.
3. A new hire flow works end to end: HR creates the user → HR assigns
   L1+L2 → user submits a leave request → both approvers act on it →
   request hits `approved` → HR can find it in `/admin/leave-requests`.
4. `npm run lint && npm run test && npm run test:e2e && npm run build`
   green on both repos.
5. `docs/universal-guidelines/module-architecture.md`'s anti-patterns table has no fresh
   violations from this work.
6. ADR 0001 is referenced from the new modules' docstrings — the
   approval semantics are a recorded decision, not folklore.

---

## 16. Notes on file placement and task tracking

- This plan lives at `docs/plans/2026-05-30-leave-correction-and-approval-chains.md` per the established convention (memory rule: every plan in `docs/plans/` starts with YYYY-MM-DD).
- The companion task checklist lives only in the gitignored `tasks/todo.md` working file — never committed to `docs/plans/` (todo snapshots are not part of the audit trail).
- The planning skill's default `tasks/plan.md` / `tasks/todo.md` landed in a gitignored folder here; we use `docs/plans/` instead.
- If you want backend-only task tracking that lives in `asima-backend/tasks/`, mirror the Phase 1/3/5 sections into `asima-backend/tasks/plan.md` (already referenced by `asima-backend/CLAUDE.md`).
