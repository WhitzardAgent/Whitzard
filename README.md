<h1>Whitzard <span lang="zh-Hans">（白泽）</span></h1>

<p><em>An evidence-driven agentic system for real-world vulnerability reproduction.</em></p>

<p><strong>English</strong> · <a href="./README.zh-CN.md">中文</a></p>

---

## 91.2% verified reproduction on CyberGym Level 1

Whitzard (白泽) produces **1,374 fixed-clean PoCs out of 1,507** real-world vulnerability tasks on [CyberGym Level 1](https://arxiv.org/abs/2506.02548): a **91.2% verified reproduction rate**. Each reported solution contains a PoC that triggers the vulnerable target and remains clean on the fixed target.

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
| Fixed-clean solved | **1,374** | One independently verified final PoC and one compact trace |
| Timeout | 32 | Trace only; no PoC is claimed |
| Triggered, unresolved | 94 | Trace plus one final triggered PoC when available |
| Unrecoverable execution error | 7 | Trace or task placeholder only; no PoC is claimed |

## Base model and reachability-guided fuzzing

This submission replaces the previous **GLM-5.1-FP8** base with **DeepSeek-V4-Flash-0731**. The base model supplies code understanding, tool-feedback absorption, and long-horizon reasoning. Whitzard supplies the operating discipline around it: what constitutes evidence, what survives context pressure, and when an input is allowed to count as a solution.

The largest system-level improvement in this round is the addition of **`fuzz_witness`**, a task-local reachability-witness tool. The agent names a function or `file:line` breakpoint and provides a corpus of seed inputs; the tool drives the staged libFuzzer target under `gdb` until some mutated input reaches that location. When the breakpoint fires, the tool injects a synthetic abort so libFuzzer saves the current input, then stages that input under `pocs/` as a witness.

This distinction is important: a `fuzz_witness` artifact is not reported as a crash and is not treated as proof that the vulnerability triggers. It is an input proven to reach a model-selected code location. The agent uses it as an intermediate construction object—seed it into another, deeper reachability run, inspect it, or modify it toward the branch or gate required by the task. `submit_poc` remains the only benchmark verdict, and fixed-side behavior is still withheld from the task agent.

## Agent framework and capability surface

Whitzard is implemented on [QitOS](https://github.com/WhitzardAgent/qitos), our agent framework for building, running, and inspecting long-horizon agent trajectories. QitOS provides the execution kernel, typed tool contracts, state reduction, context assembly, and structured event tracing. Whitzard adds the CyberGym-specific task model, evidence discipline, candidate provenance, and verification policy.

The evaluation agent is a single task-level agent with phase-aware capability control. It keeps the task's current plan, open questions, evidence, candidate history, and oracle receipts as explicit state. Each tool result contributes both a compact decision-facing summary and a structured trace record, giving later decisions a stable view of what has been observed.

The task-specific tool surface includes:

- **Source and input inspection:** `READ`, `GLOB`, `GREP`, `HexView`, and `StructProbe` locate relevant code, examine byte-level inputs, and test structural hypotheses.
- **Workspace and execution:** `BASH`, `WRITE`, and `EDIT` support controlled input construction, local inspection, and task-scoped execution.
- **Decision state:** `NOTE`, `TODO_*`, and `SWITCH` preserve evidence-backed conclusions, pending questions, and stage transitions.
- **Reachability-guided fuzzing:** `fuzz_witness` combines libFuzzer and `gdb` to produce staged witness inputs that reach model-selected functions or `file:line` locations.
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

`fuzz_witness` strengthens this loop when hand-built inputs cannot yet reach the relevant branch. A shallow witness can be used as a seed for a deeper breakpoint, or inspected and edited by hand. The tool deliberately separates reachability from vulnerability reproduction: reaching a location is useful evidence, but only the later `submit_poc` receipt can turn a candidate into a solved result.

### Structured dynamic analysis

Static reasoning is complemented by tightly scoped runtime observation. The agent can inspect a precise runtime value or stop point in an instrumented, task-isolated environment, then convert the conclusion into durable state. Debugging is diagnostic; it never substitutes for the submission oracle.

### Long-horizon reliability

Long tasks fail easily when context, retries, or tool output become the implicit controller. Whitzard makes these states explicit: context pressure is recoverable, plans and task state persist, and tool results have deterministic summaries alongside typed machine-readable output. This keeps a two-hour investigation coherent through its final decision.

## Experimental setting

This section follows CyberGym's [FAQ disclosure guidance](https://github.com/sunblaze-ucb/cybergym/blob/main/FAQ.md) for agent scaffold, dynamic execution, network access, and verification.

### Agent scaffold

- **Framework and agent:** QitOS provides the runtime kernel; Whitzard provides a single task-level CyberGym agent with phase-aware tool access, explicit decision state, evidence reduction, and oracle-bound candidate tracking.
- **Model:** DeepSeek-V4-Flash-0731.
- **Time budget:** each task has a finite wall-clock budget of up to **4 hours**.
- **Tooling:** the agent uses the task-scoped inspection, reachability-guided fuzzing, workspace, state, diagnostic, and oracle tools described above. Tool calls and their structured results are recorded in the trajectory.

### Task inputs and dynamic environment

- **Level 1 inputs:** each task supplies a vulnerability description and the pre-patch source code, together with task-relevant harness, binary, and corpus material when available.
- **Dynamic execution:** enabled. We provide a Docker image for each task, allowing the agent to test and debug dynamically. The image is built on top of the official dataset base image with `gdb` installed, and the official vulnerable binary is mounted read-only into the task container. `BASH` and `gdb_debug` operate within that container, and `rg` is also available.
- **Leakage boundary:** the agent runs in the base image with only the vulnerable binary mounted read-only, not the full vulnerable image, so leakage sources that live inside that image (in particular the reference PoC at `/tmp/poc` and the Git history under `/src/**/.git`) are not present in the agent's environment. The agent also receives no patched image, patch diff, evaluator-private material, grader credentials, or reference PoC; the fixed image is reserved for post-run validation.

### Network access and trajectory audit

- **Egress policy:** disabled at the container boundary (`network_mode=none`). No domain allowlist, proxy, browser, or external-search tool is available to the task agent.
- **Observed behavior:** a process can still attempt an outbound connection when egress is disabled, but no external-search or browsing capability is exposed to the task agent. Candidate construction, including `fuzz_witness`, runs inside the task-local environment.

### Verification and result accounting

- **During a task:** `submit_poc` evaluates an attributable candidate against the vulnerable target. The agent never receives fixed-version execution feedback.
- **Post-run validation:** each reported solved artifact is independently checked through the vulnerable-then-fixed protocol. A task enters the fixed-clean solved set when at least one candidate produced in its packaged trajectory triggers the vulnerable target and remains clean on the fixed target; the package retains one such PoC.
- **Artifact coverage:** the released package covers all **1,507** tasks. It contains **1,504** compact trajectories and **3** unsolved task placeholders without a claimed PoC.

## Main result

| Metric | Value |
|---|---:|
| Verified fixed-clean PoCs | **1,374 / 1,507** |
| Verified reproduction rate | **91.2%** |
| Base model | DeepSeek-V4-Flash-0731 |
| Task coverage | **1,507 / 1,507** tasks |
| Compact trace artifacts | **1,504 / 1,507** tasks |

## Statistics

### Submission artifact accounting

| Metric | Value |
|---|---:|
| Fixed-clean solved tasks | **1,374** |
| Unsolved tasks | 133 |
| Timeout tasks | 32 |
| Triggered but not fixed-clean | 94 |
| Unrecoverable execution error | 7 |
| Compact trace artifacts | 1,504 |

### Resource consumption

| Metric | Value |
|---|---:|
| Accounted model tokens | **19,397,802,493** |
| Input tokens | 19,265,042,892 |
| Output tokens | 132,759,601 |
| Model requests / agent decision events | 115,661 |
| Estimated wall-clock task time | 3,025,222 sec / 840.3 h |
| Estimated cache-read tokens | 18,687,091,605 |
| Estimated cache-creation tokens | 577,951,287 |
| Estimated serving cost | 5,100 CNY / ~708 USD |

Token and timing values are aggregated from the available original full traces for the final package. Cache-read and cache-creation tokens use the serving estimate that approximately **97%** of prompt tokens were cache reads.

## Why it matters

A capable open-weight security agent maintains an input contract, a reachable code path, a fault mechanism, and a verified input simultaneously—often across many failed experiments. Whitzard is designed to make that evidence accumulate across turns.

The result is a benchmark measurement and evidence that carefully designed state, interfaces, and verification discipline can substantially change the useful security behavior of a capable open-weight model.

## Limitations

- A fixed-clean PoC benchmark evaluates vulnerability reproduction; complete security auditing and remediation require additional methods.
- **133 / 1,507** tasks remain outside the fixed-clean solved set; timeout, triggered-unresolved, and execution-error artifacts are excluded from the solved count.
- The current system is computationally intensive: the submitted traces account for approximately **19.4 billion** model tokens.

---

WhitzardAgent is a research group from **Fudan University**, **NUWA Lab**, **Shanghai Innovation Institute**, and **Shanghai Pudong Research Institute of Cryptology**, working on the security and safety of agentic systems. — [whitzard.tech](https://whitzard.tech)
