# Sequelize v7 — Querying

## INSERT Queries

### `Model.create()`

```typescript
const jane = await User.create({ firstName: 'Jane', lastName: 'Doe' });
console.log(jane.id); // auto-generated
```

Shorthand for `User.build({ ... })` + `await instance.save()`.

### `Model.bulkCreate()`

```typescript
const captains = await Captain.bulkCreate([
  { name: 'Jack Sparrow' },
  { name: 'Davy Jones' },
]);
```

#### `fields` option (restrict insertable columns)

```typescript
await User.bulkCreate(
  [{ username: 'foo' }, { username: 'bar', admin: true }],
  { fields: ['username'] }, // admin: true silently dropped
);
```

### `findOrCreate`

Returns `[instance, created]`:

```typescript
const [user, created] = await User.findOrCreate({
  where: { username: 'sdepold' },
  defaults: { job: 'Technical Lead' },
});
```

Wraps in a transaction/savepoint automatically. Alternatives: `findCreateFind` (no transaction), `findOrBuild` (no save).

### Creating with Associations

```typescript
await User.create(
  { name: 'Mary', address: { city: 'Nassau', country: 'Bahamas' } },
  { include: ['address'] },
);

// HasMany / BelongsToMany — pass array
await User.create(
  { name: 'Mary', addresses: [{ city: 'Nassau' }, { city: 'London' }] },
  { include: ['addresses'] },
);
```

For existing associated models, use the foreign key directly:
```typescript
await User.create({ name: 'Mary', addressId: address.id });
```

## SELECT Queries

### Finder Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `findAll({ where })` | `Model[]` | All matching records |
| `findOne({ where })` | `Model \| null` | First matching record |
| `findByPk(pk)` | `Model \| null` | Find by primary key |
| `findByPk({ col1, col2 })` | `Model \| null` | Find by composite PK |
| `findAndCountAll({ where })` | `{ count, rows }` | Pagination helper |
| `count({ where })` | `number` | Count matching records |
| `max('column')` | `any` | Maximum value |
| `min('column')` | `any` | Minimum value |
| `sum('column')` | `number` | Sum of values |

### `rejectOnEmpty` (TypeScript-friendly)

```typescript
const project = await Project.findOne({
  where: { title: 'My Title' },
  rejectOnEmpty: true, // throws if not found; return type is Project (not Project | null)
});
```

### Selecting Attributes

```typescript
// Specific columns
User.findAll({ attributes: ['firstName', 'lastName'] });

// Exclude columns
User.findAll({ attributes: { exclude: ['password'] } });

// Computed columns
User.findAll({
  attributes: {
    include: [[sql`DATEDIFF(year, "birthdate", GETDATE())`, 'age']],
  },
});
```

### Reloading

```typescript
await jane.reload(); // Re-fetch from DB
```

### Plain Objects

```typescript
const users = await User.findAll({ raw: true }); // Returns plain JS objects, not model instances
```

## WHERE Clauses

### Basic Equality

```typescript
Post.findAll({ where: { authorId: 2 } });
// WHERE "authorId" = 2
```

### With Operators

```typescript
import { Op } from '@sequelize/core';

Post.findAll({ where: { views: { [Op.gt]: 100, [Op.lte]: 500 } } });
// WHERE "views" > 100 AND "views" <= 500
```

### Logical Combinations

```typescript
// OR
Post.findAll({ where: { [Op.or]: [{ authorId: 12 }, { status: 'active' }] } });

// NOT
Post.findAll({ where: { [Op.not]: { authorId: 12 } } });

// AND (default — multiple props in same object)
Post.findAll({ where: { authorId: 12, status: 'active' } });
```

### Casting Attributes

```typescript
User.findAll({ where: { 'createdAt::text': { [Op.like]: '2012-%' } } });
// WHERE CAST("createdAt" AS TEXT) LIKE '2012-%'
```

### Referencing Another Column

```typescript
Article.findAll({ where: { authorId: sql.attribute('reviewerId') } });
```

### Raw SQL in WHERE

```typescript
User.findAll({ where: sql`char_length(${sql.attribute('content')}) <= ${maxLength}` });
```

## Operators Reference

### Implicit Operators

| Value Type | Inferred Operator | SQL |
|------------|-------------------|-----|
| `null` | `Op.is` | `IS NULL` |
| Array | `Op.in` | `IN (...)` |
| Other | `Op.eq` | `= value` |

### Comparison

| Operator | SQL |
|----------|-----|
| `Op.eq` | `=` |
| `Op.ne` | `<>` |
| `Op.gt` | `>` |
| `Op.gte` | `>=` |
| `Op.lt` | `<` |
| `Op.lte` | `<=` |
| `Op.between` | `BETWEEN x AND y` |
| `Op.notBetween` | `NOT BETWEEN x AND y` |
| `Op.in` | `IN (...)` |
| `Op.notIn` | `NOT IN (...)` |
| `Op.is` | `IS` |
| `Op.isNot` | `IS NOT` |

### String

| Operator | SQL |
|----------|-----|
| `Op.like` | `LIKE` (case-sensitive) |
| `Op.notLike` | `NOT LIKE` |
| `Op.iLike` | `ILIKE` (case-insensitive) |
| `Op.notILike` | `NOT ILIKE` |
| `Op.regexp` | `~` |
| `Op.notRegexp` | `!~` |
| `Op.iRegexp` | `~*` |
| `Op.notIRegexp` | `!~*` |
| `Op.startsWith` | Starts with (via LIKE) |
| `Op.endsWith` | Ends with (via LIKE) |
| `Op.substring` | Contains string (via LIKE) |

### PostgreSQL Array Operators

| Operator | SQL |
|----------|-----|
| `Op.contains` | `@>` (array/JSONB contains) |
| `Op.contained` | `<@` (contained by) |
| `Op.overlap` | `&&` (overlap) |
| `Op.anyKeyExists` | `?\|` (JSONB any key exists) |
| `Op.allKeysExist` | `?&` (JSONB all keys exist) |

### Range Operators (PostgreSQL)

| Operator | SQL |
|----------|-----|
| `Op.contains` | `@>` |
| `Op.contained` | `<@` |
| `Op.overlap` | `&&` |
| `Op.adjacent` | `-\|-` |
| `Op.strictLeft` | `<<` |
| `Op.strictRight` | `>>` |
| `Op.noExtendRight` | `&<` |
| `Op.noExtendLeft` | `&>` |

### ALL, ANY, VALUES

```typescript
// Title contains BOTH "cat" AND "dog"
{ title: { [Op.iLike]: { [Op.all]: ['%cat%', '%dog%'] } } }

// authorId equals 12 OR 13
{ authorId: { [Op.any]: [12, 13] } }

// Dynamic values
{ authorId: { [Op.any]: { [Op.values]: [12, sql`12 + 45`] } } }
```

## UPDATE Queries

### Instance `save()`

```typescript
const jane = await User.create({ name: 'Jane' });
jane.name = 'Ada';
await jane.save();
```

**Detecting nested mutations** — Sequelize can't detect mutations to nested objects. Replace the value or use `changed()`:

```typescript
// Correct: replace value
jane.role = [...jane.role, 'admin'];
await jane.save();

// Alternative: force change detection
jane.role.push('admin');
jane.changed('role', true);
await jane.save();
```

**Save specific fields:**

```typescript
await jane.save({ fields: ['name'] }); // Only saves name
```

### Instance `update()`

```typescript
await jane.update({ name: 'Ada' }); // Only updates specified fields
```

### Static `Model.update()` (bulk)

```typescript
await User.update({ lastName: 'Doe' }, { where: { lastName: null } });
```

### Increment / Decrement

```typescript
await jane.increment('age', { by: 2 });
await jane.increment({ age: 2, cash: 100 }); // Different amounts
await jane.increment(['age', 'cash'], { by: 2 }); // Same amount

await jane.decrement('age', { by: 1 });
```

Static variants: `Model.increment(...)`, `Model.decrement(...)`

## DELETE Queries

```typescript
// Single instance
await jane.destroy();

// Bulk delete
await User.destroy({ where: { firstName: 'Jane' } });

// Truncate single table
await User.truncate();

// Destroy all tables (test utility)
await sequelize.destroyAll();

// Truncate all tables (faster, may fail with FK constraints)
await sequelize.truncate();
```

## Eager Loading (include)

```typescript
// Basic
const posts = await Post.findAll({ include: ['comments'] });

// Nested
const posts = await Post.findAll({
  include: [{ association: 'comments', include: ['author'] }],
});

// INNER JOIN (required)
Post.findAll({ include: [{ association: 'comments', required: true }] });

// Separate queries (avoids duplicate data for HasMany/BelongsToMany)
Post.findAll({
  include: [{ association: 'comments', separate: true, order: [['createdAt', 'DESC']] }],
});

// Filter associated models
Post.findAll({
  include: [{ association: 'comments', where: { approved: true } }],
});

// Reference included model in parent WHERE
Article.findAll({
  include: ['comments'],
  where: { '$comments.id$': { [Op.eq]: null } },
});
```

### BelongsToMany through model

```typescript
Author.findOne({
  include: [{
    association: 'books',
    through: { attributes: [], where: { role: 'reviewer' } },
  }],
});
```

## Ordering

```typescript
Subtask.findAll({ order: [['title', 'DESC']] });

// Order by association
Task.findAll({
  include: ['subtasks'],
  order: [['subtasks', 'title', 'DESC']],
});

// Raw SQL
Subtask.findAll({ order: sql`UPPERCASE(${sql.attribute('title')}) DESC` });
```

## Grouping

```typescript
Project.findAll({ group: ['name'] });
```

## Pagination

```typescript
Project.findAll({ offset: 5, limit: 10 });
```

## Raw SQL (the `sql` Tag)

The `sql` tag safely interpolates variables (auto-escaped):

```typescript
import { sql } from '@sequelize/core';

await sequelize.query(sql`SELECT * FROM users WHERE id = ${id}`);
```

### SQL Helper Functions

| Helper | Purpose | Example |
|--------|---------|---------|
| `sql.identifier('table')` | Escape identifiers | `sql.identifier('public', 'users')` → `"public"."users"` |
| `sql.attribute('name')` | Reference model attribute | Maps to column name |
| `sql.list([1,2,3])` | SQL list for IN | `IN (1, 2, 3)` |
| `sql.join(items, ', ')` | Join SQL fragments | Like `Array.join()` for SQL |
| `sql.where({ ... })` | Generate WHERE from object | Same syntax as `findAll` where |
| `sql.cast(val, type)` | CAST expression | `CAST(val AS type)` |
| `sql.fn('LOWER', arg)` | SQL function call | `LOWER("name")` |
| `sql.uuidV4` | UUID v4 generation | Dialect-specific |
| `sql.jsonPath(col, path)` | JSON extraction | Dialect-specific JSON access |
| `sql.unquote(expr)` | JSON_UNQUOTE | Removes JSON string quotes |

### Replacements vs Bind Parameters

**Replacements** (escaped by Sequelize):
```typescript
await sequelize.query('SELECT * FROM projects WHERE status = :status', {
  replacements: { status: 'active' },
});
```

**Bind Parameters** (sent separately to DB):
```typescript
await sequelize.query('SELECT * FROM projects WHERE status = $1', {
  bind: ['active'],
  type: QueryTypes.SELECT,
});
```

### `sequelize.query()` Options

```typescript
const [results, metadata] = await sequelize.query(sql`SELECT * FROM users`);

// Typed results
const users = await sequelize.query('SELECT * FROM users', {
  type: QueryTypes.SELECT,
});

// Map to model instances
const projects = await sequelize.query('SELECT * FROM projects', {
  model: Project,
  mapToModel: true,
});
```

## Querying JSON

### Nested Property Access (dot notation)

```typescript
User.findAll({ where: { 'jsonAttribute.address.country': 'Belgium' } });
// PostgreSQL: "jsonAttribute"#>ARRAY['address','country'] = '"Belgium"'
// MySQL: JSON_EXTRACT(`jsonAttribute`, '$.address.country') = '"Belgium"'
```

### Array Index Access

```typescript
User.findAll({ where: { 'gameData.passwords[0]': 451 } });
```

### Casting JSON Values

```typescript
{ 'jsonAttribute.age::integer': { [Op.gt]: 18 } }
// CAST("jsonAttribute"->'age' AS integer) > 18
```

### Unquoting JSON (strip string quotes)

```typescript
{ 'jsonAddress.country:unquote': 'Belgium' }
// PostgreSQL: "jsonAddress"->>'country' = 'Belgium'
```

### JSON null vs SQL NULL

| Code | Result |
|------|--------|
| `jsonAttribute: null` (default) | JSON `'null'` |
| `jsonAttribute: SQL_NULL` | SQL `NULL` |
| `jsonAttribute: JSON_NULL` | JSON `'null'` (explicit) |

Control default with `nullJsonStringification`: `'json'` (default), `'sql'`, or `'explicit'`.

```typescript
import { SQL_NULL, JSON_NULL, or } from '@sequelize/core';

// Query both nulls
{ jsonAttribute: or(SQL_NULL, JSON_NULL) }
```

## Subqueries

Subqueries require raw SQL via the `sql` tag:

```typescript
Post.findAll({
  where: {
    id: {
      [Op.in]: sql`
        SELECT DISTINCT "postId"
        FROM "reactions" AS "reaction"
        WHERE "reaction"."type" = ${reactionType}
      `,
    },
  },
});
```

### Reusable Subquery Pattern

```typescript
function postHasReaction(type: ReactionType) {
  return {
    id: {
      [Op.in]: sql`
        SELECT DISTINCT "postId"
        FROM "reactions"
        WHERE "type" = ${type}
      `,
    },
  };
}

Post.findAll({ where: postHasReaction(ReactionType.Laugh) });
```
