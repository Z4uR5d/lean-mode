# Lean Mode Agent Roles & Tool Specifications

This document is the canonical source of truth for agent roles, tool allocations, `define_subagent` registration contracts, and inter-agent wire protocols in Lean Mode.

---

## 1. Orchestrator (`orchestrator`)

- **Objective**: Scope classification (Task vs. Checkpoint vs. Epic), Checkpoint compilation, execution DAG planning strictly for the active Checkpoint, dynamic worker registration and dispatch, Proof Contract verification, and merge sign-off.
- **Orchestration Execution Contract**:
  1. **Classify Scope**: Determine tier (Task vs. Checkpoint sequence vs. Epics) using the structural complexity matrix.
     - *Task*: Execute directly as a single agent with a local Proof Contract.
     - *Epic*: Decompose into domain-bounded Epics; sequence or isolate active Epic.
  2. **Compile Checkpoints**: For the active Epic or multi-step subsystem, compile atomic Checkpoints (`CP-01`, `CP-02`, ...).
  3. **Select**: Identify the first incomplete Checkpoint.
  4. **Triage Topology**: Apply the Pre-Flight Topology Gate (Single-Agent Monolith vs. Multi-Agent DAG).
  5. **Plan Single-CP DAG**: Construct execution DAG *strictly for the active Checkpoint*. Never plan beyond the current Checkpoint.
  6. **Dispatch**: Dynamically register and spawn pruned workers via `define_subagent`.
  7. **Verify**: Await and validate the Proof Contract (`exit 0` required).
  8. **Merge**: Mark Checkpoint complete, merge working baseline, record feedback memory, and proceed to the next Checkpoint.
- **Allowed Tools**:
  - `send_message`: Dispatches instructions to workers and receives structured status reports.
  - `define_subagent`: Dynamically registers pruned implementer and verifier roles.
  - `invoke_subagent`: Spawns subagents with targeted context and roles.
  - `manage_subagents`: Monitors subagent lifecycle and terminates completed processes.
  - `run_command`: Strictly for pre-flight environment checks or invoking test runners at Checkpoint boundaries.
  - `view_file`: Read-only file inspection.
- **Operational Boundaries**:
  - Delegates all source code modifications to implementers; never calls `write_to_file` or `replace_file_content`.
  - Context is strictly bounded to the active Checkpoint—never models future Checkpoints concurrently.

---

## 2. Implementer (`implementer`)

- **Objective**: Focused, single-threaded execution of scoped modules for the active Checkpoint. Directly writes code, implements unit tests, executes builds, and refactors components.
- **Dynamic Registration Contract (`define_subagent`)**:
  ```python
  define_subagent(
      name="implementer",
      description="Autonomous code implementer. Writes code, executes builds, and runs unit tests in scoped modules.",
      system_prompt="Implement scoped modules according to the task specification. Write unit tests before completing changes. Keep updates concise and formatted as JSON wire payloads.",
      enable_write_tools=True,       # Grants run_command, write_to_file, replace_file_content + default read tools (view_file) and comms (send_message)
      enable_subagent_tools=False,   # Prunes define_subagent, invoke_subagent, manage_subagents
      enable_mcp_tools=False         # Prunes external MCP tool catalogs
  )
  ```
- **Allowed Tools (Pruned)**:
  - `run_command`: Compilations, package installations, and test runners (`pytest`, `npm test`, `cargo test`, `mypy`).
  - `write_to_file`: File creation.
  - `replace_file_content`: Precise chunk-based edits.
  - `view_file`: Reading implementation files.
  - `send_message`: Peer-to-peer communication with peer implementers or reporting completion to Orchestrator/Verifier.
- **Pruned / Omitted Tools**:
  - Excludes web browsing, image generation, scheduling, and subagent orchestration (`search_web`, `read_url_content`, `generate_image`, `schedule`, `define_subagent`, `invoke_subagent`).
  - Limits subagent schema exposure strictly to permitted role actions.

---

## 3. Verifier (`verifier`)

- **Objective**: Independent quality validation gate. Enforces Proof Contract criteria, executes test suites and linters with fast-failing flags, and prevents regressions.
- **Dynamic Registration Contract (`define_subagent`)**:
  ```python
  define_subagent(
      name="verifier",
      description="Independent quality validation gate. Audits test coverage, executes test suites, and enforces zero-regression gates.",
      system_prompt="Validate implementations by executing test suites and linters with fast-failing flags (-q --tb=short). Emit ACK or REJECT with exact error traces using the JSON wire format.",
      enable_write_tools=True,       # Grants run_command for test runners and view_file for inspection; source code modifications are restricted to implementers
      enable_subagent_tools=False,   # Prunes subagent management
      enable_mcp_tools=False         # Prunes external MCP tool catalogs
  )
  ```
- **Allowed Tools (Pruned)**:
  - `run_command`: Executing test harnesses (`pytest -q --tb=short`, `npm test`, etc.), linters (`ruff`, `eslint`), and typechecks (`tsc`, `mypy`).
  - `view_file`: Inspecting test reports and diffs.
  - `send_message`: Emitting terse ACK or REJECT signals with line-numbered failure traces.
- **Gate Behavior**:
  - Strict Proof Gate: Any failing assertion triggers immediate rejection back to the implementer with specific trace context.
  - "Code looks correct" without passing test suites is rejected.

---

## 4. Standard Wire Protocol Schema

Format all inter-agent messages sent via `send_message` using this compact JSON structure:

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

### Protocol Action Semantics:
- `TASK`: Orchestrator dispatches scoped task to implementer with `in_scope` file boundaries and `cp_id`.
- `HANDOFF`: Implementer transfers interface definitions or dependency exports directly to a peer implementer.
- `SYNC`: Peer-to-peer alignment query or interface contract exchange.
- `RESULT`: Implementer notifies Verifier/Orchestrator that implementation is ready for gate evaluation.
- `REJECT`: Verifier returns failed test output and line trace to implementer (`exit_code != 0`).
- `ACK`: Verifier signs off passing status (`exit_code == 0`) to Orchestrator.
