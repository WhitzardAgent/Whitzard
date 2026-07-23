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
description + vulnerable source + harness
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

## Agent framework and capability surface

Whitzard is implemented on [QitOS](https://github.com/WhitzardAgent/qitos), our agent framework for building, running, and inspecting long-horizon agent trajectories. QitOS provides the execution kernel, typed tool contracts, state reduction, context assembly, and structured event tracing. Whitzard adds the CyberGym-specific task model, evidence discipline, candidate provenance, and verification policy.

The evaluation agent is a single task-level agent with phase-aware capability control. It keeps the task's current plan, open questions, evidence, candidate history, and oracle receipts as explicit state. Each tool result contributes both a compact decision-facing summary and a structured trace record, giving later decisions a stable view of what has been observed.

The task-specific tool surface includes:

- **Source and input inspection:** `READ`, `GLOB`, `GREP`, `HexView`, and `StructProbe` locate relevant code, examine byte-level inputs, and test structural hypotheses.
- **Workspace and execution:** `BASH`, `WRITE`, and `EDIT` support controlled input construction, local inspection, and task-scoped execution.
- **Decision state:** `NOTE`, `TODO_*`, and `SWITCH` preserve evidence-backed conclusions, pending questions, and stage transitions.
- **Dynamic diagnosis and verification:** `gdb_debug` supports narrowly scoped runtime checks; `submit_poc` records the benchmark oracle verdict for an attributable candidate.

Tool availability depends on the task stage and evaluation environment. This repository documents the public design and observed behavior. Task-specific prompts, policy thresholds, execution parameters, and infrastructure configuration are intentionally omitted.

## Core design

### Evidence is the control plane

Whitzard treats the oracle as the only completion authority. Source reads, shell output, debugger stops, sanitizer diagnostics, and model confidence are all useful diagnostic evidence, but none of them is a solved result. A task closes only after a submitted input satisfies the vulnerable-trigger and fixed-clean checks.

### Durable decision state

The agent works from compact, durable decision state: an explicit plan, active questions, task list, experiment receipts, and source-bound evidence notes. This state is rebuilt into the decision context each turn. Older conversational residue can be removed without losing the causal claims that matter for the next experiment.

### Phase-aware investigation loop

The runtime separates evidence gathering, candidate construction, and re-investigation into explicit modes. Each mode exposes the tools appropriate for its immediate objective and carries forward the same decision state. A candidate submission must state the change being tested, which ties the oracle receipt back to the underlying hypothesis and makes failed attempts useful for the next decision.

### Typed tool receipts and trajectory review

Every tool call has a machine-readable result, a deterministic model-facing summary, and an event record in the trajectory. QitOS uses these records to retain inspectable long-horizon behavior while Whitzard uses them to update plans, evidence, and candidate state. The released traces expose this behavior at the artifact level without publishing the private operational configuration used to produce it.

### Root-cause plans before byte construction

A plan names the suspected sink, harness route, input carrier, gate, mechanism, and remaining unknowns. A source-backed working theory is enough to begin testing when it names the narrowest unresolved question that the next observation or candidate input can separate. This turns broad code exploration into a sequence of falsifiable constraints.

### Construction as discriminating experiments

Whitzard preserves a parseable carrier, changes one meaningful relation at a time, and records the result of each submitted candidate. After a `no_trigger`, the next action must distinguish a concrete failure mode—carrier, selector, route, gate, or mechanism—instead of producing another unexplained mutation.

### Structured dynamic analysis

Static reasoning is complemented by tightly scoped runtime observation. The agent can inspect a precise runtime value or stop point in an instrumented, task-isolated environment, then convert the conclusion into durable state. Debugging is diagnostic; it never substitutes for the submission oracle.

### Long-horizon reliability

Long tasks fail easily when context, retries, or tool output become the implicit controller. Whitzard makes these states explicit: context pressure is recoverable, plans and task state persist, and tool results have deterministic summaries alongside typed machine-readable output. This keeps a two-hour investigation coherent through its final decision.

## Experimental setting

This section follows CyberGym's [FAQ disclosure guidance](https://github.com/sunblaze-ucb/cybergym/blob/main/FAQ.md) for agent scaffold, dynamic execution, network access, and verification.

### Agent scaffold

- **Framework and agent:** QitOS provides the runtime kernel; Whitzard provides a single task-level CyberGym agent with phase-aware tool access, explicit decision state, evidence reduction, and oracle-bound candidate tracking.
- **Model:** GLM-5.1-FP8.
- **Time budget:** each task has a finite wall-clock budget of up to **2.5 hours**.
- **Tooling:** the agent uses the task-scoped inspection, workspace, state, diagnostic, and oracle tools described above. Tool calls and their structured results are recorded in the trajectory.

### Task inputs and dynamic environment

- **Level 1 inputs:** each task supplies a vulnerability description and the pre-patch source code, together with task-relevant harness, binary, and corpus material when available.
- **Dynamic execution:** enabled. Each task runs in its own Docker container with the public vulnerable target available for execution and debugging. `BASH` and `gdb_debug` operate within that task container; `gdb` and `rg` are available.
- **Leakage boundary:** the agent receives no patched image, patch diff, evaluator-private material, grader credentials, repository Git history, or reference PoC. The fixed image is reserved for post-run validation.

### Network access and trajectory audit

- **Egress policy:** disabled at the container boundary (`network_mode=none`). No domain allowlist, proxy, browser, or external-search tool is available to the task agent.
- **Observed behavior:** a process can still attempt an outbound connection when egress is disabled. We audited the executed `BASH` actions in all **1,038** packaged solved traces. Seven actions explicitly attempted public GitHub or crates.io access; none obtained external content or completed a public connection. Among 59 dependency-acquisition commands, 22 record DNS-resolution failure and 16 stop at a permission failure. The only confirmed successful network-like interaction was a task-local TLS loopback connection to `127.0.0.1`.

### Verification and result accounting

- **During a task:** `submit_poc` evaluates an attributable candidate against the vulnerable target. The agent never receives fixed-version execution feedback.
- **Post-run validation:** each reported solved artifact is independently checked through the vulnerable-then-fixed protocol. A task enters the fixed-clean solved set when at least one candidate produced in its packaged trajectory triggers the vulnerable target and remains clean on the fixed target; the package retains one such PoC.
- **Trace coverage:** the released package contains one compact trajectory for each of the **1,507** tasks.

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
