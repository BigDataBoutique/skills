# BigData Boutique Skills

Agent skills for the technologies we work with every day at [BigData Boutique](https://bigdataboutique.com) — a consultancy specializing in search, analytics, and big data platforms.

These skills follow the [Agent Skills specification](https://github.com/vercel-labs/skills) and work with Claude Code, Cursor, Cline, and other compatible coding agents.

## What are skills?

Skills are reusable instruction sets that give your coding agent deep, opinionated knowledge about a specific technology. Instead of re-explaining best practices every time, you install the skill once and the agent applies it automatically when working with that technology.

## Available Skills

| Skill | Description |
|-------|-------------|
| [`clickhouse-best-practices`](./clickhouse-best-practices/) | Schema design, query optimization, insert strategy, and cluster management for ClickHouse |
| [`opensearch-best-practices`](./opensearch-best-practices/) | Indexing, querying, vector search, neural search, and cluster management for OpenSearch |

## Installation

### Claude Code

To install individual skills:

```bash
npx skills add https://github.com/bigdataboutique/skills --skill clickhouse-best-practices
npx skills add https://github.com/bigdataboutique/skills --skill opensearch-best-practices
```

### Manual

Clone this repository and copy the skill directory into your project's `.claude/skills/` folder (or the equivalent for your agent):

```bash
git clone https://github.com/bigdataboutique/skills.git
cp -r skills/clickhouse-best-practices /your-project/.claude/skills/
```

## Usage

Once installed, your agent will automatically apply the relevant skill when you work with a supported technology. You can also invoke skills explicitly:

```
/clickhouse-best-practices
/opensearch-best-practices
```

Or just describe what you're trying to do and the agent will apply the appropriate knowledge:

> "Create a ClickHouse table for storing user events"
> "Write an OpenSearch query that combines keyword and vector search"

## Contributing

Pull requests are welcome. If you work with a technology not yet covered here, consider contributing a skill — see the existing `SKILL.md` files for the structure to follow.
