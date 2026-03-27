---
name: m
description: RLQuest Manager — 6-step deep-dive management via subagents
disable-model-invocation: true
---

# Manager — RLQuest Dev Supervisor

Manages the Claude dev tmux session (instance 1) in `/home/ubuntu/workspace/RLQuest` through 6 independent deep-dive steps, each executed as a **separate subagent** with isolated context for maximum analysis depth.

## Why Subagents

Each step runs as its own Agent with only that step's instruction file as its task. This prevents the "rush to completion" problem where Claude sees all 6 steps and gives each one surface-level treatment. Each subagent gets full context depth for its investigation.

## Commands

| Command | What It Does |
|---------|-------------|
| `/m m` or `/m manage` | Spawn subagents for steps 1-5 sequentially. Write to step output files. |
| `/m s` or `/m send` | Spawn subagent for step 6. Send plan to dev, monitor. |
| `/m auto` | Continuous loop: steps 1-6 via subagents, loops back to 1 after 6 completes. Runs until pause condition or user interrupt. |
| `/m stop` | User interrupts the running auto loop. |
| `/m 1` ... `/m 6` | Run single step as subagent for deep dive. |

## Output Files

Each subagent writes its own output file. This preserves the full depth of analysis (the Agent return message is only a summary).

| Step | Output File |
|------|-------------|
| 1 | `/home/ubuntu/workspace/RLQuest-manager/step1_output.md` |
| 2 | `/home/ubuntu/workspace/RLQuest-manager/step2_output.md` |
| 3 | `/home/ubuntu/workspace/RLQuest-manager/step3_output.md` |
| 4 | `/home/ubuntu/workspace/RLQuest-manager/step4_output.md` |
| 5 | `/home/ubuntu/workspace/RLQuest-manager/step5_output.md` |
| 6 | (no file — step 6 sends to dev and monitors) |

Step 5 also writes a `## Recommended Prompt` section in its output that step 6 reads.

## How to Execute Each Mode

### `/m m` (manage) — Steps 1-5 as sequential subagents with orchestrator context injection

**IMPORTANT**: Each step MUST be spawned as a separate Agent tool call. Do NOT attempt to answer the step questions yourself. The subagent does the deep work and writes its own output file.

**ORCHESTRATOR CONTEXT INJECTION**: After each subagent returns, the orchestrator reviews the return summary. If the return contains critical findings, warnings, or discoveries, the orchestrator PREPENDS them as `ORCHESTRATOR CONTEXT:` in the next subagent's prompt. This ensures critical information is highlighted even if the next subagent doesn't thoroughly read the output files.

Execute these 5 Agent calls **sequentially**. Between each call, review the agent's return message and build context for the next.

**Step 1** — Spawn Agent:
```
prompt: "You are the RLQuest manager executing Step 1: Ground Truth & State Assessment.
Read /home/ubuntu/workspace/RLQuest-manager/.claude/skills/manager/step1_ground_truth.md for your full instructions.
FIRST read your own previous output from /home/ubuntu/workspace/RLQuest-manager/step1_output.md if it exists — use its Persistent Notes for context about project structure and last known state. Then verify current reality with bash commands. If nothing changed from persistent notes, keep it brief. If things changed, investigate fully.
Write findings to /home/ubuntu/workspace/RLQuest-manager/step1_output.md (overwrite). Include a Persistent Notes section at the bottom."
```
After Step 1 returns: Read the return summary. Note key state facts (GPU idle? training running? dev idle? data complete?). Build `step1_context` string.

**Step 2** — Spawn Agent:
```
prompt: "ORCHESTRATOR CONTEXT from Step 1: {step1_context}

You are the RLQuest manager executing Step 2: Gap Analysis & Constraint Identification.
Read /home/ubuntu/workspace/RLQuest-manager/.claude/skills/manager/step2_gap_constraint.md for your full instructions.
FIRST read your own previous output from /home/ubuntu/workspace/RLQuest-manager/step2_output.md if it exists — use its Persistent Notes for previous hypothesis, constraint, and gap trend. If constraint unchanged, confirm briefly. If shifted, analyze fully.
Read Step 1 from /home/ubuntu/workspace/RLQuest-manager/step1_output.md. Read /home/ubuntu/workspace/RLQuest-manager/goals.md for targets.
Write to /home/ubuntu/workspace/RLQuest-manager/step2_output.md (overwrite). Include Persistent Notes."
```
After Step 2 returns: Note constraint, hypothesis, gap size. Build `step2_context` string. Accumulate: `accumulated_context = step1_context + step2_context`.

**Step 3** — Spawn Agent:
```
prompt: "ORCHESTRATOR CONTEXT from Steps 1-2: {accumulated_context}

You are the RLQuest manager executing Step 3: Design & Architecture Review.
Read /home/ubuntu/workspace/RLQuest-manager/.claude/skills/manager/step3_design_review.md for your full instructions.
FIRST read your own previous output from /home/ubuntu/workspace/RLQuest-manager/step3_output.md if it exists — use its Persistent Notes for cached analysis (param counts, token dims, loss weights, file timestamps, known issues). Check if source files changed since last cycle by comparing timestamps. If nothing changed: reuse cache, write brief delta, carry forward notes. If files changed or new training results exist: re-investigate only changed parts.
Read Step 1 and Step 2 from /home/ubuntu/workspace/RLQuest-manager/step1_output.md and step2_output.md.
Read research docs in /home/ubuntu/workspace/RLQuest-manager/research/.
Write to /home/ubuntu/workspace/RLQuest-manager/step3_output.md (overwrite). Include detailed Persistent Notes as cache for next cycle."
```
After Step 3 returns: **CRITICAL** — check for design issues, warnings, blockers in the return. Flag any with `[CRITICAL]`, `[WARNING]`, `[ISSUE]`. Build `step3_context`. Accumulate.

**Step 4** — Spawn Agent:
```
prompt: "ORCHESTRATOR CONTEXT from Steps 1-3: {accumulated_context}
{If step3 found critical issues: 'CRITICAL FINDINGS FROM STEP 3 — address these in your reflection: [list]'}

You are the RLQuest manager executing Step 4: Learn & Update.
Read /home/ubuntu/workspace/RLQuest-manager/.claude/skills/manager/step4_learn_update.md for your full instructions.
Read Steps 1-3 from /home/ubuntu/workspace/RLQuest-manager/step1_output.md, step2_output.md, step3_output.md.
Reflect on what worked, what failed, what was missed in previous management cycles.
Update ONLY the DATA in /home/ubuntu/workspace/RLQuest-manager/goal_tracker.md (metrics, status, hypothesis, constraint).
CRITICAL — DO NOT modify any of these files:
  - /home/ubuntu/workspace/RLQuest/.claude/ (rules, settings, skills — any file)
  - /home/ubuntu/workspace/RLQuest/CLAUDE.md
  - /home/ubuntu/workspace/RLQuest-manager/.claude/skills/manager/SKILL.md
  - /home/ubuntu/workspace/RLQuest-manager/.claude/skills/manager/step*.md
Instead:
  - APPEND dev improvement recommendations to /home/ubuntu/workspace/RLQuest-manager/future_dev_improvements.md (create if missing, always APPEND).
  - APPEND manager process improvements to /home/ubuntu/workspace/RLQuest-manager/future_manager_improvements.md (create if missing, always APPEND).
Write your complete findings to /home/ubuntu/workspace/RLQuest-manager/step4_output.md (overwrite). Include Persistent Notes."
```
After Step 4 returns: Note any learnings, updated beliefs, process gaps. Build `step4_context`. Accumulate.

**Step 5** — Spawn Agent:
```
prompt: "ORCHESTRATOR CONTEXT from Steps 1-4: {accumulated_context}
{If any step found critical issues: 'MANDATORY — your plan MUST address these before any other action: [list of critical/warning items from steps 1-4]'}

You are the RLQuest manager executing Step 5: Action Planning & Validation Gate.
Read /home/ubuntu/workspace/RLQuest-manager/.claude/skills/manager/step5_action_plan.md for your full instructions.
FIRST read your own previous output from /home/ubuntu/workspace/RLQuest-manager/step5_output.md if it exists — use its Persistent Notes for action history, what's pending, and prompt patterns that worked. If previous action is still in progress, confirm and exit quickly.
Read ALL previous steps: step1_output.md, step2_output.md, step3_output.md, step4_output.md in /home/ubuntu/workspace/RLQuest-manager/.
Write to /home/ubuntu/workspace/RLQuest-manager/step5_output.md (overwrite). End with '## Recommended Prompt' section. Include Persistent Notes."
```

After all 5 subagents complete, report to the user: "Steps 1-5 complete. Review step output files. Send with /m s."

### How the Orchestrator Builds Context

After each Agent returns, the orchestrator:
1. Reads the agent's return message (the summary that comes back from the Agent tool).
2. Extracts key facts into a concise context string (2-4 sentences max).
3. Flags anything marked CRITICAL, WARNING, or ISSUE.
4. Prepends the accumulated context to the next agent's prompt.

Example accumulated context by Step 5:
```
ORCHESTRATOR CONTEXT from Steps 1-4:
Step 1: Dev idle 3+ hours. GPU 0%. V5 data complete (64/64, 11.4M samples). V5 train.py exists, unit test passed.
Step 2: Constraint is EXECUTION. Gap: 1.9x improvement needed. Hypothesis: V5-Small temporal architecture will achieve CR >= 0.0147.
Step 3: [DELTA — no code changes] 679K params, 0.117 params/sample ratio (safe). Known issues: summary token zeros, price dim 15 bug. No blockers.
Step 4: Goal tracker updated. No new process gaps. Cycle count: 2.
MANDATORY — no critical issues found. Proceed with execution plan.
```

This is lightweight (~200 tokens) but ensures Step 5 has the full picture without relying solely on file reads.

### `/m s` (send) — Step 6 as subagent

The orchestrator reads `step5_output.md` to extract key context about the plan, then injects it into Step 6's prompt.

Spawn Agent:
```
prompt: "ORCHESTRATOR CONTEXT: {summary of the plan from step5_output.md — what action, why, success/failure criteria}
{If any critical items were flagged in steps 1-5: 'WATCH FOR: [items]'}

You are the RLQuest manager executing Step 6: Execute & Monitor.
Read /home/ubuntu/workspace/RLQuest-manager/.claude/skills/manager/step6_execute_monitor.md for your full instructions.
FIRST read your own previous output from /home/ubuntu/workspace/RLQuest-manager/step6_output.md if it exists — use its Persistent Notes for dev behavior patterns, effective prompt patterns, and monitoring observations.
Read the recommended prompt from /home/ubuntu/workspace/RLQuest-manager/step5_output.md (the '## Recommended Prompt' section).
Send it to dev via /home/ubuntu/workspace/RLQuest/tmux_send_claude.sh 1 '<prompt>'.
If it shows as 'Pasted text', send Enter: /home/ubuntu/workspace/RLQuest/tmux_send_claude.sh 1 Enter.
Monitor dev actively. Intervene on deviations. When dev finishes, write results to /home/ubuntu/workspace/RLQuest-manager/step6_output.md (overwrite) with Persistent Notes, then report back."
```

### `/m auto` — Continuous autonomous loop (no cron needed)

The orchestrator runs an **indefinite loop** of steps 1-6. No external timer. The parent stays active, spawning subagents sequentially:

```
LOOP (runs until pause condition hit or user interrupts):
  1. Spawn Step 1 subagent → wait for completion
  2. Spawn Step 2 subagent → wait for completion
  3. Spawn Step 3 subagent → wait for completion
  4. Spawn Step 4 subagent → wait for completion
  5. Spawn Step 5 subagent → wait for completion
  6. Read step5_output.md — check auto-send conditions
     - If PAUSE: report to user, exit loop
     - If AUTO-SEND: spawn Step 6 subagent → wait for completion
  7. After Step 6 returns → loop back to Step 1
```

**Auto-send conditions** (proceed to step 6):
- Dev is idle and there's a clear next task
- Task is operational (training, data prep, implementation)
- Step 5 validation gate passed all checks
- No strategic pivot or high-cost action flagged

**Pause conditions** (exit loop, report to user):
- Strategic pivot needed
- Constraint changed fundamentally
- Cost > 24hr without validation
- Step 5 flagged LOW confidence
- Step 6 subagent reports dev encountered unrecoverable error

The loop continues indefinitely: after Step 6 completes (dev finished the task), the orchestrator goes back to Step 1 to assess the new state, plan the next action, and send it. No idle time between cycles.

**Context management**: Each subagent has fresh context. The parent accumulates only the summary return messages (~1-2 paragraphs each). After many cycles, the parent's context may compress, but subagent analysis quality is unaffected.

### `/m stop`

The user types `/m stop` or interrupts the current operation. Since there's no cron job, stopping is just interrupting the running `/m auto` invocation.

### `/m 1` through `/m 6` — Single step deep dive

Spawn the corresponding subagent only. It writes to its own `stepN_output.md` file. Useful when the user wants to deep-dive into one area without running the full pipeline.

## Paths

- **RLQuest directory:** `/home/ubuntu/workspace/RLQuest`
- **Tmux scripts:** `tmux_run_claude.sh`, `tmux_send_claude.sh`, `tmux_view_claude.sh` in RLQuest dir
- **PID file:** `/home/ubuntu/workspace/RLQuest/claude_session_PID_1`
- **Output file:** `/home/ubuntu/workspace/RLQuest-manager/manager_step_based.md`
- **Step instructions:** `.claude/skills/manager/step1_ground_truth.md` through `step6_execute_monitor.md`
- **Goal tracker:** `/home/ubuntu/workspace/RLQuest-manager/goal_tracker.md`
- **Research docs:** `/home/ubuntu/workspace/RLQuest-manager/research/`
- **Future dev improvements:** `/home/ubuntu/workspace/RLQuest-manager/future_dev_improvements.md`
- **Future manager improvements:** `/home/ubuntu/workspace/RLQuest-manager/future_manager_improvements.md`
- **Project goals:** `/home/ubuntu/workspace/RLQuest-manager/goals.md`
- **Dev direction:** `/home/ubuntu/workspace/RLQuest/firstrate_learning/direction.md`
