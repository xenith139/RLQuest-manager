# Manager Architecture Redesign — Analysis & Framework

## Part 1: What Went Wrong — Root Cause Analysis

### The Failure

When the manager ran the "manage" command and found dev idle with V4 token prep complete, it produced this recommendation:

> "Launch V4 transformer training now."

It then spent hours supervising training execution — ensuring GPU util was >70%, checkpoints were saved, AMP was enabled. All operationally correct. **But it never asked the fundamental question: "Is the V4 architecture the right thing to train?"**

The user had to manually ask for architecture documentation, which revealed that V4 has a critical blind spot (single-day snapshot, no temporal context). This led to the V5 design. The manager missed this entirely.

### Why the Manager Missed It

Tracing through the current SKILL.md and decision_tree.md, here's exactly what happened:

```
manage command invoked
  → Step 1-4: ensure dev running ✓
  → Step 6: decision_tree.md → IDLE at prompt
  → Step 8: evaluate work alignment → "V4 token prep complete, training pending"
  → Step 9: update goal_tracker.md → "next goal: launch training"
  → Step 10: plan next goal → goal_tracker says "Launch V4 full training"
  → Output: "Launch training now"
```

The manager followed its process perfectly — and that's the problem. **The process has no step for questioning the plan itself.** It's a pure execution manager, not a strategic thinker.

Specifically:

1. **Goal tracker is a checklist, not a strategy**: `goal_tracker.md` lists "Launch V4 full training" as the next goal. The manager executed the next item on the list. It never asked "Is this the right list? Are there prerequisites missing?"

2. **Decision tree only handles operational states**: The tree branches on "is dev idle?", "is dev running a script?", "is dev off-track?" — all operational. There's no branch for "Does the current plan make technical sense? Should we review the architecture before committing GPU-hours?"

3. **No feedback loop between results and design**: When V4 showed CR=0.0184 after epoch 1, the manager celebrated. It should have asked: "Why is epoch 1 this good? What does it mean architecturally? Are there obvious improvements to capture before committing to a full 50-epoch run?"

4. **The manager doesn't learn from its own documents**: `direction.md` lists P1-P6 priorities. `model_v4_improvements.md` (which we wrote later) identifies temporal blindness. But the manager never reads these documents proactively — it only reads them when checking compliance against rules.

---

## Part 2: Framework Analysis

### Current Manager Framework (Implicit)

The current manager operates as a **reactive task executor**:

```
Observe dev → Classify state → Execute next task from list → Monitor compliance
```

This is a **simple control loop** — it maintains homeostasis (dev is running, dev is following rules) but has no mechanism for questioning the trajectory.

### What's Missing: Analysis Through Six Frameworks

#### 1. Cybernetic Control Theory

**The manager has a control loop but no error signal on strategy.**

In cybernetics, a system needs:
- **Sensor** (observe) ✓ — the manager reads tmux pane
- **Comparator** (compare to desired state) ✗ — only compares operational state, not strategic state
- **Effector** (act) ✓ — sends commands to dev

The missing comparator: "How far are we from the foresight upper bound, and is the current architecture the fastest path to close that gap?" The manager compares "is dev idle?" but never compares "is V4's captured return trajectory going to reach our annual return target?"

**Fix**: Add a **strategic comparator** that runs on every manage cycle. Compare current model capabilities to goals, not just current task to task list.

#### 2. The Scientific Method / Hypothesis-Driven Execution

**The manager executes tasks but doesn't form or test hypotheses.**

The scientific method requires:
- **Hypothesis**: "V4's single-day snapshot is sufficient to capture the patterns that predict big moves"
- **Experiment**: Train V4 and measure
- **Analysis**: Did it work? If not, why not?
- **Revised hypothesis**: "Temporal patterns matter more than we thought — V5 needs multi-day lookback"

The manager never formed the hypothesis. It went straight to "train V4" without asking "what assumption is V4 testing?" and "what would falsify that assumption?"

**Fix**: Before any major task (training, data prep, architecture change), the manager should explicitly state the hypothesis being tested and define what success/failure looks like.

#### 3. Means-Ends Analysis + Recursive Decomposition

**The manager decomposes tasks but not goals.**

Current decomposition:
```
Goal: "Launch V4 training"
  → Ensure dev running
  → Send training command
  → Monitor GPU util, checkpoints
  → Evaluate metrics
```

Missing decomposition:
```
Goal: "Close the gap to foresight upper bound"
  → What's the current gap? (170% vs ~58% annualized)
  → What are the bottlenecks? (need to analyze)
    → Is the architecture the bottleneck? (single-day snapshot)
    → Is the training recipe the bottleneck? (loss weights, lr)
    → Is the data the bottleneck? (temporal features missing)
  → Which bottleneck, if removed, unlocks the most progress? (architecture)
  → Therefore: review architecture BEFORE training
```

The manager jumped from the top-level goal to the next task without asking "what's the constraint?" This is exactly what Theory of Constraints (below) addresses.

**Fix**: Recursive goal decomposition on every manage cycle. Don't just ask "what's next on the list?" — ask "what's the bottleneck preventing us from reaching the goal?"

#### 4. OODA Loop (Boyd)

**The manager Observes and Acts but doesn't Orient or Decide strategically.**

OODA:
- **Observe**: Read tmux pane, check processes ✓
- **Orient**: Understand the broader context, synthesize information ✗
- **Decide**: Choose the highest-leverage action ✗
- **Act**: Send command to dev ✓

The Orient phase is where the manager should synthesize:
- Current model metrics vs goals
- Architecture analysis documents
- direction.md priorities
- Foresight gap analysis
- What the user has been asking about (architecture concerns, lookback questions)

The Decide phase should choose based on orientation, not just the next item on a list.

**Fix**: Add explicit Orient and Decide phases to the manage command. Orient = synthesize all context. Decide = choose highest-leverage action from all possible actions, not just the next on a list.

#### 5. Theory of Constraints (Goldratt)

**The manager optimizes non-constraints while ignoring the actual constraint.**

The current constraint was: **V4's architecture has a fundamental limitation (no temporal context) that caps its performance ceiling.** No amount of GPU optimization, checkpoint engineering, or training recipe tuning removes this constraint. The manager spent hours optimizing GPU utilization (a non-constraint) while the architecture constraint sat unaddressed.

Goldratt's 5 Focusing Steps:
1. **Identify** the constraint → V4's single-day snapshot limits captured return
2. **Exploit** the constraint → Maximize V4's performance within its limits (train well)
3. **Subordinate** everything to the constraint → Don't invest in things that don't address the constraint
4. **Elevate** the constraint → Redesign architecture (V5)
5. **Repeat** → Find the next constraint

The manager should have run Steps 1-2 before investing GPU-hours. "What is the constraint on V4's performance? Is it training (fixable with more epochs) or architecture (needs redesign)?"

**Fix**: Add constraint analysis to every manage cycle. Before recommending the next task, ask: "What is the single biggest constraint preventing goal achievement? Is the next task addressing that constraint?"

#### 6. Bayesian Reasoning for Decision-Making

**The manager has no uncertainty model.**

The manager treats the goal tracker as deterministic: "Next goal is training → do training." A Bayesian manager would maintain beliefs:

```
P(V4 architecture is optimal | data seen) = ???
P(V4 will exceed V3 | train to convergence) = ???
P(temporal context would improve V4 | foresight analysis) = ???
```

After V4 showed CR=0.0184 in epoch 1 (strong!), the manager should have updated:
- P(V4 is good) ↑ — but also asked "could it be even better?"
- The foresight analysis shows temporal features dominate → P(temporal improvements help) is high
- The cost of investigating architecture before training is low (hours) vs the cost of training the wrong thing (days)

**Expected value calculation**:
- Train V4 now: ~58% annualized (if metrics hold)
- Spend 2 hours reviewing architecture, then train improved V4/V5: potentially ~125% annualized
- The information value of architecture review >> cost of delay

**Fix**: Add expected value reasoning to the manage cycle. When choosing between "do the next task" and "investigate whether the task is right," compute the expected value of information.

---

## Part 3: Redesigned Manager Architecture

### The OODA-Constraints Manager

The new manager operates as a **strategic OODA loop with constraint analysis**, not a task executor:

```
┌──────────────────────────────────────────────────────────────┐
│                    MANAGE COMMAND                              │
│                                                              │
│  1. OBSERVE (operational)                                     │
│     └── Check dev session, capture state                      │
│                                                              │
│  2. ORIENT (strategic — THE KEY ADDITION)                     │
│     ├── Read goal_tracker.md → where are we?                  │
│     ├── Read goals.md → where do we want to be?              │
│     ├── Compute gap → what's the distance?                   │
│     ├── Identify constraint → what's blocking progress?       │
│     ├── Read direction.md → what are the known priorities?    │
│     ├── Read improvement docs → what have we learned?         │
│     ├── Form hypothesis → "the next highest-leverage action   │
│     │   is X because constraint Y is the bottleneck"         │
│     └── Assess uncertainty → how confident am I?             │
│                                                              │
│  3. DECIDE (choose from ALL possible actions)                 │
│     ├── Operational: send dev a task, fix configs, monitor    │
│     ├── Strategic: request architecture review, redesign,     │
│     │   change direction, investigate alternatives            │
│     ├── Meta: update manager skills, update dev rules,        │
│     │   create documentation, refine goals                    │
│     └── Choose action with highest expected value             │
│                                                              │
│  4. ACT                                                       │
│     └── Write manager.md with recommendation                  │
│                                                              │
│  5. LEARN (after every cycle)                                 │
│     ├── Was the last action effective?                         │
│     ├── Update beliefs (goal_tracker, improvements)           │
│     └── Update own process if a gap was found                │
└──────────────────────────────────────────────────────────────┘
```

### The Orient Phase (What Was Missing)

The Orient phase is the most critical addition. On every manage command, the manager must:

#### A. Gap Analysis (Cybernetic Comparator)

```
Current state:
  - V4 captured return: 0.0184 per 10-day period
  - Annualized estimate: ~58%
  - Architecture: single-day snapshot, 12-dim tokens

Target state (from goals.md):
  - Foresight: 170% in 50 days (~14,919% annualized)
  - Realistic target: 100-150% annualized

Gap: ~2-3x improvement needed on captured return
Question: Is the current architecture capable of closing this gap?
```

#### B. Constraint Identification (Theory of Constraints)

```
What limits V4's captured return from reaching target?

Hypothesis 1: Training recipe (lr, loss weights) → testable, low cost
Hypothesis 2: Architecture (single-day, 12-dim tokens) → high impact if true
Hypothesis 3: Data quality/quantity → unlikely, 11.4M samples is large
Hypothesis 4: Model capacity (4.77M params) → possible but V4 beats V3 with 136K

Most likely constraint: Architecture (temporal blindness)
Evidence: Foresight analysis shows momentum/z-score features are most predictive
Cost to investigate: 1-2 hours of architecture review
Cost if wrong (train wrong architecture): 10-20 hours of GPU time
Expected value of investigation: HIGH
```

#### C. Hypothesis Formation (Scientific Method)

```
Before recommending any action, state:
  Hypothesis: "[specific claim about what will improve performance]"
  Test: "[how to validate or falsify]"
  Success criteria: "[specific metrics that would confirm]"
  Failure criteria: "[what would make us abandon this approach]"
```

#### D. Action Space Enumeration (Means-Ends Analysis)

Instead of picking the next item from a list, enumerate ALL possible actions:

```
Possible actions right now:
  1. Train V4 as-is (operational — follow the list)
  2. Review V4 architecture for limitations (strategic)
  3. Analyze foresight signals vs V4's capabilities (strategic)
  4. Update direction.md with new findings (meta)
  5. Design V5 based on known gaps (strategic)
  6. Optimize V4 token preparation (operational)
  7. Review and improve dev's skills/rules (meta)
  ...

Rank by expected value:
  #2 and #3 have highest information value — investigating architecture
  before committing GPU-hours dominates all other choices.
```

### The Decide Phase (Bayesian Ranking)

For each possible action, estimate:
- **Expected improvement** to captured return or other key metric
- **Cost** (GPU-hours, dev-hours, delay to next result)
- **Risk** (probability it doesn't work, opportunity cost)
- **Information value** (does this action reduce uncertainty about the right path?)

```
Action: "Review architecture before training"
  Expected improvement: 0 (short term), HIGH (medium term if it leads to V5)
  Cost: 2 hours
  Risk: Low (worst case: confirm V4 is fine)
  Information value: VERY HIGH (resolves uncertainty about V4 vs V5)
  → Expected value: HIGH

Action: "Train V4 now"
  Expected improvement: V4 converges to ~0.02 CR
  Cost: 20+ GPU-hours
  Risk: Medium (may train wrong architecture)
  Information value: LOW (we already know V4 works from smoke test)
  → Expected value: MEDIUM

Decision: Review architecture first.
```

### The Learn Phase (Continuous Self-Improvement)

After every manage cycle, the manager asks:

1. **Was my last recommendation the right one?**
   - If yes: reinforce the reasoning pattern
   - If no: what did I miss? Update the process.

2. **Did I miss any information I should have used?**
   - Were there documents I should have read?
   - Were there questions I should have asked?

3. **Is my own process still optimal?**
   - Am I spending too much time on operational checks vs strategic thinking?
   - Are there new frameworks or patterns I should adopt?

---

## Part 4: Concrete Changes to Manager Skills

### New File: `process_framework.md`

A new file in the manager skills folder that defines the OODA-Constraints process:

```
Every manage command follows this sequence:

1. OBSERVE: Check dev session state (existing Steps 1-4)

2. ORIENT: Strategic analysis (NEW — the critical addition)
   a. Read goal_tracker.md → current state + metrics
   b. Read goals.md → target state
   c. Compute gap: current vs target
   d. Identify constraint: what single factor limits progress most?
   e. Read direction.md, improvement docs → known priorities
   f. Form hypothesis: "the next highest-leverage action is X because Y"
   g. Assess uncertainty: how confident? what would change my mind?

3. DECIDE: Enumerate actions, rank by expected value
   - Operational actions (train, fix, monitor)
   - Strategic actions (review architecture, redesign, pivot)
   - Meta actions (update skills, update rules, document)
   - Choose the action with highest expected value

4. ACT: Write manager.md with recommendation

5. LEARN: Update beliefs and process
   - Was last recommendation effective?
   - What was missed? Update process.
```

### Changes to SKILL.md

The "manage" command definition needs to change from:

```
Current: observe → decision tree → goal tracker → plan next task
```

To:

```
New: observe → ORIENT (gap analysis + constraint identification + hypothesis) →
     DECIDE (enumerate all actions, rank by expected value) →
     ACT (write recommendation) → LEARN (update beliefs)
```

### Changes to decision_tree.md

Add a new top-level branch before all operational branches:

```
[Dev session checked]
        │
        ├── (FIRST, ALWAYS) STRATEGIC ORIENT
        │       │
        │       ├── What is the current constraint?
        │       │       ├── Architecture limitation? → Review before training
        │       │       ├── Training recipe? → Experiment with hyperparams
        │       │       ├── Data quality? → Investigate data pipeline
        │       │       ├── Infrastructure? → Fix tooling
        │       │       └── Unknown? → Investigate before acting
        │       │
        │       ├── Is the current plan still the right plan?
        │       │       ├── Have new facts emerged? (metrics, analysis, user feedback)
        │       │       ├── Has the gap changed? (closer or further from goal)
        │       │       └── Should we pivot? (architecture redesign, new approach)
        │       │
        │       └── What has the highest expected value RIGHT NOW?
        │               ├── Investigate (reduce uncertainty)
        │               ├── Execute (we're confident in the plan)
        │               ├── Document (capture knowledge for future)
        │               └── Meta-improve (fix the process itself)
        │
        └── (THEN) OPERATIONAL DECISION TREE (existing branches)
```

### Changes to goal_tracker.md

Add sections for:

```
## Current Hypothesis
[What assumption is the current plan testing?]

## Constraint Analysis
[What is the single biggest bottleneck to goal achievement?]

## Belief State
[How confident are we in each part of the plan?]
  - Architecture: [high/medium/low confidence, why]
  - Training recipe: [high/medium/low]
  - Data quality: [high/medium/low]
  - Next action: [high/medium/low]
```

---

## Part 5: How This Would Have Changed the Outcome

If the redesigned manager had run when V4 token prep was complete and dev was idle:

```
OBSERVE: Dev idle, V4 token prep complete (11.4M samples)

ORIENT:
  a. Current: V4 not yet trained. Architecture: single-day, 12-dim tokens.
  b. Target: Captured return >> 0.0147 (V3). Close gap to foresight.
  c. Gap: Unknown — V4 hasn't been evaluated yet. But architecture is untested.
  d. Constraint analysis:
     - Read direction.md: P2 says "add price/momentum features (strongest predictor)"
     - V4 has 1 compressed price token with 6 values. V2/V3 had 5-day temporal encoder.
     - V4 REMOVED temporal context that V2/V3 had.
     - Foresight analysis: momentum/z-score features are most predictive.
     - CONSTRAINT IDENTIFIED: V4's single-day snapshot may miss temporal signals.
  e. Hypothesis: "V4's attention over raw contracts compensates for lost temporal context"
     - This needs testing, but the cost of being wrong is 20+ hours of GPU time.
  f. Uncertainty: MEDIUM-HIGH. We don't know if the architecture is right.

DECIDE:
  Possible actions:
  1. Train V4 now (cost: 20hr, risk: training wrong architecture)
  2. Ask dev to document V4 architecture, analyze temporal limitations (cost: 1hr, risk: none)
  3. Compare V4 design to foresight analysis signals (cost: 30min, risk: none)
  4. Run quick smoke test to get baseline metrics (cost: 4hr, risk: low)

  Expected value ranking:
  #2 + #3 dominate: information value is very high, cost is very low.
  Even if V4 is fine, documenting the architecture is valuable.
  → RECOMMEND: Architecture review before training.

ACT: Write manager.md recommending architecture documentation and gap analysis.

LEARN: Note that the goal tracker had "launch training" as next step,
       but strategic analysis overrode the list. Update process to always
       run strategic orient before operational execution.
```

This would have surfaced the temporal limitation **before** 20+ hours of GPU time on the wrong architecture.

---

## Part 6: Summary of Changes Needed

| Component | Current State | Proposed Change |
|-----------|--------------|-----------------|
| SKILL.md manage command | Observe → decision tree → next task | **Observe → ORIENT → DECIDE → ACT → LEARN** |
| decision_tree.md | Operational branches only | **Add strategic orient as first branch** |
| goal_tracker.md | Task checklist | **Add hypothesis, constraint analysis, belief state** |
| NEW: process_framework.md | Doesn't exist | **OODA-Constraints framework definition** |
| manager.md output | Task recommendation | **Include: constraint analysis, hypothesis, expected value reasoning** |
| manager_improvements.md | Reactive observations | **Proactive: strategic learnings, process improvements, belief updates** |

### The Single Most Important Change

**Before asking "what should dev do next?", always ask "what is the constraint on reaching our goal, and is the next task addressing that constraint?"**

This single question, applied on every manage cycle, would have caught the architecture limitation, the checkpoint gap, the per-run directory issue, and every other strategic miss — because it forces the manager to think about *why* before *what*.

---

## Part 7: Fastest Path Computation — How the Manager Prioritizes

### The Missing Capability: Critical Path Analysis

The OODA-Constraints framework tells the manager to identify the constraint and form hypotheses. But it doesn't tell the manager **how to choose between competing paths to the goal** when multiple valid options exist. For example, right now:

- **Path A**: Continue V4 training (~20 GPU-hours → ~58% annualized, known ceiling)
- **Path B**: Build V5 from scratch (~2 weeks dev + ~7 days training → ~125% estimated)
- **Path C**: Hybrid — train V4 on GPU while dev builds V5 in parallel

The manager needs a **critical path solver**: given the goal, resources, and known options, what sequence of actions reaches the goal fastest with the least risk?

### Framework: Expected Time-to-Goal (ETG) Analysis

For each possible path, compute:

```
ETG = T_implement + T_train + T_evaluate + T_iterate × P(iteration_needed) + T_wasted × P(wrong_path)
```

Where:
- `T_implement` = time to build/modify code
- `T_train` = GPU time for training
- `T_evaluate` = time to evaluate results
- `T_iterate` = time for one iteration cycle if results aren't good enough
- `P(iteration_needed)` = probability we need to iterate
- `T_wasted` = time lost if this path hits a ceiling and we must start over
- `P(wrong_path)` = probability this path can't reach the goal

### Applied to Current Decision: V4 vs V5

```
Path A: Continue V4 training
  T_implement: 0 hours (already built)
  T_train: 20 hours (GPU)
  T_evaluate: 1 hour
  T_iterate: 20 hours (retrain with different hyperparams)
  P(iteration_needed): 0.3 (V4 already showed CR=0.0184, likely converges well)
  T_wasted: 20 hours if V4 hits ceiling at ~58% annualized and we need V5 anyway
  P(wrong_path): 0.7 (V4's architectural ceiling is likely below target)

  ETG = 0 + 20 + 1 + 20×0.3 + 20×0.7 = 0 + 20 + 1 + 6 + 14 = 41 hours
  But: only reaches ~58% annualized, then must build V5 anyway → add V5 ETG
  Total ETG to goal: 41 + Path B ETG = 41 + ~300 = ~341 hours

Path B: Build V5, skip V4
  T_implement: 80 hours (2 weeks dev time for data prep + model + loader)
  T_train: 170 hours (25 epochs × 7 hr/epoch)
  T_evaluate: 2 hours
  T_iterate: 170 hours (retrain)
  P(iteration_needed): 0.5 (new architecture, higher uncertainty)
  T_wasted: 250 hours if V5 design is wrong
  P(wrong_path): 0.2 (design is grounded in analysis, but untested)

  ETG = 80 + 170 + 2 + 170×0.5 + 250×0.2 = 80 + 170 + 2 + 85 + 50 = 387 hours
  But: reaches ~125% annualized target directly
  Total ETG to goal: ~387 hours

Path C: Hybrid — V4 trains on GPU while dev builds V5
  Dev time: 80 hours building V5 (GPU running V4 simultaneously — no conflict)
  V4 completes in ~20 hours → gives verified baseline
  Then V5 training: 170 hours

  ETG = max(20, 80) + 170 + 2 + 170×0.4 + 250×0.15
      = 80 + 170 + 2 + 68 + 37.5 = 357.5 hours

  Benefits:
  - V4 provides verified baseline to compare V5 against (P(wrong_path) drops from 0.2 to 0.15)
  - P(iteration_needed) drops from 0.5 to 0.4 (learn from V4's training dynamics)
  - GPU is never idle during dev implementation
  - If V5 design fails, V4 is already trained as fallback
  Total ETG to goal: ~358 hours
```

### Decision Matrix

| Path | ETG (hours) | Reaches Goal? | Risk | GPU Idle Time |
|------|-------------|---------------|------|---------------|
| A: V4 only | ~341 (must add V5 later) | No (ceiling ~58%) | Low short-term, high long-term | 0 |
| B: V5 only | ~387 | Yes (~125%) | Medium (untested design) | 80 hours (during implementation) |
| **C: Hybrid** | **~358** | **Yes (~125%)** | **Lowest** (V4 baseline + V5) | **0** (V4 trains during V5 dev) |

**Path C dominates**: Lower risk than B (V4 baseline as reference), lower total time than A+B (parallel execution), zero GPU idle time.

### The Manager's Priority Rules

Based on the ETG analysis, the manager should follow these priority rules:

```
PRIORITY RULES (in order):

1. NEVER let the GPU sit idle when there's useful work to do.
   - If dev is implementing code, the GPU should be training something.
   - If the GPU is training, dev should be working on the next thing.

2. PREFER paths that generate information early.
   - A short V4 training run gives verified metrics to compare V5 against.
   - Architecture review before training prevents wasted GPU time.
   - Smoke tests before full runs prevent wasted GPU time.
   - Pattern: cheap information gathering → expensive execution.

3. PREFER parallel execution over sequential when resources allow.
   - GPU and dev are separate resources — use both simultaneously.
   - Train current model while developing next model.
   - Run evaluation while planning next experiment.

4. INVEST in design iteration over training iteration.
   - One design review (2 hours) can save 20 hours of training the wrong thing.
   - Architecture improvements compound (better design → faster convergence → less training time).
   - Training iteration (different hyperparams) gives diminishing returns.
   - Pattern: iterate on WHAT to build, not on HOW to tune what's already built.

5. ADDRESS the constraint, not the symptoms.
   - If the constraint is architecture → redesign before training.
   - If the constraint is training recipe → experiment with hyperparams.
   - If the constraint is data → fix the pipeline.
   - If the constraint is infrastructure → fix tooling.
   - Never optimize a non-constraint.

6. PREFER reversible actions over irreversible ones.
   - Architecture review is reversible (you can still train V4 after).
   - Launching a 20-hour training run is semi-reversible (can stop, but GPU time is lost).
   - Deleting data is irreversible.
   - When uncertain, choose the action you can undo.
```

### How This Applies to the V4 vs V5 Decision

Applying the priority rules:

1. **GPU idle**: V4 is ready to train → launch V4 training now (Rule 1)
2. **Parallel execution**: While V4 trains, dev starts V5 implementation (Rule 3)
3. **Information early**: V4 training gives verified metrics to calibrate V5 expectations (Rule 2)
4. **Design over training iteration**: Dev time goes to V5 design, not V4 hyperparameter tuning (Rule 4)
5. **Constraint**: The constraint is architecture → V5 addresses it, V4 tuning doesn't (Rule 5)

**Manager's recommendation would be**:
> "Launch V4 full training now (--resume from archive, --weights-only). While GPU trains V4, dev starts implementing V5 data preparation and model architecture. V4 gives us a verified baseline in ~20 hours. V5 development proceeds in parallel. When V5 is ready, compare directly against V4. This is the fastest path: zero GPU idle time, zero dev idle time, maximum information."

### The Critical Path Insight

The fastest path to the goal is **not** the shortest individual task — it's the sequence that:
1. Keeps all resources (GPU + dev) utilized
2. Generates information that reduces uncertainty
3. Addresses the actual constraint (architecture, not tuning)
4. Minimizes total wasted time across all possible outcomes

The manager should compute this on every manage cycle, not just follow a task list.

---

## Part 8: Framework Integration Summary

Each management framework contributes a specific capability to the prioritization logic:

| Framework | Contribution to Priority Decision |
|-----------|----------------------------------|
| **Cybernetic Control** | Gap measurement: how far from goal? Is the gap shrinking per unit of effort? |
| **Scientific Method** | Before training: what hypothesis are we testing? What would falsify it? |
| **Means-Ends Analysis** | Recursive decomposition: break the goal into subgoals, identify which subgoal is blocked |
| **OODA Loop** | Orient before deciding: synthesize all context before picking the next task |
| **Theory of Constraints** | One constraint at a time: what's the bottleneck? Only work on the bottleneck. |
| **Bayesian Reasoning** | ETG computation: P(wrong_path), P(iteration_needed), expected value of information |

**Combined into one question the manager asks on every cycle:**

> "Given the current gap to the goal, the identified constraint, the available resources (GPU + dev), and the uncertainty about each path — what is the sequence of actions that minimizes expected time-to-goal while keeping all resources utilized?"

This replaces the current question: "What's the next item on the task list?"
