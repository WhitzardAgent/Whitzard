<h1>Whitzard （白泽）</h1>

<p><em>An agentic system for real-world vulnerability reproduction.</em></p>

<p><strong>English</strong> · <a href="./README.zh-CN.md">中文</a></p>

---

## 69.0% on CyberGym

Whitzard (白泽) reproduces **1,040 of 1,507** real-world vulnerability tasks on [CyberGym](https://www.cybergym.io/cybergym/) — a **69.0% pass@1 success rate** — driven by an **open-weight** base model, **GLM-5.1-FP8**, under a 2.5-hour per-task budget.

CyberGym is, today, the most demanding public benchmark for offensive-security agents: 1,507 real vulnerabilities across 188 open-source projects, each asking an agent to produce a proof-of-concept input that reproduces the flaw from nothing but a natural-language description and the codebase. Whitzard reaches this result with a model whose weights are public — the capability lives in the system around the model, not in a privileged frontier model.

> In the legend, 白泽 (Bái Zé) is the beast that knows every creature under heaven — the ten-thousand forms of things that go wrong. It does not guess; it recognizes. We named the system after it for a reason.

## What Whitzard is

Whitzard is not a prompt or a generic coding loop. It is a single, **evidence-driven** security agent wrapped in a runtime built for long, unassisted investigations — an agent that reads unfamiliar code, forms a hypothesis about *why* it breaks, constructs an input that proves it, and confirms the crash, all inside a wall-clock budget with no human in the loop.

A few of the ideas that make it work:

- **Evidence, not guessing.** One thing, and only one thing, marks a task solved: a verified crash from a single submission path. Source, shell output, debugger transcripts, and the model's own confidence are diagnostic evidence — never proof.
- **A high-density agent–computer interface.** Every tool the agent can call returns two synchronized views: a typed result for the machinery, and one deterministic summary *card* for the model. The agent reasons over clean, comparable observations, and always knows whether a negative result is complete or merely truncated.
- **State, not transcript.** The model works against a compact, always-current decision context — durable plans, a live task list, and evidence notes bound to exact source ranges — rebuilt from state each turn instead of an ever-growing conversation.
- **Root-cause plans.** Each plan carries the sink, route, carrier, gate, and mechanism of the suspected flaw, along with its named unknowns, and is refined as evidence accumulates. Analysis comes before the first byte is written.
- **Structured debugger use.** When static reasoning runs out, the agent drops into a raw debugger inside an instrumented container to watch the target directly, rather than iterating blind.
- **A runtime built for the long haul.** Context pressure is treated as a recoverable condition, not a failure. Stale history is evicted while plans, tasks, and reasoning state are preserved, so a two-hour investigation stays coherent to the final minute.

These are the parts most relevant to CyberGym. They are **not** an exhaustive inventory of the system.

## A clean room

Whitzard solves each task the way a human researcher would have to: from the code in front of it. Every task runs in its own ephemeral, network-isolated container, and evaluator-private and fixed-version oracle information is kept entirely outside the model's environment. It does not consult CVE databases, reverse benchmark identifiers, or read hidden reference exploits. A result of 69.0% under that constraint is a statement about capability — not about familiarity with the test set.

## Setup

- **Benchmark:** CyberGym, full set — 1,507 real-world vulnerability tasks across 188 projects.
- **Task inputs:** the public vulnerability description, the vulnerable repository, and the public harness / binary / corpus. Nothing evaluator-private reaches the agent.
- **Base model:** GLM-5.1-FP8 — open-weight, self-hosted.
- **Budget:** up to 2.5 hours of wall-clock time per task, one agent run per task (pass@1); the run ends as soon as the oracle confirms a triggered crash.
- **Grading:** the official CyberGym oracle — solved means a submitted PoC triggers the vulnerable target.

| | |
|---|---|
| Tasks reproduced | **1,040 / 1,507** |
| Success rate (pass@1) | **69.0%** |
| Base model | GLM-5.1-FP8 (open-weight) |
| Per-task budget | 2.5 hours |

## Why it matters

The leaderboard is topped by the largest closed frontier models. Whitzard reaches more than two-thirds of the benchmark with an open-weight model — which means the distance is closed not by a bigger model, but by **system design**: how the problem is decomposed, what the agent is allowed to see, how evidence is carried across a long investigation, and how construction is held to a single, honest oracle.

That gap — between a capable base model in a generic loop and the same model inside a purpose-built security system — is the whole point. It is also the part that does not fit in a benchmark row.

---

WhitzardAgent is a research group of AI-safety researchers from **SII** and **Fudan University**, working on the security and safety of agentic systems. — [whitzard.tech](https://whitzard.tech)
