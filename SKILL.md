---
name: lean-mode
description: High-performance token-efficient multi-agent orchestration and lean execution. Uses a persistent /project-plan.md as the Single Source of Truth, deterministic Scope Classification (Epic -> Checkpoint -> Task), single-checkpoint DAG execution, dynamic tool registry pruning, peer-to-peer worker coordination, and strict proof contracts to eliminate multi-agent token tax and prevent AI slop. Use when coordinating multiple subagents, running token-constrained tasks, optimizing agent team workflows, or applying lean execution patterns.
version: 1.2.0
license: MIT
---

# Lean Mode Skill

Lean Mode is a token-efficient orchestration and execution framework designed to eliminate multi-agent coordination bloat and prevent AI slop. By maintaining `/project-plan.md` as the persistent **Single Source of Truth (SSOT)** and combining pre-flight **Scope Classification** (Epic $\to$ Checkpoint $\to$ Task) with single-checkpoint DAG execution, peer-to-peer worker coordination, dynamic tool registry pruning, and strict automated proof contracts, Lean Mode keeps repositories continuously in a verifiable, releasable state.

---

## 1. Project State (Single Source of Truth)

Lean Mode maintains a `/project-plan.md` file in the project root.

This file is the single source of truth for:
- Epics
- Checkpoints
- Tasks
- Current focus
- Completion status
- Execution history

If the file is missing, Lean Mode must create it before planning or executing any work. Every completed checkpoint must immediately update the file.

> [!CRITICAL]
> **State Authority Invariant**: Agents must never infer project state from chat history, internal model memory, or subagent transcripts when `/project-plan.md` exists. The filesystem is the only source of truth.

### Canonical `/project-plan.md` Contract

```markdown
# Project Plan

## Project
Name: Lean Mode Benchmark
Status: Active

## Epics

- [ ] EP1 Benchmark Dataset
- [ ] EP2 Runner
- [ ] EP3 Telemetry
- [ ] EP4 Reports
- [ ] EP5 History

---

## EP1 Benchmark Dataset

Status: In Progress

### Checkpoints

- [x] CP-01 Repository Manifest
- [ ] CP-02 Task Specification
- [ ] CP-03 Lite Dataset
- [ ] CP-04 Dataset Validator
- [ ] CP-05 Dataset Loader

---

## Current Focus

Epic: EP1
Checkpoint: CP-02
Task:
Define task.yaml schema

---

## Completed Log

- 2026-09-26 CP-01 completed
```

### Automatic State Update Rules

Lean Mode maintains and updates `/project-plan.md` automatically without asking for user confirmation:

| Event | Action on `/project-plan.md` |
| :--- | :--- |
| **New Project / Missing File** | Create `/project-plan.md` with Project name and initial Status. |
| **Epics Classified** | Write `EP1...EPn` list in `## Epics`. |
| **Epic Started** | Create section `## EPx <Name>`, set `Status: In Progress`, list Checkpoints. |
| **Checkpoint Started** | Update `## Current Focus` with the active Epic, Checkpoint ID, and Task. |
| **Checkpoint Completed** | Mark Checkpoint `[x]`, append timestamped entry to `## Completed Log`. |
| **Epic Completed** | Mark Epic `[x]` in `## Epics`, set Epic `Status: Completed`. |
| **New Requirement Discovered** | Insert new Epic or Checkpoint into the appropriate section. |

> **The Sync-Execute-Sync Protocol**:  
> Before any response that modifies the project, Lean Mode first synchronizes `project-plan.md`, executes the work, and then synchronizes `project-plan.md` again upon verification.

---

## 2. Scope Classification

Before planning or compiling increments, the Orchestrator classifies the request using a deterministic three-tier hierarchy: **Epic**, **Checkpoint**, or **Task**.

### The Three-Tier Hierarchy

```
[ Request ]
    │
    ├── Multiple independent subsystems? ──► [ Epic ] ──► Sequential/Isolated Checkpoint lists
    │
    ├── Single subsystem, 2–10 mergeable steps? ──► [ Checkpoint ] ──► Atomic incremental sequence (CP-01, CP-02...)
    │
    └── Single atomic change, 1 merge commit? ──► [ Task ] ──► Direct single-agent execution with local Proof Contract
```

### Classification Rules & Structural Complexity Matrix

Instead of estimating arbitrary lines of code, Lean Mode measures structural complexity:

| Attribute | Task | Checkpoint | Epic |
| :--- | :---: | :---: | :---: |
| **Independent Subsystems** | 1 | 1 | 2+ |
| **Merge Commits** | 1 | 2–10 | 10+ |
| **Proof Contracts** | 1 | Several (1 per CP) | Dozens |
| **Parallelizable** | No | Rarely | Yes |
| **Autonomous Value** | No (atomic diff) | No (intermediate increment) | Yes (standalone domain) |

- **Epic Criteria**:
  - Touches 2+ independent subsystems with distinct domains of responsibility (e.g., Benchmark = Dataset, Runner, Telemetry, Reports).
  - Subsystems could be assigned to separate teams with no shared merge contract.
  - *Action*: Record Epics (`EP1`, `EP2`...) in `project-plan.md`. Process Epics sequentially or with isolated orchestrators.
- **Checkpoint Criteria**:
  - Confined to a single subsystem or module, but requires multiple independently verifiable and revertable increments (2–10 merges).
  - *Action*: Compile into an ordered Checkpoint sequence (`CP-01`, `CP-02`...) and record under the active Epic in `project-plan.md`.
- **Task Criteria**:
  - The change is already atomic (e.g., fix HTTP 404 handler, update a config field, add a helper function).
  - Involves 1 subsystem, 1 merge commit, and 1 Proof Contract.
  - *Action*: Execute directly as a single-agent Task without Checkpoint compilation ceremony or DAG overhead.

> [!IMPORTANT]
> **Triage Invariant**: Never compile Checkpoints before determining whether Epics exist. Never create artificial Checkpoints (`CP-01`) for requests that are already atomic Tasks.

---

## 3. Checkpoint Compilation

When a request or active Epic requires multiple mergeable increments, the Orchestrator compiles the work into an ordered sequence of Checkpoints.

### Checkpoint Definition

A **Checkpoint** is the smallest independently verifiable engineering change.

Every Checkpoint must satisfy all conditions:
- **One Logical Objective**: Exactly one clear intent with a single dominant concern.
- **One Behavioral Change**: Exactly one new capability, refactor, or bugfix.
- **Bounded Scope**: Touches $\le 3$ closely related files when practical.
- **Independently Reviewable**: Can be understood and inspected in minutes.
- **Independently Revertable**: Reverting the Checkpoint leaves adjacent components functional.
- **Automatically Verifiable**: Accompanied by deterministic automated proof before completion.

A Checkpoint completes only after its **Proof Contract** passes (`exit 0`).

```
Examples of Valid Checkpoints:
  ✓ Extract TimelineLayout calculation
  ✓ Replace Login Button component
  ✓ Add Event Repository with unit tests

Not Checkpoints:
  ✗ Authentication
  ✗ Timeline Screen
  ✗ User Profile
```

### Decomposition Rules & The "And" Heuristic

Split work until every step satisfies the atomic Checkpoint criteria.
> **The "And" Heuristic**:  
> If a Checkpoint's title or objective naturally requires the word **"and"** (e.g., *"Create UserStore and update LoginView"*), it must be split into two distinct Checkpoints.

### Compilation Algorithm

```
Subsystem / Epic Scope
      │
      ▼
Analyze AST & Dependency Bounds
      │
      ▼
Split into Atomic Checkpoints (CPs)
      │
      ▼
Assign Sequential Identifiers (CP-01, CP-02...)
      │
      ▼
Define Proof Contract & Touched Files for Each CP
      │
      ▼
Select CP-01 & Record in project-plan.md
```

*The compiler produces only structure and verification contracts—it never writes code during compilation.*

### Canonical Checkpoint Specification (YAML)

Compile Checkpoints into this structure before registering them in `project-plan.md`:

```yaml
id: CP-01
title: Extract TimelineLayout

objective:
  Separate layout calculation from rendering.

depends_on: []

touches:
  - src/timeline/layout.ts
  - src/timeline/view.tsx

proof:
  - unit_tests: "npm test -- test/timeline_layout.spec.ts"
  - snapshot: "npm run test:visual -- timeline"
  - typecheck: "npx tsc --noEmit"

done_when:
  - public API unchanged
  - all proofs pass (exit 0)
```

---

## 4. Orchestration Execution Lifecycle

Replace global project orchestration with single-checkpoint execution loops:

```
1. Synchronize Project State: Read or initialize /project-plan.md.
2. Classify Scope: Determine tier (Task, Checkpoint sequence, or Epics).
   - If Task: Execute directly as single agent with local Proof Contract (exit 0). Log and finish.
   - If Epics: Record Epics in project-plan.md; select active Epic.
3. Compile Checkpoints (CP sequence) for the active Epic and record in project-plan.md.
4. Set Focus: Update ## Current Focus in project-plan.md to active CP.
5. Evaluate Pre-Flight Topology Decision Gate for that CP.
6. Build DAG ONLY for this active CP.
7. Spawn pruned workers (or execute directly if single-agent).
8. Execute implementation.
9. Verify Proof Contract (exit 0).
10. Post-Sync State: Mark CP complete [x], log in ## Completed Log, merge baseline, repeat for next CP.
```

> [!IMPORTANT]
> **Single-Checkpoint DAG Invariant**: The execution DAG exists **strictly for one active Checkpoint at a time**. The Orchestrator never holds the entire project in mind at once, preventing context bloat and hallucinated dependencies.

### Pre-Flight Topology Decision Gate

For the active Checkpoint, evaluate execution topology:

- **Single-Agent Lean Monolith**:
  - **Condition**: Checkpoint touches $\le 3$ files, involves sequential dependencies, or modifies a single component.
  - **Action**: Execute directly as a single agent. Spawning subagents on small or sequential tasks introduces a 2.35x–3.08x token penalty without parallel acceleration.
- **Multi-Agent Lean DAG**:
  - **Condition**: Checkpoint involves 2+ genuinely independent concurrent modules OR requires an adversarial verification gate.
  - **Action**: Proceed with Orchestrator DAG decomposition and pruned subagent dispatch.

---

## 5. Multi-Agent Execution Mechanics

When multi-agent execution is selected for an active Checkpoint:
1. **Dynamic Tool Registry Pruning**: Subagents receive strictly pruned tool definitions via `define_subagent`, restricting schema exposure to role-essential tools.
2. **Peer-to-Peer Direct Communication**: Implementers communicate directly via `send_message` without routing intermediate state through the Orchestrator.
3. **Structured Wire Protocol**: All inter-agent messages use compact JSON payloads tagged with the active `cp_id`.
4. **Concise Technical Register**: Communicate directly with minimal prose overhead, keeping inter-agent exchanges focused on status and deliverables.
5. **Short-Circuit TDD Gate**: Automated test execution precedes final sign-off (`exit 0` required; fast-failing flags `-q --tb=short`).

---

## 6. Standard Wire Protocol Schema

Format all inter-agent messages sent via `send_message` using this JSON contract:

```json
{
  "action": "TASK | HANDOFF | RESULT | SYNC | REJECT | ACK",
  "cp_id": "CP-01",
  "in_scope": ["path/to/file.ext"],
  "status": "IN_PROGRESS | COMPLETED | FAILED",
  "exit_code": 0,
  "payload": "Concise technical summary."
}
```

Detailed role specifications, allowed tools, and protocol semantics are defined in [`roles.md`](roles.md).

---

## 7. Specifications & Reference Architecture

- [`roles.md`](roles.md) - Canonical role contracts (Orchestrator, Implementer, Verifier), tool allocations, and wire protocol semantics.
- [Project Plan Contract](references/project-plan-contract.md) - Single Source of Truth specification, format contract, and automatic update rules.
- [Scope Classification & Checkpoint Compilation](references/checkpoint-compilation.md) - Three-tier hierarchy, structural complexity matrix, and YAML schemas.
- [DAG Orchestration & Lifecycle](references/dag-orchestration.md) - Single-checkpoint execution loop, topology gate, and P2P coordination.
- [The Proof Contract](references/proof-contract.md) - Deterministic verification categories, fast-failing flags, and zero-tolerance gates.
- [Context & Tool Registry Pruning](references/context-pruning.md) - Dynamic schema reduction, pre-flight context bounding, and empirical Decision Gate evidence.

---

## 8. Operational Constraints & Safety

- **Project State Invariant**: Maintain `/project-plan.md` in the project root as the single source of truth. Never infer project state from chat history or internal model memory when `/project-plan.md` exists. Always synchronize before and after execution.
- **Scope Classification Invariant**: Classify every request into Task, Checkpoint sequence, or Epics before constructing execution plans or spawning workers.
- **Pre-Execution Checkpoint Compilation**: For multi-step workloads, compile atomic Checkpoints before constructing an execution DAG or dispatching workers.
- **Single Active Checkpoint Invariant**: Maintain an execution DAG only for the currently active Checkpoint. Never construct a multi-agent DAG for the entire request at once.
- **Zero-Tolerance Proof Gate**: No Checkpoint or Task is marked complete without passing automated test and verification contracts (`exit 0`).
- **Pruned Tool Enforcement**: Grant subagents only the tools essential for their assigned role as specified in [`roles.md`](roles.md).
- **Concise Communication**: Keep all inter-agent messages and progress updates focused on structured deliverables and test outcomes.
- **Workspace Boundary**: Restrict all operations strictly to files within the active workspace root.
