# Sequelize v7 — Advanced Features

## Transactions

### Managed Transactions (Recommended)

Auto commit on success, auto rollback on error:

```typescript
try {
  const result = await sequelize.transaction(async () => {
    const user = await User.create({ firstName: 'Abraham', lastName: 'Lincoln' });
    await Profile.create({ userId: user.id, bio: 'President' });
    return user;
  });
  // Transaction committed, result available
} catch {
  // Transaction already rolled back automatically
}
```

CLS is **enabled by default** in v7 — all queries inside the callback automatically use the transaction. No need to pass `{ transaction }` manually.

### Unmanaged Transactions

Manual commit/rollback required:

```typescript
const transaction = await sequelize.startUnmanagedTransaction();
try {
  const user = await User.create({ firstName: 'Bart' }, { transaction });
  await transaction.commit();
} catch (error) {
  await transaction.rollback();
}
```

Unmanaged transactions are incompatible with CLS — always pass `{ transaction }` explicitly.

### Nested Transactions

```typescript
import { TransactionNestMode } from '@sequelize/core';

await sequelize.transaction(async () => {
  // Savepoint mode: child errors roll back savepoint only, parent continues
  await sequelize.transaction({ nestMode: TransactionNestMode.savepoint }, async () => {
    // ...
  });
});
```

| Mode | Behavior |
|------|----------|
| `TransactionNestMode.reuse` | (default) Child reuses parent transaction |
| `TransactionNestMode.savepoint` | Creates savepoint inside parent |
| `TransactionNestMode.separate` | Entirely new independent transaction |

### Isolation Levels

```typescript
import { IsolationLevel } from '@sequelize/core';

await sequelize.transaction(
  { isolationLevel: IsolationLevel.SERIALIZABLE },
  async () => { /* ... */ },
);

// Global default
new Sequelize({ isolationLevel: IsolationLevel.SERIALIZABLE });
```

Available levels: `READ_UNCOMMITTED`, `READ_COMMITTED`, `REPEATABLE_READ`, `SERIALIZABLE`.

### Transaction Hooks

```typescript
await sequelize.transaction(async t => {
  t.afterCommit(() => { /* runs after successful commit */ });
  t.afterRollback(() => { /* runs after rollback */ });
  t.afterTransaction(() => { /* runs on both commit and rollback (finally) */ });
});
```

### Row Locks

```typescript
const users = await User.findAll({
  limit: 1,
  lock: true,
  transaction: t,
});

// Skip locked rows
const users = await User.findAll({
  limit: 1,
  lock: true,
  skipLocked: true,
  transaction: t,
});
```

### Disabling CLS

```typescript
const sequelize = new Sequelize({ disableClsTransactions: true });
// Must now pass { transaction } to every query manually
```

### Getting Current Transaction

```typescript
sequelize.getCurrentClsTransaction(); // Returns active transaction or undefined
```

### Escaping the Transaction

```typescript
// Run a query outside the current CLS transaction
await User.findAll({ transaction: null });
```

### Constraint Checking (PostgreSQL)

```typescript
import { ConstraintChecking } from '@sequelize/core';

await sequelize.transaction(
  { constraintChecking: ConstraintChecking.DEFERRED },
  async () => { /* constraints checked at commit */ },
);
```

### Read-Only Transactions (Read Replication)

```typescript
await sequelize.transaction({ readOnly: true }, async () => { /* routed to read replica */ });
```

## Hooks

### Model Hook Decorators

```typescript
import {
  BeforeCreate, AfterCreate,
  BeforeUpdate, AfterUpdate,
  BeforeSave, AfterSave,
  BeforeDestroy, AfterDestroy,
  BeforeFind, AfterFind,
  BeforeValidate, AfterValidate,
  BeforeBulkCreate, AfterBulkCreate,
  BeforeBulkUpdate, AfterBulkUpdate,
  BeforeBulkDestroy, AfterBulkDestroy,
  BeforeSync, AfterSync,
} from '@sequelize/core/decorators-legacy';

class User extends Model {
  @BeforeCreate
  static async hashPassword(user: User) {
    user.password = await bcrypt.hash(user.password, 10);
  }

  @AfterCreate
  static logCreation(user: User) {
    console.log(`User ${user.id} created`);
  }

  @BeforeFind
  static logFindAll() {
    console.log('findAll called on User');
  }
}
```

### Hook Registration via `hooks` Property

```typescript
// On model
User.hooks.addListener('beforeCreate', async (user, options) => {
  user.password = await bcrypt.hash(user.password, 10);
});

// On Sequelize instance (fires for ALL models)
sequelize.hooks.addListener('beforeFind', () => {
  console.log('findAll called on a model');
});
```

### Constructor-Based Registration

```typescript
const sequelize = new Sequelize({
  hooks: {
    beforeDefine: () => { console.log('Model being defined'); },
  },
});
```

### Available Hooks

| Hook | Async | Triggered When |
|------|-------|----------------|
| `beforeCreate` / `afterCreate` | Yes | `Model#save()` (new model) |
| `beforeUpdate` / `afterUpdate` | Yes | `Model#update()` or `Model#save()` (existing) |
| `beforeSave` / `afterSave` | Yes | Any `save()` or `update()` |
| `beforeDestroy` / `afterDestroy` | Yes | `Model#destroy()` |
| `beforeBulkCreate` / `afterBulkCreate` | Yes | `Model.bulkCreate()` |
| `beforeBulkUpdate` / `afterBulkUpdate` | Yes | `Model.update()` |
| `beforeBulkDestroy` / `afterBulkDestroy` | Yes | `Model.destroy()` |
| `beforeFind` / `afterFind` | Yes | `Model.findAll()` (and methods that use it) |
| `beforeCount` | Yes | `Model.count()` |
| `beforeValidate` / `afterValidate` | Yes | Model validation |
| `beforeUpsert` / `afterUpsert` | Yes | `Model.upsert()` |
| `beforeSync` / `afterSync` | Yes | `Model.sync()` |
| `beforeConnect` / `afterConnect` | Yes | New DB connection |
| `beforeDisconnect` / `afterDisconnect` | Yes | Connection closed |
| `beforePoolAcquire` / `afterPoolAcquire` | Yes | Pool connection acquired |
| `beforeQuery` / `afterQuery` | Yes | SQL query runs |
| `beforeRestore` / `afterRestore` | Yes | Paranoid model restore |
| `beforeAssociate` / `afterAssociate` | No | Association declared |

### Important Hook Behaviors

- Hooks run **in series** (each waits for the previous)
- Static methods (`Model.destroy`, `Model.update`) only trigger **bulk** hooks by default
- Use `individualHooks: true` to also trigger per-instance hooks (discouraged for performance)
- Hooks do NOT fire for: `ON DELETE CASCADE`, raw queries, QueryInterface methods
- Use `options.transaction` inside hooks to stay within the same transaction

### Removing Hooks

```typescript
User.hooks.removeListener('afterCreate', listenerFn);
User.hooks.removeListener('afterCreate', 'hookName');
```

### Hooks with Transactions

```typescript
User.hooks.addListener('afterCreate', async (user, options) => {
  await AuditLog.create(
    { action: 'create', userId: user.id },
    { transaction: options.transaction }, // Stay in same transaction
  );
});
```

## Scopes

> **Warning:** Scopes are described as "a fragile feature" — not recommended beyond simple use cases.

### Defining Scopes

```typescript
Project.init({ /* attributes */ }, {
  defaultScope: {
    where: { active: true },
  },
  scopes: {
    deleted: { where: { deleted: true } },
    activeUsers: { include: [{ model: User, where: { active: true } }] },
    accessLevel(value) {
      return { where: { accessLevel: { [Op.gte]: value } } };
    },
  },
});
```

### Using Scopes

```typescript
// Default scope always applied
await Project.findAll();
// → WHERE active = true

// Named scope
await Project.scope('deleted').findAll();
// → WHERE deleted = true

// Multiple scopes (merged with AND)
await Project.scope('deleted', 'activeUsers').findAll();

// Function scope with arguments
await Project.scope({ method: ['accessLevel', 19] }).findAll();

// Remove default scope
await Project.unscoped().findAll();
await Project.scope(null).findAll();
```

### Scope Merge Rules

| Option | Behavior |
|--------|----------|
| `where` | Merged with AND |
| `include` | Merged recursively by model |
| `attributes.exclude` | Always preserved |
| `limit`, `offset`, `order`, `lock`, `raw` | Overwritten by later scope |

## Paranoid Models (Soft Delete)

### Setup

```typescript
import { DeletedAt } from '@sequelize/core/decorators-legacy';

class User extends Model<InferAttributes<User>, InferCreationAttributes<User>> {
  @DeletedAt
  declare deletedAt: Date | null;
}
```

Using `@DeletedAt` automatically enables paranoid mode.

### Behavior

```typescript
// Soft-delete (sets deletedAt timestamp)
await post.destroy();

// Hard-delete (actually removes row)
await post.destroy({ force: true });

// Restore soft-deleted record
await post.restore();
await Post.restore({ where: { likes: { [Op.gt]: 100 } } });

// Queries automatically exclude soft-deleted records
await Post.findByPk(123);                        // null if soft-deleted
await Post.findByPk(123, { paranoid: false });   // includes soft-deleted

// Include soft-deleted associations
User.findAll({
  include: [{ association: 'projects', paranoid: false }],
});
```

## Optimistic Locking

### Setup

```typescript
import { Version } from '@sequelize/core/decorators-legacy';

class User extends Model<InferAttributes<User>, InferCreationAttributes<User>> {
  @Version
  declare version: CreationOptional<number>;
}
```

Sequelize checks the version column before any modification. A mismatch throws `OptimisticLockError`.

## Connection Pool

### Configuration

```typescript
const sequelize = new Sequelize({
  pool: {
    max: 5,       // Maximum active connections (default: 5)
    min: 0,       // Minimum connections maintained (default: 0)
    acquire: 30000, // Timeout ms before throwing error (default: 30000)
    idle: 10000,    // Time ms before idle connection released (default: 10000)
  },
});
```

> Pool is **not shared** between Sequelize instances. Account for multiple processes.

### Monitoring

```typescript
const pool = sequelize.connectionManager.pool.write;
console.log(pool.size);       // Total connections
console.log(pool.available);  // Ready connections
console.log(pool.using);      // Active connections
console.log(pool.waiting);    // Queued requests
```

### Timing Acquisition

```typescript
const acquireAttempts = new WeakMap();

sequelize.hooks.addListener('beforePoolAcquire', options => {
  acquireAttempts.set(options, Date.now());
});

sequelize.hooks.addListener('afterPoolAcquire', (_connection, options) => {
  console.log(`Acquired in ${Date.now() - acquireAttempts.get(options)}ms`);
});
```

### Troubleshooting `ConnectionAcquireTimeoutError`

1. Too many concurrent requests → increase `max`
2. Slow queries → profile and optimize
3. Idle/uncommitted transactions → commit/rollback properly
4. Slow async in transactions → use `AbortSignal.timeout()`

## Read Replication

```typescript
const sequelize = new Sequelize({
  dialect: MySqlDialect,
  database: 'mydb',
  replication: {
    read: [
      { host: '8.8.8.8', user: 'read1', password: process.env.READ_DB_1_PW },
      { host: '9.9.9.9', user: 'read2', password: process.env.READ_DB_2_PW },
    ],
    write: {
      host: '1.1.1.1',
      user: 'write',
      password: process.env.WRITE_DB_PW,
    },
  },
});
```

- SELECT queries use read pool (round-robin)
- Write queries use write pool
- Top-level options (port, database) propagate to all replicas

## Migrations

### Using @sequelize/cli

The official CLI for creating and running migrations.

```bash
npm i @sequelize/cli
```

### Alternative: Umzug

The migration framework used under the hood by Sequelize CLI:

```bash
npm i umzug
```

### Strategy: Database Diff Tool

Compare current schema against desired, generate diff, apply on release. Tool: [pg-diff](https://michaelsogos.github.io/pg-diff/).

> **Important:** Do not use `sequelize.sync()` in production — it drops and recreates tables.

## Model Synchronization

For **development only**:

```typescript
// Sync all models
await sequelize.sync();

// Sync with force (DROP + CREATE)
await sequelize.sync({ force: true });

// Sync with alter (ALTER TABLE to match model)
await sequelize.sync({ alter: true });

// Sync single model
await User.sync();
```

## v6 → v7 Migration Checklist

- [ ] Update package: `sequelize` → `@sequelize/core`
- [ ] Install dialect package separately (e.g., `@sequelize/postgres`)
- [ ] Update constructor to single options object
- [ ] Move `dialectOptions` to top level
- [ ] Replace `Model.init()` with decorator-based definitions (recommended)
- [ ] Update association names to camelCase (`userId` not `UserId`)
- [ ] Replace `sequelize.transaction(callback)` for unmanaged → `sequelize.startUnmanagedTransaction()`
- [ ] Remove CLS setup code (now built-in)
- [ ] Update `BIGINT`/`DECIMAL` consumers to expect strings
- [ ] Handle JSON null change (now stores JSON `'null'`, not SQL `NULL`)
- [ ] Replace `attributes: ['*']` with proper column selection
- [ ] Replace `where: primaryKeyValue` with `findByPk()`
- [ ] Update `Op.not` usage (now always `NOT (x)`)
- [ ] Replace `fn()`, `col()`, `cast()`, `literal()` with `sql.*` equivalents
- [ ] Minimum: Node >= 18, TypeScript >= 5.0
