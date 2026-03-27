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
| `/m m` or `/m manage` | Spawn subagents for steps 1-5 sequentially. Write to `manager_step_based.md`. |
| `/m s` or `/m send` | Spawn subagent for step 6. Send plan to dev, monitor. |
| `/m auto` | Steps 1-6 via subagents. Auto-send if conditions met. Loop until idle. |
| `/m all` | Start recurring `/m auto` via CronCreate. Default 10m. |
| `/m stop` | Cancel loop via CronDelete. |
| `/m 1` ... `/m 6` | Run single step as subagent. |

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

### `/m m` (manage) — Steps 1-5 as sequential subagents

**IMPORTANT**: Each step MUST be spawned as a separate Agent tool call. Do NOT attempt to answer the step questions yourself. The subagent does the deep work and writes its own output file.

Execute these 5 Agent calls **sequentially** (each depends on the previous):

**Step 1** — Spawn Agent:
```
prompt: "You are the RLQuest manager executing Step 1: Ground Truth & State Assessment.
Read /home/ubuntu/workspace/RLQuest-manager/.claude/skills/manager/step1_ground_truth.md for your full instructions.
Execute every check described. Use bash commands to verify filesystem, processes, GPU, dev tmux pane.
Write your complete findings to /home/ubuntu/workspace/RLQuest-manager/step1_output.md (overwrite).
Include ALL bash output and evidence in the file. Do not summarize — write everything."
```

**Step 2** — Spawn Agent:
```
prompt: "You are the RLQuest manager executing Step 2: Gap Analysis & Constraint Identification.
Read /home/ubuntu/workspace/RLQuest-manager/.claude/skills/manager/step2_gap_constraint.md for your full instructions.
Read Step 1 findings from /home/ubuntu/workspace/RLQuest-manager/step1_output.md.
Read /home/ubuntu/workspace/RLQuest-manager/goals.md for targets.
Write your complete analysis to /home/ubuntu/workspace/RLQuest-manager/step2_output.md (overwrite).
Include ALL evidence and reasoning. Do not summarize — write everything."
```

**Step 3** — Spawn Agent:
```
prompt: "You are the RLQuest manager executing Step 3: Design & Architecture Review.
Read /home/ubuntu/workspace/RLQuest-manager/.claude/skills/manager/step3_design_review.md for your full instructions.
This is the MOST CRITICAL step. Do NOT give surface-level answers. You MUST:
- Read the ACTUAL CODE: model.py, config.py, data_loader.py, prepare_tokens.py in /home/ubuntu/workspace/RLQuest/firstrate_learning_v5/
- Load an actual data chunk with python and inspect: token value distributions, NaN/inf counts, min/max per dimension, day_mask distribution
- Compute actual param count by loading the model
- Calculate params/sample ratio and compare to V3 (0.023, no overfit) and V4 (0.82, overfit epoch 2)
- Verify all 20 token dimensions are populated (not zeros)
- Check the loss function components and weights — are they appropriate for this model size?
- Read research docs in /home/ubuntu/workspace/RLQuest-manager/.claude/skills/manager/research/
- Read Step 1 and Step 2 from /home/ubuntu/workspace/RLQuest-manager/step1_output.md and step2_output.md
Write your COMPLETE deep analysis to /home/ubuntu/workspace/RLQuest-manager/step3_output.md (overwrite).
Include ALL code inspection results, bash outputs, and computed values. This file should be long and detailed."
```

**Step 4** — Spawn Agent:
```
prompt: "You are the RLQuest manager executing Step 4: Learn & Update.
Read /home/ubuntu/workspace/RLQuest-manager/.claude/skills/manager/step4_learn_update.md for your full instructions.
Read Steps 1-3 from /home/ubuntu/workspace/RLQuest-manager/step1_output.md, step2_output.md, step3_output.md.
Reflect on what worked, what failed, what was missed in previous management cycles.
Update /home/ubuntu/workspace/RLQuest-manager/.claude/skills/manager/goal_tracker.md with current state from steps 1-3.
If dev skills/rules need improvement based on findings, update files in /home/ubuntu/workspace/RLQuest/.claude/rules/ or /home/ubuntu/workspace/RLQuest/CLAUDE.md.
Write your complete findings to /home/ubuntu/workspace/RLQuest-manager/step4_output.md (overwrite).
Include what was updated and why."
```

**Step 5** — Spawn Agent:
```
prompt: "You are the RLQuest manager executing Step 5: Action Planning & Validation Gate.
Read /home/ubuntu/workspace/RLQuest-manager/.claude/skills/manager/step5_action_plan.md for your full instructions.
Read ALL previous steps: step1_output.md, step2_output.md, step3_output.md, step4_output.md in /home/ubuntu/workspace/RLQuest-manager/.
Pay special attention to Step 3 design review — if it found issues, the plan MUST address them before proceeding.
Enumerate all possible actions. Check all resources (GPU, CPU, dev).
Run the validation gate on every recommended action.
Write your complete plan to /home/ubuntu/workspace/RLQuest-manager/step5_output.md (overwrite).
End with a clear '## Recommended Prompt' section containing the EXACT prompt to send to dev."
```

After all 5 subagents complete, report to the user: "Steps 1-5 complete. Review step1-5_output.md files. Send with /m s."

### `/m s` (send) — Step 6 as subagent

Spawn Agent:
```
prompt: "You are the RLQuest manager executing Step 6: Execute & Monitor.
Read /home/ubuntu/workspace/RLQuest-manager/.claude/skills/manager/step6_execute_monitor.md for your full instructions.
Read the recommended prompt from /home/ubuntu/workspace/RLQuest-manager/step5_output.md (the '## Recommended Prompt' section).
Send it to dev via /home/ubuntu/workspace/RLQuest/tmux_send_claude.sh 1 '<prompt>'.
If it shows as 'Pasted text', send Enter: /home/ubuntu/workspace/RLQuest/tmux_send_claude.sh 1 Enter.
Monitor dev actively. Intervene on deviations. Chain next task on completion.
When dev finishes, report back with results."
```

### `/m auto` — Full cycle with auto-send

Run `/m m` (steps 1-5 as subagents). Then read `step5_output.md` and check auto-send conditions:

**Auto-send** (proceed to step 6 without user):
- Dev is idle and there's a clear next task
- Task is operational (training, data prep, implementation)
- Step 5 validation gate passed all checks
- No strategic pivot or high-cost action flagged

**Pause** (report to user, don't send):
- Strategic pivot needed
- Constraint changed fundamentally
- Cost > 24hr without validation
- Step 5 flagged LOW confidence

If auto-send: run `/m s` (step 6 subagent). After dev completes, repeat from `/m m`.

### `/m all` — Recurring loop

Run `/m auto` immediately. Then use CronCreate to schedule `/m auto` recurring (default 10 min). User specifies: `/m all 5m`, `/m all 30m`.

### `/m stop` — Cancel loop

Use CronDelete to cancel the scheduled task.

### `/m 1` through `/m 6` — Single step deep dive

Spawn the corresponding subagent only. It writes to its own `stepN_output.md` file. Useful when the user wants to deep-dive into one area without running the full pipeline.

## Paths

- **RLQuest directory:** `/home/ubuntu/workspace/RLQuest`
- **Tmux scripts:** `tmux_run_claude.sh`, `tmux_send_claude.sh`, `tmux_view_claude.sh` in RLQuest dir
- **PID file:** `/home/ubuntu/workspace/RLQuest/claude_session_PID_1`
- **Output file:** `/home/ubuntu/workspace/RLQuest-manager/manager_step_based.md`
- **Step instructions:** `.claude/skills/manager/step1_ground_truth.md` through `step6_execute_monitor.md`
- **Goal tracker:** `.claude/skills/manager/goal_tracker.md`
- **Research docs:** `.claude/skills/manager/research/`
- **Project goals:** `/home/ubuntu/workspace/RLQuest-manager/goals.md`
- **Dev direction:** `/home/ubuntu/workspace/RLQuest/firstrate_learning/direction.md`
