# Entity layer transformation patterns

Domain entities own construction, invariant-preserving changes, and typed request
or item decoding. Routes own HTTP validation and response serialization;
repositories own DynamoDB I/O.

## Transformation methods

An established domain entity may implement three methods for boundary crossing:

```typescript
class EntityName {
  // 1. API Request → Domain Entity
  static fromRequest(request: EntityCreateRequest): EntityName

  // 2. DynamoDB Item → Domain Entity
  static fromItem(item: EntityFormatted): EntityName

  // 3. Domain Entity → DynamoDB Item
  toItem(): EntityInput
}
```

Import request and Toolbox item types with type-only imports. Domain models may
construct typed Effect success, failure, or absence values, but they must not
execute effects or import clients, commands, Layers, configuration, or
environment access.

## Data Flow Through Layers

### Create/Update Flow

```
HTTP Request
    ↓ fromRequest()
Domain Entity
    ↓ toItem()
DynamoDB Item
    ↓ DynamoDB save
DynamoDB Item (with timestamps)
    ↓ fromItem()
Domain Entity
    ↓ route serialization
HTTP Response
```

### Read Flow

```
HTTP Request
    ↓ repository.get(id)
DynamoDB Item
    ↓ fromItem()
Domain Entity
    ↓ route serialization
HTTP Response
```

## Complete Entity Example

```typescript
import { type InputItem, type FormattedItem } from 'dynamodb-toolbox/entity'
import { DateTime } from 'luxon'
import type { IssueCreateRequest } from '../routes/issues/schema'

// `IssueDdbEntity` is the DynamoDB Toolbox Entity declaration.
// Keep it distinct from the domain `IssueEntity` class below.
declare const IssueDdbEntity: import('dynamodb-toolbox/entity').Entity
type IssueInput = InputItem<typeof IssueDdbEntity>
type IssueFormatted = FormattedItem<typeof IssueDdbEntity>

// Domain entity
class IssueEntity {
  // Immutable properties
  public readonly owner: string
  public readonly repoName: string
  public readonly issueNumber: number
  public readonly title: string
  public readonly body?: string
  public readonly status: "open" | "closed"
  public readonly author: string
  public readonly assignees: string[]
  public readonly labels: string[]
  public readonly created: DateTime
  public readonly modified: DateTime

  constructor(props: IssueEntityOpts) {
    this.owner = props.owner
    this.repoName = props.repoName
    this.issueNumber = props.issueNumber
    this.title = props.title
    this.body = props.body
    this.status = props.status || "open"
    this.author = props.author
    this.assignees = props.assignees || []
    this.labels = props.labels || []
    this.created = props.created || DateTime.utc()
    this.modified = props.modified || DateTime.utc()
  }

  // 1. API Request → Entity
  static fromRequest(data: IssueCreateRequest): IssueEntity {
    // The route validated the HTTP shape; the entity enforces domain invariants.
    IssueEntity.validate(data)

    return new IssueEntity({
      owner: data.owner,
      repoName: data.repo_name,
      issueNumber: 0, // Set by repository after getting from counter
      title: data.title,
      body: data.body,
      status: "open",
      author: data.author,
      assignees: data.assignees || [],
      labels: data.labels || [],
    })
  }

  // 2. DynamoDB Item → Entity
  static fromItem(item: IssueFormatted): IssueEntity {
    return new IssueEntity({
      owner: item.owner,
      repoName: item.repo_name,
      issueNumber: item.issue_number,
      title: item.title,
      body: item.body,
      status: item.status,
      author: item.author,
      // DynamoDB Sets → Arrays
      assignees: item.assignees ? Array.from(item.assignees) : [],
      labels: item.labels ? Array.from(item.labels) : [],
      // ISO strings → DateTime
      created: DateTime.fromISO(item.created),
      modified: DateTime.fromISO(item.modified),
    })
  }

  // 3. Entity → DynamoDB Item
  toItem(): IssueInput {
    return {
      owner: this.owner,
      repo_name: this.repoName,
      issue_number: this.issueNumber,
      title: this.title,
      body: this.body,
      status: this.status,
      author: this.author,
      // Arrays → Sets (DynamoDB doesn't allow empty sets)
      assignees: this.assignees.length > 0
        ? new Set(this.assignees)
        : undefined,
      labels: this.labels.length > 0
        ? new Set(this.labels)
        : undefined,
      // DateTime → ISO strings handled by DynamoDB Toolbox
    }
  }

  // Domain invariant checks called by fromRequest
  private static validate(data: IssueCreateRequest): void {
    if (!data.title || data.title.trim().length === 0) {
      throw new ValidationError("title", "Title is required")
    }
    if (data.title.length > 256) {
      throw new ValidationError("title", "Title must be 256 characters or less")
    }
  }

  // Immutable update (returns new instance)
  updateIssue(opts: UpdateIssueEntityOpts): IssueEntity {
    return new IssueEntity({
      // Preserve immutable fields
      owner: this.owner,
      repoName: this.repoName,
      issueNumber: this.issueNumber,
      author: this.author,
      created: this.created,
      // Apply updates
      title: opts.title ?? this.title,
      body: opts.body ?? this.body,
      status: opts.status ?? this.status,
      assignees: opts.assignees ?? this.assignees,
      labels: opts.labels ?? this.labels,
      // Update timestamp
      modified: DateTime.utc(),
    })
  }

  // Helper: Entity key for error messages
  getEntityKey(): string {
    return `ISSUE#${this.owner}#${this.repoName}#${this.issueNumber}`
  }

  // Helper: Parent entity key for transactions
  getParentEntityKey(): string {
    return `REPO#${this.owner}#${this.repoName}`
  }
}
```

## Field Naming Conventions

### Entity Layer (TypeScript/camelCase)

```typescript
class RepositoryEntity {
  public readonly repoName: string
  public readonly isPrivate: boolean
  public readonly paymentPlanId?: string
}
```

### Database Layer (DynamoDB/snake_case)

```typescript
schema: item({
  repo_name: string().required(),
  is_private: boolean().default(false),
  payment_plan_id: string().optional(),
})
```

### Mapping in Transformations

```typescript
// fromItem: snake_case → camelCase
static fromItem(item: RepoFormatted): RepositoryEntity {
  return new RepositoryEntity({
    repoName: item.repo_name,
    isPrivate: item.is_private,
    paymentPlanId: item.payment_plan_id,
  })
}

// toItem: camelCase → snake_case
toItem(): RepoInput {
  return {
    repo_name: this.repoName,
    is_private: this.isPrivate,
    payment_plan_id: this.paymentPlanId,
  }
}

// routes/repositories.ts: route-owned response serialization
function serializeRepository(entity: RepositoryEntity): RepoResponse {
  return {
    repo_name: entity.repoName,
    is_private: entity.isPrivate,
    payment_plan_id: entity.paymentPlanId,
  }
}
```

The route serializer follows the declared HTTP response schema. DynamoDB
attribute names do not determine the public API.

## DynamoDB Set Conversion

DynamoDB doesn't support empty Sets - convert to/from Arrays.

### Reading: Set → Array

```typescript
static fromItem(item: IssueFormatted): IssueEntity {
  return new IssueEntity({
    // DynamoDB Set (or undefined) → Array
    assignees: item.assignees ? Array.from(item.assignees) : [],
    labels: item.labels ? Array.from(item.labels) : [],
  })
}
```

### Writing: Array → Set (or undefined)

```typescript
toItem(): IssueInput {
  return {
    // Array → Set (only if non-empty)
    assignees: this.assignees.length > 0
      ? new Set(this.assignees)
      : undefined,
    labels: this.labels.length > 0
      ? new Set(this.labels)
      : undefined,
  }
}
```

**Critical:** DynamoDB rejects empty Sets. Always check length before creating Set.

## Timestamp Handling

DynamoDB Toolbox auto-manages `created` and `modified` timestamps.

### Schema Configuration

```typescript
// Timestamps are added automatically as _ct and _md
// When querying through Entity, they're formatted as 'created' and 'modified'
const entity = new Entity({
  name: "USER",
  table: AppTable,
  schema: item({
    username: string().required().key(),
    // created/modified added automatically
  }),
})
```

### Reading Timestamps

```typescript
static fromItem(item: UserFormatted): UserEntity {
  return new UserEntity({
    username: item.username,
    // DynamoDB Toolbox returns ISO strings
    created: DateTime.fromISO(item.created),
    modified: DateTime.fromISO(item.modified),
  })
}
```

### Writing Timestamps

```typescript
// Don't include created/modified in toItem()
// DynamoDB Toolbox handles them automatically
toItem(): UserInput {
  return {
    username: this.username,
    email: this.email,
    // created/modified omitted - auto-managed
  }
}
```

## Immutable Update Pattern

Never mutate entity properties. Return new instance with updated values.

```typescript
class UserEntity {
  public readonly username: string  // Never changes
  public readonly email: string
  public readonly bio?: string
  public readonly created: DateTime  // Never changes
  public readonly modified: DateTime

  // Immutable update
  updateUser(opts: UpdateUserEntityOpts): UserEntity {
    return new UserEntity({
      // Preserve identity
      username: this.username,
      created: this.created,
      // Apply updates
      email: opts.email ?? this.email,
      bio: opts.bio ?? this.bio,
      // Update timestamp
      modified: DateTime.utc(),
    })
  }
}
```

**Usage:**

```typescript
// Repository layer
async update(user: UserEntity): Promise<UserEntity> {
  // Create updated entity with new timestamp
  const updated = user.updateUser({
    email: "newemail@example.com",
  })

  // Save to DynamoDB
  const result = await this.entity
    .build(PutItemCommand)
    .item(updated.toItem())
    .send()

  return UserEntity.fromItem(result.ToolboxItem)
}
```

**Why immutable?**
- Predictable: No hidden state changes
- Testable: Pure functions
- Thread-safe: No race conditions
- Traceable: Clear audit trail

## Validation in fromRequest

Validate business rules when converting API request to entity.

```typescript
static fromRequest(data: RepositoryCreateRequest): RepositoryEntity {
  // Validate required fields
  if (!data.owner || data.owner.trim().length === 0) {
    throw new ValidationError("owner", "Owner is required")
  }

  // Validate format
  if (!/^[a-zA-Z0-9_-]+$/.test(data.owner)) {
    throw new ValidationError(
      "owner",
      "Owner must contain only alphanumeric characters, hyphens, and underscores"
    )
  }

  // Validate length
  if (data.repo_name.length > 100) {
    throw new ValidationError(
      "repo_name",
      "Repository name must be 100 characters or less"
    )
  }

  return new RepositoryEntity({
    owner: data.owner,
    repoName: data.repo_name,
    description: data.description,
    isPrivate: data.is_private ?? false,
  })
}
```

**Validation Layers:**
1. **Schema validation** (DynamoDB Toolbox) - Type and required fields
2. **Business validation** (Entity `fromRequest`) - Format, length, rules
3. **Existence validation** (Repository transactions) - Foreign key checks

## Helper Methods

### getEntityKey()

Returns entity identifier for error messages.

```typescript
getEntityKey(): string {
  return `ISSUE#${this.owner}#${this.repoName}#${this.issueNumber}`
}

// Usage in error handling
throw new EntityNotFoundError("IssueEntity", issue.getEntityKey())
// → "IssueEntity 'ISSUE#alice#my-repo#00000001' not found"
```

### getParentEntityKey()

Returns parent entity identifier for transaction error handling.

```typescript
getParentEntityKey(): string {
  return `REPO#${this.owner}#${this.repoName}`
}

// Usage in transaction error handling
handleTransactionError(error, {
  entityType: "IssueEntity",
  entityKey: issue.getEntityKey(),
  parentEntityType: "RepositoryEntity",
  parentEntityKey: issue.getParentEntityKey(),
})
```

## Testing Entity Transformations

Entity transformations are tested implicitly through repository tests, not in isolation.

**Don't:**
```typescript
// ❌ Don't test transformations in isolation
describe("IssueEntity", () => {
  it("should convert to an input item", () => {
    const issue = new IssueEntity(/* ... */)
    const item = issue.toItem()
    expect(item.owner).toBe(issue.owner)
  })
})
```

**Do:**
```typescript
// ✓ Test through repository layer
describe("IssueRepository", () => {
  it("should create issue and retrieve with correct data", async () => {
    const issue = IssueEntity.fromRequest({
      owner: "alice",
      repo_name: "my-repo",
      title: "Test Issue",
      author: "bob",
    })

    const created = await issueRepo.create(issue)

    // Transformations tested implicitly
    expect(created.owner).toBe("alice")
    expect(created.repoName).toBe("my-repo")
    expect(created.title).toBe("Test Issue")
  })
})
```

**Why?** Entity transformations are an implementation detail. Test the full flow (API → DynamoDB → API) through repository tests.

## Best Practices

### ✓ Do

- Implement only the transformation methods the entity needs
- Use immutable entity properties
- Return new instances from update methods
- Convert Sets ↔ Arrays (handle empty sets)
- Parse timestamps to DateTime objects
- Validate in `fromRequest()`
- Use helper methods for entity keys
- Keep HTTP serialization in routes and DynamoDB I/O in repositories

### ✗ Don't

- Don't mutate entity properties
- Don't include timestamps in `toItem()` (auto-managed)
- Don't create empty Sets (DynamoDB rejects them)
- Don't test entity transformations in isolation
- Don't include PK/SK in entity (computed by schema)

## Entity Transformation Checklist

For each domain entity:

- [ ] **Transformations** - fromRequest, fromItem, and toItem as needed
- [ ] **Immutable properties** - All fields marked `readonly`
- [ ] **Immutable updates** - `updateEntity()` returns new instance
- [ ] **Field naming** - camelCase in entity, snake_case in DB
- [ ] **Set conversion** - Arrays ↔ Sets with empty check
- [ ] **Timestamp handling** - DateTime objects, not included in toItem
- [ ] **Validation** - Route validation at HTTP; domain invariants in fromRequest
- [ ] **Helper methods** - getEntityKey() and getParentEntityKey()
- [ ] **Type exports** - EntityInput and EntityFormatted from schema
