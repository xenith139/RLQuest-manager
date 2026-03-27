# Step 4: Learn & Update

**Purpose**: Reflect on the last executed action and all previous cycles. Did things go as expected? What was missed? Update all tracking documents. Improve the manager process and dev skills if gaps were found.

## Instructions

### 4.1 Previous Action Review
- What was the last action sent to dev? (Read the most recent manager output file if it exists.)
- Did dev complete it? Check dev pane and filesystem for evidence.
- Did the action produce the expected result? Compare actual outcome vs stated success criteria.
- If it failed or produced unexpected results: why? Root cause?

### 4.2 Process Retrospective
- Were there any questions the manager should have asked but didn't in previous cycles?
- Were there decisions made too quickly without sufficient investigation?
- Were there resources left idle that could have been working?
- Were there design issues accepted without deep review?
- Did the manager follow its own rules? (Smoke test before full run? Design doc before implementation? Performance validation?)

### 4.3 Update Goal Tracker
Write/update `goal_tracker.md` in the skills folder with:
- Current goal and status (verified from Step 1 ground truth)
- Current hypothesis (from Step 2)
- Current constraint (from Step 2)
- Belief state: confidence levels for architecture, recipe, data, next action
- Latest metrics (from actual training results, not estimates)
- Next goals priority list

### 4.4 Update Research Documents
- If Step 3 found design issues: update the relevant doc in `research/`
- If new evidence emerged (training metrics, overfitting, performance data): add to research docs
- If a design decision was validated or invalidated: document it

### 4.5 Improve Dev Skills & Rules
Check if dev's behavior in the last action revealed gaps in RLQuest `.claude/` configuration:
- Did dev follow all rules? (AMP, checkpointing, smoke test, ProgressTracker, etc.)
- Were there rules that should exist but don't?
- Were there rules that are too vague and dev interpreted incorrectly?
- If gaps found: update the specific file in `RLQuest/.claude/rules/` or `RLQuest/CLAUDE.md`

### 4.6 Improve Manager Process
- Did the 6-step process work well in this cycle, or did a step miss something?
- Should any step's questions be expanded or refined?
- Were there failure modes not covered by the current steps?

## Output Format

```
## Step 4: Learn & Update

### Last action: [description]
### Result: [success/partial/failure] — [evidence]
### Process gaps found: [list or "none"]
### Goal tracker updated: [yes/no, what changed]
### Research docs updated: [yes/no, what changed]
### Dev skills updated: [yes/no, what changed]
### Manager process improvement: [specific change or "none needed"]
```
