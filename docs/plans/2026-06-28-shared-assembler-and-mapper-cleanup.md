# Implementation Plan: Shared Assembler Helper + Dead `toPersistence` Cleanup

> Snapshot frozen 2026-06-28. Working copy lives in
> `asima-backend/tasks/plan.md`; checklist in `asima-backend/tasks/todo.md`.
> Backend-only, cross-cutting refactor; this parent-repo copy is the audit trail.

## Overview

A cross-cutting **reuse + dead-code** cleanup, prompted by the work-schedules
revamp review. Two independent concerns, each its own commit:

1. **Shared assembler helper.** All 6 module assemblers repeat the same 3
   structural-copy atoms. Extract them into one `src/utils/helpers/assemble.ts`
   so the boilerplate (and its "structural copy / snake_case 1:1" rationale)
   lives once. The assemblers keep their **static public API unchanged**
   (`XAssembler.toResponse(...)`), so **no controller changes** — only the
   method bodies delegate to the helper.

2. **Delete dead `toPersistence`.** 5 mappers define a `toPersistence` with
   **zero callers** (repos build entities via `repo.create({...})` directly).
   Remove them. **`storage`'s `toPersistence` stays** — its round-trip spec
   (`attachment.mapper.spec.ts:47`) is a live caller.

**Honest scope of the win:** moderate, not transformative. It consolidates ~10
lines of genuine duplication (the paginated + list shapes, repeated 6×) and
centralizes a doc rationale; it is past the rule-of-three (6 use sites). It does
**not** change behavior, signatures, or the wire — the existing e2e byte-parity
suites are the guard.

## Architecture decisions

1. **Free generic helper functions, NOT a base class.** Static methods don't
   inherit cleanly with generics, and an instance/abstract base would force
   every controller to change how it references the assembler (inject vs.
   static call) — a large, risky change for no behavioral gain. Free functions
   keep each assembler's static API identical; only the body changes. The helper
   is constructor-injected as a value (`new () => T`), matching the current
   `new Dto()` calls.

2. **Helper home: `src/utils/helpers/assemble.ts`** — the repo's function
   helpers live in `src/utils/helpers/` (`dates.ts`, `http-errors.ts`,
   `pagination.ts`, `pg-errors.ts`, `time-of-day.ts`); this fits the convention.
   Three functions:

   ```ts
   import { PaginatedResponse } from '@/utils/types/paginated-response.type';

   /**
    * Structural copy of a domain record/read-model onto its response DTO.
    * The wire mirrors the domain 1:1 (snake_case end-to-end, no translation
    * layer — a project invariant), so this is a faithful field copy; the e2e
    * suites guard byte-for-byte parity. Loose `src` typing matches the prior
    * `Object.assign(new Dto(), src)` call sites exactly (records sometimes
    * carry MORE fields than the DTO — e.g. list items extend records).
    */
   export function toDto<T extends object>(Dto: new () => T, src: object): T {
     return Object.assign(new Dto(), src);
   }

   /** Map a PaginatedResponse's `data` onto response DTOs, preserving the envelope. */
   export function toPaginatedDto<T extends object>(
     Dto: new () => T,
     page: PaginatedResponse<object>,
   ): PaginatedResponse<T> {
     return { ...page, data: page.data.map((item) => toDto(Dto, item)) };
   }

   /** Map a plain array onto response DTOs. */
   export function toDtoList<T extends object>(Dto: new () => T, list: object[]): T[] {
     return list.map((item) => toDto(Dto, item));
   }
   ```

3. **Bespoke methods stay bespoke.** `schedule-change.assembler.toResultResponse`
   keeps its nested `created_row` mapping — it uses `toDto` for the base shape,
   then overrides `created_row` via `WorkScheduleAssembler.toResponse`. The
   helper does not try to absorb the special case.

4. **Behavior frozen.** Every change is `Object.assign(new Dto(), src)` →
   `toDto(Dto, src)` and the paginated/list equivalents — identical runtime
   output. No signature, no controller, no DTO, no wire change. The dead-code
   removal deletes only never-called methods.

## Scope boundary

**Touched:** `src/utils/helpers/assemble.ts` (new) + the 6 assembler bodies +
5 mapper files (delete `toPersistence`). Plus `main.ts`? **No** — Swagger groups
reference DTO class names, which don't change.

**NOT touched (verified):**
- **Controllers** — assembler public API (`XAssembler.toResponse`) is unchanged.
- **`storage` `toPersistence`** — LIVE (its spec round-trips it); keep it.
- **leave-requests / leave-allocations / time-correction mappers** — they never
  defined `toPersistence` (they use `toAggregate`/`toDomain`/`toListItem`).
- **DTOs, domain, persistence ports, services** — untouched.
- The `toDomain` half of every mapper — keeps its `XEntity` import (still its
  param type), so deleting `toPersistence` leaves no unused import.

## Cross-cutting facts (verified by grep)

| Assembler | Methods using an atom | Atoms |
|---|---|---|
| `leave-allocation` | `toResponse`, `toResponseList` | `toDto`, `toDtoList` |
| `leave-request` | `toResponse`, `toListItemResponse`, `toPaginatedResponse`, `toBalanceResponse`, `toBalanceResponseList` | `toDto`×3, `toPaginatedDto`, `toDtoList` |
| `time-correction-request` | `toResponse`, `toListItemResponse`, `toPaginatedResponse` | `toDto`×2, `toPaginatedDto` |
| `time-entry` | `toResponse`, `toPaginatedResponse` | `toDto`, `toPaginatedDto` |
| `work-schedule` | `toResponse`, `toPaginatedResponse` | `toDto`, `toPaginatedDto` |
| `schedule-change` | `toImpactResponse`, `toResultResponse` (nested) | `toDto` + bespoke override |

**Dead `toPersistence` (zero callers):** `compensation`, `permissions`, `roles`,
`time-entries`, `work-schedules`. **Live (keep):** `storage`.

## Task List

### Phase 1 — Foundation

#### Task 1: Shared `assemble.ts` helper + unit test
**Description:** Create the three generic helpers; prove each preserves the exact
`Object.assign` / spread-map behavior with a small unit spec.

**Acceptance criteria:**
- [ ] `src/utils/helpers/assemble.ts` exports `toDto`, `toPaginatedDto`,
      `toDtoList` exactly as in decision #2.
- [ ] `src/utils/helpers/assemble.spec.ts`: `toDto` copies all own fields onto a
      fresh DTO instance (and returns that class instance); `toPaginatedDto`
      preserves `total`/`page`/`limit`/`has_more` and maps `data`; `toDtoList`
      maps each item. Use a throwaway `class FooDto { a!: number; b!: string }`.
- [ ] `npm test -- assemble` green.

**Verification:** `npm test -- assemble`; `npm run build`.
**Dependencies:** None. **Scope:** Small (2 files).

### Phase 2 — Migrate assemblers (behavior-preserving)

#### Task 2: Migrate the 5 simple assemblers
**Description:** Replace the inline `Object.assign` / spread-map / `list.map`
bodies in the 5 non-nested assemblers with the helper. Public method signatures
unchanged. Drop the now-redundant per-class "structural copy" doc paragraph down
to a one-line pointer (the rationale lives on `toDto` now).

**Acceptance criteria:**
- [ ] `leave-allocation`, `leave-request`, `time-correction-request`,
      `time-entry`, `work-schedule` assemblers delegate every method to
      `toDto` / `toPaginatedDto` / `toDtoList`. No remaining raw
      `Object.assign(new …)` or `{ ...page, data: … }` in these 5 files.
- [ ] `leave-request.toPaginatedResponse` passes the **list-item** DTO
      (`toPaginatedDto(LeaveRequestListItemResponseDto, page)`); same for
      `time-correction-request`.
- [ ] Imports cleaned (drop `PaginatedResponse` where the body no longer
      spells it, keep where a return type still names it).

**Verification:** `npm test` (unit); `npm run build`; lint clean. The
leave-requests / time-correction / time-entries / leave-allocations e2e suites
stay green (byte parity) — run in Task 4's gate.
**Dependencies:** Task 1. **Scope:** Medium (5 files).

#### Task 3: Migrate `schedule-change.assembler` (nested case)
**Description:** Use `toDto` for `toImpactResponse` and the base of
`toResultResponse`, keeping the bespoke `created_row` override.

**Acceptance criteria:**
- [ ] `toImpactResponse` → `toDto(ScheduleChangeImpactResponseDto, src)`.
- [ ] `toResultResponse` → `toDto(ScheduleChangeResultResponseDto, src)` then
      override `created_row` via `WorkScheduleAssembler.toResponse` (null-safe),
      identical output to today.
- [ ] No raw `Object.assign(new …)` left in the file.

**Verification:** `npm test`; `npm run build`. `schedule-changes.e2e` green in
Task 4's gate. **Dependencies:** Task 1. **Scope:** Small (1 file).

### Checkpoint A — Reuse done (commit #1)
- [ ] `npm run lint && npm run test && npm run build` green.
- [ ] Full e2e parity: `SEED_DEFAULT_PASSWORD=Asima@1234 npm run test:e2e`
      (Postgres + MinIO + `STORAGE_*`) — all 139 tests green, proving the
      assembler bodies produce byte-identical wire output.
- [ ] Commit: `refactor: extract shared response-DTO assembler helpers`.

### Phase 3 — Dead-code sweep (separate concern)

#### Task 4: Delete the 5 dead `toPersistence` methods
**Description:** Remove `toPersistence` from the `compensation`, `permissions`,
`roles`, `time-entries`, `work-schedules` mappers. **Keep `storage`'s** (live).

**Acceptance criteria:**
- [ ] The 5 `toPersistence` methods deleted; `toDomain` (and its `XEntity`
      import) untouched.
- [ ] `grep -rn "\.toPersistence(" src` returns **only**
      `attachment.mapper.spec.ts` (storage's round-trip) — confirming nothing
      else regressed.
- [ ] `storage` mapper + spec untouched and green.

**Verification:** `npm run lint && npm run test && npm run build`. (No e2e needed
— deleting uncalled methods can't change the wire, but the Checkpoint B gate runs
it anyway.) **Dependencies:** None (independent of Tasks 1–3). **Scope:** Small
(5 files, deletions only).

### Checkpoint B — Complete (commit #2)
- [ ] All gates pass: `lint && test && test:e2e && build`.
- [ ] Commit: `refactor: drop dead toPersistence mapper methods (keep storage)`.
- [ ] Both commits pushed to `main` in `asima-backend`.

## Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| `toDto` helper behaves differently than inline `Object.assign` | **High** | It IS `Object.assign(new Dto(), src)` verbatim; Task 1 unit spec + the 139-test e2e parity guard prove byte-identical output. |
| A `toPaginatedResponse` accidentally maps the base DTO instead of the list-item | Med | Task 2 criterion pins the list-item DTO for leave-request + time-correction; e2e asserts the `*_name` join fields survive. |
| Deleting a `toPersistence` that is actually called | **High** | Verified: only call-site in the repo is storage's own spec; Task 4 keeps storage's and re-greps to confirm. |
| Churn for marginal gain (skill's "maintain balance" caution) | Low | Scoped to genuine ≥6× duplication + the live e2e guard; the `toDto` atom also centralizes the structural-copy rationale (one doc home, not six). |
| Loose `src: object` typing hides a field mismatch | Low | Matches today's untyped `Object.assign`; the DTO is the wire contract and e2e guards it. Tightening to `Partial<T>` would break list-items (which carry extra fields) — deliberately not done. |

## Decisions (2026-06-28)

1. **Two commits — DECIDED.** Reuse (Tasks 1–3) and the dead-code sweep (Task 4)
   ship as separate commits: independent concerns, cleaner history, easier revert.
2. **Include `toDto` — DECIDED.** All three helpers ship (`toDto` /
   `toPaginatedDto` / `toDtoList`). `toDto` buys uniformity (every assembler
   method delegates to a named helper, no raw `Object.assign` scattered) and one
   home for the structural-copy rationale.

## Open questions

1. **`toDtoList` under the rule-of-three (review finding R-1).** It has only
   **2** call sites today (`leave-allocation.toResponseList`,
   `leave-request.toBalanceResponseList`) vs. `toPaginatedDto` (4) and `toDto`
   (10). Kept for symmetry — a list-of-DTOs is the most fundamental mapping
   shape and a 3rd use is near-certain as modules grow — but flag at
   implementation if it still feels premature; the fallback is inlining the 2 as
   `list.map((x) => toDto(Dto, x))`.
