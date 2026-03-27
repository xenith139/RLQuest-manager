# Step 2: Gap Analysis & Constraint Identification

**Purpose**: Understand where we are vs where we need to be, and identify the single biggest bottleneck.

## Persistence

**FIRST**: Read your own previous output from `/home/ubuntu/workspace/RLQuest-manager/step2_output.md` if it exists. This contains your previous gap analysis, hypothesis, and constraint identification. Use it as context — confirm or update, don't re-derive from scratch if nothing changed.

## Instructions

Read Step 1 findings from `/home/ubuntu/workspace/RLQuest-manager/step1_output.md`.
Read `/home/ubuntu/workspace/RLQuest-manager/goals.md` for targets.

**If your previous output exists and the constraint hasn't shifted**: quickly confirm "constraint unchanged, hypothesis unchanged, gap [same/narrowing/widening]" and move on. Don't re-do the full analysis.

**If something changed** (new training results, dev completed a task, state shifted): update the analysis.

### 2.1 Current Performance
- Best model performance? Which version, epoch, metrics?
- If training is in progress: read latest metrics from log files.

### 2.2 Target Performance
- From `goals.md`: what CR per 10-day period is the target?

### 2.3 Gap
- Current CR vs target CR — ratio.
- Is gap shrinking? (Compare to previous cycle's gap from persistent notes.)

### 2.4 Constraint Identification
Choose ONE: Architecture / Training recipe / Data quality / Data quantity / Infrastructure / Knowledge gap / Execution.
- Evidence for chosen constraint.
- Has it changed since last cycle?

### 2.5 Hypothesis
- State: "I believe [action] will improve [metric] because [reasoning]."
- Success criteria. Failure criteria. Cost of being wrong.

## Output Format

Write to `/home/ubuntu/workspace/RLQuest-manager/step2_output.md` (overwrite):

```
## Step 2: Gap & Constraint — [date/time]

### Best performance: [version] [epoch] CR=[value]
### Target: CR > [value]
### Gap: [ratio]x improvement needed — [shrinking/stable/growing vs last cycle]
### Constraint: [category] — [description]
### Constraint changed from last cycle: [yes/no — what shifted]
### Hypothesis: "[statement]"
### Success criteria: [metrics]
### Failure criteria: [pivot trigger]

## Persistent Notes
- Previous constraint: [what it was last cycle]
- Previous gap: [ratio from last cycle]
- Hypothesis history: [track how hypothesis evolved]
- Key evidence accumulated: [facts that inform future analysis]
- Watch items for next cycle: [what to check]
```
