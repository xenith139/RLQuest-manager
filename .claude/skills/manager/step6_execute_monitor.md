# Step 6: Execute & Monitor

**Purpose**: Send the prompt to dev, actively monitor compliance, intervene on deviations, chain next task on completion.

## Persistence

**FIRST**: Read your own previous output from `/home/ubuntu/workspace/RLQuest-manager/step6_output.md` if it exists. This contains notes about dev's behavior patterns, effective intervention strategies, and monitoring observations from previous cycles.

Note: Step 6 also writes an output file (unlike the original design) so future Step 6 subagents have context about dev interaction patterns.

## Instructions

### 6.1 Send Prompt
- Read the recommended prompt from `/home/ubuntu/workspace/RLQuest-manager/step5_output.md` (the `## Recommended Prompt` section).
- Send to dev via `/home/ubuntu/workspace/RLQuest/tmux_send_claude.sh 1 '<prompt>'`.
- If it shows as "Pasted text": send Enter via `tmux_send_claude.sh 1 Enter`.

### 6.2 Build Checkpoint Expectations
Extract milestones from the prompt. Track as a checklist.

### 6.3 Active Monitoring

**Phase 1: Initial (first 2-3 checks, every 10-15 seconds)**
- Full pane capture (30+ lines). Verify dev started working.
- If dev ignores or goes off-track: send correction.

**Phase 2: Confident (dev on track)**
- Scale by task duration: <5min → 30-60s, 5-30min → 2-5min, 30+min → 5-10min.
- Lightweight checks: 5-line tail, `ps aux`.
- Full capture only at milestones or anomalies.

**Phase 3: Near Completion**
- Increase frequency. Full capture to review results.
- Verify all checkpoints. Correct if missing.

**Phase 4: Post-Completion**
- When dev finishes and goes idle: note what was accomplished.
- This information will be used by the orchestrator to decide whether to loop back to Step 1.

### 6.4 Intervention Protocol

| Severity | Trigger | Action |
|----------|---------|--------|
| Minor | Wrong order, slightly off | "Note: please also [step]." |
| Major | Skipped critical step | "STOP. You skipped [step]. Mandatory." |
| Critical | Data loss risk, full run without validation | "STOP IMMEDIATELY. [reason]." |

After intervention: wait 10-15s, check acknowledgment. If ignored twice → report to user.

### 6.5 Cost Reduction
- `ps aux` and 5-line tail for routine checks.
- Full 30-line capture only for: initial, milestones, anomalies, completion.

## Output Format

Write to `/home/ubuntu/workspace/RLQuest-manager/step6_output.md` (overwrite):

```
## Step 6: Execute & Monitor — [date/time]

### Prompt sent: [summary of what was sent]
### Dev response: [what dev did]
### Checkpoints: [which passed, which failed]
### Interventions: [count and descriptions, or "none needed"]
### Final status: [task complete / task in progress / task failed]
### Dev idle after completion: [yes/no]

## Persistent Notes
- Dev behavior patterns: [how dev responds to prompts, common issues]
- Effective prompt patterns: [what worked well]
- Intervention history: [what corrections were needed and why]
- Monitoring observations: [timing patterns, how long tasks typically take]
- Watch items for next cycle: [what to expect from dev next]
```
