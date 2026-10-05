# MCP Customer Memory Connector for AI Agents — ContextDB

ContextDB is an OAuth MCP connector that gives ChatGPT, Claude, Cursor, Grok,
Lovable, and other MCP clients six scoped customer-memory tools. It remembers
sourced facts, recalls context, checks evidence before actions, records
confirmation, and forgets selected memory without exposing a project-wide API
key to the client.

The old workflow copied a server credential into every AI client and let the
caller choose arbitrary customer partition IDs. ContextDB OAuth grants instead
bind one account, project, scope set, and opaque memory partition on the server.

## Install in Cursor

[Add ContextDB to Cursor](https://cursor.com/link/mcp/install?name=contextdb&config=eyJ1cmwiOiJodHRwczovL2FwaS5jb250ZXh0ZGIuYWkvbWNwIn0%3D)

Or add the remote server manually:

```json
{
  "mcpServers": {
    "contextdb": {
      "url": "https://api.contextdb.ai/mcp"
    }
  }
}
```

Cursor opens ContextDB authorization on first connection. Sign in, choose one
project, review the scopes, and approve. No `cdb_` project key belongs in this
file.

## Other clients

Claude-compatible clients can load the repository's `.mcp.json`.

Grok Build can load `.grok/config.toml` or add the server directly:

```bash
grok mcp add --transport http contextdb https://api.contextdb.ai/mcp
```

For ChatGPT, Claude.ai, Grok web, or Lovable, add this remote connector URL:

```text
https://api.contextdb.ai/mcp
```

Each client should discover ContextDB's OAuth metadata and open the same
project-consent flow.

## Try the tools

After authorization, ask your AI client:

```text
Remember that I prefer email updates.
How should you contact me?
Check whether confirmed memory supports booking Friday.
Show memory that still needs confirmation.
Forget the email preference you just stored.
```

The connector exposes:

- `remember`
- `recall`
- `recall_for_action`
- `pending_confirmations`
- `confirm`
- `forget`

OAuth tool schemas do not accept `user_id`. ContextDB derives a stable opaque
partition from the authorizing account and project.

## Use cases

- Voice-agent developers carry a caller's confirmed preferences across calls.
- Support engineering leads ground replies in sourced customer context.
- Safety and governance teams call `recall_for_action` before bookings,
  refunds, or account changes and preserve `act`, `ask`, or `abstain`.

ContextDB advises. The customer application remains responsible for business
authorization, current system state, and final action execution.

## Skill: compact your context safely

Agents that compact, summarize, or rewrite their own context during long tasks
can lose what the user said or start acting on their own summary. The
[`contextdb-compaction` skill](skills/contextdb-compaction/SKILL.md) tells the
agent to save customer facts with honest source labels before it compacts, to
store facts rather than instructions, and to call `recall_for_action` before
acting afterwards.

Install it for Cursor or Claude Code:

```bash
mkdir -p ~/.cursor/skills/contextdb-compaction
curl -fsSL https://raw.githubusercontent.com/atomsai/contextdb-mcp-plugin/main/skills/contextdb-compaction/SKILL.md \
  -o ~/.cursor/skills/contextdb-compaction/SKILL.md
```

For Claude Code, use `~/.claude/skills/contextdb-compaction/` instead. Any
client that reads Agent Skills `SKILL.md` files can load the same file.

The [compaction proof](examples/compaction-proof/) runs offline on the
Apache-2.0 SDK and shows what ContextDB does with the notes the skill saves:

- the customer's own words get `act`;
- the agent's own summary gets `ask`;
- an instruction written into the agent's notes gets `abstain`.

We have not yet measured how consistently agents follow the skill. Treat it as
guidance and keep the action check in your application.

## Security and compatibility

- OAuth 2.1 authorization code with S256 PKCE
- One-hour access tokens
- Rotating 30-day refresh tokens
- Immediate grant and token revocation
- Exact project consent
- Read/write scope filtering
- Destructive-tool annotations
- Server-bound memory partition

The server uses stateless Streamable HTTP with JSON responses and MCP protocol
`2025-06-18`. It is tools-only: no SSE, sessions, resumability, resources,
prompts, or server-initiated messages.

## Current status

ContextDB Cloud and the OAuth MCP connector are Hosted Alpha with no
availability SLA. The server is published in the official MCP Registry as
`io.github.atomsai/contextdb-memory@0.1.0`. The ChatGPT plugin was submitted to
OpenAI review on September 30, 2026 and is not yet approved. The Cursor
Directory listing was submitted the same day.

## OpenAI plugin package

This repository is also the OpenAI plugin package. `plugin.json` carries the
ChatGPT listing text, icons, review test cases, demo recording, and release
notes under `extensions.com.openai`. Reviewer credentials are never stored here;
they are entered only in the OpenAI dashboard. Build the upload ZIP from the
repository root:

```bash
zip -r contextdb-openai-plugin.zip plugin.json mcp.json assets README.md LICENSE NOTICE
```

## Links

- [ContextDB MCP documentation](https://contextdb.ai/mcp)
- [Cloud quickstart](https://contextdb.ai/docs#quickstart)
- [Privacy and terms](https://contextdb.ai/legal)
- [Security](https://contextdb.ai/security)
- [Service status](https://contextdb.ai/status)
- [Official MCP Registry entry](https://registry.modelcontextprotocol.io/v0/servers?search=io.github.atomsai%2Fcontextdb-memory)
- [Open-source ContextDB SDK](https://github.com/atomsai/contextdb)
- [Context language models and agent memory](https://contextdb.ai/context-language-models-and-agent-memory)

## FAQ

### Does the plugin store my project key?

No. Marketplace clients receive revocable OAuth credentials. Project keys
remain server credentials.

### Can the AI client read another user's memory?

No. The OAuth grant fixes the account, project, and memory partition.
Caller-supplied `user_id` values are rejected.

### Does an `act` result execute a business action?

No. It means trusted memory supports the proposed action. The host still
authenticates, authorizes, checks current state, executes, and records the
result.

### Can my agent's own summary authorize an action?

Not by itself. Save summaries as `agent_inferred`. `recall_for_action` returns
`ask` for them until someone confirms the fact, and instruction-shaped notes
are flagged and cannot support an action.

### How do I revoke access?

Open ContextDB Console settings and select **Revoke access** for the connected
AI client.

