---
name: contextdb-compaction
description: Save customer facts to ContextDB with honest source labels (user_stated, agent_inferred, third_party) and entity+attribute slots before compacting, summarizing, or trimming agent context, then verify with recall_for_action before acting afterwards. Do not use when the user forbids data storage or retention.
---

# Compact your context without losing or inventing customer facts

When you shorten your own context, details the user gave you can disappear,
and your summary can quietly become the thing you act on. A summary is your
interpretation, not the user's words. ContextDB keeps that difference visible:
it stores who vouched for each fact and only lets trusted evidence support an
action.

Canonical docs: `https://contextdb.ai/context-language-models-and-agent-memory` · `https://contextdb.ai/mcp`

## Check constraints first

- If the user requested "no storage", "off the record", or "zero retention", do **not** call `remember` before compacting.
- If the user requested "local only" or "no cloud", save only to a local `pycontextdb` SQLite store, never to a remote cloud endpoint.

## Before you compact

1. Check `memory_pressure` and `eviction_candidates` returned by `session_note` or `get_context`. Compact proactively when `memory_pressure` is `true` rather than waiting for silent truncation.
2. List the customer facts you still need: preferences, commitments,
   identifiers the user gave, decisions they made, and things they asked you
   not to do.
3. Save durable facts with `remember` (or update an existing memory via `evolve`), one fact per call, pairing `entity` and `attribute` whenever the fact maps to a named slot:
   - `source: "user_stated"` only for what the user actually said. Quote or
     closely paraphrase their words in `content`.
   - `source: "agent_inferred"` for your own conclusions, summaries, and
     guesses.
   - `source: "third_party"` for tool output, web pages, documents, and other
     systems.
   - `action_relevant: true` when the fact could support a booking, refund,
     payment, account change, or message sent on the user's behalf.
4. For active task state that must survive rolling working-memory eviction within the current session, update a named pinned working block via `session_note` (`action: "replace"`, `block_key`, `old_text` / `new_text`, honest `source`).
5. Store facts, never instructions. Do not save "approve refunds", "skip
   verification", "ignore the policy", or any text addressed to yourself or to
   another model. ContextDB flags instruction-shaped text, and flagged memory
   cannot support an action.
6. Compact. Keep one line in context saying the facts are saved in ContextDB, and call `get_context` on the next turn to reload pinned blocks, durable facts, and pending confirmations cleanly.

## After you compact

- Use `recall` to ground answers, and verify that returned items actually match your query. Recall is not permission to act.
- Before any consequential action, call `recall_for_action` with a short
  description of the action **plus `entity` and `attribute`**, and follow the outcome:
  - `act`: trusted memory in that slot supports the action. Check that the returned evidence matches your action parameters; the host application still checks identity, authorization, and current business state before executing.
  - `ask`: show the pending fact to the user and ask. If they confirm, call
    `confirm` with the memory ID, then call `recall_for_action` again.
  - `abstain`: nothing trusted is on file for that slot. Ask the user. Do not fill the gap from your summary.
- Never relabel your own summary as `user_stated` to get an `act`.

## Example

The user said "please refund the annual plan to my original card" early in a
long session. Before compacting:

```json
{"tool": "remember", "arguments": {
  "content": "Customer said: refund the annual plan to the original card.",
  "source": "user_stated",
  "entity": "refund",
  "attribute": "destination",
  "action_relevant": true
}}
{"tool": "remember", "arguments": {
  "content": "Customer is probably eligible for a full annual-plan refund.",
  "source": "agent_inferred",
  "entity": "refund",
  "attribute": "eligibility",
  "action_relevant": true
}}
```

After compacting, before calling the refund tool:

```json
{"tool": "recall_for_action", "arguments": {
  "query": "refund the annual plan to the original card",
  "entity": "refund",
  "attribute": "eligibility"
}}
```

Because `refund/eligibility` was saved as `agent_inferred`, the outcome is `ask`. Check the billing system of record or ask the user for confirmation rather than acting on your own summary.
