---
name: contextdb-action-gate
description: Verify trusted customer memory before consequential agent actions (bookings, refunds, outbound emails, billing changes, or CRM/ERP writes) using ContextDB recall_for_action, entity+attribute slot scoping, user confirmation, and host-enforced act/ask/abstain branching. Do not use as a substitute for host authentication or business authorization.
---

# Gate consequential actions with `recall_for_action`

Before executing any tool or API call with external side effects that relies on remembered context, verify that the underlying memory is current, uncontested, and trusted under policy.

Canonical docs:
- Trust & action policy: `https://contextdb.ai/trust`
- MCP connector guide: `https://contextdb.ai/mcp`
- Coding-agent & HTTP receipt flow: `https://contextdb.ai/for-agents.txt`

## 1. Prerequisites

- Identify the exact external action about to run (e.g., booking an appointment, issuing a refund, changing a subscription plan, sending an external message) and the specific memory slot (`entity` + `attribute`) it depends on.
- Confirm whether ContextDB MCP (`recall_for_action`, `pending_confirmations`, `confirm`) or the backend SDK (`CloudClient.evaluate_action` / `db.factual.recall_for_action`) is configured.

## 2. Exact pre-action workflow

1. **Evaluate the required slot (`recall_for_action`):**
   - Always pass both `entity` and `attribute` (for example, `entity: "appointment"`, `attribute: "time"` or `entity: "user"`, `attribute: "billing_email"`). Omitting the slot allows unrelated trusted memories in the partition to produce `act`.
2. **Branch strictly on `outcome`:**
   - **`act`:** Trusted memory exists in that slot. Verify that the returned `memories` actually support the specific parameters of your proposed action, then apply host authentication, business authorization, and live system-of-record checks before executing.
   - **`ask`:** Evidence is tentative (`agent_inferred`), `third_party`, or contested (`pending_confirmation_ids` non-empty). **Pause execution.** Surface the pending fact to the user and ask for explicit confirmation in the current conversation.
   - **`abstain`:** No trusted or pending evidence exists for that slot. **Do not guess or act.** Ask the user for the missing information or consult the authoritative system of record.
3. **Confirm only after explicit user approval (`confirm` -> `recall_for_action`):**
   - Once the user explicitly approves the pending fact (and the host authenticates the user where required), call `confirm` with that `memory_id`.
   - Call `recall_for_action` again with the same `entity` and `attribute` so the audited decision reflects the updated policy state.
4. **Report execution in backend integrations (`POST /v1/receipts` / `report_execution`):**
   - When integrating via the HTTP API or `CloudClient`, record `succeeded`, `failed`, or `skipped` against `decision_id` with a stable `Idempotency-Key`.

## 3. Safety, privacy, and idempotency rules

- `act` means **trusted memory is on file**, NOT that the action is authorized or that downstream inventory/funds exist. Never claim ContextDB itself executed or authorized a booking, refund, or payment.
- Never self-confirm (`confirm`) an `agent_inferred` or `third_party` memory without explicit user approval in the conversation.
- On backend retries of `/v1/confirm` or `/v1/receipts`, reuse the exact same `Idempotency-Key`. If `/v1/confirm` or `/v1/evolve` returns `409 idempotency_ambiguous`, stop and inspect current state rather than generating a fresh key.

## 4. Unsupported cases

- ContextDB cannot block a host process that ignores its `ask` or `abstain` return value; enforcement must live in the caller before the side-effecting tool runs.
- ContextDB does not verify live bank balances, calendar availability, or third-party CRM state.
