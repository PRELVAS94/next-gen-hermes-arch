# Low-Level Design — Memory, Retrieval, Routing, Failure

Companion to [`HLD.md`](HLD.md). This is the layer where "use Postgres instead of
markdown files" stops being a slogan and becomes a schema.

---

## 1. The memory model

The core claim: **agent memory is a database, not a document.**

### 1.1 Core tables

```sql
-- A durable fact an agent learned. Append-only; never overwritten in place.
CREATE TABLE memory_fact (
    id              BIGSERIAL PRIMARY KEY,
    agent_id        TEXT        NOT NULL,           -- which agent learned it
    domain          TEXT        NOT NULL,           -- research | news | finance | ...
    subject         TEXT        NOT NULL,           -- normalised entity key
    predicate       TEXT        NOT NULL,           -- what is asserted
    object          TEXT        NOT NULL,           -- the asserted value
    confidence      NUMERIC(3,2) NOT NULL,          -- 0.00–1.00, extraction-scored
    source_kind     TEXT        NOT NULL,           -- web | user | tool | inference
    source_ref      TEXT,                           -- URL, message id, doc path
    observed_at     TIMESTAMPTZ NOT NULL,
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ,                    -- NULL = currently true
    superseded_by   BIGINT      REFERENCES memory_fact(id),
    embedding       VECTOR(1024),                   -- pgvector, for semantic recall
    CHECK (confidence >= 0 AND confidence <= 1),
    CHECK (valid_to IS NULL OR valid_to > valid_from)
);

-- Bitemporal: separate "when it was true" from "when we believed it".
-- This is what makes contradictions detectable rather than invisible.
CREATE TABLE memory_belief (
    fact_id         BIGINT      NOT NULL REFERENCES memory_fact(id),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    confidence      NUMERIC(3,2) NOT NULL,
    PRIMARY KEY (fact_id, recorded_at)
);

-- The episodic log: what happened, in order. Immutable.
CREATE TABLE episode (
    id              BIGSERIAL PRIMARY KEY,
    agent_id        TEXT        NOT NULL,
    session_id      TEXT        NOT NULL,
    kind            TEXT        NOT NULL,           -- turn | tool_call | observation
    payload         JSONB       NOT NULL,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Derived, rebuildable: facts that recur enough to be promoted to standing rules.
CREATE TABLE memory_rule (
    id              BIGSERIAL PRIMARY KEY,
    rule_key        TEXT UNIQUE NOT NULL,
    condition       JSONB       NOT NULL,
    action          JSONB       NOT NULL,
    occurrences     INT         NOT NULL DEFAULT 1,
    promoted_at     TIMESTAMPTZ,                    -- set once confirmed
    CHECK (occurrences >= 1)
);
```

### 1.2 Why bitemporal, specifically

A flat file cannot express *"this was true last week and is not true now."* Both
lines simply sit in the document and the model picks whichever it likes. With
`valid_from`/`valid_to` plus the `memory_belief` side-table:

- Current truth = `WHERE valid_to IS NULL`
- Truth *as of* a date = a range predicate on `valid_from`/`valid_to`
- A contradiction = two overlapping intervals on the same `(subject, predicate)` —
  **a query away**, surfacing into a review queue instead of a hallucination
- Corrections are new rows pointing at the old one (`superseded_by`), so history is
  never destroyed

### 1.3 Indexing

```sql
CREATE INDEX ON memory_fact (agent_id, domain, valid_to);
CREATE INDEX ON memory_fact (subject, predicate) WHERE valid_to IS NULL;
CREATE INDEX ON memory_fact USING hnsw (embedding vector_cosine_ops);
CREATE INDEX ON episode USING gin (payload jsonb_path_ops);
CREATE INDEX ON episode (occurred_at DESC);
```

The `WHERE valid_to IS NULL` partial index is the hot path: "what do we currently
believe about X" stays small even as the table grows.

### 1.4 Consolidation job (nightly)

1. Cluster recent episodes; extract candidate facts; score confidence.
2. Resolve against existing `(subject, predicate)`: insert, supersede, or conflict.
3. Promote facts seen ≥ N times across sessions into `memory_rule` candidates.
4. Re-embed anything whose text changed.
5. Emit a report: new facts, supersessions, unresolved conflicts.

Deterministic, scheduled, and **auditable** — every step writes to `episode`.

---

## 2. Retrieval path (the efficiency claim, concretely)

Replacing "stuff the memory file into the prompt" with a targeted query:

```
1. Intent extraction      → (agent_id, domain, subjects[], horizon)
2. Candidate generation   → lexical (tsvector) UNION semantic (hnsw, top-k)
3. Filter                 → current truth, agent scope, confidence ≥ floor
4. Rank                   → recency × confidence × source trust × similarity
5. Budget                 → trim to a token ceiling; NEVER dump the whole store
6. Assemble               → facts + provenance, ~2–5k tokens
```

**This is the architecture's central efficiency argument.** A fleet that spends ~97%
of its tokens on input and re-reads its entire memory every call is paying to
re-process the same text over and over. A retrieval step that pulls 2–5k relevant
tokens instead of 200k of everything is not a marginal saving — it is the difference
between context that grows without bound and context that stays flat as the knowledge
base grows to millions of rows.

---

## 3. Inference routing

Two hosts, one logical inference plane. The GPU host serves the heavy model; the
control plane serves the conversational one locally.

```
request ──▶ classify ──┬── orchestration / planning / synthesis ──▶ Qwen 3.8
                       │                                            (PC + RTX 5090)
                       ├── conversation / briefing / summary  ────▶ Gemma 4
                       │                                            (Mac mini, local)
                       └── explicit escalation ──────────────────▶ hosted frontier
                                                                  (opt-in, cost-logged)
```

- **Qwen 3.8 lives on the GPU host** — ~15–20 GB of 32 GB VRAM, leaving room for a
  long KV cache. 1,792 GB/s of bandwidth is what makes it fast.
- **Gemma 4 lives on the control plane** — ~14 GB resident in 24 GB, so conversational
  turns need no network hop and no GPU wake-up.
- Each agent declares its default lane; the orchestrator may override per task.
- Escalation to a hosted model is **explicit and logged** — never silent, so cost
  surprises stay impossible.

### 3.1 Crossing the machine boundary

The inference link is the one genuinely new failure mode this topology introduces, so
it is specified rather than assumed:

```
control plane                              inference host
     │                                            │
     ├─ health probe (cheap, periodic) ──────────▶ │
     │                                            ├─ model loaded? VRAM free?
     │ ◀──────────── ready / warming / absent ────┤
     │                                            │
     ├─ wake-on-demand (Wake-on-LAN / systemd) ───▶│  (if asleep)
     │                                            │
     └─ inference request (mTLS, LAN only) ───────▶│
                                                  └─ Qwen 3.8 → response
```

- **Authenticated and encrypted** — mTLS, LAN-scoped. Prompts and memory never cross
  a public network.
- **Health-checked, not assumed.** The control plane probes before routing, so a dead
  GPU host is a detected condition rather than a timeout.
- **Degradation is defined.** If the inference host is unavailable, the orchestrator
  either serves the task with Gemma 4 locally or fails the task explicitly — it does
  not silently pretend the reasoning happened.
- **Cold-start is budgeted.** A sleeping host needs power-on plus model load before it
  can answer. For latency-critical paths the host stays warm; for batch work the
  orchestrator accepts the wake latency in exchange for the idle power saving.

### 3.2 Tool calling

MCP tool schemas are large. They are **retrieved, not always-injected**: only tools
relevant to the classified intent are placed in context. The full catalogue stays
discoverable via a search tool, so capability is not lost — only the constant token
overhead is.

---

## 4. Task fabric and orchestration

Multi-step work is a row, not a conversation convention:

```sql
CREATE TABLE task (
    id              BIGSERIAL PRIMARY KEY,
    goal            TEXT        NOT NULL,
    assignee        TEXT        NOT NULL,           -- agent id, or 'orchestrator'
    state           TEXT        NOT NULL,           -- ready|running|blocked|done|failed
    parent_id       BIGINT      REFERENCES task(id),
    input           JSONB,
    output          JSONB,
    attempt         INT         NOT NULL DEFAULT 0,
    idempotency_key TEXT UNIQUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    started_at      TIMESTAMPTZ,
    finished_at     TIMESTAMPTZ
);
CREATE INDEX ON task (assignee, state) WHERE state IN ('ready','running');
```

A cross-domain question becomes one fan-out: N specialist rows + a **verifier** row
+ a **synthesiser** row, the latter gated on the former. That gives the "one
verified answer" property structurally, rather than asking the orchestrator to
remember to check.

`attempt` and `idempotency_key` make retries safe — an agent that crashes mid-run
resumes rather than duplicating side effects.

---

## 5. Integration layer

### 5.1 MCP (tool plane)

- Each capability — web fetch, filesystem, shell, database query, calendar, mail,
  browser automation — is an MCP server.
- Agents never hard-code an integration; they are handed tool schemas.
- Hermes itself can run **as** an MCP server, so external agents and IDEs drive it.

### 5.2 Webhooks (event plane)

```
external event ──▶ webhook subscription ──▶ orchestrator wake
                                            │
                                            ├─ dedupe by event id (episode table)
                                            ├─ resolve to an intent
                                            └─ create task (or ignore)
```

Wake-on-event rather than poll. A price threshold crossing, an inbound message, an
upstream callback — each becomes a targeted agent run, not a periodic sweep that
mostly finds nothing.

### 5.3 Scheduling

Recurring work (briefings, consolidation, cost audits) stays on cron. The rule:
**poll nothing you can be told about.** Schedule what is genuinely periodic; use
webhooks for everything else.

---

## 6. Failure modes and mitigations

| Failure | Behaviour | Mitigation |
|---|---|---|
| Inference host unavailable | Orchestrator degrades to local Gemma 4 or fails the task explicitly | Health probe before routing; never a silent pretend-success |
| Inference link degraded | Requests time out against a defined budget | LAN + mTLS; cold-start latency budgeted per task class |
| Control plane (Mac mini) down | Fleet is down — it holds memory, scheduling and orchestration | Backups of the Postgres store; GPU host is not the bottleneck |
| Local model unavailable | Requests fail fast; nothing is silently downgraded | Health check + explicit escalation path |
| Postgres unavailable | Agents degrade to **read-only** | Storage on the control plane; no silent writes |
| Extraction produces a bad fact | Confidence-scored; superseded, not deleted | Nightly consolidation + conflict queue |
| Agent crashes mid-task | Row stays `running`; lease expires | `attempt` + `idempotency_key` → safe retry |
| Runaway agent loop | Bounded by per-run token/turn budget | Hard caps in the runtime, not in the prompt |
| Duplicate side effect | Blocked by `idempotency_key` | Unique constraint |

The theme: **fail detectably, never silently.** A wrong answer that looks confident
is worse than an error.

---

## 7. Security and privacy posture

- Inference is local — prompts, memory and derived facts never leave the two machines
  or cross a public network. The only inter-host traffic is inference over the LAN,
  authenticated with mTLS.
- Postgres holds private data (finance, personal context); row-level access per
  agent domain is available if agents are ever untrusted relative to each other.
- MCP servers are the only egress path, so egress is **enumerable** — a finite,
  auditable list of capabilities rather than arbitrary network access from agent
  code.
- The audit trail is the `episode` table: every tool call and observation, in order.
