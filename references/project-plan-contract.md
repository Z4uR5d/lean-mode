# Lean Mode - Project Plan as Single Source of Truth

This document specifies the contract, canonical structure, and automatic update rules for `/project-plan.md`—the persistent Single Source of Truth (SSOT) for all Lean Mode execution.

---

## 1. The Single Source of Truth Invariant

In standard agent interactions, execution plans exist only ephemerally in model context, chat history, or subagent transcripts. This makes projects nondeterministic:
- New chat sessions lose track of active checkpoints and completed work.
- Worker subagents hallucinate past progress or duplicate completed refactors.
- Git branch switches or rollbacks desynchronize model assumptions from filesystem reality.

### The Lean Invariant
> [!CRITICAL]
> **The only authoritative state of the project is stored in `/project-plan.md` in the project root.**  
> No internal model thinking, conversational memory, or chat messages constitute project state. Agents must never infer project state from chat history when `/project-plan.md` exists.

If `/project-plan.md` is missing, Lean Mode must create it before any planning, compilation, or execution begins.

---

## 2. Canonical Format Contract

The file `/project-plan.md` must adhere to this exact structural format so that any agent or human can read and update it predictably:

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

---

## 3. Automatic Update & Synchronization Rules

The Orchestrator maintains `/project-plan.md` automatically without asking for user confirmation.

| Trigger Event | Required File Action |
| :--- | :--- |
| **New Project / Missing File** | Create `/project-plan.md` with Project name and initial Status. |
| **Epics Classified** | Populate the `## Epics` checklist (`EP1...EPn`). |
| **Epic Started** | Create section `## EPx <Name>`, set `Status: In Progress`, list compiled Checkpoints. |
| **Checkpoint Started** | Update `## Current Focus` with the active Epic, Checkpoint ID, and immediate Task. |
| **Checkpoint Completed** | Mark Checkpoint as `[x]`, append timestamped entry to `## Completed Log`. |
| **Epic Completed** | Mark Epic `[x]` in `## Epics`, set Epic `Status: Completed`. |
| **New Requirement / Discovery** | Insert new Epic or Checkpoint into the appropriate section. |

### The Sync-Execute-Sync Protocol
Before any tool call or code modification that advances project state:
1. **Pre-Sync**: Ensure `/project-plan.md` reflects the current focus and scope.
2. **Execute**: Implement the code changes and verify the Proof Contract (`exit 0`).
3. **Post-Sync**: Immediately mark the Checkpoint `[x]` and log completion in `/project-plan.md`.

---

## 4. Why a Single Markdown File?

Lean Mode deliberately avoids multi-file fragmentation (`epics.yaml`, `tasks.json`, `.checkpoints/*.md`) in favor of a single `/project-plan.md`:

1. **Human & Machine Legibility**: Humans read and edit it directly in markdown preview without special CLI tools.
2. **Single-Turn Context Ingestion**: The model reads the entire roadmap and current state in a single `view_file` operation (~50 lines).
3. **Clean Version Control**: Git diffs clearly highlight completed checkpoints and shifting focus per commit.
4. **Zero State Drift**: Impossible for task lists, epic statuses, and checkpoint progress to fall out of sync across multiple files.

### Scaling Beyond 100+ Checkpoints
When projects grow large, the `## Completed Log` can roll entries older than 30 days into an archive file (`/docs/plan-archive.md`), but `/project-plan.md` remains the active, consolidated source of truth.
