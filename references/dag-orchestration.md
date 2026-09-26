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
> **The Orchestrator constructs an execution DAG strictly for one active Checkpoint at a time.** It never plans or executes future Checkpoints concurrently.

---

## 2. Orchestration Execution Lifecycle

```
[ Request ]
    │
    ▼
[ 1. Synchronize project-plan.md ] ◄─────────────────────────────────────────────┐
    │ (Read or create SSOT in project root)                                      │
    ▼                                                                            │
[ 2. Scope Classification ]                                                      │
    ├── Multiple Subsystems ──► [ Epics recorded in project-plan.md ] ──┐        │
    ├── Atomic Change (1 merge) ──► [ Execute Task directly (exit 0) ]  │        │
    └── Subsystem with Increments ──────────────────────────────────────┤        │
                                                                        ▼        │
                                                      [ 3. Compile Checkpoints ] │
                                                        (Write CPs to plan)      │
                                                                        │        │
                                                                        ▼        │
                                                   ┌──► [ 4. Set Current Focus ] │
                                                   │      (Update plan header)   │
                                                   │                    │        │
                                                   │                    ▼        │
                                                   │           [ 5. Topology Gate ]
                                                   │              ├── <= 3 files ──► [ Single-Agent Monolith ]
                                                   │              └── Concurrent ──► [ Local Multi-Agent DAG ]
                                                   │                    │                    │
                                                   │                    └─────────┬──────────┘
                                                   │                              ▼
                                                   │                 [ 6. Execute Implementation ]
                                                   │                              │
                                                   │                              ▼
                                                   │                 [ 7. Enforce Proof Contract ]
                                                   │                              │
                                                   │                Passes? ──────┴────── Failing?
                                                   │                   │                     │
                                                   │                (exit 0)              (exit != 0)
                                                   │                   │                     │
                                                   │                   ▼                     ▼
                                                   │          [ 8. Post-Sync Plan ]   [ Reject with Trace ]
                                                   │            - Mark CP [x]
                                                   │            - Add Completed Log
                                                   │            - Merge baseline
                                                   │                   │
                                                   └── Checkpoints left? ──────┘
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
