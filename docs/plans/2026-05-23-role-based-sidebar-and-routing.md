# Plan: Role-Based Sidebar and Routing

**Date:** 2026-05-23
**Scope:** Frontend only. No backend changes — all required server-side infrastructure already exists.
**Status:** Planned, not yet implemented.

---

## 1. Objective

Build a permission-driven sidebar and route-gating framework on the asima
frontend so that every signed-in user sees only the navigation items and
pages they are entitled to use. Ship the framework together with one
concrete consumer — **Manage Employees** (`/admin/users`) — to validate the
end-to-end shape.

### What success looks like

- A user with role `EMPLOYEE` sees Home, Time sheet, Schedule, Profile.
  They see no admin section at all. Typing `/admin/users` shows a 403
  "Not authorized" page with a link back to `/employee/home`.
- A user with role `HR_ADMIN` sees the four employee items **plus** an
  "Administration" section containing "Employees". Visiting `/admin/users`
  loads the management table.
- A user with `system_admin = true` (SUPER_ADMIN) sees every section,
  bypassing per-permission checks just as `PermissionsGuard` does on the
  backend.
- Adding a new admin page is a 3-line config edit (one nav entry, one
  permission code) plus the new page itself — no framework changes.

---

## 2. Confirmed design decisions

| # | Decision | Rationale |
|---|---|---|
| 1 | **Single flat sidebar with section headers.** "My stuff" (Home/Timesheet/Schedule/Profile) and "Administration" stacked vertically, divider between, headers only render when the section has ≥1 visible item. | Avoids two-sidebar complexity. Keeps current `app-shell.tsx` shape. |
| 2 | **403 "Not authorized" page** when a user types a URL they lack permission for. Page shows a brief message + link to `/employee/home`. | Honest about *why* the page isn't loading. Less hostile than a 404. |
| 3 | **Manage Employees (`/admin/users`) is the first concrete admin surface.** Gated by `USER:View` for the list and `USER:Create / Update / Delete` for the actions. | Validates the framework end-to-end with the most useful first admin page. |
| 4 | **Permission-driven, not role-driven.** UI reads `GET /users/me/permissions` (flat string array). Role names never appear in frontend gating logic. | Matches the parent `CLAUDE.md` rule: "Frontend should drive UI gating from `/permissions` — never parse `role.permissions` client-side." |
| 5 | **Additive sidebar (not role-exclusive).** An HR admin still has their own profile, timesheet, and schedule — admin items are added on top, not swapped in. | Every user is a person with their own data, regardless of role. Also defers the multi-role question — see §9. |
| 6 | **Admin URLs at `/admin/*`** (e.g. `/admin/users`), matching the backend's `/api/v1/admin/<resource>` convention. | Audience boundary lives in the URL prefix. Mirrors backend's documented split. |
| 7 | **Backend is the security boundary; frontend gate is UX only.** Even with sidebar hidden, an attacker who guesses `/admin/users` hits a backend 403. `<RequirePermission>` is for friendly UX, not auth. | Defense in depth. Frontend cannot be trusted with security. |

---

## 3. Validation of existing infrastructure

| Capability | Status | Location |
|---|---|---|
| `GET /users/me/permissions` endpoint | ✅ implemented | `asima-backend/src/users/controllers/me-users.controller.ts:41` |
| Frontend wrapper for permissions endpoint | ✅ exists | `asima-frontend/src/features/profile/api.ts:20` (will relocate) |
| `MyPermissionsSchema` zod type | ✅ exists | `asima-frontend/src/features/auth/schemas.ts:46` |
| `system_admin` field on AuthUser | ✅ exists | `asima-frontend/src/features/auth/schemas.ts:17` |
| Admin Users CRUD endpoints | ✅ all implemented | `asima-backend/src/users/controllers/admin-users.controller.ts` |
| Permission codes (`USER:View` etc.) | ✅ seeded | `asima-backend/src/permissions/permissions.constants.ts` |
| `/admin/*` route group on frontend | ❌ does not exist | (Phase 2) |
| `<RequirePermission>` wrapper | ❌ does not exist | (Phase 2) |
| `403` page component | ❌ does not exist | (Phase 2) |

**Net:** the backend is 100% ready. All work is frontend.

---

## 4. Architecture

### 4.1 Data flow

```
On authenticate (login or bootstrap):
  AuthProvider sets status = 'authenticated'
       │
       ▼
  usePermissions() React Query kicks in
       │
       ├─ GET /users/me/permissions
       │       │
       │       ▼
       │  { permissions: ['USER:View', 'USER:Create', ...] }
       │       │
       │       ▼
  Cached under ['auth', 'me', 'permissions']
       │
       ▼
  Sidebar filter + <RequirePermission> read from cache (synchronous after first fetch)
```

On logout: `queryClient.removeQueries(['auth', 'me', 'permissions'])` so a
fresh login refetches.

### 4.2 Sidebar config shape

`app-shell.tsx`'s `NAV_LINKS` becomes a typed array:

```ts
type SidebarItem = {
  href: string;
  label: string;
  icon: LucideIcon;
  section: 'me' | 'admin';
  requires?: PermissionCode | PermissionCode[]; // AND-semantics when array
};

const SIDEBAR_ITEMS: SidebarItem[] = [
  { href: '/employee/home',      label: 'Home',       icon: Home,         section: 'me' },
  { href: '/employee/timesheet', label: 'Time sheet', icon: Clock,        section: 'me' },
  { href: '/employee/schedule',  label: 'Schedule',   icon: CalendarDays, section: 'me' },
  { href: '/admin/users',        label: 'Employees',  icon: Users,        section: 'admin', requires: 'USER:View' },
];
```

Items with no `requires` show to every authenticated user. Items with
`requires` show only when **all** required codes are present in the user's
permissions array (or `system_admin === true`).

Section headers render only when their section has ≥1 visible item.
Section order is fixed: `me` then `admin`.

### 4.3 Permission code typing

A new module `features/auth/permission-codes.ts` mirrors the backend's
`PERMISSION_RESOURCES` × `PERMISSION_ACTIONS` registry as a TypeScript
union:

```ts
export const PERMISSION_CODES = [
  'USER:View', 'USER:Create', 'USER:Update', 'USER:Delete',
  'ROLE:View', 'ROLE:Create', 'ROLE:Update', 'ROLE:Delete',
  'PERMISSION:View', 'PERMISSION:Create', 'PERMISSION:Update', 'PERMISSION:Delete',
  'TIME:View', 'TIME:Create', 'TIME:Update', 'TIME:Delete',
  'SCHEDULE:View', 'SCHEDULE:Create', 'SCHEDULE:Update', 'SCHEDULE:Delete',
] as const;
export type PermissionCode = typeof PERMISSION_CODES[number];
```

This is intentional duplication of a backend constant — kept in sync by
convention, not by code generation. Tradeoff documented in §10.

### 4.4 `<RequirePermission>` route wrapper

```tsx
<RequirePermission code="USER:View">
  <AdminUsersPage />
</RequirePermission>
```

Renders the `NotAuthorized` component if the current user lacks the code.
Used in admin page files (e.g. `(app)/admin/users/page.tsx` wraps its body
in this).

### 4.5 File layout (new)

```
src/
├── app/(app)/admin/users/page.tsx       NEW — wraps in <RequirePermission code="USER:View">
├── components/
│   ├── layout/app-shell.tsx             EDIT — typed SIDEBAR_ITEMS + section rendering
│   ├── not-authorized.tsx               NEW — the 403 component
│   └── require-permission.tsx           NEW — route gate wrapper
└── features/
    ├── auth/
    │   ├── api.ts                       EDIT — add permissionsApi.permissions (relocated from profile)
    │   ├── permission-codes.ts          NEW — typed PermissionCode union
    │   ├── use-permissions.ts           NEW — React Query hook
    │   └── permission-utils.ts          NEW — hasPermission(perms, codes, isSystemAdmin)
    └── admin-users/
        ├── api.ts                       NEW — adminUsersApi (list, get, create, update, delete, resetPassword)
        ├── schemas.ts                   NEW — zod schemas for admin user CRUD
        └── components/
            ├── admin-users-table.tsx    NEW
            ├── create-user-dialog.tsx   NEW
            ├── edit-user-dialog.tsx     NEW
            ├── reset-password-dialog.tsx NEW
            └── delete-user-confirm.tsx  NEW
```

---

## 5. Dependency graph

```
                       ┌──────────────────────────┐
                       │ permission-codes.ts      │
                       │ (typed constants)        │
                       └────────────┬─────────────┘
                                    │
                  ┌─────────────────┼─────────────────┐
                  │                 │                 │
                  ▼                 ▼                 ▼
   ┌────────────────────┐  ┌──────────────────┐  ┌──────────────────┐
   │ permission-utils   │  │ use-permissions  │  │ Sidebar config   │
   │ hasPermission()    │  │ React Query hook │  │ (SIDEBAR_ITEMS)  │
   └─────────┬──────────┘  └────────┬─────────┘  └────────┬─────────┘
             │                      │                     │
             └──────────────┬───────┴─────────────────────┘
                            ▼
              ┌─────────────────────────────┐
              │ <RequirePermission> wrapper │
              │ app-shell sidebar filter    │
              └──────────────┬──────────────┘
                             │
                             ▼
              ┌─────────────────────────────┐
              │ /admin/users page           │
              │ (first concrete consumer)   │
              └─────────────────────────────┘
```

Critical sequencing: **permission-codes → utils + hook → wrapper + sidebar filter → admin page**.

---

## 6. Phased task breakdown

Each task has acceptance criteria and a verification step. Tasks within a
phase can be parallelized only where noted; otherwise complete in order.

### Phase 0 — Foundations (no UI yet)

- [ ] **Task 0.1 — Define typed permission codes**
  - Files: `features/auth/permission-codes.ts` (new)
  - Acceptance: exports `PERMISSION_CODES` array and `PermissionCode` union
    covering every code in backend's `PERMISSION_RESOURCES × PERMISSION_ACTIONS`.
  - Verify: `npm run typecheck` passes.

- [ ] **Task 0.2 — Relocate permissions fetch to auth feature**
  - Files: `features/auth/api.ts` (edit), `features/profile/api.ts` (remove `permissions` method),
    callers of `profileApi.permissions` (zero today, but grep to confirm).
  - Acceptance: `authApi.permissions()` returns `Promise<MyPermissions>`. `profileApi.permissions` deleted.
  - Verify: `grep -rn profileApi.permissions src` returns nothing; `npm run typecheck` passes.

- [ ] **Task 0.3 — `usePermissions()` React Query hook**
  - Files: `features/auth/use-permissions.ts` (new)
  - Acceptance: hook returns `{ permissions: string[], isLoading, isError }`. Query key
    `['auth', 'me', 'permissions']`. `enabled: status === 'authenticated'`. Stale time 5 min.
  - Verify: hook only fires when user is authenticated; manual: log in, check Network tab fires once.

- [ ] **Task 0.4 — `hasPermission()` helper**
  - Files: `features/auth/permission-utils.ts` (new), `features/auth/permission-utils.spec.ts` (new)
  - Acceptance: `hasPermission(userPerms, required, isSystemAdmin)` returns true iff
    `isSystemAdmin === true` OR every code in `required` (string or string[]) is in `userPerms`.
  - Verify: `npm test -- permission-utils` passes covering: empty required, single required hit/miss,
    array required (all-or-nothing), system_admin bypass.

- [ ] **Task 0.5 — Clear permissions cache on logout**
  - Files: `features/auth/auth-provider.tsx` (edit `logout()`)
  - Acceptance: `logout()` calls `queryClient.removeQueries({ queryKey: ['auth', 'me', 'permissions'] })`.
  - Verify: log in as user A, log out, log in as user B — second session's first
    `/users/me/permissions` request fires (not served from stale cache).

**Phase 0 checkpoint:** all helpers in place, no UI changes yet. Sidebar still
renders the existing 3 employee items unchanged.

### Phase 1 — Sidebar gating

- [ ] **Task 1.1 — Refactor `NAV_LINKS` → `SIDEBAR_ITEMS`**
  - Files: `components/layout/app-shell.tsx` (edit)
  - Acceptance: new `SidebarItem` type and `SIDEBAR_ITEMS` array as specified in §4.2.
    Existing 3 employee items carry `section: 'me'` and no `requires`.
  - Verify: sidebar renders identically to today; `npm run typecheck` passes.

- [ ] **Task 1.2 — Filter sidebar by permissions + render section headers**
  - Files: `components/layout/app-shell.tsx` (edit)
  - Acceptance:
    - Calls `usePermissions()`.
    - Items hidden when `requires` not satisfied (using `hasPermission` + `user.system_admin`).
    - Section header renders **only** when ≥1 item in that section is visible.
    - Section order: `me` → divider+header "Administration" → admin items.
    - "My stuff" section header: not rendered (it's the default; only "Administration" gets a label).
  - Verify: temporarily add a fake admin item with `requires: 'USER:View'`. Log in as
    `EMPLOYEE` (no such perm) → no admin section visible. Log in as `HR_ADMIN` →
    Administration header + item visible.

**Phase 1 checkpoint:** sidebar correctly hides admin section for employees and
shows it for users with the required permissions. No real admin page yet — the
admin nav entry's `href` can `404` for now, fixed in Phase 3.

### Phase 2 — Route gating

- [ ] **Task 2.1 — `NotAuthorized` 403 component**
  - Files: `components/not-authorized.tsx` (new)
  - Acceptance: client component, renders heading "Not authorized", body
    "You don't have permission to view this page.", and a `<Link href="/employee/home">Go to Home</Link>`
    styled as primary button. Uses existing theme tokens.
  - Verify: temporarily route any admin URL to the component; visually inspect.

- [ ] **Task 2.2 — `<RequirePermission>` wrapper**
  - Files: `components/require-permission.tsx` (new)
  - Acceptance: takes `code: PermissionCode | PermissionCode[]` and `children`. While
    `usePermissions()` is loading, renders a small spinner/placeholder. On success,
    renders children iff `hasPermission(...)` true, else `<NotAuthorized />`.
  - Verify: wrap a known-allowed dummy page → renders. Wrap a known-denied dummy
    page → renders 403. Verify loading flicker is brief (< 200ms once cache warm).

**Phase 2 checkpoint:** the framework is complete. No real admin pages yet.

### Phase 3 — First admin surface: Manage Employees

- [ ] **Task 3.1 — Admin users API client + schemas**
  - Files: `features/admin-users/api.ts` (new), `features/admin-users/schemas.ts` (new)
  - Acceptance: `adminUsersApi.{list, get, create, update, delete, resetPassword}` matching
    backend's `admin-users.controller.ts`. All inputs/outputs zod-validated. List query
    accepts `{ page, limit }` and returns `PaginatedResponse<AdminUser>` matching the
    `{ data, total, page, limit, has_more }` shape from the parent `CLAUDE.md`.
  - Verify: `npm run typecheck` passes. Manual test of one endpoint via React Query
    devtools once page exists.

- [ ] **Task 3.2 — Admin users list page (read path)**
  - Files: `app/(app)/admin/users/page.tsx` (new),
    `features/admin-users/components/admin-users-table.tsx` (new)
  - Acceptance: page wraps in `<RequirePermission code="USER:View">`. Table shows
    columns: name, email, title, role, active, created_at. Paginator like
    `(app)/employee/timesheet/page.tsx`. Loading and error states match existing patterns.
  - Verify: log in as `HR_ADMIN`, navigate to `/admin/users`, table loads. As `EMPLOYEE`,
    navigate to `/admin/users` → 403. As `SUPER_ADMIN`, table loads.

- [ ] **Task 3.3 — Create user dialog**
  - Files: `features/admin-users/components/create-user-dialog.tsx` (new)
  - Acceptance: button "Add employee" on the table page (gated by `USER:Create`).
    Opens dialog with form: first_name, last_name, email, role_id, title (optional),
    initial password. Submits via `adminUsersApi.create`. Invalidates list query on success.
  - Verify: create a user as `HR_ADMIN`, confirm row appears in table. As an `EMPLOYEE`
    spoofing somehow, button is not visible (gated by permission predicate).

- [ ] **Task 3.4 — Edit user dialog**
  - Files: `features/admin-users/components/edit-user-dialog.tsx` (new)
  - Acceptance: row "Edit" action (gated by `USER:Update`). Form pre-fills, submits via
    `adminUsersApi.update`. Invalidates list + individual `['admin-users', id]` queries.
  - Verify: edit a user's title, confirm change persists after reload.

- [ ] **Task 3.5 — Reset password dialog**
  - Files: `features/admin-users/components/reset-password-dialog.tsx` (new)
  - Acceptance: row action "Reset password" (gated by `USER:Update`). Single password
    field + confirm. Calls `adminUsersApi.resetPassword`. Toasts success.
  - Verify: reset a seeded user's password, log out, log in as that user with new password.

- [ ] **Task 3.6 — Delete user confirmation**
  - Files: `features/admin-users/components/delete-user-confirm.tsx` (new)
  - Acceptance: row action "Delete" (gated by `USER:Delete`). Confirmation dialog
    "Delete <name>? This soft-deletes the user." Calls `adminUsersApi.delete`.
    Invalidates list.
  - Verify: delete a test user, row disappears from table.

- [ ] **Task 3.7 — Wire admin sidebar entry**
  - Files: `components/layout/app-shell.tsx` (edit)
  - Acceptance: add to `SIDEBAR_ITEMS`:
    `{ href: '/admin/users', label: 'Employees', icon: Users, section: 'admin', requires: 'USER:View' }`.
  - Verify: as `HR_ADMIN`, "Administration → Employees" appears in sidebar. As
    `EMPLOYEE`, doesn't.

**Phase 3 checkpoint:** Manage Employees ships end-to-end. The framework is
validated through one real consumer.

### Phase 4 — Validation matrix

- [ ] **Task 4.1 — Confirm HR_ADMIN seed exists**
  - Files: read `asima-backend/src/database/seeds/data/`
  - Acceptance: at least one seeded user has role `HR_ADMIN`. If not, add one
    to the seed JSON (idempotent — natural key by email).
  - Verify: `npm run db:fresh && npm run seed` from backend; query DB for an
    HR_ADMIN row.

- [ ] **Task 4.2 — Manual end-to-end matrix**
  - Acceptance: walk each row of the table below and confirm behavior matches.

    | As role | `/employee/home` | `/admin/users` (nav) | `/admin/users` (typed) | Expected |
    |---|---|---|---|---|
    | EMPLOYEE | loads | hidden in sidebar | 403 page | ✓ |
    | HR_ADMIN | loads | visible in sidebar | loads | ✓ |
    | SUPER_ADMIN | loads | visible in sidebar | loads | ✓ |

  - Verify: screenshot or note each row.

- [ ] **Task 4.3 — Backend defense-in-depth spot check**
  - Acceptance: log in as `EMPLOYEE`, open browser devtools, manually `fetch('/api/v1/admin/users')`
    with the access token. Expect 403 from `PermissionsGuard`. Confirms frontend
    gate isn't the only defense.

**Phase 4 checkpoint:** framework is production-ready and validated.

---

## 7. Risk register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Permission codes drift between backend and frontend | Med | Med | Manual sync per `permission-codes.ts`. ADR for code generation if list grows past ~30. |
| `usePermissions` race on first load — sidebar flicker | Low | Low | Cache warms during auth bootstrap. Sidebar items with `requires` simply don't render until first response arrives (no flash). |
| `<RequirePermission>` "loading" state shows briefly even when cache is warm (StrictMode dev only) | Low | Low | Accept the dev-only flicker; production has no double mount. |
| Admin user-create flow lets an admin assign `SUPER_ADMIN` role to a new account (privilege escalation by admin) | Med | High | Out of scope for this plan. Filing as known gap — needs a separate "role assignment policy" decision. The form will expose role_id as a dropdown of all roles for now; future hardening restricts the dropdown server-side. |
| HR_ADMIN can also see their own profile/timesheet/schedule (additive sidebar) — confirmed desired in §2 row 5 | n/a | n/a | Documented decision. If we later need pure role-exclusive UI, reopen. |

---

## 8. What is explicitly out of scope

- **Multi-role support.** Single role per user remains. ADR required to change.
- **Role assignment policy.** Who can assign which role to whom. (See risk register.)
- **Admin sections beyond Employees.** Admin Roles, Admin Permissions, Admin Schedules, etc. — each follows the same pattern but is filed as separate work.
- **A `useHasPermission(code)` hook for inline gating** of individual buttons.
  The plan uses page-level `<RequirePermission>` and config-level sidebar
  predicates. If button-level gating is needed (e.g. hiding "Delete" from
  read-only admins), add this hook in a follow-up.
- **Cypress/Playwright tests for the matrix in Task 4.2.** Manual verification
  is acceptable given the scope. E2E is a separate effort.
- **`/admin` index page.** Hitting `/admin` with no sub-path can 404 — every
  admin route is named explicitly.

---

## 9. The deferred multi-role question

The user asked whether a user can be both HR_ADMIN and EMPLOYEE simultaneously.
Single role per user remains today. The additive-sidebar design (§2 row 5) makes
the question moot for the common case: every authenticated user gets the "My
stuff" section regardless of role, and admin sections layer on top.

The remaining case multi-role would address — a user who's both HR_ADMIN and
PROJECT_MANAGER simultaneously needing both menus — is genuinely a schema change
(many-to-many `user_roles`). When that need is concrete, write an ADR covering:
schema migration, permission union semantics (OR across roles), seed format,
and UI changes (probably none under this framework — `hasPermission` already
just reads a flat array).

Tracking: file as `asima-backend/docs/adr/000X-multi-role-users.md` *when* the
case lands, not before.

---

## 10. Open questions for human review

1. **Section header label.** "Administration" vs "Admin" vs "Manage". Plan says
   "Administration"; happy to swap.
2. **Permission codes module location.** Currently planned at
   `features/auth/permission-codes.ts`. Could equally live at
   `lib/permission-codes.ts` since it's used across features. Recommendation:
   keep under `features/auth/` since permissions are conceptually an auth
   concern.
3. **Create-user role dropdown** (Risk register). Acceptable to ship as
   "any role pickable" for now and harden later, or should we restrict to
   non-SUPER_ADMIN roles in the create dialog from day one?
4. **`/admin/users` page title.** "Manage employees", "Employees", "Users"?
   Plan uses "Employees" in the sidebar, but the page heading can differ.

---

## 11. Acceptance criteria for the plan as a whole

The plan is complete when, after implementation:

- [ ] An `EMPLOYEE` cannot see or reach any `/admin/*` route.
- [ ] An `HR_ADMIN` sees "Administration → Employees" in the sidebar and can
  CRUD users at `/admin/users`.
- [ ] A `SUPER_ADMIN` bypasses per-permission checks (UI and backend).
- [ ] Adding a hypothetical "Admin → Roles" page requires only:
  (a) a new sidebar entry with the right `requires`, and
  (b) a new page wrapped in `<RequirePermission>` — no framework edits.
- [ ] Manual fetch to a protected backend endpoint as an unprivileged user
  returns 403 (defense in depth confirmed).
