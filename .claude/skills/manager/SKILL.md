---
name: m
description: Ensure RLQuest Claude tmux session 1 is running and has an active conversation
disable-model-invocation: true
---

# Manager — RLQuest Claude Session Monitor

Ensure the RLQuest Claude tmux session (instance 1) is running and has an active conversation.

## Terminology

- **"dev"** refers to the Claude tmux session instance 1. When the user says "dev", they mean this session.
- **"manage"** (also **"mng"** or **"m"**) — when the user says any of these words (with no other arguments), the manager must:
  1. Run the full procedure (Steps 1-4) to ensure dev is running
  2. Capture dev's current state from the tmux pane
  3. Apply the decision tree (`decision_tree.md`) to analyze dev's situation
  4. Write/overwrite a file at `/home/ubuntu/workspace/RLQuest-manager/manager.md` containing the manager's analysis and recommended prompt/instructions for dev
  5. Do NOT send anything to dev — only output to `manager.md`. The user will decide whether to send it.

  The `manager.md` file serves as the manager's review report. It should include:
  - Timestamp of the review
  - What was observed in dev's session
  - Decision tree outcome (what branch was taken)
  - The recommended prompt/instruction for dev (ready to copy-paste or send)
  - Any config changes made (if compliance issues were found)

- **"send"** (also **"s"**) — when the user says "send" or "s":
  1. Read the current `manager.md` and extract the recommended prompt from the "Recommended Prompt to Dev" section
  2. Build a **checkpoint expectations list** from the prompt — the specific milestones dev should hit (e.g., "add --smoke-test flag", "implement parallelization", "run smoke test", "measure CPU"). Write this list to `manager.md` under a "## Checkpoint Expectations" section.
  3. Send the prompt to dev via `tmux_send_claude.sh 1 "<prompt>"`. Note: the send script sends text + Enter, but Claude Code may show it as "Pasted text". If a subsequent check shows the prompt sitting unsent at the input, send an additional `Enter` via `tmux_send_claude.sh 1 Enter` to submit it.
  4. Enter the **active monitoring loop** (see below)
  5. When dev finishes (returns to idle prompt or reports completion), do a final full review and provide status summary

  **Active Monitoring Loop (full autonomy — always intervene):**

  The manager monitors dev and **actively intervenes** when deviations are detected. This is not passive observation — the manager is an autonomous supervisor.

  **Phase 1: Initial Verification (first 2-3 checks, every 10-15 seconds)**
  - Full pane capture (30+ lines) to confirm dev received and started working on the prompt
  - Verify dev is addressing the instructions in order
  - If dev ignores the prompt or goes off-track immediately → send correction via `tmux_send_claude.sh`
  - Apply the **Unexpected Dev Behavior** branch from `decision_tree.md` on every check

  **Phase 2: Confident Monitoring (dev is on track)**
  - Scale back check frequency based on estimated task duration:
    - Tasks <5 min: check every 30-60 seconds
    - Tasks 5-30 min: check every 2-5 minutes
    - Tasks 30+ min: check every 5-10 minutes
  - Use lightweight checks (5-line tail, `ps aux`) for routine checks
  - Do full pane capture when checking off a milestone from the expectations list
  - **On every check, evaluate against checkpoint expectations:**
    - Is dev progressing through the expected milestones?
    - Has dev skipped a milestone? → Intervene immediately
    - Is dev doing something unexpected? → Full pane capture, apply decision tree, intervene if needed

  **Phase 3: Near Completion**
  - Increase frequency to catch the finish
  - Full pane capture to review final results
  - Verify all checkpoint expectations were met
  - If dev declares "done" but milestones are missing → send correction

  **Intervention Protocol:**

  When the manager detects a deviation during monitoring:

  1. **Assess severity:**
     - **Minor** (wrong order, slightly off approach): send a brief nudge via `tmux_send_claude.sh 1 "Note: please also [missing step]. This is required per project standards."`
     - **Major** (skipped critical step like smoke test, no parallelization): send a firm correction via `tmux_send_claude.sh 1 "STOP. You skipped [step]. This is mandatory. Please [specific action] before continuing."`
     - **Critical** (about to overwrite data without backup, about to run full dataset without validation): send immediate stop via `tmux_send_claude.sh 1 "STOP IMMEDIATELY. Do not proceed. [reason]. You must [required action] first."`

  2. **After intervening:**
     - Wait 10-15 seconds, then check if dev acknowledged and adjusted
     - If dev ignores the intervention, escalate: send again with more detail
     - If dev ignores twice, report to user and pause monitoring

  3. **Apply decision tree compliance review** (from `decision_tree.md` "Unexpected Dev Behavior" section):
     - Identify the violation
     - Diagnose root cause (rule missing? rule vague? dev context issue?)
     - If config gap found: fix the RLQuest `.claude/` config files immediately
     - Send correction to dev

  4. **Update `manager.md`** after any intervention:
     - Log what was detected, what was sent, and dev's response
     - Update checkpoint expectations status

  5. **Continuous improvement tracking** — during ALL monitoring (manage, send, and any dev interaction), continuously update `/home/ubuntu/workspace/RLQuest-manager/manager_improvements.md` with:
     - What went well (dev behaviors to reinforce)
     - Gaps identified (in manager skills, dev skills/rules, or workflows)
     - Improvement candidates (specific proposed fixes with priority)
     - Do NOT modify skills/rules directly from this — only log observations. The user will decide which improvements to implement.

  **Cost Reduction Techniques:**
  - Prefer lightweight bash commands over full pane captures when possible:
    - `tmux capture-pane -t "$sessionId" -p -S -5` (last 5 lines only) instead of 30+ lines
    - Check process status: `ps aux | grep "script_name"` to see if a script is still running
  - Only do full pane captures (30+ lines) when:
    - Initial assessment (Phase 1)
    - Checking off a milestone from expectations list
    - Something looks wrong (unexpected process state)
    - Checking results after task completion
  - For intermediate checks, a 5-line tail of the pane is usually sufficient to confirm dev is still working

## Paths

- **RLQuest directory:** `/home/ubuntu/workspace/RLQuest`
- **View script:** `/home/ubuntu/workspace/RLQuest/tmux_view_claude.sh`
- **Run script:** `/home/ubuntu/workspace/RLQuest/tmux_run_claude.sh`
- **Send script:** `/home/ubuntu/workspace/RLQuest/tmux_send_claude.sh`
- **PID file:** `/home/ubuntu/workspace/RLQuest/claude_session_PID_1`

## Procedure

### Step 1: Check if Claude tmux session 1 is running

Run this to check if the session exists:

```bash
cd /home/ubuntu/workspace/RLQuest && bash -c '
INSTANCE_ID=1
pidfile="claude_session_PID_${INSTANCE_ID}"
if [ ! -f "$pidfile" ]; then
    echo "NO_SESSION"
    exit 1
fi
sessionId=$(cat "$pidfile")
if ! tmux has-session -t "$sessionId" 2>/dev/null; then
    echo "NO_SESSION"
    exit 1
fi
echo "SESSION_RUNNING: $sessionId"
exit 0
'
```

- If the output is `SESSION_RUNNING: <id>`, the session is alive — proceed to **Step 3**.
- If the output is `NO_SESSION`, proceed to **Step 2**.

### Step 2: Start Claude tmux session 1

Run the run script to start a new session:

```bash
cd /home/ubuntu/workspace/RLQuest && ./tmux_run_claude.sh 1
```

Wait 3 seconds for the session to initialize, then verify it started by re-running the check from Step 1. If it still fails, report the error and stop.

### Step 3: Check conversation state

Capture the last 30 lines of the tmux pane to determine if Claude is in an active conversation or at a fresh/clear prompt:

```bash
cd /home/ubuntu/workspace/RLQuest && bash -c '
sessionId=$(cat claude_session_PID_1)
tmux capture-pane -t "$sessionId" -p -S -30
'
```

Analyze the captured output:

- **Already in a conversation** (you see ongoing dialogue, tool calls, or assistant responses): Do nothing — leave the session as-is.
- **Fresh/clear prompt** (you see the Claude welcome message, a blank prompt like `>`, or output indicating no active conversation such as "What would you like to do?"): Proceed to **Step 4**.

### Step 4: Resume the latest conversation

Use the send script to type `/resume` into the Claude session:

```bash
cd /home/ubuntu/workspace/RLQuest && ./tmux_send_claude.sh 1 "/resume"
```

Wait 3 seconds for the resume picker to appear, then send `Enter` to select the most recent (top) conversation:

```bash
cd /home/ubuntu/workspace/RLQuest && ./tmux_send_claude.sh 1 Enter
```

Wait 3 seconds, then capture the pane output again (as in Step 3) to confirm the conversation has resumed successfully.

## Decision Tree & Optimization

After ensuring dev is running and in a conversation, follow the **decision tree** in `decision_tree.md` to determine the next action based on what dev is doing.

If dev is running a long-running script or task:
- Consult `long_running_script_guide.md` to evaluate optimization quality
- Check `verified_scripts.json` to see if this script was previously verified
- Validate runtime behavior (CPU/GPU/memory) even for verified scripts
- Decide whether to wait or instruct dev to stop and optimize

## Goal Tracking & Work Planning

On **every** "manage" command, after evaluating dev's status, also perform goal tracking:

1. **Identify** what dev is working on from the pane output (files edited, scripts running, conversation topic)
2. **Map** the work to project goals using `goal_tracker.md` and `goals.md` (project root)
3. **Evaluate progress** toward current goal:
   - If training: check loss curves, val metrics, GPU util per `training_evaluation_guide.md`
   - If data pipeline: check progress, output quality, status.json
   - If evaluation: check metrics vs V3 baseline and foresight upper bound
4. **Update `goal_tracker.md`** with current goal status, latest metrics, and next goals
5. **Plan next goal** if dev is idle or current goal is complete:
   - Read `direction.md` priorities and `goal_tracker.md` next goals list
   - Include the recommended next task in `manager.md`

When the "send" command is used, include **goal context** in the prompt to dev — explain not just what to do, but why it matters for the project goals.

### Reference Files

- `goal_tracker.md` — living goal/metric tracker (update on every manage)
- `training_evaluation_guide.md` — how to evaluate training progress and decide next steps
- `goals.md` (project root) — high-level project goals
- `firstrate_learning/direction.md` — detailed improvement priorities and training recipe

## Summary

1. Check if session 1 exists (PID file + tmux has-session).
2. If not running, start it with `tmux_run_claude.sh 1`.
3. Inspect the tmux pane content to determine conversation state.
4. If fresh/clear, send `/resume` via `tmux_send_claude.sh 1 "/resume"`.
5. Send `Enter` to select the most recent conversation from the resume picker.
6. Follow `decision_tree.md` to evaluate what dev is doing.
7. If long-running script detected, apply `long_running_script_guide.md`.
8. **Evaluate work alignment** — map dev's work to project goals.
9. **Update `goal_tracker.md`** with current status and metrics.
10. **Plan next goal** if dev is idle or current goal complete.
11. Report final status to the user (include goal alignment in `manager.md`).
