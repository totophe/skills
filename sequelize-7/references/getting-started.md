# Sequelize v7 — Getting Started

## Installation

```bash
# Core package
npm i @sequelize/core@alpha

# Install the dialect package for your database
npm i @sequelize/postgres    # PostgreSQL
npm i @sequelize/mysql       # MySQL
npm i @sequelize/mariadb     # MariaDB
npm i @sequelize/sqlite3     # SQLite
npm i @sequelize/mssql       # Microsoft SQL Server
npm i @sequelize/db2         # DB2 for LUW
npm i @sequelize/db2-ibmi    # DB2 for IBM i
npm i @sequelize/snowflake   # Snowflake
```

> **Important:** v7 uses `@sequelize/core`, not `sequelize` (v6 package name).

## Connecting to a Database

Create a `Sequelize` instance with a dialect class and connection options:

```typescript
import { Sequelize } from '@sequelize/core';
import { PostgresDialect } from '@sequelize/postgres';

const sequelize = new Sequelize({
  dialect: PostgresDialect,
  host: 'localhost',
  port: 5432,
  database: 'mydb',
  user: 'user',
  password: 'pass',
  models: [User, Post], // Register models here
});
```

### SQLite Example

```typescript
import { Sequelize } from '@sequelize/core';
import { SqliteDialect } from '@sequelize/sqlite3';

const sequelize = new Sequelize({
  dialect: SqliteDialect,
});
```

### v7 Constructor Changes

- Only accepts a **single options object** (no positional URL/database/user/pass args)
- URL connections use the `url` option: `new Sequelize({ dialect: PostgresDialect, url: 'postgres://...' })`
- `dialectOptions` removed — put those settings at top level

## Testing the Connection

```typescript
try {
  await sequelize.authenticate();
  console.log('Connection established successfully.');
} catch (error) {
  console.error('Unable to connect:', error);
}
```

## Closing the Connection

```typescript
await sequelize.close();
```

After `close()`, no new connections can be opened. Create a new `Sequelize` instance to reconnect.

## TypeScript Configuration

Sequelize v7 has built-in TypeScript support. Requirements:
- TypeScript >= 5.0
- Install `@types/node` matching your Node.js version

In `tsconfig.json`, set one of:
- `"moduleResolution": "node16"` or `"nodenext"` or `"bundler"`
- **OR** `"resolvePackageJsonExports": true`

## ESM vs CommonJS

**ESM (recommended):**
```typescript
import { Sequelize, Op, Model, DataTypes } from '@sequelize/core';
```

**CommonJS:**
```javascript
const { Sequelize, Op, Model, DataTypes } = require('@sequelize/core');
```

## Logging

```typescript
const sequelize = new Sequelize({
  dialect: SqliteDialect,
  logging: false,              // Disable logging (default)
  logging: console.log,        // Log to console
  logging: (...msg) => logger.debug(msg), // Custom logger
});
```

The `logging` option is also available on individual model methods for per-query logging.

## Registering Models

### Explicit registration (recommended)

```typescript
const sequelize = new Sequelize({
  dialect: SqliteDialect,
  models: [User, Post, Comment],
});
```

### Dynamic loading via glob

```typescript
import { importModels } from '@sequelize/core';
import { fileURLToPath } from 'node:url';
import { dirname } from 'node:path';

const __dirname = dirname(fileURLToPath(import.meta.url));

const sequelize = new Sequelize({
  dialect: SqliteDialect,
  models: await importModels(__dirname + '/**/*.model.{ts,js}'),
});
```

`importModels` uses `fast-glob` and ESM `import()`.
