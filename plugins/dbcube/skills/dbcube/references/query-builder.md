# Dbcube query builder (verified API)

> Source of truth for the installed version:
> `node_modules/@dbcube/query-builder/dist/index.d.ts` (re-exported by
> `node_modules/dbcube`). If a signature/option isn't listed below, read that
> file (e.g. `grep -n "methodName" node_modules/@dbcube/query-builder/dist/index.d.ts`).

`const db = dbcube.database('<name>')`. Type with `db.table<T>('users')`.

## Reads
```ts
db.table('users').get();                      // → array of rows
db.table('users').select(['id','name']).get();
db.table('users').first();                    // → row | null
db.table('users').find(3);                    // by primary key
db.table('users').find('a@b.com', 'email');   // by any column
db.table('users').distinct().select(['status']).get();
```

### WHERE (operator is ALWAYS required)
```ts
.where('age', '>', 30)
.where('email', 'IS NULL')            // only 2-arg form allowed
.orWhere('age', '>', 80)
.whereGroup(q => { q.where('age','<',40).orWhere('age','>',80); }) // (…)
.whereIn('id', [1,2,3])
.whereNotIn('status', ['inactive'])
.whereBetween('age', [30, 60])
.whereNull('deleted_at')
.whereNotNull('email')
```
Operators: `=` `!=` `<>` `>` `<` `>=` `<=` `LIKE` `NOT LIKE` `IN` `NOT IN`
`BETWEEN` `NOT BETWEEN`, plus `IS NULL` / `IS NOT NULL` (2-arg).

### Order / limit / joins
```ts
.orderBy('age', 'DESC')          // 'ASC' | 'DESC'
.limit(20).offset(40)            // manual pagination
db.table('orders').join('users', 'orders.user_id', '=', 'users.id')
  .select(['orders.product', 'users.name']).get();   // SQL engines
```

## Aggregations (run immediately → return a value; NOT chainable)
```ts
await db.table('users').count();                       // number
await db.table('users').where('status','=','active').count();
await db.table('users').sum('balance');                // number
await db.table('users').avg('age');                    // number
await db.table('users').max('age'); / .min('age');     // number
await db.table('users').where('age','>',80).exists();  // boolean (SELECT 1 … LIMIT 1)
```
Grouped:
```ts
await db.table('users')
  .selectRaw(['status', 'COUNT(*) AS n'])
  .groupBy('status')
  .having('n', '>', 0)
  .get();
```

## Pagination & large datasets
```ts
const page = await db.table('users').orderBy('id','ASC').paginate(1, 10);
// → { items, total, totalPages, hasNext, hasPrev, ... }
await db.table('users').chunk(1000, (rows, pageNum) => {
  // process in flat memory; return false to stop early
});
```

## Writes
```ts
// insert ALWAYS takes an array; returns the inserted rows (with generated ids/uuids)
const created = await db.table('users').insert([{ name:'Ada', email:'a@b.com', age:36 }]);

// update & delete REQUIRE a where() (throw otherwise)
await db.table('users').where('id','=',1).update({ status:'verified' });
await db.table('users').where('id','=',1).delete();

// upsert: insert or update on conflict (key column must be UNIQUE)
await db.table('settings').upsert([{ key:'theme', value:'dark' }], ['key']);

// atomic counters (no read-modify-write race)
await db.table('users').where('id','=',1).increment('balance', 250);
await db.table('users').where('id','=',1).increment('balance', 1, { status:'vip' }); // + extra cols
await db.table('users').where('id','=',2).decrement('balance', 100);

// truncate: the ONLY write allowed without a where()
await db.table('settings').truncate();
```

## Transactions
```ts
// interactive — use when you must READ intermediate results and branch
await db.transaction(async (trx) => {
  const u = await trx.table('users').find(1);     // sees uncommitted changes
  if (u.balance < 0) throw new Error('…');         // throw → automatic ROLLBACK
  await trx.table('users').where('id','=',1).update({ … });
});

// batch — fastest: begin + writes + commit in ONE round-trip (no reads inside)
await db.batch((b) => {
  b.table('users').where('id','=',1).decrement('balance', 200);
  b.table('users').where('id','=',2).increment('balance', 200);
});
```
Works on MySQL, PostgreSQL, SQLite and MongoDB (Mongo needs a replica set).

## Relations (eager loading, no N+1)
```ts
// auto from the .cube `foreign` definition
await db.table('users').with('orders').get();
// explicit options
await db.table('orders').with('buyer', { table:'users', foreignKey:'user_id', type:'one' }).get();
await db.table('users').with('orders', { table:'orders', foreignKey:'user_id', type:'many' }).get();
```

## Raw
```ts
// SQL — ALWAYS bind params with ?, never interpolate values
const rows = await db.raw('SELECT name, age FROM users WHERE age > ? AND status = ?', [30, 'active']);
await db.raw('CREATE INDEX IF NOT EXISTS idx_users_status ON users(status)');
const typed = await db.raw<{ status: string; n: number }>('SELECT status, COUNT(*) AS n FROM users GROUP BY status');

// MongoDB — raw is a JSON command document
const res = await mongo.raw(JSON.stringify({ find: 'users', filter: { age: { $gt: 30 } }, sort: { age: -1 } }));
```

## TypeScript
`npx dbcube generate` writes `dbcube/types.ts`. Then `db.table<User>('users')`
flows the row type through the whole chain; `insert` takes `Partial<T>`;
`paginate` → `PaginatedResult<T>`; `db.raw<R>()` types ad-hoc projections.

## MongoDB notes
Same builder API. No autoincrement → provide ids yourself. `LIKE` becomes an
anchored case-insensitive regex. Transactions require a replica set.
