# Step 4: Learn & Update

**Purpose**: Reflect on previous actions, update goal tracker, log improvement recommendations. Do NOT modify dev or manager instruction files directly.

## Persistence

**FIRST**: Read your own previous output from `/home/ubuntu/workspace/RLQuest-manager/step4_output.md` if it exists. This contains your previous reflections, what was updated, and persistent notes about process gaps and patterns observed across cycles.

**If nothing significant changed since last cycle** (same constraint, same training status, no new results): write brief "No significant changes. Previous learnings still apply." + carry forward persistent notes. This should take <1 minute.

## Instructions

Read Steps 1-3 from `/home/ubuntu/workspace/RLQuest-manager/step1_output.md`, `step2_output.md`, `step3_output.md`.

### 4.1 Previous Action Review
- What was the last action sent to dev?
- Did dev complete it? Check evidence from Step 1.
- Did it produce the expected result? Compare to success criteria from previous step5_output.md.
- Root cause if it failed or was unexpected.

### 4.2 Process Retrospective
- Questions the manager should have asked but didn't?
- Decisions made too quickly?
- Resources left idle?
- Design issues accepted without review?

### 4.3 Update Goal Tracker
Update `/home/ubuntu/workspace/RLQuest-manager/goal_tracker.md` with:
- Current goal, status, hypothesis, constraint, belief state, metrics, priority queue.
- Only update DATA — do not change the file's structure.

### 4.4 Update Research Documents
- Update relevant docs in `/home/ubuntu/workspace/RLQuest-manager/research/` if new evidence emerged.

### 4.5 Log Dev Improvement Recommendations

**DO NOT modify any files in `/home/ubuntu/workspace/RLQuest/`.**
APPEND to `/home/ubuntu/workspace/RLQuest-manager/future_dev_improvements.md` (create if missing):
```
### [Date] — [Issue Title]
- **Found in**: Step [N]
- **File to change**: [path]
- **Recommended change**: [specific]
- **Evidence**: [what revealed this gap]
```

### 4.6 Log Manager Improvement Recommendations

**DO NOT modify any files in `.claude/skills/manager/`.**
APPEND to `/home/ubuntu/workspace/RLQuest-manager/future_manager_improvements.md` (create if missing):
```
### [Date] — [MANAGER] [Issue Title]
- **Found in**: Step [N]
- **Step file to change**: [e.g., step3_design_review.md]
- **Recommended change**: [specific]
- **Evidence**: [what was missed]
```

## Output Format

Write to `/home/ubuntu/workspace/RLQuest-manager/step4_output.md` (overwrite):

```
## Step 4: Learn & Update — [date/time]

### Last action: [description]
### Result: [success/partial/failure] — [evidence]
### Process gaps: [list or "none"]
### Goal tracker updated: [yes/no, what changed]
### Research docs updated: [yes/no]
### Dev improvements logged: [yes/no, count]
### Manager improvements logged: [yes/no, count]

## Persistent Notes
- Cycle count: [N] (increment each run)
- Pattern log: [recurring issues observed across cycles]
- Process effectiveness: [which steps are working well, which need refinement]
- Accumulated learnings: [key insights that should inform future cycles]
- Watch items for next cycle: [specific things Step 4 should check]
```
