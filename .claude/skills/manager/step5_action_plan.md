# Step 5: Action Planning & Validation Gate

**Purpose**: Decide what to do next, validate it's ready, write the prompt.

## Persistence

**FIRST**: Read your own previous output from `/home/ubuntu/workspace/RLQuest-manager/step5_output.md` if it exists. This contains your previous action plan, validation results, and persistent notes about what actions were taken and what's pending.

**If the previous plan's action is still in progress** (e.g., training still running): confirm "previous action still executing, no new plan needed" + carry forward persistent notes. This should take <1 minute.

**If the previous action completed or the constraint shifted**: plan the next action.

## Instructions

Read ALL previous steps: `step1_output.md`, `step2_output.md`, `step3_output.md`, `step4_output.md` in `/home/ubuntu/workspace/RLQuest-manager/`.

### 5.1 Enumerate All Possible Actions
List EVERY action: operational, strategic, meta, parallel.

### 5.2 Resource Check
- GPU: idle or busy? What could use it?
- CPU: idle or busy? What could use it?
- Dev: idle or busy? What could dev work on?
Never leave a resource idle with available work.

### 5.3 Rank by Expected Value
Top 3 candidates: expected improvement, cost, risk, information value.

### 5.4 Validation Gate — MANDATORY

**Data pipeline?**
- [ ] Smoke test on representative data?
- [ ] CPU multi-core verified?
- [ ] Memory stable?
- [ ] Throughput measured?

**Training?**
- [ ] Unit test passed?
- [ ] GPU util >70%?
- [ ] AMP+FP16?
- [ ] torch.compile?
- [ ] Checkpoint/resume (all 9 fields)?
- [ ] Per-run directory?
- [ ] Design doc complete?

**Implementation?**
- [ ] Design doc in `research/` with complete specs?

**ALL actions:**
- [ ] Chained to follow-up?
- [ ] Success criteria defined?
- [ ] Failure criteria defined?

If any check fails: add the validation step BEFORE the action in the prompt.

### 5.5 Write the Plan & Prompt
Include: what to do, why (constraint reasoning), prerequisites, chained follow-ups, success/failure criteria.

## Output Format

Write to `/home/ubuntu/workspace/RLQuest-manager/step5_output.md` (overwrite):

```
## Step 5: Action Plan — [date/time]

### Status: [NEW PLAN / PREVIOUS PLAN STILL EXECUTING / NO ACTION NEEDED]
### All possible actions: [numbered list]
### Resources: GPU=[idle/busy], CPU=[idle/busy], Dev=[idle/busy]
### Chosen action(s): [description with reasoning]
### Validation gate: [checklist results]
### Parallel tracks: [if applicable]
### Success criteria: [specific]
### Failure criteria: [specific]

## Recommended Prompt
[Exact prompt to send to dev]

## Persistent Notes
- Actions taken history: [list of actions sent in previous cycles and their outcomes]
- Pending items: [things deferred to future cycles]
- Validation patterns: [which checks tend to fail, what to watch for]
- Effective prompts: [prompt patterns that worked well with dev]
- Watch items for next cycle: [what to plan for]
```
