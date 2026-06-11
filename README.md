# TENG_CLAW Usage Guide

This document introduces the public-facing usage of TengClaw ahead of the open-source code release. It focuses on what TengClaw does, which tools are exposed, how to read the outputs, and how to structure a typical research workflow.

Web access url: https://tactics-inspections-hotels-tigers.trycloudflare.com

Should our website be of assistance to you, kindly leave us a star.

The paper introducing the physical core and implementation methodology of this platform will soon be released on arXiv. Previous foundational theoretical work:

1. Zhao, Hongfa, et al. "Theoretical modeling of contact-separation mode triboelectric nanogenerators from initial charge distribution." Energy & Environmental Science 17.6 (2024): 2228-2247. DOI: 10.1039/d3ee04143c
2. Zhao, Hongfa, et al. "Theoretical analysis of triboelectric nanogenerators: Charge mechanisms, energy conversion, and multifunctional applications." Nano Energy (2025): 111382. DOI: 10.1016/j.nanoen.2025.111382
3. Wang, Baiqiao, et al. "Ionic-electrostatic modeling of solid-liquid triboelectric nanogenerators." Iontronics 2.2 (2026): N-A. DOI: 10.20517/iontronics.2026.009

Citations are welcome.

## 1. What TengClaw Is

TengClaw is a physics-grounded multi-agent simulation lab for TENG research.

The main workflow is:

```text
Natural-language research goal
  -> TENG-IR
  -> deterministic critic preflight
  -> Physics Critic
  -> Research Planner
  -> stable or experimental execution
  -> Experiment Graph
```

TengClaw is designed to help users:

- describe TENG research goals in natural language
- convert goals into structured simulation requests
- check physical and capability constraints before execution
- run stable TENG solvers
- compare candidate designs
- inspect time-series, field snapshots, and field animations
- preserve results through run artifacts, experiment graphs, and session memory
- understand when a request is unsupported and, in local/admin research mode, propose controlled extensions for supported extension families

In addition to conversational orchestration, TengClaw also supports a manual simulation workflow through the workspace UI. This is useful when the user already knows the target mode, geometry, material parameters, motion definition, and output type, and wants a more direct path to execution without relying on natural-language compilation.

## 2. Core Architecture

The high-level chain is:

```text
Control interface
  -> TengClaw orchestration layer
     -> Compiler
     -> critic_preflight
     -> Physics Critic
     -> Research Planner
     -> Research Reporter
     -> stable or experimental backend
  -> Python worker
     -> stable solvers
     -> capability registry
     -> experiment graph / research state
     -> experimental registry
```

The main internal roles are:

| Role | Responsibility |
|---|---|
| `Guide` | Handles the user-facing research conversation, clarification, and explanation. |
| `Compiler` | Converts a natural-language goal into strict JSON `TENG-IR`. |
| `Physics Critic` | Explains physical feasibility, missing information, approximations, and fallback options. |
| `Research Planner / Reporter` | Selects backend actions, reuses previous runs, summarizes results, and suggests next steps. |

The deterministic execution gate is:

- `critic_preflight`

It uses the capability registry as the source of truth and checks:

- unit consistency
- mode and geometry compatibility
- boundary-condition completeness
- solver support
- observable availability
- default fields that can be safely filled
- whether an unsupported request is a candidate for controlled extension

The critic verdict can be:

| Verdict | Meaning |
|---|---|
| `pass` | The request can be executed by the stable backend. |
| `clarify` | Key information is missing, so TengClaw asks a minimal follow-up question. |
| `approximate` | The original request is not fully supported, but a nearby stable path can run with explicit assumptions. |
| `unsupported` | The stable backend cannot handle the request. TengClaw may propose a controlled extension if the request matches an allowed extension family. |

## 3. Stable and Experimental Backends

### 3.1 Stable Backend

The stable backend currently supports:

- `simulate`
- `timeseries`
- `optimize`
- `field_snapshot`
- `field_animation`
- `study`
- `report`

These actions are backed by the solver and worker code in the repository.

### 3.2 Experimental Backend

The experimental backend is used only for controlled extension scenarios in local/admin research mode.

It follows three rules:

- It does not overwrite stable solvers.
- It requires proposal, confirmation, and validation.
- It is registered only after validation succeeds.

The current narrow extension family is:

- `CS/SE`
- `circle` / `mask`
- field visualization adaptor

TengClaw does not automatically invent arbitrary new physical solvers for unsupported requests.

In the public trial workspace, controlled extension requests are reported as unsupported with stable alternatives instead of generating or registering experimental modules.

## 4. Public Tools

The recommended entry point is:

- `tengclaw_orchestrate`

Use it when:

- the task is described in natural language
- some parameters are incomplete
- the task needs Physics Critic checks
- the task involves comparison, run reuse, or result interpretation
- the task should be tracked in the experiment graph or session memory

The public tools are:

| Tool | Purpose | Typical Output |
|---|---|---|
| `tengclaw_orchestrate` | Natural-language research entry point | `teng_trace`, `teng_result`, graph memory |
| `teng_simulate` | Single-state simulation | charge distribution, primary charge, snapshot report |
| `teng_timeseries` | Time-series simulation | transfer charge or electric current curve |
| `teng_optimize` | Parameter scan and optimization | best parameters, ranking, trend plot |
| `teng_field_snapshot` | Static potential or electric-field visualization | potential / field preview |
| `teng_field_animation` | Dynamic field visualization | inline video, MP4, poster, time-range summary |
| `teng_study` | Multi-candidate comparison | ranking, best candidate, run list |
| `teng_report` | Historical run retrieval | report, summary, artifact paths |

Lower-level tools are best used when:

- all parameters are already known
- strict reproduction is required
- one backend input/output needs debugging
- multi-agent orchestration is unnecessary

Internal actions such as `critic_preflight` and `extend_solver` are not intended to be called directly by ordinary users. Public trial tenants cannot run `extend_solver`.

## 5. Manual Simulation Mode

In addition to the chat-first workflow, TengClaw provides a manual simulation mode in the workspace UI through `Manual Sim`.

Manual simulation mode is recommended when:

- the simulation parameters are already fully known
- a user wants a direct and explicit configuration flow
- strict reproduction of a known setup matters
- the user wants to inspect one concrete solver path without conversational orchestration

Typical manual-simulation usage includes:

- entering a fixed geometry and material configuration
- choosing a specific stable action such as `simulate`, `timeseries`, or `field_snapshot`
- setting motion, resolution, and device parameters directly
- running a solver with minimal interpretation overhead

Compared with `tengclaw_orchestrate`:

- `tengclaw_orchestrate` is better for research exploration, incomplete requests, comparison planning, and result interpretation
- manual simulation mode is better for explicit parameter entry, controlled reproduction, and quick solver execution

A practical recommendation is:

- start with `tengclaw_orchestrate` when the research question is still open-ended
- switch to manual simulation mode once the physical setup is stable and you want repeatable runs or targeted debugging

## 6. Outputs and Research State

### 6.1 `teng_result`

`teng_result` is the result card. It typically contains:

- title and summary
- key numerical metrics
- preview image or animation entry
- report path
- raw data path
- animation path, when available

It answers the practical question: what result did this run produce?

### 6.2 `teng_trace`

`teng_trace` is the research trace. It typically contains:

- user intent
- critic verdict
- capability gap
- execution plan
- reused run ids
- next ideas
- extension status
- confirmation hint, when an extension proposal is pending

It answers the reasoning question: why did TengClaw choose this path?

### 6.3 Experiment Graph

The experiment graph records structured research nodes. A graph entry may include:

- `graph_id`
- `task_id`
- `teng_ir`
- `critic`
- `execution_plan`
- `run_ids`
- `metrics`
- `conclusion`
- `artifacts`

### 6.4 Research State

Session-level research state may include:

- recent run ids
- recent graph ids
- critic history
- pending extension plan
- experimental registry references
- recent conclusions and next actions

This lets TengClaw continue a research thread without requiring the user to restate every detail.

## 7. Key Data Structures

### 7.1 `TENG-IR`

`TENG-IR` is the structured research object compiled from the user request. Core fields include:

- `task_id`
- `intent`
- `mode`
- `geometry`
- `material`
- `motion`
- `observables`
- `objective`
- `constraints`
- `assumptions`
- `missing_info`
- `artifact_needs`
- `reuse_run_ids`
- `backend_plan`
- `support_status`
- `capability_gap`

### 7.2 `PhysicsCriticReport`

`PhysicsCriticReport` describes physical executability and risk. Core fields include:

- `verdict`
- `supported`
- `blocking_issues`
- `warnings`
- `unit_checks`
- `boundary_checks`
- `capability_checks`
- `fallback_plan`
- `extension_candidate`
- `capability_gap`
- `candidate_action`
- `source`

### 7.3 `ExperimentGraphEntry`

`ExperimentGraphEntry` is a run-level research node. Core fields include:

- `graph_id`
- `session_key`
- `task_id`
- `parent_graph_ids`
- `parent_run_ids`
- `user_goal`
- `hypothesis`
- `teng_ir`
- `critic`
- `execution_plan`
- `run_ids`
- `metrics`
- `conclusion`
- `next_actions`
- `artifacts`
- `timestamp_utc`
- `extension`
- `experimental_registry_ref`

## 8. Run Artifacts

A typical run directory contains:

```text
output/runs/<run_id>/
  |- request.normalized.json
  |- summary.json
  |- raw.json
  |- report.md
  |- plot.png / poster.png
  |- field_animation.mp4
  |- experiment_graph.json
```

Common artifact types are:

- normalized request JSON
- raw numerical output
- Markdown report
- time-series plot
- field snapshot image
- animation poster
- dynamic field animation
- per-run experiment graph entry

## 9. Example Workflow

### Step 1: Start with a Research Goal

Example prompt:

```text
Study a CS-mode TENG device. First create a runnable baseline configuration, then analyze its output current and field distribution, and tell me what should be compared next.
```

TengClaw will:

- compile the request into `TENG-IR`
- run `critic_preflight`
- generate `teng_trace`
- choose a backend plan
- execute a supported stable action when possible
- return `teng_result` when a run is produced

If the setup is already known in advance, the same study can also start from `Manual Sim` instead of chat. That path is often faster for benchmark reproduction or parameter-controlled testing.

### Step 2: Run a Single Snapshot

Use `teng_simulate` when a fully specified single-state simulation is needed.

Example request:

```text
Call teng_simulate with:
mode=CS
outputKind=charge_distribution
geometry={kind:"rectangle", n_length:40, n_width:40, b:0.01}
material={rt:0.0001, d:0.06, ebsenr:2.2}
motion={kind:"static", value:0.1}
resolution={snapshot_time:0.0}
device=cuda
```

Expected output:

- charge distribution
- primary charge
- voltage
- report artifact

### Step 3: Analyze Current over Time

Use `teng_timeseries` for transfer-charge or current curves.

Example request:

```text
Call teng_timeseries with:
mode=CS
outputKind=electric_current
geometry={kind:"rectangle", n_length:50, n_width:50, b:0.02}
material={rt:0.0001, d:0.1, ebsenr:2.2}
motion={kind:"expression", expression:"0.10-0.03*sin(2*pi*t)"}
resolution={t_start:0.0, t_end:1.0, steps:120}
device=cpu
```

Expected output:

- sampled current curve
- peak absolute current
- plot image
- report artifact

### Step 4: Optimize a Parameter

Use `teng_optimize` for parameter scanning.

Example goal:

```text
In CS mode, scan parameter d. The objective is to maximize peak current while keeping the other parameters fixed.
```

Expected output:

- best parameter value
- best metric
- candidate ranking
- trend plot or optimization summary

Recommended pattern:

- validate a baseline with `teng_timeseries`
- run a small scan with `teng_optimize`
- inspect the best candidate with `teng_field_snapshot` or `teng_field_animation`

### Step 5: Compare Multiple Candidates

Use `teng_study` for structured comparison.

Example goal:

```text
Compare two CS-mode candidate devices under the same motion condition and report which one gives the higher peak current.
```

Expected output:

- candidate ranking
- best candidate
- associated run ids
- study report

### Step 6: Inspect a Static Field

Use `teng_field_snapshot` for static potential or electric-field visualization.

Example request:

```text
Call teng_field_snapshot with:
mode=CS
geometry={kind:"rectangle", n_length:40, n_width:40, b:0.025}
material={rt:5e-09, d:0.1, ebsenr:2.2}
motion={kind:"static", value:0.1}
resolution={snapshot_time:0.0}
field={grid:{nx:150,nz:150}}
device=cuda
```

Expected output:

- potential preview
- field preview, when requested
- image artifact
- report artifact

### Step 7: Generate a Dynamic Field Animation

Use `teng_field_animation` when the field changes over time.

Example goal:

```text
Generate a dynamic potential and electric-field animation from 0 to 2 seconds, and return both the video artifact and a poster preview.
```

Expected output:

- dynamic field summary
- poster image
- MP4 animation
- report artifact

### Step 8: Read a Previous Run

Use `teng_report` to retrieve a historical result.

Example request:

```text
Call teng_report with run_id=<run_id> and return the Markdown report.
```

Use this when:

- reviewing an old run
- bringing a previous result into a new conversation
- checking the exact artifact paths

### Step 9: Continue from Session Memory

Example prompt:

```text
Continue the previous study. Keep the geometry unchanged, compare different motion frequencies, and use the recent field animation as context.
```

TengClaw can reuse:

- recent run ids
- recent graph ids
- previous critic conclusions
- previous metrics and next actions

### Step 10: Handle an Extension Proposal

For a request unsupported by the stable backend but covered by an extension family, TengClaw should propose an extension instead of silently generating new solver code.

Example request:

```text
Generate a dynamic potential and electric-field animation for a circular CS-mode device.
```

Expected behavior:

1. `critic_preflight` detects that the stable backend does not support the exact request.
2. TengClaw checks whether the request is an extension candidate.
3. If eligible, `teng_trace` includes:
   - `extensionStatus`
   - `extensionPlanId`
   - `confirmHint`
4. The user explicitly confirms before the extension is attempted.

Example confirmation:

```text
Confirm the proposed extension plan and continue.
```

## 10. Common Questions

### Why do I see a trace but no execution result?

The critic may have returned `clarify` or `unsupported`.

In that case, TengClaw either asks for missing information or explains the capability gap before execution.

### Why are some unsupported requests not automatically extended?

TengClaw is not an open-ended automatic programming system. Controlled extension is available only for narrow, validated template families.

### Which extension scenarios are currently covered?

The current extension family is limited to:

- `CS/SE`
- `circle` / `mask`
- field visualization adaptor

### When should I use `tengclaw_orchestrate` instead of a lower-level tool?

Use `tengclaw_orchestrate` for research tasks, comparison, exploration, incomplete parameters, or interpretation.

Use a lower-level tool for strict reproduction, fully specified inputs, or backend debugging.

## 11. Contact information
If there is any question or you need technical support, plaease contact: zhaoh1040@163.com; 2493620867@qq.com.

## 12. Third-party notices

TengClaw is built on top of OpenClaw.

- Upstream: https://github.com/openclaw/openclaw
- License: MIT
- Copyright: Copyright (c) 2026 OpenClaw Foundation

MIT License

Copyright (c) 2026 OpenClaw Foundation

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

```bibtex
@misc{openclaw,
  title        = {OpenClaw},
  author       = {OpenClaw contributors},
  howpublished = {\url{https://github.com/openclaw/openclaw}}
}
```

