# High-Level Design — Local-First Multi-Agent Runtime

**Target architecture.** Split across two machines: an always-on **Mac mini M4 (24 GB)**
control plane and a dedicated **RTX 5090** inference host. Local inference throughout;
PostgreSQL as the memory substrate; MCP for tools; webhooks for event-driven work.

---

## 1. Design goals

| Goal | How the design meets it |
|---|---|
| Predictable running cost | Local inference — fixed capex, electricity measured in tens of euros a year |
| Memory that persists and compounds | PostgreSQL, not markdown — queryable, versioned, provenance-tracked |
| Data that never leaves the building | No inference or memory traffic to a third party |
| Right silicon for each job | GPU host for reasoning throughput; low-power Apple silicon for orchestration |
| Specialisation without chaos | One orchestrator + six domain agents, one shared memory, one task fabric |
| Integrations without bespoke glue | MCP servers for tools; webhooks for activation |

## 2. Component model

```
                        ┌──────────────────────────────────────────┐
   Telegram / chat ────▶│  ORCHESTRATOR AGENT      (Mac mini M4)   │
   Webhook events  ────▶│  · classifies intent (single vs multi-   │
   Schedules (cron)────▶│    domain) and decomposes work           │
                        │  · routes to specialists, verifies their │
                        │    output, synthesises ONE answer        │
                        └───────────────┬──────────────────────────┘
                                        │  task fabric (Kanban)
        ┌───────────┬───────────┬───────┴────┬───────────┬───────────┐
        ▼           ▼           ▼            ▼           ▼           ▼
    RESEARCH     NEWS      FINANCE     ENGINEERING    BOOKS      GAMING
     agent       agent      agent        agent        agent      agent
        └───────────┴───────────┴────────────┴───────────┴───────────┘
                                        │
                        ┌───────────────┴──────────────────────────┐
                        │  TOOL LAYER — MCP servers                │
                        │  web · files · shell · databases ·       │
                        │  calendars · mail · browser              │
                        └───────────────┬──────────────────────────┘
                                        │
                        ┌───────────────┴──────────────────────────┐
                        │  MEMORY — PostgreSQL       (Mac mini)    │
                        │  facts · provenance · embeddings ·       │
                        │  episodic log · task state · audit       │
                        └──────────────────────────────────────────┘

   ══════════════════ control plane ═╪═ inference plane ═════════════════

   ┌───────────────────────────────┐  │  ┌──────────────────────────────┐
   │  MAC MINI M4 · 24 GB          │  │  │  PC + RTX 5090 · 32 GB       │
   │  170 GB/s · ~10 W always-on   │  │  │  1,792 GB/s · 575 W TDP      │
   │  CONTROL PLANE                │◀─┼─▶│  INFERENCE PLANE             │
   │                               │  │  │                              │
   │  GEMMA 4 (26B-A4B, ~4B active)│  │  │  QWEN 3.8                    │
   │  conversation · briefings     │  │  │  orchestration · planning    │
   │  summarisation · chat surfaces│  │  │  deep reasoning · synthesis  │
   │  ~14 GB resident              │  │  │  ~15–20 GB + long KV cache   │
   └───────────────────────────────┘  │  └──────────────────────────────┘
     orchestration · memory · tools   │    dedicated GPU inference,
     scheduling · always available    │    wake-on-demand
```

## 3. Why two machines, and why these two

An earlier iteration of this design put everything on a single Mac Studio. Splitting
it across two hosts is a better fit on three axes.

**1. Bandwidth is what decode speed actually tracks.**
Token generation is memory-bandwidth-bound, not compute-bound. The RTX 5090 moves
**1,792 GB/s** against roughly **170 GB/s** on the Mac mini M4 — an order of magnitude
of headroom for the heavy reasoning model, and *the* reason Qwen 3.8 belongs on the
GPU rather than the Mac.

**2. The always-on tier should be low-power.**
Most of a fleet's wall-clock time is orchestration: routing, waiting, writing to a
database, deciding what to do next. That work is not GPU-shaped. A Mac mini M4
averaging roughly **10 W** is an excellent always-available control plane, and it lets
the **575 W** GPU host stay powered down until something genuinely needs reasoning.

**3. Failure domains separate.**
Memory, orchestration and scheduling survive the inference host being rebooted,
upgraded or busy. That is the opposite of the single-host design, where one restart
took the whole fleet down.

### 3.1 The model split, and why each model fits its host

| | Host | Footprint | Why it fits there |
|---|---|---|---|
| **Gemma 4** 26B-A4B | Mac mini M4 (24 GB) | ~14 GB, ~4 GB headroom | A mixture-of-experts model with only ~4B parameters *active* per token — so it stays responsive even on modest bandwidth, and fits permanently resident in 24 GB |
| **Qwen 3.8** | RTX 5090 (32 GB VRAM) | ~15–20 GB + KV cache | Fits in VRAM **with room for a long context window**, which is precisely what the GPU's 32 GB buys |

The split is by workload **character**, not by size: Qwen carries planning and
synthesis, where a wrong answer is expensive and latency is tolerable; Gemma carries
high-frequency conversational surfaces, where fluency matters more than depth. Each
model is resident on its own host, so there is no swap penalty between a chat turn and
an orchestration turn.

## 4. The memory substrate

`MEMORY.md`-style files are replaced by a relational store. The short version of why:

| Flat-file memory | PostgreSQL memory |
|---|---|
| Retrieved by stuffing into context | Retrieved by **query** — targeted, cheap |
| Contradictions invisible (both lines present) | Contradictions **detectable** via `valid_from`/`valid_to` |
| No provenance | Every fact carries source, timestamp, confidence |
| Not joinable | Joins across agents, domains, time |
| Grows context linearly | Grows storage, not context |
| No transactional guarantees | ACID — no partial writes, no torn state |

The deployed fleet this models runs at **~97% input tokens**. "Retrieve precisely
instead of re-reading everything" is therefore the architecture's biggest single
efficiency win. Full schema in [`LLD.md`](LLD.md).

## 5. How work enters the system

Three entry paths, deliberately distinct:

1. **Conversational** — a human message arrives (chat platform). The orchestrator
   classifies it: single-domain questions go to one agent; cross-domain questions
   decompose into a fan-out of specialists plus a verifier and a synthesiser.
2. **Scheduled** — cron-style jobs for recurring work (morning briefings, portfolio
   snapshots, memory consolidation).
3. **Event-driven** — **webhooks** invert polling into wake-on-event: a price
   threshold, an inbound message, an external system's callback. On this design that
   is doubly valuable, because a wake-up can also *power on the GPU host*. The agent
   runs — and the expensive silicon wakes — only when something actually happened.

## 6. Why MCP matters here

MCP (Model Context Protocol) makes tools a **standard interface** rather than
bespoke code per integration. Practically: an agent can be handed a new capability
without touching agent logic, and the runtime can itself be exposed *as* an MCP server
so other agents or IDEs can drive it. It is the difference between seven scripts that
each know how to call the web, and one system where capability is a plugin.

## 7. Cost shape

See [`COST_MODEL.md`](COST_MODEL.md) for the arithmetic. In one line:
**capex × amortisation + electricity**, largely independent of volume — versus a
metered bill that scales with every token and is capped by rate limits. The split
design lowers capex *and* lowers the power bill, because the high-draw GPU sits idle
most of the day.

## 8. Honest limitations

Stated because a design document that only lists strengths is marketing, not
engineering.

- **The GPU host is a second failure domain, and the network is the seam.** If the
  inference link is down, reasoning degrades to whatever the control plane can serve
  locally. Mitigation: the orchestrator detects the loss and degrades *explicitly*
  rather than silently.
- **575 W is real.** At sustained inference the GPU host draws like a space heater.
  Wake-on-demand is not a nicety here; it is a design requirement.
- **Local models trail frontier models on the hardest reasoning.** The two-model split
  mitigates it; it does not eliminate it. An escape hatch to burst a hard step to a
  hosted frontier model is deliberately left in the design, and cost-logged.
- **24 GB is a hard ceiling on the control plane.** The Mac mini runs Gemma 4
  comfortably and Qwen 3.8 *not* comfortably — that constraint is exactly why the
  design is two machines rather than one.
- **Memory quality is the real risk.** A relational store makes retrieval precise;
  it does not make what you stored *correct*. Extraction and confidence scoring need
  as much care as the schema.
