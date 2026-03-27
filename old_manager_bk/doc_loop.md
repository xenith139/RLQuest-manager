# Manager Autonomous Loop Design

## Concept

The manager loop is an autonomous cycle where the manager continuously:
1. Runs the **manage** workflow (OBSERVE → ORIENT → DECIDE → VALIDATE → ACT)
2. Runs the **send** workflow (send prompt to dev → monitor → intervene)
3. Waits for dev to complete
4. Loops back to step 1

The user can start and stop this loop at will. When running, the manager operates autonomously — no user intervention needed between cycles.

---

## How It Works in Claude Code

### The `/loop` Command

Claude Code has a built-in `/loop` command that runs a prompt or slash command on a recurring interval:

```
/loop 5m /m                     # Run /m every 5 minutes
/loop 10m check on dev          # Run a prompt every 10 minutes
/loop 30m /m s                  # Run manage+send every 30 minutes
```

**Characteristics:**
- Runs only while Claude Code is open and idle (session-scoped)
- Interval: seconds, minutes, hours, days (default 10m if omitted)
- Auto-expires after 3 days
- Max 50 scheduled tasks per session
- Uses `CronCreate`, `CronList`, `CronDelete` tools internally

### Hooks (settings.json)

Hooks are deterministic shell commands that fire at lifecycle events. Relevant events:

| Event | When It Fires | Use |
|-------|---------------|-----|
| `Stop` | When Claude finishes a response | Could trigger next step |
| `TaskCompleted` | When a background task completes | Could check if dev finished |
| `SessionStart` | When a new session starts | Could auto-start the loop |

**Hook configuration** lives in `.claude/settings.json`:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/manager_loop.sh",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

**Important limitation**: Hooks cannot directly trigger new prompts or slash commands. They can only:
- Block/allow the current action (exit code 0 = allow, 2 = block)
- Inject context text into Claude's conversation
- Return JSON decisions

This means hooks alone can't create a full autonomous loop. They need to be combined with `/loop` or a wrapper script.

---

## Three Implementation Options

### Option A: `/loop` + Unified Skill (Recommended — Simplest)

Create a single skill that runs both manage and send in sequence, then use `/loop` to repeat it.

**How it works:**
1. User types: `/loop 10m /m auto`
2. Every 10 minutes, Claude runs the `/m auto` skill
3. The skill runs manage (OODA loop), then if there's a recommendation, automatically sends it
4. After dev completes, the cycle ends — `/loop` fires again in 10 minutes

**New keyword in SKILL.md: "auto"**

When the user says "auto", the manager runs:
1. The full manage workflow (OBSERVE → ORIENT → DECIDE → VALIDATE → ACT)
2. If manager.md contains a recommendation → immediately execute the send workflow
3. Monitor dev until task completion
4. Run one more ORIENT cycle (Phase 4 post-completion)
5. If there's more work → send the next task
6. When dev is idle and no more immediate work → return control to `/loop` timer

**Entering the loop:**
```
/loop 10m /m auto              # Start the autonomous loop (10 min interval)
/loop 5m /m auto               # More aggressive (5 min checks)
/loop 30m /m auto              # Relaxed (30 min checks, for long training)
```

**Exiting the loop:**
```
cancel the manager loop        # Natural language
/m stop-loop                   # Or explicit command
```
Claude will use `CronDelete` to remove the scheduled task.

**Checking loop status:**
```
what loops are running?        # Natural language
/m loop-status                 # Or explicit command
```

### Option B: Wrapper Script (Most Autonomous — For Unattended Operation)

A bash script that runs Claude Code in a loop externally:

```bash
#!/bin/bash
# manager_loop.sh — Run from terminal, survives Claude restarts

LOOP_FILE="/home/ubuntu/workspace/RLQuest-manager/.manager_loop_active"
touch "$LOOP_FILE"

while [ -f "$LOOP_FILE" ]; do
    echo "=== Manager cycle $(date) ==="

    # Run manage + send in one Claude invocation
    claude -p "/m auto" --dangerously-skip-permissions \
        --working-dir /home/ubuntu/workspace/RLQuest-manager

    # Wait between cycles
    sleep 600  # 10 minutes
done

echo "Manager loop stopped."
```

**Entering**: `nohup ./manager_loop.sh > manager_loop.log 2>&1 &`
**Exiting**: `rm /home/ubuntu/workspace/RLQuest-manager/.manager_loop_active`

Pros: Survives Claude session restarts, fully unattended.
Cons: Each cycle is a fresh Claude session (no conversation context carried over).

### Option C: Stop Hook + Context Injection (Most Integrated — Complex)

Use a `Stop` hook to detect when the manage workflow ends and inject context telling Claude to continue:

```bash
#!/bin/bash
# .claude/hooks/manager_loop.sh

INPUT=$(cat)

# Check if the manager loop is active
LOOP_FILE="/home/ubuntu/workspace/RLQuest-manager/.manager_loop_active"
if [ ! -f "$LOOP_FILE" ]; then
    exit 0  # Loop not active, do nothing
fi

# Output context that Claude will see on next turn
echo "MANAGER LOOP ACTIVE: The autonomous manager loop is running. Run /m auto for the next cycle."
exit 0
```

This injects a reminder into Claude's context after every response, prompting it to continue the loop. The user controls start/stop via the loop file.

---

## Implemented Design: Option A (`/loop` + "auto" keyword)

This is the simplest, most Claude-native approach. No external scripts, no hooks. The Stop hook approach (Option C) was tested and doesn't work — hooks can inject context but can't trigger new prompts.

### Changes Needed

#### 1. Add "auto" keyword to SKILL.md

```
- **"auto"** — when the user says "auto" (typically via `/loop 10m /m auto`):
  1. Run the full manage workflow (OBSERVE → ORIENT → DECIDE → VALIDATE → ACT)
  2. If manager.md contains a recommendation with a prompt:
     a. Automatically execute the send workflow (extract prompt, send to dev, monitor)
     b. After dev completes, run Phase 4 post-completion ORIENT
     c. If more work available → send next task immediately
     d. Repeat until dev is idle with no immediate work
  3. Return control (loop timer handles the next cycle)
```

#### 2. Add loop control keywords

```
- **"start-loop"** or **"loop"** — start the autonomous loop:
  Use CronCreate to schedule `/m auto` at the specified interval.
  Default: 10 minutes. User can specify: `/m loop 5m`, `/m loop 30m`

- **"stop-loop"** — stop the autonomous loop:
  Use CronDelete to remove the scheduled task.
  Delete any loop state files.

- **"loop-status"** — show current loop state:
  Use CronList to show active scheduled tasks.
```

### User Experience

**Starting the loop:**
```
> /m loop 10m
Manager autonomous loop started. Running /m auto every 10 minutes.
To stop: /m stop-loop
First cycle starting now...

[Manager runs OBSERVE → ORIENT → DECIDE → VALIDATE → ACT]
[If recommendation exists → automatically sends to dev → monitors]
[Returns after dev completes or goes idle]

... 10 minutes later ...

[Next cycle starts automatically]
```

**Stopping the loop:**
```
> /m stop-loop
Manager autonomous loop stopped. 3 cycles completed.
```

**Checking status:**
```
> /m loop-status
Manager loop: ACTIVE
Interval: 10 minutes
Last cycle: 2026-03-27 17:30 (completed in 45s)
Next cycle: 2026-03-27 17:40
Cycles completed: 7
```

---

## Loop Behavior Rules

### When to Send vs When to Wait

The "auto" mode needs rules for when to automatically send vs when to stop and wait for user:

```
AUTO-SEND (proceed without user):
  - Dev is idle and there's a clear next task from goal_tracker
  - The task is operational (training, data prep, implementation)
  - The VALIDATE step passes all checks
  - The action is within the current goal trajectory

PAUSE AND REPORT (wait for user):
  - Strategic pivot needed (e.g., switch from V4 to V5 direction)
  - Constraint changed (new information that changes the plan)
  - Dev encountered an error that requires user judgment
  - Cost of next action is very high (>24 hours GPU time)
  - The manager is uncertain (belief confidence < MEDIUM)
  - User explicitly requested review before send
```

### Interval Recommendations

| Situation | Interval | Reason |
|-----------|----------|--------|
| Dev implementing code | 5-10m | Code changes happen fast, check frequently |
| Training running (monitored) | 30-60m | Training is autonomous, just check metrics |
| Data pipeline running | 15-30m | Monitor progress, check for issues |
| Dev idle, waiting for long process | 60m | Nothing to do until process completes |

### Cost Management

The loop consumes tokens on every cycle. To manage costs:
- Use lightweight checks (5-line tail, `ps aux`) for intermediate cycles
- Only do full ORIENT with document reads when state has changed
- If nothing changed since last cycle → short "no change" report, skip full analysis
- Scale interval up when dev is in a long task (training, large data prep)

---

## Integration with Existing Manager Skills

The loop doesn't replace the existing `/m m` and `/m s` commands — it automates them:

```
/m m          — manual: run manage once, write manager.md, stop
/m s          — manual: send manager.md prompt to dev, monitor, stop
/m auto       — one full cycle: manage + send + monitor + post-completion orient
/m loop 10m   — start autonomous loop (repeats /m auto every 10m)
/m stop-loop  — stop the loop
/m loop-status — check loop state
```

The user can always interrupt the loop by typing `/m stop-loop` or just interacting with Claude directly (which pauses the loop until Claude is idle again).

---

## What This Solves

The autonomous loop prevents the two failure modes we identified:

1. **Dev sitting idle for hours** — the loop detects idle resources and sends the next task automatically
2. **Token prep not started while training ran** — the loop would have caught the idle CPU and queued the data prep run

The loop makes the manager truly autonomous: observe → orient → decide → validate → act → monitor → learn → repeat.
