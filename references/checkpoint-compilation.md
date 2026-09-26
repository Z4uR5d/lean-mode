# Lean Mode - Checkpoint Compilation

Checkpoint Compilation is the mandatory preliminary phase of Lean Mode. It decomposes complex requests into an ordered sequence of atomic, independently verifiable engineering increments before any execution begins.

---

## 1. What is a Checkpoint?

A **Checkpoint** is the smallest independently verifiable engineering change that can be merged safely into a repository without breaking system integrity.

Every Checkpoint must satisfy all five criteria:
1. **One Logical Objective**: Exactly one clear intent with a single dominant concern (e.g. data schema, business logic, or UI presentation—never blended).
2. **One Behavioral Change**: Exactly one new capability, refactor, or bugfix.
3. **Bounded Scope**: Touches $\le 3$ closely related files when practical.
4. **Independently Reviewable & Revertable**: Can be understood and inspected in minutes; reverting it does not break adjacent components.
5. **Automatically Verifiable**: Accompanied by deterministic automated proof before completion.

---

## 2. Decomposition Rules & The "And" Heuristic

The compiler splits work until every step meets the atomic criteria.

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

## 3. The Compilation Algorithm

```
User Request
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

## 4. Canonical Checkpoint Specification (YAML)

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

## 5. Feedback Memory Loop Across Checkpoints

- As Checkpoints progress sequentially (`CP-01` $\to$ `CP-02` $\to$ ...), the Orchestrator records lessons learned, linter quirks, and test failure patterns from completed Checkpoints.
- This feedback memory is preserved across the sequence, allowing subsequent workers to execute with higher autonomy and fewer errors.
