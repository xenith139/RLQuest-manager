# Step 1: Ground Truth & State Assessment

**Purpose**: Establish what is ACTUALLY happening right now. Trust nothing cached. Verify everything independently.

## Persistence

**FIRST**: Read your own previous output from `/home/ubuntu/workspace/RLQuest-manager/step1_output.md` if it exists. This contains your findings from last cycle and persistent notes about the project structure. Use it as context — but VERIFY everything against current reality. Previous findings may be outdated.

If the previous output has a `## Persistent Notes` section, read it carefully — it contains knowledge you accumulated about file locations, project structure, and what to watch for.

## Instructions

Answer every question below with evidence from the system. Use bash commands to verify.

**If your previous output exists and describes the state**: only re-check things that could have changed (processes, GPU, training progress, dev pane). Skip re-discovering things that are stable (file locations, code structure, nproc). Reference your persistent notes for stable facts.

**If no previous output exists**: full discovery — check everything.

### 1.1 Dev Session
- Is the dev tmux session (instance 1) running? Check PID file + tmux has-session in `/home/ubuntu/workspace/RLQuest`.
- If not running: start it with `tmux_run_claude.sh 1`, resume with `/resume` + Enter.
- Capture last 10 lines of dev pane: what is dev currently doing? (idle, coding, running script, waiting)

### 1.2 CPU State
- `ps aux | grep python` — what processes are running? List PIDs, CPU%, memory.
- Is CPU idle or busy? What specifically is using it?

### 1.3 GPU State
- `nvidia-smi` — GPU utilization %, memory used/total.
- Is GPU idle or training? If training: what model, what epoch, what metrics?

### 1.4 Filesystem State (skip if unchanged from persistent notes)
- V5 code: which .py files exist in `firstrate_learning_v5/`?
- V5 tokens: `cat firstrate_learning_v5/cache/tokens/meta.json` — complete?
- Latest training log: `tail -5` any active training logs.
- V5 model run dirs: `ls firstrate_learning_v5/models/run_*/`

### 1.5 Cross-Reference
- Read `/home/ubuntu/workspace/RLQuest-manager/goal_tracker.md`.
- Does goal_tracker match reality? Note discrepancies.

## Output Format

Write to `/home/ubuntu/workspace/RLQuest-manager/step1_output.md` (overwrite). Use this structure:

```
## Step 1: Ground Truth — [date/time]

### Dev: [running/not running] — [idle/coding/running script]
### CPU: [idle/busy] — [what's running, % utilization]
### GPU: [idle/busy] — [utilization %, what's training]
### V5 Code: [list of existing files]
### V5 Data: [complete/incomplete — samples, chunks]
### V5 Training: [not started / epoch N of M / complete — latest metrics]
### Goal tracker discrepancy: [none / description]

## Persistent Notes
(Carry forward from previous output + add new discoveries. These survive across cycles.)
- Project structure: [stable facts about file locations, code layout]
- Key paths: [important file paths discovered]
- nproc: [N] cores
- GPU: [model, VRAM]
- Last known state change: [what changed this cycle vs last]
- Watch items for next cycle: [specific things to check]
```
