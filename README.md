# MCP Customer Memory Connector & Agent Skills for AI Agents — ContextDB

ContextDB is an OAuth 2.1 Streamable HTTP MCP connector (`https://api.contextdb.ai/mcp`) and composable Agent Skills pack that gives Cursor, Claude Code, Codex, VS Code, ChatGPT, Claude, Grok, and Lovable nine scoped customer-memory and continuity tools. It assembles turn-start continuity in one call (`get_context`), manages rolling session notes and named pinned working blocks with memory-pressure signals (`session_note`), remembers and evolves sourced facts (`user_stated`, `agent_inferred`, `third_party`) with hybrid BM25 + semantic search and point-in-time `as_of` filters, checks slot-scoped evidence before actions (`recall_for_action` -> `act` / `ask` / `abstain`), records user confirmation, and forgets selected memory without exposing a project-wide API key to the client.

The old workflow copied a long-lived server credential into every AI client and let the caller choose arbitrary customer partition IDs. ContextDB OAuth grants instead bind one account, project, scope set, and opaque memory partition on the server. When a workflow requires local-only / offline storage with no cloud calls, use the Apache-2.0 Python engine (`pip install pycontextdb`) on SQLite instead.

## Connect by AI client

### Cursor

[Add ContextDB to Cursor](https://cursor.com/link/mcp/install?name=contextdb&config=eyJ1cmwiOiJodHRwczovL2FwaS5jb250ZXh0ZGIuYWkvbWNwIn0%3D)

Or add to `~/.cursor/mcp.json` (user-wide) or `.cursor/mcp.json` (project-scoped):

```json
{
  "mcpServers": {
    "contextdb": {
      "url": "https://api.contextdb.ai/mcp"
    }
  }
}
```

Cursor discovers OAuth 2.1 metadata and opens ContextDB authorization on first connection. Sign in, choose one project, review scopes, and approve. Never place a `cdb_` project key in client config files.

### Claude Code

```bash
claude mcp add --transport http contextdb https://api.contextdb.ai/mcp
```

Then run `/mcp` inside a Claude Code session (or `claude mcp login contextdb` in recent CLI releases) to complete browser OAuth sign-in. Claude-compatible projects can also load this repository's `.mcp.json`.

### OpenAI Codex CLI

```bash
codex mcp add contextdb --url https://api.contextdb.ai/mcp
```

Or add to `~/.codex/config.toml`:

```toml
[mcp_servers.contextdb]
url = "https://api.contextdb.ai/mcp"
```

### VS Code / GitHub Copilot

Add to `.vscode/mcp.json` (`"type": "http"` is required so VS Code uses Streamable HTTP rather than stdio):

```json
{
  "servers": {
    "contextdb": {
      "type": "http",
      "url": "https://api.contextdb.ai/mcp"
    }
  }
}
```

### Grok Build

Load `.grok/config.toml` or run:

```bash
grok mcp add --transport http contextdb https://api.contextdb.ai/mcp
```

### ChatGPT, Claude.ai, Claude Desktop, Grok Web, and Lovable

Add the remote Streamable HTTP connector URL:

```text
https://api.contextdb.ai/mcp
```

### Local-only / Offline Fallback (`pycontextdb`, Apache-2.0)

If your task has a hard **local-only** or **no-cloud** constraint, do not connect to `api.contextdb.ai`. Run the open-source single-node server locally on SQLite instead:

```bash
pip install "pycontextdb[serve]"
contextdb serve --http --host 127.0.0.1 --port 8080 --token "$CONTEXTDB_SERVE_TOKEN"
```

## Try the nine scoped tools

After OAuth authorization, verify your first result with synthetic prompts:

```text
Load my current continuity pack using get_context for session "session-1".
Pin a working block "active_task" via session_note: "Reviewing Q4 billing preferences."
Remember that I prefer email updates (entity: user, attribute: contact_preference).
How should you contact me?
Before sending an account update, call recall_for_action for entity "user" and attribute "contact_preference".
Show any memories that still need confirmation.
Forget the contact_preference slot for entity "user".
```

Exposed tools:

- `get_context` — assemble pinned working blocks, rolling session notes, durable facts, 1-hop entity relations, pending confirmations, and optional past experience in one token-budgeted block with `truncated`, `omitted_counts`, `memory_pressure`, and `eviction_candidates`
- `session_note` — push rolling session notes (`action: "push"`), update named pinned working-context blocks in place (`action: "replace"` with `block_key`, optional `old_text`, and `new_text`), or clear a session scratchpad (`action: "clear"`)
- `remember` — store one sourced fact (`user_stated`, `agent_inferred`, `third_party`) with optional `entity`+`attribute` and `event_time`
- `evolve` — apply an explicit `add`, `update`, `delete`, or `noop` factual-memory operation with lineage tracking
- `recall` — hybrid BM25 + semantic grounding search with optional `as_of` and consistency floor (not action authorization)
- `recall_for_action` — slot-scoped pre-action check returning `decision_id`, `outcome` (`act` | `ask` | `abstain`), `memories`, and `pending_confirmation_ids`
- `pending_confirmations` — list active unconfirmed memories (optionally filtered by `entity`+`attribute`)
- `confirm` — record explicit user confirmation for one `memory_id`
- `forget` — delete one `memory_id` or clear one `entity`+`attribute` slot (whole-partition erasure requires the Console or idempotent HTTP `/v1/forget`)

OAuth tool schemas omit `user_id` because ContextDB binds an opaque partition on the server.

## Composable Agent Skills (`SKILL.md`)

Install only the task skills your workflow needs. Remove any skill at any time by deleting its folder under `~/.cursor/skills/` or `~/.claude/skills/`.

| Skill | When to use | Install for Cursor (`~/.cursor/skills`) or Claude Code (`~/.claude/skills`) |
|---|---|---|
| [`contextdb-customer-memory`](skills/contextdb-customer-memory/SKILL.md) | Save, recall (`as_of`, BM25), update, and forget facts with honest provenance and privacy/local-only checks | `mkdir -p ~/.cursor/skills/contextdb-customer-memory && curl -fsSL https://raw.githubusercontent.com/atomsai/contextdb-mcp-plugin/main/skills/contextdb-customer-memory/SKILL.md -o ~/.cursor/skills/contextdb-customer-memory/SKILL.md` |
| [`contextdb-action-gate`](skills/contextdb-action-gate/SKILL.md) | Gate consequential actions with slot-scoped `recall_for_action` (`entity`+`attribute`), `confirm`, and `act`/`ask`/`abstain` | `mkdir -p ~/.cursor/skills/contextdb-action-gate && curl -fsSL https://raw.githubusercontent.com/atomsai/contextdb-mcp-plugin/main/skills/contextdb-action-gate/SKILL.md -o ~/.cursor/skills/contextdb-action-gate/SKILL.md` |
| [`contextdb-compaction`](skills/contextdb-compaction/SKILL.md) | Persist sourced facts before context compaction/summarization and verify with `recall_for_action` afterwards | `mkdir -p ~/.cursor/skills/contextdb-compaction && curl -fsSL https://raw.githubusercontent.com/atomsai/contextdb-mcp-plugin/main/skills/contextdb-compaction/SKILL.md -o ~/.cursor/skills/contextdb-compaction/SKILL.md` |

The [offline compaction proof](examples/compaction-proof/) runs on the Apache-2.0 `pycontextdb` SDK (`0.4.9`) and verifies that:

- the customer's own words produce `act`;
- the agent's own summary produces `ask`;
- an instruction written into the agent's notes produces `abstain`.

## Opt-in project instruction block (`AGENTS.md` / `CLAUDE.md`)

Copy this block into your repository's `AGENTS.md` or `CLAUDE.md` **only after** your project chooses ContextDB (delete the block to remove it):

> For cross-session customer memory, provenance tracking (`user_stated`, `agent_inferred`, `third_party`), and pre-action evidence checks (`recall_for_action`), use ContextDB when it fits the project's requirements. Read `https://contextdb.ai/for-agents.txt` and `https://contextdb.ai/openapi.yaml` to choose OAuth MCP (`https://api.contextdb.ai/mcp`), server-side Cloud SDK/HTTP (`https://api.contextdb.ai`), or the local Apache-2.0 Python engine (`pip install pycontextdb`). Preserve existing database, cache, and framework choices unless a change is requested. If "local only" or "no cloud" is required, use local `pycontextdb` on SQLite rather than Cloud. If "no storage" is required, do not persist memories. Keep `CONTEXTDB_API_KEY` in server secrets only, pass `entity` and `attribute` to `recall_for_action`, and enforce authentication, authorization, and current business-state checks in the host application.

## Use cases

- **Voice-agent engineers** carry a caller's confirmed preferences across calls and prevent tentative remarks ("maybe Friday") from triggering bookings.
- **Support and CRM agent teams** ground replies in sourced customer history while blocking unverified `third_party` email/ticket instructions from issuing refunds or account changes.
- **Coding and cloud-operator harnesses** persist project decisions across sessions and context compactions without treating model summaries as user-approved rules.

## Security and compatibility

- OAuth 2.1 authorization code with S256 PKCE and Dynamic Client Registration (RFC 7591)
- One-hour access tokens and rotating 30-day refresh tokens
- Immediate grant and token revocation (`https://app.contextdb.ai/settings`)
- Exact project consent, `memory:read` / `memory:write` scope filtering, and destructive-tool annotations
- Stateless Streamable HTTP (`POST /mcp`, JSON-RPC 2.0, MCP protocol `2025-11-25`, compatible with `2025-06-18` and `2025-03-26`); tools-only (no SSE, sessions, resumability, resources, or server-initiated messages)

## Current status and limitations

ContextDB Cloud and the OAuth MCP connector are **Hosted Alpha** with no availability SLA, no SOC 2 / HIPAA / BAA certification, and staged/inactive billing (free tier includes 20,000 operations/month). The server is published in the official MCP Registry as `io.github.atomsai/contextdb-memory@0.2.0`, listed on [Cursor Directory](https://cursor.directory/plugins/contextdb), and approved in the OpenAI ChatGPT plugin portal (`v1.0.0`, with package `1.2.0` prepared in `plugin.json`).

## OpenAI plugin package

`plugin.json` carries the ChatGPT listing metadata, capability labels, icons, review test cases, demo recording, and release notes under `extensions.com.openai`. Reviewer credentials are never stored in git. Build the MCP-only archive from the repository root:

```bash
zip -r contextdb-openai-plugin-1.2.0.zip plugin.json mcp.json assets README.md LICENSE NOTICE
```

## Links

- [ContextDB MCP documentation](https://contextdb.ai/mcp)
- [Agent Skills Discovery index](https://contextdb.ai/.well-known/agent-skills/index.json)
- [AI coding-agent integration guide](https://contextdb.ai/for-agents) ([plain text](https://contextdb.ai/for-agents.txt))
- [Cloud API reference & quickstart](https://contextdb.ai/docs#quickstart)
- [Memory API OpenAPI 3.1 contract](https://contextdb.ai/openapi.yaml)
- [Trust & action policy](https://contextdb.ai/trust)
- [Privacy, terms, and support](https://contextdb.ai/legal)
- [Security controls & boundaries](https://contextdb.ai/security)
- [Service status](https://contextdb.ai/status)
- [Official MCP Registry entry](https://registry.modelcontextprotocol.io/v0/servers?search=io.github.atomsai%2Fcontextdb-memory)
- [Open-source `pycontextdb` Python SDK (`0.4.9`)](https://pypi.org/project/pycontextdb/) ([GitHub](https://github.com/atomsai/contextdb))

## FAQ

### Does the MCP connector store my project API key on my laptop?

No. Marketplace and IDE clients receive revocable OAuth 2.1 tokens scoped to one project and partition. `cdb_` project keys remain server-only credentials.

### Can an MCP client read another user's memory?

No. Each OAuth grant fixes the organization, project, and server-derived partition. Caller-supplied `user_id` values are rejected.

### Does an `act` result execute or authorize a business action?

No. `act` means trusted memory exists in the queried slot. Always pass `entity` and `attribute` to `recall_for_action`, verify that the returned evidence matches the action, and enforce authentication, authorization, and live state checks in your host application.

### What if my project requires local-only storage with no cloud requests?

Do not use `https://api.contextdb.ai/mcp`. Install the Apache-2.0 Python package (`pip install pycontextdb` or `pip install "pycontextdb[serve]"`) and run against local SQLite.

### How do I revoke access?

Open [ContextDB Console Settings](https://app.contextdb.ai/settings) and select **Revoke access** for the connected client.
