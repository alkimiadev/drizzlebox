# Typemap Architecture Research

## Overview

`@alkdev/typemap` is a translation system that converts between four schema formats: TypeBox, Valibot, Zod, and Syntax (a TypeBox DSL string format). It implements a **N-way translation matrix** where each target can translate from any source, including itself (identity).

The architecture has three key properties we want to understand and potentially reuse:
1. **Module structure enabling tree-shaking**
2. **Translation target isolation**
3. **Guard/detection layer for runtime type dispatch**

---

## 1. Module Structure for Tree-Shaking

### Directory Layout

```
src/
  index.ts              # Public API barrel export
  guard.ts              # Runtime type detection
  options.ts            # Shared option types
  static.ts             # Static type inference utility
  syntax/
    syntax.ts           # Dispatcher (Syntax function)
    syntax-from-syntax.ts    # Identity: syntax -> syntax
    syntax-from-typebox.ts   # TypeBox -> Syntax
    syntax-from-valibot.ts   # Valibot -> Syntax (via TypeBox)
    syntax-from-zod.ts       # Zod -> Syntax (via TypeBox)
  typebox/
    typebox.ts          # Dispatcher (TypeBox function)
    typebox-from-syntax.ts   # Syntax -> TypeBox
    typebox-from-typebox.ts  # Identity: TypeBox -> TypeBox
    typebox-from-valibot.ts  # Valibot -> TypeBox
    typebox-from-zod.ts      # Zod -> TypeBox
  valibot/
    valibot.ts          # Dispatcher (Valibot function)
    valibot-from-syntax.ts   # Syntax -> Valibot (via TypeBox)
    valibot-from-typebox.ts  # TypeBox -> Valibot
    valibot-from-valibot.ts  # Identity: Valibot -> Valibot
    valibot-from-zod.ts      # Zod -> Valibot (via TypeBox)
    common.ts                # Shared Valibot type aliases
  zod/
    zod.ts               # Dispatcher (Zod function)
    zod-from-syntax.ts    # Syntax -> Zod (via TypeBox)
    zod-from-typebox.ts  # TypeBox -> Zod
    zod-from-valibot.ts  # Valibot -> Zod (via TypeBox)
    zod-from-zod.ts      # Identity: Zod -> Zod
  compile/
    compile.ts           # High-level compile API
    validator.ts         # Standard-schema wrapper for TypeCompiler
    environment.ts       # Detects eval() support
    path.ts             # JSON Pointer -> Standard Schema path conversion
    standard.ts          # Standard Schema V1 interface definition
```

### How Tree-Shaking Works

Each `from-*` file is a **self-contained translation unit** that only imports:
- The source library (e.g., `valibot`) for type definitions and runtime introspection
- The `@alkdev/typebox` runtime utilities (`ValueGuard`) for structural checks
- `../guard.ts` in the dispatcher files only

The `index.ts` re-exports everything from every `from-*` file:

```ts
export * from './typebox/typebox-from-syntax'
export * from './typebox/typebox-from-typebox'
export * from './typebox/typebox-from-valibot'
export * from './typebox/typebox-from-zod'
export { type TTypeBox, TypeBox } from './typebox/typebox'
// ... same pattern for valibot/, zod/, syntax/
```

**Tree-shaking mechanism**: When a bundler processes `import { TypeBox } from '@alkdev/typemap'`, it:
1. Parses `index.ts` and sees the re-exports
2. Follows into `typebox/typebox.ts` which imports all `from-*` files
3. The `TypeBox()` dispatcher function conditionally calls `TypeBoxFromSyntax`, `TypeBoxFromTypeBox`, `TypeBoxFromValibot`, or `TypeBoxFromZod` based on guards

**Critical limitation**: Because the dispatchers (`TypeBox()`, `Valibot()`, `Zod()`, `Syntax()`) use **runtime guards** to decide which path to take, bundlers **cannot** eliminate the unused branches. If you call `TypeBox(zodSchema)`, all four `from-*` modules are still included because the bundler cannot know at build time which guard branch will execute.

The real tree-shaking opportunity exists at the **named export level**. A consumer who only imports `TypeBoxFromZod` directly (not via the `TypeBox` dispatcher) can avoid pulling in valibot/typebox-from-syntax/etc:

```ts
import { TypeBoxFromZod } from '@alkdev/typemap'
```

However, the current `package.json` exports map has a **single entry point** (`"."`), which means there's no sub-path exports for individual translation units. The tree-shaking effectiveness depends entirely on the bundler's ability to eliminate unused `export *` re-exports.

### Build System

The `build.mjs` produces two output formats:
- **CJS**: `tsc` with `--module Node16` -> `target/build/cjs/`
- **ESM**: `tsc` with `--module ESNext` -> `target/build/esm/`, then mutates `.js` -> `.mjs` and rewrites import specifiers

The generated `package.json` for the published package:

```json
{
  "types": "./build/cjs/index.d.ts",
  "main": "./build/cjs/index.js",
  "module": "./build/esm/index.mjs",
  "esm.sh": { "bundle": false },
  "exports": {
    ".": {
      "require": { "types": "./build/cjs/index.d.ts", "default": "./build/cjs/index.js" },
      "import": { "types": "./build/esm/index.d.mts", "default": "./build/esm/index.mjs" }
    }
  }
}
```

The `"esm.sh": { "bundle": false }` directive tells esm.sh CDN not to bundle peer dependencies, which is important for keeping external libraries external.

**No sub-path exports are defined**, meaning consumers can't do granular imports like `@alkdev/typemap/typebox-from-zod`. This is a missed tree-shaking opportunity.

---

## 2. Translation Target Isolation

### The Dispatcher Pattern

Each target directory has a `dispatcher.ts` file (e.g., `typebox/typebox.ts`) that:

1. **Imports all `from-*` modules** in the same target directory
2. **Imports `guard.ts`** for runtime type detection
3. **Exposes a single overloaded function** with conditional dispatch

Example from `typebox/typebox.ts`:

```ts
import { TypeBoxFromSyntax } from './typebox-from-syntax'
import { TypeBoxFromTypeBox } from './typebox-from-typebox'
import { TypeBoxFromValibot } from './typebox-from-valibot'
import { TypeBoxFromZod } from './typebox-from-zod'
import * as g from '../guard'

export function TypeBox(...args: any[]): never {
  const [parameter, type, options] = g.Signature(args)
  return (
    g.IsSyntax(type)    ? TypeBoxFromSyntax(ContextFromParameter(parameter), type, options) :
    g.IsTypeBox(type)   ? TypeBoxFromTypeBox(type) :
    g.IsValibot(type)   ? TypeBoxFromValibot(type) :
    g.IsZod(type)       ? TypeBoxFromZod(type) :
    t.Never()
  ) as never
}
```

Key observations:
- The function uses **rest args** (`...args: any[]`) and `g.Signature()` to normalize the overloaded call signatures
- Runtime dispatch via ternary chain over guard functions
- The same pattern is duplicated for `Valibot()`, `Zod()`, and `Syntax()`

### The from-* File Pattern

Each `from-*` file is a pure translation module:

```
{name}-from-{source}.ts
```

Where `{name}` is the target and `{source}` is the input format. These files:
- Import only the **source** library types and the **target** library types
- Implement the full translation mapping (every type node in source -> target)
- Are self-contained: no cross-dependencies on other `from-*` files **except**:

### The Two-Hop Translation Pattern

Some translations don't have a direct path. Instead, they go through TypeBox as an intermediate:

```ts
// syntax-from-valibot.ts - Valibot -> Syntax (via TypeBox)
export function SyntaxFromValibot<Type extends v.BaseSchema<any, any, any>>(type: Type): TSyntaxFromValibot<Type> {
  const typebox = TypeBoxFromValibot(type)   // Valibot -> TypeBox
  const result = SyntaxFromTypeBox(typebox)  // TypeBox -> Syntax
  return result as never
}

// syntax-from-zod.ts - Zod -> Syntax (via TypeBox)
export function SyntaxFromZod<Type extends z.ZodTypeAny | z.ZodEffects<any>>(type: Type): TSyntaxFromZod<Type> {
  const typebox = TypeBoxFromZod(type)      // Zod -> TypeBox
  const result = SyntaxFromTypeBox(typebox)  // TypeBox -> Syntax
  return result as never
}
```

This means TypeBox acts as a **hub/intermediate representation (IR)**. The translation graph looks like:

```
           Syntax
          /     \
    SyntaxFrom   SyntaxFrom
        |           |
    TypeBoxFrom   TypeBoxFrom
       / \         / \
      /   \       /   \
  Valibot  Zod  Valibot  Zod
```

TypeBox is the **canonical IR**. All translations between non-TypeBox formats go through TypeBox first, then to the target. This reduces the N^2 translation problem to 2N translations (source->IR, IR->target).

---

## 3. Guard/Detection Layer

The `guard.ts` module provides two mechanisms:

### Type-Level Guards

```ts
export type SyntaxType = string
export type TypeBoxType = t.TSchema
export type ValibotType = v.BaseSchema<any, any, v.BaseIssue<any>>
export type ZodType = z.ZodTypeAny | z.ZodEffects<any>
```

These are used in the conditional type system of each dispatcher:

```ts
export type TTypeBox<Parameter extends TParameter, Type extends object | string, Result = (
  Type extends g.SyntaxType  ? TTypeBoxFromSyntax<...> :
  Type extends g.TypeBoxType ? TTypeBoxFromTypeBox<Type> :
  Type extends g.ValibotType ? TTypeBoxFromValibot<Type> :
  Type extends g.ZodType     ? TTypeBoxFromZod<Type> :
  t.TNever
)> = Result
```

### Runtime Guards

```ts
export function IsSyntax(value: unknown): value is string {
  return t.ValueGuard.IsString(value)
}

export function IsTypeBox(type: unknown): type is t.TSchema {
  return t.KindGuard.IsSchema(type)          // checks for [Symbol.for('@alkdev/typebox/Kind')]
}

export function IsValibot(type: unknown): type is v.AnySchema {
  return (
    t.ValueGuard.IsObject(type) &&
    t.ValueGuard.HasPropertyKey(type, '~standard') &&
    t.ValueGuard.IsObject(type['~standard']) &&
    t.ValueGuard.HasPropertyKey(type['~standard'], 'vendor') &&
    type['~standard'].vendor === 'valibot'
  )
}

export function IsZod(type: unknown): type is z.ZodTypeAny {
  return (
    t.ValueGuard.IsObject(type) &&
    t.ValueGuard.HasPropertyKey(type, '~standard') &&
    t.ValueGuard.IsObject(type['~standard']) &&
    t.ValueGuard.HasPropertyKey(type['~standard'], 'vendor') &&
    type['~standard'].vendor === 'zod'
  )
}
```

Key design decisions:
- **TypeBox detection** uses the internal `[Kind]` symbol (via `KindGuard.IsSchema`)
- **Valibot and Zod detection** use the `~standard` property from the Standard Schema spec
- **Syntax detection** just checks `typeof === 'string'`
- All guards use `@alkdev/typebox`'s `ValueGuard` utility functions rather than raw JS, ensuring consistency

### Signature Resolution

The `Signature()` function normalizes overloaded arguments:

```ts
// (parameter, syntax, options) -> [parameter, type, options]
// (syntax, options)            -> [{}, type, options]
// (parameter, options)          -> [parameter, type, {}]
// (syntax | type)              -> [{}, type, {}]
```

This allows the API to accept multiple calling conventions:
```ts
TypeBox({ Users: UsersSchema }, '{ id: number, name: string }', options)  // with context
TypeBox('{ id: number, name: string }', options)                          // syntax with options
TypeBox(zodSchema)                                                        // just a schema
```

---

## 4. Compile Directory

The `compile/` directory provides the high-level `Compile()` function that:
1. Accepts any schema type (Syntax string, TypeBox, Valibot, Zod)
2. Converts it to TypeBox via the `TypeBox()` dispatcher
3. Compiles the TypeBox schema into a `TypeCheck` validator using `@alkdev/typebox/compiler`
4. Wraps it in a `Validator` class that implements the Standard Schema V1 interface

This is the **consumer-facing API** that ties everything together. The `Compile` function uses the same guard/signature pattern:

```ts
export function Compile(...args: any[]): never {
  const [parameter, type, options] = g.Signature(args)
  const schema = t.ValueGuard.IsString(type) ? TypeBox(parameter, type, options) : TypeBox(type)
  const check = ResolveTypeCheck(schema)
  return new Validator(check) as never
}
```

The `Validator` class (`compile/validator.ts`) implements `StandardSchemaV1` with `~standard` property, providing:
- `.Check(value)` - validation
- `.Parse(value)` - decode/transform
- `.Errors(value)` - error iterator
- `.Code()` - generated validation code string

The `environment.ts` module detects if `eval()` is available (for JIT compilation) and falls back to dynamic validation if not.

---

## 5. Adaptation for DrizzleBox (Database Dialect Targets)

### Mapping the Pattern

| Typemap Concept | DrizzleBox Equivalent |
|---|---|
| TypeBox | **Drizzle IR** (dialect-agnostic schema representation) |
| Valibot | **SQLite dialect** |
| Zod | **PostgreSQL dialect** |
| Syntax string | *no direct equivalent* (or: SQL string templates) |
| TypeBox->Valibot | DrizzleIR->SQLite DDL |
| TypeBox->Zod | DrizzleIR->PostgreSQL DDL |
| Valibot->TypeBox | SQLite introspection->DrizzleIR |
| Guard system | Dialect detection from schema objects |

### Proposed Module Structure

```
src/
  index.ts
  guard.ts               # Detect drizzle dialect type (sqlite/postgres/mysql)
  options.ts             # Shared dialect options
  ir/
    ir.ts                # DrizzleIR dispatcher (IR function)
    ir-from-sqlite.ts    # SQLite schema -> DrizzleIR
    ir-from-postgres.ts  # PostgreSQL schema -> DrizzleIR
    ir-from-mysql.ts     # MySQL schema -> DrizzleIR
    ir-from-ir.ts        # Identity
  sqlite/
    sqlite.ts            # SQLite dispatcher
    sqlite-from-ir.ts    # DrizzleIR -> SQLite DDL/types
    sqlite-from-sqlite.ts # Identity
    sqlite-from-postgres.ts # Postgres -> SQLite (via IR)
    sqlite-from-mysql.ts   # MySQL -> SQLite (via IR)
  postgres/
    postgres.ts          # PostgreSQL dispatcher
    postgres-from-ir.ts  # DrizzleIR -> PostgreSQL DDL/types
    postgres-from-postgres.ts # Identity
    postgres-from-sqlite.ts  # SQLite -> PostgreSQL (via IR)
    postgres-from-mysql.ts   # MySQL -> PostgreSQL (via IR)
  mysql/
    mysql.ts             # MySQL dispatcher
    mysql-from-ir.ts     # DrizzleIR -> MySQL DDL/types
    mysql-from-mysql.ts  # Identity
    mysql-from-sqlite.ts # SQLite -> MySQL (via IR)
    mysql-from-postgres.ts # PostgreSQL -> MySQL (via IR)
```

### Guard Adaptation

```ts
// guard.ts
import { sqliteTable } from 'drizzle-orm/sqlite-core'
import { pgTable } from 'drizzle-orm/pg-core'
import { mysqlTable } from 'drizzle-orm/mysql-core'

export type SqliteType = ReturnType<typeof sqliteTable>
export type PostgresType = ReturnType<typeof pgTable>
export type MysqlType = ReturnType<typeof mysqlTable>

export function IsSqlite(schema: unknown): schema is SqliteType {
  // Detect via Drizzle's internal dialect markers or symbol properties
  return typeof schema === 'function' && /* check dialect symbol */
}

export function IsPostgres(schema: unknown): schema is PostgresType {
  // Similar detection
}

export function IsMysql(schema: unknown): schema is MysqlType {
  // Similar detection
}
```

### Improving Tree-Shaking Over Typemap

Typemap's current architecture has a tree-shaking weakness: the dispatcher functions pull in all translation paths. For DrizzleBox, we can improve this with **sub-path exports**:

```json
{
  "exports": {
    ".": {
      "import": "./build/esm/index.mjs",
      "require": "./build/cjs/index.js"
    },
    "./sqlite": {
      "import": "./build/esm/sqlite/sqlite.mjs",
      "require": "./build/cjs/sqlite/sqlite.js"
    },
    "./postgres": {
      "import": "./build/esm/postgres/postgres.mjs",
      "require": "./build/cjs/postgres/postgres.js"
    },
    "/mysql": {
      "import": "./build/esm/mysql/mysql.mjs",
      "require": "./build/cjs/mysql/mysql.js"
    }
  }
}
```

This allows consumers to import only what they need:

```ts
// Only pulls in sqlite + ir code
import { Sqlite } from '@alkdev/drizzlebox/sqlite'
```

Rather than the single-entry-point approach typemap uses, where the entire translation matrix is always imported.

### IR-as-Hub Pattern

Following typemap's TypeBox-as-IR pattern, DrizzleBox should use a **dialect-agnostic intermediate representation** as the hub:

```
         SQLite DDL
        /         \
  sqlite-from    from-sqlite
       |              |
       IR (Drizzle Intermediate Representation)
       |              |
  postgres-from   from-postgres
        \         /
       PostgreSQL DDL
```

This means we only need to write:
- **4 translation modules per dialect**: `dialect-from-ir` (generate), `from-dialect` (parse), `dialect-from-dialect` (identity), plus cross-dialect shortcuts that go through IR
- **Cross-dialect translations** (e.g., SQLite->PostgreSQL) are automatically composed: `PostgreSQL(IR(SQLite(schema)))`

### Key Differences from Typemap

1. **Type safety is the output, not the input**: Typemap's schemas are validation schemas. DrizzleBox's schemas are database table definitions. The "translation" is generating column types, constraints, and DDL.

2. **Dialect-specific features need escape hatches**: PostgreSQL has `JSONB`, MySQL has `ENUM`, SQLite has limited `ALTER TABLE`. The IR needs a way to express "dialect-specific" types that don't translate losslessly. This is similar to how typemap handles Valibot-specific types (like `Blob`, `Custom`) by creating custom TypeBox kinds.

3. **Peer dependency handling**: Typemap uses `peerDependencies` for valibot/zod - users only install what they use. DrizzleBox should do the same with `drizzle-orm/sqlite-core`, `drizzle-orm/pg-core`, `drizzle-orm/mysql-core`.

4. **No identity bypass needed**: In typemap, `TypeBoxFromTypeBox` just clones. In DrizzleBox, a dialect-to-same-dialect translation might normalize/validate rather than clone.

### Recommended Architecture

```ts
// Each dialect module exports:
// 1. A dispatcher function (like TypeBox()) that auto-detects input
// 2. Direct from-* functions for explicit, tree-shakeable usage
// 3. Type-only exports for the generated types

// postgres/postgres.ts
export function Pg<Schema>(schema: Schema): TPg<Schema> { /* dispatch */ }
export { PgFromIR } from './postgres-from-ir'
export { PgFromSqlite } from './postgres-from-sqlite'
export { PgFromMysql } from './postgres-from-mysql'
export { PgFromPg } from './postgres-from-pg'
```

This gives consumers two usage modes:
- **Convenience**: `Pg(schema)` - auto-detects, pulls in everything
- **Tree-shakeable**: `PgFromIR(irSchema)` - explicit, minimal imports