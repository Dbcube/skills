# Dbcube — Claude Code skill

Make Claude Code **natively understand Dbcube**: `.cube` files
(`.table.cube`, `.seeder.cube`, `.trigger.cube`, `.alter.cube`),
`dbcube.config.js`, the type-safe query builder (transactions, relations,
`raw`), the CLI, and cloud config (Supabase, Turso, PlanetScale, Atlas, RDS).

The skill encodes the **real, verified API** so Claude doesn't invent methods.

## Install (recommended — as a plugin from GitHub)

In Claude Code:

```
/plugin marketplace add Dbcube/skills
/plugin install dbcube@dbcube
```

Then restart Claude Code if prompted. The `dbcube` skill activates automatically
whenever you work in a Dbcube project (a `dbcube.config.js`, a `dbcube/` folder
with `.cube` files, or `dbcube`/`@dbcube/*` in `package.json`).

> `Dbcube/skills` is `https://github.com/Dbcube/skills`. Use your own
> `owner/repo` if you fork it.

## Install (manual — personal or project skill)

Copy the skill folder into one of:

- **Personal** (all your projects): `~/.claude/skills/dbcube/`
- **Project** (committed to a repo, shared with the team): `.claude/skills/dbcube/`

```bash
# personal
mkdir -p ~/.claude/skills
cp -r plugins/dbcube/skills/dbcube ~/.claude/skills/dbcube

# or project-local
mkdir -p .claude/skills
cp -r plugins/dbcube/skills/dbcube .claude/skills/dbcube
```

The skill is the folder containing `SKILL.md` plus its `references/`.

## What's inside

```
skills/
├── .claude-plugin/marketplace.json     # marketplace listing (for /plugin)
└── plugins/dbcube/
    ├── .claude-plugin/plugin.json
    └── skills/dbcube/
        ├── SKILL.md                    # entry point: golden rules + quick API
        └── references/
            ├── config.md               # connection config, pools, cloud/TLS
            ├── query-builder.md        # every read/write/aggregation/relation
            ├── cube-files.md           # .table/.seeder/.alter/.trigger syntax
            └── cli.md                  # every CLI command + workflows
```

## Verify it loaded

Ask Claude Code something like *"how do I write an update query in Dbcube?"* — it
should answer with `db.table(...).where(col, op, val).update({...})` and remind
you that `update()` requires a `where()`.

## License

MIT.
