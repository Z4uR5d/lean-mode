# Lean Mode - The Proof Contract

The Proof Contract is the non-negotiable verification mechanism of Lean Mode. It guarantees that no Checkpoint is merged into the codebase without deterministic, automated proof of correctness.

---

## 1. Core Principles

> [!CRITICAL]
> **"Code looks correct" is never proof.**  
> A Checkpoint completes and merges only after all automated verification commands return exit code 0. Subjective conversational approvals or speculative assumptions are strictly rejected.

1. **Deterministic Verification**: Proof must be executable via automated commands (`run_command`), producing unambiguous binary outcomes (`PASS` / `FAIL`).
2. **Fast-Failing Assertions**: Test runners must use fast-failing flags (`pytest -q --tb=short`, `vitest run`, etc.) to prevent long stack traces from bloating agent context windows.
3. **Revertability**: If an implementation fails its Proof Contract and cannot be resolved cleanly, the Checkpoint is reverted to the preceding clean Checkpoint baseline.

---

## 2. Accepted Proof Categories

| Category | Typical Command / Tool | Verification Scope |
| :--- | :--- | :--- |
| **Unit Tests** | `pytest -q --tb=short`, `npm test`, `cargo test` | Verifies isolated logic, algorithmic correctness, and edge cases. |
| **Type Checking** | `npx tsc --noEmit`, `mypy --strict` | Proves interface consistency and eliminates null/type bugs. |
| **Linting & AST Audits**| `ruff check`, `eslint --max-warnings 0` | Enforces code formatting, syntax hygiene, and style invariant rules. |
| **Integration / Headless CLI**| Direct CLI execution without UI simulators | Verifies decoupled business logic and cross-module state transitions. |
| **Snapshots / Visual Diffs**| Automated visual regression diffs | Ensures UI layout fidelity against baseline design tokens. |

---

## 3. Verifier Gate Protocol

When an implementer completes coding for a Checkpoint:
1. Implementer sends a `RESULT` wire message to the Verifier.
2. The Verifier executes all declared `proof` commands independently.
3. **If all pass (`exit 0`)**: Verifier sends `ACK` to the Orchestrator with execution summary.
4. **If any fails (`exit != 0`)**: Verifier sends `REJECT` back to the implementer with line-numbered error traces, halting the merge loop until fixed.
