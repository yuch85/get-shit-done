<purpose>
Lean supervisor implementing the RLM (Recursive Language Model) pattern for autonomous multi-phase execution. Each phase is delegated to a Phase Runner subagent with its own fresh 200K context window. The supervisor NEVER performs phase-level work.
</purpose>

<architecture>
## RLM Architecture — Context Window Survival by Design

Based on the Recursive Language Model pattern (Zhang, Kraska, Khattab — MIT CSAIL):

```
┌─────────────────────────────────────────────────────────┐
│  YOLO SUPERVISOR (Root LLM)                             │
│  Context budget: ~40K tokens across entire run          │
│                                                         │
│  Reads: YOLO-STATE.json (small, targeted)               │
│  Spawns: one Phase Runner per phase (fresh 200K each)   │
│  Checks: phase status field after runner returns        │
│  Handles: escalations (presents LOW confidence to user) │
│  Batch boundaries: logged every N phases (no respawn)   │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │  PHASE RUNNER (Sub-LLM) — fresh 200K per phase   │  │
│  │                                                   │  │
│  │  Handles: discuss → plan → execute → verify →     │  │
│  │           debug → finalize (the FULL pipeline)    │  │
│  │                                                   │  │
│  │  Confidence gates after discuss, research, plan:  │  │
│  │    HIGH → proceed silently                        │  │
│  │    MEDIUM → proceed, log to ATTENTION.md          │  │
│  │    LOW → STOP, write ESCALATION.md, return        │  │
│  │                                                   │  │
│  │  Spawns its own subagents:                        │  │
│  │    ├── gsd-planner (discuss/plan)                 │  │
│  │    ├── gsd-phase-researcher (research)            │  │
│  │    ├── gsd-plan-checker (verify plans)            │  │
│  │    ├── gsd-executor (execute tasks)               │  │
│  │    ├── gsd-verifier (verify phase)                │  │
│  │    └── gsd-executor (debug+fix)                   │  │
│  │                                                   │  │
│  │  Updates: YOLO-STATE.json with phase results      │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  State persistence: YOLO-STATE.json (the "REPL pickle") │
│  File handoffs: all artifacts on disk, never in context  │
│  Resume: re-invoke /gsd:yolo → detects state → resumes  │
└─────────────────────────────────────────────────────────┘
```

**RLM Mapping:**
| RLM Concept | YOLO Implementation |
|---|---|
| Large document | Full milestone (N phases) |
| Chunks | Individual phases |
| Root LLM | YOLO Supervisor (this workflow) |
| Sub-LLM | YOLO Phase Runner (fresh context per phase) |
| Persistent REPL state | YOLO-STATE.json |
| Buffers | Phase results accumulated in state file |
| `llm_query()` subcall | Phase Runner subagent spawn |

**Why this works:**
- Supervisor uses ~3K tokens per phase (spawn prompt + return + state read/write)
- Over 9 phases: ~35K tokens total (well within 200K)
- Each Phase Runner gets full 200K for its single phase
- Phase Runners spawn their own subagents (each with fresh 200K)
- No context accumulation across phases — state lives on disk
- Batch boundaries logged every N phases for auditability (no agent respawn needed)

**Context budget comparison:**
| Architecture | 9-Phase Token Usage | Fits in 200K? |
|---|---|---|
| Original (inline everything) | ~202K (happy), ~354K (worst) | NO |
| RLM (supervisor + runners) | ~35K supervisor + 200K per runner | YES |
</architecture>

<core_principle>
The supervisor is DUMB ON PURPOSE. It does exactly five things per phase:
1. Read YOLO-STATE.json (targeted extraction, not full parse)
2. Spawn one Phase Runner subagent
3. Read the phase status from YOLO-STATE.json after runner returns
4. Handle escalations if status is `needs_input` (read ESCALATION.md, ask user, write DECISION.md, re-spawn)
5. Log one line and loop

Everything else — discussing, planning, researching, executing, verifying, debugging — happens inside Phase Runners with their own fresh context windows. The supervisor NEVER reads ROADMAP.md (after init), REQUIREMENTS.md, PROJECT.md, PLAN.md, SUMMARY.md, VERIFICATION.md, or any source code. It only reads YOLO-STATE.json and (when escalated) ESCALATION.md.
</core_principle>

<required_reading>
Read ONLY `.planning/YOLO-STATE.json` (or `.planning/config.json` + `.planning/ROADMAP.md` during initial setup if YOLO-STATE.json doesn't exist yet).

Exception: when a phase returns `needs_input`, read the ESCALATION.md file for that phase (small, structured file with the items needing human decision).

Do NOT read STATE.md, REQUIREMENTS.md, PROJECT.md, or any phase files. The Phase Runner reads those.
</required_reading>

<process>

<step name="prerequisites" priority="first">

**1. Check prerequisites:**
- Verify `.planning/PROJECT.md` exists (just check existence, don't read it)
- Verify `.planning/ROADMAP.md` exists (just check existence)
- If either missing: abort with "No project initialized. Run /gsd:new-project first."

**2. Check for existing YOLO-STATE.json (resume support):**

```bash
cat .planning/YOLO-STATE.json 2>/dev/null
```

**If exists and has pending phases:** This is a RESUME. Report what was completed, what remains, and continue from `current.phase_index`. Skip to step `phase_loop`.

**If exists and all phases complete:** Report "All phases complete. Run /gsd:audit-milestone" and exit.

**If does not exist:** This is a FRESH RUN. Continue below.

**3. Read config and build phase list:**

Read `.planning/config.json` for:
- `model_overrides` → resolve all models (default to `opus` on any failure)
- `workflow.research`, `workflow.plan_check`, `workflow.verifier`
- `yolo_max_debug_retries` (default: 2)
- `yolo_skip_discuss` (default: false)
- `yolo_continue_on_block` (default: true)
- `yolo_phases_per_batch` (default: 4 — batch boundary logging interval)
- `parallelization`

Read `.planning/ROADMAP.md` ONCE (this is the only time the supervisor reads it):
- Extract phase list: number, name, status, depends_on for each phase
- This is the LAST time the supervisor reads ROADMAP.md — Phase Runners handle all subsequent reads

**4. Parse arguments:**
- Phase number provided → start from that phase
- "all" → start from phase 1
- No argument → start from first incomplete phase
- All phases complete → report and exit

**5. Initialize YOLO-STATE.json:**

```json
{
  "version": 2,
  "architecture": "rlm",
  "started_at": "{ISO timestamp}",
  "config": {
    "start_phase": 3,
    "max_debug_retries": 2,
    "skip_discuss": false,
    "continue_on_block": true,
    "phases_per_batch": 4,
    "parallelization": true,
    "workflow": {
      "research": true,
      "plan_check": true,
      "verifier": true
    },
    "models": {
      "planner": "opus",
      "researcher": "opus",
      "executor": "opus",
      "verifier": "opus",
      "debugger": "opus",
      "checker": "opus"
    }
  },
  "phases": [
    {
      "number": 3,
      "name": "Listings UI and Map",
      "dir": "03-listings-ui-and-map",
      "depends_on": [2],
      "status": "pending",
      "tasks_completed": 0,
      "commits": 0,
      "debug_cycles": 0,
      "step": "not_started",
      "escalation_step": null
    }
  ],
  "current": {
    "phase_index": 0,
    "batch_start": 0,
    "batch_number": 1
  },
  "stats": {
    "phases_completed": 0,
    "phases_blocked": 0,
    "phases_escalated": 0,
    "total_commits": 0,
    "total_debug_cycles": 0
  }
}
```

Write YOLO-STATE.json.

**6. Report:**
```
YOLO MODE — RLM Architecture (Recursive Language Model)

Phases to execute: {start_phase} through {last_phase} ({count} phases)
Max debug retries: {max_debug_retries}
Phases per batch: {phases_per_batch}
Auto-discuss: {enabled/disabled}
Confidence gates: enabled (LOW → escalate to human, MEDIUM → log for review)
Models: all {model} (from config overrides)

Architecture: supervisor → phase runners → GSD agents
Each phase gets a fresh 200K context window.

Starting...
```

</step>

<step name="phase_loop">

**CRITICAL: This is the lean core of the RLM supervisor. MINIMAL context consumption per iteration.**

For each pending phase at `current.phase_index`:

**1. Read YOLO-STATE.json — targeted extraction only:**

```bash
cat .planning/YOLO-STATE.json
```

Extract ONLY:
- `phases[current.phase_index]` → number, name, dir, depends_on, status
- `config.continue_on_block`
- `current.batch_start`, `current.phase_index`
- `config.phases_per_batch`

**2. Batch boundary check (lightweight — no subagent spawn):**

If `(current.phase_index - current.batch_start) >= config.phases_per_batch`:
→ This batch is done. Log the transition and continue in the same supervisor.

Update YOLO-STATE.json: `current.batch_start = current.phase_index`, increment `batch_number`.

Log:
```
--- Batch {batch_number} complete ({phases_per_batch} phases). Starting batch {batch_number + 1}. ---
```

Continue the phase_loop — do NOT spawn a continuation supervisor. The supervisor's per-phase token cost (~2K) is low enough to handle 20+ phases within a single 200K context window. Spawning a continuation supervisor adds an extra nesting level that pushes Phase Runner subagents past Claude Code's practical agent depth limit (Main → Supervisor → Continuation Supervisor → Phase Runner → GSD agents = 4+ levels, which causes `[Tool result missing due to internal error]`).

**3. Dependency check:**

Check if all `depends_on` phase numbers for this phase have `status: "complete"` in the `phases` array (read from YOLO-STATE.json, already loaded in step 1).

If a dependency has status `"blocked"`:
- Update this phase's status to `"blocked_dependency"` in YOLO-STATE.json
- Log: `Phase {N}: SKIPPED (dependency blocked)`
- Increment `current.phase_index`, loop

**4. Spawn Phase Runner subagent:**

This is the key RLM delegation. The Phase Runner gets its own fresh 200K context and handles EVERYTHING for this phase.

```
Agent(
  prompt="First, read ./.claude/agents/gsd-yolo-phase-runner.md for your full role and instructions.\n\n" +
    "<phase>\n" +
    "  <number>{N}</number>\n" +
    "  <name>{phase_name}</name>\n" +
    "  <dir>{phase_dir}</dir>\n" +
    "  <phase_index>{current.phase_index}</phase_index>\n" +
    "</phase>\n\n" +
    "<state_file>.planning/YOLO-STATE.json</state_file>\n\n" +
    "<instruction>\n" +
    "Execute the complete GSD lifecycle for Phase {N}: {phase_name}.\n" +
    "Read YOLO-STATE.json for your config, models, and workflow settings.\n" +
    "Run: discuss → plan → execute → verify → debug (if needed) → finalize.\n" +
    "Confidence gates are active: check for conflicts with prior decisions at each step.\n" +
    "If LOW confidence detected: write ESCALATION.md, set status to needs_input, return.\n" +
    "Update YOLO-STATE.json with your results before returning.\n" +
    "Return ONE LINE: PHASE {N} {STATUS}: {tasks} tasks, {commits} commits, {debug_cycles} debug cycles\n" +
    "</instruction>",
  subagent_type="general-purpose",
  model="opus",
  description="YOLO Phase {N}: {phase_name}"
)
```

**5. After Phase Runner returns — read result from disk (NOT from return message):**

```bash
cat .planning/YOLO-STATE.json
```

Read ONLY `phases[current.phase_index].status` to determine result.

**6. Handle status:**

**If `"complete"`:**
- Increment `stats.phases_completed`, add commits/debug_cycles to totals
- Log: `Phase {N}: {phase_name} — COMPLETE ({commits} commits, {debug_cycles} debug cycles)`
- Increment `current.phase_index`, loop

**If `"blocked"`:**
- Increment `stats.phases_blocked`, add debug_cycles to totals
- Log: `Phase {N}: {phase_name} — BLOCKED (see {phase_num}-YOLO-BLOCKED.md)`
- If `config.continue_on_block` is true: increment `current.phase_index`, loop
- If false: stop and report

**If `"needs_input"` — ESCALATION HANDLING:**

This is the confidence gate path. The Phase Runner found a LOW confidence decision that needs human input.

a) Read the escalation step from YOLO-STATE.json: `phases[current.phase_index].escalation_step`

b) Read ONLY the ESCALATION.md file (small, structured):
```bash
cat .planning/phases/{phase_dir}/{phase_num}-ESCALATION.md
```

c) Present to user via AskUserQuestion. Format:
```
Phase {N} ({phase_name}) needs your input before proceeding.

The {escalation_step} step found decisions that conflict with prior project direction:

{paste ESCALATION.md content here}

For each item, tell me which option you prefer (a/b/c), or provide your own direction.
You can also say "auto" to let the agent use its auto-inferred preference for all items.
```

d) Write the user's response to DECISION.md:
```bash
# Write .planning/phases/{phase_dir}/{phase_num}-DECISION.md
```
Format:
```markdown
---
phase: {N}
step: {escalation_step}
timestamp: {ISO}
---

## Human Decisions

{user's response, verbatim}
```

e) Increment `stats.phases_escalated`

f) Log: `Phase {N}: {phase_name} — ESCALATED at {step}, user input received. Re-spawning runner.`

g) Re-spawn the Phase Runner (it will detect `needs_input` status + DECISION.md and resume):

```
Agent(
  prompt="First, read ./.claude/agents/gsd-yolo-phase-runner.md for your full role and instructions.\n\n" +
    "<phase>\n" +
    "  <number>{N}</number>\n" +
    "  <name>{phase_name}</name>\n" +
    "  <dir>{phase_dir}</dir>\n" +
    "  <phase_index>{current.phase_index}</phase_index>\n" +
    "</phase>\n\n" +
    "<state_file>.planning/YOLO-STATE.json</state_file>\n\n" +
    "<instruction>\n" +
    "RESUME Phase {N}: {phase_name} after human escalation.\n" +
    "Your status is needs_input. Read DECISION.md for the user's choices.\n" +
    "Resume from after the {escalation_step} step, incorporating the human decisions.\n" +
    "Continue through the remaining lifecycle steps.\n" +
    "Return ONE LINE: PHASE {N} {STATUS}: {tasks} tasks, {commits} commits, {debug_cycles} debug cycles\n" +
    "</instruction>",
  subagent_type="general-purpose",
  model="opus",
  description="YOLO Phase {N} (resumed): {phase_name}"
)
```

h) After resumed runner returns: go back to step 5 (read status from disk). The runner may complete, block, or escalate again (different step). Handle accordingly.

</step>

<step name="completion">

When all phases have been processed (no `"pending"` entries in YOLO-STATE.json):

**1. Read final stats from YOLO-STATE.json:**

```bash
cat .planning/YOLO-STATE.json
```

Extract `stats.*` and each phase's status.

**2. Spawn report generator subagent** (keeps report logic out of supervisor context):

```
Agent(
  prompt="Generate the final YOLO execution report.\n\n" +
    "<files_to_read>\n" +
    "- .planning/YOLO-STATE.json (phase results and stats)\n" +
    "- .planning/phases/*-YOLO-BLOCKED.md (if any exist)\n" +
    "- .planning/phases/*/CONTEXT.md (for auto-inferred decisions list)\n" +
    "- .planning/phases/*/*-ATTENTION.md (for medium-confidence items to review)\n" +
    "- .planning/phases/*/*-DECISION.md (for escalation history)\n" +
    "</files_to_read>\n\n" +
    "<instruction>\n" +
    "Write .planning/YOLO-REPORT.md with sections:\n" +
    "- Overview: total phases, completed, blocked, escalated, commits, debug cycles\n" +
    "- Phase Results table: Phase | Status | Tasks | Commits | Debug Cycles | Escalated?\n" +
    "- Blocked Phases (if any): references to YOLO-BLOCKED.md files\n" +
    "- Escalation History (if any): what was escalated, user decisions made\n" +
    "- Medium Confidence Items for Review: aggregated from all ATTENTION.md files\n" +
    "- Auto-Inferred Decisions: CONTEXT.md files with [AUTO-INFERRED] tags\n" +
    "- Next Steps: /gsd:verify-work, /gsd:audit-milestone, /gsd:complete-milestone\n" +
    "</instruction>",
  subagent_type="general-purpose",
  model="opus",
  description="YOLO final report"
)
```

**3. Final state update:**
Update YOLO-STATE.json: set top-level `status` to `"complete"`, add `completed_at` timestamp.

**4. Report to user (derived from YOLO-STATE.json stats, already loaded):**

```
YOLO COMPLETE

{phases_completed}/{total_phases} phases completed successfully
{phases_blocked} phases need manual review
{phases_escalated} phases required human input during execution
{total_commits} total commits made
{total_debug_cycles} debug cycles used

{if phases_blocked > 0:}
Blocked phases:
  {for each blocked phase:}
  - Phase {N}: {name} — see {phase_num}-YOLO-BLOCKED.md
{end if}

{if any ATTENTION.md files exist:}
Medium-confidence items logged for your review — see YOLO-REPORT.md
{end if}

Full report: .planning/YOLO-REPORT.md
Run /gsd:audit-milestone to verify milestone goals.
```

</step>

</process>

<context_budget>
## Projected Token Usage (9-Phase Run)

### Supervisor (this workflow)

| Source | Tokens per Phase | Total (9 phases) |
|--------|-----------------|-------------------|
| YOLO-STATE.json read (1 per phase) | ~800 | ~7,200 |
| Phase Runner spawn prompt | ~500 | ~4,500 |
| Phase Runner return message | ~200 | ~1,800 |
| YOLO-STATE.json write (1 per phase) | ~400 | ~3,600 |
| Log line | ~50 | ~450 |
| Batch boundary logging | — | ~500 |
| Escalation handling (if triggered) | ~2,000 | ~4,000 (est. 2 escalations) |
| **Per-phase subtotal** | **~1,950** | — |

| Fixed Costs | Tokens |
|-------------|--------|
| Workflow prompt (this file) | ~5,000 |
| Prerequisites (config + ROADMAP read) | ~5,000 |
| Completion + report spawn | ~2,000 |
| **Fixed subtotal** | **~12,000** |

**Total supervisor: ~12,000 + (1,950 x 9) + ~4,000 escalation = ~33,550 tokens**

Available headroom: 200K - 34K = ~166K tokens of safety margin.

### Phase Runner (per phase)
Each gets a fresh 200K window. Typical usage:
- Agent instructions (read from disk): ~3,000
- Context files: ~10,000
- 6-8 subagent spawns + returns: ~15,000
- State reads/writes: ~3,000
- Decision logic: ~5,000
- **Total: ~36,000** (18% of 200K — massive headroom)

### Worst Case (debug loops + escalation)
- Phase Runner with 2 debug cycles: ~50,000 tokens (25% of 200K)
- Still well within budget.

</context_budget>

<batch_boundaries>
## Batch Boundaries (Lightweight Tracking)

After every `phases_per_batch` phases (default: 4), the supervisor logs a batch boundary and continues. This provides:

1. **Auditability** — batch transitions logged and tracked in YOLO-STATE.json
2. **State continuity** — YOLO-STATE.json carries all accumulated results
3. **Unlimited phase capacity** — projects with 20+ phases work fine (supervisor uses ~2K tokens/phase)
4. **Crash recovery** — re-running `/gsd:yolo` detects YOLO-STATE.json and resumes
5. **Flat nesting** — supervisor stays at one level, Phase Runners at two, GSD agents at three

The agent tree (constant depth regardless of phase count):
```
Main conversation
  └── YOLO Supervisor (all batches — single agent)
        ├── Phase Runner (phase 1)  ─── GSD agents
        ├── Phase Runner (phase 2)  ─── GSD agents
        ├── --- batch boundary ---
        ├── Phase Runner (phase 5)  ─── GSD agents
        └── ...
```

**Why no self-re-invocation:** The original design spawned a continuation supervisor as a subagent, creating nested supervisors (Main → Supervisor → Continuation Supervisor → Phase Runner → GSD agent = 4+ levels). This exceeded Claude Code's practical subagent depth limit, causing `[Tool result missing due to internal error]` on batch 2+. Since the supervisor only uses ~2K tokens per phase (~30K for 9 phases, ~50K for 20 phases), it has ample headroom within a single 200K context window and does not need a hard context reset.
</batch_boundaries>

<guardrails>

1. **Supervisor never reads phase files:** No ROADMAP.md (after init), no REQUIREMENTS.md, no PLANs, no SUMMARYs, no source code. Only YOLO-STATE.json and ESCALATION.md (when escalated).
2. **Phase Runners are disposable:** Each gets fresh 200K. If one fails catastrophically, the state on disk tells the supervisor what happened.
3. **Confidence gates prevent silent conflicts:** LOW confidence items always escalate to human before proceeding. MEDIUM items are logged for later review.
4. **Escalation is lightweight:** Supervisor reads only the ESCALATION.md (small structured file), asks user, writes DECISION.md, re-spawns runner. Adds ~2K tokens per escalation.
5. **Atomic commits:** Every task is independently revertable.
6. **Full audit trail:** Every decision, plan, execution, debug attempt, escalation, and user decision documented in files.
7. **Auto-inferred decisions marked:** [AUTO-INFERRED] tags for human review.
8. **Git safety:** No force pushes, no history rewrites, no destructive operations.
9. **Dependency awareness:** Blocked dependencies cascade to dependent phases.
10. **Model enforcement:** All models from `.planning/config.json` model_overrides. Default: `opus`.
11. **Flat nesting:** Supervisor → Phase Runner → GSD agents (max 3 levels, never deeper).
12. **Resume from crash:** YOLO-STATE.json + phase artifacts = full recovery. Escalation state persists across crashes.

</guardrails>

<configuration>

Settings in `.planning/config.json`:

```json
{
  "mode": "yolo",
  "yolo_max_debug_retries": 2,
  "yolo_skip_discuss": false,
  "yolo_continue_on_block": true,
  "yolo_phases_per_batch": 4,
  "workflow": {
    "research": true,
    "plan_check": true,
    "verifier": true
  }
}
```

| Setting | Default | Description |
|---------|---------|-------------|
| `yolo_max_debug_retries` | `2` | Max debug cycles per phase before flagging |
| `yolo_skip_discuss` | `false` | Skip auto-discuss step |
| `yolo_continue_on_block` | `true` | Continue to next phase when one is blocked |
| `yolo_phases_per_batch` | `4` | Phases per batch (for logging and state tracking) |
| `workflow.research` | `true` | Run phase research before planning |
| `workflow.plan_check` | `true` | Run plan verification after planning |
| `workflow.verifier` | `true` | Run phase verification after execution |

</configuration>
