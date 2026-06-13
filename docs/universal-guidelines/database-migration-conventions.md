# Database migration conventions — one table, one CREATE migration

This is the authoritative answer to "I need to change a table's schema —
do I edit the CREATE migration or add an ALTER one?" for the Asima
backend. Read it before writing any file under
`asima-backend/src/database/migrations/`.

The short version: **while the project is in the development phase, each
table is defined by exactly one `Create<Table>Table` migration, and that
file holds the table's complete specification.** You do not add an
`Alter<Table>Table` migration to patch a table whose CREATE migration
has not shipped yet — you edit the CREATE migration directly.

This sits alongside, and refines, the migration-naming table in
`asima-backend/CLAUDE.md`. That table is correct; this doc says *when*
each row of it applies.

---

## The rule

1. **One CREATE migration per table, and it is the whole truth.** The
   `Create<Table>Table` migration must declare every column, default,
   `CHECK`, enum, index, unique constraint, and foreign key the table is
   meant to have. Reading that one file should tell you the table's full
   shape — no hunting through a trail of later ALTERs to reconstruct it.

2. **Do not ALTER a table whose CREATE migration is still unreleased.**
   If you realize the table needs another column, a different index, or
   an extra enum value, and that table's CREATE migration has never run
   anywhere but local/dev machines — **open the CREATE migration and add
   it there.** Delete any ALTER you were about to ship for it.

3. **Keep the migration set small.** Every extra file is another thing to
   review, order, and revert. A `Create` immediately followed by an
   `Alter` that fixes the same brand-new table is two files describing
   one decision. Collapse them into one.

The CREATE migration is the **standard**: it is the single, canonical
definition of the table for the whole project. Treat it the way you'd
treat the table's `CREATE TABLE` DDL in a hand-written schema file —
because that is exactly what it is.

---

## Why this is safe (and why it's better)

We are still in the development phase. No CREATE migration has shipped to
a real, data-bearing environment yet, so **the migration files are freely
editable.** Nothing downstream depends on their historical contents.

`npm run db:fresh` (`schema:drop` → `migration:run` → `seed`) drops the
entire public schema — including the `migrations` ledger — and replays
every migration from scratch. So:

- The net schema after `db:fresh` is **identical** whether a tweak lives
  in the CREATE migration or in a follow-up ALTER. Editing the CREATE
  costs nothing at apply time.
- A create-then-alter pair on a brand-new table preserves **no** history
  worth keeping — the table has never existed anywhere else. The ALTER is
  pure noise.
- One authoritative file per table is easier to read, review, and reason
  about than a table whose real shape is `CREATE + 3 ALTERs` scattered by
  timestamp.

---

## Worked example

Two changes landed against tables that were created in the same
unreleased batch. Both were originally written as separate ALTER
migrations; both were folded back into their CREATE migration and the
ALTER files deleted.

**Before — two files per table:**

```
1778084598000-CreateUsersTable.ts                       # plain UNIQUE(email)
1778200000000-AlterUsersTableAddEmailLowerUniqueIndex.ts # drops it, adds LOWER() partial index

1778100000000-CreateTimeEntriesTable.ts                 # enum ('manual','biometric','admin')
1778500000000-AlterTimeEntriesTableAddCorrectionSource.ts # ADD VALUE 'correction'
```

**After — one file per table, each holding the final shape:**

```
1778084598000-CreateUsersTable.ts
  └─ creates the LOWER("email") WHERE deleted_at IS NULL partial unique
     index directly; no plain UNIQUE(email) ever exists.

1778100000000-CreateTimeEntriesTable.ts
  └─ CREATE TYPE "time_source" AS ENUM
       ('manual','biometric','admin','correction')   -- complete from the start
```

The two `Alter…` files were removed. Any code comment that named a
deleted migration (e.g. in an entity file) was repointed at the CREATE
migration.

---

## When the rule flips back to ALTER

The "fold it into CREATE" behavior is scoped to the **development
phase** — specifically, to tables whose CREATE migration has **not yet
been applied to a real (non-local) environment.**

The moment a CREATE migration has shipped somewhere with data that we
can't `db:fresh` away — staging, production, a shared environment a
teammate has run against and seeded — **that migration is frozen.** From
then on, every change to that table is an **additive `Alter<Table>Table`
migration**, exactly as the `asima-backend/CLAUDE.md` migration-naming
table prescribes (`AlterUsersTableAddPhoneNumber`,
`AlterUsersTableDropTitle`, etc.). You never rewrite a migration that has
run against data you intend to keep — that's how schema and ledger drift,
and how other environments break.

So the two policies are one policy:

| Table's CREATE migration… | A schema tweak is… |
|---|---|
| Not yet released (dev only, `db:fresh`-able) | Edited **into the CREATE migration**. No ALTER file. |
| Released to a real/shared environment | A new **additive ALTER migration**. CREATE is frozen. |

When in doubt about whether a CREATE has "shipped," ask. If a teammate or
a deployed environment has run it against data they care about, treat it
as frozen.

---

## Still true regardless of phase

These rules from `asima-backend/CLAUDE.md` hold in both phases — folding
tweaks into a CREATE migration does **not** relax them:

- **One schema operation on one table per file.** A CREATE migration is
  still one table. Don't bundle two tables, and don't mix a CREATE for
  one table with an ALTER for another in the same file.
- **Class name matches filename** (sans timestamp).
- **Junction tables are their own table** with their own `Create…Table`
  migration, created after both parents so the FKs land in the same file.
- **Audit fields from day one** (`created_by`, `updated_by`,
  `deleted_by`, `deleted_at`) — they belong in the CREATE migration's
  column list, never retrofitted.
- **`down()` must reverse `up()` exactly.** When you add something to a
  CREATE migration's `up()`, update its `down()` to drop it.

---

## How to write a CREATE migration — the standard shape

Everything below is the convention the existing migrations already follow
(`CreateUsersTable`, `CreateRolesTable`, `CreatePermissionsTable`,
`CreateRolePermissionsTable`, `CreateTimeEntriesTable`,
`CreateApprovalChainsTable`, `CreateLeaveRequestsTable`, …). A new CREATE
migration is a copy of this shape, not a fresh invention. `migration:generate`
may be used to scaffold, but you **must** normalize its output to these
names and ordering before committing — TypeORM's auto-generated hashed index
names (`IDX_a1b2c3…`) and inline FKs are not what we ship.

### The skeleton

```ts
import { MigrationInterface, QueryRunner } from 'typeorm';

/**
 * `<table>` — one-line purpose, then any non-obvious invariants.
 * (Optional for trivial tables like roles/permissions; required when a
 * table has versioning, partial-unique invariants, snapshots, or
 * anything a reader couldn't infer from the columns. See
 * CreateApprovalChainsTable / CreateLeaveRequestsTable for the bar.)
 */
export class Create<Plural>Table<timestamp> implements MigrationInterface {
  name = 'Create<Plural>Table<timestamp>';

  public async up(queryRunner: QueryRunner): Promise<void> {
    // 1. CREATE TYPE … AS ENUM   (every enum the table uses, first)
    // 2. CREATE TABLE            (columns, then table-level constraints)
    // 3. CREATE [UNIQUE] INDEX   (plain, then partial/functional)
    // 4. ALTER TABLE … ADD CONSTRAINT … FOREIGN KEY   (FKs last)
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    // Exact reverse of up():
    // 4'. DROP the FKs   (reverse order)
    // 3'. DROP the indexes   (reverse order, schema-qualified)
    // 2'. DROP TABLE
    // 1'. DROP TYPE   (reverse order)
  }
}
```

### File and class naming

- **Filename:** `<unixMillisTimestamp>-Create<Plural>Table.ts`, e.g.
  `1778084598000-CreateUsersTable.ts`. Lower timestamp runs first.
  Hand-authored migrations use **round, spaced-out** timestamps
  (`1778100000000`, `1778200000000`, `1778300000000`) so there's room to
  slot a new table between two existing ones without renumbering.
- **Class name = the `Create<Plural>Table<timestamp>` portion of the
  filename**, and the `name` property is the **exact same string**. TypeORM
  matches the ledger on `name`; if it drifts from the class name, reverts and
  re-runs misbehave.
- Table name is **plural, snake_case** (`leave_requests`). Junction tables
  are `<table_a>_<table_b>` (`role_permissions`) and get their own
  `Create…Table` migration after both parents exist.

### up() — operation order is fixed

1. **Enums first.** `CREATE TYPE "leave_type" AS ENUM ('vacation', …)` before
   the `CREATE TABLE` that references it. Enum type name is **singular,
   snake_case**, unquoted-friendly (`time_source`, `leave_request_status`,
   `decision_path`).
2. **`CREATE TABLE`** — columns top to bottom, then **table-level
   constraints** (`PK_`, `UQ_`, `CHK_`) inside the same statement. Put
   `PRIMARY KEY` first among the constraints, then `CHECK`s, then table-level
   `UNIQUE`s (matches `CreateApprovalChainsTable`).
3. **Indexes** as separate `CREATE INDEX` / `CREATE UNIQUE INDEX` statements
   — never inline in the table body. Plain indexes first, then any
   partial/functional indexes (each preceded by a `/* … */` comment
   explaining the invariant it enforces).
4. **Foreign keys last**, each via `ALTER TABLE … ADD CONSTRAINT … FOREIGN
   KEY`. Never declare FKs inline in `CREATE TABLE` — they go after the table
   (and after the referenced table's own migration has run).

### down() is up() played backwards

Every object `up()` creates, `down()` drops, in **reverse creation order**:
FKs → indexes → table → enum types. Two details the existing files are
consistent about:

- **Indexes are schema-qualified on drop:** `DROP INDEX "public"."IDX_…"`.
  (They are *not* schema-qualified on create.)
- **Enum types are dropped after the table**, since the table depends on
  them. `DROP TYPE "leave_type"` comes after `DROP TABLE "leave_requests"`.

### Identifier naming — these prefixes are load-bearing

| Object | Pattern | Example |
|---|---|---|
| Primary key | `PK_<table>` | `PK_users` |
| Foreign key | `FK_<table>_<column>` | `FK_leave_requests_employee_id` |
| Unique constraint | `UQ_<table>_<cols>` | `UQ_approval_chains_employee_step_effective` |
| Check constraint | `CHK_<table>_<what>` | `CHK_time_entries_time_out_after_in` |
| Index | `IDX_<table>_<cols>` | `IDX_leave_requests_employee_status` |

Special-purpose partial/functional indexes may use a descriptive
`<table>_<what>_uq` name when that reads better than the `IDX_` form
(`users_email_lower_uq`, `IDX_approval_chains_active_step_uq`) — but keep the
table prefix so the index is greppable to its table.

All SQL identifiers are **double-quoted**; all SQL keywords are **UPPERCASE**.

### Column standards

- **PK:** `"id" SERIAL NOT NULL` + `CONSTRAINT "PK_<table>" PRIMARY KEY ("id")`.
  (Junction tables use a composite PK instead — `PK_role_permissions
  PRIMARY KEY ("role_id", "permission_id")` — and have no `id`.)
- **Strings:** `character varying(N)` with an explicit length (`(50)`,
  `(100)`, `(255)`, `(500)`). No bare `text` / `varchar`.
- **Timestamps:** always `TIMESTAMP WITH TIME ZONE`. `created_at` / `updated_at`
  are `NOT NULL DEFAULT now()`; event timestamps that may be absent
  (`decided_at`, `deleted_at`, `last_login_at`) are nullable, no default.
- **Dates without time:** `date` (`work_date`, `start_date`).
- **Integers / FKs:** `integer`. FK columns that are required are `NOT NULL`;
  optional relationships are nullable.
- **Booleans:** `boolean NOT NULL DEFAULT false` (or `true`).
- **Audit columns on every business table, from day one** (the order they
  appear in the existing tables):

  ```
  "created_by" integer,
  "updated_by" integer,
  "deleted_by" integer,
  "created_at" TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
  "updated_at" TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
  "deleted_at" TIMESTAMP WITH TIME ZONE
  ```

  `*_by` columns are nullable `integer` FKs to `users` (often left without an
  explicit FK constraint to avoid a cycle on `users` itself — match the
  neighboring table).

- **Exceptions to audit columns** (and they are the *only* exceptions):
  - **Junction tables** (`role_permissions`) carry just their FK columns and
    the composite PK — no audit, no soft-delete.
  - **Seed-managed config** (`permissions`) carries only `created_at` /
    `updated_at` — no `*_by`, no `deleted_at`.
  - **Logical-end tables** (`approval_chains`) replace soft-delete with an
    `ended_at` timestamp; they keep `created_by` / `updated_by` but omit
    `deleted_by` / `deleted_at`.

  If a new table skips audit columns, it must be one of these shapes — say
  why in the class JSDoc.

### Enums

- `CREATE TYPE "<singular_snake>" AS ENUM ('a', 'b', 'c')`. Keep the value
  list **narrow and final** — an enum you'll need to widen later still gets
  its full value set now while the CREATE migration is unreleased (that's the
  whole point of this doc: add the value to the CREATE, don't ship
  `ALTER TYPE … ADD VALUE`).
- Column references the type by name: `"status" "leave_request_status" NOT
  NULL DEFAULT 'pending_l1'`.
- `down()` drops every type the migration created, after the table.

### Foreign keys

- Added **after** the table, via `ALTER TABLE … ADD CONSTRAINT "FK_…"
  FOREIGN KEY (…) REFERENCES "<parent>"("id")`.
- **Referential action by relationship type:**
  - Protecting real data (a user, an employee) → `ON DELETE RESTRICT ON
    UPDATE NO ACTION`. This is the default for almost every FK.
  - A pure junction row whose existence is meaningless without both parents
    (`role_permissions`) → `ON DELETE CASCADE ON UPDATE CASCADE`.
- **Index every FK column** you'll join or filter on, with its own
  `IDX_<table>_<column>` (see `IDX_users_role_id`,
  `IDX_role_permissions_role_id`).
- **Many FKs to the same parent** → DRY them with a `for…of` over
  `[name, column]` tuples in `up()`, and loop the names in **reverse** in
  `down()` (see `CreateLeaveRequestsTable`'s five FKs to `users`).

### Partial / functional indexes — the invariant tools

Several tables enforce "at most one active row" or case-insensitive,
soft-delete-aware uniqueness at the **DB level** with a partial unique index,
because service-layer checks have a concurrent-write race window. When you add
one:

- Precede it with a `/* … */` comment naming the invariant and why the index
  (not just the service) is the source of truth.
- Examples to mirror: `IDX_time_entries_one_open_per_employee`
  (`WHERE status = 'open' AND deleted_at IS NULL`),
  `IDX_approval_chains_active_step_uq` (`WHERE ended_at IS NULL`),
  `users_email_lower_uq` (`ON users (LOWER("email")) WHERE deleted_at IS NULL`).

### Comments

- **Class-level JSDoc** for the table's rationale, invariants, and links to the
  ADR / plan that justify it.
- **Inline `/* … */`** immediately above any index or constraint whose purpose
  isn't obvious from its name.
- Don't comment the boring parts — a plain `IDX_users_role_id` needs no note.

### SQL formatting

- Multi-line statements (`CREATE TABLE`, multi-clause `CREATE INDEX … WHERE`,
  `ALTER TABLE … ADD CONSTRAINT`) use a template literal indented two spaces
  inside the `queryRunner.query(\` … \`)`.
- Short single statements stay on one line; if Prettier wraps the call, the
  SQL string goes on its own line. Don't fight the formatter.

---

## Checklist before you add a migration file

- [ ] Is this change to a table whose CREATE migration has **not** shipped
      to a real environment? → **Edit the CREATE migration.** Don't create
      a new file.
- [ ] Has that CREATE migration shipped to staging/prod/a shared env? →
      Write an additive `Alter<Table>Table…` migration; leave CREATE
      untouched.
- [ ] If editing a CREATE migration, did you also update its `down()` and
      repoint any code comments that named a now-deleted ALTER?

Coding standard (every CREATE migration):

- [ ] One table per file; class name **and** `name` property both equal the
      `Create<Plural>Table<timestamp>` string; table name plural snake_case.
- [ ] `up()` order: enums → `CREATE TABLE` (PK → CHECK → UNIQUE inside) →
      indexes (plain, then partial) → FKs via `ALTER TABLE ADD CONSTRAINT`.
- [ ] `down()` is the exact reverse: FKs → indexes (`DROP INDEX
      "public"."IDX_…"`) → `DROP TABLE` → `DROP TYPE`.
- [ ] Identifiers follow `PK_/FK_/UQ_/CHK_/IDX_<table>_…`; SQL keywords
      UPPERCASE; identifiers double-quoted.
- [ ] Audit columns present (`created_by/updated_by/deleted_by`,
      `created_at/updated_at` default `now()`, `deleted_at`) — unless the
      table is a junction / seed-config / logical-end table, justified in the
      class JSDoc.
- [ ] Types/lengths match the standard (`character varying(N)`,
      `TIMESTAMP WITH TIME ZONE`, `date`, `integer`, `boolean … DEFAULT …`).
- [ ] FK actions correct (`RESTRICT/NO ACTION` for data, `CASCADE` for
      junctions); every FK column indexed.
- [ ] Any partial/functional index has a `/* … */` comment naming the
      invariant it guards.
