# TypeDef Kind-Based Extension Pattern

## Overview

The `@alkdev/typebox` TypeDef example (`example/typedef/typedef.ts`) demonstrates how to build a fully custom type system on top of TypeBox's Kind-based extensibility. It replaces JSON Schema semantics with a flat, binary-protocol-oriented type vocabulary while reusing TypeBox's infrastructure (Kind symbol, TypeRegistry, schema interfaces).

This document analyzes the pattern in detail and maps it to a future `DbType` system for drizzlebox.

---

## 1. The Kind Symbol — Core Extension Mechanism

**Location**: `@alkdev/typebox/src/type/symbols/symbols.ts:38`

```ts
export const Kind = Symbol.for('TypeBox.Kind')
```

`Kind` is a global Symbol (`Symbol.for`) used as a property key on every schema object. Its value is a string that identifies the type's identity, e.g. `'String'`, `'TypeDef:String'`, `'Object'`.

Every TypeBox schema interface extends `TSchema`, which extends `TKind`:

```ts
// schema.ts:49-57
export interface TKind {
  [Kind]: string
}
export interface TSchema extends TKind, SchemaOptions {
  [ReadonlyKind]?: string
  [OptionalKind]?: string
  [Hint]?: string
  params: unknown[]
  static: unknown
}
```

**Key insight**: The `[Kind]` property is the single dispatch key for the entire type system. Validation, type guarding, compilation — everything dispatches on `schema[Kind]`.

**Namespacing convention**: TypeDef uses a colon-separated namespace: `'TypeDef:String'`, `'TypeDef:Int8'`, `'TypeDef:Struct'`. This avoids collisions with TypeBox's built-in kinds (`'String'`, `'Number'`, `'Object'`).

---

## 2. Defining Custom Kind Interfaces

Each custom type declares a TypeScript interface extending `Types.TSchema` (imported as `Types from '@alkdev/typebox/type'`):

```ts
export interface TString extends Types.TSchema {
  [Types.Kind]: 'TypeDef:String'
  type: 'string'
  static: string
}

export interface TInt8 extends Types.TSchema {
  [Types.Kind]: 'TypeDef:Int8'
  type: 'int8'
  static: number
}

export interface TStruct<T extends TFields = TFields> extends Types.TSchema, StructMetadata {
  [Types.Kind]: 'TypeDef:Struct'
  static: StructStatic<T, this['params']>
  optionalProperties: { [K in Assert<OptionalKeys<T>, keyof T>]: T[K] }
  properties: { [K in Assert<RequiredKeys<T>, keyof T>]: T[K] }
}
```

**Pattern anatomy**:
- `[Types.Kind]` — literal string type for dispatch identity
- `static` — mapped TypeScript type for `Static<T>` inference
- Domain-specific properties (`type`, `elements`, `properties`, `optionalProperties`, `discriminator`, `mapping`, `enum`, `additionalProperties`)
- Generic type parameters for compositional types (`TStruct<T>`, `TArray<T>`, `TRecord<T>`)

**For DbType**: We would define interfaces like:

```ts
export interface TDbVarChar<TInner extends TSchema = TSchema> extends TSchema {
  [Kind]: 'DbType:VarChar'
  static: Static<TInner>
  inner: TInner        // the validation schema (e.g. TString with maxLength)
  maxLength: number    // database metadata
}

export interface TDbSerial<TInner extends TSchema = TSchema> extends TSchema {
  [Kind]: 'DbType:Serial'
  static: Static<TInner>
  inner: TInner
  dataType: 'integer'
}
```

The `inner` field carries the validation schema. The database-specific metadata (`maxLength`, `dataType`, `precision`, etc.) lives alongside it.

---

## 3. TypeDefBuilder.Create() — Metadata Attachment

**Location**: `typedef.ts:522-525`

```ts
export class TypeDefBuilder {
  protected Create(schema: Record<PropertyKey, any>, metadata: Record<keyof any, any>): any {
    const keys = globalThis.Object.getOwnPropertyNames(metadata)
    return keys.length > 0 ? { ...schema, metadata: { ...metadata } } : { ...schema }
  }
  // ...
}
```

Each builder method calls `this.Create(...)` with:
1. **The schema object** — containing `[Kind]` and structural properties
2. **A metadata bag** — arbitrary key-value pairs

If metadata is non-empty, it's spread into a `metadata` sub-object. Otherwise, just the schema is returned plain.

Example builder methods:

```ts
public Int8(metadata: Metadata = {}): TInt8 {
  return this.Create({ [Types.Kind]: 'TypeDef:Int8', type: 'int8' }, metadata)
}

public Struct<T extends TFields>(fields: T, metadata: StructMetadata = {}): TStruct<T> {
  // ... computes optionalProperties and properties ...
  return this.Create({ [Types.Kind]: 'TypeDef:Struct', ...requiredObject, ...optionalObject }, metadata)
}
```

**For DbType**: The `Create()` pattern works well, but we'd modify it to:
- Flatten database metadata into the schema directly (rather than nesting in a `metadata` sub-object) since tools like drizzle-kit need to inspect these properties
- Or keep the `metadata` pattern but make it type-safe with a `DbMetadata` interface

---

## 4. TypeRegistry.Set — Validation Function Registration

**Location**: `@alkdev/typebox/src/type/registry/type.ts`

```ts
export type TypeRegistryValidationFunction<TSchema> = (schema: TSchema, value: unknown) => boolean

const map = new Map<string, TypeRegistryValidationFunction<any>>()

export function Set<TSchema = unknown>(kind: string, func: TypeRegistryValidationFunction<TSchema>) {
  map.set(kind, func)
}
export function Get(kind: string) {
  return map.get(kind)
}
export function Has(kind: string) {
  return map.has(kind)
}
```

TypeDef registers all its custom kinds at module level:

```ts
Types.TypeRegistry.Set<TInt8>('TypeDef:Int8', (schema, value) => ValueCheck.Check(schema, value))
Types.TypeRegistry.Set<TString>('TypeDef:String', (schema, value) => ValueCheck.Check(schema, value))
Types.TypeRegistry.Set<TStruct>('TypeDef:Struct', (schema, value) => ValueCheck.Check(schema, value))
// ... etc
```

**How TypeBox's built-in ValueCheck uses this** (`value/check/check.ts:423-500`):
```ts
function FromKind(schema: TSchema, references: TSchema[], value: unknown): boolean {
  if (!TypeRegistry.Has(schema[Kind])) return false
  const func = TypeRegistry.Get(schema[Kind])!
  return func(schema, value)
}

function Visit(schema, references, value) {
  switch (schema[Kind]) {
    case 'String': return FromString(...)
    case 'Number': return FromNumber(...)
    // ... all built-in kinds
    default:
      if (!TypeRegistry.Has(schema[Kind])) throw new ValueCheckUnknownTypeError(schema)
      return FromKind(schema, references, value)
  }
}
```

**Key observation**: TypeBox's own `Visit()` handles all built-in kinds in a switch statement. Custom kinds hit the `default` branch and dispatch through `TypeRegistry`. This means TypeDef provides its **own** `ValueCheck.Visit()` that dispatches on `TypeDef:*` kinds — it does NOT rely on TypeBox's `Visit()` at all. TypeDef's `ValueCheck` is a separate, independent validation path.

**For DbType**: There are two approaches:
1. **TypeDef-style**: Build a completely separate `DbValueCheck.Visit()` with its own switch statement. Full control but duplicates infrastructure.
2. **Registry-style**: Register `DbType:*` validation functions with `TypeRegistry.Set()` and let TypeBox's existing `ValueCheck` dispatch through `FromKind`. This is simpler and integrates with TypeBox's existing validation pipeline.

Option 2 is preferable for DbType since we want to compose with existing TypeBox types, not replace them.

---

## 5. TypeGuard — Schema Structure Validation

**Location**: `typedef.ts:376-477`

TypeDef implements its own `TypeGuard` namespace that validates the **structure** of schema objects (not values, but the schemas themselves):

```ts
export namespace TypeGuard {
  export function TInt8(schema: unknown): schema is TInt8 {
    return IsObject(schema) && schema[Types.Kind] === 'TypeDef:Int8' && schema['type'] === 'int8'
  }

  export function TStruct(schema: unknown): schema is TStruct {
    if(!(IsObject(schema) && schema[Types.Kind] === 'TypeDef:Struct' && IsOptionalBoolean(schema['additionalProperties']))) return false
    // ... validate properties and optionalProperties
  }

  export function TSchema(schema: unknown): schema is Types.TSchema {
    return (
      TArray(schema) ||
      TBoolean(schema) ||
      // ... all TypeDef kinds ...
      TStruct(schema) ||
      TTimestamp(schema) ||
      (TKind(schema) && Types.TypeRegistry.Has(schema[Types.Kind]))  // fallback to registry
    )
  }
}
```

**Fallback to registry**: The last clause `(TKind(schema) && Types.TypeRegistry.Has(schema[Types.Kind]))` is crucial — it allows kinds registered with `TypeRegistry` but not known to the static `TSchema()` guard to still pass. This enables extensibility.

**For DbType**: We need a `DbGuard` namespace that:
- Validates `DbType:*` schema shapes (checks that `inner` is a valid schema, that `maxLength` is a number, etc.)
- Falls back to TypeBox's built-in `TypeGuard.TSchema()` for non-DbType schemas
- Provides a unified `isDbSchema()` function

---

## 6. TypeSystem.Type() — The "Simple" Custom Type API

**Location**: `@alkdev/typebox/src/system/system.ts:55-58`

```ts
export namespace TypeSystem {
  export function Type<Type, Options = Record<PropertyKey, unknown>>(
    kind: string,
    check: (options: Options, value: unknown) => boolean
  ): TypeFactoryFunction<Type, Options> {
    if (TypeRegistry.Has(kind)) throw new TypeSystemDuplicateTypeKind(kind)
    TypeRegistry.Set(kind, check)
    return (options: Partial<Options> = {}) => Unsafe<Type>({ ...options, [Kind]: kind })
  }
}
```

This is TypeBox's built-in shortcut for simple custom types:
1. Registers a validation function with `TypeRegistry`
2. Returns a factory that creates `TUnsafe<Type>` schemas with the custom `[Kind]`

The `TUnsafe` type (`unsafe.ts`):
```ts
export interface TUnsafe<T> extends TSchema {
  [Kind]: string
  static: T
}
export function Unsafe<T>(options: UnsafeOptions = {}): TUnsafe<T> {
  return CreateType({ [Kind]: options[Kind] ?? 'Unsafe' }, options) as never
}
```

**Limitation for DbType**: `TUnsafe` provides static type inference but no structural guarantees. A `DbType:VarChar` created via `TypeSystem.Type()` would have `static: string` but the schema object wouldn't encode `inner`, `maxLength`, etc. in a type-safe way. TypeDef's pattern of explicit interfaces is strictly better for complex types.

---

## 7. Relationship Between TypeDef Types and TypeBox Built-in Types

TypeDef **replaces** all JSON Schema types. It does not compose with them:
- TypeBox `String` has `kind: 'String'`, supports `minLength`, `maxLength`, `pattern`, `format`
- TypeDef `String` has `kind: 'TypeDef:String'`, supports only `type: 'string'` and metadata

TypeDef's `ValueCheck` is entirely separate from TypeBox's `ValueCheck`. They dispatch on disjoint Kind namespaces (`'TypeDef:*'` vs `'*'`).

TypeDef's `TypeGuard.TSchema()` does reference `Types.TypeRegistry.Has()` as a fallback, allowing registered types to pass. But the structural validation of TypeDef schemas is wholly custom.

**For DbType**: We want **composition**, not replacement. A `DbType:VarChar` should wrap a TypeBox `TString` (with `maxLength`) and add `dB: { kind: 'varchar', maxLength: 255 }`. This means DbType schemas should carry a reference to the inner TypeBox schema, not reimplement validation logic.

---

## 8. What the Current drizzlebox/src Approach Takes From This Pattern

Looking at `column.ts:66-67`:
```ts
TypeRegistry.Set('Buffer', (_, value) => value instanceof Buffer);
export const bufferSchema: BufferSchema = { [Kind]: 'Buffer', type: 'buffer' } as any;
```

This is a **minimal** application of the Kind/TypeRegistry pattern — a single custom type for Buffer validation. It uses the `TypeSystem.Type()`-style approach: register a check function, create a schema object with `[Kind]`.

Similarly, `utils.ts:18-27` defines `JsonSchema` and `BufferSchema` as interfaces extending `TSchema`:
```ts
export interface JsonSchema extends TSchema {
  [Kind]: 'Union'
  static: Json
  anyOf: Json
}
export interface BufferSchema extends TSchema {
  [Kind]: 'Buffer'
  static: Buffer
  type: 'buffer'
}
```

The rest of `column.ts` maps drizzle column types to **standard TypeBox types** (`t.String()`, `t.Integer()`, `t.Number()`, etc.) — meaning all the database type information (that something is a `PgInteger` vs a `MySqlInt`) is lost. The schema only preserves validation semantics.

---

## 9. Proposed DbType Pattern — Key Design Decisions

### 9.1 Wrap, Don't Replace

Each DbType schema should carry an `inner` TypeBox schema AND database metadata:

```ts
export interface TDbInteger<TInner extends TSchema = TSchema> extends TSchema {
  [Kind]: 'DbType:Integer'
  static: Static<TInner>
  inner: TInner
  db: {
    dataType: 'integer'
    columnType: string  // 'PgInteger' | 'MySqlInt' | etc.
    unsigned?: boolean
    hasDefault?: boolean
    notNull?: boolean
  }
}
```

This preserves both validation semantics AND database semantics in one schema object.

### 9.2 Register with TypeRegistry for Composition

```ts
TypeRegistry.Set<TDbInteger>('DbType:Integer', (schema, value) => {
  return Value.Check(schema.inner, value)  // delegate to inner schema
})
```

This lets DbType schemas compose with TypeBox's existing validation pipeline.

### 9.3 Separate TypeGuard for DbType Schemas

```ts
export namespace DbGuard {
  export function TDbInteger(schema: unknown): schema is TDbInteger {
    return IsObject(schema)
      && schema[Kind] === 'DbType:Integer'
      && IsObject(schema['db'])
      && TypeGuard.TSchema(schema['inner'])
  }
  // ... etc
}
```

### 9.4 Builder Pattern with Column Metadata

```ts
export class DbBuilder {
  protected Create<T extends TSchema>(schema: Record<PropertyKey, any>, inner: T, db: DbColumnMeta): any {
    return { ...schema, inner, db }
  }

  public Integer(column: Column, inner?: TSchema): TDbInteger {
    const defaultInner = inner ?? t.Integer({ minimum: ..., maximum: ... })
    return this.Create(
      { [Kind]: 'DbType:Integer' },
      defaultInner,
      { dataType: 'integer', columnType: column.columnType, ... }
    )
  }
}
```

### 9.5 What This Enables

- **Validation**: `Value.Check(dbSchema, value)` works via TypeRegistry delegation
- **Schema introspection**: `dbSchema.inner` for validation-only, `dbSchema.db` for database metadata
- **Type extraction**: `Static<typeof dbSchema>` correctly resolves through `inner`
- **Migration generation**: Walk `dbSchema.db` to produce DDL
- **Drizzle integration**: Replace `columnToSchema()` with `DbType.fromColumn()` that producesDbType schemas

---

## 10. Key Differences from TypeDef's Approach

| Aspect | TypeDef | Proposed DbType |
|--------|---------|-----------------|
| Relation to TypeBox | Replacement | Composition (wrapping) |
| Validation | Custom ValueCheck entirely | Delegate to inner via TypeRegistry |
| TypeGuard | Fully custom | Compose with TypeBox's TypeGuard |
| Metadata | Flat `metadata: {}` bag | Structured `db: DbColumnMeta` |
| Kind namespace | `TypeDef:*` | `DbType:*` |
| Inner schema reference | None (TypeDef IS the schema) | `inner: TSchema` field |
| Purpose | Binary protocol types | Database column types with validation |

The composition approach is essential because DbType must preserve TypeBox's rich validation capabilities (`format`, `pattern`, `minimum`/`maximum`, etc.) while layering database semantics on top. TypeDef's flat replacement approach works because binary protocol types have simpler validation needs (int8 range checks, ISO timestamp formatting, etc.).