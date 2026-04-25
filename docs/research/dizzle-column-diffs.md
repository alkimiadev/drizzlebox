# Drizzle ORM Column Builder Differences: SQLite vs PostgreSQL vs MySQL

Research based on `drizzle-orm@0.38.4` source in `node_modules/drizzle-orm`.

## 1. Column Types by Dialect

### SQLite (`drizzle-orm/sqlite-core/columns/`)

| Function     | columnType         | dataType   | Notes                                        |
|--------------|--------------------|-----------|----------------------------------------------|
| `integer()`  | `SQLiteInteger`    | `number`  | mode: `'number'` (default)                   |
| `integer()`  | `SQLiteTimestamp`  | `date`    | mode: `'timestamp'` or `'timestamp_ms'`     |
| `integer()`  | `SQLiteBoolean`   | `boolean` | mode: `'boolean'`                             |
| `text()`      | `SQLiteText`       | `string`  | mode: `'text'` (default), supports `enum`, `length` |
| `text()`      | `SQLiteTextJson`   | `json`    | mode: `'json'`                                |
| `real()`      | `SQLiteReal`       | `number`  | -                                            |
| `blob()`      | `SQLiteBlobJson`   | `json`    | mode: `'json'` (default)                     |
| `blob()`      | `SQLiteBlobBuffer` | `buffer`  | mode: `'buffer'`                              |
| `blob()`      | `SQLiteBigInt`     | `bigint`  | mode: `'bigint'`                              |
| `numeric()`   | `SQLiteNumeric`    | `string`  | -                                            |
| `customType()`| `SQLiteCustomColumn`| `custom` | -                                            |

### PostgreSQL (`drizzle-orm/pg-core/columns/`)

| Function         | columnType         | dataType   | Notes                        |
|------------------|--------------------|-----------|------------------------------|
| `integer()`      | `PgInteger`        | `number`  | -                            |
| `smallint()`     | `PgSmallInt`       | `number`  | -                            |
| `bigint()`        | `PgBigInt`         | `bigint`  |                              |
| `serial()`       | `PgSerial`         | `number`  | notNull + hasDefault built-in|
| `smallserial()`  | `PgSmallSerial`    | `number`  | notNull + hasDefault built-in|
| `bigserial()`    | `PgBigSerial`      | `bigint`  | notNull + hasDefault built-in|
| `boolean()`      | `PgBoolean`        | `boolean` | -                            |
| `text()`          | `PgText`           | `string`  | supports `enum`              |
| `varchar()`      | `PgVarchar`        | `string`  | supports `enum`, `length`    |
| `char()`          | `PgChar`           | `string`  | supports `length`, `enum`    |
| `numeric()`       | `PgNumeric`        | `string`  | precision/scale config       |
| `real()`          | `PgReal`           | `number`  | -                            |
| `doublePrecision()`| `PgDoublePrecision`| `number`  | -                            |
| `json()`          | `PgJson`           | `json`    | -                            |
| `jsonb()`         | `PgJsonb`          | `json`    | PG-specific                  |
| `uuid()`          | `PgUUID`           | `string`  | has `.defaultRandom()`       |
| `date()`          | `PgDate`/`PgDateString` | `date`/`string` | mode toggle          |
| `timestamp()`     | `PgTimestamp`/`PgTimestampString`| `date`/`string` | mode, withTimezone, precision |
| `time()`          | `PgTime`/`PgTimeString` | `string` | precision, withTimezone  |
| `interval()`      | `PgInterval`      | `string`  | PG-specific                  |
| `inet()`          | `PgInet`           | `string`  | PG network type              |
| `cidr()`          | `PgCidr`           | `string`  | PG network type              |
| `macaddr()`       | `PgMacaddr`        | `string`  | PG network type              |
| `macaddr8()`      | `PgMacaddr8`       | `string`  | PG network type              |
| `point()`         | `PgPoint`          | `string`  | PG geometric type            |
| `line()`          | `PgLine`           | `string`  | PG geometric type            |
| `pgEnum()`        | `PgEnumColumn`     | `string`  | Requires pre-declared enum   |
| `customType()`    | `PgCustomColumn`   | `custom`  | -                            |

### MySQL (`drizzle-orm/mysql-core/columns/`)

| Function         | columnType              | dataType   | Notes                              |
|------------------|-------------------------|-----------|--------------------------------------|
| `int()`           | `MySqlInt`             | `number`  | `unsigned` config, `.autoincrement()`|
| `smallint()`      | `MySqlSmallInt`        | `number`  | `unsigned`, `.autoincrement()`      |
| `mediumint()`     | `MySqlMediumInt`       | `number`  | `unsigned`, `.autoincrement()`      |
| `tinyint()`       | `MySqlTinyInt`         | `number`  | `unsigned`, `.autoincrement()`      |
| `bigint()`        | `MySqlBigInt`          | `bigint`  | `unsigned`, mode toggle             |
| `serial()`        | `MySqlSerial`          | `number`  | autoIncrement+primaryKey+notNull+default |
| `boolean()`       | `MySqlBoolean`         | `boolean` | -                                   |
| `float()`         | `MySqlFloat`           | `number`  | precision config                     |
| `double()`        | `MySqlDouble`          | `number`  | precision config                     |
| `decimal()`       | `MySqlDecimal`         | `string`  | precision/scale config               |
| `real()`          | `MySqlReal`            | `number`  | -                                    |
| `text()`          | `MySqlText`            | `string`  | supports `enum`, textType variants   |
| `tinytext()`      | `MySqlText`            | `string`  | textType: 'tinytext'                 |
| `mediumtext()`    | `MySqlText`            | `string`  | textType: 'mediumtext'               |
| `longtext()`      | `MySqlText`            | `string`  | textType: 'longtext'                  |
| `varchar()`       | `MySqlVarChar`        | `string`  | `length`, `enum`                     |
| `char()`           | `MySqlChar`            | `string`  | `length`, `enum`                     |
| `json()`          | `MySqlJson`            | `json`    | -                                    |
| `date()`          | `MySqlDate`/`MySqlDateString` | `date`/`string` | mode toggle                |
| `datetime()`      | `MySqlDateTime`/`MySqlDateTimeString` | `date`/`string` | mode, fsp          |
| `timestamp()`      | `MySqlTimestamp`/`MySqlTimestampString` | `date`/`string` | mode, fsp |
| `time()`          | `MySqlTime`/`MySqlTimeString` | `string` | fsp                             |
| `binary()`        | `MySqlBinary`          | `buffer`  | length config                        |
| `varbinary()`     | `MySqlVarBinary`       | `buffer`  | length config                        |
| `year()`          | `MySqlYear`            | `number`  | -                                    |
| `mysqlEnum()`     | `MySqlEnumColumn`     | `string`  | inline enum values                   |
| `customType()`    | `MySqlCustomColumn`   | `custom`  | -                                    |

## 2. Builder API Naming Differences

### Factory Function Names

The same concept has **different factory function names** per dialect. A tool like drizzlebox that must handle all three cannot assume a universal name space.

| Concept          | SQLite          | PostgreSQL     | MySQL           |
|-----------------|-----------------|---------------|-----------------|
| Integer         | `integer`      | `integer`     | `int`           |
| Small int       | —               | `smallint`    | `smallint`      |
| Big int         | —               | `bigint`      | `bigint`        |
| Auto-increment  | `.primaryKey()` on `integer` | `serial`/`bigserial`/`smallserial` | `.autoincrement()` on `int` etc., or `serial` |
| Boolean         | `integer({ mode: 'boolean' })` | `boolean`     | `boolean` / `tinyint` |
| Text/varchar    | `text`          | `text` / `varchar` | `text` / `varchar` |
| JSON            | `text({ mode: 'json' })` / `blob({ mode: 'json' })` | `json` / `jsonb` | `json` |
| Enum            | `text({ enum: [...] })` | `pgEnum` (separate declaration) | `mysqlEnum` |
| Timestamp       | `integer({ mode: 'timestamp' })` | `timestamp`   | `timestamp` / `datetime` |
| UUID            | `text()` (manual) | `uuid`        | —               |
| Numeric/decimal | `numeric`       | `numeric` / `decimal` | `decimal`       |
| Real/float      | `real`          | `real`        | `float` / `double` / `real` |

### Builder Class Names

All builder classes are dialect-prefixed:

- **SQLite**: `SQLiteTextBuilder`, `SQLiteIntegerBuilder`, `SQLiteColumnBuilder` (base)
- **PG**: `PgTextBuilder`, `PgIntegerBuilder`, `PgColumnBuilder` (base)
- **MySQL**: `MySqlTextBuilder`, `MySqlIntBuilder`, `MySqlColumnBuilder` (base), `MySqlColumnBuilderWithAutoIncrement`

### Column Class Names (the `columnType` field)

All column classes are also dialect-prefixed:

- **SQLite**: `SQLiteText`, `SQLiteInteger`, `SQLiteTimestamp`, `SQLiteBoolean`, `SQLiteTextJson`, `SQLiteReal`, `SQLiteNumeric`, `SQLiteBlobJson`, `SQLiteBlobBuffer`, `SQLiteBigInt`, `SQLiteCustomColumn`
- **PG**: `PgText`, `PgInteger`, `PgBoolean`, `PgJson`, `PgJsonb`, `PgUUID`, `PgEnumColumn`, `PgNumeric`, `PgVarchar`, `PgSerial`, etc.
- **MySQL**: `MySqlText`, `MySqlInt`, `MySqlBoolean`, `MySqlJson`, `MySqlEnumColumn`, `MySqlDecimal`, `MySqlVarChar`, `MySqlSerial`, etc.

## 3. Shared Column Builder API (from base `ColumnBuilder`)

All three dialect builder hierarchies share a common base class `ColumnBuilder` (in `drizzle-orm/column-builder.d.ts`) that provides these universal methods:

| Method              | Description |
|---------------------|-------------|
| `.notNull()`        | Makes column not null |
| `.default(value)`   | Set a default value |
| `.$defaultFn(fn)` / `.$default` | Dynamic runtime default |
| `.$onUpdateFn(fn)` / `.$onUpdate` | Dynamic runtime update value |
| `.primaryKey()`     | Makes column a primary key (implies notNull) |
| `.$type<T>()`       | Override the TypeScript type |
| `.generatedAlwaysAs(as, config?)` | Generated column (overridden per-dialect) |

The `ColumnBuilderRuntimeConfig` shared by all dialects:

```ts
{
  name: string;
  keyAsName: boolean;
  notNull: boolean;
  default: TData | SQL | undefined;
  defaultFn: (() => TData | SQL) | undefined;
  onUpdateFn: (() => TData | SQL) | undefined;
  hasDefault: boolean;
  primaryKey: boolean;
  isUnique: boolean;
  uniqueName: string | undefined;
  uniqueType: string | undefined;
  dataType: string;
  columnType: string;
  generated: GeneratedColumnConfig<TData> | undefined;
  generatedIdentity: GeneratedIdentityConfig | undefined;
}
```

The `ColumnBaseConfig` (shared, on the Column side):

```ts
{
  name: string;
  tableName: string;
  dataType: ColumnDataType;       // 'string' | 'number' | 'boolean' | 'array' | 'json' | 'date' | 'bigint' | 'custom' | 'buffer'
  columnType: string;             // Dialect-specific e.g. 'PgText', 'SQLiteInteger'
  data: unknown;
  driverParam: unknown;
  notNull: boolean;
  hasDefault: boolean;
  isPrimaryKey: boolean;
  isAutoincrement: boolean;
  hasRuntimeDefault: boolean;
  enumValues: string[] | undefined;
}
```

## 4. Dialect-Specific Differences

### 4.1 Dialect Discriminator

Every builder and column carries a `dialect` type tag in its `TTypeConfig`:
- **SQLite**: `{ dialect: 'sqlite' }`
- **PG**: `{ dialect: 'pg' }`
- **MySQL**: `{ dialect: 'mysql' }`

This is used in the `BuildColumn` conditional type in `column-builder.d.ts` to route to the correct column class.

### 4.2 SQLite-Specific

- **`integer()` modes**: The `integer()` function accepts `{ mode: 'number' | 'timestamp' | 'timestamp_ms' | 'boolean' }`. Based on mode, it returns different builder classes (`SQLiteIntegerBuilder`, `SQLiteTimestampBuilder`, `SQLiteBooleanBuilder`). This is a compile-time type-level dispatch, not a runtime polymorphic thing.
- **`text()` modes**: `text()` accepts `{ mode: 'text' | 'json' }`. With `mode: 'json'`, it returns `SQLiteTextJsonBuilder` (dataType `'json'`), which has `mapFromDriverValue`/`mapToDriverValue` for JSON serialization.
- **`blob()` modes**: `blob()` accepts `{ mode: 'buffer' | 'json' | 'bigint' }`. Default is `'json'` (not `'buffer'`!).
- **No native boolean/date types**: SQLite uses `integer` with mode overrides instead of dedicated boolean or date column classes.
- **`primaryKey()` on integer**: SQLite's `primaryKey()` on `integer` implies auto-increment (ROWID). The `SQLiteBaseIntegerBuilder` has a `PrimaryKeyConfig` with `autoIncrement?: boolean` and `onConflict`.
- **`generatedAlwaysAs`**: Accepts `{ mode?: 'virtual' | 'stored' }` config for generated columns.
- **Enum handling**: Enums are just `text({ enum: [...] })` — a constraint on `text`, not a separate type.

### 4.3 PostgreSQL-Specific

- **`pgEnum`**: Enums are a top-level declaration, not a column config option. You call `pgEnum('name', ['val1', 'val2'])` at module scope to create a `PgEnum` object, then use it as a column: `myEnum()`. This is fundamentally different from SQLite/MySQL enum handling.
- **Array support**: `PgColumnBuilder` has an `.array(size?)` method that returns a `PgArrayBuilder`. No other dialect has this.
- **`.unique()` with nulls**: PG's `.unique()` accepts `{ nulls: 'distinct' | 'not distinct' }` — a PG-specific option.
- **`timestamp`/`timestamptz`**: `timestamp()` has `{ withTimezone?: boolean, precision?: number, mode?: 'date' | 'string' }`. With `mode: 'string'`, returns `PgTimestampStringBuilder` (dataType `'string'`).
- **`jsonb`**: PG-specific JSON storage type distinct from `json`.
- **`uuid`**: Has `.defaultRandom()` method shortcut for `defaultRANDOM()`.
- **Identity columns**: PG supports `generatedAlwaysAsIdentity()`/`generatedByDefaultAsIdentity()` (not just `generatedAlwaysAs`), using PG sequences.
- **Index operator classes**: `ExtraConfigColumn` on PG columns supports `.asc()`, `.desc()`, `.nullsFirst()`, `.nullsLast()`, `.op(opClass)` for index definitions.
- **Network/geometry types**: `inet`, `cidr`, `macaddr`, `macaddr8`, `point`, `line` — all PG-only.
- **`interval`**: PG-specific date/time interval type.
- **`date()`**: Has `{ mode: 'date' | 'string' }` similar to timestamp.

### 4.4 MySQL-Specific

- **`.autoincrement()`**: MySQL int types inherit from `MySqlColumnBuilderWithAutoIncrement` instead of plain `MySqlColumnBuilder`. This adds an `.autoincrement()` method. `serial()` is shorthand for `int().primaryKey().notNull().default(autoincrement)`.
- **`unsigned`**: MySQL int types (`int`, `smallint`, `mediumint`, `tinyint`, `bigint`) accept `{ unsigned?: boolean }`.
- **`generatedAlwaysAs`**: Accepts `{ mode?: 'virtual' | 'stored' }` like SQLite.
- **Enum handling**: `mysqlEnum(values)` is a standalone column factory (like PG's `pgEnum`), but takes inline values instead of a pre-declared enum object.
- **Text variants**: `tinytext()`, `mediumtext()`, `longtext()` are MySQL-specific shorthands.
- **Datetime types**: `datetime()` is MySQL-specific (distinct from `timestamp`). Both have `{ mode?: 'date' | 'string', fsp?: number }`.
- **`year()`**: MySQL-specific.
- **Binary types**: `binary()`, `varbinary()` with length — MySQL-specific.
- **No array type**: MySQL has no `.array()` method.

### 4.5 Column Builder Inheritance Differences

```
ColumnBuilder (base, drizzle-orm/column-builder.js)
├── SQLiteColumnBuilder (sqlite-core/columns/common.js)
│   ├── SQLiteTextBuilder
│   ├── SQLiteTextJsonBuilder
│   ├── SQLiteBaseIntegerBuilder (adds .primaryKey() with autoIncrement config)
│   │   ├── SQLiteIntegerBuilder
│   │   ├── SQLiteTimestampBuilder
│   │   └── SQLiteBooleanBuilder
│   ├── SQLiteRealBuilder
│   ├── SQLiteNumericBuilder
│   ├── SQLiteBlobBufferBuilder / SQLiteBlobJsonBuilder / SQLiteBigIntBuilder
│   └── SQLiteCustomColumnBuilder
│
├── PgColumnBuilder (pg-core/columns/common.js, adds .array(), .unique() with nulls)
│   ├── PgTextBuilder, PgVarcharBuilder, PgCharBuilder
│   ├── PgIntegerBuilder, PgSmallIntBuilder, etc.
│   ├── PgBooleanBuilder
│   ├── PgJsonBuilder, PgJsonbBuilder
│   ├── PgUUIDBuilder (adds .defaultRandom())
│   ├── PgEnumColumnBuilder
│   ├── PgDateColumnBaseBuilder (adds .defaultNow())
│   │   ├── PgTimestampBuilder
│   │   └── PgDateBuilder
│   ├── PgNumericBuilder
│   ├── PgCustomColumnBuilder
│   └── ...network/geometry types
│
└── MySqlColumnBuilder (mysql-core/columns/common.js)
    ├── MySqlColumnBuilderWithAutoIncrement (adds .autoincrement())
    │   ├── MySqlIntBuilder
    │   ├── MySqlSmallIntBuilder
    │   ├── MySqlSerialBuilder
    │   └── ...other int types
    ├── MySqlTextBuilder
    ├── MySqlVarCharBuilder
    ├── MySqlBooleanBuilder
    ├── MySqlJsonBuilder
    ├── MySqlEnumColumnBuilder
    ├── MySqlDateColumnBaseBuilder (adds .defaultNow())
    │   ├── MySqlTimestampBuilder
    │   ├── MySqlDateTimeBuilder
    │   └── MySqlDateBuilder
    ├── MySqlDecimalBuilder
    ├── MySqlCustomColumnBuilder
    └── ...MySQL-specific types
```

## 5. Table Creation Functions

### Common Pattern

All three dialects use the same general pattern:

```ts
dialectTable(name, columns, extraConfig?)
dialectTable(name, (columnTypes) => columns, extraConfig?)
```

| Dialect  | Table Function    | Schema Variant                   |
|---------|-------------------|----------------------------------|
| SQLite  | `sqliteTable`     | `sqliteTableCreator(fn)`         |
| PG      | `pgTable`         | `pgTableCreator(fn)`             |
| MySQL   | `mysqlTable`      | `mysqlTableCreator(fn)`          |

Key differences:

- **PG**: `pgTable` extra config gets `BuildExtraConfigColumns` (columns get `ExtraConfigColumn` with `.asc()`, `.desc()`, `.nullsFirst()`, `.nullsLast()`, `.op()` for index op classes). Also supports `.enableRLS()`.
- **SQLite/MySQL**: Extra config gets `BuildColumns` without the `ExtraConfigColumn` wrapper.
- **SQLite**: Extra config types include `IndexBuilder | CheckBuilder | ForeignKeyBuilder | PrimaryKeyBuilder | UniqueConstraintBuilder`.
- **PG**: Extra config types include `AnyIndexBuilder | CheckBuilder | ForeignKeyBuilder | PrimaryKeyBuilder | UniqueConstraintBuilder | PgPolicy`.
- **MySQL**: Extra config types include `AnyIndexBuilder | CheckBuilder | ForeignKeyBuilder | PrimaryKeyBuilder | UniqueConstraintBuilder`.

The column builder callback receives dialect-specific column builders:

```ts
// SQLite
sqliteTable('users', (t) => ({ id: t.integer().primaryKey() }))

// PG
pgTable('users', (t) => ({ id: t.serial().primaryKey() }))

// MySQL
mysqlTable('users', (t) => ({ id: t.serial() }))
```

## 6. Key Takeaways for a Transform Registry / DbType Approach

### 6.1 The `columnType` String is the Key Discriminator

Each column has a unique `columnType` string (e.g., `'SQLiteText'`, `'PgJsonb'`, `'MySqlInt'`). This is the most reliable way to identify a column's dialect and specific type at runtime. The `dataType` field (`'string' | 'number' | 'boolean' | 'date' | 'json' | 'bigint' | 'array' | 'buffer' | 'custom'`) is shared across dialects but too coarse for schema generation.

### 6.2 The `dataType` Enum is the Universal Type Categories

The `ColumnDataType` union is:
```ts
'string' | 'number' | 'boolean' | 'array' | 'json' | 'date' | 'bigint' | 'custom' | 'buffer'
```

This can serve as a common "logical type" for cross-dialect schema generation, but you need dialect-specific handling for:
- `'string'` columns that are enums (check `enumValues`)
- `'date'` columns (timestamp vs date, timezone handling)
- `'json'` columns (json vs jsonb, mode-based detection)
- `'number'` columns (integer vs float vs decimal semantics)

### 6.3 Enum Handling is Radically Different

| Aspect          | SQLite                          | PG                              | MySQL                          |
|----------------|----------------------------------|--------------------------------|--------------------------------|
| Declaration    | `text({ enum: [...] })`         | `pgEnum()` at module scope     | `mysqlEnum(values)` inline     |
| Column builder | `SQLiteTextBuilder`              | `PgEnumColumnBuilder`          | `MySqlEnumColumnBuilder`        |
| columnType     | `SQLiteText`                    | `PgEnumColumn`                 | `MySqlEnumColumn`              |
| Storage        | TEXT with enum constraint        | Native `CREATE TYPE ... AS ENUM`| ENUM column type               |
| `enumValues`   | Set on builder config            | Set via `PgEnum` instance      | Set on builder constructor     |

For a schema generator, all three surfaces the `enumValues` field on the built column, so you can reliably extract enum values regardless of dialect. But the *declaration* style varies, and PG requires a pre-declared enum type.

### 6.4 Mode-Based Polymorphism

Several column types use a `mode` config to change the TypeScript type at compile time:

| Dialect  | Column      | Modes                                          | Effect on dataType |
|---------|-------------|------------------------------------------------|--------------------|
| SQLite  | `integer()` | `'number'` / `'timestamp'` / `'timestamp_ms'` / `'boolean'` | `number` / `date` / `boolean` |
| SQLite  | `text()`    | `'text'` / `'json'`                            | `string` / `json`  |
| SQLite  | `blob()`    | `'buffer'` / `'json'` / `'bigint'`             | `buffer` / `json` / `bigint` |
| PG      | `timestamp()` | `'date'` / `'string'`                        | `date` / `string`  |
| PG      | `date()`    | `'date'` / `'string'`                          | `date` / `string`  |
| PG      | `time()`    | `'string'` (default) / `'date'`?              | `string` / `date`  |
| MySQL   | `timestamp()` | `'date'` / `'string'`                        | `date` / `string`  |
| MySQL   | `datetime()` | `'date'` / `'string'`                         | `date` / `string`  |
| MySQL   | `date()`    | `'date'` / `'string'`                          | `date` / `string`  |
| MySQL   | `bigint()`  | `'number'` / `'bigint'`                       | `number` / `bigint`|

The `mode` is **not** stored in the `dataType` field on the column config directly — it's stored in the dialect-specific runtime config. The `dataType` on the built column does reflect the mode correctly (because each mode produces a different builder class with a different `dataType`).

### 6.5 Runtime Config Differences

Every column builder's `config` object extends `ColumnBuilderRuntimeConfig` with dialect-specific fields:

- **SQLite integer**: `{ autoIncrement: boolean }` (on `SQLiteBaseInteger`)
- **SQLite text**: `{ length?: number, enumValues?: string[] }`
- **PG varchar**: `{ length?: number, enumValues?: string[] }`
- **PG timestamp**: `{ withTimezone: boolean, precision?: number }`
- **PG numeric**: `{ precision?: number, scale?: number }`
- **MySQL int**: `{ unsigned?: boolean }` plus `{ autoIncrement: boolean }`
- **MySQL text**: `{ textType: 'tinytext' | 'text' | 'mediumtext' | 'longtext', enumValues?: string[] }`
- **MySQL timestamp/datetime**: `{ fsp?: number }`

### 6.6 Implications for a DbType / Transform Registry

**Recommended approach:**

1. **Use `dataType` as primary classification** — it's the cross-dialect enum that maps naturally to TypeBox schema types.
2. **Use `columnType` for dialect-specific dispatch** — when you need to handle a column differently based on its SQL type (e.g., `PgJsonb` vs `MySqlJson`).
3. **Check `enumValues`** to detect enum columns regardless of how they were declared.
4. **Don't rely on builder class hierarchy** — the builders are compile-time types that don't exist at runtime in a useful way for introspection. Instead, inspect the built column's properties (`dataType`, `columnType`, `enumValues`, runtime config).
5. **Mode detection**: For columns with mode variants, the `dataType` already reflects the mode (e.g., `integer({ mode: 'boolean' })` produces `dataType: 'boolean'`), so you don't need to check mode separately for most schema generation.
6. **Handle PG arrays specially**: PG's `.array()` wraps any column type into an array type with `dataType: 'array'`. The base column type info is preserved in `config.baseBuilder`.
7. **PG identity columns**: PG has `generatedIdentity` on columns (via `generatedAlwaysAsIdentity()`/`generatedByDefaultAsIdentity()`), which is separate from regular defaults. Check `column.generatedIdentity` or `builder._.identity`.

### 6.7 Column Introspection at Runtime

At runtime, all built columns extend `Column` which has these useful properties:

```ts
column.name          // Column name
column.dataType      // ColumnDataType: 'string' | 'number' | ... 
column.columnType    // Dialect-specific string: 'PgJsonb', 'SQLiteText', etc.
column.notNull       // boolean
column.hasDefault    // boolean
column.primary       // boolean (isPrimaryKey)
column.default       // Default value or SQL
column.enumValues    // string[] | undefined
column.isUnique      // boolean
column.uniqueName    // string | undefined
column.generated     // GeneratedColumnConfig | undefined
```

Plus dialect-specific properties accessible via the column class (e.g., `column.withTimezone` on PgTimestamp, `column.mode` on SQLiteTimestamp, etc.).

### 6.8 The `entityKind` Discriminator

Every builder and column class has a static `[entityKind]` string that uniquely identifies the class. This can be used for runtime type checking:

```
'SQLiteTextBuilder' / 'SQLiteText'
'PgTextBuilder' / 'PgText'
'MySqlTextBuilder' / 'MySqlText'
'PgColumnBuilder' (base)
'SQLiteColumnBuilder' (base)
'MySqlColumnBuilder' (base)
// etc.
```

This is set via `entityKind` symbol from `drizzle-orm/entity.js`.

## 7. Summary Table: Cross-Dialect Type Mapping

| Logical Type | SQLite Factory         | PG Factory           | MySQL Factory          | dataType |
|-------------|------------------------|---------------------|------------------------|----------|
| String      | `text()`               | `text()` / `varchar()` | `text()` / `varchar()` | `string` |
 Enum        | `text({ enum: [...] })`| `pgEnum()()`        | `mysqlEnum()()`        | `string` |
 JSON        | `text({ mode: 'json' })`| `json()` / `jsonb()`| `json()`               | `json`   |
 Number      | `integer()` / `real()` | `integer()` / `real()`| `int()` / `float()`   | `number` |
 BigInt      | `blob({ mode: 'bigint' })`| `bigint()`         | `bigint()`             | `bigint` |
 Boolean     | `integer({ mode: 'boolean' })`| `boolean()`      | `boolean()` / `tinyint()`| `boolean`|
 Date        | `integer({ mode: 'timestamp' })`| `timestamp()`   | `timestamp()` / `datetime()`| `date` |
 UUID        | `text()` (manual)      | `uuid()`            | —                      | `string` |
 Decimal     | `numeric()`            | `numeric()` / `decimal()`| `decimal()`       | `string` |
 Buffer      | `blob({ mode: 'buffer' })`| —                 | `binary()` / `varbinary()`| `buffer`|
 Array       | —                      | `.array()` on any col| —                     | `array`  |