# Apify for Kimi Code

Official Apify plugin for Kimi Code CLI. It adds the Apify MCP server, one `using-apify` router skill, and five bundled skills for the main Apify workflows: using existing Actors, building and deploying your own Actors, actorizing existing projects, generating Actor output schemas, and integrating Apify into existing applications.

> **Apify** is the largest marketplace of tools for AI: ready-made **Actors** you can run, or build your own. Find your Actor at [Apify Store](https://apify.com/store).

## What you get

| Component | Name | Purpose |
|---|---|---|
| Router skill (entry point) | `using-apify` | Loaded automatically at session start. Routes every Apify request to the right skill or transport path. **You don't need to invoke this — it's already in context.** |
| MCP server | `apify` (`https://mcp.apify.com/`) | Lets the agent search the Apify Store, fetch Actor details, run Actors, and read the Apify docs. |
| Skill | `apify-actor-development` | Create, debug, and deploy a brand new Apify Actor from scratch. |
| Skill | `apify-actorization` | Convert an existing JS/TS, Python, or CLI project into an Apify Actor. |
| Skill | `apify-generate-output-schema` | Generate `dataset_schema.json` / `output_schema.json` / `key_value_store_schema.json` for an existing Actor. |
| Skill | `apify-sdk-integration` | Add Apify Actor execution to an existing application using the `apify-client` package. |
| Skill | `apify-ultimate-scraper` | CLI-driven data extraction workflow for selecting, configuring, and running pre-built Actors across 15+ platforms. |

> **Coming from the Claude Code plugin?** Kimi Code CLI has no custom subagents — only the built-in `coder`, `explore`, and `plan`. The `apify` agent is replaced here by the `using-apify` skill, which the plugin manifest loads via `sessionStart` so routing happens without you invoking anything. The five workflow skills are byte-identical to their Claude Code counterparts.

## Installation

Install directly from the repository:

```bash
/plugins install https://github.com/apify/apify-kimi-code-plugin
```

Then start a fresh session so the skills and MCP server load:

```bash
/new
```

Validate the installation and check for manifest diagnostics:

```bash
/plugins info apify
```

Connect to the Apify MCP server:

Uses **OAuth**. Unlike Claude Code, Kimi Code does not trigger the browser handshake automatically on the first authenticated tool call — you run it explicitly, once:

```bash
/mcp-config login apify
```

That opens `console.apify.com` in your browser.

### Prerequisites

- **Kimi Code CLI** installed and authenticated (`npm install -g @moonshot-ai/kimi-code`, Node.js 22.19.0+, then `/login`).
- Verify with `kimi --version` — this plugin targets Kimi Code (`0.x`). It does **not** work with the legacy Python `kimi-cli` (`1.4x`), which uses a different plugin format.

## First-run setup

The plugin uses **three setup paths** depending on what you're doing. The `using-apify` router will guide you through the right one, but here is the high-level map.

### Path 1 — Using existing Actors through MCP


Read-only tools such as `search-actors`, `fetch-actor-details`, `search-apify-docs`, and `fetch-apify-docs` work without auth, so you can browse the Store before logging in.

### Path 2 — CLI workflows for Actor development, actorization, or scraper runs

These skills expect the local `apify` CLI to be available. Install it first:

```bash
npm install -g apify-cli
```

For interactive use, authenticate with:

```bash
apify login
```

In headless or CI environments, export an **`APIFY_TOKEN`** instead; the CLI can read it automatically:

```bash
export APIFY_TOKEN="apify_api_xxxxxxxxxxxx"
```

Generate a token at [console.apify.com/settings/integrations](https://console.apify.com/settings/integrations). Don't have an account? [Sign up free](https://console.apify.com/sign-up) — no credit card required.

### Path 3 — SDK integration into an existing application

Uses an **`APIFY_TOKEN`** environment variable with the `apify-client` package or the REST API:

```bash
export APIFY_TOKEN="apify_api_xxxxxxxxxxxx"
```

### Working in headless / SSH environments (no browser)

`/mcp-config login` needs a browser, and `kimi -p` (non-interactive prompt mode) has no way to complete the handshake. Options:

1. **Authenticate first on a machine with a browser.** Run `/mcp-config login apify` there, then point the headless machine at the same `KIMI_CODE_HOME` so it picks up `credentials/mcp/`.
2. **Skip OAuth and use a static token.** Override the MCP entry in `~/.kimi-code/mcp.json` or `.kimi-code/mcp.json` with `bearerTokenEnvVar` or a static `headers` value. A project-level entry with the same name overrides the plugin's declaration.
3. **Use the CLI-based skills.** `apify-actor-development`, `apify-actorization`, and `apify-ultimate-scraper` work headless when the `apify` CLI is installed and `APIFY_TOKEN` is exported.
4. **Use the SDK integration skill.** `apify-sdk-integration` uses `apify-client` and only needs `APIFY_TOKEN`.

## How to use it

Start a Kimi Code session and describe what you need. The `using-apify` skill is already loaded, reads the request, chooses between MCP, CLI, or SDK-based workflows, and dispatches to the right skill or tool.

```
find me 5 well-rated coffee shops in Seattle and export to CSV
build me an Actor that scrapes a sitemap and stores titles
add Apify to this Next.js app so I can run a scraper from /api/scrape
generate output schemas for the Actor in this folder
```

You can also jump straight to a workflow — every bundled skill is registered as a slash command:

```
/apify-actor-development
/apify-ultimate-scraper scrape reviews from a Google Maps listing
```

Text after the command is appended to the skill prompt. The canonical form is `/skill:<name>`; the shorthand above works as long as the name isn't taken by a built-in command.

## Components reference

### MCP server

The `apify` MCP server is declared in the plugin manifest (`kimi.plugin.json`) rather than in a separate config file:

```json
{
  "mcpServers": {
    "apify": { "url": "https://mcp.apify.com/" }
  }
}
```

It is **enabled by default** on install. Toggle it with `/plugins mcp disable apify apify` / `/plugins mcp enable apify apify`, followed by `/reload`.

It exposes:

- `search-actors` — search the Apify Store by keyword (no auth)
- `fetch-actor-details` — Actor specs, input schema, pricing (no auth)
- `run-actor` — execute an Actor and return results (OAuth)
- `get-dataset-items` — retrieve dataset rows from a previous run (OAuth)
- `search-apify-docs` / `fetch-apify-docs` — Apify documentation lookup

Kimi Code namespaces MCP tools as `mcp__<server>__<tool>`, so these appear as `mcp__apify__search-actors`, `mcp__apify__run-actor`, and so on. To stop approving read-only lookups every session, add a rule to `~/.kimi-code/config.toml`:

```toml
[[permission.rules]]
decision = "allow"
pattern = "mcp__apify__search-actors"

[[permission.rules]]
decision = "allow"
pattern = "mcp__apify__fetch-actor-details"
```

Avoid a blanket `mcp__apify__*` allow — `run-actor` consumes platform credits.

### Bundled scripts

This plugin does **not** include standalone helper scripts. Instead, the bundled skills ship with markdown references that the agent uses while working:

- `skills/apify-actor-development/references/` — Actor config, schemas, logging, standby mode, and README guidance
- `skills/apify-actorization/references/` — JS/TS, Python, and CLI actorization guides plus schema/output notes
- `skills/apify-ultimate-scraper/references/` — Actor index, gotchas, and workflow playbooks for common scraping use cases

The executable requirements come from the skills themselves: Route 1 uses the MCP server, Route 2 relies on the local `apify` CLI, and Route 3 uses the `apify-client` SDK.

## Troubleshooting

**Nothing happens after installing.** Kimi Code does not apply plugin changes to the running session. Run `/reload`, or start a fresh session with `/new`.

**MCP server shows as disconnected.** Check `/mcp` for connection status, then `/plugins info apify` for manifest diagnostics. The default startup timeout is 30 s.

**OAuth browser never opens / hangs.** Run `/mcp-config login apify` explicitly rather than waiting for an automatic prompt. If authorization is corrupted, delete `~/.kimi-code/credentials/mcp/` and log in again — note that `/logout` clears provider credentials but **not** MCP credentials. For browserless setups, see "Working in headless / SSH environments" above.

**Your local edits to the plugin aren't taking effect.** Installs are copied to `~/.kimi-code/plugins/managed/apify/` and Kimi always runs from that copy. Reinstall after changing the source.

**You installed from a branch but got something older.** A bare GitHub URL installs the latest **release** if one exists, not the default branch. Pin a ref explicitly: `/plugins install https://github.com/apify/apify-kimi-code-plugin/tree/main`.

**`apify` CLI not found.** Install it with `npm install -g apify-cli` before using `apify-actor-development`, `apify-actorization`, or `apify-ultimate-scraper`.

**`APIFY_TOKEN` not found.** Export `APIFY_TOKEN` in your shell before starting Kimi Code when using headless CLI auth or the `apify-sdk-integration` skill.

**The wrong skill keeps getting picked.** Kimi Code has no equivalent of Claude Code's `user-invocable: false`, so all five workflow skills are visible as slash commands and are also auto-invocable by the model. If routing looks wrong, either describe the goal more explicitly — use existing Actors, build an Actor, actorize a project, generate output schemas, or integrate Apify into an app — or invoke the skill directly with `/skill:<name>`.

**`apify` vs `apify-client`** — these are two different npm packages. The `apify` package is the SDK for **building** Actors (used inside an Actor's code, on the Apify platform). The `apify-client` package is the API client for **calling** Actors from your own application. The agent picks the right one for you; if you're installing manually, double-check.

## Resources

- Apify Console — [console.apify.com](https://console.apify.com)
- Apify Store — [apify.com/store](https://apify.com/store)
- Docs (LLM-friendly) — [docs.apify.com/llms.txt](https://docs.apify.com/llms.txt)
- Docs (full) — [docs.apify.com/llms-full.txt](https://docs.apify.com/llms-full.txt)
- Kimi Code plugin docs — [kimi.com/code/docs/en/kimi-code-cli/customization/plugins.html](https://www.kimi.com/code/docs/en/kimi-code-cli/customization/plugins.html)
- Kimi Code MCP docs — [kimi.com/code/docs/en/kimi-code-cli/customization/mcp.html](https://www.kimi.com/code/docs/en/kimi-code-cli/customization/mcp.html)
- Source repo — [github.com/apify/apify-kimi-code-plugin](https://github.com/apify/apify-kimi-code-plugin)
- Claude Code version — [github.com/apify/apify-claude-code-plugin](https://github.com/apify/apify-claude-code-plugin)
- Issues / feedback — open an issue on the source repo, or email [support@apify.com](mailto:support@apify.com)

## License

Apache-2.0. See [LICENSE](./LICENSE).
