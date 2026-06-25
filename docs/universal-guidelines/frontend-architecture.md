# Frontend Architecture

Authoritative reference for how `asima-frontend` is structured. Slice-level
rules also live in `asima-frontend/CLAUDE.md` (loaded automatically when
working under that directory). This document is the deeper "why".

## 1. The shape

`asima-frontend` is a Next.js App Router SPA organized as **Feature-Sliced
Design (FSD)**: vertical feature slices over a small set of shared layers.

```
app/                 Next.js App Router. Route groups (auth)/(app); admin/ +
                     employee/ segments. Thin shells composing feature components.
   │ (downward deps only)
features/<slice>/    Vertical slices:
   ├─ api.ts         HTTP + zod .parse() → anti-corruption boundary.
   │                 Injectable ApiClient (testable).
   ├─ schemas.ts     zod schemas + inferred types; mirror backend domain
   │                 (snake_case end-to-end).
   ├─ keys.ts        Query-key factory — the only place this slice's cache
   │                 keys are defined.
   ├─ hooks/         React Query wrappers (queries + mutations).
   ├─ format.ts /    Pure feature-local helpers.
   │  datetime.ts
   └─ components/     UI: *-page.tsx, drawers, dialogs, cells.
components/          Shared cross-feature UI + shadcn primitives under ui/.
lib/                 api-client (single HTTP entry; token+refresh injected at
                     runtime by AuthProvider), query-client, env, cn, tz,
                     request-id. Never imports from features.
```

## 2. The decision: harden FSD; don't mirror the backend's layering

We deliberately do **not** port the backend's Domain-Driven Design layering
(aggregates / value objects / ports / mappers) onto the frontend.

**Why not.** That layering exists to isolate pure business logic from
infrastructure. On a frontend, the **backend owns the domain**, and **React
Query is the infrastructure/state layer**. Adding `domain/ → ports/ →
adapters/` would mostly produce pass-through mappers wrapping data already
validated by zod — backend-level ceremony protecting a layer with little
independent logic. The two ideas from that layering that do pay off on a
frontend are already present:

- **Anti-corruption boundary:** every `api.ts` validates responses with zod,
  so the frontend has its own contract decoupled from raw backend JSON.
- **Dependency inversion:** `lib/api-client` is feature-agnostic and exposes
  `setAccessToken`/`setRefreshHandler`; `AuthProvider` is the only injector.
  `features → lib`, never the reverse.

## 3. The five rules

1. **zod at the boundary.** Responses are parsed in `api.ts`; components
   consume validated types, never raw JSON.
2. **All server-state in `hooks/`.** Components and `app/.../page.tsx` never
   call `api.ts`, `useQuery`, or `useMutation` directly — they call a named
   hook. Orchestration (invalidations, optimistic updates) lives in the hook,
   in one place, so it is testable and reusable.
3. **Typed query-key factory per slice (`keys.ts`).** No inline key arrays.
   Queries use the leaf (`leaveKeys.list(page)`); invalidation uses a parent
   (`leaveKeys.all`) to prefix-match. A renamed key is one type-checked edit,
   not a hunt across ~30 stringly-typed sites. Enforced by an ESLint
   `no-restricted-syntax` rule forbidding inline `queryKey` arrays.
4. **Slice public API + enforced boundary.** Each slice exposes an `index.ts`
   barrel; cross-slice imports go through it, never into another slice's
   internals. Enforced by ESLint `no-restricted-imports`.
5. **Documented, including the rejected option.** `asima-frontend/CLAUDE.md`
   records the above plus the explicit decision not to mirror the backend's
   DDD layering, and the rationale, so it is not re-litigated.

## 4. State model

React Query for server-state; React context for session/auth (`AuthProvider`).
No Redux/Zustand — do not add a global store without an ADR.

## 5. Migration status

These rules are being adopted incrementally; the plan and progress live in
`asima-frontend/docs/plans/2026-06-14-frontend-architecture-hardening.md` and
`asima-frontend/tasks/`. New code must comply; existing code is migrated
phase by phase.
