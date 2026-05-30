# Module Architecture — Guide for Creating a New Backend Module

This is the **authoritative blueprint** for adding any new feature module to
`asima-backend`. Every existing module (`users`, `roles`, `time-entries`,
`work-schedules`, `permissions`) follows the shape below; new ones must
mirror it.

The older `asima-backend/docs/module-architecture.md` was written before the
first modules landed and describes a richer domain-model style we did not
adopt. This document supersedes it and reflects how modules are actually
built in this repo today.

> **TL;DR — when creating a new module called `<feature>` (plural,
> kebab-case path; singular PascalCase class name):**
> 1. Scaffold the file tree in §2.
> 2. Build outside-in but ship a layered slice: domain → persistence →
>    service → controller(s) → module. A half-landed slice is worse than
>    no change.
> 3. Audience matters at the **edge** (DTO + controller), never at the
>    service. If the resource has both admin and self-service surfaces,
>    create `dto/admin/`, `dto/me/`, `admin-<feature>.controller.ts`,
>    `me-<feature>.controller.ts` — and ONE service / ONE repo.
> 4. Run the §11 checklist before opening a PR.

---

## 1. The architectural onion

Dependencies point inwards. Domain knows nothing about the database;
controllers know nothing about TypeORM.

```
┌───────────────────────────────────────────────────────────┐
│ Interface Layer (HTTP)                                    │
│   admin-<feature>.controller.ts                           │
│   me-<feature>.controller.ts        (when applicable)     │
└───────────────────────┬───────────────────────────────────┘
                        │ depends on
┌───────────────────────▼───────────────────────────────────┐
│ Application Layer (Use cases)                             │
│   <feature>.service.ts                                    │
│   dto/admin/*.dto.ts    dto/me/*.dto.ts                   │
└───────────────────────┬───────────────────────────────────┘
                        │ depends on (port, not impl)
┌───────────────────────▼───────────────────────────────────┐
│ Domain Layer (Pure TS — no @nestjs/* runtime, no typeorm) │
│   <feature>.ts                  ← data class              │
│   <feature>-search-criteria.ts  ← filter shape            │
│   <feature>-inputs.ts           ← service↔repo types      │
│   find-all-<feature>.ts         ← PaginatedResponse alias │
└───────────────────────▲───────────────────────────────────┘
                        │ implements port
┌───────────────────────┴───────────────────────────────────┐
│ Infrastructure Layer (Persistence)                        │
│   persistence/base-<feature>.repository.ts  ← ABSTRACT    │
│   persistence/repositories/<feature>.repository.ts        │
│   persistence/entities/<feature>.entity.ts                │
│   persistence/mappers/<feature>.mapper.ts                 │
│   persistence/persistence.module.ts (binds Base→Concrete) │
└───────────────────────────────────────────────────────────┘
```

The single rule that holds all of this together: **the service depends on
the abstract `Base<Feature>Repository`** (the port), not on the concrete
`Repository`. Tests mock the abstract — that's the whole point of the
port pattern.

---

## 2. Required file layout

```
src/<feature>/
├── domain/                                   # PURE TS — no @nestjs/* runtime, no typeorm.
│   ├── <feature>.ts                          # Domain data class (anemic; data only).
│   ├── <feature>-search-criteria.ts          # Filter shape consumed by repo.findAll().
│   ├── <feature>-inputs.ts                   # CreateInput / UpdatePatch / *Persistence types.
│   └── find-all-<feature>.ts                 # PaginatedResponse<Foo> alias.
├── persistence/
│   ├── base-<feature>.repository.ts          # ABSTRACT CLASS — the port the service depends on.
│   ├── entities/
│   │   └── <feature>.entity.ts               # TypeORM @Entity. Extends EntityHelper.
│   ├── mappers/
│   │   └── <feature>.mapper.ts               # toDomain / toPersistence. Static methods.
│   ├── repositories/
│   │   └── <feature>.repository.ts           # Concrete impl. Extends Base*Repository.
│   └── persistence.module.ts                 # TypeOrmModule.forFeature + Base→Concrete binding.
├── dto/                                      # class-validator inputs. Split by audience.
│   ├── admin/                                # Wide field set. Used by admin-* controllers only.
│   │   ├── create-<feature>.dto.ts
│   │   ├── update-<feature>.dto.ts
│   │   ├── query-<feature>.dto.ts
│   │   └── reset-<feature>-password.dto.ts   # (only if the resource has credentials)
│   └── me/                                   # Narrow field set. Used by me-* controllers only.
│       ├── update-me.dto.ts                  # (only the fields users may change themselves)
│       └── change-my-password.dto.ts         # (only if applicable)
├── controllers/
│   ├── admin-<feature>.controller.ts         # /admin/<feature>. @Permissions gated.
│   └── me-<feature>.controller.ts            # /<feature>/me OR /users/me/<sub>. JWT-only.
├── <feature>.service.ts                      # Single service used by both controllers.
├── <feature>.module.ts                       # Composes everything.
├── <feature>.constants.ts                    # Stable enums, protected names, regexes (optional).
└── <feature>.service.spec.ts                 # Unit tests, mocking Base*Repository.
```

Skip `dto/me/`, `me-*.controller.ts`, and `<feature>-inputs.ts` only
when the resource truly has **no self-service surface and no credential
handling** (rare — `roles` is the only current example, because employees
never manage roles themselves).

Path alias: use `@/` (resolves to `src/`). Never write `../../../`.

---

## 3. Layer-by-layer rules and templates

### 3.1 Domain layer

**Hard rules:**

1. Zero `@nestjs/*` runtime imports. The only allowed NestJS imports are
   `@nestjs/swagger` decorators (`@ApiProperty`, `@ApiPropertyOptional`) —
   they're stripped at runtime, so they're free.
2. Zero `typeorm` imports. No `@Entity()`, no `@Column()`, no
   `@PrimaryGeneratedColumn()` on a domain class — that breaks every
   service test that mocks the repository.
3. Definite-assignment (`!`) on every field. The mapper populates the
   class — TypeScript can't see those assignments, so
   `strictPropertyInitialization` flags every field whose type doesn't
   already include `undefined`. `field: string | null` STILL needs `!`
   (null ≠ undefined).
4. **Anemic by design.** No constructor validation, no behaviors. Domain
   classes are data containers. Invariants and business rules live in
   the **service**. (This is a deliberate departure from the older
   "rich domain" blueprint.)
5. Snake_case field names: `created_at`, `is_active`, `last_login_at`.
   These match the DB columns AND the JSON payload — no translation layer
   at any boundary.

**`domain/<feature>.ts` template:**

```ts
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';

/**
 * <Feature> domain class.
 *
 * Pure TS — no @nestjs/* runtime or typeorm imports. `@nestjs/swagger`
 * decorators are runtime-stripped, so they're allowed.
 */
export class <Feature> {
  @ApiProperty({ example: 1 })
  id!: number;

  // ... business fields ...

  @ApiPropertyOptional({ example: 1, nullable: true })
  created_by!: number | null;

  @ApiPropertyOptional({ example: 1, nullable: true })
  updated_by!: number | null;

  @ApiPropertyOptional({ example: null, nullable: true })
  deleted_by!: number | null;

  @ApiProperty({ type: String, format: 'date-time' })
  created_at!: Date;

  @ApiProperty({ type: String, format: 'date-time' })
  updated_at!: Date;

  @ApiPropertyOptional({ type: String, format: 'date-time', nullable: true })
  deleted_at!: Date | null;
}
```

**`domain/<feature>-search-criteria.ts` template:**

```ts
export type <Feature>SearchCriteria = {
  // domain-specific filters …
  search?: string;
  is_active?: boolean;
  includeDeleted?: boolean;   // default false; admin opt-in
  page?: number;
  limit?: number;
};
```

This is the "systemized filter" — a clean shape the service hands to the
repo, and the repo translates into messy QueryBuilder calls. Adding a new
filter is two lines (criteria field + a `qb.andWhere` in the repo).

**`domain/find-all-<feature>.ts`:**

```ts
import { <Feature> } from './<feature>';
import { PaginatedResponse } from '@/utils/types/paginated-response.type';

export type FindAll<Feature> = PaginatedResponse<<Feature>>;
```

Always returns `{ data, total, page, limit, has_more }`.

**`domain/<feature>-inputs.ts` (create when the service needs to massage
inputs before they reach the repo — most modules do):**

```ts
/**
 * Service-facing input types.
 *
 * `Create<Feature>Input` is what the service accepts (e.g. cleartext
 * password). `Create<Feature>Persistence` is what the repo accepts
 * (hashed). The service hashes between the two — credentials never
 * travel as cleartext past the service layer.
 */
export type Create<Feature>Input = {
  // every field the service needs, including pre-hash secrets
};

export type Create<Feature>Persistence =
  Omit<Create<Feature>Input, 'password'> & { password_hash: string };

export type Update<Feature>Patch = {
  // every field a service may PATCH on this row.
  // Privileged identity fields (e.g. `email`) are intentionally NOT here.
  updated_by?: number | null;
};
```

If your module has no credential transition (e.g. `time-entries`), this
file can be omitted and the service can accept simple inline types.

---

### 3.2 Persistence layer

**Hard rules:**

1. The **port is an abstract class**, not a TS interface. NestJS DI
   needs a runtime token; an interface compiles away to nothing.
2. The repo **never returns entities** — always maps to domain first.
3. The repo **never throws `NotFoundException`** — it returns
   `Foo | null`. The service is the sole owner of the not-found check
   and the friendly message.
4. The repo **respects `deleted_at IS NULL`** on every read path unless
   the caller explicitly opts in via `includeDeleted`.
5. When a partial unique index protects an invariant, **trust the
   index**. Wrap the insert in try/catch and map Postgres `23505` to
   `ConflictException`.

**`persistence/base-<feature>.repository.ts` template:**

```ts
import { <Feature> } from '@/<feature>/domain/<feature>';
import { <Feature>SearchCriteria } from '@/<feature>/domain/<feature>-search-criteria';
import { FindAll<Feature> } from '@/<feature>/domain/find-all-<feature>';
import { Create<Feature>Persistence, Update<Feature>Patch } from '@/<feature>/domain/<feature>-inputs';

export abstract class Base<Feature>Repository {
  abstract findAll(criteria: <Feature>SearchCriteria): Promise<FindAll<Feature>>;

  abstract findById(id: number): Promise<<Feature> | null>;

  abstract create(input: Create<Feature>Persistence): Promise<<Feature>>;

  /**
   * Trusts the caller has already verified the row exists.
   * Does NOT throw on missing id — that's the service's job.
   */
  abstract update(id: number, patch: Update<Feature>Patch): Promise<<Feature>>;

  abstract softDelete(id: number, deleted_by: number | null): Promise<void>;
}
```

**`persistence/entities/<feature>.entity.ts` template:**

```ts
import {
  Column, CreateDateColumn, DeleteDateColumn, Entity, Index,
  PrimaryGeneratedColumn, UpdateDateColumn,
} from 'typeorm';
import { EntityHelper } from '@/utils/entity-helper';

@Entity({ name: '<features>' })  // snake_case plural
@Index(['is_active'])
export class <Feature>Entity extends EntityHelper {
  @PrimaryGeneratedColumn()
  id!: number;

  // business columns …

  // ── audit ──────────────────────────────────────────────
  // Nullable INT, no FK constraint to users.id by default.
  // Adding the FK is a follow-up migration; the bootstrap
  // problem (first super-admin has no creator) makes it
  // awkward to bake in at table-creation time.
  @Column({ type: 'int', nullable: true })
  created_by!: number | null;

  @Column({ type: 'int', nullable: true })
  updated_by!: number | null;

  @Column({ type: 'int', nullable: true })
  deleted_by!: number | null;

  @CreateDateColumn({ type: 'timestamptz' })
  created_at!: Date;

  @UpdateDateColumn({ type: 'timestamptz' })
  updated_at!: Date;

  @DeleteDateColumn({ type: 'timestamptz', nullable: true })
  deleted_at!: Date | null;
}
```

For credential-bearing columns:

```ts
@Column({ type: 'varchar', length: 255, select: false })
password_hash!: string;
```

`select: false` excludes the column from default `find()` calls. The repo
must use `addSelect('foo.password_hash')` on the dedicated
`findByEmailWithCredentials` / `findByIdWithCredentials` methods to opt
back in.

For unique business keys, prefer a **partial functional unique index in a
migration** over `@Column({ unique: true })`:
- Plain unique indexes treat `'Jane@…'` and `'jane@…'` as distinct rows.
- Plain unique indexes block re-use after soft-delete.
- A partial functional index — e.g.
  `CREATE UNIQUE INDEX users_email_lower_uq ON users (LOWER(email)) WHERE deleted_at IS NULL`
  — avoids both pitfalls.

**`persistence/mappers/<feature>.mapper.ts` template:**

```ts
import { <Feature> } from '@/<feature>/domain/<feature>';
import { <Feature>Entity } from '@/<feature>/persistence/entities/<feature>.entity';

export class <Feature>Mapper {
  /**
   * Persistence → domain. Never copies credential columns.
   *
   * Fails fast if a required relation wasn't loaded — every read path
   * must join that relation, and the failure should surface at the
   * mapper seam rather than further down as
   * "Cannot read 'permissions' of undefined".
   */
  static toDomain(raw: <Feature>Entity): <Feature> {
    // if (!raw.someRelation) throw new Error('...mapper requires someRelation join...');

    const out = new <Feature>();
    out.id = raw.id;
    // copy every domain field …
    out.created_by = raw.created_by;
    out.updated_by = raw.updated_by;
    out.deleted_by = raw.deleted_by;
    out.created_at = raw.created_at;
    out.updated_at = raw.updated_at;
    out.deleted_at = raw.deleted_at;
    return out;
  }
}
```

A `toPersistence(domain: Partial<<Feature>>): <Feature>Entity` method is
optional — most modules build entity instances inline in
`repo.create()`, which is fine. Add `toPersistence` if you find yourself
constructing entities in more than one place.

**`persistence/repositories/<feature>.repository.ts` patterns:**

```ts
async findById(id: number): Promise<<Feature> | null> {
  const entity = await this.repo.findOne({
    where: { id },
    relations: [ /* every relation the mapper requires */ ],
  });
  return entity ? <Feature>Mapper.toDomain(entity) : null;
}

async create(input: Create<Feature>Persistence): Promise<<Feature>> {
  const entity = this.repo.create({ /* …explicit field copy… */ });
  const saved = await this.repo.save(entity);
  return this.requireById(saved.id);
}

async update(id: number, patch: Update<Feature>Patch): Promise<<Feature>> {
  const existing = await this.repo.findOneOrFail({ where: { id } });
  if (patch.someField !== undefined) existing.someField = patch.someField;
  // … one line per patchable field …
  await this.repo.save(existing);
  return this.requireById(id);
}

/** Atomic soft-delete: ONE statement sets both deleted_at and deleted_by. */
async softDelete(id: number, deleted_by: number | null): Promise<void> {
  await this.repo
    .createQueryBuilder()
    .update(<Feature>Entity)
    .set({ deleted_at: () => 'NOW()', deleted_by })
    .where('id = :id', { id })
    .execute();
}

/** Reload with relations for return after a save. */
private async requireById(id: number): Promise<<Feature>> {
  const entity = await this.repo.findOneOrFail({
    where: { id },
    relations: [ /* every relation the mapper requires */ ],
  });
  return <Feature>Mapper.toDomain(entity);
}
```

**Two TypeORM quirks worth knowing:**

- `.update()` (the direct one-statement form) does NOT fire
  `@UpdateDateColumn` listeners — only `.save()` does. If you use
  `.update()` for a side write (e.g. password rotation, `last_login_at`),
  manually set `updated_at: new Date()` in the same statement.
- `.softDelete()` calls TypeORM's built-in helper which sets `deleted_at`
  but NOT `deleted_by`. Use the QueryBuilder form above, or call
  `softDelete()` after a `.save()` that wrote `deleted_by` — the
  QueryBuilder form is preferred because it's atomic.

**`findAll` template (the systemized-filter implementation):**

```ts
async findAll(criteria: <Feature>SearchCriteria): Promise<FindAll<Feature>> {
  const page = criteria.page ?? PAGINATION_DEFAULTS.page;
  const limit = Math.min(criteria.limit ?? PAGINATION_DEFAULTS.limit,
                         PAGINATION_DEFAULTS.maxLimit);

  const qb = this.repo
    .createQueryBuilder('f')
    .leftJoinAndSelect('f.someRelation', 'r')
    .orderBy('f.created_at', 'DESC')
    .skip((page - 1) * limit)
    .take(limit);

  if (!criteria.includeDeleted) qb.andWhere('f.deleted_at IS NULL');
  if (criteria.search) {
    qb.andWhere('(f.name ILIKE :s OR f.email ILIKE :s)', { s: `%${criteria.search}%` });
  }
  if (criteria.is_active !== undefined) {
    qb.andWhere('f.is_active = :a', { a: criteria.is_active });
  }

  const [entities, total] = await qb.getManyAndCount();
  return {
    data: entities.map(<Feature>Mapper.toDomain),
    total,
    page,
    limit,
    has_more: page * limit < total,
  };
}
```

**`persistence/persistence.module.ts` template:**

```ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { <Feature>Entity } from '@/<feature>/persistence/entities/<feature>.entity';
import { <Feature>Repository } from '@/<feature>/persistence/repositories/<feature>.repository';
import { Base<Feature>Repository } from '@/<feature>/persistence/base-<feature>.repository';

@Module({
  imports: [TypeOrmModule.forFeature([<Feature>Entity])],
  providers: [
    <Feature>Repository,
    { provide: Base<Feature>Repository, useClass: <Feature>Repository },
  ],
  exports: [Base<Feature>Repository, <Feature>Repository, TypeOrmModule],
})
export class <Feature>PersistenceModule {}
```

Re-exporting `TypeOrmModule` lets other modules inject `<Feature>Entity`
when their repos need to join — e.g. `RoleRepository` injects
`PermissionEntity` from `PermissionPersistenceModule`.

---

### 3.3 Application layer (service)

**Hard rules:**

1. Service **depends on the port**: `private readonly repository: Base<Feature>Repository`.
2. Service owns **business invariants**: uniqueness checks, cross-table
   lookups (e.g. role exists), hashing, derived field computation (e.g.
   `status` from `time_out`).
3. Service is the **sole owner of `NotFoundException`**. Repo returns
   `null`; service translates.
4. Service may inject other features' `Base*Repository` (e.g.
   `UsersService` injects `BaseRoleRepository` to validate `role_id`) —
   never the other feature's concrete repository.
5. **No `if (audience === 'admin')` branches.** If self-service needs a
   different rule (e.g. self-service password change verifies the current
   password first), give it its own service method
   (`changeMyPassword`) — don't fork a shared method.

**`<feature>.service.ts` skeleton:**

```ts
import { Injectable, NotFoundException, ConflictException,
         UnprocessableEntityException } from '@nestjs/common';
import { Base<Feature>Repository } from '@/<feature>/persistence/base-<feature>.repository';
import { <Feature> } from '@/<feature>/domain/<feature>';
import { <Feature>SearchCriteria } from '@/<feature>/domain/<feature>-search-criteria';
import { FindAll<Feature> } from '@/<feature>/domain/find-all-<feature>';

@Injectable()
export class <Feature>sService {
  constructor(private readonly repository: Base<Feature>Repository) {}

  findAll(criteria: <Feature>SearchCriteria): Promise<FindAll<Feature>> {
    return this.repository.findAll(criteria);
  }

  async findById(id: number): Promise<<Feature>> {
    const row = await this.repository.findById(id);
    if (!row) throw new NotFoundException(`<Feature> with id ${id} not found`);
    return row;
  }

  async create(input: Create<Feature>Input): Promise<<Feature>> {
    // 1. uniqueness pre-checks (nice 422 errors before the DB constraint trips)
    // 2. cross-table FK existence checks
    // 3. credential hashing
    // 4. delegate to repo
    try {
      return await this.repository.create({ /* persistence input */ });
    } catch (err) {
      if (isUniqueViolation(err)) {
        throw new ConflictException('Concurrent insert won the race — retry.');
      }
      throw err;
    }
  }

  async update(id: number, patch: Update<Feature>Patch): Promise<<Feature>> {
    await this.findById(id);   // owns the 404
    // FK validation if any patch field is a FK
    return this.repository.update(id, patch);
  }

  async softDelete(id: number, deleted_by: number | null): Promise<void> {
    await this.findById(id);
    await this.repository.softDelete(id, deleted_by);
  }
}
```

**The error-shape contract:**

| Situation | Throw |
|---|---|
| Row not found by id | `NotFoundException` with `<Feature> with id ${id} not found` |
| Field-level validation that can't live on the DTO (e.g. cross-table FK existence) | `UnprocessableEntityException({ status: 422, errors: { field: 'msg' } })` |
| Hard uniqueness conflict on a business key | `ConflictException` |
| Forbidden action (e.g. delete a built-in role) | `ForbiddenException` |
| Authn-related (wrong current password) | `UnauthorizedException` |

The global `QueryFailedFilter` sanitizes leaked Postgres errors, but
your service should still try/catch around inserts protected by a
partial unique index and translate `23505` into a nice `ConflictException`.

---

### 3.4 DTO layer (`dto/admin/` and `dto/me/`)

**Hard rules:**

1. **One DTO file per audience per verb.** Never share a DTO between
   admin and `/me` routes. The audience is part of the security
   contract — conditional `@IsOptional()` is not a boundary.
2. **`/me` DTOs declare ONLY the fields the user may change.** The
   global `ValidationPipe` has `forbidNonWhitelisted: true`, so a field
   not declared on the DTO is rejected with 400. That's how
   `PATCH /users/me` refuses `role_id` — not a runtime check, but the
   DTO not declaring it. Adding a field to a `me/*.dto.ts` is the same
   security boundary change as adding a route.
3. **Privileged operations get their own endpoint, not extra fields.**
   Password change is never a field on the generic update DTO. Self-
   service: `PATCH /<feature>/me/password` with current-password
   re-verify. Admin: `POST /admin/<feature>/:id/reset-password` with no
   such check.
4. **Query DTOs need explicit transforms.** Querystrings arrive as
   strings — wrap booleans / numbers in `@Transform(...)` before the
   `@IsInt()` / `@IsBoolean()` validators run.

**Admin create-DTO skeleton:**

```ts
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';
import { IsBoolean, IsEmail, IsInt, IsOptional, IsString, MaxLength } from 'class-validator';

export class Create<Feature>Dto {
  @ApiProperty({ example: 'jane.smith@asima.inc' })
  @IsEmail()
  @MaxLength(255)
  email!: string;

  // … wide field set …

  @ApiPropertyOptional({ example: true, default: true })
  @IsOptional()
  @IsBoolean()
  is_active?: boolean;
  // ! DO NOT expose system_admin or any other bypass flag here.
  //   Such flags are seed-only — adding them to a public DTO turns a
  //   write permission into unconditional privilege bypass.
}
```

**Me update-DTO skeleton (narrow allowlist):**

```ts
import { ApiPropertyOptional } from '@nestjs/swagger';
import { IsOptional, IsString, MaxLength } from 'class-validator';

/**
 * Self-service profile update — narrow allowlist.
 * Anything not declared here is rejected by ValidationPipe.
 *
 * Privileged fields intentionally absent:
 *  - email      (admin-only, would change login identity)
 *  - role_id    (admin-only, privilege escalation)
 *  - is_active  (admin-only, self-deactivation footgun)
 *  - password   (separate endpoint with current-password re-verification)
 */
export class UpdateMeDto {
  @ApiPropertyOptional({ example: 'Jane' })
  @IsOptional() @IsString() @MaxLength(100)
  first_name?: string;

  @ApiPropertyOptional({ example: 'Smith' })
  @IsOptional() @IsString() @MaxLength(100)
  last_name?: string;
}
```

---

### 3.5 Interface layer (controllers)

**Hard rules:**

1. `@Controller({ path: '<segment>', version: API_VERSION })` — import
   `API_VERSION` from `@/utils/constants/api.constants`.
2. Admin controller path: `admin/<feature>`. Self-service path:
   `<feature>/me` or `users/me/<sub>` if the resource is "the user's X".
3. **No `:id` on any `me-*` route.** Identity comes from `req.user.id`
   via `@CurrentUser()` — never from the URL.
4. Every admin route declares its `@Permissions({ RESOURCE: 'Action' })`
   gate. Self-service routes declare nothing — identity is the gate
   (the global `JwtAuthGuard` populates `req.user`, and
   `PermissionsGuard` passes when no `@Permissions(...)` metadata is
   present).
5. Never use `@UseGuards(JwtAuthGuard)` — it's already global. The only
   place per-route guards appear is `JwtRefreshGuard` on `/auth/refresh`
   and `@Public()` on the three exempt routes (`/auth/login`,
   `/auth/refresh`, `/health`).
6. Swagger annotations on every route: `@ApiOperation`, `@ApiResponse`,
   and class-level `@ApiTags` + `@ApiBearerAuth`. Tag convention:
   - Admin controller: `@ApiTags('Admin - <Resource>')`.
   - Me controller: `@ApiTags('<Resource>')` (plain).
7. Privileged routes get a per-route throttler tier:
   `@Throttle({ password: { limit: 5, ttl: 60_000 } })`.

**Admin controller skeleton:**

```ts
@ApiTags('Admin - <Resource>')
@ApiBearerAuth()
@Controller({ path: 'admin/<feature>', version: API_VERSION })
export class Admin<Feature>Controller {
  constructor(private readonly service: <Feature>sService) {}

  @Get()
  @Permissions({ RESOURCE: 'View' })
  @ApiOperation({ summary: 'List …' })
  findAll(@Query() query: Query<Feature>Dto): Promise<FindAll<Feature>> {
    return this.service.findAll(query);
  }

  @Post()
  @Permissions({ RESOURCE: 'Create' })
  @ApiOperation({ summary: 'Create …' })
  @ApiResponse({ status: 201 })
  create(@Body() dto: Create<Feature>Dto, @CurrentUser() actor: User): Promise<<Feature>> {
    return this.service.create({ ...dto, created_by: actor.id });
  }

  @Patch(':id')
  @Permissions({ RESOURCE: 'Update' })
  update(@Param('id', ParseIntPipe) id: number,
         @Body() dto: Update<Feature>Dto,
         @CurrentUser() actor: User): Promise<<Feature>> {
    return this.service.update(id, { ...dto, updated_by: actor.id });
  }

  @Delete(':id')
  @Permissions({ RESOURCE: 'Delete' })
  @HttpCode(HttpStatus.NO_CONTENT)
  async remove(@Param('id', ParseIntPipe) id: number,
               @CurrentUser() actor: User): Promise<void> {
    await this.service.softDelete(id, actor.id);
  }
}
```

**Me controller skeleton:**

```ts
@ApiTags('<Resource>')
@ApiBearerAuth()
@Controller({ path: '<feature>/me', version: API_VERSION })
export class Me<Feature>Controller {
  constructor(private readonly service: <Feature>sService) {}

  @Get()
  me(@CurrentUser() actor: User): Promise<<Feature>> {
    return this.service.findById(actor.id);
  }

  @Patch()
  update(@Body() dto: UpdateMeDto, @CurrentUser() actor: User): Promise<<Feature>> {
    return this.service.update(actor.id, { ...dto, updated_by: actor.id });
  }
}
```

---

### 3.6 Module wiring

```ts
import { Module } from '@nestjs/common';
import { <Feature>sService } from '@/<feature>/<feature>.service';
import { <Feature>PersistenceModule } from '@/<feature>/persistence/persistence.module';
import { Admin<Feature>Controller } from '@/<feature>/controllers/admin-<feature>.controller';
import { Me<Feature>Controller } from '@/<feature>/controllers/me-<feature>.controller';
// import { <Dependency>Module } from '@/<dependency>/<dependency>.module';

@Module({
  imports: [
    <Feature>PersistenceModule,
    // <Dependency>Module,   // if service injects another feature's Base*Repository
  ],
  controllers: [Admin<Feature>Controller, Me<Feature>Controller],
  providers: [<Feature>sService],
  exports: [<Feature>sService, <Feature>PersistenceModule],
})
export class <Feature>sModule {}
```

**Why re-export the persistence module from the feature module:**
downstream features need access to both the service AND the
`Base*Repository` port (e.g. `UsersModule` needs `BaseRoleRepository` to
validate `role_id`; the auth module needs `BaseUserRepository` to look
up credentials). Re-exporting the persistence module keeps a single
import line for any consumer.

---

## 4. Cross-cutting conventions (don't deviate)

### 4.1 Snake_case end-to-end

DB columns, domain field names, JSON wire payloads all use
`created_at`, `is_active`, `password_hash`. Do NOT add a naming-
strategy plugin or a case-translation layer at the boundary. If the
frontend wants camelCase, that's a frontend concern.

### 4.2 Definite assignment (`!`)

Every field of every data class needs `!`:

```ts
id!: number;                    // non-nullable    → needs !
email!: string;                 // non-nullable    → needs !
title!: string | null;          // nullable        → STILL needs ! (null ≠ undefined)
last_login_at?: Date;           // optional        → no ! (the ? makes it undefined-able)
```

`tsconfig.json` keeps `strictPropertyInitialization: true` so CI catches
anyone who skips `!` on a new field. Do not turn the flag off.

### 4.3 Audit fields on every entity from day 1

`created_by`, `updated_by`, `deleted_by` (nullable INT, no explicit FK
to users.id by default) + `deleted_at` (soft delete via
`@DeleteDateColumn`). Adding these later is a painful migration.

The one exception is **seed-managed configuration tables** (e.g.
`permissions`) — those have no user actor and can omit the audit
columns. Document the exception in the entity file's docstring.

### 4.4 Snake_case enums as const objects (not TS enums)

```ts
export const TIME_ENTRY_STATUSES = {
  open: 'open',
  confirmed: 'confirmed',
} as const;

export type TimeEntryStatus =
  (typeof TIME_ENTRY_STATUSES)[keyof typeof TIME_ENTRY_STATUSES];
```

Match the TypeORM column type:
`@Column({ type: 'enum', enum: ['open', 'confirmed'], enumName: 'time_entry_status' })`.

Store stable enum values in `<feature>.constants.ts`.

### 4.5 Path alias

Always `@/foo/bar`. Configured in `tsconfig.json`, `nest-cli.json`, and
Jest's `moduleNameMapper`. No `../../../`.

### 4.6 Pagination

`PaginatedResponse<T>` from `@/utils/types/paginated-response.type`.
Defaults from `PAGINATION_DEFAULTS` (`page=1`, `limit=20`,
`maxLimit=100`). Repo builds the response; service and controller pass
it through.

### 4.7 Swagger

- `/docs` is non-prod only.
- Class-level `@ApiTags(...)` + `@ApiBearerAuth()` on every controller.
- `@ApiOperation` + at least one `@ApiResponse` on every route.
- `@ApiProperty` / `@ApiPropertyOptional` on every field of the domain
  class (these are stripped at runtime, so domain stays "pure").
- Schema grouping: if you add a new DTO group, register it in `main.ts`
  via the `schema-groups.ts` helper. See `asima-backend/CLAUDE.md` for
  details.

### 4.8 Migrations

- One operation per migration file (one table, one ALTER). Naming
  patterns are in `asima-backend/CLAUDE.md`.
- `npm run migration:generate src/database/migrations/<Name>` →
  `npm run migration:run`.
- Never `synchronize: true`.
- Each new entity needs its `Create<Plural>Table` migration BEFORE any
  cross-table FK or partial unique index migration.

---

## 5. Admin vs self-service split

The rule that makes the split work: **the audience is enforced at the
edge** (DTO + controller). The domain, persistence, and service layers
are oblivious to who is calling.

| Layer | Admin-aware? | How |
|---|---|---|
| Controller | YES | Distinct files. `admin-foo` carries `@Permissions(...)`; `me-foo` does not, and uses `req.user.id` instead of `:id`. |
| DTO | YES | Distinct folders. `dto/admin/*` exposes the wide field set; `dto/me/*` declares only what the user may change. |
| Service | NO | One service. Self-service-specific logic (e.g. `changeMyPassword` re-verifying current password) gets its OWN method, not an `if (audience)` branch. |
| Domain | NO | One class. |
| Persistence | NO | One repo. Credential-loading methods are per-call-site (`findByEmailWithCredentials` for login, `findByIdWithCredentials` for self-service password change). |

**Where to look for a working example:** `src/users/`. The flow:

- `PATCH /admin/users/:id` accepts `UpdateUserDto` (wide), updates any row.
- `PATCH /users/me` accepts `UpdateMeDto` (narrow: `first_name`,
  `last_name` only), updates `req.user.id`.
- Both controllers call the SAME `usersService.update(id, patch)`.
- The narrow DTO is the security boundary — `role_id` isn't on
  `UpdateMeDto`, so `ValidationPipe`'s `forbidNonWhitelisted: true`
  rejects it with 400 before the service ever sees the request.

---

## 6. Data flow walkthroughs

### 6.1 `POST /api/v1/admin/users` — create a user

1. **Interface.** `AdminUsersController.create` receives the request.
   `ValidationPipe` validates `CreateUserDto` (`@IsEmail`, password
   complexity regex, `is_active` boolean, etc.). `@Permissions({ USER: 'Create' })` is enforced by `PermissionsGuard`.
   `@CurrentUser() actor` is extracted from `req.user`.
2. **Application.** Controller calls
   `usersService.create({ ...dto, created_by: actor.id })`.
   - Service normalizes email to lowercase.
   - Service checks `existsByEmail(email)` — throws 422 if taken (nicer
     error than the DB unique constraint).
   - Service validates `role_id` exists via `BaseRoleRepository` — throws
     422 with `{ role_id: 'Unknown role id: ...' }` if not.
   - Service bcrypts the password (cost 12).
   - Service calls `repository.create({ ...persistence input… })`.
3. **Infrastructure.** `UserRepository.create` builds the entity via
   `repo.create(...)`, calls `repo.save(entity)`, then `requireById` to
   reload with `role` + `role.permissions` joined.
4. **Mapping back.** `UserMapper.toDomain` builds a `User` from the
   entity. `password_hash` is NOT copied — it stops at this seam.
5. **Return.** Service returns the `User`. Controller serializes it.
   Swagger has already documented the shape from `@ApiProperty`s on the
   domain class.

### 6.2 `POST /api/v1/users/me/time-entries/punch` — toggle punch

1. **Interface.** `MeTimeEntriesController.punch` runs under the global
   `JwtAuthGuard`. No `@Permissions(...)` decorator — identity is the
   gate. `@CurrentUser() actor` provides the employee id.
2. **Application.** `timeEntriesService.punch(actor)`:
   - Calls `repository.findOpenForEmployee(actor.id)`.
   - **If an open entry exists** → calls `repository.update(open.id, { time_out: now, status: 'confirmed', updated_by: actor.id })`.
   - **If none exists** → calls `repository.create({ ... source: 'manual', status: 'open' ... })`.
   - The create call is wrapped in try/catch. The partial unique index
     `(employee_id) WHERE status='open' AND deleted_at IS NULL` is the
     source of truth for "at most one open per employee" — if two punches
     race, the loser gets `23505` and the service maps it to 409. The
     client retries and now sees the open entry and closes it.
3. **Infrastructure.** Repo writes the row, reloads via `requireById`,
   maps to domain, returns.

### 6.3 Soft-delete

`DELETE /api/v1/admin/users/:id` →
`usersService.softDelete(id, actor.id)`:

1. Service calls `findById(id)` — throws 404 if missing.
2. Service calls `repository.softDelete(id, actor.id)`.
3. Repo runs ONE statement:
   `UPDATE users SET deleted_at = NOW(), deleted_by = :id WHERE id = :id`.
   Avoid the older two-call pattern (`save({deleted_by}); softDelete()`) —
   if the second call fails, the row is half-deleted.
4. All subsequent `findAll` calls skip the row because
   `qb.andWhere('u.deleted_at IS NULL')` is the default unless
   `criteria.includeDeleted` is true.

---

## 7. Concurrency & DB-level invariants

For any "at most N rows where …" rule, **enforce it at the DB level
with a partial unique index** and let the service translate the
constraint violation into a friendly error.

Two cases in the current codebase:

- `users`: partial functional unique index
  `users_email_lower_uq ON users(LOWER(email)) WHERE deleted_at IS NULL`.
  Allows email re-use after soft-delete; treats `'Jane@…'` and `'jane@…'`
  as the same login.
- `time_entries`: partial unique index
  `(employee_id) WHERE status='open' AND deleted_at IS NULL`. Source of
  truth for "one open punch per employee."
- `work_schedules`: partial unique index
  `(employee_id, day_of_week) WHERE effective_to IS NULL AND deleted_at IS NULL`.
  Source of truth for "one active schedule per (employee, weekday)."

Service-layer pre-checks (`existsByEmail`, `findOpenForEmployee`) give
nicer error messages on the common path, but they are NOT the safety
net — the DB index is. The service must still try/catch the insert and
map `23505` → `ConflictException` for the race-loser case.

---

## 8. Testing

- Co-locate unit specs as `<feature>.service.spec.ts`.
- Mock the **abstract `Base<Feature>Repository`** — that's the whole
  point of the port:

  ```ts
  let repo: jest.Mocked<Base<Feature>Repository>;

  beforeEach(() => {
    repo = {
      findAll: jest.fn(),
      findById: jest.fn(),
      create: jest.fn(),
      update: jest.fn(),
      softDelete: jest.fn(),
      // …every abstract method…
    };
    service = new <Feature>sService(repo as unknown as Base<Feature>Repository);
  });
  ```

- Domain classes have no NestJS imports → trivially unit-testable with
  no DI graph.
- Never mock the concrete repository or `Repository<FooEntity>` — that
  binds your test to TypeORM details and defeats the port.
- E2E: `npm run test:e2e`. Coverage: `npm run test:cov`.

---

## 9. When the resource needs the `/me` half

A resource has a self-service surface when **the user themselves needs
to read or modify it** without going through HR. Examples in this
codebase:

| Resource | `/admin` | `/me` |
|---|---|---|
| `users` | YES (HR manages anyone) | YES (everyone updates own profile / password) |
| `time-entries` | YES (HR back-fills, edits) | YES (everyone punches in/out, lists own) |
| `roles` | YES | NO (employees never manage roles) |
| `permissions` | (read-only, seed-managed) | NO |

A new module probably needs the `/me` half if any answer is "yes":

- Will the user view this resource on their own dashboard?
- Will the user create/edit it themselves (e.g. submit a leave request)?
- Does the resource carry credentials they need to rotate?

If yes, plan the `/me` controller + narrow DTOs from day 1 — adding
them later is fine, but the DTO split is a security boundary, so
revisit the access rules at that point.

---

## 10. Module boundaries with other features

When your service needs another feature, inject the **abstract
repository** of that other feature, never its concrete class:

```ts
// Good: depend on the port. Mockable. Survives the other feature's
// internal refactors.
constructor(
  private readonly repository: Base<Feature>Repository,
  private readonly roleRepository: BaseRoleRepository,
) {}
```

For this to work, the other feature's module must export the
persistence module (which exports `BaseRoleRepository`) — which is the
default in the templates above. Then in your `<feature>.module.ts`:

```ts
imports: [<Feature>PersistenceModule, RolesModule],
```

Avoid circular module imports — if A and B both need each other, one
side should import only the other's persistence module
(`OtherPersistenceModule`), not the full `OtherModule`.

---

## 11. New-module checklist

Run through this before opening a PR for a new `<feature>` module:

**Domain**
- [ ] `domain/<feature>.ts` exists, has zero `@nestjs/*` runtime
      imports and zero `typeorm` imports.
- [ ] Every field uses `!` (including `field: string | null` ones).
- [ ] Audit fields (`created_by`, `updated_by`, `deleted_by`,
      `created_at`, `updated_at`, `deleted_at`) all present.
- [ ] `<feature>-search-criteria.ts` defines the filter shape.
- [ ] `find-all-<feature>.ts` exports the `PaginatedResponse` alias.
- [ ] `<feature>-inputs.ts` exists if the service massages inputs (e.g.
      credentials → hash).

**Persistence**
- [ ] `base-<feature>.repository.ts` is an `abstract class` (not an
      interface).
- [ ] Concrete `<feature>.repository.ts` extends the abstract class,
      `@Injectable()`, uses `@InjectRepository(<Feature>Entity)`.
- [ ] `<feature>.entity.ts` extends `EntityHelper`, has audit columns,
      uses `@DeleteDateColumn` for `deleted_at`.
- [ ] `<feature>.mapper.ts::toDomain` never copies credential columns
      and fails fast on missing required relations.
- [ ] `findAll` paginates, respects `includeDeleted`, returns the
      `PaginatedResponse` shape.
- [ ] `softDelete` is atomic (one SQL statement sets both `deleted_at`
      and `deleted_by`).
- [ ] If you use `.update()` (not `.save()`) for a side write,
      `updated_at: new Date()` is set explicitly.
- [ ] `persistence.module.ts` binds `{ provide: Base<Feature>Repository, useClass: <Feature>Repository }` and re-exports `TypeOrmModule`.
- [ ] A migration exists to create the table; any business-key
      uniqueness is a partial functional unique index (not
      `unique: true` on the column).

**DTO**
- [ ] `dto/admin/*.dto.ts` carries the wide field set.
- [ ] `dto/me/*.dto.ts` declares ONLY the fields the user may change
      themselves. No `role_id`, no `is_active`, no privilege flags.
- [ ] Query DTOs use `@Transform(...)` to coerce querystring values.
- [ ] Password fields (if any) live on dedicated DTOs
      (`ChangeMyPasswordDto`, `ResetUserPasswordDto`), never folded
      into a generic update DTO.

**Controllers**
- [ ] `admin-<feature>.controller.ts` uses
      `@Controller({ path: 'admin/<feature>', version: API_VERSION })`,
      `@ApiTags('Admin - <Resource>')`, `@ApiBearerAuth()`.
- [ ] Every admin route declares
      `@Permissions({ RESOURCE: 'Action' })`.
- [ ] `me-<feature>.controller.ts` uses
      `@Controller({ path: '<feature>/me', version: API_VERSION })`,
      `@ApiTags('<Resource>')`, has NO `:id` segment on any route,
      drives identity from `@CurrentUser()`.
- [ ] No `@UseGuards(JwtAuthGuard)` per route (already global).
- [ ] Privileged routes (password) have `@Throttle(...)`.
- [ ] Every route has `@ApiOperation` + at least one `@ApiResponse`.

**Service**
- [ ] Depends on `Base<Feature>Repository` (port), not the concrete.
- [ ] Owns the `NotFoundException` translation; repo returns null.
- [ ] Uniqueness pre-checks for nice 422 + try/catch for
      `23505` → 409.
- [ ] Self-service variants are distinct methods, not
      `if (audience)` branches.

**Module**
- [ ] `<feature>.module.ts` imports `<Feature>PersistenceModule` and
      any dependency `*Module`s, registers both controllers, provides
      the service, exports `{ service, <Feature>PersistenceModule }`.

**Tests**
- [ ] `<feature>.service.spec.ts` mocks the abstract repository.
- [ ] At least the create / update / softDelete / one filter case is
      covered.

**Cross-cutting**
- [ ] Snake_case used everywhere (DB columns, domain fields, JSON
      payloads).
- [ ] Path alias `@/...` used in every import — no `../../../`.
- [ ] Migration runs cleanly via `npm run db:fresh` (dev only).
- [ ] `npm run lint && npm run test && npm run build` all green.

---

## 12. Anti-patterns (don't do these)

| Anti-pattern | Why it's wrong | Do this instead |
|---|---|---|
| `@Injectable()` or `@Entity()` on a domain class | Breaks every test that mocks the repo, leaks framework into the core. | Keep domain pure; entity is a separate class in `persistence/entities/`. |
| Rich domain methods (`user.activate()`, `time.close()`) | We deliberately use anemic domain models — business logic lives in the service. | Put the rule in `<feature>.service.ts`. |
| Sharing a DTO between admin and `/me` routes (with `@IsOptional()` on privileged fields) | The audience IS the security boundary; conditional optionality doesn't enforce it. | Two DTOs, two folders. |
| `:id` on a `/me` route, or reading `params.id` in a me-controller | Lets a caller pivot identity by editing the URL. | Use `@CurrentUser().id`. |
| `if (caller.role === 'admin') { … }` inside the service | Branching like that is exactly how privilege escalation gets shipped. | Move the privileged path to its own service method called from the admin controller. |
| `@UseGuards(JwtAuthGuard)` on a controller | It's already global. Double-applying does nothing useful and adds noise. | Delete it. Use `@Public()` for the rare exempt routes. |
| Throwing `NotFoundException` from the repository | Splits the 404 contract across layers. | Repo returns null; service throws. |
| Two-call soft-delete (`save({deleted_by}); softDelete()`) | Non-atomic — a failure between calls leaves the row half-deleted. | One QueryBuilder statement sets both fields. |
| Direct `.update()` on a column you care about timestamping | TypeORM's `@UpdateDateColumn` only fires on `.save()` — your `updated_at` will silently lag. | Set `updated_at: new Date()` in the same `.update()` call. |
| `@Column({ unique: true })` on a business key (e.g. email) | Doesn't fold case, blocks re-use after soft-delete. | Use a partial functional unique index in a migration. |
| Returning entities from the repo, or letting controllers see entities | Couples your interface to TypeORM's reflective surface; breaks the port. | Map at the persistence boundary — always. |
| Adding `system_admin: true` (or any bypass flag) as an exposed DTO field | Turns the `RESOURCE:Create` permission into unconditional privilege bypass. | Bypass flags are seed-only. |
| Mocking the concrete repository in a service spec | Defeats the point of the port; the test now depends on TypeORM internals. | Mock `Base<Feature>Repository`. |
| `synchronize: true` in any TypeORM config | Silently rewrites schema; loses migration history. | Always `false`; use migrations. |
| Adding a logger library, multi-tenancy, OAuth, or any other big architectural piece | v0 is deliberately narrow. | Write an ADR first; get sign-off before changing the shape. |

---

## 13. Authoritative cross-references

When a rule here conflicts with one of these, the **specific document
wins** for its area:

- `asima-backend/CLAUDE.md` — backend rules in operational detail
  (permissions seed, throttler tiers, migration command list,
  time-entry / work-schedule invariants).
- `asima-parent/CLAUDE.md` — system-level conventions (snake_case end-
  to-end, permission code shape, admin/me contract).
- `asima-backend/docs/adr/` — architectural decisions. Read the relevant
  ADR before changing roles, auth, or approval logic.
- `asima-backend/tasks/plan.md` — current phase plan; tells you what's
  built and what's coming.

If you spot a divergence between this guide and what the code actually
does in `users`, `roles`, or `time-entries`, the code is the source of
truth — open a PR updating this document.
