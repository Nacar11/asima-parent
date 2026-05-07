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

## Where authoritative docs live

- `asima-backend/tasks/plan.md` — current implementation plan and phase
  checkpoints. Source of truth for what's built vs. coming.
- `asima-backend/tasks/todo.md` — active work items.
- `asima-backend/docs/adr/` — architectural decisions. Read the relevant
  ADR before changing the area it covers.
- `asima-backend/module-architecture.md` — hexagonal layering blueprint
  every backend module follows.
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
