# Profile page input mirroring (demo) — design

**Date:** 2026-07-16
**Scope:** asima-frontend only, `/employee/profile` page
**Status:** Demo behavior, intended to be easy to remove later

## Context

For demonstration purposes, the employee **My profile** page should mirror
typed input across every input field on the page: whenever the user types in
any field, all five fields (First name, Last name, Current password, New
password, Confirm new password) adopt that same value. This follows the
earlier demo change that gave all five inputs the placeholder "Admin".

Clarified requirements:

1. **Live mirror from any field.** Every keystroke in any field propagates to
   all fields; the most recently typed field wins.
2. **All five fields, both cards.** The Identity card (2 fields) and Password
   card (3 fields) share one mirrored value, even though each card is its own
   component with its own `react-hook-form` instance.
3. **Real form state.** Mirroring writes through react-hook-form
   (`setValue` with `shouldDirty` + `shouldValidate`), so buttons enable,
   validation messages react, and submits send the mirrored values.

## Design

Lift a tiny "mirror" state to the page and pass it into both forms as
optional props (chosen over a context provider — YAGNI for two consumers —
and over raw DOM `.value` syncing, which would bypass react-hook-form state
and violate requirement 3).

### Files

| File | Change |
|---|---|
| `src/features/profile/mirror.ts` (new) | `export type MirrorEvent = { value: string; source: string }` |
| `src/app/(app)/employee/profile/page.tsx` | `useState<MirrorEvent \| null>`; pass `mirror` + `onMirrorInput` to both forms |
| `src/features/profile/components/profile-form.tsx` | Optional `mirror` / `onMirrorInput` props; broadcast on change; apply mirror to `first_name`, `last_name` |
| `src/features/profile/components/password-change-form.tsx` | Same for `current_password`, `new_password`, `confirm_password` |
| `tests/unit/features/profile/input-mirroring.spec.tsx` (new) | RTL spec: typing in one field mirrors to all five |

### Mechanics

- Each input broadcasts via `register(name, { onChange })` →
  `onMirrorInput({ value, source: name })`. A fresh object per keystroke means
  the receiving `useEffect`s re-run even when the string value repeats.
- Each form applies an incoming `MirrorEvent` to all of its fields **except
  `source`** (skipping the field being typed avoids caret jumps), using
  `form.setValue(field, value, { shouldDirty: true, shouldValidate: true })`.
- `setValue` does not fire DOM change handlers, so there is no feedback loop.
- Props are optional; forms without them (if reused elsewhere) behave as
  before.

### Behavior notes

- Prefilled name values stay until the first keystroke anywhere on the page.
- Password fields render the mirrored value masked — expected.
- Confirm-password always matches new-password (identical values), so the
  match rule passes; the strength rule validates whatever was typed.
- After a successful profile save, the existing refetch-reset effect restores
  server values in the Identity card until the next keystroke. Acceptable for
  a demo.

## Verification

- `npm run typecheck && npm run lint && npm run build && npm test` in
  `asima-frontend` (pre-push hook re-runs the same gate).
- New spec passes: type "Admin1!" into First name → all five inputs hold it;
  type into a password field → name fields update too.
- Manual: `/employee/profile`, type in any field, watch all five mirror.
