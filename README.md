# @alkdev/drizzlebox

Generate [TypeBox](https://github.com/alkdev/typebox) schemas from [Drizzle ORM](https://orm.drizzle.team) schemas.

This is a fork of [drizzle-typebox](https://github.com/drizzle-team/drizzle-orm/tree/main/drizzle-typebox) by the Drizzle Team, adapted for use with `@alkdev/typebox` (a maintained fork of `@sinclair/typebox`).

## Install

```bash
npm install @alkdev/drizzlebox
npm install @alkdev/typebox
npm install drizzle-orm
```

## Features

- Create select schemas for tables, views, and enums
- Create insert and update schemas for tables
- Supports all dialects: PostgreSQL, MySQL, and SQLite
- Custom TypeBox instance support via `createSchemaFactory`

## Usage

```ts
import { pgEnum, pgTable, serial, text, timestamp } from 'drizzle-orm/pg-core';
import { createInsertSchema, createSelectSchema, createUpdateSchema } from '@alkdev/drizzlebox';
import { Type } from '@alkdev/typebox';
import { Value } from '@alkdev/typebox/value';

const users = pgTable('users', {
	id: serial('id').primaryKey(),
	name: text('name').notNull(),
	email: text('email').notNull(),
	role: text('role', { enum: ['admin', 'user'] }).notNull(),
	createdAt: timestamp('created_at').notNull().defaultNow(),
});

// Schema for inserting a user
const insertUserSchema = createInsertSchema(users);

// Schema for updating a user
const updateUserSchema = createUpdateSchema(users);

// Schema for selecting a user
const selectUserSchema = createSelectSchema(users);

// Overriding fields
const insertUserSchema = createInsertSchema(users, {
	role: Type.String(),
});

// Refining fields
const insertUserSchema = createInsertSchema(users, {
	id: (schema) => Type.Number({ ...schema, minimum: 0 }),
	role: Type.String(),
});

// Validation
const isUserValid: boolean = Value.Check(insertUserSchema, {
	name: 'John Doe',
	email: 'johndoe@test.com',
	role: 'admin',
});
```

## Differences from drizzle-typebox

- Uses `@alkdev/typebox` instead of `@sinclair/typebox`
- Standalone package (no monorepo dependency)
- Published as `@alkdev/drizzlebox` on npm

## Attribution

Based on [drizzle-typebox](https://github.com/drizzle-team/drizzle-orm/tree/main/drizzle-typebox) by the Drizzle Team, licensed under Apache-2.0.