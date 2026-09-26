# Lean Mode - Scope Classification & Checkpoint Compilation

This document specifies the deterministic three-tier scope classification (Epic $\to$ Checkpoint $\to$ Task), the decomposition rules, and the Checkpoint compilation protocol.

---

## 1. Scope Classification (Pre-Flight Triage)

Before planning or compiling increments, the Orchestrator classifies the request using a deterministic three-tier hierarchy:

```
[ Request ]
    │
    ├── Multiple independent subsystems? ──► [ Epic ] ──► Sequential/Isolated Checkpoint lists
    │
    ├── Single subsystem, 2–10 mergeable steps? ──► [ Checkpoint ] ──► Atomic incremental sequence (CP-01, CP-02...)
    │
    └── Single atomic change, 1 merge commit? ──► [ Task ] ──► Direct single-agent execution with local Proof Contract
```

### Structural Complexity Matrix

Instead of estimating arbitrary lines of code, Lean Mode measures structural complexity:

| Attribute | Task | Checkpoint | Epic |
| :--- | :---: | :---: | :---: |
| **Independent Subsystems** | 1 | 1 | 2+ |
| **Merge Commits** | 1 | 2–10 | 10+ |
| **Proof Contracts** | 1 | Several (1 per CP) | Dozens |
| **Parallelizable** | No | Rarely | Yes |
| **Autonomous Deliverable Value** | No (atomic diff) | No (intermediate increment) | Yes (standalone domain) |

### The Triage Decision Algorithm

```
IF request touches multiple independent subsystems
    → Decompose into Epics (EPIC-01, EPIC-02...)

FOR each Epic (or single-subsystem request):
    IF work requires multiple independently verifiable increments
        → Compile into Checkpoints (CP-01, CP-02...)
    ELSE
        → Execute as a single atomic Task
```

### Concrete Tier Examples

| Tier | Request Example | Resolution Strategy |
| :--- | :--- | :--- |
| **Epic** | *"Build distributed benchmark system"* | Decompose into independent domains: Dataset, Runner, Telemetry, Reports, Storage. |
| **Checkpoint** | *"Implement Telemetry subsystem"* | Compile into atomic CPs: `CP-01 Event Schema`, `CP-02 Token Collector`, `CP-03 JSON Writer`. |
| **Task** | *"Fix incorrect HTTP 404 response on /health"* | Execute directly as a single-agent change with test proof. No Checkpoint overhead. |

> [!IMPORTANT]
> **Heuristic**: If the deliverable could be assigned to two engineering teams to build concurrently without merge conflicts, it is an **Epic**. If an increment is already atomic (1 merge, 1 Proof Contract), it is a **Task**.

---

## 2. What is a Checkpoint?

A **Checkpoint** is the smallest independently verifiable engineering change that can be merged safely into a repository without breaking system integrity.

Every Checkpoint must satisfy all five criteria:
1. **One Logical Objective**: Exactly one clear intent with a single dominant concern (e.g. data schema, business logic, or UI presentation—never blended).
2. **One Behavioral Change**: Exactly one new capability, refactor, or bugfix.
3. **Bounded Scope**: Touches $\le 3$ closely related files when practical.
4. **Independently Reviewable & Revertable**: Can be understood and inspected in minutes; reverting it leaves adjacent components intact.
5. **Automatically Verifiable**: Accompanied by deterministic automated proof before completion.

---

## 3. Decomposition Rules & The "And" Heuristic

The compiler splits work within an Epic or subsystem until every step meets the atomic criteria.

> **The "And" Heuristic**:  
> If a Checkpoint's title or objective naturally requires the word **"and"** (e.g., *"Create UserStore and update LoginView"*), it must be split into two distinct Checkpoints.

### Examples: Good vs. Bad Decomposition

| Bad (Blended / Coarse) | Good (Atomic Checkpoint) | Rationale |
| :--- | :--- | :--- |
| ✗ *Implement Authentication* | ✓ `CP-01: Add TokenValidator with unit tests` | Isolates validation logic with pure unit tests. |
| ✗ *Refactor Database & API Models* | ✓ `CP-02: Migrate UserSchema to v2 with zero API change` | Single concern; leaves database backward-compatible. |
| ✗ *Build Timeline Screen* | ✓ `CP-03: Extract TimelineLayout calculation from view` | Decouples computation from rendering. |
| ✗ *Fix Cache Bugs and Update Styles* | ✓ `CP-04: Fix cache invalidation TTL in Store` | Eliminates unrelated diff coupling. |

---

## 4. The Compilation Algorithm

```
Subsystem / Epic Scope
         │
         ▼
[ 1. Analyze AST & Dependency Boundaries ]
         │
         ▼
[ 2. Split into Atomic Increments ]
         │
         ▼
[ 3. Assign Sequential Identifiers (CP-01, CP-02...) ]
         │
         ▼
[ 4. Define Explicit Proof Contract & Touched Files ]
         │
         ▼
[ 5. Select CP-01 -> Hand off to Orchestrator ]
```

*The compiler produces only structure and verification contracts. It never executes implementation code or spawns subagents during compilation.*

---

## 5. Canonical Checkpoint Specification (YAML)

Every compiled Checkpoint must follow this structured schema:

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
  - layout calculation API remains backward-compatible
  - all proofs exit with code 0
  - repository passes full lint gate
```

---

## 6. Feedback Memory Loop Across Checkpoints

- As Checkpoints progress sequentially (`CP-01` $\to$ `CP-02` $\to$ ...), the Orchestrator records lessons learned, linter quirks, and test failure patterns from completed Checkpoints.
- This feedback memory is preserved across the sequence, allowing subsequent workers to execute with higher autonomy and fewer errors.
