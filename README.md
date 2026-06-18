# ecom-owner — Claude Code plugin

Read-only e-commerce data access for the business owner, packaged as a Claude Code
plugin. Bundles:

- the **`ecom-owner` MCP server** registration (curated analytics tools +
  `get_schema` / `run_sql` for ad-hoc read-only SQL),
- the **`run-sql-ecomowner` skill** (teaches the schema-first → query workflow and
  the read-only guard rules),
- the **`/ecom-owner`** slash command (answer a data question end-to-end).

The plugin only contains the MCP *registration* — it points at a **running**
`ecom-owner` server (part of the `inx` solution: `src/services/mcp/Inx.Mcp.EcomOwner`).

## Prerequisites

A bearer token for the `ecom-owner` MCP server. The plugin points at the hosted server
`https://mcp.devreactor.com/` (set in `plugins/ecom-owner/.mcp.json`); edit that `url`
to target a different server (e.g. `http://localhost:5099`) for local development.

## Install

```text
/plugin marketplace add inventorix-net/ecom-owner
/plugin install ecom-owner@inventorix
```

## Configure (environment variables)

The plugin reads these from your environment — no secrets are stored in the repo:

| Variable | Required | Default | Meaning |
|----------|----------|---------|---------|
| `ECOM_OWNER_TOKEN` | yes | — | Bearer token sent as `Authorization: Bearer …` |

Set the token in your OS/user environment before launching Claude Code, e.g. (PowerShell):

```powershell
setx ECOM_OWNER_TOKEN "<bearer-token>"
```

The server URL is hard-coded in `plugins/ecom-owner/.mcp.json` (`https://mcp.devreactor.com/`) —
edit it there to point elsewhere.

### Minting a long-lived bearer token

The server issues short-lived tokens by default. For a static token that lives in
`ECOM_OWNER_TOKEN`, mint one against a server configured with a long lifetime:

1. On the server, set a long token lifetime (e.g. one year) and a client:
   `OAuth__AccessTokenLifetimeMinutes=525600`,
   `OAuth__SigningKey=<≥32 bytes>`,
   `OAuth__Clients__0__ClientId=plugin`, `OAuth__Clients__0__ClientSecret=<secret>`.
2. Request a token:
   ```bash
   curl -X POST "$ECOM_OWNER_URL/connect/token" \
     -d "grant_type=client_credentials&client_id=plugin&client_secret=<secret>"
   ```
3. Copy `access_token` from the response into `ECOM_OWNER_TOKEN`.

> ⚠️ A long-lived bearer is a static secret: keep it in your environment (never in
> the repo), and it can only be revoked by rotating the server's `OAuth:SigningKey`
> (which invalidates all tokens). If you need per-credential revocation instead,
> use the server's `X-Api-Key` fallback: swap the header in `plugins/ecom-owner/.mcp.json`
> to `"X-Api-Key": "${ECOM_OWNER_API_KEY}"` and set `ECOM_OWNER_API_KEY`.

## Use

- Ask a data question naturally — the **skill** auto-triggers (e.g. “pull last
  month’s top 20 SKUs by units”).
- Or run the command: `/ecom-owner top 20 SKUs by units sold in May`.

## Layout

```text
.claude-plugin/marketplace.json     # this repo as a marketplace
plugins/ecom-owner/
  .claude-plugin/plugin.json        # plugin manifest
  .mcp.json                         # ecom-owner MCP server (bearer via env)
  skills/run-sql-ecomowner/SKILL.md # read-only SQL workflow + guard rules
  commands/ecom-owner.md            # /ecom-owner command
```

## Local development

Load the plugin without the marketplace:

```bash
claude --plugin-dir ./plugins/ecom-owner
```

Then confirm `/ecom-owner` is listed, the skill triggers on a data question, and the
MCP tools resolve (requires the server running and `ECOM_OWNER_TOKEN` set).

## Releasing updates

Installed copies (Claude Code CLI **and** the Claude app) are served from this
**GitHub repo**, not from a local checkout — edits only reach users after they are
**pushed** and the client pulls a newer version. Update detection is by version:
`plugins/ecom-owner/.claude-plugin/plugin.json` `version` first, else the git commit SHA.

Release flow for every change:

1. Edit the plugin files (`.mcp.json`, `skills/…`, `commands/…`).
2. **Bump `version`** in `plugins/ecom-owner/.claude-plugin/plugin.json` (semver).
   Without a bump, clients that pin the version may not see an update.
3. `git commit` and **`git push origin main`**.
4. Pull the update on the client:
   - **Claude app** → Customize → *Ecom owner* → the **Update** button activates → click it.
   - **Claude Code CLI** → `/plugin marketplace update inventorix` (then `/mcp` reconnect
     if `.mcp.json` changed).

> The **Update** button stays greyed out when the installed version already matches
> `origin/main` — i.e. there is nothing newer to pull. Push a version-bumped commit first.
