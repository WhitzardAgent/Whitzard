<h1>Whitzard <span lang="zh-Hans">（白泽）</span></h1>

<p><em>An evidence-driven agentic system for real-world vulnerability reproduction.</em></p>

<p><strong>English</strong> · <a href="./README.zh-CN.md">中文</a></p>

---

## 68.9% verified reproduction on CyberGym Level 1

Whitzard (白泽) produces **1,038 fixed-clean PoCs out of 1,507** real-world vulnerability tasks on [CyberGym Level 1](https://arxiv.org/abs/2506.02548): a **68.9% verified reproduction rate**. Each reported solution contains a PoC that triggers the vulnerable target and remains clean on the fixed target.

CyberGym Level 1 supplies a vulnerability description and the pre-patch codebase. The agent must bridge the gap from that evidence to concrete input bytes that reproduce the vulnerability. The benchmark evaluates construction, execution, and verification.

> In Chinese legend, 白泽 (Bái Zé) knows the many forms of things that go wrong. It recognizes them through knowledge. That is the standard we set for an agent: preserve evidence, explain the route, and let the oracle decide.

## Benchmark

CyberGym Level 1 measures the full loop of automated vulnerability reproduction:

```text
description + vulnerable source + public harness
  -> source-backed trigger hypothesis
  -> attributable candidate input
  -> vulnerable trigger + fixed-clean verification
```

The benchmark contains **1,507** tasks across **188** open-source projects. The submission package covers every task exactly once:

| Category | Tasks | Artifact policy |
|---|---:|---|
| Fixed-clean solved | **1,038** | One independently verified final PoC and one compact trace |
| Timeout | 418 | Trace only; no PoC is claimed |
| Triggered, unresolved | 50 | Trace plus one final triggered PoC |
| Unrecoverable execution error | 1 | Trace only |

## Base model

This submission uses **GLM-5.1-FP8**. The base model supplies code understanding, tool-feedback absorption, and long-horizon reasoning. Whitzard supplies the operating discipline around it: what constitutes evidence, what survives context pressure, and when an input is allowed to count as a solution.

## Core design

### Evidence is the control plane

Whitzard treats the oracle as the only completion authority. Source reads, shell output, debugger stops, sanitizer diagnostics, and model confidence are all useful diagnostic evidence, but none of them is a solved result. A task closes only after a submitted input satisfies the vulnerable-trigger and fixed-clean checks.

### Durable decision state

The agent works from compact, durable decision state: an explicit plan, active questions, task list, experiment receipts, and source-bound evidence notes. This state is rebuilt into the decision context each turn. Older conversational residue can be removed without losing the causal claims that matter for the next experiment.

### Root-cause plans before byte construction

A plan names the suspected sink, harness route, input carrier, gate, mechanism, and remaining unknowns. A source-backed working theory is enough to begin testing when it names the narrowest unresolved question that the next observation or candidate input can separate. This turns broad code exploration into a sequence of falsifiable constraints.

### Construction as discriminating experiments

Whitzard preserves a parseable carrier, changes one meaningful relation at a time, and records the result of each submitted candidate. After a `no_trigger`, the next action must distinguish a concrete failure mode—carrier, selector, route, gate, or mechanism—instead of producing another unexplained mutation.

### Structured dynamic analysis

Static reasoning is complemented by tightly scoped runtime observation. The agent can inspect a precise runtime value or stop point in an instrumented, task-isolated environment, then convert the conclusion into durable state. Debugging is diagnostic; it never substitutes for the submission oracle.

### Long-horizon reliability

Long tasks fail easily when context, retries, or tool output become the implicit controller. Whitzard makes these states explicit: context pressure is recoverable, plans and task state persist, and tool results have deterministic summaries alongside typed machine-readable output. This keeps a two-hour investigation coherent through its final decision.

## Evaluation setting

- **Task inputs:** the public vulnerability description, vulnerable codebase, and public harness/binary/corpus.
- **Isolation:** each task runs in a task-exclusive, network-isolated container. Evaluator-private material and fixed-version oracle information are not exposed to the agent.
- **Time:** a finite wall-clock budget of up to **2.5 hours** per task.
- **Verification:** reported solved artifacts are checked through the vulnerable-then-fixed protocol.

## Main result

| Metric | Value |
|---|---:|
| Verified fixed-clean PoCs | **1,038 / 1,507** |
| Verified reproduction rate | **68.9%** |
| Base model | GLM-5.1-FP8 |
| Trace coverage | **1,507 / 1,507** tasks |

## Statistics

### Solved-task trace duration

The distribution below covers the **1,038 fixed-clean solved tasks**. Duration is measured from the first to the last timestamped runtime event in each packaged trace.

| Duration | Tasks | Share |
|---|---:|---:|
| <10 min | 368 | 35.45% |
| 10–30 min | 367 | 35.36% |
| 30–60 min | 212 | 20.42% |
| 1–2 h | 91 | 8.77% |

Median solved-task duration is **16.4 minutes**; the 75th percentile is **33.9 minutes** and the 95th percentile is **80.7 minutes**. A solved task uses a median of **45 agent decision steps** (mean: **57.2**).

### Resource consumption

| Metric | Value |
|---|---:|
| SGLang-accounted model tokens | **11,140,976,522** |
| Prompt tokens | 11,079,987,901 |
| Completion tokens | 60,988,621 |
| Model requests | 134,286 |
| Agent decision steps | 134,282 |
| Solved-task model tokens | 3,923,353,416 |
| Median solved-task model tokens | 1,999,829 |

Token values come from serving-host aggregation across all **1,507** packaged traces when the runtime records `meter_source=sglang_usage`; coverage is **1,507 / 1,507**. “Agent decision steps” are the per-task `step_count` recorded by the compact trace footer, and the model-request count is the number of recorded model-output events.

## Why it matters

A capable open-weight security agent maintains an input contract, a reachable code path, a fault mechanism, and a verified input simultaneously—often across many failed experiments. Whitzard is designed to make that evidence accumulate across turns.

The result is a benchmark measurement and evidence that carefully designed state, interfaces, and verification discipline can substantially change the useful security behavior of a capable open-weight model.

## Limitations

- A fixed-clean PoC benchmark evaluates vulnerability reproduction; complete security auditing and remediation require additional methods.
- **469 / 1,507** tasks remain outside the fixed-clean solved set; timeout and triggered-unresolved traces are retained for analysis and excluded from the solved count.
- The current system is computationally intensive: the submitted traces account for more than 11 billion model tokens.

---

WhitzardAgent is a research group from **Fudan University**, **Shanghai Innovation Institute**, and **Shanghai Pudong Research Institute of Cryptology**, working on the security and safety of agentic systems. — [whitzard.tech](https://whitzard.tech)
