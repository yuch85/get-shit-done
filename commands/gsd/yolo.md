---
name: gsd:yolo
description: Run full GSD lifecycle autonomously with RLM architecture — each phase gets a fresh 200K context via Phase Runner subagents
argument-hint: "[phase-number | all]"
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
  - Task
  - WebFetch
  - AskUserQuestion
---
<objective>
Run the entire GSD lifecycle autonomously using the RLM (Recursive Language Model) architecture. Each phase is delegated to a Phase Runner subagent with its own fresh 200K context window, eliminating context exhaustion across multi-phase builds.

The supervisor stays ultra-lean (~30K tokens across 9 phases). Phase Runners handle discuss, plan, execute, verify, and debug independently. State persists via YOLO-STATE.json.

**Architecture:** Supervisor (Root LLM) → Phase Runners (Sub-LLM, one per phase) → GSD agents (per-task)
**Self-re-invokes** every N phases for hard context reset.
**Resumes** from YOLO-STATE.json if interrupted.
</objective>

<execution_context>
@./.claude/get-shit-done/workflows/yolo.md
@./.claude/agents/gsd-yolo-phase-runner.md
@./.claude/get-shit-done/references/ui-brand.md
</execution_context>

<context>
Arguments: $ARGUMENTS (optional: starting phase number, or "all" for full milestone)

- Phase number → start from that phase
- "all" → start from phase 1 regardless of completion
- No argument → start from first incomplete phase
- Detects existing YOLO-STATE.json → offers to resume

Recommended: run with `claude --dangerously-skip-permissions`
</context>

<process>
Execute the YOLO workflow from @./.claude/get-shit-done/workflows/yolo.md end-to-end.

**You are the lean RLM supervisor.** Your job:
1. Initialize YOLO-STATE.json with phase list and config
2. For each phase: spawn ONE Phase Runner subagent (reads its own instructions from .claude/agents/gsd-yolo-phase-runner.md)
3. After each runner returns: read phase status from YOLO-STATE.json (NOT from runner's return message)
4. Self-re-invoke after every `yolo_phases_per_batch` phases for fresh context
5. Generate completion report via subagent

**You NEVER:** read ROADMAP.md after init, read REQUIREMENTS.md, read PLANs, read source code, run verify/debug steps yourself. Phase Runners handle all of that.

All subagents use models from `.planning/config.json` model_overrides. Default: `opus`.
</process>
