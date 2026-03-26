---
name: mgr
description: Ensure RLQuest Claude tmux session 1 is running and has an active conversation
disable-model-invocation: true
---

# Manager — RLQuest Claude Session Monitor

Ensure the RLQuest Claude tmux session (instance 1) is running and has an active conversation.

## Terminology

- **"dev"** refers to the Claude tmux session instance 1. When the user says "dev", they mean this session.

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

## Summary

1. Check if session 1 exists (PID file + tmux has-session).
2. If not running, start it with `tmux_run_claude.sh 1`.
3. Inspect the tmux pane content to determine conversation state.
4. If fresh/clear, send `/resume` via `tmux_send_claude.sh 1 "/resume"`.
5. Send `Enter` to select the most recent conversation from the resume picker.
6. Follow `decision_tree.md` to evaluate what dev is doing.
7. If long-running script detected, apply `long_running_script_guide.md`.
8. Report final status to the user.
