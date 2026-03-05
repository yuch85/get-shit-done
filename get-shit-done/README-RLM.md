# GSD Fork — RAGStack Custom Build

**Base:** [get-shit-done](https://github.com/gsd-build/get-shit-done) v1.22.4
**Fork purpose:** RLM-enhanced autonomous execution with confidence gates and all-Opus model enforcement.

## What Changed

### 1. RLM Architecture for `/gsd:yolo` (NEW)

Implements the [Recursive Language Model](https://arxiv.org/abs/2512.24601) pattern (Zhang, Kraska, Khattab — MIT CSAIL) for autonomous multi-phase execution without context exhaustion.

**Architecture:**
```
Supervisor (Root LLM, ~30K tokens for 9 phases)
  |-- Phase Runner 1 (fresh 200K) -> discuss -> plan -> execute -> verify -> debug
  |-- Phase Runner 2 (fresh 200K) -> reads STATE.md -> full lifecycle
  |-- Phase Runner N (fresh 200K) -> ...
```

- **Supervisor** stays ultra-lean, reads only `YOLO-STATE.json`
- **Phase Runners** get fresh 200K context per phase, handle the entire GSD lifecycle
- **State passes via files on disk** (`YOLO-STATE.json`, `STATE.md`, `SUMMARY.md`) — no context accumulation
- **Resume from crash**: re-run `/gsd:yolo` to detect `YOLO-STATE.json` and continue

**Files:**
- `.claude/agents/gsd-yolo-phase-runner.md` — Phase Runner agent (Sub-LLM)
- `.claude/get-shit-done/workflows/yolo.md` — Supervisor workflow (Root LLM)
- `.claude/commands/gsd/yolo.md` — Command entry point

### 2. Three-Tier Confidence Gates (NEW)

Prevents the autonomous pipeline from silently making decisions that conflict with established project direction.

| Level | Meaning | Action |
|---|---|---|
| **HIGH** | Clear path, consistent with prior decisions | Proceed silently |
| **MEDIUM** | Ambiguity with reasonable default | Proceed, log to `ATTENTION.md` for user review |
| **LOW** | Direct conflict with prior decision or architectural surprise | STOP, write `ESCALATION.md`, return to supervisor for human input |

**Escalation flow:**
1. Subagent (researcher/planner/auto-discuss) detects conflict with prior `CONTEXT.md` decisions
2. Writes `ESCALATION.md` with options (a/b/c) and auto-inferred preference
3. Phase Runner sets status to `needs_input`, returns to supervisor
4. Supervisor reads `ESCALATION.md`, presents to user via `AskUserQuestion`
5. User responds, supervisor writes `DECISION.md`, re-spawns Phase Runner
6. Phase Runner resumes from the interrupted step incorporating human decisions

**Triggers for LOW confidence:**
- Research invalidates a library/pattern chosen in a prior phase
- Auto-discuss generates a decision contradicting a locked prior decision
- Phase goal requires structural changes conflicting with PROJECT.md architecture
- A requirement cannot be met without significant rework of a prior approach

**MEDIUM items** accumulate in `ATTENTION.md` files and are aggregated in the final `YOLO-REPORT.md` for post-run review.

### 3. All-Opus Model Enforcement

Quality profile changed so ALL agents resolve to Opus (via `inherit`):

**`bin/lib/core.cjs` MODEL_PROFILES changes:**
- `gsd-research-synthesizer`: sonnet -> opus
- `gsd-codebase-mapper`: sonnet -> opus
- `gsd-verifier`: sonnet -> opus
- `gsd-plan-checker`: sonnet -> opus
- `gsd-integration-checker`: sonnet -> opus
- `gsd-nyquist-auditor`: sonnet -> opus
- Unknown agent fallback: sonnet -> inherit (opus)
- Missing profile fallback: sonnet -> opus

### 4. YOLO Config Settings

Added to `.planning/config.json`:

```json
{
  "yolo_max_debug_retries": 2,
  "yolo_skip_discuss": false,
  "yolo_continue_on_block": true,
  "yolo_phases_per_batch": 4
}
```

| Setting | Value | Description |
|---|---|---|
| `yolo_max_debug_retries` | 2 | Debug cycles per phase before marking blocked |
| `yolo_skip_discuss` | false | Auto-generates CONTEXT.md unless one already exists |
| `yolo_continue_on_block` | true | Moves to next phase if one gets stuck |
| `yolo_phases_per_batch` | 4 | Batch boundary logging interval |

## Usage

```bash
claude --dangerously-skip-permissions
/gsd:yolo          # Start from first incomplete phase
/gsd:yolo 3        # Start from phase 3
/gsd:yolo all      # Re-run all phases from 1
```

Monitor progress: check `.planning/YOLO-STATE.json` or `git log --oneline`.

## Upstream Compatibility

Based on upstream v1.22.4. All standard GSD commands work unchanged. The fork only adds:
- 2 new files (yolo workflow + phase runner agent)
- 1 modified file (core.cjs model profiles)
- 1 new command (yolo.md)

To update from upstream: pull new files, re-apply model profile patch to `core.cjs`, keep custom yolo/phase-runner files.
