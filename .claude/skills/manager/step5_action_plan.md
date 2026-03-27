# Step 5: Action Planning & Validation Gate

**Purpose**: Decide what to do next and verify it's ready to execute. Enumerate all options, pick highest value, validate prerequisites, write the prompt.

## Instructions

### 5.1 Enumerate All Possible Actions
List EVERY action that could be taken right now. Include:
- **Operational**: launch training, run data prep, run smoke test, run unit test, fix code
- **Strategic**: review architecture, redesign model, pivot approach, investigate alternatives
- **Meta**: update skills, update rules, document findings, improve process
- **Parallel**: things that can run simultaneously (GPU + CPU + dev)

### 5.2 Resource Check
For each resource, state:
- **GPU**: idle or busy? If idle, what could use it?
- **CPU**: idle or busy? If idle, what could use it?
- **Dev**: idle or busy? If idle, what could dev work on?

Priority: NEVER leave a resource idle when there's available work for it.

### 5.3 Rank by Expected Value
For the top 3 candidate actions, evaluate:
- Expected improvement to the key metric (captured return)
- Cost (GPU-hours, dev-hours, wall clock time)
- Risk (probability it fails or wastes time)
- Information value (does it reduce uncertainty about the right path?)

### 5.4 Validation Gate — MANDATORY
For EACH recommended action, check ALL that apply:

**Data pipeline (prepare_*, process_*)?**
- [ ] Smoke test on REPRESENTATIVE data (large quarters, not tiny test)?
- [ ] CPU multi-core utilization measured?
- [ ] Memory stability verified?
- [ ] Throughput measured and extrapolated?
- If ANY unchecked: add validation step BEFORE the action.

**Training (train*.py)?**
- [ ] Unit test passed?
- [ ] Smoke test with GPU util >70%?
- [ ] AMP+FP16 enabled?
- [ ] torch.compile enabled?
- [ ] Checkpoint/resume working (all 9 fields)?
- [ ] Per-run directory?
- [ ] Design doc complete in research/?
- If ANY unchecked: add validation step BEFORE the action.

**Implementation (new code)?**
- [ ] Design doc in research/ with complete specs?
- If NO: investigate and write design FIRST.

**ALL actions:**
- [ ] Chained to follow-up? (After X, immediately do Y.)
- [ ] Success criteria defined?
- [ ] Failure criteria defined?
- [ ] Never ends with "report when ready" without next step.

### 5.5 Write the Plan & Prompt
Compose the prompt to dev that:
- Explains WHAT to do and WHY (constraint reasoning)
- Includes all prerequisite validation steps
- Chains follow-up tasks
- Defines success and failure criteria
- Specifies parallel tracks if applicable

## Output Format

```
## Step 5: Action Plan

### All possible actions: [numbered list]
### Resources: GPU=[idle/busy], CPU=[idle/busy], Dev=[idle/busy]
### Chosen action(s): [description with reasoning]
### Validation gate:
  - [checklist results, what passed, what was added]
### Parallel tracks: [if applicable]
### Success criteria: [specific]
### Failure criteria: [specific]

## Recommended Prompt
[The exact prompt to send to dev, ready to copy-paste or auto-send]
```
