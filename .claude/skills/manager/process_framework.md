# Manager Process Framework — OODA-Constraints

This is the core process the manager follows on every "manage" command. It replaces the previous task-list approach with strategic reasoning.

## The OODA-Constraints Loop

Every manage command follows this sequence. **No step may be skipped.**

### 1. OBSERVE (Operational)

Check dev session state. Capture what dev is doing. This is the existing Steps 1-4 from SKILL.md.

### 2. ORIENT (Strategic — The Critical Phase)

Before deciding what to do, synthesize all available context:

#### a. Gap Analysis

```
Read goal_tracker.md → current state + latest metrics
Read goals.md → target state
Compute gap: how far from the goal?
Is the gap shrinking? At what rate? When will we reach the goal at current pace?
```

#### b. Constraint Identification (Theory of Constraints)

```
What is the SINGLE biggest factor limiting progress toward the goal?

Categories:
  - Architecture limitation → Design is the bottleneck, not execution
  - Training recipe → Hyperparameters need tuning
  - Data quality/quantity → Pipeline issues
  - Infrastructure → Tooling, performance, checkpointing
  - Knowledge gap → We don't understand something well enough to proceed
  - Unknown → Need investigation before any action

The constraint is what to work on. Everything else is a non-constraint — don't optimize it.
```

#### c. Hypothesis Formation (Scientific Method)

```
Before recommending ANY action, state:
  Hypothesis: "Doing X will improve [metric] by [amount] because [reasoning]"
  Test: "We can validate this by [specific experiment]"
  Success criteria: "[metric] exceeds [threshold]"
  Failure criteria: "[metric] below [threshold] after [timeframe] means we should [pivot]"
  Cost of being wrong: "[hours/resources wasted] if hypothesis is false"
```

#### d. Path Enumeration (Means-Ends Analysis)

```
List ALL possible actions, not just the next item on a list:

  Operational: train, fix code, run pipeline, monitor
  Strategic: review architecture, redesign, analyze results, pivot approach
  Meta: update skills, update rules, document findings, improve process
  Parallel: things that can run simultaneously (GPU + dev)

For each action, estimate:
  - Expected improvement to key metric (captured return)
  - Cost (GPU-hours, dev-hours, elapsed wall time)
  - Risk (P(wrong_path), P(iteration_needed))
  - Information value (does this reduce uncertainty?)
```

#### e. Expected Time-to-Goal (ETG) Computation

```
For each candidate path:

ETG = T_implement + T_train + T_evaluate + T_iterate × P(iterate) + T_wasted × P(wrong)

Pick the path with lowest ETG that reaches the goal.
Factor in parallel execution: GPU and dev are separate resources.
```

### 3. DECIDE (Rank by Expected Value)

Apply the **Priority Rules** (in order):

```
1. NEVER let GPU sit idle when there's useful work to do.
2. PREFER actions that generate information early (cheap investigation before expensive execution).
3. PREFER parallel execution (GPU trains while dev designs/implements).
4. INVEST in design iteration over training iteration (2hr architecture review > 20hr retraining).
5. ADDRESS the constraint, not symptoms (don't optimize non-constraints).
6. PREFER reversible actions when uncertain (investigate before committing).
```

Choose the action (or combination of parallel actions) with highest expected value.

### 4. ACT

Write `manager.md` with:
- Observation summary
- Constraint analysis (what's the bottleneck?)
- Hypothesis (what are we testing?)
- ETG comparison (why this path?)
- Recommended action(s) — including parallel tracks if applicable
- Success/failure criteria

### 5. LEARN

After every cycle:
- Was the last recommendation effective? (Check results)
- What was missed? (Any information not used?)
- Update goal_tracker.md: metrics, hypothesis, constraint, belief state
- Update management_improvements.md if a process gap was found
- Is the manager's own process still optimal?

---

## Priority Rules Reference

| # | Rule | Rationale |
|---|------|-----------|
| 1 | Never let GPU sit idle | GPU-hours are the scarcest resource. Train something while dev works on the next thing. |
| 2 | Generate information early | Cheap investigation (1-2hr) prevents expensive mistakes (20hr training wrong architecture). |
| 3 | Parallel execution | GPU + dev are independent resources. Use both simultaneously. |
| 4 | Design over tuning | Architecture improvements compound. Hyperparameter tuning has diminishing returns. |
| 5 | Address the constraint | Only the bottleneck matters. Optimizing a non-constraint is waste. |
| 6 | Reversible over irreversible | When uncertain, pick actions you can undo. Investigate before committing. |

---

## ETG Quick Reference

```
ETG = T_implement + T_train + T_evaluate + T_iterate × P(iterate) + T_wasted × P(wrong)

Compare paths:
  Path with lowest ETG that reaches the goal wins.
  If multiple paths have similar ETG, prefer the one with:
    - Lower P(wrong) → less risk
    - Higher information value → learns more even if it fails
    - Better parallelization → keeps GPU busy
```

---

## Constraint Categories & Typical Actions

| Constraint | Signal | Typical Action |
|-----------|--------|----------------|
| Architecture | Metrics plateau despite good training (loss converges, metrics don't improve) | Review architecture, analyze limitations, redesign |
| Training recipe | Loss not converging, or converging to wrong solution | Experiment with lr, loss weights, schedule |
| Data quality | Good architecture + training but poor metrics | Investigate data pipeline, check for bugs, label quality |
| Infrastructure | Slow training, crashes, no checkpointing | Fix tooling before wasting GPU time |
| Knowledge gap | Uncertain which path is right | Investigate, document, analyze before committing resources |

---

## Anti-Patterns to Avoid

| Anti-Pattern | Why It's Wrong | Correct Behavior |
|---|---|---|
| Following the task list without questioning | The list may be stale or wrong | Run ORIENT every cycle — the list is a starting point, not a mandate |
| Training before understanding the architecture | Wastes GPU-hours if architecture is wrong | Architecture review costs hours, training costs days |
| Optimizing GPU util while the architecture is the bottleneck | Non-constraint optimization | Address the constraint first |
| Celebrating early metrics without questioning why | CR=0.0184 in epoch 1 is great — but ask "can it be better?" | Every result is an opportunity to orient: what does this tell us? |
| Starting full training without smoke test | Hours wasted if something is broken | Unit test → smoke test → full (always) |
| Sequential when parallel is possible | Dev waits while GPU trains, or GPU waits while dev codes | Identify parallel tracks on every cycle |
