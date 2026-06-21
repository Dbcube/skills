# `.cube` files

> The real column/row types for THIS project are in `dbcube/types.ts`
> (`npx dbcube generate`). `.cube` parsing/handling lives in
> `node_modules/@dbcube/schema-builder`. Check existing `dbcube/*.cube` files in
> the project for the conventions actually used.

Declarative files in the `dbcube/` folder. Syntax uses **decorators**
(`@database`, `@meta`, `@columns`, …) and **semicolons** inside objects. When
showing `.cube` in Markdown, fence it as `ts` (no `cube` lexer in Shiki).

## Table — `*.table.cube`
```ts
@database("app");

@meta({
  name: "users";
  description: "User accounts";
});

@columns({
  id: {
    type: "int";
    options: ["primary", "autoincrement"];
  };
  name: {
    type: "varchar";
    length: "255";
    options: ["not null"];
  };
  email: {
    type: "varchar";
    length: "255";
    options: ["not null", "unique"];
  };
  status: {
    type: "varchar";
    length: "20";
    defaultValue: "active";
    options: ["not null"];
  };
  created_at: {
    type: "timestamp";
    options: ["null"];
  };
});
```
- Common `type`s: `int`, `varchar` (needs `length`), `text`, `boolean`, `date`,
  `datetime`, `timestamp`, `decimal`, `float`, `double`.
- `options`: `"primary"`, `"autoincrement"`, `"not null"`, `"null"`, `"unique"`,
  `"index"`.
- `defaultValue` for a column default.

### Foreign keys / relations
Declare a `foreign` on the child column so `with('<relation>')` resolves
automatically; otherwise pass explicit `{ table, foreignKey, type }` to `with()`.

## Seeder — `*.seeder.cube`
```ts
@database("app");
@table("users");

@fields("name", "email", "age", "status");

@dataset(
  ("Ada Lovelace",   "ada@example.com",   36, "active"),
  ("Linus Torvalds", "linus@example.com", 54, "active")
);
```
Run with `npx dbcube run seeder:add`. `@fields` lists columns; each `@dataset`
tuple is a row in that order.

## Alter — `*.alter.cube`
Precise, declarative changes to an existing table (rename, add/drop columns).
```ts
@database("app");
@table("users");

@addColumn({
  phone: {
    type: "varchar";
    length: "20";
    options: ["null"];
  };
});

@renameColumn({ last: "last_name"; new: "surname" });
@changeName({ last: "users"; new: "customers" });   // rename the table
```
Apply with `npx dbcube run table:alter` (preserves data, unlike `table:fresh`
which drops & recreates).

## Trigger — `*.trigger.cube`
JS hooks that run in the service instance performing the write (NOT DB triggers).
Fire exactly once per executed write, even across replicas.
```ts
@database("app");
@table("users");

@beforeAdd((newData, db, gdb) => {
  // mutate newData, validate, etc. (runs before insert)
});

@afterAdd((newData, db, gdb) => {
  // side effects (email, webhook, audit log)
});
// also: @beforeUpdate / @afterUpdate (oldData, newData, db, gdb)
//       @beforeDelete / @afterDelete (oldData, db, gdb)
```
- `db` = this connection, `gdb` = global accessor, `oldData`/`newData` the row.
- Make side effects idempotent (a retried request is a second write).
- Don't enforce global invariants in triggers — use transactions + UNIQUE.
Install with `npx dbcube run trigger:fresh`.

## Computed fields
Virtual columns resolved at read time (declared in the schema). Use them for
derived values you want available on read without storing them.

## Workflow
1. Write/edit `.cube` files.
2. `npx dbcube run table:fresh` (new) or `table:refresh` / `table:alter` (changes).
3. `npx dbcube run seeder:add` to seed.
4. `npx dbcube generate` to refresh `dbcube/types.ts`.
