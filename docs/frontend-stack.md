# Frontend stack — what we use and why

This is the authoritative answer to "what library does this come from?"
for the Asima frontend. If you're about to install a new package, check
this doc first — most needs already have an owner.

The frontend is **one codebase, one set of rules**. Don't mix two
libraries that solve the same problem (two modal libraries, two state
libraries, two icon sets). The whole point of writing this down is to
make the boundaries obvious so you don't have to relitigate them.

---

## The mental model

Every UI capability in the app sits on one of three layers. Knowing the
layer tells you where to look and what to install.

```
┌─────────────────────────────────────────────────────────┐
│  Layer 3 — App code                                     │
│  Your feature components: CreateUserDrawer, LoginForm…  │
└─────────────────────────────────────────────────────────┘
                          ↑ composes
┌─────────────────────────────────────────────────────────┐
│  Layer 2 — Owned primitives (src/components/ui/)        │
│  shadcn-pattern source: sheet.tsx, dialog.tsx, …        │
│  These are styled with Tailwind and wrap Layer 1.       │
└─────────────────────────────────────────────────────────┘
                          ↑ wraps
┌─────────────────────────────────────────────────────────┐
│  Layer 1 — Headless behavior (npm packages)             │
│  Radix UI primitives: @radix-ui/react-dialog, …         │
│  Provide a11y + focus trap + portals. No styling.       │
└─────────────────────────────────────────────────────────┘
```

A right-side Drawer in this app is therefore:

- **Layer 1**: `@radix-ui/react-dialog` (a generic modal primitive — same
  package powers centered dialogs *and* side sheets).
- **Layer 2**: `src/components/ui/sheet.tsx` — shadcn-pattern source,
  copied into our repo, owned by us.
- **Layer 3**: `src/features/admin-users/components/edit-user-drawer.tsx`
  — composes `<Sheet>` with a React Hook Form and TanStack Query
  mutation.

That layering also answers "what's the difference between Radix and
shadcn?" — they're not alternatives, they're stacked.

---

## What is Radix UI?

**Radix UI is an npm package family of headless React primitives.**

- "Headless" means: it ships behavior, not appearance. The Dialog
  primitive gives you portal mounting, focus trap, ESC-to-close, ARIA
  attributes, click-outside-to-close, animation hooks — but zero CSS.
  You decide what it looks like.
- It is installed: each primitive is a separate package
  (`@radix-ui/react-dialog`, `@radix-ui/react-popover`,
  `@radix-ui/react-dropdown-menu`, etc.). Today we only have
  `@radix-ui/react-dialog`. More land as we adopt them.
- Why it matters here: Radix is the part of the stack that makes the
  app *accessible by default*. Hand-rolling focus traps, scroll locks,
  and ARIA attributes is where most homegrown UIs fail accessibility.
- Compare to alternatives: Headless UI (Tailwind Labs) and React Aria
  are the closest siblings. We chose Radix because **shadcn/ui is
  built on Radix**, and we want one ecosystem.

You will almost never `import` Radix directly. You'll import a Layer-2
primitive that wraps it.

---

## What is shadcn/ui?

**shadcn/ui is NOT a library you install. It is a pattern + a CLI tool.**

This is the most-misunderstood thing on the stack, so read this twice:

- There is no `shadcn` package in `package.json`. There never will be.
- shadcn provides component **source code** that combines Radix
  primitives with Tailwind classes. You copy that source into your
  repo via `npx shadcn@latest add <component>`. From that moment on,
  you *own* the file.
- The CLI doesn't pull a runtime dependency. It writes a `.tsx` file
  to `src/components/ui/` and installs the matching Radix package as
  a peer dependency. Example: `npx shadcn@latest add sheet` writes
  `sheet.tsx` and `npm install`s `@radix-ui/react-dialog`.
- `components.json` at the repo root tells the CLI where to put files
  and what styling conventions to use (base color, CSS variables,
  `cn` alias). Don't edit hand-rolled primitives in
  `src/components/ui/` to add feature logic — extend by composition
  in feature components instead.

Why it matters here: shadcn lets us get accessible, well-designed,
animated primitives **without** taking on a third-party runtime
dependency for the UI layer. If shadcn disappeared tomorrow, our code
keeps working — because the code is in our repo.

**Rule:** all modal-layer / popover-layer / menu-layer primitives
(Sheet, Dialog, Dropdown, Select, Tooltip, Popover, Tabs, Toast — once
needed) come from shadcn. Don't hand-roll, don't bring in Headless UI
or Material as a parallel source.

---

## Where does Zustand fit?

Short answer: **Zustand is for client-only state shared across
components that's awkward to express via React Context or prop
drilling. We don't use it yet, and we shouldn't add it until we have a
concrete case for it.**

To pick the right tool, classify what kind of state you're storing:

| Kind of state | Example in Asima | Owner |
|---|---|---|
| **Server state** — anything that lives in Postgres, fetched via the API, can become stale | Employee list, role list, current user profile, pending approvals | **TanStack Query** (already in use) |
| **URL state** — anything a user should be able to bookmark or share | Current page, filter selections on the employees table, search query | **Next.js `useSearchParams` / `useRouter`** (right place — survives refresh, shareable) |
| **Identity / session** — who am I, what permissions do I have | `useAuth`, `usePermissions` | **React Context** (`auth-provider.tsx`, already in use) |
| **Component-local UI state** — open/closed, hovered, focused, draft input | Drawer `open` boolean on the employees page | **`useState`** (already in use) |
| **Cross-component client state** — shared by components that don't share a parent and aren't really server data | Currently: **none.** | **Zustand**, if and when it appears |

The "Zustand zone" is small on purpose. Most things that *feel* like
they need global client state actually belong to one of the rows above.
Before reaching for Zustand, ask:

1. Is this data that came from (or will go to) the backend? → TanStack
   Query, not Zustand. Don't duplicate cached server state into a
   client store.
2. Should it survive a refresh or be shareable via URL? → URL state via
   Next.js.
3. Is it identity-shaped (auth, theme, locale)? → Context.
4. Does only one subtree need it? → `useState` lifted to the lowest
   common ancestor.
5. None of the above, and prop-drilling has become painful across
   unrelated routes? → Zustand.

### When we'd actually adopt Zustand

A realistic Asima example: a multi-step "punch correction request"
form that spans three routes and needs to keep its draft between
navigations without round-tripping through the server. Today that
flow doesn't exist; when it does, Zustand becomes a sensible answer.
Until then, **don't pre-install it.**

### If we adopt it, where it goes in the stack

It's a Layer-3 concern (app code), not Layer 1 or 2. Suggested
convention:

- Package: `zustand` (no middleware unless we need persistence).
- Location: `src/features/<feature>/store.ts` for feature-scoped
  stores. Avoid one global `useAppStore` — feature-scoped stores age
  better.
- Naming: `useDraftCorrectionStore`, not `useStore`.
- Rule: a Zustand store **must not** hold a copy of server data. If
  you find yourself writing `setEmployees(…)` from inside an
  `onSuccess`, that's TanStack Query's job — back out.

This doc gets updated the day we install `zustand`. Until then,
mentioning it in code review is a flag, not a green light.

---

## The full frontend stack table

The same table from the SPEC, kept here for quick reference. If this
diverges from `asima-frontend/SPEC.md`, the SPEC wins.

| Layer | Tool | Role |
|---|---|---|
| Runtime | Node ≥ 20 | Dev + build runtime |
| Framework | Next.js 15 (App Router) | Routing, RSC boundary, build pipeline |
| UI library | React 19 (RC) | Component runtime |
| Language | TypeScript 5.6 | Type system |
| Styling | Tailwind CSS 3.4 | Utility-first CSS; design tokens via CSS vars in `globals.css` |
| Class helpers | `clsx` + `tailwind-merge` (wrapped as `cn` in `src/lib/cn.ts`) | Class composition + Tailwind-aware dedup |
| Animation utilities | `tailwindcss-animate` | `animate-in`, `slide-in-from-right`, `fade-in-0`, etc. |
| Component primitives (Layer 1) | **Radix UI** (`@radix-ui/react-*`) | Headless, accessible behavior |
| Owned primitives (Layer 2) | **shadcn/ui** pattern (no runtime package) | Copy-in source under `src/components/ui/`; configured by `components.json` |
| Variant helper | `class-variance-authority` (cva) | Typed style variants (used inside shadcn primitives) |
| Icons | `lucide-react` | One icon set across the app (shadcn default) |
| Server state | TanStack Query v5 | Caching, refetch, mutation invalidation |
| HTTP client | `fetch` + `src/lib/api-client.ts` | Thin wrapper, request-ID propagation, refresh interceptor |
| Forms | React Hook Form + `@hookform/resolvers` | Form state |
| Validation | Zod | Schema-first; same schemas validate wire payloads |
| Toasts | `sonner` | shadcn-integrated, accessible |
| Dates | `date-fns` | Date math (no moment, no dayjs) |
| Identity / auth state | React Context (`auth-provider.tsx`) | Who-am-I + permissions |
| Cross-route client state | **Zustand** — *not installed yet; see "Where does Zustand fit?" above* | Reserved slot |
| Testing | Vitest + React Testing Library + jsdom | Unit + component tests |
| Lint / format | ESLint (`eslint-config-next`) + Prettier (`prettier-plugin-tailwindcss`) | Code quality, class sorting |

---

## Hard rules — don't break these without an ADR

1. **One source per layer.** Don't introduce Headless UI, Material UI,
   Chakra, Mantine, Ant Design, or any other UI primitive library.
   shadcn (on top of Radix) is it.
2. **No CSS-in-JS.** No styled-components, emotion, vanilla-extract.
   Tailwind only. Inline `style={…}` for one-off computed values is
   fine (e.g. dynamic widths from a calculation).
3. **No second server-state library.** TanStack Query is canonical.
   Don't add SWR, Apollo, RTK Query.
4. **No global client-state store before a Zustand-shaped need
   exists.** Especially: don't add Redux, MobX, Recoil, Jotai. When
   Zustand becomes warranted, it follows the rules in the
   "Where does Zustand fit?" section.
5. **shadcn primitives live in `src/components/ui/` and stay close to
   their shadcn source.** Need a tweak? Wrap or compose at Layer 3,
   don't fork the primitive.
6. **One icon library.** `lucide-react`. No emoji as structural icons.

---

## Adding a new dependency — the checklist

Before `npm install <thing>`, answer these:

1. Which row of the stack table above does this fit into?
2. Is there already a tool in that row?
   - Yes → use that one. If you think it's the wrong tool, write an ADR
     in `docs/adr/` and get agreement before swapping.
   - No → continue.
3. Is it a UI primitive (modal, menu, popover, etc.)?
   - Yes → it must come via shadcn (`npx shadcn@latest add <name>`),
     not a direct install.
4. Does adding it change the rules above? → Write an ADR.

---

## Glossary

- **Headless component** — provides behavior + accessibility, no
  styling. Radix Dialog is headless.
- **Primitive** — a small, single-purpose UI building block (Dialog,
  Popover, Menu) as opposed to a feature-shaped composition
  (CreateUserDrawer).
- **Owned source** — code copied into our repo from a source like
  shadcn; we control it forever. Contrast with a runtime dependency,
  where the package owns the code and we get version updates.
- **Layer 1 / 2 / 3** — see the mental-model diagram at the top.
