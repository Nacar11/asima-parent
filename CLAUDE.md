# asima

Asima is an Ashima-inspired **Employee Time Management** system: identity →
leave / approvals → schedules → time entries → workforce. Single-tenant
(one company per deployment). v0 ships only the identity foundation.

This file is the **system-level** brief — it covers the whole product across
frontend and backend. Backend-specific rules live in
`asima-backend/CLAUDE.md` and load automatically when working under that
directory.

## Repo layout

```
asima-parent/
├── asima-backend/   # NestJS + TypeORM + Postgres API. See its own CLAUDE.md.
└── asima-frontend/  # not yet scaffolded
```

When asked to work on the frontend before it exists, scaffold it as a
sibling to `asima-backend/` and add a matching `asima-frontend/CLAUDE.md`.
Don't grow frontend code inside the backend tree.

## Cross-cutting concepts (read before touching auth, users, or approvals)

The terminology below is **load-bearing** — the backend ADR
`asima-backend/docs/adr/0001-roles-and-approval-design.md` records why these
are kept orthogonal. Frontend code must use the same vocabulary.

- **Role** (`users.role_id`) — global capability, drives permission gates.
  One per user. Examples: `SUPER_ADMIN`, `HR_ADMIN`, `PROJECT_MANAGER`,
  `TECHNICAL_DIRECTOR`, `EMPLOYEE`.
- **Title** (`users.title`) — freeform display string ("Senior PM", "Acting
  TD"). **Never** drives auth or routing. UI shows it; code ignores it.
- **Approval chain** (per-employee assignment table, lands with the leave
  module) — drives *who* approves *this* request. Distinct from role.

A user's role says whether they can call `LEAVE:Approve` at all. The
approval chain says whether they're the right approver for *this* employee's
*this* request. Don't conflate them on either side of the wire.

## Permission codes

Shape: `RESOURCE:Action` (e.g. `USER:Create`, `LEAVE:Approve`). Frontend
should drive UI gating from `GET /api/v1/users/me/permissions` (a flat
string array) — never parse `role.permissions` client-side.

## Admin vs. self-service: same resource, two surfaces

Every resource that has both an admin-management surface AND a
self-service surface (Users today; OT, leave, profile photo, etc. later)
follows this contract on both sides of the wire:

1. **Two routes, two audiences.** Admin lives at `/admin/<resource>` and
   is gated by `RESOURCE:Action` permissions. Self-service lives at
   `/<resource>/me` (or `/me/<sub-resource>` once the noun isn't users)
   and is gated only by JWT identity.
2. **Identity comes from the token, never from the URL.** `/users/me`
   has no `:id` segment — the server derives it from `req.user.id`. This
   is why the answer to "what if an admin hits /me?" is uninteresting:
   they update *themselves*, with the same narrow field set as any
   employee. To act on someone else, they must use `/admin/<resource>/:id`.
3. **The narrow surface is enforced by DTO, not by `if (admin)`.** The
   self-service DTO declares only the fields the user is allowed to
   change. The global `ValidationPipe` (`forbidNonWhitelisted: true`)
   rejects anything else with 400. There must be no runtime branch that
   reads `caller.role` to decide which fields to accept — branching
   like that is exactly how privilege escalation gets shipped.
4. **Privileged operations get their own endpoints, not extra fields.**
   Password change is the canonical example: `PATCH /users/me/password`
   takes `{ current_password, new_password }`; admin force-reset uses
   `POST /admin/users/:id/reset-password` and skips the current-password
   check. Don't fold either into a generic `PATCH` body — credentials
   shouldn't ride alongside a profile patch.
5. **Same domain, same persistence, same service.** The split lives at
   the *edge* (DTO + controller). Business invariants (uniqueness,
   referential integrity, hashing) must NOT diverge by audience. If
   `/me` skips a check the admin path enforces, that's a bug.

## API contract conventions

- All routes mounted under `/api/v1/...` (URI versioning). Bump via the
  shared `API_VERSION` constant on the backend.
- List endpoints return `{ data, total, page, limit, has_more }`. Frontend
  pagination components should target this shape directly.
- Field names are **snake_case end-to-end** — DB columns, domain objects,
  and JSON wire payloads (`created_at`, `password_hash`, `is_active`). Don't
  introduce a camelCase translation layer at the boundary.
- Every response carries `X-Request-ID`. Propagate it from frontend logs and
  any onward HTTP calls so requests can be correlated.
- Bearer JWT auth. Access token 15m, refresh token 7d with rotation on
  `/auth/refresh`. Logout is stateless (client drops tokens).

## Plans and todos — `tasks/` vs `docs/plans/`

Every project root in this repo (parent, backend, frontend) follows the
same convention for plan + todo files:

| Location | Tracked? | Role | Lifetime |
|---|---|---|---|
| `tasks/plan.md` | **Gitignored** | The currently-active plan for the feature being worked on right now. Contents change every time the active feature changes. | Mutable working file |
| `tasks/todo.md` | **Gitignored** | The currently-active checklist. Same file pair as `tasks/plan.md`. | Mutable working file |
| `docs/plans/YYYY-MM-DD-<slug>.md` | **Committed** | Audit snapshot of the plan as it was at the time the feature started. Permanent record. | Frozen at write time |
| `docs/plans/YYYY-MM-DD-<slug>-todo.md` | **Committed** | Audit snapshot of the todo. | Frozen at write time |

**The workflow when starting a new major feature:**

1. Write the plan + todo as `docs/plans/YYYY-MM-DD-<slug>.md` and
   `docs/plans/YYYY-MM-DD-<slug>-todo.md`. Commit them — this is the
   audit trail.
2. Copy both files to `tasks/plan.md` and `tasks/todo.md`. These
   become the working files. They're gitignored, so day-to-day edits
   don't pollute git history.
3. As work progresses, edit `tasks/plan.md` and `tasks/todo.md`
   freely. Tick boxes, add notes, revise estimates.
4. If a plan revision is significant enough to need a record (a
   pivot, a new phase, a reversed decision), update the
   `docs/plans/` snapshot too and commit.
5. When the feature ships, the `docs/plans/` files remain as the
   historical record of what was planned and why. The `tasks/` files
   get overwritten by the next feature's plan.

**Why this shape:** `tasks/` is a *workspace*, not a *record*.
Committing every checkbox tick would bury PR diffs in noise. But
forgetting why a decision was made six months later is its own pain.
The `docs/plans/` snapshot solves both — it's what we agreed to do at
the start; the `tasks/` file is how we're doing it now.

**Conventions:**

- `docs/plans/` filenames MUST start with `YYYY-MM-DD-`
  (e.g. `2026-05-30-leave-correction-and-approval-chains.md`).
- The `tasks/` file pair has only the two canonical names:
  `tasks/plan.md` and `tasks/todo.md`. No date prefix; the active
  feature is whichever one is in those files right now.
- The Claude Code planning skill defaults to `tasks/plan.md` /
  `tasks/todo.md` — this convention aligns with that default and
  adds the audit copy in `docs/plans/`.

## Where authoritative docs live

- **Active plan** for whatever feature is in flight:
  - Parent (cross-cutting features): `asima-parent/tasks/plan.md` + `tasks/todo.md`.
  - Backend-only: `asima-backend/tasks/plan.md` + `tasks/todo.md`.
  - Frontend-only: `asima-frontend/tasks/plan.md` + `tasks/todo.md`.
- **Historical plans**: `docs/plans/YYYY-MM-DD-*.md` at the appropriate root.
- `asima-backend/docs/adr/` — architectural decisions. Read the relevant
  ADR before changing the area it covers.
- `asima-parent/docs/universal-guidelines/module-architecture.md` —
  hexagonal layering blueprint every backend module follows.
- `asima-parent/docs/universal-guidelines/database-migration-conventions.md`
  — one table = one CREATE migration; edit CREATE (not a new ALTER) while a
  table's migration is still unreleased.
- `asima-parent/docs/universal-guidelines/frontend-stack.md` —
  authoritative stack table; one source per layer.
- `asima-parent/docs/universal-guidelines/frontend-component-blueprint.md`
  — prompt-context rules for creating any frontend component
  (shadcn-only, no parallel UI sources).
- `asima-backend/reference/` — exemplar code (categories module) used as a
  pattern reference. Not shipping code; don't import from it.

## Working style

- Don't add features beyond what the task requires. v0 is deliberately
  scoped to identity; flag scope drift instead of silently expanding it.
- Prefer editing existing files over creating new ones. The hexagonal
  layout means every concern already has a home — find it before adding a
  new file.
- For any non-trivial backend change, the layered file set
  (domain → persistence → service → controller) must move together. A
  half-landed slice (entity without mapper, controller without DTO) is
  worse than no change.

## Git workflow

- **Commit straight to `main` and push.** Do NOT create feature branches or
  open PRs unless the user explicitly asks for one in that request. The
  default for every change — in `asima-parent`, `asima-backend`, and
  `asima-frontend` — is: make the change on `main`, commit, `git push origin
  main`. (This overrides the usual "branch first if on the default branch"
  default.)
- Still **verify before pushing**: run the repo's lint / test / build (and
  `npm ci` if a lock file changed) and only push when they pass. Direct-to-main
  doesn't mean skipping the checks — it means skipping the branch + PR ceremony.
- Each repo has its own `origin` and its own `main`; push to the one whose
  files changed. Keep commits scoped to one concern even when several land in
  the same session.
