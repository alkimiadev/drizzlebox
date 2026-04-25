---
status: draft
last_updated: 2026-04-25
---

# DrizzleBox Architecture: Schema-First Multi-Dialect TypeBox/Drizzle Bridge

## Design Philosophy

DrizzleBox bridges TypeBox and Drizzle ORM in both directions:

1. **Drizzle → TypeBox** (current): Given a Drizzle table definition, produce a TypeBox validation schema. This is what drizzlebox does today.
2. **TypeBox → Drizzle** (new): Define a schema once using TypeBox-based custom kinds, then generate Drizzle table definitions for any supported dialect. Plugin authors write schemas; hosts decide storage backend.

The key insight is that both directions share the same **DbType IR** — a set of custom TypeBox kinds that carry both validation semantics and database metadata in one schema object. This is the hub-and-spoke pattern: the IR is the hub, dialects are the spokes, and translations go through the IR rather than directly between formats.

```
                    TypeBox Validation
                         ↑
                         │  validate / infer types
                         │
┌─────────────────────────────────────────────────────────────────┐
│                      DbType IR (the hub)                         │
│  Custom TypeBox Kinds: DbType:String, DbType:Integer, ...       │
│  Each carries inner validation schema + db metadata              │
└────────┬────────────────────────────────┬───────────────────────┘
         │                                 │
         │  toDrizzle(schema, 'sqlite')    │  toDrizzle(schema, 'postgres')
         │                                 │
         ▼                                 ▼
┌───────────────────────────┐   ┌───────────────────────────┐
│  SQLite Transform Module   │   │  PostgreSQL Transform Module│
│  @alkdev/drizzlebox/sqlite │   │  @alkdev/drizzlebox/pg      │
│  (peerDep: drizzle-orm     │   │  (peerDep: drizzle-orm      │
│   sqlite-core only)        │   │   pg-core only)             │
└───────────────────────────┘   └───────────────────────────┘
```

### Principles

1. **Schema is source of truth** — validation and database structure derive from the same definition
2. **Compose, don't replace** — DbType kinds wrap inner TypeBox schemas, they don't reimplement validation
3. **Common options first, overrides only when needed** — `primaryKey`, `notNull`, `unique` are cross-dialect; dialect-specific options only appear when they diverge
4. **Tree-shakeable by default** — import only the dialect you need; don't bundle sqlite transforms if you only use postgres
5. **Extensible** — plugin authors can register custom column types and transform rules
6. **Bidirectional eventually** — the IR enables both Drizzle→TypeBox and TypeBox→Drizzle, but we start with TypeBox→Drizzle

## The DbType IR

### Custom Kind Pattern

DbType uses TypeBox's `[Kind]` symbol as the dispatch key, following the established `TypeDef:*` namespace convention. Each DbType kind wraps an inner TypeBox schema and attaches structured database metadata:

```typescript
import { Kind, TypeRegistry, TSchema, Static } from '@alkdev/typebox'

// The core pattern: compose, don't replace
export interface TDbColumn<TInner extends TSchema = TSchema> extends TSchema {
  [Kind]: string                    // e.g. 'DbType:String', 'DbType:Integer'
  static: Static<TInner>            // TypeScript infers from inner schema
  inner: TInner                     // The TypeBox validation schema
  columnName: string                // Set by DbType.Table, not by individual columns
  db: DbColumnMeta                  // Database metadata (cross-dialect + overrides)
}
```

All column kinds share the same `TDbColumn` interface. The `[Kind]` value distinguishes them at runtime — `'DbType:String'`, `'DbType:Integer'`, `'DbType:Boolean'`, etc.

**Why wrap instead of replace?** TypeBox's built-in types carry rich validation metadata (`format`, `pattern`, `minLength`, `minimum`, `maximum`). DbType preserves all of this in `inner` while layering database semantics in `db`. A `DbType.VarChar(255)` wraps `Type.String({ maxLength: 255 })` — when you call `Value.Check(dbSchema, value)`, validation delegates to the inner schema.

### Metadata Structure

```typescript
interface DbColumnMeta {
  // Cross-dialect options — apply to all dialects unless overridden
  primaryKey?: boolean
  notNull?: boolean
  unique?: boolean
  references?: DbReferences
  default?: DbDefault

  // Dialect-specific overrides — only set when they differ from the cross-dialect default
  sqlite?: SqliteColumnOpts
  postgres?: PgColumnOpts
  mysql?: MySqlColumnOpts
}

interface DbReferences {
  table: string
  column: string
  onDelete?: 'cascade' | 'set null' | 'restrict' | 'no action'
  onUpdate?: 'cascade' | 'set null' | 'restrict' | 'no action'
}

// Symbolic defaults — each dialect translates these to native SQL
type DbDefault =
  | 'now'            // SQLite: strftime, PG: now(), MySQL: NOW()
  | 'uuid'           // SQLite: (lower(hex(randomblob(16)))), PG: gen_random_uuid()
  | 'autoincrement'   // SQLite: INTEGER PRIMARY KEY, PG: SERIAL, MySQL: AUTO_INCREMENT
  | 'current_timestamp' // Alias for 'now' with timezone context
  | SQL<unknown>     // Raw SQL expression (drizzle-orm's sql tag)
```

The key design choice: **`primaryKey`, `notNull`, `unique`, and `references` are cross-dialect by default**. You only specify `sqlite` or `postgres` overrides when a dialect needs different treatment. This eliminates the duplication problem from the original storage.md design:

```typescript
// BEFORE (storage.md — duplicated options)
DbType.String({ sqlite: { primaryKey: true }, postgres: { primaryKey: true } })

// AFTER (this design — cross-dialect by default)
DbType.String({ primaryKey: true })
```

When a dialect-specific override is needed, it merges with and can override the cross-dialect defaults:

```typescript
// JSON storage: SQLite uses text({ mode: 'json' }), PG uses jsonb()
DbType.Array(DbType.String(), { mode: 'json' })
// The transform for 'json' mode knows to use the right dialect-specific type
// No manual overrides needed for this case

// When you DO need a dialect override:
DbType.String({ format: 'uuid', postgres: { type: 'uuid' } })
// Default: text() everywhere, PG override: uuid()
```

### DbDefault: Symbolic Defaults

SQL default expressions are inherently dialect-specific. Rather than requiring users to write both `sql\`(strftime('%s', 'now'))\`` and `sql\`now()\``, we introduce symbolic defaults:

| Symbol | SQLite | PostgreSQL | MySQL |
|--------|--------|------------|-------|
| `'now'` | `strftime('%s', 'now')` (as integer epoch) | `now()` (as timestamptz) | `NOW()` |
| `'uuid'` | `lower(hex(randomblob(16)))` | `gen_random_uuid()` | `(UUID())` |
| `'autoincrement'` | Implicit on `INTEGER PRIMARY KEY` | `SERIAL` type | `AUTO_INCREMENT` |
| `'current_timestamp'` | `CURRENT_TIMESTAMP` | `CURRENT_TIMESTAMP` | `CURRENT_TIMESTAMP` |

For cases not covered by symbolic defaults, raw SQL is available via the `sql` tag:

```typescript
DbType.String({ default: sql\`(lower(hex(randomblob(4))))` })
```

### TDbTable

Table definitions group columns and carry table-level options:

```typescript
export interface TDbTable extends TSchema {
  [Kind]: 'DbType:Table'
  tableName: string
  columns: Record<string, TDbColumn>
  indexes?: TDbIndex[]
  constraints?: DbTableConstraints
}

export interface TDbIndex {
  name: string
  columns: string[]
  unique?: boolean
}
```

### DbTypeBuilder

Following TypeBox's `Type` and TypeDef's `TypeDefBuilder` pattern, `DbTypeBuilder` provides factory methods:

```typescript
class DbTypeBuilder {
  protected Create<TInner extends TSchema>(
    kind: string,
    inner: TInner,
    opts: DbColumnOpts
  ): TDbColumn<TInner> {
    const { sqlite, postgres, mysql, ...common } = opts
    return {
      [Kind]: kind,
      inner,
      columnName: '',  // Set by Table()
      db: {
        ...common,
        ...(sqlite ? { sqlite } : {}),
        ...(postgres ? { postgres } : {}),
        ...(mysql ? { mysql } : {}),
      },
    }
  }

  String(opts: DbColumnOpts & StringDbOpts = {}): TDbColumn<TString> {
    const { maxLength, format, ...dbOpts } = opts
    const inner = Type.String({ maxLength, format })
    return this.Create('DbType:String', inner, dbOpts)
  }

  Integer(opts: DbColumnOpts = {}): TDbColumn<TInteger> {
    return this.Create('DbType:Integer', Type.Integer(), opts)
  }

  Boolean(opts: DbColumnOpts = {}): TDbColumn<TBoolean> {
    return this.Create('DbType:Boolean', Type.Boolean(), opts)
  }

  Timestamp(opts: DbColumnOpts & TimestampDbOpts = {}): TDbColumn<TNumber> {
    // Stored as Unix epoch seconds (number), validated as number
    return this.Create('DbType:Timestamp', Type.Number(), opts)
  }

  Array<T extends TSchema>(items: T, opts: DbColumnOpts & { mode: 'json' } = {}): TDbColumn<TArray<T>> {
    return this.Create('DbType:Array', Type.Array(items), opts)
  }

  Object<T extends TProperties>(properties: T, opts: DbColumnOpts & { mode: 'json' } = {}): TDbColumn<TObject<T>> {
    return this.Create('DbType:Object', Type.Object(properties), opts)
  }

  Record<V extends TSchema>(values: V, opts: DbColumnOpts & { mode: 'json' } = {}): TDbColumn<TRecord<V>> {
    return this.Create('DbType:Record', Type.Record(Type.String(), values), opts)
  }

  Any(opts: DbColumnOpts & { mode: 'json' } = {}): TDbColumn<TUnknown> {
    return this.Create('DbType:Any', Type.Unknown(), opts)
  }

  Enum<T extends string[]>(values: [...T], opts: DbColumnOpts = {}): TDbColumn<TUnion<TLiteral<T[number]>[]>> {
    const inner = Type.Union(values.map(v => Type.Literal(v)))
    return this.Create('DbType:Enum', inner, { ...opts, enumValues: values })
  }

  VarChar(maxLength: number, opts: DbColumnOpts = {}): TDbColumn<TString> {
    return this.Create('DbType:VarChar', Type.String({ maxLength }), opts)
  }

  Uuid(opts: DbColumnOpts = {}): TDbColumn<TString> {
    return this.Create('DbType:Uuid', Type.String({ format: 'uuid' }), opts)
  }

  /** Mark a column as optional (nullable in DB, excluded from insert schema) */
  Optional<T extends TDbColumn>(column: T): T {
    return { ...column, [TypeBox.Optional]: true } as T
  }

  Table(name: string, columns: Record<string, TDbColumn>, opts?: DbTableOpts): TDbTable {
    const namedColumns: Record<string, TDbColumn> = {}
    for (const [key, col] of Object.entries(columns)) {
      namedColumns[key] = { ...col, columnName: key }
    }
    return {
      [Kind]: 'DbType:Table',
      tableName: name,
      columns: namedColumns,
      indexes: opts?.indexes,
      constraints: opts?.constraints,
    }
  }
}

export const DbType = new DbTypeBuilder()
```

### Kind Registration

DbType kinds register with TypeBox's TypeRegistry so that `Value.Check()` and `Value.Parse()` work on DbType schemas:

```typescript
// Delegate validation to inner schema
TypeRegistry.Set<TDbColumn>('DbType:String', (schema, value) => Value.Check(schema.inner, value))
TypeRegistry.Set<TDbColumn>('DbType:Integer', (schema, value) => Value.Check(schema.inner, value))
TypeRegistry.Set<TDbColumn>('DbType:Boolean', (schema, value) => Value.Check(schema.inner, value))
TypeRegistry.Set<TDbColumn>('DbType:Timestamp', (schema, value) => Value.Check(schema.inner, value))
TypeRegistry.Set<TDbColumn>('DbType:Array', (schema, value) => Value.Check(schema.inner, value))
TypeRegistry.Set<TDbColumn>('DbType:Object', (schema, value) => Value.Check(schema.inner, value))
TypeRegistry.Set<TDbColumn>('DbType:Record', (schema, value) => Value.Check(schema.inner, value))
TypeRegistry.Set<TDbColumn>('DbType:Any', (schema, value) => Value.Check(schema.inner, value))
TypeRegistry.Set<TDbColumn>('DbType:Enum', (schema, value) => Value.Check(schema.inner, value))
TypeRegistry.Set<TDbColumn>('DbType:VarChar', (schema, value) => Value.Check(schema.inner, value))
TypeRegistry.Set<TDbColumn>('DbType:Uuid', (schema, value) => Value.Check(schema.inner, value))
// TDbTable validates each column
TypeRegistry.Set<TDbTable>('DbType:Table', (schema, value) => {
  return Object.entries(schema.columns).every(
    ([key, col]) => Value.Check(col, value[key])
  )
})
```

### TypeGuard

A `DbGuard` namespace validates the structure of DbType schema objects (not values, but the schemas themselves):

```typescript
export namespace DbGuard {
  export function TDbColumn(schema: unknown): schema is TDbColumn {
    return IsObject(schema)
      && Kind in schema
      && typeof schema[Kind] === 'string'
      && (schema[Kind] as string).startsWith('DbType:')
      && IsObject(schema['db'])
      && TypeGuard.TSchema(schema['inner'])
  }

  export function TDbTable(schema: unknown): schema is TDbTable {
    return IsObject(schema)
      && schema[Kind] === 'DbType:Table'
      && typeof schema['tableName'] === 'string'
      && IsObject(schema['columns'])
  }

  // ... specific Kind guards
}
```

## Dialect Transforms

### Module Structure (Tree-Shakeable)

```
@alkdev/drizzlebox/
  src/
    index.ts                    # DbType IR, builder, guard, registry
    dbtype/
      types.ts                  # TDbColumn, TDbTable, DbColumnMeta interfaces
      builder.ts                # DbTypeBuilder class
      guard.ts                  # DbGuard namespace
      registry.ts               # Kind registration with TypeRegistry
      defaults.ts               # Symbolic default translations
      common.ts                 # Common column definitions (id, createdAt, updatedAt)
    sqlite/
      index.ts                  # Public API for SQLite dialect
      transform.ts              # Transform registry rules
      columns.ts                # Column mapping functions
    pg/
      index.ts                  # Public API for PostgreSQL dialect
      transform.ts              # Transform registry rules
      columns.ts                # Column mapping functions
    mysql/                       # Future
      index.ts
      transform.ts
      columns.ts
    drizzle/                     # Future: Drizzle → DbType direction
      index.ts
      from-column.ts             # Introspect Drizzle columns into DbType IR
```

Package exports for tree-shaking:

```json
{
  "exports": {
    ".": {
      "import": "./index.mjs",
      "require": "./index.cjs"
    },
    "./sqlite": {
      "import": "./sqlite.mjs",
      "require": "./sqlite.cjs",
      "peerDependencies": { "drizzle-orm": ">=0.36.0" }
    },
    "./pg": {
      "import": "./pg.mjs",
      "require": "./pg.cjs",
      "peerDependencies": { "drizzle-orm": ">=0.36.0" }
    },
    "./common": {
      "import": "./common.mjs",
      "require": "./common.cjs"
    }
  },
  "peerDependencies": {
    "@alkdev/typebox": ">=0.34.49"
  }
}
```

The core package (`@alkdev/drizzlebox`) depends only on `@alkdev/typebox`. The dialect modules (`/sqlite`, `/pg`) have `drizzle-orm` as a peer dependency. Users who only use SQLite never import PG transforms.

### Usage

```typescript
import { DbType } from '@alkdev/drizzlebox'
import { toSqlite } from '@alkdev/drizzlebox/sqlite'
// Only imports sqlite-core from drizzle-orm

const UserSchema = DbType.Table('users', {
  id: DbType.Uuid({ primaryKey: true, default: 'uuid' }),
  name: DbType.String({ notNull: true }),
  email: DbType.String({ notNull: true, format: 'email' }),
  scopes: DbType.Array(DbType.String(), { mode: 'json' }),
  createdAt: DbType.Timestamp({ notNull: true, default: 'now' }),
})

// Generate Drizzle SQLite table
const users = toSqlite(UserSchema)
// Equivalent to: sqliteTable('users', { id: text('id').primaryKey().$defaultFn(genRandomUUID), ... })
```

```typescript
import { DbType } from '@alkdev/drizzlebox'
import { toPg } from '@alkdev/drizzlebox/pg'
// Only imports pg-core from drizzle-orm

const users = toPg(UserSchema)
// Equivalent to: pgTable('users', { id: uuid('id').primaryKey().defaultRandom(), ... })
```

### Transform Registry

Each dialect module uses a priority-sorted rule registry to map DbType kinds to Drizzle column builders:

```typescript
// sqlite/transform.ts
import { TransformRegistry } from '../dbtype/registry.ts'

interface TransformContext {
  dialect: 'sqlite' | 'postgres' | 'mysql'
  ancestors: TDbColumn[]
  metadata: Record<string, unknown>
}

type ColumnTransformResult = DrizzleColumnBuilder  // from drizzle-orm

interface TransformRule {
  name: string
  match: (schema: TDbColumn, ctx: TransformContext) => boolean
  transform: (schema: TDbColumn, ctx: TransformContext) => ColumnTransformResult
  priority: number  // Lower = higher priority
}

const sqliteTransforms = new TransformRegistry<TransformRule>()

sqliteTransforms.register({
  name: 'sqlite-string',
  priority: 0,
  match: (col) => col[Kind] === 'DbType:String',
  transform: (col, ctx) => {
    const db = col.db
    const opts = resolveOpts(db, 'sqlite')
    let builder = sqliteText(col.columnName)
    if (opts.primaryKey) builder = builder.primaryKey()
    if (opts.notNull) builder = builder.notNull()
    if (opts.unique) builder = builder.unique()
    if (opts.default !== undefined) builder = applyDefault(builder, opts.default, 'sqlite')
    return builder
  },
})

sqliteTransforms.register({
  name: 'sqlite-uuid',
  priority: -1,  // Higher priority than generic string
  match: (col) => col[Kind] === 'DbType:Uuid',
  transform: (col, ctx) => {
    const db = col.db
    const opts = resolveOpts(db, 'sqlite')
    let builder = sqliteText(col.columnName)
    if (opts.primaryKey) builder = builder.primaryKey()
    if (opts.notNull) builder = builder.notNull()
    if (opts.default === 'uuid') {
      builder = builder.$defaultFn(() => crypto.randomUUID())
    }
    return builder
  },
})

sqliteTransforms.register({
  name: 'sqlite-boolean',
  priority: 0,
  match: (col) => col[Kind] === 'DbType:Boolean',
  transform: (col, ctx) => {
    const opts = resolveOpts(col.db, 'sqlite')
    let builder = sqliteInteger(col.columnName, { mode: 'boolean' })
    if (opts.notNull) builder = builder.notNull()
    if (opts.default !== undefined) builder = applyDefault(builder, opts.default, 'sqlite')
    return builder
  },
})

sqliteTransforms.register({
  name: 'sqlite-timestamp',
  priority: 0,
  match: (col) => col[Kind] === 'DbType:Timestamp',
  transform: (col, ctx) => {
    const opts = resolveOpts(col.db, 'sqlite')
    let builder = sqliteInteger(col.columnName, { mode: 'timestamp' })
    if (opts.notNull) builder = builder.notNull()
    if (opts.default) builder = applyDefault(builder, opts.default, 'sqlite')
    return builder
  },
})

sqliteTransforms.register({
  name: 'sqlite-json',
  priority: -1,  // Higher priority than string/array/object
  match: (col) => col[Kind] === 'DbType:Array' || col[Kind] === 'DbType:Object' || col[Kind] === 'DbType:Record' || col[Kind] === 'DbType:Any',
  transform: (col, ctx) => {
    const opts = resolveOpts(col.db, 'sqlite')
    let builder = sqliteText(col.columnName, { mode: 'json' })
    if (opts.notNull) builder = builder.notNull()
    if (opts.default !== undefined) builder = applyDefault(builder, opts.default, 'sqlite')
    return builder
  },
})
```

```typescript
// pg/transform.ts — analogous but using pg-core builders
const pgTransforms = new TransformRegistry<TransformRule>()

pgTransforms.register({
  name: 'pg-uuid',
  priority: -1,
  match: (col) => col[Kind] === 'DbType:Uuid',
  transform: (col, ctx) => {
    const opts = resolveOpts(col.db, 'postgres')
    let builder = pgUuid(col.columnName)
    if (opts.primaryKey) builder = builder.primaryKey()
    if (opts.notNull) builder = builder.notNull()
    if (opts.default === 'uuid') builder = builder.defaultRandom()
    return builder
  },
})

pgTransforms.register({
  name: 'pg-jsonb',
  priority: -1,
  match: (col) => ['DbType:Array', 'DbType:Object', 'DbType:Record', 'DbType:Any'].includes(col[Kind]),
  transform: (col, ctx) => {
    const opts = resolveOpts(col.db, 'postgres')
    let builder = pgJsonb(col.columnName)
    if (opts.notNull) builder = builder.notNull()
    if (opts.default !== undefined) builder = applyDefault(builder, opts.default, 'postgres')
    return builder
  },
})

pgTransforms.register({
  name: 'pg-timestamp',
  priority: 0,
  match: (col) => col[Kind] === 'DbType:Timestamp',
  transform: (col, ctx) => {
    const opts = resolveOpts(col.db, 'postgres')
    let builder = pgTimestamptz(col.columnName, { withTimezone: true })
    if (opts.notNull) builder = builder.notNull()
    if (opts.default === 'now') builder = builder.default(sql`now()`)
    return builder
  },
})
```

### Option Resolution

The `resolveOpts` function merges cross-dialect options with dialect-specific overrides:

```typescript
function resolveOpts(db: DbColumnMeta, dialect: 'sqlite' | 'postgres' | 'mysql'): ResolvedColumnOpts {
  const dialectOpts = db[dialect] ?? {}
  return {
    primaryKey: dialectOpts.primaryKey ?? db.primaryKey,
    notNull: dialectOpts.notNull ?? db.notNull,
    unique: dialectOpts.unique ?? db.unique,
    references: dialectOpts.references ?? db.references,
    default: dialectOpts.default ?? db.default,
    ...dialectOpts,  // Any dialect-specific extras
  }
}
```

### Symbolic Default Resolution

```typescript
function applyDefault(
  builder: ColumnBuilder,
  defaultVal: DbDefault | unknown,
  dialect: 'sqlite' | 'postgres' | 'mysql'
): ColumnBuilder {
  if (typeof defaultVal === 'string') {
    switch (defaultVal) {
      case 'now':
        return dialect === 'sqlite'
          ? builder.default(sql`(strftime('%s', 'now'))`)
          : dialect === 'postgres'
            ? builder.default(sql`now()`)
            : builder.default(sql`NOW()`)
      case 'uuid':
        return dialect === 'sqlite'
          ? builder.$defaultFn(() => crypto.randomUUID())
          : dialect === 'postgres'
            ? builder.defaultRandom()
            : builder.$defaultFn(() => crypto.randomUUID())
      case 'autoincrement':
        // Handled differently per dialect — usually implicit in primaryKey
        return builder
      case 'current_timestamp':
        return builder.default(sql`CURRENT_TIMESTAMP`)
    }
  }
  // SQL expression or literal value
  if (defaultVal instanceof SQL) return builder.default(defaultVal)
  return builder.default(defaultVal)
}
```

## Type Mapping Table

| DbType Kind | SQLite Column | PG Column | MySQL Column | Inner TypeBox |
|-------------|---------------|-----------|--------------|---------------|
| `DbType:String` | `text()` | `text()` | `text()` | `Type.String()` |
| `DbType:Uuid` | `text()` | `uuid()` | `varchar(36)` | `Type.String({ format: 'uuid' })` |
| `DbType:VarChar` | `text()` | `varchar(n)` | `varchar(n)` | `Type.String({ maxLength: n })` |
| `DbType:Integer` | `integer()` | `integer()` | `int()` | `Type.Integer()` |
| `DbType:Boolean` | `integer({ mode: 'boolean' })` | `boolean()` | `boolean()` | `Type.Boolean()` |
| `DbType:Timestamp` | `integer({ mode: 'timestamp' })` | `timestamptz()` | `timestamp()` | `Type.Number()` |
| `DbType:Number` | `real()` | `double precision()` | `double()` | `Type.Number()` |
| `DbType:Array` (mode: 'json') | `text({ mode: 'json' })` | `jsonb()` | `json()` | `Type.Array(T)` |
| `DbType:Object` (mode: 'json') | `text({ mode: 'json' })` | `jsonb()` | `json()` | `Type.Object(T)` |
| `DbType:Record` (mode: 'json') | `text({ mode: 'json' })` | `jsonb()` | `json()` | `Type.Record(T)` |
| `DbType:Any` (mode: 'json') | `text({ mode: 'json' })` | `jsonb()` | `json()` | `Type.Unknown()` |
| `DbType:Enum` | `text({ enum: [...] })` | `pgEnum()()` or `text()` | `mysqlEnum()()` | `Type.Union([...Type.Literal()])` |
| `DbType:Real` | `real()` | `real()` | `float()` | `Type.Number()` |

### Notes on Specific Mappings

**UUID**: SQLite has no native UUID type. We use `text()` with a JS-side `$defaultFn` for UUID generation. PG gets the native `uuid()` type with `.defaultRandom()`. This is a case where the DbType kind (`DbType:Uuid`) maps to entirely different column types per dialect.

**Timestamp**: SQLite stores as integer epoch seconds, PG as `timestamptz`. The symbolic default `'now'` resolves to `strftime('%s', 'now')` for SQLite (returning a Unix epoch integer) and `now()` for PG (returning a timestamptz). This is the other case where dialect divergence is hidden behind a single DbType kind.

**JSON**: All compound types (`Array`, `Object`, `Record`, `Any`) with `mode: 'json'` map to `text({ mode: 'json' })` in SQLite, `jsonb()` in PG, and `json()` in MySQL. The transform registry picks the right one based on dialect.

**Enum**: This is the most problematic mapping. PG requires `pgEnum()` at module scope (a separate type declaration), while SQLite uses `text({ enum: [...] })` and MySQL uses `mysqlEnum()()`. See Open Question #1.

## Common Columns

```typescript
// dbtype/common.ts
export const commonCols = {
  id: DbType.Uuid({ primaryKey: true, default: 'uuid' }),
  createdAt: DbType.Timestamp({ notNull: true, default: 'now' }),
  updatedAt: DbType.Timestamp({ notNull: true, default: 'now' }),
}

// Usage:
const UserSchema = DbType.Table('users', {
  ...commonCols,
  name: DbType.String({ notNull: true }),
  email: DbType.String({ notNull: true, format: 'email' }),
})
```

Compare to the storage.md version:

```typescript
// BEFORE — repeated dialect config for identical behavior
createdAt: DbType.Timestamp({
  sqlite: { notNull: true, default: sql`(strftime('%s', 'now'))` },
  postgres: { notNull: true, default: sql`now()` }
}),

// AFTER — cross-dialect defaults + symbolic default
createdAt: DbType.Timestamp({ notNull: true, default: 'now' }),
```

## Validation Schemas from DbType

Because DbType schemas carry `inner` TypeBox schemas, extracting validation schemas is straightforward:

```typescript
export function createSelectSchema(table: TDbTable): TObject {
  const properties: Record<string, TSchema> = {}
  for (const [name, col] of Object.entries(table.columns)) {
    properties[name] = col[Kind] === 'DbType:Table'
      ? col.inner  // Unwrap to get the inner TypeBox schema
      : col.inner
  }
  return Type.Object(properties)
}

export function createInsertSchema(table: TDbTable): TObject {
  const properties: Record<string, TSchema> = {}
  for (const [name, col] of Object.entries(table.columns)) {
    if (col.db.primaryKey && col.db.default === 'autoincrement') continue  // Skip auto-increment PKs
    let schema = col.inner
    if (isOptional(col)) schema = Type.Optional(schema)
    if (!col.db.notNull) schema = Type.Optional(Type.Union([schema, Type.Null()]))
    properties[name] = schema
  }
  return Type.Object(properties)
}
```

This means plugins define schemas once and get both validation and Drizzle table generation from the same source.

## Bidirectional Support (Future)

The IR design enables both directions:

```
Drizzle Column ──→ fromDrizzle(column) ──→ DbType IR ──→ toDrizzle(schema, dialect)
                                              │
                                              └──→ inner ──→ TypeBox validation schema
```

`fromDrizzle()` would introspect a Drizzle column and produce a `TDbColumn` with populated `db` metadata and an inferred `inner` TypeBox schema. This would be a straightforward mapping since Drizzle columns carry all the metadata we need (`dataType`, `columnType`, `notNull`, `hasDefault`, `enumValues`, etc.).

The current `columnToSchema()` in drizzlebox already does the column→TypeBox part. The enhancement would be wrapping the result in a `TDbColumn` with the `db` metadata preserved.

## Nullability Convention

Following the storage.md convention but simplified:

- `DbType.Optional(column)` — nullable in DB, excluded from insert schema
- `{ notNull: true }` — required (non-nullable) in DB, included in insert schema
- Neither — technically nullable, but prefer explicit `Optional()` or `notNull: true`

## Open Questions

### 1. PostgreSQL Enum Handling

**Problem**: PG requires `pgEnum()` at module scope before tables can reference it:

```typescript
// PG requires this:
const moodEnum = pgEnum('mood', ['happy', 'sad', 'neutral'])
const users = pgTable('users', { mood: moodEnum('mood') })

// vs. SQLite:
const users = sqliteTable('users', { mood: text('mood', { enum: ['happy', 'sad', 'neutral'] }) })
```

**Options**:

**A. Generate enum declarations separately**: `toPg(schema)` returns both enum declarations and table definitions:
```typescript
const { enums, tables } = toPg(UserSchema)
// enums: { mood: pgEnum('mood', ['happy', 'sad', 'neutral']) }
// tables: { users: pgTable('users', { mood: moodEnum('mood') }) }
```

**B. Start with `text()` for all dialects**: Don't generate native PG enums initially. Use `text()` with a check constraint or validation-only enum. Add `pgEnum` support as an explicit opt-in later.

**C. Per-column opt-in**: `DbType.Enum({ values: [...], postgres: { nativeEnum: true } })` — only generates `pgEnum` when explicitly requested.

**Current leaning**: **B** — start simple, add native enum support later as an opt-in feature. This avoids structural differences in the output between dialects.

### 2. Mode Inference vs. Explicit Annotation

**Problem**: Should `DbType.Object({...})` without `mode: 'json'` automatically infer `mode: 'json'` for storage?

**Options**:

**A. Require explicit mode**: All compound types must specify `mode: 'json'`. More verbose but unambiguous.

**B. Auto-infer**: `DbType.Array()`, `DbType.Object()`, `DbType.Record()`, `DbType.Any()` automatically infer `mode: 'json'` since there's no other reasonable storage mode for compound types in relational databases.

**Current leaning**: **B** — auto-infer `mode: 'json'` for compound types that must be stored as JSON. This matches the storage.md proposal. Scalar columns that happen to store JSON (like a string column holding JSON data) would need explicit annotation.

### 3. The `inner` Schema: Who Constructs It?

**Problem**: Should the DbTypeBuilder auto-infer the TypeBox inner schema from the column type, or should users provide it explicitly?

**Options**:

**A. Auto-infer**: `DbType.String({ notNull: true })` automatically creates `Type.String()` as inner. Users can override with explicit `inner` if they want validation constraints:
```typescript
DbType.String({ notNull: true })  // inner = Type.String()
DbType.String({ notNull: true, inner: Type.String({ format: 'email', maxLength: 255 }) })
```

**B. Always explicit**: Users always provide the inner schema:
```typescript
DbType.String(Type.String({ format: 'email' }), { notNull: true })
```

**C. Builder methods with validation sugar**: Builder methods that set both inner and db metadata:
```typescript
DbType.Email()  // inner = Type.String({ format: 'email' }), [Kind] = 'DbType:String'
DbType.Uuid()   // inner = Type.String({ format: 'uuid' }), [Kind] = 'DbType:Uuid'
```

**Current leaning**: **A+C** — auto-infer by default, with convenience builder methods for common patterns. The `inner` field is an escape hatch for custom validation constraints.

### 4. Dialect-Specific Types That Don't Map Cleanly

**Problem**: Some PG types have no SQLite equivalent (geometric types, `inet`, `cidr`, `macaddr`, array types). Some SQLite modes have no PG equivalent (`blob({ mode: 'bigint' })`).

**Options**:

**A. Dialect-specific Kind escapes**: `DbType.PgGeometry()`, `DbType.SqliteBlob()` — kinds that only work in one dialect, fail in others.

**B. Common abstraction where possible, escape hatches otherwise**: `DbType.Array(inner, { mode: 'json' })` works everywhere (stores as JSON). PG-native arrays via `{ postgres: { nativeArray: true } }` override.

**C. Only support the common subset**: Don't generate dialect-specific types from DbType at all. Users write raw Drizzle for those columns.

**Current leaning**: **B** — start with the common subset, provide dialect-specific overrides for escape hatches. The `postgres` and `sqlite` options bags exist for this reason.

### 5. Should `toDrizzle` Return Table Objects or Builder Callbacks?

**Problem**: Drizzle tables are typically created with a callback that receives column builders:

```typescript
sqliteTable('users', (t) => ({ id: t.integer().primaryKey(), name: t.text() }))
```

But our transform produces column-by-column. Should we produce:

**A. Table object directly**: Return the result of `sqliteTable(name, columns)` — simpler, but the callback pattern gives access to `t` for dialect-specific features not expressible in DbType.

**B. Column definitions only**: Return just the columns record, let users call `sqliteTable(name, columns)` themselves — more flexible but more verbose.

**C. Table object with extra config callback**: Return the table but accept an `extraConfig` callback for indexes, unique constraints, etc.

**Current leaning**: **C** — return the table object, handle indexes from `TDbTable.indexes`. For extra config not expressible in DbType, users can use the table extra config pattern separately. The `toDrizzle` function should handle the common 90%.

### 6. Default Value Types: Symbolic vs Raw

**Problem**: The `DbDefault` type currently supports a fixed set of symbolic strings plus raw SQL. What about:
- Literal defaults: `{ default: 0 }` or `{ default: '' }`
- JS-side defaults: `{ default: () => crypto.randomUUID() }`
- The difference between SQL defaults and JS-side defaults (Drizzle's `.default()` vs `.$defaultFn()`)

**Current approach**: Symbolic strings for common patterns, `sql` tagged template for SQL expressions, literal values for simple cases. The transform layer decides `.default()` vs `.$defaultFn()` based on the dialect and symbol.

**Open question**: Should we also support a function form for JS-side defaults?

```typescript
DbType.String({
  default: 'uuid',  // Symbolic — transform decides implementation
  // vs.
  default: () => crypto.randomUUID(),  // JS-side — always uses $defaultFn
})
```

**Current leaning**: Support both. Symbolic defaults are translated to the appropriate mechanism per dialect. JS function defaults always use `$defaultFn`. Raw SQL uses `.default(sql\`...\`)`.

### 7. Relation Definitions

**Problem**: Foreign keys work via column config `references`, but complex relations (many-to-many, join tables) need explicit relation definitions. Should these be part of `DbType.Table` or separate?

**Status**: Deferred, same as storage.md. `references` on column config covers the common case. Complex relations can be added later via `TDbRelation` or through Drizzle's relation API directly.

### 8. Migration Generation

**Problem**: When should migrations be generated — at build time or at runtime?

**Status**: Same as storage.md — build-time for now via `drizzle-kit`. The DbType schema → Drizzle table → `drizzle-kit generate` pipeline. Dynamic plugin registration is a future concern.

### 9. Current drizzlebox Direction: Keep, Evolve, or Replace?

**Problem**: The current `drizzlebox` does Drizzle → TypeBox (generating validation schemas from existing Drizzle tables). The new DbType IR does TypeBox → Drizzle (generating Drizzle tables from TypeBox-based schemas). These are opposite directions.

**Options**:

**A. Keep both directions in one package**: The current `columnToSchema()` becomes `fromDrizzle()`, the new transform registry becomes `toDrizzle()`. Both use the same DbType IR as intermediate. Package exports both directions.

**B. Keep current direction, add new as separate sub-package**: The current `@alkdev/drizzlebox` continues as-is. The new TypeBox→Drizzle direction lives in `@alkdev/drizzlebox/schema` or a separate package.

**C. Replace current direction eventually**: Phase out the current Drizzle→TypeBox in favor of the schema-first direction. Users define DbType schemas, get both validation and Drizzle for free. No need to reverse-engineer schemas from Drizzle.

**Current leaning**: **A** — keep both. The Drizzle→TypeBox direction is useful for existing Drizzle users who want validation without rewriting their schemas. The TypeBox→Drizzle direction is for the schema-first use case. They coexist using the same IR as a bridge. The `fromDrizzle(column)` function introspects an existing Drizzle column and produces a `TDbColumn` with the inner TypeBox schema and `db` metadata.

### 10. Naming Boundaries

**Question**: The current package is `@alkdev/drizzlebox` with a focus on TypeBox↔Drizzle. The new DbType IR is more general — it could be its own package. Should:

- The DbType IR live in `@alkdev/drizzlebox` (keeping everything together)?
- The DbType IR be a separate `@alkdev/dbtype` package that drizzlebox depends on?
- The DbType IR live in `@alkdev/typebox` as an extension (since it uses TypeBox's Kind/TypeRegistry)?

**Current leaning**: Keep in `@alkdev/drizzlebox` for now. The DbType IR's sole purpose is bridging TypeBox and Drizzle. If it becomes useful outside that context, we can factor it out later. The package name `drizzlebox` is already ambiguous enough to encompass both directions.

## Example: Full Table Definition

```typescript
import { DbType } from '@alkdev/drizzlebox'
import { toSqlite } from '@alkdev/drizzlebox/sqlite'
import { toPg } from '@alkdev/drizzlebox/pg'

const IdentitySchema = DbType.Table('identities', {
  id: DbType.Uuid({ primaryKey: true, default: 'uuid' }),
  keyHash: DbType.String({ notNull: true, unique: true }),
  ownerId: DbType.String({ notNull: true }),
  type: DbType.Enum(['api_key', 'node_identity'], { notNull: true }),
  scopes: DbType.Array(DbType.String(), { notNull: true }),
  roles: DbType.Optional(DbType.Array(DbType.String())),
  resources: DbType.Optional(DbType.Record(DbType.Array(DbType.String()))),
  name: DbType.Optional(DbType.String()),
  enabled: DbType.Boolean({ default: true }),
  createdAt: DbType.Timestamp({ notNull: true, default: 'now' }),
  lastUsedAt: DbType.Optional(DbType.Timestamp()),
  revokedAt: DbType.Optional(DbType.Timestamp()),
}, {
  indexes: [
    { name: 'idx_identities_owner', columns: ['ownerId'] },
    { name: 'idx_identities_type', columns: ['type'] },
  ],
})

// Generate for SQLite
const sqliteIdentities = toSqlite(IdentitySchema)

// Generate for PostgreSQL
const pgIdentities = toPg(IdentitySchema)

// Validation (extract inner TypeBox schemas)
import { createSelectSchema, createInsertSchema } from '@alkdev/drizzlebox'

const SelectIdentity = createSelectSchema(IdentitySchema)
const InsertIdentity = createInsertSchema(IdentitySchema)
```

Compare with the storage.md version:
- No per-dialect config for `primaryKey`, `notNull`, `unique`, `references` — same meaning everywhere
- Default values use symbolic `'now'` and `'uuid'` instead of dialect-specific SQL
- `mode: 'json'` is auto-inferred for `Array`, `Record`, `Object`
- Dialect overrides are only needed when types actually differ (e.g., native UUID in PG)

## Benefits

| Benefit | Description |
|---------|-------------|
| Single source of truth | Schema defined once, used for both validation and DB structure |
| No duplication | Cross-dialect options specified once, not per-dialect |
| Tree-shakeable | Import only the dialect you need |
| Type safety | TypeScript types from same schema as DB |
| Validation built-in | TypeBox schemas work for request/response validation |
| Extensible | Custom column kinds via TypeRegistry |
| Symbolic defaults | Common patterns like `'now'` and `'uuid'` translate automatically |
| Bidirectional (future) | Same IR supports Drizzle→TypeBox and TypeBox→Drizzle |