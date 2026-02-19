# Storage — Backends and Custom Storage

Umzug uses storage backends to track which migrations have been executed. The storage records migration names when they run and removes them on revert.

## SequelizeStorage (Recommended for Sequelize v7)

Stores migration records in a database table (default: `SequelizeMeta`).

```typescript
import { Umzug, SequelizeStorage } from 'umzug';

const umzug = new Umzug({
  migrations: { glob: 'migrations/*.ts' },
  context: sequelize.getQueryInterface(),
  storage: new SequelizeStorage({ sequelize }),
  logger: console,
});
```

### Configuration Options

| Option | Default | Description |
|--------|---------|-------------|
| `sequelize` | — | Sequelize instance (required unless `model` is provided) |
| `model` | — | Custom Sequelize model (alternative to `sequelize`) |
| `modelName` | `'SequelizeMeta'` | Model name for the tracking table |
| `tableName` | model name | Override the table name |
| `columnName` | `'name'` | Column storing migration names |
| `columnType` | `DataTypes.STRING` | Column data type |
| `timestamps` | `false` | Add `createdAt`/`updatedAt` columns |
| `schema` | `undefined` | Database schema (PostgreSQL) |

### With Custom Table Name

```typescript
new SequelizeStorage({
  sequelize,
  tableName: 'migrations_log',
  columnName: 'migration_name',
});
```

### With Timestamps

```typescript
new SequelizeStorage({
  sequelize,
  timestamps: true, // adds createdAt and updatedAt
});
```

### With Schema (PostgreSQL)

```typescript
new SequelizeStorage({
  sequelize,
  schema: 'my_schema',
});
```

### With Custom Model

Instead of letting SequelizeStorage create its own model, pass one explicitly:

```typescript
import { Model, DataTypes, InferAttributes, InferCreationAttributes, CreationOptional } from '@sequelize/core';
import { Attribute, PrimaryKey, NotNull } from '@sequelize/core/decorators-legacy';

class MigrationMeta extends Model<InferAttributes<MigrationMeta>, InferCreationAttributes<MigrationMeta>> {
  @Attribute(DataTypes.STRING)
  @PrimaryKey
  @NotNull
  declare name: string;

  declare createdAt: CreationOptional<Date>;
  declare updatedAt: CreationOptional<Date>;
}

const umzug = new Umzug({
  storage: new SequelizeStorage({ model: MigrationMeta }),
  // ...
});
```

### MySQL/MariaDB Charset

SequelizeStorage automatically applies `charset: 'utf8'` and `collate: 'utf8_unicode_ci'` for MySQL and MariaDB dialects. For longer migration names or utf8mb4, override `columnType`:

```typescript
new SequelizeStorage({
  sequelize,
  columnType: DataTypes.STRING(190), // safe for utf8mb4 + InnoDB key limit
});
```

## JSONStorage

Stores executed migration names in a JSON file. Default file: `umzug.json`.

```typescript
import { Umzug, JSONStorage } from 'umzug';

const umzug = new Umzug({
  migrations: { glob: 'migrations/*.ts' },
  storage: new JSONStorage(), // writes to ./umzug.json
  logger: console,
});
```

### Custom Path

```typescript
new JSONStorage({ path: './data/migrations.json' });
```

## MemoryStorage

Stores migration state in-memory. Useful for testing:

```typescript
import { Umzug, memoryStorage } from 'umzug';

const umzug = new Umzug({
  migrations: { glob: 'migrations/*.ts' },
  storage: memoryStorage(),
  logger: console,
});
```

## MongoDBStorage

Creates a `migrations` collection in MongoDB.

```typescript
import { Umzug, MongoDBStorage } from 'umzug';
import { MongoClient } from 'mongodb';

const client = new MongoClient('mongodb://localhost:27017');
await client.connect();

const umzug = new Umzug({
  migrations: { glob: 'migrations/*.ts' },
  storage: new MongoDBStorage({
    collection: client.db('mydb').collection('migrations'),
  }),
  logger: console,
});
```

### Or with Connection String

```typescript
new MongoDBStorage({
  connection: 'mongodb://localhost:27017',
  collectionName: 'migrations',
});
```

## Custom Storage

Implement the `UmzugStorage` interface:

```typescript
import { UmzugStorage } from 'umzug';

class CustomStorage implements UmzugStorage {
  private executed: string[] = [];

  async logMigration({ name }: { name: string }): Promise<void> {
    this.executed.push(name);
  }

  async unlogMigration({ name }: { name: string }): Promise<void> {
    this.executed = this.executed.filter((n) => n !== name);
  }

  async executed(): Promise<string[]> {
    return this.executed;
  }
}

const umzug = new Umzug({
  storage: new CustomStorage(),
  // ...
});
```

### Storage Interface

| Method | Signature | Description |
|--------|-----------|-------------|
| `logMigration` | `({ name: string, context?: T }) => Promise<void>` | Record a migration as executed |
| `unlogMigration` | `({ name: string, context?: T }) => Promise<void>` | Remove a migration from executed list |
| `executed` | `({ context?: T }) => Promise<string[]>` | Return all executed migration names |
