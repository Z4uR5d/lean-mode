# Lean Mode - DAG Orchestration & Execution Lifecycle

This document specifies the execution lifecycle, the single-checkpoint DAG invariant, and peer-to-peer worker coordination in Lean Mode.

---

## 1. The Single-Checkpoint DAG Invariant

Traditional multi-agent frameworks attempt to plan an entire multi-phase project into a single global Directed Acyclic Graph (DAG) before execution begins. In practice, this introduces critical failure modes:
- **Context Bloat**: The orchestrator's context window rapidly saturates with speculative task dependencies and intermediate chatter.
- **Hallucinated Dependencies**: Early assumptions about later modules become invalid as implementation progresses, causing cascade failures.
- **Coordination Tax**: Spawning subagents across large sequential task graphs introduces a 2.35x–3.08x token penalty without delivering parallel acceleration.

### The Lean Solution
> [!IMPORTANT]
> **The Orchestrator constructs an execution DAG strictly for ONE active Checkpoint at a time.** It never plans or executes future Checkpoints concurrently.

---

## 2. Orchestration Execution Lifecycle

```
[ Compile Checkpoints: CP-01, CP-02, ... CP-N ]
                     │
                     ▼
       ┌──► [ Select Next Incomplete CP ]
       │             │
       │             ▼
       │    [ Evaluate Topology Gate ]
       │       ├── Touches <= 3 files / Sequential ──► [ Execute Single-Agent Monolith ]
       │       └── Concurrent Modules / Review Gate ──► [ Plan DAG & Dispatch Workers ]
       │             │                                           │
       │             └───────────────────┬───────────────────────┘
       │                                 ▼
       │                     [ Execute Implementation ]
       │                                 │
       │                                 ▼
       │                     [ Enforce Proof Contract ]
       │                                 │
       │                    Passes? ─────┴───── Failing?
       │                       │                    │
       │                    (exit 0)             (exit != 0)
       │                       │                    │
       │                       ▼                    ▼
       │              [ Merge CP Baseline ]    [ Reject to Worker with Trace ]
       │                       │
       └── Checkpoints left? ──┘
```

---

## 3. Pre-Flight Topology Decision Gate

Before spawning any subagent for an active Checkpoint, evaluate task topology:

1. **Single-Agent Lean Monolith**:
   - **Condition**: Checkpoint touches $\le 3$ files, involves sequential dependencies, or modifies a single component.
   - **Action**: Execute directly as a single agent. Spawning subagents on small or sequential tasks incurs severe token overhead without speedup.
2. **Multi-Agent Lean DAG**:
   - **Condition**: Checkpoint involves 2+ genuinely independent, concurrent sub-modules OR requires an adversarial verification gate.
   - **Action**: Decompose into a local DAG and dispatch dynamically registered workers with pruned tools.

---

## 4. Peer-to-Peer Worker Coordination

When multi-agent execution is active:
- Implementers exchange interfaces, schemas, and types directly via `send_message(Recipient=subagent_id)`.
- Communication completely bypasses the Orchestrator's context window, eliminating centralized relay bloat.
- All messages use the compact Standard Wire Protocol tagged with `cp_id`.
