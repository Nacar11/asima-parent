# Approval status: pending approver + "(you)" everywhere

**Date:** 2026-06-19
**Repos:** `asima-backend` (read-model) + `asima-frontend` (UI) — cross-cutting
**Status:** Planned (not started)
**Skills used:** `agent-skills:planning-and-task-breakdown` (vertical slices,
AC + verification, checkpoints) and `ui-ux-pro-max` (status-badge semantics,
contrast, tabular figures, truncation, a11y).

---

## 1. Why this plan

Today the **Pending Approvals (Leaves)** inbox shows a bare `STEP` column with
a raw number (`1`). The employee-side **My requests** table already shows a rich
status cell ("Pending L1 / Awaiting L1 · Daniel Aguilar"), but nothing tells the
viewer when *they* are the pending approver.

**Goal:** one consistent approval-status presentation across every surface that
shows where a request sits in the chain — **status + who is pending + an explicit
"(you)" marker when the viewer is the current approver** (e.g.
`Danielle Aguilar (you)`). Applies to leave AND time-correction, on both the
approver inbox and the employee-facing tables.

### Evidence (current state)

| Surface | File | Today | Gap |
|---|---|---|---|
| Leave approvals inbox | `approvals/components/approvals-table.tsx` (`Step` col → `{row.current_step}`) | raw number | no status label, no approver name, no "(you)" |
| TC approvals inbox | same generic `ApprovalsTable` | raw number | same (fixed together) |
| Employee leave "My requests" | `leave/components/leave-request-status-cell.tsx` | badge + "Awaiting L1 · {name}" | no "(you)" |
| Employee/admin TC tables | `time-entries/components/entries-table.tsx` (`ApproverCell`), `time-correction` detail | approver name + state | no "(you)" |

### Root data gap (load-bearing)

The approvals read-model `PendingApproval`
(`approvals/schemas.ts` ⇄ `asima-backend/src/approvals/domain/pending-approval.ts`)
carries `current_approver_id` + `current_step` but **no approver name**. The
employee `LeaveRequest` / `TimeCorrectionRequest` read-models already carry
`l1_approver_name` / `l2_approver_name` and `l1_approver_id` / `l2_approver_id`.

So the inbox needs a **backend read-model enrichment** (`current_approver_name`),
while the employee tables already have everything needed for "(you)" (compare
`l*_approver_id` to the viewer's id from `useAuth`).

The backend already batch-resolves employee names in
`ApprovalsService.resolveNames` (one `findByIds`) — adding approver names folds
into the **same batch**, so there is **no N+1** (mirrors the backend reuse pass).

---

## 2. Non-negotiable guardrails

- **snake_case end-to-end.** New wire field is `current_approver_name` — DB/domain
  → JSON → zod type. No camelCase boundary.
- **zod stays at `api.ts`/`schemas.ts`.** Components consume the validated type;
  add the field to `PendingApprovalSchema` only.
- **Backend layers move together** (ADR/hexagonal): domain → service mapping →
  controller/serialization → unit/e2e test land in one slice.
- **FSD on the frontend.** The shared status cell is cross-feature *UI* →
  `components/`; the pure "is this me / label" logic is a tiny pure helper (unit
  tested). No new server-state in components.
- **Identity from the token.** "(you)" compares ids against `useAuth().user.id`
  — never the URL, never a name-string match.
- **Behavior-preserving.** The employee status cell keeps its current lines; we
  only *append* the "(you)" marker. The inbox `Step`→`Status` swap changes
  presentation, not the query/actions.

---

## 3. UI/UX direction (from `ui-ux-pro-max`)

Data-dense dashboard table; align to the app's **existing** status tokens
(`LEAVE_STATUS_META` amber/emerald/rose on neutral surfaces) rather than a new
palette — consistency over novelty.

- **`color-not-only`** — status is a text badge ("Pending L1"), never color alone.
- **`color-accessible-pairs`** — badge fg/bg ≥ 4.5:1 (reuse vetted token classes).
- **`number-tabular`** — step/level numerals tabular to avoid jitter.
- **`truncation-strategy`** — long approver names wrap (cell is not
  `whitespace-nowrap`); never clip the "(you)" marker.
- **"(you)" emphasis** — semantic, not color-only: a subtle pill/medium-weight
  suffix (`font-medium text-neutral-900` + `· you`), readable in the row.
- **`visual-hierarchy` / `whitespace-balance`** — badge on top, supporting line
  beneath, matching the employee cell's existing rhythm.
- **a11y** — `aria-label` on the status cell summarizing state; row hover/focus
  states already exist; respect `prefers-reduced-motion` (no new motion needed).
- **Responsive** — verify the cell wraps cleanly at 375px (the inbox table
  already scrolls-x in a bordered container).

---

## 4. Plan — vertical slices, dependency-ordered

### Phase 0 — Foundation: read-model + shared helper (no visible behavior change)

- **0.1 Backend — enrich `PendingApproval` with `current_approver_name`.**
  `pending-approval.ts` domain field; in `ApprovalsService.mapLeaves` /
  `mapCorrections`, fold `current_approver_id` into the existing `resolveNames`
  batch and set `item.current_approver_name`; ensure controller serialization
  exposes it. **+ backend unit test**: read-model includes the approver name;
  **+ e2e** assertion on the `/approvals` payload shape.
  - *AC:* `GET /api/v1/approvals` rows include `current_approver_name`; single
    `findByIds` (no extra query per row); backend `test` + `test:e2e` green.
- **0.2 Frontend — extend the zod contract.** Add
  `current_approver_name: z.string()` to `PendingApprovalSchema`.
  - *AC:* `npm run typecheck` green; `api.spec`/`schemas.spec` updated.
- **0.3 Frontend — shared pure helper.** `approverLabel(name, isSelf)` →
  `"Danielle Aguilar (you)"` / `"Danielle Aguilar"`; `null` → `"—"`. Lives in a
  shared module (`components/approval-status/` or `lib/`), **+ Vitest** (self,
  not-self, null).
  - *AC:* unit test green; pure, no feature imports.
- **Checkpoint A:** backend + frontend `typecheck/lint/build/test` green; the new
  field round-trips; nothing renders it yet (pure addition).

### Phase 1 — Pilot: leave approvals inbox `STEP` → `STATUS` (the screenshot)

- **1.1 `ApprovalStatusCell`** (`components/approval-status/approval-status-cell.tsx`):
  status badge `Pending L{current_step}` (reuse the amber "pending" token) + a
  supporting line `Awaiting L{step} · {approverLabel(current_approver_name, isSelf)}`
  where `isSelf = current_approver_id === viewerId`. **+ Vitest** (self shows
  "(you)"; other shows plain name; step→L1/L2).
- **1.2 Wire into `ApprovalsTable`**: rename the `Step` header to `Status`, render
  `<ApprovalStatusCell row viewerId />`; thread `viewerId` from `ApprovalsInbox`
  (`useAuth().user.id`) through the table. Cell is wrap-friendly (drop
  `whitespace-nowrap` for that column).
  - *AC:* leave inbox shows `Pending L1` + `Awaiting L1 · <name>(you)` for the
    viewer's own queue and `· <other name>` in the `canSeeAll` view; existing
    `approvals-table` / `approvals-inbox` specs updated and green; manual smoke
    of `/leave-approvals`.
- **Checkpoint B:** reference implementation reviewed before reuse.

### Phase 2 — Time-correction inbox (free via the generic table)

- **2.1** Confirm the TC inbox (`time-correction-approvals`) renders the same
  `ApprovalStatusCell` (it shares `ApprovalsInbox`/`ApprovalsTable`); add/adjust
  the TC inbox spec.
  - *AC:* TC inbox shows the same status + "(you)" treatment; specs green; manual
    smoke of `/time-correction-approvals`.

### Phase 3 — Employee-side consistency ("(you)" on the tables the requester sees)

- **3.1 Leave "My requests" status cell** (`leave-request-status-cell.tsx`):
  append "(you)" via `approverLabel` when `l1_approver_id`/`l2_approver_id`
  equals the viewer's id (thread `viewerId` from the page's `useAuth`). Keep all
  existing lines; only the name span gains the marker.
  - *AC:* a request whose pending approver is the viewer shows `· <name> (you)`;
    `leave-request-status-cell` / `employee-leaves-page` specs updated; behavior
    otherwise identical.
- **3.2 (EVALUATE) Time-correction tables** — `entries-table` `ApproverCell`
  (timesheet) and the TC detail/admin views that show approver names: append
  "(you)" via the same helper where it reads naturally; skip any spot where the
  approver isn't shown or the marker would clutter (note it).
  - *AC:* adopted spots show "(you)" consistently; cross-slice imports via
    `index.ts`; specs green.
- **Checkpoint C:** `grep` confirms one shared `approverLabel`/`ApprovalStatusCell`
  is the only place this text is built; all four surfaces consistent.

### Phase 4 — UI/UX polish + a11y pass (`ui-ux-pro-max`)

- **4.1** Apply §3 rules: badge contrast AA (verify token pairs), tabular numerals
  on the L-level, long-name wrapping (375px check), `aria-label` summary on the
  status cell, focus/hover parity, no color-only signalling, no new motion (or
  reduced-motion-safe).
  - *AC:* `--domain ux "accessibility color contrast table"` checklist passes;
    375 / 768 / 1024 widths verified; no bundle regression.
- **Checkpoint D (final):** full `typecheck + lint + build + test` green in both
  repos; manual smoke of all four surfaces; one `current_approver_name` field and
  one shared status presentation.

---

## 5. Dependency graph

```
0.1 backend current_approver_name ─┐
0.2 zod field ─────────────────────┼─→ 1.1 ApprovalStatusCell ─→ 1.2 inbox wire ─→ 2.1 TC inbox
0.3 approverLabel helper ──────────┘                         └─────────────→ 3.1 leave My-requests
                                                                              └→ 3.2 TC tables (eval)
                                                              all ───────────────→ 4.x UX polish
```

Phase 0 lands first (the inbox cannot show an approver name without 0.1; "(you)"
needs 0.3). Phase 1 is the reviewed reference; 2–3 reuse it; 4 polishes.

## 6. Verification model

Per slice — **backend:** `npm run test` + `npm run test:e2e` (read-model shape).
**frontend:** `npm run typecheck`, `npm run lint:ci`, `npm run build`,
`npm run test` (Vitest for the pure helper + the status cell + updated table/inbox
specs). Plus **manual smoke** of each surface. The shared types make a mismatch a
compile error; one surface per task keeps regressions bisectable.

## 7. Out of scope (separate plan / ADR)

- Changing the approval-chain model or who-approves-what logic (ADR 0001 stands).
- Real-time/websocket inbox updates.
- Surfacing approver *email*/avatar (name + "(you)" only).
- Any pagination/filter changes (covered by the prior reuse plan).

## 8. Rollback

Each slice is an isolated, behavior-preserving commit (backend and frontend
separately revertible). 0.1/0.2 are additive (an unused field); reverting a
surface restores its prior cell without touching the shared helper.
