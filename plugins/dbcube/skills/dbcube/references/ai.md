# Data + AI — vector search, auto-embeddings and RAG (6.0.0+)

Signatures below are copied from the published `@dbcube/query-builder@6.0.0` types.
If a method is not listed here, it does not exist — use `raw()` instead of inventing one.

## Install (optional packages)
```bash
npm install @dbcube/ai @dbcube/vector @dbcube/rag
```
They are **optional peer dependencies**. The query builder loads them lazily, only when
an AI method is called; without them the classic builder works unchanged. Calling an AI
method without them installed throws an error telling you what to install.

## Configure (`dbcube.config.js`)
```js
module.exports = function (config) {
  config.set({
    databases: {
      app: {
        type: "postgres",
        config: { URL: process.env.DATABASE_URL },
        ai: {
          embedding: {                       // must be able to embed (openai | ollama | mock)
            provider: "openai",
            apiKey: process.env.OPENAI_API_KEY,
            embeddingModel: "text-embedding-3-small",
          },
          agent: {                           // generation for answer()/chat()
            provider: "anthropic",
            apiKey: process.env.ANTHROPIC_API_KEY,
            model: "claude-sonnet-5",
          },
          dimension: 1536,                   // REQUIRED — must match the embedding model
          metric: "cosine",                  // optional: cosine | euclidean | dotProduct
          chunk: { size: 800, overlap: 100 },// optional: RAG chunking
          // vector: { driver: "qdrant", url: "http://localhost:6333" }  // optional
        },
      },
    },
  });
};
```

**Trap — `embeddingModel` vs `model`.** In a provider config, `model` is the CHAT model and
`embeddingModel` is the EMBEDDING model. `model: "text-embedding-3-large"` inside
`embedding` is ignored: you silently get the default `text-embedding-3-small` (1536 dims),
and a 3072-dimension column rejects every insert.

- Providers: `openai` (embed + chat), `ollama` (embed + chat, local), `anthropic` (chat
  only — no embeddings API; pair it as `agent` with an embedding provider), `mock`
  (deterministic, no keys — for tests/CI).
- Vector backend: `ai.vector.driver` if set (`pgvector` | `qdrant` | `memory`); otherwise a
  Postgres database uses **pgvector** automatically.
- Global fallback for providers: `import { AI } from "@dbcube/ai"; AI.configure({...})`.

## Mode 1 — vector COLUMN on a normal table (PostgreSQL + pgvector)
Declare it in the `.cube` (see `cube-files.md`):
```ts
embedding: {
  type: "vector";
  dimension: 1536;                 // required
  metric: "cosine";                // cosine (default) | euclidean | dotProduct
  index: "hnsw";                   // hnsw (default) | ivfflat | none
  from: ["name", "description"];   // auto-embed source columns
  sync: "auto";                    // auto (default when `from` is set) | manual
};
```
Then `npx dbcube run table:fresh` (or `table:refresh`). With `from`, **`insert()` fills the
embedding automatically** and `update()` re-embeds only when a `from` column changed.
Never write the vector yourself; `npx dbcube generate` excludes it from `New<Table>`.

Search returns **full relational rows**:
```ts
const rows = await db.table("products")
  .where("price", "<", 100)          // real SQL filter → true hybrid search
  .search("wireless gaming mouse")
  .topK(10)
  .withScore()                       // adds a score; omitted otherwise
  .get();
```

## Mode 2 — RAG collection (documents, any table name)
```ts
await db.table("kb").add("Dbcube supports pgvector.", { source: "notes", metadata: { lang: "en" } });
await db.table("kb").import("./guide.md");            // txt/md/html/csv; pdf, docx (optional deps)
await db.table("kb").importDirectory("./docs");       // recursive by default
const { answer, sources } = await db.table("kb").answer("Which vector stores are supported?");
const chat = db.table("kb").chat();
const r1 = await chat.ask("What is Dbcube?");
chat.reset();
```
PDF needs `npm install pdf-parse`; DOCX needs `npm install mammoth` (clear error otherwise).

## Chainable methods (return `this`, end with `.get()`)
```ts
search(text: string): this
vector(embedding: number[]): this
similarTo(id: string | number): this        // excludes the row itself
topK(k: number): this
minScore(score: number): this
metric(m: "cosine" | "euclidean" | "dotProduct"): this
cosine(): this
euclidean(): this
dotProduct(): this
withScore(): this
```

## Terminal methods
```ts
add(text: string, options?: { source?: string; metadata?: Record<string, unknown> }): Promise<number>
import(filePath: string, options?: { metadata?: Record<string, unknown> }): Promise<{ source: string; chunks: number }>
importDirectory(dirPath: string, options?: { recursive?: boolean; metadata?: Record<string, unknown> }):
  Promise<{ files: number; chunks: number; skipped: string[]; imported: Array<{ source: string; chunks: number }> }>
answer(question: string): Promise<{ answer: string; sources: Array<Record<string, unknown>> }>
chat(): { ask(question: string): Promise<{ answer: string; sources: Array<Record<string, unknown>> }>; reset(): void }
reEmbed(): Promise<number>                  // backfill vectors; scope it with .where()
clearEmbeddings(): Promise<void>
```

## Rules
- `where()` still takes an operator: `where(col, op, value)`. Supported in hybrid
  filters: `=`, `!=`, `<>`, `>`, `>=`, `<`, `<=`, `IN`, combined with AND.
- Scores are normalized so **higher = more similar** for every metric and backend, so
  `minScore` means the same everywhere.
- `dimension` is locked per column/collection; changing embedding models means re-creating
  it and calling `reEmbed()`.
- Vector columns are PostgreSQL-only; on other engines `table:fresh` fails with a clear error.
