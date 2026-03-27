# Manager Process Framework — OODA-Constraints

This is the core process the manager follows on every "manage" command. It replaces the previous task-list approach with strategic reasoning.

## The OODA-Constraints Loop

Every manage command follows this sequence. **No step may be skipped.**

### 1. OBSERVE (Operational)

Check dev session state. Capture what dev is doing. This is the existing Steps 1-4 from SKILL.md.

### 2. ORIENT (Strategic — The Critical Phase)

Before deciding what to do, synthesize all available context:

#### a. Ground Truth Verification

```
Do NOT trust cached documents alone. Independently verify the actual state:
  - Check filesystem: ls models/, ls cache/, check run dirs exist
  - Check running processes: ps aux | grep train/prepare
  - Check GPU state: nvidia-smi (is GPU idle or busy?)
  - Check latest log files: tail training/pipeline logs
  - Check checkpoint files: do they exist? what epoch? what metrics?

Only THEN cross-reference with goal_tracker.md.
If reality differs from goal_tracker → update goal_tracker, trust reality.
```

#### b. Gap Analysis

```
Read goal_tracker.md → current state + latest metrics (verified against reality above)
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

### 2f. Design Readiness Check

Before recommending any implementation task, verify a sufficient design exists:

```
Is there an implementation task in the candidate actions?
  │
  ├── YES → Check: does a design document exist in research/?
  │       │
  │       ├── NO design doc → Constraint is KNOWLEDGE GAP
  │       │       Action: Investigate first. Read existing code (model.py, config.py,
  │       │       direction.md, foresight analysis). Write design to research/.
  │       │       Do NOT tell dev to implement without a design.
  │       │
  │       ├── Design doc exists but INCOMPLETE (missing token spec, architecture,
  │       │   loss function, training recipe, or data format) → Fill the gaps
  │       │       Action: Investigate the missing parts, update research/ doc.
  │       │
  │       └── Design doc exists and COMPLETE → Ready to implement
  │               Verify: token spec, architecture, loss, recipe, data format all present.
  │               Include the research/ doc path in the prompt to dev.
```

**Research folder**: `.claude/skills/manager/research/`
- The manager writes all investigations, architecture analyses, and design documents here
- Seeded with existing designs (e.g., `v5_design.md`)
- The manager reads and iterates on these documents across manage cycles
- When a design is ready, it's referenced in the prompt to dev

**Manual docs**: `/home/ubuntu/workspace/RLQuest-manager/manual_docs/`
- Contains documents written during manual user-manager sessions
- The manager may read these for context but writes its own research to `research/`

### 3. DECIDE (Rank by Expected Value)

Apply the **Priority Rules** (in order):

```
1. NEVER let ANY resource sit idle (GPU, CPU, dev) when there's useful work to do.
2. PREFER actions that generate information early (cheap investigation before expensive execution).
3. PREFER parallel execution (GPU trains while dev designs/implements).
4. INVEST in design iteration over training iteration (2hr architecture review > 20hr retraining).
5. ADDRESS the constraint, not symptoms (don't optimize non-constraints).
6. PREFER reversible actions when uncertain (investigate before committing).
```

Choose the action (or combination of parallel actions) with highest expected value.

### 3b. VALIDATE (Operational Gate — MANDATORY before ACT)

**CRITICAL: Before writing any recommendation, run it through the operational checks below. The manager must enforce the same rules on its OWN recommendations that it enforces on dev. This step exists because strategic ORIENT can identify the right action but skip the operational prerequisites.**

For EVERY recommended action, check:

```
DATA PIPELINE (prepare_*, process_*)?
  □ Smoke test on representative data (include large quarters)?
  □ CPU multi-core utilization measured?
  □ Memory stability verified?
  □ Throughput measured and extrapolated to full run?
  □ If ANY unchecked → add validation step BEFORE the action in the prompt
  □ Ref: long_running_script_guide.md §8, rules/data-pipeline.md

TRAINING (train*.py)?
  □ Unit test passed?
  □ Smoke test passed, GPU util >70%?
  □ AMP+FP16, torch.compile enabled?
  □ Checkpoint/resume verified?
  □ Design doc in research/ complete?
  □ If ANY unchecked → add validation step BEFORE the action in the prompt
  □ Ref: training_evaluation_guide.md, rules/training.md

IMPLEMENTATION (new code)?
  □ Design doc in research/ with complete specs?
  □ If NO → investigate and write design FIRST
  □ Ref: process_framework.md §2f

ALL ACTIONS:
  □ Does this chain to a follow-up? → Include it: "After X, immediately do Y"
  □ Never end with "report when ready" without including the next step
  □ Include success/failure criteria for each action
```

**If any check fails**: Modify the recommendation to include the missing step. Do NOT skip validation because the action is "high priority." High priority = validate faster, not skip validation.

### 4. ACT

Write `manager.md` with:
- Observation summary
- Constraint analysis (what's the bottleneck?)
- Hypothesis (what are we testing?)
- ETG comparison (why this path?)
- **Validation checklist** (which operational checks passed/failed, what was added)
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
| 1 | Never let ANY resource sit idle (GPU, CPU, dev) | GPU-hours, CPU-hours, and dev-hours are all scarce. If GPU is training, CPU should be running data prep. If data prep is running, dev should be writing code. Check all three on every cycle. |
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
| Trusting goal_tracker without verifying filesystem | Cached info may be stale — files moved, runs deleted, processes killed | Always `ls`, `ps aux`, `tail` logs before referencing goal_tracker |
| Prompt that stops at "report when ready" instead of chaining next step | Dev completes task then sits idle for hours while resources waste | Always chain: "After X passes, immediately proceed to Y. Then start Z." Never leave dev idle without the next task. |
| Send monitoring that ends when dev finishes one task | CPU/dev may sit idle while there's more work to do | After dev completes prompted task, run one more ORIENT cycle: "Is there idle resource + available work? If yes, send the next task immediately." |
| Only checking GPU idle, not CPU idle | CPU-bound work (token prep) can run parallel to GPU training | Priority Rule #1 applies to ALL scarce resources: GPU, CPU, dev time. Check all three. |
| Accepting time estimates without validation | "~1-2 hours" was a guess, not measured. V5 processes 5x more data than V4 which took 47min. | Always require performance test on 5+ quarters before accepting ETG estimates. Apply the pre-run validation cycle to ALL scripts, including data prep. Never skip smoke test → perf test → iterate. |
| Skipping pre-run validation for "already tested" scripts | Unit test passed ≠ performance validated. Unit test ran 3 tiny quarters, not representative. | Unit test validates correctness. Performance test on large quarters validates speed. Both required. |
