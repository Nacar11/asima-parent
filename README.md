# asima

**asima** is an Ashima-inspired **Employee Time Management** system —
single-tenant (one company per deployment). It grows by feature:
identity → leave / approvals → schedules → time entries → workforce.

This is the **parent** repo: it holds the system-level brief and all
committed documentation. The application code lives in two sibling repos,
each with its own git `origin` and `main`.

## Repo layout

| Path | Repo | What |
|---|---|---|
| `asima-backend/` | [`Nacar11/asima-backend`](https://github.com/Nacar11/asima-backend) | NestJS + TypeORM + PostgreSQL API. Hexagonal per module. |
| `asima-frontend/` | [`Nacar11/asima-frontend`](https://github.com/Nacar11/asima-frontend) | Next.js (App Router) SPA. Feature-sliced. |
| `docs/` | this repo | All committed docs (see below). |
| `CLAUDE.md` | this repo | System-level brief across frontend + backend. |

Each sub-repo has its own `README.md` (runbook) and `CLAUDE.md`
(repo-specific rules) that load on top of this one.

## Documentation lives here

**All committed documentation for every repo lives in this parent under
`docs/`.** The sub-repos keep no `docs/` directory — they reference these
paths as `../docs/...`.

- `docs/universal-guidelines/` — cross-cutting guidelines (module +
  frontend architecture, migration conventions, frontend stack + component
  blueprint, CI/CD + quality gates, the DBML schema).
- `docs/plans/YYYY-MM-DD-*.md` — frozen plan snapshots (audit trail) for
  every repo's features.
- `docs/adr/` — architecture decision records.

The only doc-like files that stay inside a sub-repo are the **gitignored**
`tasks/plan.md` / `tasks/todo.md` working files — a private workspace, never
committed. See the "Plans and todos" section of `CLAUDE.md` for the full
convention.

## Getting started

Run the backend first, then the frontend — each lives in its own repo with
its own runbook (cloned as a sibling directory next to this one):

- **Backend:** [`asima-backend` README](https://github.com/Nacar11/asima-backend#readme)
  — Postgres + MinIO in Docker, API on the host at `:3000`.
- **Frontend:** [`asima-frontend` README](https://github.com/Nacar11/asima-frontend#readme)
  — Next.js SPA at `:3001`, talks to the backend's `/api/v1`.

## Where to look first

- [`CLAUDE.md`](./CLAUDE.md) — the system-level brief: cross-cutting
  terminology, the admin / self-service contract, API conventions.
- [`docs/universal-guidelines/`](./docs/universal-guidelines/) — the
  authoritative guidelines, one source per layer.
- Each sub-repo's `CLAUDE.md` — repo-specific rules (hexagonal layout on the
  backend, feature-sliced layering on the frontend).
