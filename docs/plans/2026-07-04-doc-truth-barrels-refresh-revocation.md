# Plan: Doc-truth alignment · Barrel-boundary enforcement · Refresh-token revocation

_Snapshot date: 2026-07-04. Frozen audit copy — the working checklist lives in
the gitignored `asima-parent/tasks/`._

## Context — why this change

An external review of Asima praised the architecture but flagged one dominant
theme: **docs-vs-reality drift** in the governance layer, plus one real piece of
security debt. Verified against the repos:

- `docs/adr/` **did not exist**, yet `adr/0001-roles-and-approval-design.md` was
  cited as load-bearing in three places: `asima-parent/CLAUDE.md:27`,
  `asima-backend/CLAUDE.md`, and inside `CreateApprovalChainsTable` migration.
- The frontend barrel-import rule was **fiction**: `0` `index.ts` files across all
  12 slices, no ESLint rule, and **76 deep cross-slice imports** across 9 slices —
  while `frontend-architecture.md` §5 *claimed the migration complete*.
- `asima-parent/CLAUDE.md` repo-layout still said `asima-frontend/ # not yet
  scaffolded`; it is fully scaffolded (12 slices).
- Refresh tokens are stateless: a leaked refresh token stays valid its full 7-day
  life; `logout()` is a no-op. `asima-backend/CLAUDE.md` forbids a sessions/
  refresh-tokens table "without an ADR."

Goal: make the governance repo tell the truth, turn one fake rule into a real
CI-enforced one, and close the 7-day refresh-token window.

## Locked decisions (from clarification)

- **Sessions:** revoke-ALL of a user's refresh tokens on logout (multi-device).
- **Store:** Postgres `refresh_tokens` table (Redis rejected — no new infra in v0).
- **Barrel rule:** KEEP and ENFORCE (implement, not retire).
- **Unified bring-up:** DROPPED. asima-parent stays documentation/governance only.
- Access token behavior unchanged (≤15-min natural expiry). Revocation targets the
  7-day refresh window only; full access-token denylist is out of scope.

## Slices (each commits to its own repo's main)

### Slice 1 — Doc-truth alignment (asima-parent)
- **1.1** Fix the stale "frontend not scaffolded" claim in `asima-parent/CLAUDE.md`.
- **1.2** Write `docs/adr/0001-roles-and-approval-design.md` (Nygard) capturing
  Role/Title/Approval-chain orthogonality **and** the role-catalog evolution
  (v0 `SUPER_ADMIN`/`ADMIN`/`EMPLOYEE` → +`HR_ADMIN`/`PROJECT_MANAGER`/
  `TECHNICAL_DIRECTOR`). Every `adr/0001` pointer must resolve.

### Slice 2 — Enforce the barrel boundary (asima-frontend)
76 deep cross-slice imports today; heaviest consumers `admin-users` (17),
`approvals` (16), `admin-schedule` (10), `time-correction` (9).
- **2.1** Add `src/features/<slice>/index.ts` for all 12 slices (public surface;
  additive; `export type` for type-only to avoid barrel value-cycles).
- **2.2** Rewrite the 76 deep imports to `@/features/<slice>`, batched by consumer;
  convert same-slice absolute deep imports to relative. Zero behavior change.
- **2.3** Add `no-restricted-imports` (`@/features/*/*`, `error`) to `.eslintrc.json`;
  prove a deep import fails lint. `frontend-architecture.md` §5 becomes TRUE.

### Slice 3 — Refresh-token revocation (asima-backend)
Auth today: `signTokens()` mints stateless access+refresh JWTs (no `jti`);
`refresh()` re-signs; `logout()` is a no-op.
- **3.0** Write `docs/adr/0002-refresh-token-revocation.md`; update
  `asima-backend/CLAUDE.md` (out-of-scope + Auth) to cite it.
- **3.1** `CreateRefreshTokensTable` migration (id, user_id FK CASCADE, jti uuid,
  expires_at, revoked_at, timestamps; UNIQUE(jti), INDEX(user_id, revoked_at)).
- **3.2** Persistence (entity/mapper/Base+concrete repo/binding) + `RefreshTokenService`
  (issue/isActive/rotate/revokeAllForUser/pruneExpired) under `src/auth/`; add `jti`
  to the refresh payload. DDD-lite (infra, not a rich aggregate) but keep the port.
- **3.3** `JwtRefreshStrategy` verifies `isActive(jti)` + attaches jti; `refresh()`
  rotates (revoke old + issue new); `logout()` `revokeAllForUser`. Update comments.
- **3.4** Unit (`RefreshTokenService.spec`, update `auth.service.spec`) + e2e
  (rotated & post-logout refresh → 401). Run with CI-parity env.

## Risks & mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Barrel-to-barrel value cycles (`admin-users` ↔ `admin-approvers`) → TDZ | Med | `export type` for type-only; else expose narrowly/`components/`; `tsc` catches |
| `no-restricted-imports @/features/*/*` also catches same-slice absolute imports | Med | 2.2 converts same-slice absolute deep imports to relative before flipping to `error` |
| Refresh e2e flakes without full env (Postgres+MinIO+`STORAGE_*`, CI seed pwd) | Med | Run e2e with CI-parity env |
| Refresh rotation race (two concurrent refreshes) | Low | UNIQUE(`jti`) + revoked_at check; acceptable for v0 |
| Scope creep into access-token denylist | Low | Out of scope; access token keeps ≤15-min expiry |

## Verification (per slice)
- **Slice 1:** `grep -rn "adr/0001\|not yet scaffolded" asima-parent asima-backend`.
- **Slice 2:** cross-slice deep-import inventory returns 0; lint/build/test green; a
  deliberate deep import fails lint.
- **Slice 3:** e2e shows rotated & post-logout refresh → 401; test + build green.
