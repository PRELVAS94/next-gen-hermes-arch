# Running-Cost Analysis

**All figures are modelled, not quoted.** Assumptions are stated explicitly so you can
substitute your own numbers. Prices are indicative and change.

---

## 1. The workload being priced

The deployed fleet provides a real measurement to model against (last 7 days):

| Metric | Measured |
|---|---|
| Volume (7 days) | **90.9M tokens** — 88.2M input / 2.6M output |
| Shape | **~97% input**, ~3% output |
| Scheduled runs | 314 per week (172 produced no output — silent monitors) |
| Prompt-cache reuse | **89%** of repeat context |

Annualised: **~4.59B input / ~0.135B output tokens.**

The input-heavy shape is the important detail — it is why caching and precise
retrieval dominate every other optimisation.

**One nuance worth stating precisely, because it is easy to get wrong:** ~97% of the
*tokens* are input, but that is not the same as ~97% of the *bill*. On a cheap tier
where input costs fractions of a cent per million, a small output volume still
dominates the invoice — on the cheapest tier modelled below, output is ~96% of the
cost on only ~3% of the tokens. On frontier-tier pricing, input dominates at ~87% of
the bill. The efficiency argument holds either way; the *reason* it holds differs by
tier, and stating it loosely would be misleading.

---

## 2. Cost of the same workload, three ways

| Approach | Per week | **Per year** |
|---|---:|---:|
| Local — split topology, electricity only | €3–6 | **€168–305** |
| Cheapest metered tier (DeepSeek V4.1 Flash class) | $6.50 | **$338** |
| Mid tier (~$0.30 / $2.50 per M) | $32.96 | **$1,714** |
| Frontier tier (~$3 / $15 per M) | $303.60 | **$15,787** |

### 2.1 What the local figure is made of

The local number is a *two-machine* electricity bill at €0.20/kWh:

| Component | Draw | Per year |
|---|---|---:|
| **Mac mini M4** — always-on control plane | ~10 W average | **€18** |
| **RTX 5090 host** — always-on (80 W idle, 500 W under inference, ~20% duty) | ~164 W average | **€287** |
| **RTX 5090 host** — wake-on-demand (sleep between jobs, ~4 h/day inference) | ~86 W average | **€150** |
| **Total, always-on** | | **€305** |
| **Total, wake-on-demand** | | **€168** |

Two things worth noting. First, the always-on control plane costs **eighteen euros a
year** — Apple silicon at ~10 W is genuinely negligible, which is the whole argument
for putting orchestration there. Second, wake-on-demand **halves the GPU host's power
bill**, and that is a real design choice rather than a rounding: a 575 W card left
idling is most of the cost of running this at all.

---

## 3. The honest caveat — read this before quoting any of it

**Local inference does not win on pure arithmetic against the cheapest metered
tier.** Against that tier it is genuinely close — a ~€3,850 two-machine setup pays
back in years, and only on the lower power profile. Pretending otherwise would be the
easy lie.

Where local wins, and wins decisively, is against the tiers you would actually
choose for serious work — and on the shape of the cost:

| vs | Annual saving | Payback on ~€3,850 capex |
|---|---:|---:|
| Cheapest metered tier | ~$33–170 | ~23–120 years |
| Mid tier | ~$1,409–1,546 | **~2.6–2.8 years** |
| Frontier tier | ~$15,482–15,619 | **~3–4 months** |

**Capex assumption:** Mac mini M4 24 GB ~€900; PC with an RTX 5090 ~€2,950 (card
~€1,850 plus platform ~€1,100). Substitute your own quotes — the *shape* of the
conclusion holds, the exact years do not.

Three further points that a pure price table misses:

1. **The cost stops scaling with volume.** The metered numbers above are linear in
   tokens — double the work, double the bill. The local figure is not. The more
   ambitious the fleet becomes, the more the gap widens, and that is the real
   argument.
2. **Rate limits are a capability ceiling, not just a bill.** Metered tiers cap how
   much you can run per minute. A local host is bounded by memory bandwidth, not by
   a vendor's allowance.
3. **Marginal cost is genuinely zero.** A new agent, a longer context, an extra
   scheduled run — none of them add a line item to a monthly bill. That changes what
   you are willing to build.

---

## 4. Where the *efficiency* (not just cost) comes from

Cost is downstream of token efficiency. The architecture attacks input tokens, which
are **~97% of the volume** — though, as noted above, input is not necessarily the
dominant line *on the bill* at every tier:

| Lever | Mechanism | Effect |
|---|---|---|
| **Postgres retrieval** | Query for the relevant 2–5k tokens instead of injecting the whole memory | Removes unbounded context growth |
| **Bitemporal validity** | Retrieve *current* truth by default, not all history | Keeps the working set small |
| **Tool-schema retrieval** | Inject only tools matching the intent | Avoids a constant large prefix |
| **Prompt caching** | Stable prefixes stay cacheable | Measured at 89% reuse |
| **Hard run budgets** | Cap tokens/turns per run in the runtime | Stops runaway loops at the source |
| **Event-driven wake** | Webhooks instead of polling | Eliminates work that finds nothing |

The last row deserves emphasis: in the measured fleet, **172 of 314 weekly runs
produced no output** — silent monitors working as designed. Those runs still cost
tokens. Converting them to webhooks is free efficiency.

---

## 5. What is *not* modelled

Stated plainly, because the omission would otherwise flatter the design:

- **Capital cost of the machines** is treated as sunk in the per-year comparison.
- **Engineering time** to build and maintain the platform — typically the largest
  real cost in any self-hosted stack.
- **Failover capacity.** Neither host is redundant; replacing the GPU host on failure
  doubles that half of the capex.
- **The inference-link overhead.** LAN latency is small but not zero, and moving tokens
  between two machines is a cost the single-host version did not have.
- **Model quality trade-off.** Local models give up some peak reasoning capability
  versus frontier tiers. That is a real cost, paid in capability rather than cash.
- **Price drift.** Hosted prices have fallen historically; local electricity has not,
  though hardware performance per euro has improved.

---

## 6. Bottom line

- For **privacy-bound personal workloads**, local inference is close to a non-decision:
  the data cannot leave your machines, so the cost comparison is moot.
- For **volume-heavy, latency-tolerant agent fleets**, local wins against every tier
  you would realistically run serious work on, and the advantage grows with ambition.
- For a **tiny workload on the cheapest available tier**, metered is cheaper and will
  stay cheaper. Saying otherwise would be dishonest.

The strategic point is not "local is always cheaper." It is: **local turns a variable
cost into a fixed one, and removes the vendor's rate limit from your roadmap.**
