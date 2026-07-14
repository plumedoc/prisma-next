---
name: "prisma-next"
description: "Prisma Next is a contract-first TypeScript rewrite of Prisma ORM supporting PostgreSQL and MongoDB. Load this skill for tasks like: authoring or editing a Prisma Next contract (models/fields/relations), running the `prisma-next` CLI (contract emit, db init/update/verify/sign, migration plan/show/status/check, migrate, ref), wiring `prisma-next.config.ts`, and writing queries against a Prisma Next database via the SQL query builder or the ORM `Collection` client."
---

# Using Prisma Next

Prisma Next is a contract-first TypeScript rewrite of Prisma ORM (Early Access, APIs still evolving). Instead of generating an opaque client from a schema, it compiles a **contract** — models/fields/relations authored in PSL or TypeScript — into two artifacts: `contract.json` (canonical, hashed IR) and `contract.d.ts` (types). Migrations, the ORM client, and the SQL query builder all consume these artifacts. Never hand-edit `contract.json`/`contract.d.ts` — they are generated (and carry warning headers); edit the source (`contract.prisma` or `contract.ts`) and re-run `prisma-next contract emit`.

## Mental model in one paragraph

You author a **contract** (source of truth). `prisma-next contract emit` compiles it to `contract.json`/`contract.d.ts`. A `prisma-next.config.ts` wires which database family/target/adapter/driver you're using and where the contract lives. `prisma-next db init` (fresh DB, additive-only) or `prisma-next db update` (any DB, full reconciliation including destructive changes) makes the live database's schema match the contract, and writes a **marker** row recording the contract hash it now satisfies. Application code creates one runtime client (`db`) from a target facade package and queries either through the **SQL builder** (`db.sql...`, one-query-one-statement, compiles to a `Plan`) or the **ORM client** (`db.<Model>...`, a `Collection`, may issue multiple queries for things like `include`).

## Task: set up a new project

```bash
npm create prisma@next        # interactive scaffolder (new project)
# or, in an existing project:
npx prisma-next@latest init
```

`init` writes `prisma-next.config.ts`, a starter contract, `src/prisma/db.ts`, installs the runtime, emits the contract, and registers agent skill files (`.claude/skills/*/SKILL.md`, `.agents/skills/*/SKILL.md`). Requires Node.js 24+.

## Task: author or edit a contract

Use the target's `contract-builder` module. Pattern (works the same across targets — swap package names):

```typescript
import { defineContract, rel } from '@prisma-next/sqlite/contract-builder'; // or postgres/mongo
import { textColumn, datetimeColumn } from '@prisma-next/adapter-sqlite/column-types';

export const contract = defineContract({}, ({ field, model }) => {
  const User = model('User', {
    fields: {
      id: field.id.uuidv4String(),
      email: field.column(textColumn),
      createdAt: field.column(datetimeColumn).defaultSql('now()'),
    },
  });
  const Post = model('Post', {
    fields: { id: field.id.uuidv4String(), title: field.column(textColumn), userId: field.uuidString() },
  });

  return {
    models: {
      User: User.relations({ posts: rel.hasMany(Post, { by: 'userId' }) }).sql({ table: 'user' }),
      Post: Post.relations({ user: rel.belongsTo(User, { from: 'userId', to: 'id' }) })
        .sql(({ cols, constraints }) => ({
          table: 'post',
          foreignKeys: [constraints.foreignKey(cols.userId, User.refs.id, { name: 'post_userId_fkey' })],
        })),
    },
  };
});
```

Relation kinds: `rel.belongsTo(Target, { from, to })` (N:1), `rel.hasMany(Target, { by })` (1:N), `rel.manyToMany(() => Target, { through: () => Junction, from, to })` (M:N via an explicit junction model). A many-to-many junction that has ONLY the two FK columns ("pure") gets nested `create`/`connect`/`disconnect` sugar on the ORM client; a junction with an extra required payload column disables that sugar at the type level — populate it via its own model instead.

After editing, always re-emit:

```bash
prisma-next contract emit
```

Adding a model/field workflow: edit the contract source → `prisma-next contract emit` → `prisma-next db update --dry-run` to preview → `prisma-next db update -y` (or without `-y` and confirm interactively) to apply → the marker updates automatically.

## Task: wire `prisma-next.config.ts`

Prefer a target facade for simple cases:

```typescript
import { defineConfig } from '@prisma-next/sqlite/config'; // or postgres/mongo
export default defineConfig({
  contract: './prisma/contract.ts',
  outputPath: './src/prisma',
  db: { connection: process.env['DATABASE_URL'] ?? './demo.db' },
});
```

For explicit control (multiple extension packs, custom adapters), use `@prisma-next/cli/config-types` directly with `family`/`target`/`adapter`/`driver`/`extensionPacks`/`contract` fields — see the Config Reference page for the full shape. `driver` is required for any command that touches a live database (`db verify/sign/init/update/schema`, `contract infer`) but NOT for `contract emit` (offline).

## Task: run the CLI (most common commands)

```bash
prisma-next contract emit                          # offline: contract.ts/.prisma -> contract.json/.d.ts
prisma-next db init --db $DATABASE_URL              # first-time bootstrap (additive only)
prisma-next db update --db $DATABASE_URL --dry-run  # preview reconciliation (additive+widening+destructive)
prisma-next db update --db $DATABASE_URL            # apply (prompts for destructive ops unless -y)
prisma-next db verify --db $DATABASE_URL            # read-only: marker + schema check
prisma-next db sign --db $DATABASE_URL              # verify schema, then write/update the marker
prisma-next db schema --db $DATABASE_URL            # read-only live schema introspection
prisma-next contract infer --db $DATABASE_URL       # brownfield: DB -> contract.prisma
prisma-next migration plan --name add_column        # offline: diff contract-to-contract, scaffold a migration
prisma-next migration show                          # inspect the latest migration package
prisma-next migration status --db $DATABASE_URL      # migration graph + applied status
prisma-next migrate --to <contract-ref>              # advance a live DB along the migration graph (forward-only)
```

All commands accept `--config <path>` (default `./prisma-next.config.ts`), `--json`, `-q/--quiet`, `-v/--verbose`, `--no-color`. `--db <url>` overrides `config.db.connection`.

Pick `db init` vs `db update`: use `db init` only for a brand-new/empty database (fails loudly if a marker exists and mismatches); use `db update` for every subsequent schema change on any database — it's idempotent and safe to run repeatedly (0 operations if already in sync).

If a database-touching command fails with a wiring-validation error, it means the contract's `extensionPacks`/`target`/`family` don't match what's declared in `prisma-next.config.ts` — add the missing descriptor(s) to `config.extensionPacks` (matched by `id`) and re-run.

## Task: query the database from application code

Create exactly one runtime client per app, in one `db.ts` file:

```typescript
import postgres from '@prisma-next/postgres/runtime'; // or sqlite/mongo
import type { Contract, TypeMaps } from './contract.d';
import contractJson from './contract.json' with { type: 'json' };

export const db = postgres<Contract, TypeMaps>({ contractJson, url: process.env['DATABASE_URL']! });
```

Then import `db` everywhere else. Two query surfaces, same runtime:

**SQL builder** — chained, one-statement, explicit:

```typescript
import { db } from './prisma/db';
const plan = db.sql.user.select('id', 'email').limit(10).build();
const rows = await db.runtime().execute(plan);
```

**ORM client** (`Collection`) — higher-level, immutable chaining, terminal call executes:

```typescript
const user = await db.User.first({ id: userId });
const withPosts = await db.User.include('posts', (p) => p.take(5)).where({ id: userId }).first();
const created = await db.User.create({ id, email, displayName, createdAt: new Date() });
const matches = await db.Post.where((p) => p.tags.some((t) => t.label.eq('typescript'))).all();
```

Use `ResultType<typeof plan>` (from `@prisma-next/sql-query/types`) to derive a plan's row type instead of hand-writing one.

Wrap a read-then-decide-then-write sequence (e.g. quota check before insert) in `db.transaction(async (tx) => { ... })` — mixing a SQL-builder read and an ORM-client write inside one transaction callback is the normal pattern for atomic check-then-act logic.

## Pitfalls

- Don't hand-edit `contract.json`/`contract.d.ts` — they're generated; edit the contract source and re-emit.
- `contract emit` is fully offline; don't add a `driver` to config just for it — but DO add one for anything that touches a live DB.
- `db init` is for first-time bootstrap only (additive-only, fails on marker mismatch); use `db update` for ongoing schema evolution.
- Destructive `db update` plans require `-y/--yes` in non-interactive contexts (CI) or interactive confirmation.
- `migrate` never reverses a database; to roll back, `migration plan --to <migration-dir>^` to author a reverse edge, then `migrate` to it.
- A many-to-many junction with any required non-FK payload column loses ORM nested-write sugar (`connect`/`create` through the relation become `never` at the type level) — this is intentional, not a bug.
- Keep exactly one `db.ts` runtime entrypoint per app; avoid re-exporting aliases of `db` from multiple files.
- `--config` defaults to `./prisma-next.config.ts` in the CWD with no upward directory search — run CLI commands from the project root that contains it (or pass `--config` explicitly).

For exhaustive per-command flags/error codes and the config field table, see the CLI Command Reference and Config Reference pages in the docs site.

## Staying current

This skill was generated by plumedoc from `prisma/prisma-next@5bd63fc3804a8af57747df1aade107c4425424c0` on 2026-07-14.
It carries task-level knowledge only. For exhaustive reference, anything outside
the ground it covers, or if `prisma/prisma-next` has moved past this commit,
consult the live plumedoc docs at https://plumedoc.com/prisma/prisma-next (and its `llms.txt`), which track the source.
