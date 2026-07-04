# ADR 0002 — Refresh-token revocation via a server-side `refresh_tokens` store

- **Status:** Accepted (2026-07-04). Supersedes the v0 "stateless, no revocation"
  posture recorded in `asima-backend/CLAUDE.md`.
- **Date:** 2026-07-04
- **Deciders:** Asima core

## Context

v0 auth was **stateless**: `AuthService.signTokens` minted an access token (15m)
and a refresh token (7d) with no server-side record; `POST /auth/refresh` re-signed
a new pair; `POST /auth/logout` was a no-op. Consequences:

- **A leaked refresh token is valid for its full 7 days.** Nothing can invalidate
  it early — not logout, not rotation. This is the single security gap a review
  called out.
- **Rotation didn't actually rotate anything revocable.** The "old" refresh token
  kept working until natural expiry, so token theft wasn't contained.

`asima-backend/CLAUDE.md` deliberately put "a sessions / refresh-tokens table" out of
scope **"without an ADR."** This is that ADR.

## Decision

Introduce a **server-side refresh-token store** and make refresh + logout operate
against it.

1. **Storage: a Postgres `refresh_tokens` table.** Redis was rejected — it would add
   a new piece of infrastructure to a stack that is deliberately Postgres + MinIO
   only, for a low-write, low-read revocation ledger that a table serves fine. One
   row per issued refresh token.
2. **Identify each refresh token by a `jti`.** `signTokens` generates a
   `crypto.randomUUID()` and embeds it as the `jti` claim of the **refresh** token
   only (the access token is unchanged). A row `{ user_id, jti, expires_at,
   revoked_at }` is inserted on every issue (login and rotation).
3. **We store the `jti`, not the token.** The JWT signature already proves
   authenticity; the row is purely a **revocation ledger**. No secret material is
   persisted.
4. **Rotation invalidates the presented token, atomically.** `POST /auth/refresh`
   performs a conditional `UPDATE … SET revoked_at = now() WHERE jti = :jti AND
   revoked_at IS NULL`. If it affects 0 rows the token was already used or
   revoked → **401** (this also detects refresh-token reuse). On success a new
   `jti` row is issued. The `JwtRefreshStrategy` additionally rejects a
   revoked/expired `jti` at the guard (defense-in-depth).
5. **Logout revokes ALL of a user's refresh tokens (revoke-all / multi-device).**
   `POST /auth/logout` sets `revoked_at = now()` on every active row for
   `req.user.id`. Logging out kills every session/device.
6. **The access token is unchanged.** A revoked session's access token still works
   until its ≤15-minute natural expiry. A full access-token denylist stays out of
   scope; revocation targets the 7-day refresh window, which was the actual risk.

## Consequences

**Positive**

- The 7-day refresh window is now **revocable**: logout and rotation both invalidate
  server-side. A stolen refresh token can be cut off before natural expiry.
- Refresh-token **reuse is detected** (the atomic conditional revoke returns 0 rows).

**Costs / deviations**

- **Audit-column deviation (intentional):** `refresh_tokens` carries `created_at` /
  `updated_at` but omits `created_by` / `updated_by` / `deleted_by`. These rows are
  **infrastructure, not user-authored CRUD** — the acting subject *is* `user_id`.
  This mirrors the existing exception the codebase already makes for seed-managed
  `permissions`. Revocation is a hard `revoked_at` timestamp, not a soft delete.
- **Expired-row pruning is opportunistic** (a `deleteExpired(now)` repo method), not
  a scheduled job — acceptable for v0 volumes; a cron can be added later.
- **Residual access-token window:** up to 15 minutes after revocation. Documented and
  accepted.

## References

- `asima-backend/CLAUDE.md` → "Auth" and "Out of scope for v0" (updated by this ADR).
- `asima-backend/src/auth/` → `auth.service.ts`, `auth.controller.ts`,
  `strategies/jwt-refresh.strategy.ts`, `persistence/`.
- Migration: `*-CreateRefreshTokensTable.ts`.
