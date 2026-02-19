# Sequelize v7 — Associations

## Overview

Sequelize supports three relationship types:

| Relationship | Association | FK Location |
|-------------|------------|-------------|
| One-to-One | `@HasOne` + `@BelongsTo` | Target model |
| One-to-Many | `@HasMany` + `@BelongsTo` | Target model |
| Many-to-Many | `@BelongsToMany` | Junction/through table |

All decorators import from `@sequelize/core/decorators-legacy`.

## HasOne (One-to-One)

The foreign key lives on the **target** model.

```typescript
import { Model, InferAttributes, InferCreationAttributes, CreationOptional, NonAttribute, DataTypes } from '@sequelize/core';
import { Attribute, PrimaryKey, AutoIncrement, NotNull, HasOne, Unique } from '@sequelize/core/decorators-legacy';

class Person extends Model<InferAttributes<Person>, InferCreationAttributes<Person>> {
  @Attribute(DataTypes.INTEGER)
  @AutoIncrement
  @PrimaryKey
  declare id: CreationOptional<number>;

  @HasOne(() => DrivingLicense, 'ownerId')
  declare drivingLicense?: NonAttribute<DrivingLicense>;
}

class DrivingLicense extends Model<InferAttributes<DrivingLicense>, InferCreationAttributes<DrivingLicense>> {
  @Attribute(DataTypes.INTEGER)
  @AutoIncrement
  @PrimaryKey
  declare id: CreationOptional<number>;

  @Attribute(DataTypes.INTEGER)
  @NotNull
  @Unique // Enforces true one-to-one at DB level
  declare ownerId: number;
}
```

### With Inverse Configuration

```typescript
class Person extends Model<InferAttributes<Person>, InferCreationAttributes<Person>> {
  @HasOne(() => DrivingLicense, {
    foreignKey: 'ownerId',
    inverse: { as: 'owner' },
  })
  declare drivingLicense?: NonAttribute<DrivingLicense>;
}

class DrivingLicense extends Model<InferAttributes<DrivingLicense>, InferCreationAttributes<DrivingLicense>> {
  /** Defined by {@link Person.drivingLicense} */
  declare owner?: NonAttribute<Person>;

  @Attribute(DataTypes.INTEGER)
  @NotNull
  declare ownerId: number;
}
```

### HasOne Association Methods

Declare on the source model for TypeScript:

```typescript
import {
  HasOneGetAssociationMixin,
  HasOneSetAssociationMixin,
  HasOneCreateAssociationMixin,
} from '@sequelize/core';

class Person extends Model<InferAttributes<Person>, InferCreationAttributes<Person>> {
  @HasOne(() => DrivingLicense, 'ownerId')
  declare drivingLicense?: NonAttribute<DrivingLicense>;

  declare getDrivingLicense: HasOneGetAssociationMixin<DrivingLicense>;
  declare setDrivingLicense: HasOneSetAssociationMixin<DrivingLicense, DrivingLicense['id']>;
  declare createDrivingLicense: HasOneCreateAssociationMixin<DrivingLicense, 'ownerId'>;
}
```

```typescript
const license = await person.getDrivingLicense();
await person.setDrivingLicense(license);  // by instance
await person.setDrivingLicense(5);        // by PK
await person.setDrivingLicense(null);     // unset
const newLicense = await person.createDrivingLicense({ number: '123456' });
```

### HasOne Options

| Option | Purpose |
|--------|---------|
| `foreignKey` | FK column name on target model |
| `sourceKey` | Source attribute FK references (default: PK) |
| `inverse.as` | Alias for auto-created BelongsTo inverse |

## HasMany (One-to-Many)

FK lives on the **target** model.

```typescript
import { HasMany, BelongsTo } from '@sequelize/core/decorators-legacy';

class Post extends Model<InferAttributes<Post>, InferCreationAttributes<Post>> {
  @Attribute(DataTypes.INTEGER)
  @AutoIncrement
  @PrimaryKey
  declare id: CreationOptional<number>;

  @HasMany(() => Comment, 'postId')
  declare comments?: NonAttribute<Comment[]>;
}

class Comment extends Model<InferAttributes<Comment>, InferCreationAttributes<Comment>> {
  @Attribute(DataTypes.INTEGER)
  @AutoIncrement
  @PrimaryKey
  declare id: CreationOptional<number>;

  @Attribute(DataTypes.INTEGER)
  @NotNull
  declare postId: number;
}
```

### With Inverse

```typescript
@HasMany(() => Comment, {
  foreignKey: 'postId',
  inverse: { as: 'post' },
})
declare comments?: NonAttribute<Comment[]>;
```

### HasMany Association Methods

```typescript
import {
  HasManyGetAssociationsMixin,
  HasManySetAssociationsMixin,
  HasManyAddAssociationMixin,
  HasManyAddAssociationsMixin,
  HasManyRemoveAssociationMixin,
  HasManyRemoveAssociationsMixin,
  HasManyCreateAssociationMixin,
  HasManyHasAssociationMixin,
  HasManyHasAssociationsMixin,
  HasManyCountAssociationsMixin,
} from '@sequelize/core';

class Post extends Model<InferAttributes<Post>, InferCreationAttributes<Post>> {
  @HasMany(() => Comment, 'postId')
  declare comments?: NonAttribute<Comment[]>;

  declare getComments: HasManyGetAssociationsMixin<Comment>;
  declare setComments: HasManySetAssociationsMixin<Comment, Comment['id']>;
  declare addComment: HasManyAddAssociationMixin<Comment, Comment['id']>;
  declare addComments: HasManyAddAssociationsMixin<Comment, Comment['id']>;
  declare removeComment: HasManyRemoveAssociationMixin<Comment, Comment['id']>;
  declare removeComments: HasManyRemoveAssociationsMixin<Comment, Comment['id']>;
  declare createComment: HasManyCreateAssociationMixin<Comment, 'postId'>;
  declare hasComment: HasManyHasAssociationMixin<Comment, Comment['id']>;
  declare hasComments: HasManyHasAssociationsMixin<Comment, Comment['id']>;
  declare countComments: HasManyCountAssociationsMixin<Comment>;
}
```

```typescript
const comments = await post.getComments();
await post.setComments([comment1, comment2]);    // Replace all
await post.addComment(comment1);                 // Add one
await post.addComments([comment1, comment2]);    // Add many
await post.removeComment(comment1);              // Remove one
await post.removeComments([1, 2, 3]);            // Remove by PKs
const comment = await post.createComment({ content: 'Hello' }); // FK auto-set
const exists = await post.hasComment(comment1);  // Check one
const allExist = await post.hasComments([1, 2]); // Check all
const count = await post.countComments();
```

**Caution with set/remove:**
- Non-nullable FK → previously associated models are **deleted**
- Nullable FK → FK set to `null` by default
- Use `{ destroyPrevious: true }` on set or `{ destroy: true }` on remove to force deletion

## BelongsTo (Inverse of HasOne/HasMany)

FK lives on the **source** model (the one declaring `@BelongsTo`).

```typescript
class Comment extends Model<InferAttributes<Comment>, InferCreationAttributes<Comment>> {
  @BelongsTo(() => Post, 'postId')
  declare post?: NonAttribute<Post>;

  @Attribute(DataTypes.INTEGER)
  @NotNull
  declare postId: number;
}
```

### BelongsTo with Inverse

Unlike other associations, BelongsTo does **not** auto-create the inverse. You must specify it:

```typescript
@BelongsTo(() => Post, {
  foreignKey: 'postId',
  inverse: {
    as: 'comments',
    type: 'hasMany', // 'hasOne' | 'hasMany'
  },
})
declare post?: NonAttribute<Post>;
```

### BelongsTo Association Methods

```typescript
import {
  BelongsToGetAssociationMixin,
  BelongsToSetAssociationMixin,
  BelongsToCreateAssociationMixin,
} from '@sequelize/core';

class Comment extends Model<InferAttributes<Comment>, InferCreationAttributes<Comment>> {
  declare getPost: BelongsToGetAssociationMixin<Post>;
  declare setPost: BelongsToSetAssociationMixin<Post, Comment['postId']>;
  declare createPost: BelongsToCreateAssociationMixin<Post>;
}
```

```typescript
const post = await comment.getPost();
await comment.setPost(post);     // by instance
await comment.setPost(1);        // by PK
const newPost = await comment.createPost({ title: 'New' }); // Not recommended — create parent first
```

### BelongsTo Options

| Option | Purpose |
|--------|---------|
| `foreignKey` | FK attribute name on the source model |
| `targetKey` | Attribute on target model FK references (default: PK) |
| `inverse.as` | Name of inverse association |
| `inverse.type` | `'hasOne'` or `'hasMany'` |

## BelongsToMany (Many-to-Many)

Uses a **junction/through table** with FKs for both models.

```typescript
import { BelongsToMany } from '@sequelize/core/decorators-legacy';

class Person extends Model<InferAttributes<Person>, InferCreationAttributes<Person>> {
  @BelongsToMany(() => Toot, {
    through: 'LikedToot',
    inverse: { as: 'likers' },
  })
  declare likedToots?: NonAttribute<Toot[]>;
}

class Toot extends Model<InferAttributes<Toot>, InferCreationAttributes<Toot>> {
  /** Defined by {@link Person.likedToots} */
  declare likers?: NonAttribute<Person[]>;
}
```

### Custom Through Model (with extra attributes)

```typescript
class Person extends Model<InferAttributes<Person>, InferCreationAttributes<Person>> {
  @BelongsToMany(() => Toot, {
    through: () => LikedToot,
    inverse: { as: 'likers' },
  })
  declare likedToots?: NonAttribute<Toot[]>;
}

class LikedToot extends Model<InferAttributes<LikedToot>, InferCreationAttributes<LikedToot>> {
  declare likerId: number;
  declare likedTootId: number;

  @Attribute(DataTypes.DATE)
  declare likedAt: Date | null;
}
```

### BelongsToMany Options

| Option | Purpose |
|--------|---------|
| `through` | Through model name (string) or `() => Model` reference or `{ model, unique }` |
| `inverse.as` | Name for auto-created inverse association |
| `foreignKey` | FK in through table pointing to source |
| `otherKey` | FK in through table pointing to target |
| `sourceKey` | Attribute on source model that FK references |
| `targetKey` | Attribute on target model that otherKey references |
| `through.unique` | Enable/disable unique constraint on FK pair (default: true) |
| `throughAssociations` | Names for the 4 intermediary associations |

### BelongsToMany Association Methods

```typescript
import {
  BelongsToManyGetAssociationsMixin,
  BelongsToManySetAssociationsMixin,
  BelongsToManyAddAssociationMixin,
  BelongsToManyAddAssociationsMixin,
  BelongsToManyRemoveAssociationMixin,
  BelongsToManyRemoveAssociationsMixin,
  BelongsToManyCreateAssociationMixin,
  BelongsToManyHasAssociationMixin,
  BelongsToManyHasAssociationsMixin,
  BelongsToManyCountAssociationsMixin,
} from '@sequelize/core';

class Author extends Model<InferAttributes<Author>, InferCreationAttributes<Author>> {
  @BelongsToMany(() => Book, { through: 'AuthorBook' })
  declare books?: NonAttribute<Book[]>;

  declare getBooks: BelongsToManyGetAssociationsMixin<Book>;
  declare setBooks: BelongsToManySetAssociationsMixin<Book, Book['id']>;
  declare addBook: BelongsToManyAddAssociationMixin<Book, Book['id']>;
  declare addBooks: BelongsToManyAddAssociationsMixin<Book, Book['id']>;
  declare removeBook: BelongsToManyRemoveAssociationMixin<Book, Book['id']>;
  declare removeBooks: BelongsToManyRemoveAssociationsMixin<Book, Book['id']>;
  declare createBook: BelongsToManyCreateAssociationMixin<Book, 'postId'>;
  declare hasBook: BelongsToManyHasAssociationMixin<Book, Book['id']>;
  declare hasBooks: BelongsToManyHasAssociationsMixin<Book, Book['id']>;
  declare countBooks: BelongsToManyCountAssociationsMixin<Book>;
}
```

### Through Model in Queries

```typescript
const author = await Author.findOne({ include: ['books'] });
author.books[0].BookAuthor; // → { bookId: 1, authorId: 1 }

// Filter/exclude through attributes
Author.findOne({
  include: [{
    association: 'books',
    through: {
      attributes: [],                  // Exclude all through attributes
      where: { role: 'reviewer' },     // Filter through model
    },
  }],
});
```

## Eager Loading Patterns

### Basic Include

```typescript
const posts = await Post.findAll({ include: ['comments'] });
```

### Nested Include

```typescript
const posts = await Post.findAll({
  include: [{ association: 'comments', include: ['author'] }],
});
```

### INNER JOIN (required)

```typescript
Post.findAll({ include: [{ association: 'comments', required: true }] });
```

### Separate Queries (for HasMany/BelongsToMany)

Avoids duplicate parent rows from JOINs:

```typescript
Post.findAll({
  include: [{ association: 'comments', separate: true, order: [['createdAt', 'DESC']] }],
});
```

### Filter on Association

```typescript
Post.findAll({
  include: [{ association: 'comments', where: { approved: true } }],
});
```

Using `where` on include sets `required: true` by default.

### Reference Association in Parent WHERE

```typescript
Article.findAll({
  include: ['comments'],
  where: { '$comments.id$': { [Op.eq]: null } },
});
```

Does **not** work with `separate: true`.

## v7 Association Changes from v6

- Default association names are now **camelCase** (`userId` not `UserId`)
- Duplicate association names now throw errors
- `as` in `include` is deprecated — use `association`
- Bidirectional options must match; use `inverse` option
- `BelongsTo` does NOT auto-create inverse — must specify `inverse.type`
