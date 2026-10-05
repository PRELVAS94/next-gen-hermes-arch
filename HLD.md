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

   ══════════════════ control plane (Mac mini) ══════════════════════════

                        ┌───────────────┴──────────────────────────┐
                        │  MANIFEST GATEWAY — inference routing    │
                        │  ONE OpenAI-compatible endpoint for the  │
                        │  whole fleet · model:"auto" · fallback   │
                        │  ladder · per-route params · token and   │
                        │  cost observability                      │
                        └──┬───────────────┬──────────────┬────────┘
                           │               │              │
        ┌──────────────────┴──┐ ┌──────────┴────────┐ ┌───┴────────────────┐
        │  GEMMA 4            │ │  QWEN 3.8         │ │  CLOUD LANE        │
        │  Mac mini · resident│ │  PC + RTX 5090    │ │  OpenRouter · xAI  │
        │  26B-A4B, ~4B active│ │  wake-on-demand   │ │  keys + subs       │
        │  ~14 GB · 170 GB/s  │ │  ~15–20 GB of 32  │ │  frontier escape   │
        │  conversation·chat  │ │  deep reasoning   │ │  hatch + fallback  │
        └─────────────────────┘ └───────────────────┘ └────────────────────┘
           ~10 W, always on        575 W, sleeps         metered, last resort
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

## 4. The routing plane — Manifest Gateway

Between the agents and every model sits **one** gateway. [Manifest](https://manifest.build)
is an open-source LLM gateway (self-hosted, Docker) that exposes a single
OpenAI-compatible endpoint in front of many backends. It is the component that makes
"cloud or local, depending on the task" a routing *policy* rather than scattered
per-agent logic.

### 4.1 What it unifies

The gateway fronts four kinds of backend **simultaneously** — the exact mix this design
needs:

| Backend kind | In this design | Why it matters here |
|---|---|---|
| **Local server** | Qwen 3.8 (RTX 5090), Gemma 4 (Mac mini) via Ollama / llama.cpp | The privacy and zero-marginal-cost lane |
| **API keys** | OpenRouter, plus any direct provider | Burst capacity and frontier capability |
| **Provider subscriptions** | Subscription-based access used as-is | Reuses capacity already paid for rather than re-buying per token |
| **Custom OpenAI-compatible endpoints** | Any bespoke or self-built server | Escape hatch for future local or private models |

Every backend is reached the same way, so an agent can mix local, subscription and
metered models transparently — no per-agent provider code.

### 4.2 How routing decisions are made

```
agent request
     │
     ▼
Manifest Gateway ── policy ──┬─▶ local  (privacy / zero marginal cost)  ← default
     │                       ├─▶ subscription (already-paid capacity)
     │                       ├─▶ metered API (burst, frontier capability)
     │                       └─▶ escalation (a step the local model can't carry)
     │
     ├─ fallback ladder: provider A → provider B → local, per request
     ├─ autofix: repair malformed upstream requests instead of failing the run
     └─ observability: every request, token and cost — per agent, key, provider
```

The policy the gateway enforces:

1. **Local first.** Anything the local models can carry stays local — that is the
   cost and privacy default.
2. **Escalate on need, not on habit.** A step the local model can't carry is routed to
   the cloud lane *explicitly and observably*, never silently.
3. **Fall back in order.** If the RTX host is unavailable, the ladder steps to the
   cloud lane rather than failing the task — this is what makes the second-machine
   topology safe to run.
4. **Never a surprise bill.** Because every request is attributed to an agent and a
   provider, metered spend is visible per call instead of discovered at month end.

### 4.3 Why not just call providers directly

Four reasons, each of which the design would otherwise have to build itself:

- **One endpoint, many backends.** Adding or swapping a model is a gateway config
  change, not a code change in six agents.
- **Fallback and repair are infrastructure concerns.** Retry ladders, provider outages
  and malformed-request repair belong in a proxy, not in agent prompts.
- **Cost attribution comes free.** Per-agent, per-key token accounting is exactly the
  instrumentation this design otherwise has to hand-roll — and it directly serves the
  efficiency claims in [`COST_MODEL.md`](COST_MODEL.md).
- **It keeps the routing policy honest.** With one choke point, "local first, cloud on
  exception" is a rule the system *enforces* and *logs*, rather than an intention each
  agent is trusted to follow.

*Note:* the gateway is the control-plane component that makes this policy real. Local
servers it routes to still live on their own hosts, so the two-machine split is
unchanged — the gateway decides *where* a request goes, not where the models run.

## 5. The memory substrate

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

## 6. How work enters the system

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

## 7. Why MCP matters here

MCP (Model Context Protocol) makes tools a **standard interface** rather than
bespoke code per integration. Practically: an agent can be handed a new capability
without touching agent logic, and the runtime can itself be exposed *as* an MCP server
so other agents or IDEs can drive it. It is the difference between seven scripts that
each know how to call the web, and one system where capability is a plugin.

## 8. Cost shape

See [`COST_MODEL.md`](COST_MODEL.md) for the arithmetic. In one line:
**capex × amortisation + electricity**, largely independent of volume — versus a
metered bill that scales with every token and is capped by rate limits. The split
design lowers capex *and* lowers the power bill, because the high-draw GPU sits idle
most of the day.

## 9. Honest limitations

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
- **The gateway is a single routing point, and a single point of failure.** Every model
  call passes through it, so it must be supervised and restartable, and its own health
  is part of the control plane's monitoring. That is a real cost of the simplification —
  accepted because one configurable choke point beats six agents each implementing
  their own provider logic and fallbacks.
- **Memory quality is the real risk.** A relational store makes retrieval precise;
  it does not make what you stored *correct*. Extraction and confidence scoring need
  as much care as the schema.
