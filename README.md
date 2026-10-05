# Next-Gen Hermes Architecture — Reference Design

> **Status: reference architecture / design blueprint.** This is a forward-looking
> target design, not a deployed production topology. Published for engineering
> discussion.

A multi-agent AI runtime where **all reasoning runs locally** — a low-power **Mac mini
M4** control plane beside a dedicated **RTX 5090** inference host — and **all memory
lives in PostgreSQL** instead of markdown files.

---

## The one-paragraph version

Seven specialised agents share one relational memory and one scheduling fabric. A
dedicated orchestrator plans and routes; six domain agents (research, news, finance,
engineering, books, gaming) each own a slice of the work. Agents reach the outside
world through **MCP servers** for tools and **webhooks** for event-driven activation.
Every fact an agent learns is written to **Postgres** with provenance, so memory is
queried, versioned and joined — not grepped out of a blob of markdown. Inference is
local and split by workload: **Qwen 3.8 on the RTX 5090** for orchestration and deep
reasoning, **Gemma 4 on the Mac mini** for conversational surfaces. Marginal cost per
token: zero.

---

## Documents

| File | What it is | Read time |
|---|---|---|
| [`HLD.md`](HLD.md) | High-level design — components, data flow, why it's shaped this way | 5 min |
| [`LLD.md`](LLD.md) | Low-level design — the Postgres memory model, retrieval path, routing, failure modes | 7 min |
| [`COST_MODEL.md`](COST_MODEL.md) | Running-cost analysis, with assumptions stated and the honest caveat | 4 min |
| `diagrams/` | Rendered architecture diagram | — |

---

## The topology in one line

**Mac mini M4 (24 GB)** runs the always-on control plane — orchestrator, memory,
tools, scheduling — at ~10 W. **PC + RTX 5090 (32 GB VRAM)** runs the heavy inference
model at 1,792 GB/s, waking on demand.

---

## The three claims this design makes

**1. Persistent relational memory beats flat-file memory.**
Markdown memory is a context blob: you retrieve it by stuffing it into the prompt and
hoping the model notices the right line. Postgres memory is a *query*. Facts carry
`valid_from`/`valid_to`, provenance, confidence and embeddings — so retrieval is
targeted, contradictions are surfaceable, and memory is auditable. On a fleet whose
token profile is **~97% input**, replacing "re-read everything" with "retrieve
precisely" is the single largest efficiency lever available.

**2. Splitting the silicon by bandwidth changes the cost *shape*, not just the *number*.**
Token generation tracks memory bandwidth, so the heavy model belongs on the GPU and
the always-on orchestration belongs on something that sips power. Metered APIs scale
linearly with usage — the chattier the fleet, the bigger the bill, and rate limits cap
how ambitious you can be. Local inverts that: a fixed capex and an electricity bill
that barely moves with workload. The trade is stated plainly in
[`COST_MODEL.md`](COST_MODEL.md) — local wins decisively against frontier-class pricing;
it does *not* beat the very cheapest metered tier on pure arithmetic, and it is not
claimed to.

**3. MCP + webhooks is what makes it a system rather than seven scripts.**
MCP gives agents a standard, reusable tool interface. Webhooks invert the model from
"agent polls" to "event wakes agent" — and here the wake-up can power on the GPU host
too. Together they let one orchestrator coordinate specialists without hard-coding
every integration.

---

## Use cases this shape is built for

- **System-level architecture design** — long-horizon reasoning over a codebase or
  infra estate, where context must persist across sessions.
- **Deep research** — multi-source gathering, then synthesis with citations retained.
- **Automated news** — scheduled and event-driven briefings, deduplicated against
  what the agent already knows.
- **Personal finance management** — portfolio tracking against private data that
  should never leave your machines.
