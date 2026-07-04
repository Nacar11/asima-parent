# ADR 0001 — Roles, titles, and approval chains are orthogonal

- **Status:** Accepted. Documented retroactively on 2026-07-04; the decision was
  made and implemented earlier (it predates the leave/approval-chain module). This
  ADR records the reasoning that `CLAUDE.md`, `asima-backend/CLAUDE.md`, and the
  `CreateApprovalChainsTable` migration already cite as load-bearing.
- **Date:** 2026-07-04 (retroactive)
- **Deciders:** Asima core (single-tenant v0)

## Context

Asima must answer three different questions about a user. They look similar and
are easy to collapse into one field — which is exactly how privilege escalation
and mis-routed approvals get shipped:

1. **Capability** — "May this user perform action X *at all*?" e.g. may they call
   `LEAVE:Approve`, `USER:Create`. This is a global, coarse gate.
2. **Approval routing** — "Is this user the right approver for *this* employee's
   *this* request?" Two users may both hold `LEAVE:Approve`, but only one is *this*
   employee's assigned approver at *this* step.
3. **Display** — "What is this person's job title?" ("Senior PM", "Acting TD").
   Human-facing text with no security meaning.

The tempting shortcuts, and why each is a foot-gun:

- **Drive permissions from the job title.** Titles are freeform and edited for HR/
  display reasons; letting them gate actions means a display edit silently grants
  power. Unauditable and unsafe.
- **Drive approval routing from the role.** "Anyone with `LEAVE:Approve` can approve
  anyone" ignores org structure — an employee's leave should route to *their*
  manager, not to any approver in the company. It also can't express two-level
  chains or per-employee reassignment.
- **Collapse role and title into one field.** Then you can neither have two "Senior
  PMs" with different permissions, nor an `EMPLOYEE`-role user displayed as "Acting
  TD" during a transition.

Permissions themselves are modeled as `RESOURCE:Action` codes (`USER:Create`,
`LEAVE:Approve`), seeded config, joined to roles via a `role_permissions` M2M.

## Decision

Keep the three concepts as **three orthogonal axes**, each with its own storage and
its own job. None may stand in for another.

### 1. Role — `users.role_id` (global capability)

- Exactly **one role per user**. Drives the `PermissionsGuard`: a route annotated
  `@Permissions('LEAVE:Approve')` passes only if that code is in the user's role's
  permission set (AND-semantics across multiple codes).
- `SUPER_ADMIN` short-circuits the guard via a `system_admin: true` flag on the user
  record — reserved for ops/infra, not a normal grant.
- **Role catalog and its evolution:** v0 seeds `SUPER_ADMIN` / `ADMIN` / `EMPLOYEE`.
  This ADR is the authority under which the catalog **evolves to add** `HR_ADMIN`,
  `PROJECT_MANAGER`, and `TECHNICAL_DIRECTOR` as the leave/approval and workforce
  modules land. Adding a role is a seed change (`roles` + `role_permissions`), not a
  schema or controller change.
- **The cleanest proof of orthogonality:** `PROJECT_MANAGER` and `TECHNICAL_DIRECTOR`
  are seeded with an **identical permission set**. What distinguishes a 1st-level
  from a 2nd-level approver is *not* their role — it is their **step in a specific
  employee's approval chain**. Role answers "may approve leave at all"; the chain
  answers "approves *this* request, at *this* level." (A seed grant-matrix test
  asserts this equality and cites this ADR.)

### 2. Title — `users.title` (freeform display, never auth)

- A freeform string ("Senior PM", "Acting TD"). **The UI shows it; code ignores it.**
- Title **never** drives authorization or routing. There is deliberately no `titles`
  lookup table and no code path that branches on title.

### 3. Approval chain — per-employee assignment table (who approves *this*)

- A versioned, per-employee table (`approval_chains`): "for THIS employee, at THIS
  step, THIS approver may act, effective from … until … ." It answers the *routing*
  question, independent of what role the approver holds.
- Reassignment is **logical-end + insert** (mirrors `work_schedules`): stamp the
  active row's `ended_at`, insert a new row. Historical requests snapshot their
  approver at submit time, so past requests still resolve to the approver who was
  active then. At most one active approver per `(employee, step)` — enforced by a
  partial unique index, not just service code.

### The dividing line, on both sides of the wire

> A user's **role** says whether they can call `LEAVE:Approve` at all. The **approval
> chain** says whether they're the right approver for *this* employee's *this*
> request. Don't conflate them on either side of the wire.

## Consequences

**Positive**

- No privilege escalation through a display edit; titles are safe to let HR change.
- Approver reassignment never touches roles or permissions, and vice versa.
- Two-level chains and per-employee routing are expressible; historical requests
  remain auditable against the approver active at submit time.
- The admin/self-service split is enforced by **DTO shape** (`forbidNonWhitelisted`
  rejects fields the self-service DTO doesn't declare), *not* by a runtime
  `if (caller.role === admin)` branch — the same orthogonality applied at the edge.

**Costs / obligations**

- Three concepts must be kept straight by everyone touching auth, users, or
  approvals. This ADR is the canonical reference; read it before changing those areas.
- **Frontend must use the same vocabulary** and drive UI gating from
  `GET /api/v1/users/me/permissions` (a flat string array) — never parse
  `role.permissions` client-side, and never gate on `title`.
- Authorization to *act on a request* is computed from the approval chain, not from
  role permissions; a service that checks only the role is a bug.

## References

- `CLAUDE.md` → "Cross-cutting concepts" and "Permission codes".
- `asima-backend/CLAUDE.md` → "Permissions / roles" and the auth/guards pipeline
  (`PermissionsGuard`, `system_admin` bypass).
- `asima-backend/src/database/migrations/*-CreateApprovalChainsTable.ts` — the
  versioned per-employee approver table and its partial unique index.
