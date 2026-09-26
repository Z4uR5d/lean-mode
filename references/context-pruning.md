# Lean Mode - Context & Tool Registry Pruning

This document specifies the context bounding protocols, dynamic tool registry pruning, and the empirical evidence base grounding the Pre-Flight Topology Decision Gate.

---

## 1. Dynamic Tool Registry Pruning

Default agent environments inject JSON Schema declarations for every globally registered tool on every turn. In large tool catalogs, this adds substantial per-turn context overhead and exposes agents to tools outside their operational scope.

Lean Mode instantiates subagents dynamically via `define_subagent` with only the exact tools required for their assigned role:

- **Implementers** receive only: `run_command`, `write_to_file`, `replace_file_content`, `view_file`, and `send_message`.
  - Omitted: `generate_image`, `read_url_content`, `search_web`, `schedule`, `manage_task`, `ask_question`, `define_subagent`, `invoke_subagent`.
- **Verifiers** receive only: `run_command`, `view_file`, and `send_message`.
  - Omitted: All file write tools, subagent management tools, web browsing, and image generation.
- **Operational Impact**: Prevents tool hallucination, enforces security boundaries, and minimizes schema footprint in worker context.

---

## 2. Pre-Flight Context Bounding

Blind workspace dumps bloat context and lead to hallucinated file boundaries. Lean Mode strictly bounds worker context:

1. **Explicit `in_scope` Boundaries**: Every task dispatched by the Orchestrator or handed off between peers specifies exact target files.
2. **Local AST Inspection**: Implementers use targeted `view_file` queries with line-number slices rather than dumping entire codebases into prompts.
3. **Headless Execution**: Business logic is decoupled from heavy UI trees, allowing state inspection and execution in milliseconds without heavy simulator output.

---

## 3. Concise Technical Register

Inter-agent communication is conducted in a structured, direct technical register:
- **No Conversational Pleasantries**: Agents omit greetings, conversational filler, and verbose tool narration.
- **Structured Wire Payloads**: Messages use the compact JSON Wire Protocol schema (`action`, `cp_id`, `in_scope`, `status`, `exit_code`, `payload`).
- **Bounded Status Updates**: Progress reports communicate binary state, error traces, and test outcomes.

---

## 4. Empirical Evidence for the Decision Gate

The Pre-Flight Topology Decision Gate is grounded in measured empirical benchmarks comparing single-agent and multi-agent execution:

1. **The Negative Result (Functional Serialization)**:
   - On tasks with sequential dependencies or small surfaces ($\le 3$ files), multi-agent dispatch incurs a **2.35x–3.08x token penalty** with **zero latency acceleration** (e.g. 155.5s multi-agent vs. 153.8s single-agent).
   - Downstream workers block or poll waiting on upstream file artifacts, completely eliminating parallel speedups while multiplying token spend.
2. **Decision Gate Rule**:
   - **Single-Agent Monolith**: Mandatory whenever a Checkpoint touches $\le 3$ files, involves sequential dependencies, or modifies a single component.
   - **Multi-Agent DAG**: Permitted strictly when modules are genuinely concurrent and independent, or when an adversarial verification gate is required.
