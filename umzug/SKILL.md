---
name: umzug
description: "Reference for Umzug v3, the framework-agnostic migration tool for Node.js, adapted for Sequelize v7 (@sequelize/core). Use when writing database migrations, setting up a migration runner, configuring migration storage (SequelizeStorage, JSON, MongoDB, custom), running or reverting migrations programmatically or via CLI, creating migration files with queryInterface, or managing migration ordering and templates."
metadata:
  version: "1.0.0"
  umzug-version: "3.x"
  sequelize-version: "7.x-alpha"
  author: totophe
---

# Umzug v3 — Claude Skill (Sequelize v7 Adapted)

## Key Concepts

- Umzug is **not** an ORM — it orchestrates running/reverting migration functions and tracking which have been executed
- The `context` option passes any value to every migration's `up`/`down` — for Sequelize, use `sequelize.getQueryInterface()`
- `SequelizeStorage` tracks executed migrations in a database table (`SequelizeMeta` by default)
- Migrations are discovered via glob patterns or provided as an inline array
- Built-in CLI via `umzug.runAsCLI()` — supports `up`, `down`, `pending`, `executed`, `create`
- Import from `'umzug'`: `Umzug`, `SequelizeStorage`, `JSONStorage`, `MongoDBStorage`, `memoryStorage`, `MigrationError`

## Sequelize v7 Specifics

- Import Sequelize from `@sequelize/core` (not `sequelize`)
- Import dialect classes: `PostgresDialect`, `SqliteDialect`, `MysqlDialect`, etc.
- `DataTypes` from `@sequelize/core` for migration column definitions
- Sequelize v7 constructor takes a single options object (no URL string as first arg)
- CLS transactions enabled by default — use `sequelize.transaction()` for managed, `sequelize.startUnmanagedTransaction()` for unmanaged
- `dialectOptions` removed — settings go in top-level options

## Skill Files

| File | Contents |
|------|----------|
| [getting-started.md](references/getting-started.md) | Installation, Sequelize v7 setup, TypeScript config, context pattern, logging |
| [migrations.md](references/migrations.md) | Migration files, glob discovery, resolvers, SQL migrations, ordering, creating migrations, QueryInterface reference |
| [storage.md](references/storage.md) | SequelizeStorage, JSONStorage, MemoryStorage, MongoDBStorage, custom storage |
| [api-and-cli.md](references/api-and-cli.md) | up/down API, CLI commands, events, error handling, seeder pattern, transactions |

## Quick Reference — Common Patterns

### Migrator Entry Point (TypeScript + Sequelize v7)

```typescript
import { Sequelize } from '@sequelize/core';
import { PostgresDialect } from '@sequelize/postgres';
import { Umzug, SequelizeStorage } from 'umzug';

const sequelize = new Sequelize({
  dialect: PostgresDialect,
  host: 'localhost',
  port: 5432,
  database: 'mydb',
  user: 'user',
  password: 'pass',
});

const umzug = new Umzug({
  migrations: { glob: 'migrations/*.ts' },
  context: sequelize.getQueryInterface(),
  storage: new SequelizeStorage({ sequelize }),
  logger: console,
});

export type Migration = typeof umzug._types.migration;

if (require.main === module) {
  umzug.runAsCLI();
}
```

### Migration File

```typescript
import type { Migration } from '../migrator';
import { DataTypes } from '@sequelize/core';

export const up: Migration = async ({ context: queryInterface }) => {
  await queryInterface.createTable('users', {
    id: { type: DataTypes.INTEGER, primaryKey: true, autoIncrement: true },
    name: { type: DataTypes.STRING, allowNull: false },
    createdAt: { type: DataTypes.DATE, allowNull: false },
    updatedAt: { type: DataTypes.DATE, allowNull: false },
  });
};

export const down: Migration = async ({ context: queryInterface }) => {
  await queryInterface.dropTable('users');
};
```

### Run Migrations

```typescript
await umzug.up();                           // all pending
await umzug.up({ step: 2 });               // next 2
await umzug.up({ to: 'migration-name' });   // up to specific
await umzug.down();                          // revert last
await umzug.down({ to: 0 });               // revert all
```

### CLI Usage

```bash
npx tsx migrator.ts up
npx tsx migrator.ts down
npx tsx migrator.ts pending
npx tsx migrator.ts create --name create-posts.ts --folder migrations
```
