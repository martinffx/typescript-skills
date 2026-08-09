---
name: typescript-dynamodb-toolbox
description: DynamoDB single-table design using dynamodb-toolbox v2. Use when creating entities, defining key patterns, designing GSIs, writing queries, implementing pagination, or working with any DynamoDB data layer in TypeScript projects.
user-invocable: false
---

# DynamoDB with dynamodb-toolbox v2

Inspect the owning package and existing implementation first. Reuse established
project types, helpers, errors, lifecycle behavior, and test utilities. The
patterns below are options, not an implementation checklist. Introduce one only
when the current task requires it.

Type-safe DynamoDB interactions with Entity and Table abstractions for single-table design.

## Prerequisites

- Use Node.js 18+ and TypeScript 5+ with `strict: true`.
- Install `dynamodb-toolbox` with its peer dependencies, `@aws-sdk/client-dynamodb` and `@aws-sdk/lib-dynamodb`.
- Give each `Table` a `DynamoDBDocumentClient` through `documentClient` (or assign it before sending commands). Configure `removeUndefinedValues: true` when records may contain optional fields.

## Migrating v1 Code to v2

- Replace `entityAttributeName` and `entityAttributeHidden` with `entityAttribute: { name, hidden }`. Keep it enabled for single-table designs; use `entityAttribute: false` only for independently queried entity-per-table data.
- Keep `entityAttributeSavedAs` on the `Table`, not the `Entity`; it must be identical for every entity in a shared table.
- Schemas no longer need `.freeze()`. Call `.check()` only when validating a standalone schema; `new Entity(...)` already checks its schema.
- Use `item(...)` for root entity schemas, or pass a `map(...)` schema directly. `schema` and `s` replace the former `attr` shorthands.
- Replace transformer `parse`/`format` with `encode`/`decode`, `ReadItem` with `DecodedItem`, and `ReadValue` with `DecodedValue`.
- Replace instance `.name` reads with `.entityName` and `.tableName`.
- Records keyed by a string enum are complete by default. Call `.partial()` when missing enum keys are valid.

## When to Use DynamoDB

✓ **Use when:**
- Access patterns are known upfront and stable
- Need predictable sub-10ms performance at scale
- Microservice with clear data boundaries
- Willing to commit to single-table design

✗ **Avoid when:**
- Prototyping with fluid requirements
- Need ad-hoc analytical queries
- Team lacks DynamoDB expertise
- GraphQL resolvers drive access patterns

> DynamoDB inverts the relational paradigm: design for known access patterns, not flexible querying.

## Modeling Checklist

Before implementing:
1. **Define Entity Relationships** - Create ERD with all entities and relationships
2. **Create Entity Chart** - Map each entity to PK/SK patterns
3. **Design GSI Strategy** - Plan secondary access patterns
4. **Document Access Patterns** - List every query the application needs

See [references/modeling.md](references/modeling.md) for detailed methodology.

## Table Configuration

```typescript
import { DynamoDBClient } from '@aws-sdk/client-dynamodb'
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb'
import { Table } from 'dynamodb-toolbox/table'

const documentClient = DynamoDBDocumentClient.from(new DynamoDBClient(), {
  marshallOptions: { removeUndefinedValues: true },
})

const AppTable = new Table({
  name: process.env.TABLE_NAME || "AppTable",
  documentClient,
  partitionKey: { name: "PK", type: "string" },
  sortKey: { name: "SK", type: "string" },
  indexes: {
    GSI1: {
      type: "global",
      partitionKey: { name: "GSI1PK", type: "string" },
      sortKey: { name: "GSI1SK", type: "string" },
    },
    GSI2: {
      type: "global",
      partitionKey: { name: "GSI2PK", type: "string" },
      sortKey: { name: "GSI2SK", type: "string" },
    },
    GSI3: {
      type: "global",
      partitionKey: { name: "GSI3PK", type: "string" },
      sortKey: { name: "GSI3SK", type: "string" },
    },
  },
  entityAttributeSavedAs: "_et", // shared by every entity in this table
});
```

### Index Purpose

| Index | Purpose |
|-------|---------|
| Main Table (PK/SK) | Primary entity access |
| GSI1 | Collection queries (issues by repo, members by org) |
| GSI2 | Entity-specific queries and relationships (forks) |
| GSI3 | Hierarchical queries with temporal sorting (repos by owner) |

## Entity Definition (v2 syntax)

### Basic Pattern with Linked Keys

```typescript
import { Entity } from 'dynamodb-toolbox/entity'
import { item } from 'dynamodb-toolbox/schema/item'
import { boolean } from 'dynamodb-toolbox/schema/boolean'
import { string } from 'dynamodb-toolbox/schema/string'

const UserEntity = new Entity({
  name: "USER",
  table: AppTable,
  schema: item({
    // Business attributes
    username: string().required().key(),
    email: string().required(),
    bio: string().optional(),
  }).and(_schema => ({
    // Computed keys (PK/SK/GSI keys derived from business attributes)
    PK: string().key().link<typeof _schema>(
      ({ username }) => `ACCOUNT#${username}`
    ),
    SK: string().key().link<typeof _schema>(
      ({ username }) => `ACCOUNT#${username}`
    ),
    GSI1PK: string().link<typeof _schema>(
      ({ username }) => `ACCOUNT#${username}`
    ),
    GSI1SK: string().link<typeof _schema>(
      ({ username }) => `ACCOUNT#${username}`
    ),
  })),
});
```

### With Validation

```typescript
const RepoEntity = new Entity({
  name: "REPO",
  table: AppTable,
  schema: item({
    owner: string()
      .required()
      .validate((value: string) => /^[a-zA-Z0-9_-]+$/.test(value))
      .key(),
    repo_name: string()
      .required()
      .validate((value: string) => /^[a-zA-Z0-9_-]+$/.test(value))
      .key(),
    description: string().optional(),
    is_private: boolean().default(false),
  }).and(_schema => ({
    PK: string().key().link<typeof _schema>(
      ({ owner, repo_name }) => `REPO#${owner}#${repo_name}`
    ),
    SK: string().key().link<typeof _schema>(
      ({ owner, repo_name }) => `REPO#${owner}#${repo_name}`
    ),
    // GSI3 for temporal sorting (repos by owner, newest first)
    GSI3PK: string().link<typeof _schema>(
      ({ owner }) => `ACCOUNT#${owner}`
    ),
    GSI3SK: string()
      .default(() => `#${new Date().toISOString()}`)
      .savedAs("GSI3SK"),
  })),
});
```

## Entity Chart (Key Patterns)

| Entity | PK | SK | Purpose |
|--------|----|----|---------|
| User | `ACCOUNT#{username}` | `ACCOUNT#{username}` | Direct access |
| Repository | `REPO#{owner}#{name}` | `REPO#{owner}#{name}` | Direct access |
| Issue | `ISSUE#{owner}#{repo}#{padded_num}` | Same as PK | Direct access + enumeration |
| Comment | `REPO#{owner}#{repo}` | `ISSUE#{padded_num}#COMMENT#{id}` | Comments under issue |
| Star | `ACCOUNT#{username}` | `STAR#{owner}#{repo}#{timestamp}` | Adjacency list pattern |

**Key Pattern Rules:**
- `ENTITY#{id}` - Simple identifier
- `PARENT#{id}#CHILD#{id}` - Hierarchy
- `TYPE#{category}#{identifier}` - Categorization
- `#{timestamp}` - Temporal sorting (# prefix ensures ordering)

## Type Safety

```typescript
import { type InputItem, type FormattedItem } from 'dynamodb-toolbox/entity'

// Type exports
type UserRecord = typeof UserEntity
type UserInput = InputItem<typeof UserEntity>      // For writes
type UserFormatted = FormattedItem<typeof UserEntity> // For reads

// Usage in entities
class User {
  static fromRecord(record: UserFormatted): User { /* ... */ }
  toRecord(): UserInput { /* ... */ }
}
```

See [references/entity-layer.md](references/entity-layer.md) for transformation patterns.

## Repository Pattern

```typescript
import { DeleteItemCommand } from 'dynamodb-toolbox/entity/actions/delete'
import { GetItemCommand } from 'dynamodb-toolbox/entity/actions/get'
import { PutItemCommand } from 'dynamodb-toolbox/entity/actions/put'

class UserRepository {
  constructor(private entity: UserRecord) {}

  // CREATE with duplicate check
  async create(user: User): Promise<User> {
    try {
      const result = await this.entity
        .build(PutItemCommand)
        .item(user.toRecord())
        .options({
          condition: { attr: "PK", exists: false }, // Prevent duplicates
        })
        .send()

      return User.fromRecord(result.ToolboxItem)
    } catch (error) {
      if (error instanceof ConditionalCheckFailedException) {
        throw new DuplicateEntityError("User", user.username)
      }
      throw error
    }
  }

  // GET by key
  async get(username: string): Promise<User | undefined> {
    const result = await this.entity
      .build(GetItemCommand)
      .key({ username })
      .send()

    return result.Item ? User.fromRecord(result.Item) : undefined
  }

  // UPDATE with existence check
  async update(user: User): Promise<User> {
    try {
      const result = await this.entity
        .build(PutItemCommand)
        .item(user.toRecord())
        .options({
          condition: { attr: "PK", exists: true }, // Must exist
        })
        .send()

      return User.fromRecord(result.ToolboxItem)
    } catch (error) {
      if (error instanceof ConditionalCheckFailedException) {
        throw new EntityNotFoundError("User", user.username)
      }
      throw error
    }
  }

  // DELETE
  async delete(username: string): Promise<void> {
    await this.entity.build(DeleteItemCommand).key({ username }).send()
  }
}
```

See [references/error-handling.md](references/error-handling.md) for error patterns.

## Query Patterns

### Query GSI

```typescript
import { QueryCommand } from 'dynamodb-toolbox/table/actions/query'

// List issues for a repository using GSI1
async listIssues(owner: string, repoName: string): Promise<Issue[]> {
  const result = await this.table
    .build(QueryCommand)
    .entities(this.issueEntity)
    .query({
      partition: `ISSUE#${owner}#${repoName}`,
      index: "GSI1",
    })
    .send()

  return result.Items?.map(item => Issue.fromRecord(item)) || []
}
```

### Multi-Entity Queries

Pass every expected entity to `.entities(...)`. When all entities retain the shared internal entity attribute, DynamoDB Toolbox applies an entity filter and formats each returned item by its tag. If legacy items lack the tag, use `entityAttrFilter: false` only during migration and choose an explicit unmatched-item policy:

```typescript
const { Items } = await AppTable
  .build(QueryCommand)
  .entities(UserEntity, RepoEntity)
  .query({ partition: `ACCOUNT#${username}` })
  .options({
    entityAttrFilter: false,
    noEntityMatchBehavior: "DISCARD",
  })
  .send()
```

Without entity tags, Toolbox tries each entity schema in order, which is slower and can throw for an item that matches none. Do not disable the filter for steady-state single-table data.

### Query with Range Filter

```typescript
// List by status using beginsWith on SK
async listOpenIssues(owner: string, repoName: string): Promise<Issue[]> {
  const result = await this.table
    .build(QueryCommand)
    .entities(this.issueEntity)
    .query({
      partition: `ISSUE#${owner}#${repoName}`,
      index: "GSI4",
      range: {
        beginsWith: "ISSUE#OPEN#", // Filter to open issues only
      },
    })
    .send()

  return result.Items?.map(item => Issue.fromRecord(item)) || []
}
```

### Pagination

```typescript
// Encode/decode pagination tokens
function encodePageToken(lastEvaluated?: Record<string, unknown>): string | undefined {
  return lastEvaluated
    ? Buffer.from(JSON.stringify(lastEvaluated)).toString("base64")
    : undefined
}

function decodePageToken(token?: string): Record<string, unknown> | undefined {
  return token ? JSON.parse(Buffer.from(token, "base64").toString()) : undefined
}

// Query with pagination
async listReposByOwner(owner: string, limit = 50, offset?: string) {
  const result = await this.table
    .build(QueryCommand)
    .entities(this.repoEntity)
    .query({
      partition: `ACCOUNT#${owner}`,
      index: "GSI3",
      range: { lt: "ACCOUNT#" }, // Filter to only repos (not account itself)
    })
    .options({
      reverse: true,                              // Newest first
      exclusiveStartKey: decodePageToken(offset), // Continue from cursor
      limit,
    })
    .send()

  return {
    items: result.Items?.map(item => Repo.fromRecord(item)) || [],
    nextOffset: encodePageToken(result.LastEvaluatedKey),
  }
}
```

## Transactions

See [references/transactions.md](references/transactions.md) for:
- Multi-entity transactions (`PutTransaction` + `ConditionCheck`)
- Atomic counters with `$add(1)`
- TransactionCanceledException handling

## Testing

See [references/testing.md](references/testing.md) for:
- DynamoDB Local setup
- Test fixtures and factories
- Concurrency and temporal sorting tests

## Quick Reference

**Schema:**
- Use `item({})` for schema definition
- Mark key attributes with `.key()`
- Separate business attributes from computed keys using `.and()`
- Use `.link<typeof _schema>()` to compute PK/SK/GSI keys
- Use `.validate()` for field validation
- Use `.savedAs()` when DynamoDB name differs from schema name

**Types:**
- `InputItem<T>` for writes (excludes computed attributes)
- `FormattedItem<T>` for formatted reads (excludes hidden attributes)
- `DecodedItem<T>` when a read must include hidden attributes

**Repository:**
- Use `PutItemCommand` with `{ attr: "PK", exists: false }` for creates
- Use `PutItemCommand` with `{ attr: "PK", exists: true }` for updates
- Use `GetItemCommand` with `.key()` for reads
- Use `QueryCommand` with `.entities()` for type-safe queries

**Errors:**
- `ConditionalCheckFailedException` → DuplicateEntityError (create) or EntityNotFoundError (update)
- Always catch and convert to domain errors

**Testing:**
- Use unique IDs per test run (timestamp-based)
- Clean up test data after each test
- Use DynamoDB Local for development
