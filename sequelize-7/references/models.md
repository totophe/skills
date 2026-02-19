# Sequelize v7 — Models

## Defining a Model (TypeScript + Decorators)

Models extend `Model` and use decorators from `@sequelize/core/decorators-legacy`.

```typescript
import { Model, InferAttributes, InferCreationAttributes, CreationOptional, DataTypes } from '@sequelize/core';
import { Attribute, PrimaryKey, AutoIncrement, NotNull, Default, Table } from '@sequelize/core/decorators-legacy';

export class User extends Model<InferAttributes<User>, InferCreationAttributes<User>> {
  @Attribute(DataTypes.INTEGER)
  @PrimaryKey
  @AutoIncrement
  declare id: CreationOptional<number>;

  @Attribute(DataTypes.STRING)
  @NotNull
  declare firstName: string;

  @Attribute(DataTypes.STRING)
  declare lastName: string | null; // nullable by default

  declare createdAt: CreationOptional<Date>;
  declare updatedAt: CreationOptional<Date>;
}
```

### JavaScript (without types)

```javascript
export class User extends Model {
  @Attribute(DataTypes.INTEGER)
  @PrimaryKey
  @AutoIncrement
  id;

  @Attribute(DataTypes.STRING)
  @NotNull
  firstName;

  @Attribute(DataTypes.STRING)
  lastName;
}
```

## Decorator Reference

| Decorator | Import | Purpose |
|-----------|--------|---------|
| `@Attribute(DataType)` | `decorators-legacy` | Define column with data type |
| `@PrimaryKey` | `decorators-legacy` | Mark as primary key (disables auto `id`) |
| `@AutoIncrement` | `decorators-legacy` | Auto-increment integer PK |
| `@NotNull` | `decorators-legacy` | SQL NOT NULL constraint |
| `@Default(value)` | `decorators-legacy` | Static or dynamic default value |
| `@Unique` | `decorators-legacy` | Unique constraint |
| `@Index` | `decorators-legacy` | Create index on column |
| `@Table({ ... })` | `decorators-legacy` | Table-level options |
| `@ColumnName('col')` | `decorators-legacy` | Override database column name |
| `@CreatedAt` | `decorators-legacy` | Rename/configure createdAt timestamp |
| `@UpdatedAt` | `decorators-legacy` | Rename/configure updatedAt timestamp |
| `@DeletedAt` | `decorators-legacy` | Enable paranoid mode + configure deletedAt |
| `@Version` | `decorators-legacy` | Enable optimistic locking |
| `@HasOne(() => Target, fk)` | `decorators-legacy` | One-to-one association |
| `@HasMany(() => Target, fk)` | `decorators-legacy` | One-to-many association |
| `@BelongsTo(() => Target, fk)` | `decorators-legacy` | Inverse of HasOne/HasMany |
| `@BelongsToMany(() => Target, { through })` | `decorators-legacy` | Many-to-many association |

## TypeScript Type Helpers

| Type | Purpose |
|------|---------|
| `InferAttributes<M>` | Extracts attribute types from model class |
| `InferCreationAttributes<M>` | Same, but `CreationOptional` fields become optional |
| `CreationOptional<T>` | Marks field as optional during `Model.create()` |
| `NonAttribute<T>` | Excludes property from attribute inference (for associations, methods) |
| `PartialBy<T, K>` | Utility to make specific keys optional (manual typing, from `@sequelize/utils`) |

### Rules

- Nullable fields (`string | null`) are automatically optional at creation — no `CreationOptional` needed
- Getters/setters must be excluded via `NonAttribute` or `InferAttributes<User, { omit: 'prop' }>`
- `InferAttributes` ignores: static fields, methods, `NonAttribute<T>`, `Model` base class props

## Nullability

Attributes are **nullable by default**. Use `@NotNull` to enforce non-null:

```typescript
@Attribute(DataTypes.STRING)
@NotNull
declare firstName: string;      // NOT NULL

@Attribute(DataTypes.STRING)
declare lastName: string | null; // nullable (default)
```

## Default Values

### Static Default

```typescript
@Attribute(DataTypes.STRING)
@NotNull
@Default('John')
declare firstName: CreationOptional<string>;
```

### Dynamic JS Default (handled by Sequelize, not DB)

```typescript
@Attribute(DataTypes.DATE)
@NotNull
@Default(DataTypes.NOW)
declare registeredAt: CreationOptional<Date>;
```

> **Caution:** JS defaults (`DataTypes.NOW`, functions) only work through Sequelize. They won't apply in raw queries or migrations.

### Dynamic SQL Default (handled by DB)

```typescript
import { sql } from '@sequelize/core';

@Attribute(DataTypes.FLOAT)
@Default(sql.fn('random'))
declare randomNumber: CreationOptional<number>;
```

### UUID Defaults

```typescript
@Attribute(DataTypes.UUID.V4)
@PrimaryKey
@Default(sql.uuidV4)
declare id: CreationOptional<string>;
```

`sql.uuidV4` / `sql.uuidV1` use native SQL if available, otherwise fall back to JS generation.

## Primary Keys

### Auto-generated (default)

If no `@PrimaryKey` is specified, Sequelize adds `id: INTEGER AUTO_INCREMENT PRIMARY KEY`.

```typescript
// Declare the auto-generated id for TypeScript
declare id: CreationOptional<number>;
```

### Custom Primary Key

```typescript
@Attribute(DataTypes.INTEGER)
@PrimaryKey
@AutoIncrement
declare internalId: CreationOptional<number>;
```

### Composite Primary Key

```typescript
class UserRole extends Model<InferAttributes<UserRole>, InferCreationAttributes<UserRole>> {
  @Attribute(DataTypes.INTEGER)
  @PrimaryKey
  declare userId: number;

  @Attribute(DataTypes.INTEGER)
  @PrimaryKey
  declare roleId: number;
}
```

## Data Types

### Strings

| Type | Description |
|------|-------------|
| `DataTypes.STRING` | VARCHAR(255) |
| `DataTypes.STRING(100)` | VARCHAR(100) |
| `DataTypes.STRING.BINARY` | VARCHAR BINARY (MySQL/MariaDB) |
| `DataTypes.TEXT` | TEXT / CLOB |
| `DataTypes.TEXT('tiny')` | TINYTEXT (MySQL/MariaDB) |
| `DataTypes.TEXT('medium')` | MEDIUMTEXT |
| `DataTypes.TEXT('long')` | LONGTEXT |
| `DataTypes.CHAR` | CHAR(255) |
| `DataTypes.CHAR(100)` | CHAR(100) |
| `DataTypes.CITEXT` | Case-insensitive text (PostgreSQL, SQLite) |
| `DataTypes.TSVECTOR` | Full-text search vector (PostgreSQL) |

### Numbers

| Type | Description |
|------|-------------|
| `DataTypes.TINYINT` | Tiny integer |
| `DataTypes.SMALLINT` | Small integer |
| `DataTypes.MEDIUMINT` | Medium integer (MySQL/MariaDB) |
| `DataTypes.INTEGER` | Standard integer |
| `DataTypes.BIGINT` | Big integer (returned as string in v7) |
| `DataTypes.FLOAT` | Single-precision float |
| `DataTypes.DOUBLE` | Double-precision float |
| `DataTypes.DECIMAL(p, s)` | Exact decimal (returned as string in v7) |

Modifiers (MySQL/MariaDB): `.UNSIGNED`, `.ZEROFILL`
```typescript
DataTypes.INTEGER.UNSIGNED
DataTypes.INTEGER(1).UNSIGNED.ZEROFILL
```

### Dates

| Type | Description |
|------|-------------|
| `DataTypes.DATE` | TIMESTAMP WITH TIME ZONE (PostgreSQL), DATETIME (MySQL) |
| `DataTypes.DATE(6)` | With fractional seconds precision |
| `DataTypes.DATEONLY` | DATE only (no time) |
| `DataTypes.TIME` | TIME |
| `DataTypes.NOW` | Default value helper (JS-side, not SQL) |

### Other Types

| Type | Description |
|------|-------------|
| `DataTypes.BOOLEAN` | BOOLEAN / TINYINT(1) |
| `DataTypes.UUID` | UUID type |
| `DataTypes.UUID.V4` | UUID v4 variant |
| `DataTypes.UUID.V1` | UUID v1 variant |
| `DataTypes.BLOB` | Binary data |
| `DataTypes.BLOB('tiny'/'medium'/'long')` | Sized BLOB |
| `DataTypes.ENUM('val1', 'val2')` | Enum (PostgreSQL, MySQL, MariaDB only) |
| `DataTypes.JSON` | JSON column |
| `DataTypes.JSONB` | Binary JSON (PostgreSQL only) |
| `DataTypes.VIRTUAL(returnType, deps)` | Virtual attribute — no DB column |
| `DataTypes.GEOMETRY` | Geometry (PostGIS for PostgreSQL) |
| `DataTypes.GEOGRAPHY` | Geography (PostgreSQL only) |
| `DataTypes.HSTORE` | Key-value store (PostgreSQL only) |

### PostgreSQL-Only Types

```typescript
DataTypes.ARRAY(DataTypes.STRING)                  // VARCHAR(255)[]
DataTypes.ARRAY(DataTypes.ARRAY(DataTypes.STRING)) // VARCHAR(255)[][]
DataTypes.RANGE(DataTypes.INTEGER)                 // int4range
DataTypes.RANGE(DataTypes.DATE)                    // tstzrange
DataTypes.CIDR                                     // CIDR
DataTypes.INET                                     // INET
DataTypes.MACADDR                                  // MACADDR
```

## Naming Strategies

### Table Names

| Option | Effect |
|--------|--------|
| Default | Pluralized model name (`User` → `Users`) |
| `@Table({ underscored: true })` | snake_case (`User` → `users`) |
| `@Table({ freezeTableName: true })` | Exact model name (`User` → `User`) |
| `@Table({ tableName: 'my_users' })` | Manual name |

### Column Names

| Option | Effect |
|--------|--------|
| Default | Same as attribute name |
| `underscored: true` (on model) | snake_case (`createdAt` → `created_at`) |
| `@ColumnName('first_name')` | Manual override per column |

Global option:
```typescript
new Sequelize({ define: { underscored: true }, models: [...] });
```

## Auto-Generated Timestamps

Sequelize auto-adds `createdAt` and `updatedAt` (managed in JS, not SQL triggers).

```typescript
// TypeScript: declare them with CreationOptional
declare createdAt: CreationOptional<Date>;
declare updatedAt: CreationOptional<Date>;
```

### Disabling

```typescript
@Table({ timestamps: false })         // Disable all
@Table({ createdAt: false })          // Disable only createdAt
@Table({ updatedAt: false })          // Disable only updatedAt
```

### Renaming

```typescript
import { CreatedAt, UpdatedAt } from '@sequelize/core/decorators-legacy';

@CreatedAt
declare creationDate: CreationOptional<Date>;

@UpdatedAt
declare lastUpdateDate: CreationOptional<Date>;
```

## Validations & Constraints

### Attribute Validators

```typescript
import { ValidateAttribute } from '@sequelize/core/decorators-legacy';

@Attribute(DataTypes.STRING)
@NotNull
@ValidateAttribute((value: unknown) => {
  if (typeof value === 'string' && value.length === 0) {
    throw new Error('Name cannot be empty');
  }
})
declare name: string;
```

- Only run when attribute value has changed
- Skipped when value is `null` (nullability checked separately)

### Using `@sequelize/validator.js`

```typescript
import { IsEmail, IsUrl, Min, Max } from '@sequelize/validator.js';

@Attribute(DataTypes.STRING)
@NotNull
@IsEmail
declare email: string;
```

### Model Validators

Validate the entire model (useful for multi-attribute validation):

```typescript
import { ModelValidator } from '@sequelize/core/decorators-legacy';

@ModelValidator
validateCoords() {
  if ((this.latitude === null) !== (this.longitude === null)) {
    throw new Error('Either both latitude and longitude, or neither!');
  }
}
```

- **Always run**, even when values are `null`
- Support async (return a Promise)

## Indexes

### Single-Column Index

```typescript
@Attribute(DataTypes.STRING)
@NotNull
@Index
declare email: string;
```

### Composite Index (shared name)

```typescript
@Index({ name: 'name-index' })
declare firstName: string;

@Index({ name: 'name-index' })
declare lastName: string;
```

### Complex Index with `createIndexDecorator`

```typescript
import { createIndexDecorator } from '@sequelize/core/decorators-legacy';

const NameIndex = createIndexDecorator('NameIndex', {
  name: 'firstName-lastName',
  type: 'fulltext',
  concurrently: true,
});

class User extends Model {
  @NameIndex
  declare firstName: string;

  @NameIndex({ collate: 'case_insensitive' })
  declare lastName: string;
}
```

### Unique Constraints

```typescript
@Unique
declare email: string;

// Composite unique
@Unique('first-last')
declare firstName: string;
@Unique('first-last')
declare lastName: string;
```

### Via `@Table`

```typescript
@Table({
  indexes: [{
    name: 'firstName-lastName',
    type: 'fulltext',
    fields: ['firstName', { name: 'lastName', collate: 'case_insensitive' }],
  }],
})
class User extends Model {}
```

## Getters, Setters & Virtual Attributes

### Getter

```typescript
@Attribute(DataTypes.STRING)
@NotNull
get username(): string {
  return this.getDataValue('username').toUpperCase();
}
```

### Setter

```typescript
@Attribute(DataTypes.STRING)
@NotNull
set username(value: string) {
  this.setDataValue('username', value.toUpperCase());
}
```

> **Important:** Use `this.getDataValue()` / `this.setDataValue()` inside getters/setters to avoid infinite recursion. Static methods (`User.update()`) bypass setters.

### Virtual Attribute

```typescript
@Attribute(DataTypes.VIRTUAL(DataTypes.STRING, ['firstName', 'lastName']))
get fullName(): string {
  return `${this.firstName} ${this.lastName}`;
}
```

- No database column created
- Dependencies auto-loaded when included in `attributes`
- **Cannot be used in `where` clauses**

## Database Schema

```typescript
@Table({ schema: 'public' })
export class User extends Model {}

// Or globally
new Sequelize({ schema: 'public', models: [...] });
```

## Model Methods

```typescript
class User extends Model<InferAttributes<User>, InferCreationAttributes<User>> {
  @Attribute(DataTypes.STRING) @NotNull declare firstname: string;
  @Attribute(DataTypes.STRING) @NotNull declare lastname: string;

  getFullname() {
    return [this.firstname, this.lastname].join(' ');
  }

  static classLevelMethod() {
    return 'foo';
  }
}
```
