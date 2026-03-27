# Step 6: Execute & Monitor

**Purpose**: Send the prompt to dev, actively monitor compliance, intervene on deviations, chain next task on completion.

## Instructions

### 6.1 Send Prompt
- Read the recommended prompt from `manager_step_based.md` (Step 5 output).
- Send to dev via `tmux_send_claude.sh 1 "<prompt>"`.
- If prompt appears as "Pasted text" without submitting: send `Enter` via `tmux_send_claude.sh 1 Enter`.

### 6.2 Build Checkpoint Expectations
From the prompt, extract the specific milestones dev should hit. Write them as a checklist.

### 6.3 Active Monitoring Loop

**Phase 1: Initial (first 2-3 checks, every 10-15 seconds)**
- Full pane capture (30+ lines).
- Verify dev received the prompt and started working.
- If dev ignores or goes off-track: send correction.

**Phase 2: Confident (dev on track)**
- Scale frequency by task duration: <5min → 30-60s, 5-30min → 2-5min, 30+min → 5-10min.
- Lightweight checks: 5-line pane tail, `ps aux`, process status.
- Full capture only at milestones or anomalies.
- On every check: is dev progressing? Skipped a step? Something unexpected?

**Phase 3: Near Completion**
- Increase frequency. Full pane capture to review results.
- Verify all checkpoints met. If dev says "done" but milestones missing → correct.

**Phase 4: Post-Completion**
- When dev finishes and goes idle: immediately check all resources.
- Are GPU, CPU, dev all utilized? Is there available next work?
- If yes: formulate and send the next task immediately. Do NOT wait for user.
- Always chain: "After X, immediately do Y."

### 6.4 Intervention Protocol

| Severity | Trigger | Action |
|----------|---------|--------|
| Minor | Wrong order, slightly off | Nudge: "Note: please also [step]." |
| Major | Skipped critical step (smoke test, perf validation) | "STOP. You skipped [step]. This is mandatory." |
| Critical | About to lose data, overwrite without backup, full run without validation | "STOP IMMEDIATELY. [reason]. You must [action] first." |

After intervention: wait 10-15s, check acknowledgment. If ignored twice → report to user and pause.

### 6.5 Cost Reduction
- Use `ps aux` and 5-line tail for routine checks.
- Full 30-line capture only for: initial assessment, milestone verification, anomalies, completion review.
- For long-running tasks (30+ min), use process-based checks (`ps aux | wc -l`) between full captures.
