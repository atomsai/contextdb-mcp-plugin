# Compaction proof: a self-written note cannot authorize an action

This example shows what ContextDB does with the notes an agent writes for
itself while compacting its context. It runs offline on the Apache-2.0
`pycontextdb` SDK with a deterministic mock embedder. It needs no API key, no
ContextDB Cloud account, and no network after install. All customer IDs and
content are synthetic.

```bash
python3 -m venv .venv
.venv/bin/pip install pycontextdb==0.4.4
.venv/bin/python compaction_proof.py
```

The SDK logs one warning that no LLM key is configured. Nothing in this
example calls an LLM.

Output verified on October 5, 2026 with `pycontextdb==0.4.4` on Python 3.14:

```json
[
  {
    "note": "customer's own words",
    "source": "user_stated",
    "outcome": "act",
    "reason": "trusted facts available"
  },
  {
    "note": "agent's own compaction summary",
    "source": "agent_inferred",
    "outcome": "ask",
    "reason": "facts present but require confirmation"
  },
  {
    "note": "instruction written into the agent's own notes",
    "source": "third_party",
    "injection_suspect": true,
    "outcome": "abstain",
    "reason": "nothing on file (relevance floor or empty store)"
  },
  {
    "note": "agent's summary after the customer confirms it",
    "source": "agent_inferred",
    "outcome": "act",
    "reason": "trusted facts available"
  }
]
```

What each row shows:

1. A fact the customer stated, saved as `user_stated` and marked
   action-relevant, can support the refund.
2. The agent's own summary, saved as `agent_inferred`, cannot. The outcome is
   `ask` until someone confirms it.
3. An instruction the agent wrote into its own notes is flagged at write time.
   ContextDB demotes it to `third_party` at confidence 0, and the outcome is
   `abstain`.
4. After the customer confirms the summary, it can support the refund.

`act` is advice, not execution. The application still authenticates the
customer, authorizes the refund, and checks current account state.

The injection screen is deliberately high-precision. It catches
instruction-shaped text such as `SYSTEM:` or "ignore previous instructions".
It is not a complete defense against every manipulation, which is why
agent-written notes need confirmation regardless.
