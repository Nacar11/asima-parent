# Spec: Approvals page + role-aware sidebar

- **Status:** Draft (pending human review before Phase 2)
- **Date:** 2026-05-25
- **Scope:** asima-backend + asima-frontend
- **Related ADR:** `asima-backend/docs/adr/0001-roles-and-approval-design.md`
- **Related prior plan:** `docs/plans/2026-05-23-role-based-sidebar-and-routing.md`

## 1. Objective

Give every role a sidebar that shows only what its permissions allow,
expand the admin shell to a three-section layout (Me / Approvals /
Administration), and introduce an `Approvals` page that PM / TD / OM /
HR can land on.

The Approvals page ships as a **UI shell** in this phase: the route,
the sidebar entry, the backend endpoint, the permission codes, and the
HR-override branch are all real, but the underlying request data
(leave, corrections) does not exist yet. The endpoint returns an
empty paginated response. This unblocks frontend layout work and
locks the contract before the leave module lands.

### Why now

- Today's sidebar collapses Employees under a single Administration
  cluster that's effectively HR_ADMIN-only. PM / TD / OM users see
  zero admin items and have no destination for their role's primary
  workflow.
- ADR 0001 already decided how approvals route (per-employee chain,
  role gates the surface, chain gates the row). Building the
  `APPROVAL:View` permission and the empty endpoint now means the
  leave module's first slice can be "add data," not "add gates +
  data + sidebar."
- The HR override (`APPROVAL:ApproveAny`) is wired now so the
  contract is set: "filter by current_approver_id unless you carry
  the override." Adding the override later is a behavior change that
  would silently expand what HR sees.

### Out of scope (deferred to leave module)

- The `approval_chains` table itself.
- The `leave_requests` / `time_correction_requests` tables and their
  state machines.
- Per-resource permission codes (`LEAVE:Approve`, `CORRECTION:Approve`).
- Any approve/reject action endpoint. The page renders a read-only
  empty list in this phase; action buttons are not in the UI.
- Real-time / notification flow for pending approvals.

## 2. Tech Stack

Unchanged from current. No new dependencies on either side.

- **Backend:** NestJS 10, TypeORM 0.3, PostgreSQL 16, Node ≥ 20.
- **Frontend:** Next.js (App Router), React, TanStack Query, Tailwind,
  zod for response schemas.
- **Auth:** existing JWT + global `PermissionsGuard` pipeline.

## 3. Cross-cutting decisions

Lock these now — every downstream task depends on them.

### 3.1 Roles are NOT extended; permissions ARE

Per ADR 0001 the role taxonomy is stable (SUPER_ADMIN, HR_ADMIN,
OPERATIONS_MANAGER, PROJECT_MANAGER, TECHNICAL_DIRECTOR, EMPLOYEE).
This spec does not add or rename roles. It adds two permission codes
and re-maps them onto the existing roles via the existing
`role_permissions` table.

### 3.2 Single role per user — not multi-role

`users.role_id` stays single-valued. The user's prompt mentioned
"multi-roles," but the existing schema and ADR 0001 are single-role.
Going N:M would require a `user_roles` join table, a permissions
union resolver, and breaking changes to `/users/me/permissions`.
**That is a separate ADR and out of scope here.**

If we ever need multi-role, the pattern is "User has many Roles,
permissions is the SET UNION across roles." Until then,
`users.role_id` is the single source.

### 3.3 Two new permission codes (this is the entire permission delta)

| Code | Meaning | Why a code, not a role check |
|---|---|---|
| `APPROVAL:View` | Can see the Approvals page and call `GET /approvals/pending`. | The frontend gate matches the backend gate by code, not by `role.name`. Adding a new approver role later (e.g. `FINANCE_DIRECTOR`) is a permission grant, not a frontend edit. |
| `APPROVAL:ApproveAny` | HR override: the pending list returns **every** pending request, not just rows where `current_approver_id === me.id`. | The override is a single conditional in the service. Encoding it as a permission means the override can be granted to other roles (e.g. an interim COO) without code change. |

These follow the existing `RESOURCE:Action` shape. They are added to
`permissions.json` and seeded via the idempotent seeder — no migration
required.

**These do NOT replace per-resource action codes.** When the leave
module lands it will introduce `LEAVE:Approve`, `LEAVE:Reject`, etc.
Those will be required to act on a specific row. `APPROVAL:View` is
the page-gate; `LEAVE:Approve` will be the action-gate. Two distinct
layers, deliberately.

### 3.4 Role → permission map (after this change)

Only the additions matter; existing grants are unchanged.

| Role | Adds | Notes |
|---|---|---|
| SUPER_ADMIN | none (already bypasses) | `system_admin: true` short-circuits the guard. |
| HR_ADMIN | `APPROVAL:View`, `APPROVAL:ApproveAny` | HR sees every pending approval. |
| OPERATIONS_MANAGER | `APPROVAL:View` | Sees only rows where they're the current approver. |
| PROJECT_MANAGER | `APPROVAL:View` | Same as OM. |
| TECHNICAL_DIRECTOR | `APPROVAL:View` | Same as OM. |
| EMPLOYEE | none | Self-service only. |

PM and TD still have empty `LEAVE:*` permissions because LEAVE:*
doesn't exist yet. They now have exactly one permission
(`APPROVAL:View`) so the sidebar Approvals item lights up. The
endpoint will return `[]` for them until approval_chains lands.

### 3.5 Sidebar gets a third section

`SECTION_ORDER` changes from `['me', 'admin']` to
`['me', 'approvals', 'admin']`. The new section header is
`Approvals`. Items rendered in the section:

- **Pending approvals** — `/approvals`, requires `APPROVAL:View`.

Existing logic in `app-shell.tsx` already filters sections that have
zero visible items, so EMPLOYEE / SUPER_ADMIN-without-mapped-roles
won't see an empty header.

### 3.6 The endpoint is real, the data is empty

`GET /api/v1/approvals/pending` ships in this phase. Its contract is
the standard `PaginatedResponse<PendingApproval>`. In v0 the service
returns `{ data: [], total: 0, page: 1, limit: 20, has_more: false }`
regardless of caller — but the **gating logic is real**:

- `@Permissions({ APPROVAL: 'View' })` on the controller.
- Service inspects `req.user.permissions` for `APPROVAL:ApproveAny`
  and `req.user.system_admin`. The branch is wired even though both
  paths currently return empty.

This means when the leave module fills in `approval_chains` and
`leave_requests`, only the service body changes — controller,
permissions, DTO, frontend, and sidebar all stay put.

## 4. Backend design

### 4.1 New module: `src/approvals/`

Follows the standard hexagonal layout. No persistence layer in v0
(no table), so the `persistence/` folder is **omitted**, not stubbed
with empty files. When the leave module lands, persistence is added
alongside the read query that joins `approval_chains` + `leave_requests`
+ (later) `time_correction_requests`.

```
src/approvals/
├── domain/
│   ├── pending-approval.ts             # union shape of an approvable row
│   └── find-pending-approvals.ts       # PaginatedResponse<PendingApproval>
├── dto/
│   └── query-pending-approvals.dto.ts  # page, limit only in v0
├── controllers/
│   └── approvals.controller.ts         # GET /approvals/pending
├── approvals.service.ts                # v0 returns empty paginated payload
└── approvals.module.ts
```

### 4.2 Domain shape

`PendingApproval` is a forward-looking union. v0 ships the type so
the frontend can write its zod schema today; the backend just never
populates a row yet.

```ts
// src/approvals/domain/pending-approval.ts
export type PendingApprovalKind = 'leave' | 'time_correction';

export class PendingApproval {
  id!: number;                       // request_id (leave_request.id or time_correction.id)
  kind!: PendingApprovalKind;
  employee_id!: number;
  employee_name!: string;            // denormalized for table display
  requested_at!: Date;
  current_step!: number;
  current_approver_id!: number;
  summary!: string;                  // human-readable one-liner
}
```

`kind` is the discriminator the frontend uses to decide which detail
route to deep-link into (`/approvals/leave/:id` vs
`/approvals/corrections/:id`). Neither detail route ships in this
phase, but reserving the field now avoids a breaking change later.

### 4.3 Controller

```ts
// src/approvals/controllers/approvals.controller.ts
@ApiTags('Approvals')
@ApiBearerAuth()
@Controller({ path: 'approvals', version: API_VERSION })
export class ApprovalsController {
  constructor(private readonly service: ApprovalsService) {}

  @Get('pending')
  @Permissions({ APPROVAL: 'View' })
  @ApiOperation({ summary: 'List approvals where the caller is the current approver (or all, if APPROVAL:ApproveAny).' })
  @ApiResponse({ status: 200, type: FindPendingApprovals })
  findPending(
    @CurrentUser() user: AuthenticatedUser,
    @Query() query: QueryPendingApprovalsDto,
  ): Promise<FindPendingApprovals> {
    return this.service.findPending(user, query);
  }
}
```

Path is `/api/v1/approvals/pending` (no `admin/` prefix — Approvals is
not an admin surface even though HR can see all rows). The fact that
HR's view is wider is a permission concern, not a URL concern.

### 4.4 Service (v0 body)

```ts
async findPending(
  user: AuthenticatedUser,
  query: QueryPendingApprovalsDto,
): Promise<FindPendingApprovals> {
  const canSeeAll =
    user.system_admin || user.permissions.includes('APPROVAL:ApproveAny');

  // Branch is wired now so the leave module only adds the data fetch,
  // not the permission split. canSeeAll = full org query;
  // !canSeeAll = filter by current_approver_id = user.id.
  void canSeeAll;

  const { page = 1, limit = 20 } = query;
  return { data: [], total: 0, page, limit, has_more: false };
}
```

### 4.5 Permission seed delta

Add two entries to `src/database/seeds/data/permissions.json`:

```json
{ "code": "APPROVAL:View",       "resource": "APPROVAL", "action": "View",
  "description": "View pending approvals where the caller is the current approver" },
{ "code": "APPROVAL:ApproveAny", "resource": "APPROVAL", "action": "ApproveAny",
  "description": "Override: see every pending approval regardless of approval-chain placement" }
```

Update `src/database/seeds/data/roles.json` per §3.4. Re-running
`npm run seed` is idempotent (upsert on `code` / `name`).

No migration. The permission and role tables already exist; the seed
just inserts new rows and links them through `role_permissions`.

### 4.6 No new ADR

This spec is incremental on top of ADR 0001. The decisions here
(two new codes, three-section sidebar, empty-payload UI shell) are
implementation details of ADR 0001, not a deviation from it. If the
leave module later wants to introduce a different gating strategy
(e.g. event-sourced approval state), that warrants its own ADR.

## 5. Frontend design

### 5.1 New route

```
src/app/(app)/approvals/
└── page.tsx
```

Server component shape:

```tsx
import { RequirePermission } from '@/components/require-permission';
import { ApprovalsPage } from '@/features/approvals/components/approvals-page';

export default function Page() {
  return (
    <RequirePermission code="APPROVAL:View">
      <ApprovalsPage />
    </RequirePermission>
  );
}
```

### 5.2 New feature folder

Mirrors `src/features/admin-users/`:

```
src/features/approvals/
├── api.ts                       # fetchPendingApprovals(page, limit)
├── schemas.ts                   # zod for PendingApproval + paginated wrapper
├── hooks/
│   └── use-pending-approvals.ts # TanStack Query wrapper
└── components/
    ├── approvals-page.tsx       # full-page layout (search bar shell, table)
    ├── approvals-table.tsx      # columns: Kind, Employee, Requested, Step
    └── approvals-empty-state.tsx
```

The empty state copy intentionally describes the steady-state UX
("No requests need your approval"), not the v0 reality ("Backend not
implemented yet"). Reviewer-visible TODO: copy needs a follow-up when
the leave module ships actual data.

### 5.3 Sidebar update (`app-shell.tsx`)

Three diffs in this one file:

1. `SidebarSection` union gains `'approvals'`.
2. `SECTION_ORDER` becomes `['me', 'approvals', 'admin']`.
3. `SECTION_HEADERS.approvals = 'Approvals'` added.
4. New item appended to `SIDEBAR_ITEMS`:
   ```ts
   {
     href: '/approvals',
     label: 'Pending approvals',
     icon: Inbox, // from lucide-react
     section: 'approvals',
     requires: 'APPROVAL:View',
   }
   ```

The filter-by-permission and "hide-empty-section" logic that already
runs on the existing 'admin' section handles the new section
unchanged.

### 5.4 Permission-codes mirror

Append to `src/features/auth/permission-codes.ts`:

```ts
'APPROVAL:View',
'APPROVAL:ApproveAny',
```

Per the file's existing convention, this mirror is kept in sync by
hand. The `as const` on the array means a stale code in any consumer
(sidebar, route gate) fails typecheck.

### 5.5 What the page renders in v0

A title bar, an empty-state card (`<EmptyState>` primitive already
exists in `src/components/empty-state.tsx`), and the table component
hidden behind a `total > 0` check. The table component is built — it
just renders nothing while the API returns no rows. This keeps the
v0 → v1 diff small.

## 6. Project structure

Final layout after this spec is implemented:

```
asima-backend/src/
├── approvals/                                 # NEW
│   ├── controllers/approvals.controller.ts
│   ├── dto/query-pending-approvals.dto.ts
│   ├── domain/{pending-approval,find-pending-approvals}.ts
│   ├── approvals.service.ts
│   └── approvals.module.ts
├── database/seeds/data/
│   ├── permissions.json                       # +2 entries
│   └── roles.json                             # APPROVAL:* added to 4 roles
└── app.module.ts                              # registers ApprovalsModule

asima-frontend/src/
├── app/(app)/approvals/page.tsx               # NEW
├── components/layout/app-shell.tsx            # sidebar gains 'approvals' section
├── features/
│   ├── approvals/                             # NEW
│   │   ├── api.ts
│   │   ├── schemas.ts
│   │   ├── hooks/use-pending-approvals.ts
│   │   └── components/{approvals-page,approvals-table,approvals-empty-state}.tsx
│   └── auth/permission-codes.ts               # +2 entries
```

## 7. Commands

Unchanged. The relevant runs for this spec:

- Backend regen + seed: `cd asima-backend && npm run seed`
- Backend run: `cd asima-backend && npm run start:dev`
- Backend tests: `cd asima-backend && npm run test && npm run test:e2e`
- Frontend run: `cd asima-frontend && npm run dev`
- Frontend lint: `cd asima-frontend && npm run lint`

No migrations are generated by this spec. If a migration appears in
the diff during implementation, **stop** — something is wrong with
the seed-only assumption.

## 8. Code style

Mirror existing modules. Two specifics that came up in design:

**Backend domain class field shape** (per asima-backend/CLAUDE.md
section on definite-assignment):

```ts
export class PendingApproval {
  id!: number;                     // non-nullable → !
  kind!: PendingApprovalKind;      // non-nullable → !
  current_approver_id!: number;    // non-nullable → !
}
```

**Frontend zod schema** (matches the wire snake_case, no camelCase
translation):

```ts
export const PendingApprovalSchema = z.object({
  id: z.number().int().positive(),
  kind: z.enum(['leave', 'time_correction']),
  employee_id: z.number().int().positive(),
  employee_name: z.string(),
  requested_at: z.string().datetime(),
  current_step: z.number().int().nonnegative(),
  current_approver_id: z.number().int().positive(),
  summary: z.string(),
});
```

## 9. Testing strategy

### 9.1 Backend

- **Unit (service):** mock the (eventual) repository port; assert that
  `canSeeAll` is true for `system_admin` and for callers with
  `APPROVAL:ApproveAny`, false otherwise. v0 returns empty either way;
  test that the empty payload has the right `{ page, limit, total: 0,
  has_more: false }` shape.
- **Unit (controller):** assert `@Permissions({ APPROVAL: 'View' })`
  metadata is set. Spec file should grep for the decorator presence
  rather than mocking the whole guard chain.
- **E2E:** `test/approvals.e2e-spec.ts`. Cases:
  1. Unauthenticated → 401.
  2. EMPLOYEE token → 403 (no `APPROVAL:View`).
  3. PROJECT_MANAGER token → 200, empty list.
  4. HR_ADMIN token → 200, empty list (and would-be `canSeeAll: true`
     branch is hit — verify via a coverage assertion or test-only
     instrumentation, NOT a leaky response field).
  5. SUPER_ADMIN token → 200, empty list, bypass branch.

### 9.2 Frontend

- **Sidebar visibility (unit):** render `AppShell` with a mocked
  `usePermissions` returning `['APPROVAL:View']` → assert the
  "Pending approvals" link is in the DOM. With `[]` → assert it is not.
- **Route gate (unit):** render `<RequirePermission code="APPROVAL:View">`
  with no permissions → `<NotAuthorized />`; with the permission →
  children.
- **Page integration:** stub `GET /approvals/pending` to return the
  empty paginated payload → assert empty-state card renders, table
  does not.

### 9.3 What we are NOT testing in this phase

- Real chain filtering (`current_approver_id` match). The data does
  not exist; testing the branch with mocks risks lying about behavior.
  Add this test in the leave-module slice that introduces the data.
- Approve / reject actions. No endpoint exists yet.

## 10. Boundaries

**Always:**

- Run `npm run seed` after pulling this change (permissions are seed-driven).
- Add new permission codes to BOTH `permissions.json` and the
  frontend `permission-codes.ts` in the same PR. CI typecheck will
  catch a stale frontend mirror only if a consumer references the
  new code.
- Gate the sidebar item with `requires: 'APPROVAL:View'` — never with
  `role.name === 'HR_ADMIN'` or any string compare against role.
- Keep the v0 service body returning `{ data: [], total: 0, ... }`.
  If you find yourself adding a stub row to "make the table render,"
  use the frontend test instead.

**Ask first:**

- Adding a third `APPROVAL:*` code beyond `View` and `ApproveAny`
  (e.g. `APPROVAL:Delegate`). The spec deliberately keeps the
  permission surface minimal.
- Changing `users.role_id` to `users` ↔ `roles` M2M. That is a
  separate ADR, not a follow-up to this spec.
- Splitting Approvals into multiple routes (`/approvals/leave`,
  `/approvals/corrections`). The spec commits to one shared route
  with a `kind` discriminator on each row.
- Adding `LEAVE:*` codes here. They belong to the leave module.

**Never:**

- Switch on `role.name` in the backend service to decide what to
  return. The override path is `APPROVAL:ApproveAny` (a code), not
  a role string compare.
- Add a `?as=hr` or similar override query param to the endpoint.
  Override authority comes from the token, not the URL.
- Drop the `<RequirePermission>` wrapper on the page "because the
  sidebar already hides it." The wrapper is the route-level gate
  against direct navigation; the sidebar is just discovery.

## 11. Success criteria

This spec is "done" when all of the following are true:

1. PROJECT_MANAGER, TECHNICAL_DIRECTOR, OPERATIONS_MANAGER, and
   HR_ADMIN users see a "Pending approvals" link in the sidebar
   under an "Approvals" section header.
2. EMPLOYEE users do NOT see the link or the section header.
3. `GET /api/v1/approvals/pending` returns 200 with an empty
   paginated payload for the four roles above; returns 403 for
   EMPLOYEE; returns 401 unauthenticated.
4. HR_ADMIN's response goes through the `canSeeAll = true` branch
   (verifiable by service-level test).
5. Navigating directly to `/approvals` as an EMPLOYEE shows the
   `<NotAuthorized />` component (not a redirect, not a 404).
6. `npm run seed` re-applied against a fresh DB grants the right
   permissions to the right roles per §3.4 — verifiable by querying
   `role_permissions` after seed.
7. Swagger at `/docs` (non-prod) shows the new endpoint under an
   `Approvals` tag.
8. No migration files have been added in this PR.

## 12. Open questions

Resolved before this draft (carried here for the human reviewer's
audit trail):

- **What gets approved in this phase?** UI shell only. (Confirmed.)
- **HR sees overrides?** Yes, via `APPROVAL:ApproveAny`. (Confirmed.)
- **Sidebar layout?** Three sections — Me / Approvals /
  Administration. (Confirmed.)
- **PM vs TD vs OM differentiation?** Same page, server-side filter
  by `current_approver_id` (or full org if `APPROVAL:ApproveAny`).
  (Confirmed.)

Open for human input:

- **Empty-state copy.** Spec proposes "No requests need your
  approval." HR-with-override might warrant a slightly different
  string ("No pending requests across the org"). Decide before
  Phase 4.
- **Icon choice.** Spec uses lucide's `Inbox`. `ClipboardCheck` and
  `ListChecks` are alternates that arguably read more like
  "approvals." Pick one before sidebar diff lands.
- **Swagger tag.** Spec uses `'Approvals'`. Per
  asima-backend/CLAUDE.md convention, admin-gated controllers use
  `'Admin - X'`. Approvals is gated but not under `/admin`, so a
  bare `'Approvals'` matches the URL — confirm.

---

## Phase 2: Implementation plan (preview — full plan after spec approval)

Two-thread plan, ordered:

1. **Backend slice** (must land first — frontend imports rely on it):
   - Update `permissions.json` + `roles.json`
   - Scaffold `src/approvals/` module
   - Register in `app.module.ts`
   - Unit + E2E tests
   - Swagger annotations + group entry in `main.ts`

2. **Frontend slice** (after backend ships):
   - Add `APPROVAL:View`, `APPROVAL:ApproveAny` to `permission-codes.ts`
   - Build `src/features/approvals/`
   - Add `/approvals/page.tsx` with `<RequirePermission>`
   - Update `app-shell.tsx` for three-section sidebar
   - Component tests for sidebar visibility + route gate

Verification checkpoint between phases: hit
`GET /api/v1/approvals/pending` with each role's JWT and confirm
status codes + payload shape match §11.1–4 before starting the
frontend slice.
