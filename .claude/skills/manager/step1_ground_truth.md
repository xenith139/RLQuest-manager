# Step 1: Ground Truth & State Assessment

**Purpose**: Establish what is ACTUALLY happening right now. Trust nothing cached. Verify everything independently.

## Instructions

Answer every question below with evidence from the system. Use bash commands to verify. Write answers to the output file.

### 1.1 Dev Session
- Is the dev tmux session (instance 1) running? Check PID file + tmux has-session.
- If not running: start it with `tmux_run_claude.sh 1`, resume with `/resume` + Enter.
- Capture last 10 lines of dev pane: what is dev currently doing? (idle, coding, running script, waiting)

### 1.2 CPU State
- `ps aux | grep python` — what processes are running? List PIDs, CPU%, memory.
- How many cores available? (`nproc`)
- Is CPU idle or busy? What specifically is using it?

### 1.3 GPU State
- `nvidia-smi` — GPU utilization %, memory used/total.
- Is GPU idle or training? If training: what model, what epoch, what metrics?
- If GPU idle: is there work that should be using it?

### 1.4 Filesystem State
Check each item exists and report contents:
- V4 models: `ls firstrate_learning_v4/models/run_*/` — what run dirs exist? Latest checkpoints?
- V5 code: does `firstrate_learning_v5/` exist? Which .py files? (`config.py`, `model.py`, `data_loader.py`, `train.py`)
- V5 tokens: `cat firstrate_learning_v5/cache/tokens/status.json` — quarters complete? Total samples?
- V5 token prep: `cat firstrate_learning_v5/cache/tokens/meta.json` — is it complete?
- Latest log files: `tail -3` any active training or pipeline logs.
- Research docs: `ls research/` — what design documents exist?

### 1.5 Cross-Reference
- Read `goal_tracker.md` (if it exists in the skills folder).
- Does goal_tracker match reality? If not, note the discrepancies. Reality wins.

## Output Format

Write findings as:
```
## Step 1: Ground Truth

### Dev: [running/not running] — [idle/coding/running script]
### CPU: [idle/busy] — [what's running, % utilization]
### GPU: [idle/busy] — [utilization %, what's training]
### V5 Code: [list of existing files]
### V5 Data: [quarters done]/64, complete: [yes/no]
### V5 Training: [not started / epoch N / complete]
### Key discrepancy with goal_tracker: [none / description]
```
