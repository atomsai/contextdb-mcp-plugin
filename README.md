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
`io.github.atomsai/contextdb-memory@0.1.0`. Vendor-specific marketplace reviews
remain separate.

## Links

- [ContextDB MCP documentation](https://contextdb.ai/mcp)
- [Cloud quickstart](https://contextdb.ai/docs#quickstart)
- [Privacy and terms](https://contextdb.ai/legal)
- [Security](https://contextdb.ai/security)
- [Service status](https://contextdb.ai/status)
- [Official MCP Registry entry](https://registry.modelcontextprotocol.io/v0/servers?search=io.github.atomsai%2Fcontextdb-memory)
- [Open-source ContextDB SDK](https://github.com/atomsai/contextdb)

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

### How do I revoke access?

Open ContextDB Console settings and select **Revoke access** for the connected
AI client.

