# Plan — Reusable `Checkbox` primitive (in-stack, shadcn pattern)

**Date:** 2026-05-30
**Owner:** frontend
**Scope:** asima-frontend
**Stack decision:** **No new UI source required.** shadcn ships a
Checkbox component built on `@radix-ui/react-checkbox` — the same
Layer-1/Layer-2 pattern documented in
[`docs/universal-guidelines/frontend-stack.md`](../universal-guidelines/frontend-stack.md) and already used for
Sheet (`@radix-ui/react-dialog` → `src/components/ui/sheet.tsx`).

> If shadcn did NOT ship a Checkbox, this plan would have included an
> ADR to introduce one. It does, so we follow the established pipeline:
> `npx shadcn@latest add checkbox` → owned source at
> `src/components/ui/checkbox.tsx` → compose in feature code.

---

## 1. Problem and demonstration target

`edit-user-drawer.tsx:131` (and the matching spot in
`create-user-drawer.tsx`) currently renders the **Active** field as a
raw `<input type="checkbox">`:

```tsx
<input
  type="checkbox"
  className="h-4 w-4 rounded border-neutral-300"
  {...form.register('is_active')}
/>
```

Screenshots show the resulting affordance is inconsistent across
browsers (Chrome's blue-rounded box vs. Brave's larger native control)
and the keyboard / a11y story relies on whatever the browser ships.

The user-facing goal: a single owned `Checkbox` that
- looks the same in every browser (Tailwind-styled, design-token aware),
- is accessibly labeled and keyboard-operable by default (Radix
  guarantees ARIA + focus management),
- plugs into React Hook Form without a one-off `Controller` per
  call site,
- and is reusable for every future "boolean toggle inline with a label"
  — not just Active.

---

## 2. Why this is in-stack (and what would change that)

Per the hard rules in `docs/universal-guidelines/frontend-stack.md` §"Hard rules":

| Layer | Tool | Status for Checkbox |
|---|---|---|
| Layer 1 — Headless behavior | `@radix-ui/react-checkbox` | **Will be added** by the shadcn CLI. Same family as `@radix-ui/react-dialog` we already use. No new ecosystem. |
| Layer 2 — Owned primitive | shadcn-pattern source at `src/components/ui/checkbox.tsx` | **Will be added.** Style with Tailwind; reads from the existing `cn` helper and design tokens. |
| Layer 3 — App code | `edit-user-drawer.tsx`, `create-user-drawer.tsx`, future forms | Composes `<Checkbox>` + RHF. |

**This plan would need an ADR if** any of the following were true (none
are):
- We wanted to source the primitive from Headless UI, Material, Chakra,
  or any non-shadcn library — forbidden by Hard Rule #1.
- We wanted CSS-in-JS for styling — forbidden by Hard Rule #2.
- We wanted to fork the shadcn primitive to add feature logic — Hard
  Rule #5 says compose at Layer 3 instead.

---

## 3. Dependency graph

```
┌────────────────────────────────────────────────────────────┐
│ Task 1 — Bring in the primitive                            │
│   npx shadcn@latest add checkbox                           │
│   → installs @radix-ui/react-checkbox                      │
│   → writes src/components/ui/checkbox.tsx                  │
└─────────────────────────┬──────────────────────────────────┘
                          │
              ┌───────────┴────────────┐
              │                        │
              ▼                        ▼
┌──────────────────────────┐   ┌──────────────────────────────┐
│ Task 2 — Form-friendly   │   │ Task 3 — Unit test the       │
│ Field wrapper / convention│   │ primitive (controlled +      │
│ (so RHF callsites are 1  │   │ uncontrolled, a11y label,    │
│ line, not 5)             │   │ keyboard space toggles)      │
└────────────┬─────────────┘   └──────────────────────────────┘
             │
   ┌─────────┼──────────┐
   │         │          │
   ▼         ▼          ▼
┌────────┐┌────────┐┌────────────────────────┐
│ Task 4 ││ Task 5 ││ Task 6 — Document the  │
│ Replace││ Replace││ pattern: add a row to  │
│ in     ││ in     ││ frontend-stack.md and  │
│ edit-  ││ create-││ a short note in the    │
│ user-  ││ user-  ││ admin-users README (if │
│ drawer ││ drawer ││ one exists) so future  │
│        ││        ││ "Active" / "Required"  │
│        ││        ││ / "Send invite email"  │
│        ││        ││ flags use it.          │
└────────┘└────────┘└────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│ Task 7 — Manual browser verification                       │
│   Edit + create drawers in Chrome and Brave (the two       │
│   browsers in the screenshots). Confirm visual parity,     │
│   keyboard toggle, screen-reader label.                    │
└────────────────────────────────────────────────────────────┘
```

Critical edges:
- Tasks 2 and 3 depend only on Task 1.
- Tasks 4 and 5 depend on Task 2 (they're the first consumers of the
  wrapped form-friendly API).
- Task 6 (docs) can run in parallel with Tasks 4–5 once Task 2 is done.
- Task 7 is the final acceptance gate.

---

## 4. Vertical slices (one complete path per task)

Each task ships a fully working end-to-end change. No "scaffold now,
wire up later" — half-landed slices have been a problem here before.

### Task 1 — Install the primitive via the shadcn CLI

**Action:**

```bash
cd asima-frontend
npx shadcn@latest add checkbox
```

**Expected side-effects:**
- New file `src/components/ui/checkbox.tsx` (owned source — we may
  restyle it but the canonical shadcn shape stays).
- `@radix-ui/react-checkbox` added to `package.json` `dependencies`.
- `package-lock.json` updated.

**Acceptance criteria:**
- [ ] `src/components/ui/checkbox.tsx` exists and matches the current
      shadcn template (don't hand-edit until Task 2).
- [ ] `@radix-ui/react-checkbox` appears in `package.json`.
- [ ] `npm run build` succeeds (catches a peer-dep version drift on
      install).
- [ ] No other UI primitive package was added by mistake.

**Verification commands:**

```bash
cat src/components/ui/checkbox.tsx | head -5
grep '"@radix-ui/react-checkbox"' package.json
npm run build
```

---

### Task 2 — Form-friendly Field convention

The shadcn `Checkbox` exposes Radix's `CheckedState` API (which can be
`true | false | 'indeterminate'`). React Hook Form's `register()` wants
`boolean`. To keep call sites one line and the convention uniform,
either:

**Option A (preferred):** Use RHF's `Controller` inside a thin Layer-3
wrapper `LabeledCheckbox` at
`src/features/admin-users/components/labeled-checkbox.tsx` for the
admin-user forms, and promote it to `src/components/labeled-checkbox.tsx`
the second a third call site needs it.

```tsx
// labeled-checkbox.tsx (sketch — final code lives in the PR)
'use client';
import { Controller, type Control, type FieldPath, type FieldValues } from 'react-hook-form';
import { Checkbox } from '@/components/ui/checkbox';
import { cn } from '@/lib/cn';

type Props<T extends FieldValues> = {
  control: Control<T>;
  name: FieldPath<T>;
  label: string;
  description?: string;
  disabled?: boolean;
  className?: string;
};

export function LabeledCheckbox<T extends FieldValues>({
  control, name, label, description, disabled, className,
}: Props<T>) {
  return (
    <Controller
      control={control}
      name={name}
      render={({ field }) => (
        <label className={cn('flex items-start gap-2 text-sm text-neutral-800', className)}>
          <Checkbox
            checked={field.value as boolean}
            onCheckedChange={(v) => field.onChange(v === true)}
            disabled={disabled}
            aria-describedby={description ? `${String(name)}-desc` : undefined}
          />
          <span className="leading-5">
            {label}
            {description && (
              <span id={`${String(name)}-desc`} className="block text-xs text-neutral-500">
                {description}
              </span>
            )}
          </span>
        </label>
      )}
    />
  );
}
```

**Option B:** Skip the wrapper; teach each form to use `Controller`
inline. Cheaper now, more friction at every future call site.

**Decision:** Option A. The whole point of the user's request was
"create a reusable checkbox," not "land another raw Controller block
in every drawer."

**Acceptance criteria:**
- [ ] `labeled-checkbox.tsx` exists.
- [ ] It only imports from `@/components/ui/checkbox`, `@/lib/cn`, and
      `react-hook-form` — no new package dependencies.
- [ ] The `checked` API normalizes to `boolean` (no `'indeterminate'`
      leaks into the form value unless the caller opts in via a
      future prop — not in scope today).
- [ ] TypeScript inference works without `as` casts at the call site
      (generic on `<T extends FieldValues>`).

**Verification commands:**

```bash
npx tsc --noEmit
npm run lint
```

---

### Task 3 — Unit-test the primitive

Tests live in `tests/unit/components/checkbox.spec.tsx` (mirrors the
existing `tests/unit/components/require-permission.spec.tsx`).

Coverage targets:
- Renders unchecked by default.
- Clicking toggles checked state (controlled and uncontrolled).
- Spacebar toggles checked state when focused (Radix behavior, but we
  verify our wrapper didn't break it).
- `aria-label` / associated `<label>` is exposed to the accessibility
  tree.
- `disabled` prop prevents toggle on click AND on space.

**Acceptance criteria:**
- [ ] `npm run test -- checkbox` passes 5+ assertions.
- [ ] No `act()` warnings in console.

---

### Task 4 — Replace in `edit-user-drawer.tsx`

Replace lines 130–133 (current raw `<input type="checkbox">` + raw
`<label>`) with:

```tsx
<LabeledCheckbox
  control={form.control}
  name="is_active"
  label="Active"
  description="Inactive employees can't sign in."
/>
```

**Acceptance criteria:**
- [ ] No `<input type="checkbox">` remains in
      `edit-user-drawer.tsx`.
- [ ] Form still submits with `is_active: boolean` — verified by the
      existing admin-users tests passing AND a manual edit.
- [ ] Visual parity with the rest of the drawer (label color,
      spacing, focus ring).

**Verification commands:**

```bash
npm run test -- admin-users
npm run dev   # then manually open Edit drawer
```

---

### Task 5 — Replace in `create-user-drawer.tsx`

Same swap on the create surface. The create flow also defaults
`is_active: true` (see `create-user-drawer.tsx:50`), so verify the
default checked state lands.

**Acceptance criteria:**
- [ ] No raw checkbox input in `create-user-drawer.tsx`.
- [ ] Default check state is `true` on open.
- [ ] Creating a user with the box left checked persists
      `is_active: true`; unchecking and submitting persists
      `is_active: false` (e2e through the dev server).

---

### Task 6 — Document the pattern

Two doc updates so the next contributor doesn't reinvent this:

1. **`docs/universal-guidelines/frontend-stack.md`** — add Checkbox to the stack table row
   structure (it slots into "Component primitives (Layer 1)" via
   `@radix-ui/react-checkbox`, and "Owned primitives (Layer 2)" gains
   `checkbox.tsx`). One-line mention; the rules already cover it.
2. **`docs/universal-guidelines/module-architecture.md`** is backend-only — do NOT touch
   it. Frontend conventions live in `frontend-stack.md`.
3. If/when a third call site for `LabeledCheckbox` appears, promote
   the file from `src/features/admin-users/components/` to
   `src/components/labeled-checkbox.tsx`. Note this as a follow-up,
   don't pre-promote.

**Acceptance criteria:**
- [ ] `docs/universal-guidelines/frontend-stack.md` mentions Checkbox + the
      `@radix-ui/react-checkbox` Layer-1 package.
- [ ] Diff is small (a few lines), not a rewrite of the table.

---

### Task 7 — Manual browser verification

The user's screenshots compared Chrome (default OS look) vs Brave
(scaled-up native control). The point of the primitive is visual
parity — verify in both.

**Checklist:**
- [ ] Open Edit drawer in Chrome → Active checkbox renders with
      Tailwind styling (filled-blue when checked, outline when
      unchecked, focus ring on Tab).
- [ ] Open the same drawer in Brave → identical look.
- [ ] Toggle with mouse, with `Space`, with `Enter` (Enter should NOT
      toggle — Radix behavior is space-only; document if the user
      wanted Enter too).
- [ ] Screen reader (macOS VoiceOver via Cmd+F5) announces the label
      "Active" and the state ("checked" / "not checked").
- [ ] Tab order through the drawer: First name → Last name → Title →
      Role → Active → Cancel → Save changes.

---

## 5. Checkpoints (review gates)

Stop and get sign-off before crossing each line.

### Checkpoint A — after Task 1

Decision needed: did the shadcn install bring in any unexpected
peer dep or write to any unexpected file? `git status` should show
exactly:
- `M  package.json`
- `M  package-lock.json`
- `A  src/components/ui/checkbox.tsx`

If anything else moved, stop. Probably means `components.json` is
stale or the CLI version drifted.

### Checkpoint B — after Task 2

Decision needed: is the `LabeledCheckbox` API right BEFORE we sprinkle
it across two drawers? Specifically:
- Is the `description` prop pulling its weight, or should the caller
  pass arbitrary JSX as `children`?
- Do we want `indeterminate` support in v1, or save it for a
  follow-up?

Better to debate the API on one file than to refactor it across
several.

### Checkpoint C — after Tasks 4 + 5

Decision needed: does the visual result match the design intent the
user had when filing this work? Pause for a screenshot review before
moving on to docs and verification.

### Checkpoint D — before merge

Final gate: every box in Tasks 1–7 ticked, `npm run lint && npm run
test && npm run build` all green, and visual verification has been
captured (ideally with a before/after screenshot pair in the PR).

---

## 6. Out of scope

Calling these out so they don't slip in mid-PR:

- Adding **Radio / RadioGroup** primitives. Different shadcn component,
  warrants its own plan when a real use case appears.
- Adding **Switch** (the on/off toggle). Different shape from
  Checkbox; if we want a "Active" switch instead of a checkbox, that's
  a design decision that belongs upstream of this plan.
- Replacing the raw `<select>` for Role on the same drawer. Out of
  scope here — would be a separate shadcn `Select` plan and lands as
  its own slice.
- Theming / dark mode. The shadcn primitive already reads from the
  `neutral` base configured in `components.json`; dark mode is a
  cross-cutting effort and not gated by this plan.
- Indeterminate state. The Radix primitive supports it; our wrapper
  does not expose it in v1. Add when a real use case shows up.

---

## 7. Risks and mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| shadcn CLI version drift writes a different template than expected | Low | Pin to `shadcn@latest` (matches how Dialog / Sheet were added); review the diff at Checkpoint A. |
| RHF `Controller` boilerplate bleeds into call sites if `LabeledCheckbox` API is wrong | Medium | Checkpoint B exists for exactly this. Land the wrapper on ONE drawer first, get the shape right, then propagate. |
| Visual regression on the existing "Active" affordance (users used to the native browser look) | Low | Minor UI change; mitigated by the manual verification in Task 7 and the screenshot comparison at Checkpoint C. |
| Form value lands as `'indeterminate'` instead of `boolean` | Medium | Wrapper normalizes in `onCheckedChange`: `field.onChange(v === true)`. Covered by Task 3 unit tests. |

---

## 8. Done definition

This plan is "done" when:

1. Tasks 1–7 are all complete with their acceptance criteria ticked.
2. `npm run lint && npm run test && npm run build` are green on a
   clean checkout.
3. A PR description includes a before/after screenshot of the Edit
   drawer's Active control in at least Chrome.
4. The PR follows the repo's commit-message convention
   (`feat(frontend): …`) and is squashed to no more than 2 commits
   (one for the primitive + wrapper + tests, one for the drawer
   call-site replacements + docs) — mirroring how the shadcn drawer
   migration landed.
