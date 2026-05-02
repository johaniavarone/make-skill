# make-expert

A senior-level Claude skill for designing, building, debugging, and refactoring [Make.com](https://www.make.com) (formerly Integromat) automations. Covers the Make REST API, blueprint JSON internals, IML expression language, built-in modules, error handling, webhooks, data stores, the official Make MCP Server, custom apps development, and migration paths to and from n8n.

Built for Claude Code, Claude Desktop, and the Claude.ai web interface — anywhere skills are supported.

## What it does

When loaded in a Claude session, this skill triggers automatically whenever the user mentions Make, Integromat, scenarios, blueprints, IML, or describes an automation task that would naturally be built in Make. Claude then has access to a modular reference covering:

- The full Make API v2 (CRUD on scenarios, blueprints, hooks, data stores, executions, custom variables, organizations)
- Blueprint JSON structure and the gotchas around `__IMTCONN__`, zone prefixes, and the string-encoded blueprint payload
- IML syntax, references, conditionals, and the complete catalog of built-in functions (general, math, text, date/time, array)
- Built-in modules (Webhooks, Routers, Iterators, Aggregators, HTTP, JSON, Text Parser, Tools)
- The five error directives (Break, Resume, Rollback, Commit, Ignore), Incomplete Executions Queue, exponential backoff, circuit breaker, and DLQ patterns
- Webhooks (HMAC validation, queues, mailhooks) and Data Stores (dedup ledgers, caches, rate limit trackers)
- The official Make MCP Server (OAuth vs MCP token, Streamable HTTP / SSE, scenarios as tools, async retrieval)
- Naming conventions, idempotence patterns, ops optimization, security, monitoring, and refactoring strategies
- Migration mapping between Make and n8n (modules, expressions, conceptual differences)
- Custom Apps (architecture, IML, RPCs, custom IML functions, App Review)

## Repository structure

```
make-expert/
├── README.md
├── SKILL.md                    # Entry point read by Claude
└── references/
    ├── 01-api-make.md          # Make REST API v2
    ├── 02-blueprint-json.md    # Blueprint internals
    ├── 03-iml-expressions.md   # IML language
    ├── 04-functions-reference.md   # Built-in functions catalog
    ├── 05-builtin-modules.md   # Webhooks, Router, Iterator, Aggregator, HTTP, JSON
    ├── 06-error-handling.md    # Directives, retry, DLQ, circuit breaker
    ├── 07-webhooks-datastores.md   # Webhook patterns, data store recipes
    ├── 08-mcp-server.md        # Make MCP Server
    ├── 09-best-practices.md    # Naming, idempotence, ops, refactoring
    ├── 10-migration-n8n.md     # Make ↔ n8n migration
    └── 11-custom-apps.md       # Custom Apps development
```

The structure is modular by design: `SKILL.md` is short and stays in Claude's context permanently, while the reference files are loaded on demand based on the task at hand. This keeps the token footprint minimal until specific knowledge is needed.

## Installation

### Claude Code (CLI)

Clone the repo into your skills directory:

```bash
git clone https://github.com/<your-username>/make-expert.git ~/.claude/skills/make-expert
```

Claude Code auto-discovers skills under `~/.claude/skills/`. Restart your session and the skill will activate on relevant prompts.

### Claude Desktop / Claude.ai (custom skills)

If your account supports custom skills via the UI, upload the repo as a ZIP through the skills management interface. The exact path depends on your plan — check Settings → Capabilities or the equivalent section.

### Inside a Claude Project

Drop the `SKILL.md` and the `references/` folder into the project's knowledge files. Claude will use them as reference material across all conversations in that project.

## Usage

Once installed, the skill triggers naturally:

- *"Help me build a Make scenario that watches a Notion database and creates Pennylane invoices"* → Claude pulls `02-blueprint-json.md`, `05-builtin-modules.md`, and writes a blueprint
- *"My iterator is producing empty bundles, what's wrong?"* → Claude pulls `06-error-handling.md` and walks through the diagnostic
- *"Migrate this Make scenario to n8n"* → Claude pulls `10-migration-n8n.md` and produces the equivalent workflow
- *"Build a Custom App for our internal CRM"* → Claude pulls `11-custom-apps.md` and scaffolds the IML

No need to explicitly invoke the skill or quote its name in prompts.

## Sources

The content is derived from the official Make documentation (`developers.make.com`, `apps.make.com`, `help.make.com`), the Make MCP Server reference, hands-on production experience with Make scenarios at scale, and the community knowledge captured in the [D1DX/make-skill](https://github.com/D1DX/make-skill) repository which inspired the modular approach.

## License

MIT — feel free to fork, adapt, and redistribute.

## Contributing

Issues and pull requests welcome. If you spot an outdated API endpoint, a missing module, or a better pattern for a recurring problem, open an issue or submit a PR. The skill is designed to evolve with Make itself.

## Author

Maintained by [Johan Iavarone](https://johaniavarone.com) — Product Designer and no-code developer based in Paris.
