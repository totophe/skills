# API, CLI, Events, and Error Handling

## API Methods

### Running Migrations (`up`)

```typescript
// Run all pending migrations
const migrations = await umzug.up();

// Run up to and including a specific migration
await umzug.up({ to: '2024.01.15T10.30.00.create-users' });

// Run a limited number of migrations
await umzug.up({ step: 2 });

// Run specific migrations by name (ignores order)
await umzug.up({
  migrations: ['2024.01.15T10.30.00.create-users', '2024.01.16T09.00.00.create-posts'],
});

// Re-run already executed migrations
await umzug.up({
  migrations: ['2024.01.15T10.30.00.create-users'],
  rerun: 'ALLOW', // or 'SKIP' or 'THROW' (default)
});
```

### Reverting Migrations (`down`)

```typescript
// Revert the last executed migration
const migration = await umzug.down();

// Revert a specific number of migrations
await umzug.down({ step: 2 });

// Revert to a specific migration (inclusive)
await umzug.down({ to: '2024.01.15T10.30.00.create-users' });

// Revert ALL migrations
await umzug.down({ to: 0 });

// Revert specific migrations by name
await umzug.down({
  migrations: ['2024.01.16T09.00.00.create-posts'],
});
```

### Query Methods

```typescript
// Get all pending (not yet executed) migrations
const pending = await umzug.pending();
// Returns: [{ name: '...', path?: '...' }, ...]

// Get all already-executed migrations
const executed = await umzug.executed();
// Returns: [{ name: '...', path?: '...' }, ...]
```

### Rerun Behavior

| Value | Description |
|-------|-------------|
| `'THROW'` | (default) Error if migration is already executed / not executed |
| `'SKIP'` | Silently skip already-executed / not-executed migrations |
| `'ALLOW'` | Re-run regardless of current state |

**Important:** If any migration name in the list is not found at all, an error is thrown and **no** migrations run, even with `rerun: 'ALLOW'`.

## CLI

### Setup

```typescript
// migrator.ts
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

// Enable CLI when run directly
if (require.main === module) {
  umzug.runAsCLI();
}
```

### CLI Commands

```bash
# Apply all pending migrations
npx tsx migrator.ts up

# Revert last migration
npx tsx migrator.ts down

# Apply up to a specific migration
npx tsx migrator.ts up --to 2024.01.15T10.30.00.create-users

# Apply N migrations
npx tsx migrator.ts up --step 2

# Apply specific named migrations
npx tsx migrator.ts up --name 2024.01.15T10.30.00.create-users --name 2024.01.16T09.00.00.create-posts

# Rerun behavior
npx tsx migrator.ts up --rerun ALLOW
npx tsx migrator.ts up --rerun SKIP

# Revert ALL migrations
npx tsx migrator.ts down --to 0

# List pending migrations
npx tsx migrator.ts pending

# List executed migrations
npx tsx migrator.ts executed

# JSON output
npx tsx migrator.ts pending --json
npx tsx migrator.ts executed --json

# Create a new migration
npx tsx migrator.ts create --name create-posts.ts
npx tsx migrator.ts create --name create-posts.ts --folder migrations
npx tsx migrator.ts create --name create-posts.ts --prefix DATE
```

### CLI Command Reference

| Command | Description |
|---------|-------------|
| `up` | Apply pending migrations |
| `down` | Revert migrations |
| `pending` | List pending migrations |
| `executed` | List executed migrations |
| `create` | Create a new migration file |

### CLI Flags for `up` / `down`

| Flag | Description |
|------|-------------|
| `--to NAME` | Run/revert up to and including this migration |
| `--step N` | Number of migrations to apply/revert |
| `--name MIGRATION` | Specific migration(s) (repeatable) |
| `--rerun {THROW,SKIP,ALLOW}` | Behavior for already-run migrations (default: `THROW`) |

## Events

Umzug uses [emittery](https://www.npmjs.com/package/emittery) for type-safe events.

### Per-Migration Events

Payload: `{ name: string, path?: string, context: T }`

| Event | When |
|-------|------|
| `migrating` | Migration is about to run |
| `migrated` | Migration completed successfully |
| `reverting` | Migration is about to be reverted |
| `reverted` | Migration reverted successfully |

### Command-Level Events

Payload: `{ context: T }`

| Event | When |
|-------|------|
| `beforeCommand` | Before any `up`, `down`, `executed`, or `pending` command |
| `afterCommand` | After any command (even if it threw an error) |

### Listening to Events

```typescript
umzug.on('migrating', (ev) => {
  console.log(`Running migration: ${ev.name} (${ev.path})`);
});

umzug.on('migrated', (ev) => {
  console.log(`Completed: ${ev.name}`);
});

umzug.on('reverting', (ev) => {
  console.log(`Reverting: ${ev.name}`);
});

umzug.on('reverted', (ev) => {
  console.log(`Reverted: ${ev.name}`);
});

umzug.on('beforeCommand', (ev) => {
  console.log('Starting command...');
});

umzug.on('afterCommand', (ev) => {
  console.log('Command finished.');
});
```

### Removing Listeners

```typescript
const listener = (ev) => console.log(ev.name);
umzug.on('migrating', listener);
umzug.off('migrating', listener);
```

## Error Handling

Migration errors are wrapped in `MigrationError`, which preserves the original error:

```typescript
import { MigrationError } from 'umzug';

try {
  await umzug.up();
} catch (e) {
  if (e instanceof MigrationError) {
    console.error(`Migration "${e.migration}" failed:`);
    console.error(e.cause); // original error
  }
  throw e;
}
```

`MigrationError` extends `Error` and includes:
- Migration metadata (name, path)
- The original error as `.cause`
- Uses [verror](https://npmjs.com/package/verror) for error chaining

## Programmatic Seeder Pattern

Use a separate Umzug instance for seeders:

```typescript
const seeder = new Umzug({
  migrations: { glob: 'seeders/*.ts' },
  context: sequelize.getQueryInterface(),
  storage: new SequelizeStorage({
    sequelize,
    modelName: 'SeederMeta', // separate tracking table
  }),
  logger: console,
});

// Run all seeders
await seeder.up();

// Revert all seeders
await seeder.down({ to: 0 });
```

## Wrapping in Transactions

Umzug does not wrap migrations in transactions automatically. For transactional migrations, handle it inside each migration or use events:

```typescript
// Per-migration transactions
export const up: Migration = async ({ context: queryInterface }) => {
  const transaction = await queryInterface.sequelize.startUnmanagedTransaction();
  try {
    await queryInterface.createTable('users', { /* ... */ }, { transaction });
    await queryInterface.addIndex('users', ['email'], { transaction });
    await transaction.commit();
  } catch (error) {
    await transaction.rollback();
    throw error;
  }
};
```

Or using Sequelize v7 managed transactions:

```typescript
export const up: Migration = async ({ context: queryInterface }) => {
  await queryInterface.sequelize.transaction(async () => {
    // CLS auto-propagation passes the transaction to all queries
    await queryInterface.createTable('users', { /* ... */ });
    await queryInterface.addIndex('users', ['email']);
  });
};
```

**Note:** DDL transactions are not supported by all databases. PostgreSQL supports transactional DDL; MySQL/MariaDB do not (DDL commits implicitly).
