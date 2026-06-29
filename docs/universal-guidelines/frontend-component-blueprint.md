# Universal Guideline — Creating reusable frontend components

> **Purpose.** This file is **prompt context**. Attach it to any session
> where you (the model) will create, modify, or evaluate a frontend
> component in `asima-frontend`. It exists so the assistant does NOT
> reach for Material UI, Chakra, Ant Design, Headless UI, daisyUI, NextUI,
> Mantine, Bootstrap, Tailwind UI, Flowbite, Radix Themes, or any other
> UI source. There is exactly **one** UI source in this codebase, and
> it is described below.
>
> Read this before you read anything else about styling, components,
> dialogs, drawers, dropdowns, popovers, tooltips, menus, comboboxes,
> tabs, accordions, switches, sliders, toggles, checkboxes, radios,
> selects, date pickers, file uploaders, command palettes, alerts,
> banners, badges, avatars, breadcrumbs, pagination controls,
> progress bars, skeletons, cards, separators, scroll areas, or
> resizable panels.

---

## TL;DR — the hard answer

When you need any UI primitive:

1. **Check first:** does shadcn ship it? → `npx shadcn@latest add <name>`.
   The CLI writes owned source to `src/components/ui/<name>.tsx` and
   installs the matching `@radix-ui/react-<name>` package.
2. **If shadcn does NOT ship it:** STOP. Surface the gap to the user
   and ask for an ADR decision. Do not silently pick an alternative.
3. **Compose in Layer 3** (`src/features/<feature>/components/...` or
   `src/components/<name>.tsx` once shared). Do not edit
   `src/components/ui/*` to add feature logic.

The full reasoning, layering model, and stack table live in
[`docs/universal-guidelines/frontend-stack.md`](./frontend-stack.md). That file is
canonical; this file is the operational rulebook the assistant should
load **before** starting component work.

---

## The mental model (memorize this)

```
Layer 3  →  src/features/.../components/edit-user-drawer.tsx     (feature code)
              uses ↓
Layer 2  →  src/components/ui/sheet.tsx                          (owned shadcn source)
              wraps ↓
Layer 1  →  @radix-ui/react-dialog                               (npm package, headless)
```

- **Layer 1** is the only place where an npm UI package lives. Every
  Layer-1 package is from the `@radix-ui/react-*` family. Period.
- **Layer 2** is shadcn-pattern source files **owned by this repo**.
  They are added by `npx shadcn@latest add`, never `npm install`-ed,
  and they live in `src/components/ui/`.
- **Layer 3** is everything else — feature code, page-specific
  compositions, form-bound wrappers, decorated variants. This is where
  reusability lives at the application level.

---

## The unbreakable rules

These are restated from `docs/universal-guidelines/frontend-stack.md` §"Hard rules" for
prompt-context clarity. Violating any of these requires an ADR in
`asima-backend/docs/adr/` (yes, ADRs are recorded backend-side; frontend
ADRs go in the same folder) **before** the change lands.

1. **One source per layer.** No Material UI, Chakra, Ant Design,
   Headless UI (Tailwind Labs), Mantine, NextUI, daisyUI, Flowbite,
   Bootstrap, Radix Themes (the styled variant — we use *unstyled*
   Radix primitives only), or anything else.
2. **No CSS-in-JS.** Tailwind only. No styled-components, emotion,
   vanilla-extract, stitches, linaria. Inline `style={{...}}` for
   *computed* values (e.g. dynamic widths) is fine.
3. **No second server-state library.** TanStack Query is canonical.
   Don't add SWR, Apollo, RTK Query, urql.
4. **No global client-state store before a Zustand-shaped need exists**
   (and even then, follow the rules in `frontend-stack.md` §"Where
   does Zustand fit?"). Especially: no Redux, MobX, Recoil, Jotai.
5. **shadcn primitives stay close to the shadcn source.** Need a tweak?
   Wrap or compose at Layer 3. Don't fork `src/components/ui/dialog.tsx`
   to add a "loading" state — make a `LoadingDialog` wrapper in feature
   code instead.
6. **One icon library: `lucide-react`.** No `react-icons`, no
   `@heroicons/react`, no emoji as structural icons.
7. **One animation system: `tailwindcss-animate` utility classes.** No
   `framer-motion`, no `react-spring`, no GSAP unless a separate ADR
   authorizes it for a specific high-fidelity surface.
8. **One date library: `date-fns`.** No moment, no dayjs, no luxon.

---

## Decision flowchart — "I need a [thing]"

```
            ┌─────────────────────────────────────────┐
            │ Does it already exist at Layer 2 or 3?  │
            │ (check src/components/ui/ AND           │
            │  src/components/ AND src/features/)     │
            └─────────────┬───────────────────────────┘
                          │
                  yes ────┴──── no
                   │              │
                   ▼              ▼
        ┌──────────────────┐   ┌─────────────────────────────────┐
        │ USE IT. Compose. │   │ Does shadcn ship a primitive    │
        │ Do not duplicate.│   │ for it?                          │
        └──────────────────┘   │ (check https://ui.shadcn.com)    │
                               └──────────┬───────────────────────┘
                                          │
                                 yes ─────┴───── no
                                  │               │
                                  ▼               ▼
                   ┌──────────────────────┐  ┌──────────────────────┐
                   │ npx shadcn@latest    │  │ STOP. This is the    │
                   │ add <name>           │  │ ONLY case where you  │
                   │ Then compose at      │  │ ask the user for an  │
                   │ Layer 3.             │  │ ADR. See "Escape     │
                   └──────────────────────┘  │ hatch" below.        │
                                             └──────────────────────┘
```

If you are an LLM reading this and the user has not told you which row
you are on, **ask before installing anything**. Wrong-source installs
are expensive to back out.

---

## Standard recipe (shadcn primitive available)

This is how Dialog and Sheet landed. Mirror it exactly for any new
shadcn primitive.

1. **Add the primitive** (from `asima-frontend/`):

   ```bash
   npx shadcn@latest add <component>
   ```

   This writes `src/components/ui/<component>.tsx` and installs
   `@radix-ui/react-<component>` as a runtime dep. No other side
   effects should land — verify with `git status`.

2. **Do not hand-edit** `src/components/ui/<component>.tsx` to add
   business logic. The file is owned source, but it stays close to the
   shadcn template so we can re-run the CLI in the future without
   resolving conflicts.

3. **Build a form-bound or feature-bound wrapper at Layer 3** when the
   primitive needs to plug into React Hook Form, TanStack Query, or a
   feature-specific layout. Example wrapper file paths:
   - Feature-scoped wrapper first:
     `src/features/<feature>/components/<wrapper>.tsx`.
   - Promote to `src/components/<wrapper>.tsx` only when **two or more
     features** use it. Premature promotion produces dead generic code.

4. **Unit-test the wrapper** at
   `tests/unit/components/<name>.spec.tsx` or
   `tests/unit/features/<feature>/<name>.spec.tsx` depending on where
   it lives. Cover keyboard interaction explicitly — that's the part
   of Radix we rely on most.

5. **Replace existing raw HTML usage** in the same PR if any exists.
   No "we'll migrate the old `<select>` later" — half-landed
   migrations breed inconsistency.

6. **Add one line to** `docs/universal-guidelines/frontend-stack.md` if a new Layer-1
   package landed (so the stack table stays the canonical list).

---

## Escape hatch — when shadcn does NOT ship it

This is rare. As of this writing, every primitive we have needed has a
shadcn equivalent. If you genuinely cannot find one:

1. **Do not search the web for "best React date picker."** That is how
   Mantine ends up in `package.json`.
2. **Surface the gap to the user.** Explicitly: *"shadcn does not ship
   a `<Foo>` primitive. The options are: (a) build it on top of a
   Radix primitive ourselves, (b) bring in a third-party package
   `<X>`, (c) drop the requirement. Each option needs an ADR — which
   direction do you want to go?"*
3. **Default recommendation** is option (a) for anything that's
   "behavior + a11y" (use a Radix primitive directly if there is one)
   and option (c) for anything where the requirement is itself
   suspect.
4. **The ADR** goes in `asima-backend/docs/adr/` (frontend and backend
   share the ADR folder for now) and gets reviewed before the install.

Things that have historically tempted people to break the rule —
**none of these justify a new UI source**:

| Temptation | Correct answer |
|---|---|
| "Material has a nicer DatePicker" | shadcn ships `Calendar` (built on `react-day-picker`) and `Popover` — compose them. |
| "I want a fancy DataTable with sorting/filtering" | TanStack Table (headless) + shadcn `Table` markup. No DataTable libraries. |
| "Headless UI's Combobox is easier" | shadcn ships `Command` (built on `cmdk`) — it's the combobox. |
| "I want a slick toast" | We use `sonner` already (shadcn-integrated). No `react-hot-toast`, no `react-toastify`. |
| "I need a rich text editor" | This requires an ADR. Don't pick one unilaterally. |
| "I want drag-and-drop" | This requires an ADR. `dnd-kit` is the likely answer but only after we agree on the use case. |

---

## Reusability principles

**Three call sites is the promotion threshold.** A component lives where
it's used:

- **One feature uses it** → `src/features/<feature>/components/<name>.tsx`.
- **Two features use it** → still feature-local, but flag in the PR
  description that the next use case should promote.
- **Three+ features use it** → promote to `src/components/<name>.tsx`.

**Do not pre-generalize.** A component built for "any possible future
caller" reliably becomes harder to use than three small focused
components. Build for the use case in front of you, refactor at the
third repetition.

**Props shape — be specific over flexible.**
- Prefer a named prop (`description?: string`) over a generic
  `children: ReactNode` when the shape is known. Named props
  document the intent and type-check the layout.
- Prefer a `variant`/`size` discriminated union over a bag of boolean
  flags. (`class-variance-authority` aka `cva` is already in the
  stack for this exact purpose; see how shadcn primitives use it.)
- If a prop is required by every call site, make it required. Optional
  props with sensible defaults are fine, optional props with
  "everyone has to pass this anyway" defaults are noise.

**One file = one component family.** Sub-components (DialogHeader,
DialogFooter, etc.) live in the same file as their parent. Don't
explode a Dialog into seven files.

**Naming.**
- Layer 2 (shadcn): exact shadcn name. `Dialog`, `Sheet`, `Checkbox`,
  `Command`. Never `MyDialog`, never `AppCheckbox`.
- Layer 3 wrapper: prefix or suffix that conveys the wrapping concern.
  `LabeledCheckbox`, `ConfirmDialog`, `FormSheet`. Avoid generic
  `EnhancedX` or `CustomX` names.
- Feature compositions: name them for the thing they do, not the
  primitive they wrap. `CreateUserDrawer`, not `UserDialogContainer`.

**Already-promoted shared primitives — reuse, don't re-roll.** These cross-
feature pieces already exist; a new form / drawer / list MUST use them instead
of re-deriving the same logic locally (re-rolling re-introduces the duplication
the 2026-06-19 reuse pass removed):

- `@/components/form/field` — the `<Field label error htmlFor>` wrapper; the one
  source for a labeled field with inline error text.
- `@/components/form-drawer` — the Sheet/Dialog shell (title, body, submit/cancel
  footer, pending state). Use for new drawers unless the layout is genuinely bespoke.
- `@/components/pagination` + `@/lib/use-pagination` — the page cursor + prev/next
  controls for the `{ data, total, has_more }` list contract.
- `@/lib/api-error` — `errorMessage` / `firstFieldError` / `fieldErrors` for the
  backend `{ status, errors?, message? }` envelope. Never re-cast `err.body` inline.
- `@/lib/format` — the tz-aware render path (`formatDateInTz`, `formatTimeInTz`,
  `formatRelative`, …). Never call `date-fns.format()` / `toLocaleString()` directly.

---

## Forbidden imports (cheat sheet)

If you see any of these in a diff, reject the change:

```ts
// ❌ NEVER — would introduce a parallel UI source
import { Dialog } from '@headlessui/react';
import { Button, Box } from '@mui/material';
import { Modal } from '@chakra-ui/react';
import { Modal } from 'antd';
import { Modal } from '@mantine/core';
import { Button } from 'react-bootstrap';
import { Modal } from 'flowbite-react';

// ❌ NEVER — CSS-in-JS
import styled from 'styled-components';
import { css } from '@emotion/react';

// ❌ NEVER — parallel server state
import useSWR from 'swr';
import { useQuery } from '@apollo/client';

// ❌ NEVER — parallel global state
import { configureStore } from '@reduxjs/toolkit';
import { create } from 'jotai';

// ❌ NEVER — parallel icon set
import { HeartIcon } from '@heroicons/react/24/outline';
import { FaHeart } from 'react-icons/fa';

// ❌ NEVER — parallel animation engine (unless ADR'd)
import { motion } from 'framer-motion';
import { useSpring } from 'react-spring';

// ❌ NEVER — parallel date lib
import dayjs from 'dayjs';
import moment from 'moment';

// ✅ ALWAYS — the only correct sources
import * as DialogPrimitive from '@radix-ui/react-dialog';
import { Dialog, DialogContent, DialogHeader } from '@/components/ui/dialog';
import { useQuery } from '@tanstack/react-query';
import { Heart } from 'lucide-react';
import { format } from 'date-fns';
import { cn } from '@/lib/cn';
```

---

## Prompt-priming snippet (copy this into your first message)

When starting a session that involves frontend component work, paste
the following at the top of your message so the assistant cannot drift:

> The asima-frontend stack is fixed: Next.js 15 + React 19 + Tailwind 3.4 +
> TypeScript 5.6. UI primitives come from shadcn (owned source under
> `src/components/ui/`) which wraps Radix UI primitives
> (`@radix-ui/react-*`) — that is the **only** source of UI primitives.
> Forms: React Hook Form + Zod. Server state: TanStack Query v5. Icons:
> `lucide-react`. Dates: `date-fns`. Toasts: `sonner`. Class helper:
> `cn` from `@/lib/cn`.
>
> Do NOT suggest Material UI, Chakra, Ant Design, Headless UI, Mantine,
> NextUI, daisyUI, Flowbite, Bootstrap, react-icons, heroicons,
> framer-motion, styled-components, emotion, SWR, Apollo, Redux, Jotai,
> moment, or dayjs. If a primitive isn't covered by shadcn, STOP and ask
> me — don't pick a substitute.
>
> Full rules: `asima-parent/docs/universal-guidelines/frontend-component-blueprint.md`.

---

## Checklist — before opening a PR for a new component

- [ ] Did I check `src/components/ui/`, `src/components/`, and
      `src/features/*/components/` first? (Don't duplicate.)
- [ ] Did the primitive come from `npx shadcn@latest add ...`? (Not
      hand-written, not from another library.)
- [ ] Did I leave `src/components/ui/<primitive>.tsx` close to the
      shadcn template? (No feature logic in Layer 2.)
- [ ] Is my wrapper at the right scope — feature-local until 3+
      call sites?
- [ ] Did I unit-test the wrapper, including keyboard interaction?
- [ ] Did I replace EVERY existing raw HTML occurrence of the same
      affordance in this PR?
- [ ] Did I add a line to `docs/universal-guidelines/frontend-stack.md` if a new Layer-1
      Radix package landed?
- [ ] Are my imports clean — no entry in the "Forbidden imports"
      list above?
- [ ] Does `npm run lint && npm run test && npm run build` pass?

---

## When to update THIS file

Update this guideline when:

- A new shadcn primitive becomes part of the established pattern (add
  a row to the "Standard recipe" example list if non-obvious).
- An ADR authorizes an exception (record it briefly in the "Escape
  hatch" section so the assistant knows the carve-out exists).
- The stack itself changes (which should also update
  `docs/universal-guidelines/frontend-stack.md` — that file is canonical, this one
  references it).

Do **not** update it to add aspirational rules ("we should also avoid
X"). Aspirations belong in an ADR. This file documents what is true
about the codebase right now.
