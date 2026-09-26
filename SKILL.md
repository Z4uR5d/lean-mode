---
name: lean-mode
description: High-performance token-efficient multi-agent orchestration and lean execution. Uses mandatory Checkpoint Compilation, single-checkpoint DAG execution, dynamic tool registry pruning, peer-to-peer worker coordination, and strict proof contracts to eliminate multi-agent token tax and prevent AI slop. Use when coordinating multiple subagents, running token-constrained tasks, optimizing agent team workflows, or applying lean execution patterns.
version: 1.0.0
license: MIT
---

# Lean Mode Skill

Lean Mode is a token-efficient orchestration and execution framework designed to eliminate multi-agent coordination bloat and prevent AI slop. By combining pre-flight **Checkpoint Compilation** with single-checkpoint DAG execution, peer-to-peer worker coordination, dynamic tool registry pruning, and strict automated proof contracts, Lean Mode keeps repositories continuously in a verifiable, releasable state.

---

## 1. Checkpoint Compilation (MANDATORY)

Before creating an execution DAG or spawning any worker, the Orchestrator **must compile the request into an ordered sequence of Checkpoints**.

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
Input Request
      ↓
Analyze AST & Dependency Bounds
      ↓
Split into Atomic Checkpoints (CPs)
      ↓
Assign Sequential Identifiers (CP-01, CP-02...)
      ↓
Define Proof Contract & Touched Files for Each CP
      ↓
Select CP-01
```

*The compiler produces only structure and verification contracts—it never writes code during compilation.*

### Canonical Checkpoint Specification (YAML)

Compile Checkpoints into this structure:

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

## 2. Orchestration Execution Lifecycle

Replace global project orchestration with single-checkpoint execution loops:

```
1. Compile Checkpoints (CP sequence).
2. Select first incomplete CP.
3. Evaluate Topology Decision Gate for that CP.
4. Build DAG ONLY for this active CP.
5. Spawn pruned workers (or execute directly if single-agent).
6. Execute implementation.
7. Verify Proof Contract (exit 0).
8. Merge CP baseline into repository.
9. Record feedback memory and repeat with next CP.
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

## 3. Multi-Agent Execution Mechanics

When multi-agent execution is selected for an active Checkpoint:
1. **Dynamic Tool Registry Pruning**: Subagents receive strictly pruned tool definitions via `define_subagent`, restricting schema exposure to role-essential tools.
2. **Peer-to-Peer Direct Communication**: Implementers communicate directly via `send_message` without routing intermediate state through the Orchestrator.
3. **Structured Wire Protocol**: All inter-agent messages use compact JSON payloads tagged with the active `cp_id`.
4. **Concise Technical Register**: Communicate directly with minimal prose overhead, keeping inter-agent exchanges focused on status and deliverables.
5. **Short-Circuit TDD Gate**: Automated test execution precedes final sign-off (`exit 0` required; fast-failing flags `-q --tb=short`).

---

## 4. Standard Wire Protocol Schema

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

## 5. Specifications & Reference Architecture

- [`roles.md`](roles.md) - Canonical role contracts (Orchestrator, Implementer, Verifier), tool allocations, and wire protocol semantics.
- [Checkpoint Compilation](references/checkpoint-compilation.md) - Decomposition rules, compilation algorithms, and YAML schemas.
- [DAG Orchestration & Lifecycle](references/dag-orchestration.md) - Single-checkpoint execution loop, topology gate, and P2P coordination.
- [The Proof Contract](references/proof-contract.md) - Deterministic verification categories, fast-failing flags, and zero-tolerance gates.
- [Context & Tool Registry Pruning](references/context-pruning.md) - Dynamic schema reduction, pre-flight context bounding, and token benchmarks.

---

## 6. Operational Constraints & Safety

- **Mandatory Checkpoint Compilation**: Never begin execution or worker dispatch without a compiled sequence of atomic Checkpoints.
- **Single Active Checkpoint Invariant**: Maintain an execution DAG only for the currently active Checkpoint. Never construct a multi-agent DAG for the entire request at once.
- **Zero-Tolerance Proof Gate**: No Checkpoint is marked complete without passing automated test and verification contracts (`exit 0`).
- **Pruned Tool Enforcement**: Grant subagents only the tools essential for their assigned role as specified in [`roles.md`](roles.md).
- **Concise Communication**: Keep all inter-agent messages and progress updates focused on structured deliverables and test outcomes.
- **Workspace Boundary**: Restrict all operations strictly to files within the active workspace root.
