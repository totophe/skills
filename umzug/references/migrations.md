# Migrations — Configuration, Resolvers, Ordering, Templates

## Migration File Format

Every migration exports `up` and optionally `down`:

```typescript
// CommonJS
module.exports = {
  async up({ context: queryInterface }) { /* ... */ },
  async down({ context: queryInterface }) { /* ... */ },
};

// ESM / TypeScript
export const up = async ({ context: queryInterface }) => { /* ... */ };
export const down = async ({ context: queryInterface }) => { /* ... */ };
```

The `context` value matches whatever was passed to the Umzug constructor.

## Glob-Based Discovery

```typescript
const umzug = new Umzug({
  migrations: { glob: 'migrations/*.ts' },
  context: sequelize.getQueryInterface(),
  storage: new SequelizeStorage({ sequelize }),
  logger: console,
});
```

### Multiple Directories

```typescript
const umzug = new Umzug({
  migrations: {
    glob: '{migrations/*.ts,seeds/*.ts}',
  },
  // ...
});
```

### Glob with Options

```typescript
const umzug = new Umzug({
  migrations: {
    glob: ['migrations/*.ts', { cwd: __dirname, ignore: ['**/*.test.ts'] }],
  },
  // ...
});
```

## Direct Migration List

Instead of file discovery, provide migrations inline:

```typescript
const umzug = new Umzug({
  migrations: [
    {
      name: '00-create-users',
      async up({ context: queryInterface }) {
        await queryInterface.createTable('users', { /* ... */ });
      },
      async down({ context: queryInterface }) {
        await queryInterface.dropTable('users');
      },
    },
    {
      name: '01-create-posts',
      async up({ context: queryInterface }) {
        await queryInterface.createTable('posts', { /* ... */ });
      },
      async down({ context: queryInterface }) {
        await queryInterface.dropTable('posts');
      },
    },
  ],
  context: sequelize.getQueryInterface(),
  logger: console,
});
```

## Custom Resolver

Override how migration files are loaded:

```typescript
const umzug = new Umzug({
  migrations: {
    glob: 'migrations/*.js',
    resolve: ({ name, path, context }) => {
      const migration = require(path!);
      return {
        name,
        up: async () => migration.up(context),
        down: async () => migration.down(context),
      };
    },
  },
  context: sequelize.getQueryInterface(),
  storage: new SequelizeStorage({ sequelize }),
  logger: console,
});
```

### Resolver for sequelize-cli Format

Migrations written for `sequelize-cli` pass `(queryInterface, Sequelize)` as positional args:

```typescript
import { DataTypes } from '@sequelize/core';

const umzug = new Umzug({
  migrations: {
    glob: 'migrations/*.js',
    resolve: ({ name, path, context }) => {
      const migration = require(path!);
      return {
        name,
        up: async () => migration.up(context, DataTypes),
        down: async () => migration.down(context, DataTypes),
      };
    },
  },
  context: sequelize.getQueryInterface(),
  storage: new SequelizeStorage({ sequelize }),
  logger: console,
});
```

## SQL File Migrations

```typescript
import fs from 'node:fs';

const umzug = new Umzug({
  migrations: {
    glob: 'migrations/*.up.sql',
    resolve: ({ name, path, context: sequelize }) => ({
      name,
      up: async () => {
        const sql = fs.readFileSync(path!).toString();
        return sequelize.query(sql);
      },
      down: async () => {
        const sql = fs.readFileSync(path!.replace('.up.sql', '.down.sql')).toString();
        return sequelize.query(sql);
      },
    }),
  },
  context: sequelize, // passing full sequelize instance for .query()
  logger: console,
});
```

## Mixed File Types

Handle `.ts`, `.js`, and `.sql` files together using the default resolver as fallback:

```typescript
const umzug = new Umzug({
  migrations: {
    glob: 'migrations/*.{ts,js,up.sql}',
    resolve: (params) => {
      if (!params.path!.endsWith('.sql')) {
        return Umzug.defaultResolver(params);
      }
      return {
        name: params.name,
        up: async () => {
          const sql = fs.readFileSync(params.path!).toString();
          return params.context.query(sql);
        },
        down: async () => {
          const sql = fs.readFileSync(params.path!.replace('.up.sql', '.down.sql')).toString();
          return params.context.query(sql);
        },
      };
    },
  },
  context: sequelize,
  logger: console,
});
```

`Umzug.defaultResolver` is the built-in resolver — use it as a fallback when customizing only some file types.

## Migration Ordering

Files discovered via glob are sorted **lexicographically by full path**. Important implications:

- `m1.js`, `m10.js`, `m11.js`, ..., `m2.js` — numeric names sort incorrectly without zero-padding
- Multi-directory globs sort by full path: `one/m1.js` < `three/m3.js` < `two/m2.js`
- **Best practice**: keep all migrations in one folder with timestamp-prefixed names

### Recommended Naming Conventions

```
2024.01.15T10.30.00.create-users.ts       # TIMESTAMP prefix (default in CLI)
2024-01-15-create-users.ts                 # DATE prefix
0001-create-users.ts                       # Zero-padded numeric
```

### Custom Sort Order

```typescript
const parent = new Umzug({
  migrations: { glob: 'migrations/**/*.ts' },
  context: sequelize.getQueryInterface(),
});

const umzug = new Umzug({
  ...parent.options,
  migrations: async (ctx) =>
    (await parent.migrations()).sort((a, b) => b.path!.localeCompare(a.path!)),
});
```

## Creating Migrations

### Via CLI

```bash
# Creates timestamped file next to most recent migration
node migrator create --name create-users.ts

# First migration (must specify folder)
node migrator create --name create-users.ts --folder migrations

# With date prefix instead of timestamp
node migrator create --name create-users.ts --prefix DATE

# No prefix
node migrator create --name create-users.ts --prefix NONE
```

### Via API

```typescript
await umzug.create({ name: 'create-users.ts' });
```

### Custom Template

```typescript
import fs from 'node:fs';

const umzug = new Umzug({
  migrations: { glob: 'migrations/*.ts' },
  create: {
    template: (filepath) => [
      [filepath, fs.readFileSync('path/to/template.ts').toString()],
    ],
  },
  // ...
});
```

The template function receives the target filepath and returns an array of `[filepath, content]` tuples. This allows generating multiple files per migration (e.g., `.up.sql` + `.down.sql`):

```typescript
create: {
  template: (filepath) => [
    [filepath.replace('.ts', '.up.sql'), '-- up migration\n'],
    [filepath.replace('.ts', '.down.sql'), '-- down migration\n'],
  ],
},
```

### Create CLI Flags

| Flag | Description |
|------|-------------|
| `--name NAME` | Filename (e.g., `create-users.ts`) |
| `--prefix {TIMESTAMP,DATE,NONE}` | Prefix format (default: `TIMESTAMP`) |
| `--folder PATH` | Target directory (required for first migration) |
| `--allow-extension EXT` | Allow non-default extensions |
| `--skip-verify` | Skip post-creation detection check |
| `--allow-confusing-ordering` | Disable ordering safety checks |

Built-in templates exist for `.js`, `.ts`, `.cjs`, `.mjs`, and `.sql`.

## Common Sequelize v7 Migration Operations

Using `queryInterface` from `@sequelize/core`:

```typescript
import type { Migration } from '../migrator';
import { DataTypes } from '@sequelize/core';

export const up: Migration = async ({ context: queryInterface }) => {
  // Create table
  await queryInterface.createTable('posts', {
    id: { type: DataTypes.INTEGER, primaryKey: true, autoIncrement: true },
    title: { type: DataTypes.STRING(255), allowNull: false },
    body: { type: DataTypes.TEXT, allowNull: true },
    userId: {
      type: DataTypes.INTEGER,
      allowNull: false,
      references: { table: 'users', key: 'id' },
      onUpdate: 'CASCADE',
      onDelete: 'CASCADE',
    },
    createdAt: { type: DataTypes.DATE, allowNull: false },
    updatedAt: { type: DataTypes.DATE, allowNull: false },
  });

  // Add index
  await queryInterface.addIndex('posts', ['userId']);

  // Add column
  await queryInterface.addColumn('users', 'role', {
    type: DataTypes.STRING,
    defaultValue: 'user',
  });

  // Rename column
  await queryInterface.renameColumn('users', 'name', 'fullName');

  // Change column type
  await queryInterface.changeColumn('users', 'email', {
    type: DataTypes.STRING(512),
    allowNull: false,
  });

  // Remove column
  await queryInterface.removeColumn('users', 'tempField');

  // Add unique constraint
  await queryInterface.addConstraint('users', {
    fields: ['email'],
    type: 'unique',
    name: 'unique_email',
  });

  // Raw query
  await queryInterface.sequelize.query(
    `UPDATE users SET role = 'admin' WHERE id = 1`
  );
};

export const down: Migration = async ({ context: queryInterface }) => {
  await queryInterface.removeConstraint('users', 'unique_email');
  await queryInterface.renameColumn('users', 'fullName', 'name');
  await queryInterface.removeColumn('users', 'role');
  await queryInterface.removeIndex('posts', ['userId']);
  await queryInterface.dropTable('posts');
};
```

### QueryInterface Method Reference

| Method | Description |
|--------|-------------|
| `createTable(name, attributes)` | Create a new table |
| `dropTable(name)` | Drop a table |
| `addColumn(table, column, definition)` | Add a column |
| `removeColumn(table, column)` | Remove a column |
| `changeColumn(table, column, definition)` | Alter column type/options |
| `renameColumn(table, oldName, newName)` | Rename a column |
| `addIndex(table, columns, options?)` | Add an index |
| `removeIndex(table, columns)` | Remove an index |
| `addConstraint(table, options)` | Add a constraint |
| `removeConstraint(table, constraintName)` | Remove a constraint |
| `renameTable(before, after)` | Rename a table |
| `showAllTables()` | List all tables |
| `describeTable(name)` | Describe table columns |
| `createSchema(name)` | Create a schema (PostgreSQL) |
| `dropSchema(name)` | Drop a schema |
| `sequelize.query(sql)` | Execute raw SQL |
| `bulkInsert(table, records)` | Insert multiple rows |
| `bulkDelete(table, where)` | Delete rows matching condition |
| `bulkUpdate(table, values, where)` | Update rows matching condition |
