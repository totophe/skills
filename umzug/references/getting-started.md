# Getting Started — Umzug with Sequelize v7

## Installation

```bash
npm install umzug
```

Umzug v3.x is the current stable release (3.8.2+). It is framework-agnostic but has built-in Sequelize storage support.

## Minimal Setup (TypeScript + Sequelize v7)

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

// Export migration type helper for use in migration files
export type Migration = typeof umzug._types.migration;
```

## Minimal Setup (JavaScript)

```javascript
const { Sequelize } = require('@sequelize/core');
const { PostgresDialect } = require('@sequelize/postgres');
const { Umzug, SequelizeStorage } = require('umzug');

const sequelize = new Sequelize({
  dialect: PostgresDialect,
  host: 'localhost',
  port: 5432,
  database: 'mydb',
  user: 'user',
  password: 'pass',
});

const umzug = new Umzug({
  migrations: { glob: 'migrations/*.js' },
  context: sequelize.getQueryInterface(),
  storage: new SequelizeStorage({ sequelize }),
  logger: console,
});
```

## Sequelize v7 Differences for Migrations

Key differences from v6 that affect migration setup:

| Area | v6 | v7 |
|------|----|----|
| Package import | `require('sequelize')` | `require('@sequelize/core')` |
| Dialect setup | `dialect: 'postgres'` (string) | `dialect: PostgresDialect` (class import) |
| Constructor | Accepts URL string as first arg | Single options object only |
| `dialectOptions` | Nested under `dialectOptions` | Top-level options |
| DataTypes import | `const { DataTypes } = require('sequelize')` | `const { DataTypes } = require('@sequelize/core')` |
| `QueryInterface` | `sequelize.getQueryInterface()` | Same API, still works |

## Migration File with TypeScript Type Helper

```typescript
// migrator.ts — entry point
import { Sequelize } from '@sequelize/core';
import { PostgresDialect } from '@sequelize/postgres';
import { Umzug, SequelizeStorage } from 'umzug';

const sequelize = new Sequelize({
  dialect: PostgresDialect,
  host: 'localhost',
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

// Run as CLI if executed directly
if (require.main === module) {
  umzug.runAsCLI();
}
```

```typescript
// migrations/2024.01.01T00.00.00.create-users.ts
import type { Migration } from '../migrator';
import { DataTypes } from '@sequelize/core';

export const up: Migration = async ({ context: queryInterface }) => {
  await queryInterface.createTable('users', {
    id: {
      type: DataTypes.INTEGER,
      allowNull: false,
      primaryKey: true,
      autoIncrement: true,
    },
    name: {
      type: DataTypes.STRING,
      allowNull: false,
    },
    email: {
      type: DataTypes.STRING,
      allowNull: true,
    },
    createdAt: {
      type: DataTypes.DATE,
      allowNull: false,
    },
    updatedAt: {
      type: DataTypes.DATE,
      allowNull: false,
    },
  });
};

export const down: Migration = async ({ context: queryInterface }) => {
  await queryInterface.dropTable('users');
};
```

## Running with ts-node

If not using a bundler, register `ts-node` at the top of your entry file:

```typescript
require('ts-node/register');
```

Or use `tsx` to run directly:

```bash
npx tsx migrator.ts up
npx tsx migrator.ts down
npx tsx migrator.ts pending
```

## Logging

```typescript
// Full console logging
const umzug = new Umzug({ logger: console, /* ... */ });

// Disable logging
const umzug = new Umzug({ logger: undefined, /* ... */ });

// Custom logger
const umzug = new Umzug({
  logger: {
    info: (message) => myLogger.info(message),
    warn: (message) => myLogger.warn(message),
    error: (message) => myLogger.error(message),
    debug: (message) => myLogger.debug(message),
  },
  /* ... */
});
```

## Context Pattern

The `context` option in the Umzug constructor is passed to every migration's `up` and `down` functions. For Sequelize, this is typically `queryInterface`:

```typescript
const umzug = new Umzug({
  context: sequelize.getQueryInterface(),
  // ...
});

// In migration files, destructure it:
export const up: Migration = async ({ context: queryInterface }) => {
  // queryInterface is the Sequelize QueryInterface
};
```

You can pass any value as context — it doesn't have to be `queryInterface`. For example, you could pass the entire `sequelize` instance:

```typescript
const umzug = new Umzug({
  context: sequelize,
  // ...
});

// In migration:
export const up = async ({ context: sequelize }) => {
  const qi = sequelize.getQueryInterface();
  // or use sequelize.query() directly
};
```
