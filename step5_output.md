## Step 5: Action Plan — 2026-03-28 04:10 UTC (Cycle 7)

### Status: NEW PLAN — Implement 2 step-based features, then launch full training

### All possible actions:
1. **IMPLEMENT subset validation every 2000 steps** (~20 lines in train.py) — MANDATORY before full training
2. **IMPLEMENT step-based early stopping** (~20 lines in train.py) — MANDATORY before full training
3. **LAUNCH full training** with P4 recipe: `--full --lr 1e-4 --constant-lr --no-smoke` — AFTER items 1-2
4. **Investigate return_mae=NaN** — return head broken, wastes 27% gradient budget (deferred, non-blocking for CR)
5. Fix summary token dims 14-19 (deferred)
6. Fix price token dim 15 (deferred)
7. Optimize IntraDayEncoder day-loop (deferred)
8. Fix config.d_ff unused (deferred)
9. Clean up zombie processes (cosmetic, now 10)
10. Consider loss weight rebalancing (future experiment)
11. Design V5-Medium (premature)

### Resources: GPU=IDLE, CPU=IDLE, Dev=IDLE (at prompt)

### Chosen action(s):

**Actions #1, #2, #3 in strict sequence: Implement subset eval -> Implement step-based early stopping -> Unit test both features -> Launch full training.**

**Reasoning:** The constraint is EXECUTION. The recipe is known (constant lr=1e-4, validated by P4 prove-out at 1.2x V3 baseline). Two features remain from the step_based_training.md spec:
- Subset eval every 2000 steps: provides divergence detection in ~13 min vs ~72 min (5.5x faster feedback)
- Step-based early stopping: max waste 104 min vs 18 hours (10x faster stopping)

These are ~40 lines total and ~30 min of coding. The 30 min investment prevents up to 17+ hours of wasted GPU time if full training diverges. GPU is idle — every minute spent coding is cheaper than a minute of divergent training.

**Chain: implement -> unit test -> verify -> launch full training -> monitor first 2000 steps.**

### Validation gate:

**Implementation prerequisites:**
- [ ] Subset eval: `eval_every_steps` parameter added to `train_epoch` (default: 2000)
- [ ] Subset eval: `eval_fn` callable passed to `train_epoch` that runs 10% val data
- [ ] Subset eval: `model.train()` called after each subset eval
- [ ] Subset eval: metrics logged at each step-eval (val_loss, CR at minimum)
- [ ] Step-based early stopping: `step_patience_counter` tracks consecutive non-improving step-evals
- [ ] Step-based early stopping: `step_eval_patience` parameter (default: 8 eval cycles)
- [ ] Step-based early stopping: `train_epoch` returns early_stop signal when patience exhausted
- [ ] Step-based early stopping: outer epoch loop respects early_stop signal

**Unit test before full training:**
- [ ] Run `--unit-test` or short prove-out to verify both features work without error
- [ ] Step-eval fires at expected interval (batch 2000, 4000, etc.)
- [ ] Step-eval metrics are logged and non-NaN (at least val_loss)
- [ ] Step-based early stopping does NOT trigger during short test (patience should be large enough)

**Training launch prerequisites:**
- [ ] Unit test passed with both features active
- [ ] GPU util >70% during unit test
- [ ] AMP+FP16 enabled (already confirmed)
- [ ] torch.compile active (already confirmed, mode='default')
- [ ] Checkpoint/resume working (already confirmed — 9 fields in checkpoint)
- [ ] Per-run directory created (already confirmed — run_TIMESTAMP_full_* pattern)

**All actions:**
- [x] Chained to follow-up: implement -> test -> launch -> monitor first 2000 steps
- [x] Success criteria defined (below)
- [x] Failure criteria defined (below)

### Parallel tracks:
| Resource | During Implementation (~30 min) | During Unit Test (~5 min) | During Full Training |
|----------|-------------------------------|--------------------------|---------------------|
| GPU | Idle | Unit test | Full training with P4 recipe |
| CPU | Idle | Data loading | Data loading |
| Dev | Coding both features | Verify features | Monitor first 2000-step eval, then let run |

### Success criteria:
- **Implementation:** Both features added in ~40 lines, no regressions in existing train.py functionality
- **Unit test:** Step-eval fires correctly, metrics logged, no crashes, no NaN in subset val loss
- **Full training launch:** First 2000-step eval shows val_loss < 0.70 (reasonable range based on prove-out VL=0.61-0.70), prec > 0.3, rec > 0.3
- **Full training completion:** Test CR > 0.0176 (exceed P4 prove-out), prec > 0.5, rec > 0.5, stable prec/rec through training
- **Ultimate target:** Test CR >= 0.0192 (match V1 best ever)

### Failure criteria:
- **Implementation bugs:** Step-eval crashes or corrupts training state (model not returned to train mode, optimizer state corrupted). Mitigation: unit test catches this before full run.
- **Full training collapse:** Prec/rec collapse at full data scale despite constant LR. If this happens: the 5x data scale changes dynamics. Try lr=5e-5 constant or reduce loss weights to sum=1.0.
- **return_mae=NaN persists:** Expected (known issue #13), not blocking. Investigate after full training completes. The return head (loss weight=0.8) is consuming 27% of gradient budget for zero useful signal — fixing this could meaningfully improve CR.
- **Step-eval overhead too high:** Subset eval should take ~49 sec per call (127 batches at 156 batches/min). If >2 min, reduce subset to 3 chunks (5%) instead of 6 chunks (10%).

---

## Recommended Prompt

```
Implement two step-based training features in train.py, then launch full training. These are the last items from the step-based training spec before we can run the full training with the proven P4 recipe.

## Feature 1: Subset Validation Every 2000 Steps

Add mid-epoch evaluation to catch divergence early (13 min vs 72 min for full-epoch eval).

### Changes to `train_epoch()`:

Add two new parameters:
- `eval_every_steps: int = None` — if set, run subset validation every N batches
- `eval_fn: callable = None` — function that runs subset validation and returns metrics dict

Inside the batch loop, AFTER the existing checkpoint logic (around line 162), add:

```python
# Subset validation every eval_every_steps batches
if eval_fn is not None and eval_every_steps and batch_idx > 0 and batch_idx % eval_every_steps == 0:
    sub_metrics = eval_fn()
    logger.info(f"    [step-eval @ batch {batch_idx}] VL={sub_metrics['loss']:.4f} "
                f"CR={sub_metrics.get('captured_return', 0):.4f} "
                f"Prec={sub_metrics.get('precision', 0):.3f} Rec={sub_metrics.get('recall', 0):.3f}")
    model.train()  # CRITICAL: evaluate() sets model.eval(), must restore train mode
    # Check step-based early stopping
    if sub_metrics.get('early_stop', False):
        logger.info(f"    Step-based early stopping triggered at batch {batch_idx}")
        return {'loss': running_loss / (batch_idx + 1), 'early_stop': True}
```

### Changes to `train()`:

Build a subset validation loader with ~10% of val data (6 of 63 chunks):
```python
# Create subset val loader for step-based evaluation
subset_val_dataset = V5SequentialDataset(val_chunks[:6], ...)  # 10% of val data
subset_val_loader = DataLoader(subset_val_dataset, batch_size=config.batch_size, ...)
```

Create the eval_fn as a closure that:
1. Calls `evaluate(model, subset_val_loader, criterion, device)`
2. Tracks best subset val score and patience counter
3. Returns metrics dict with an `early_stop` flag if patience exhausted

Pass `eval_every_steps=2000` and `eval_fn` to `train_epoch()`.

## Feature 2: Step-Based Early Stopping

Instead of patience counted in full epochs (which could waste 18 hours), count patience in step-eval cycles (max waste ~104 min).

### Implementation:

In `train()`, create a closure for eval_fn that tracks:
- `best_step_score` — best captured_return seen at any step-eval
- `step_patience_counter` — incremented when step-eval doesn't improve best_step_score
- `step_eval_patience = 8` — number of non-improving step-evals before stopping

The eval_fn closure should:
1. Run `evaluate(model, subset_val_loader, criterion, device)`
2. Compare `captured_return` to `best_step_score`
3. If improved: reset patience counter, update best_step_score
4. If not improved: increment patience counter
5. If patience exhausted: set `early_stop = True` in returned metrics
6. Return full metrics dict including `early_stop` flag

In the outer epoch loop, check the return value from `train_epoch()`. If it contains `early_stop: True`, break out of the epoch loop.

**Important:** The step-based early stopping should work ALONGSIDE the existing epoch-based early stopping, not replace it. The step-based check catches fast divergence mid-epoch. The epoch-based check (patience=15 on full validation) catches gradual degradation across epochs.

## After Implementation: Unit Test

Run a quick verification:
```bash
cd /home/ubuntu/workspace/RLQuest && python -m firstrate_learning_v5.train --prove-out --lr 1e-4 --constant-lr --tag step_eval_test
```

Let it run for at least 2001 batches (one step-eval trigger). Verify:
1. Step-eval fires at batch 2000
2. Step-eval metrics are logged (VL, CR, Prec, Rec)
3. No crash, no NaN in subset val loss
4. Model continues training after step-eval (train mode restored)
5. Kill after verification — don't need full prove-out

## After Unit Test: Launch Full Training

```bash
cd /home/ubuntu/workspace/RLQuest && python -m firstrate_learning_v5.train --full --lr 1e-4 --constant-lr --no-smoke --tag full_constant_v1
```

This is the P4 recipe (proven stable on 20% data, 1.2x V3 baseline) with step-based eval for safety.

Expected behavior at full scale:
- ~11,325 batches/epoch, ~72 min/epoch
- Step-eval at batches 2000, 4000, 6000, 8000, 10000 (~5 per epoch)
- First divergence signal at ~13 min
- If stable: let it run. Best epoch may be later than prove-out (more data = slower convergence per epoch).
- Step-based early stopping (patience=8 step-evals) catches plateau in ~104 min max.

## What to Report After Launch

After the first step-eval fires (batch 2000, ~13 min), report:
1. Subset val loss
2. Subset CR, precision, recall
3. Train loss trend (is it declining?)
4. GPU utilization
5. Any errors or warnings

Then let it continue running. The step-based eval and early stopping will manage it automatically.
```

---

## Persistent Notes
- **Actions taken history:**
  - Cycle 1 (~20:35 UTC): Wrote "launch training" prompt. NOT DELIVERED to dev.
  - Cycle 2 (~22:00 UTC): Wrote improved "launch training" prompt. NOT DELIVERED to dev.
  - Cycle 3 (~22:30 UTC): Identified delivery bottleneck. Training WAS launched independently.
  - Cycle 4 (~22:35 UTC): Training crashed at batch 2350 (OOM). Analyzed root cause. Planned 3 fixes.
  - Cycle 5 (~22:45 UTC): Planned OOM fix + relaunch. Fixes were applied, training relaunched successfully.
  - Cycle 6 (~00:25 UTC): Planned stop + prove-out + LR sweep. Dev executed all 4 experiments successfully.
  - Cycle 7 (~04:10 UTC): THIS CYCLE. Planning step-based feature implementation + full training launch.
- **Pending items:**
  - return_mae=NaN investigation (after full training — return head broken, 27% gradient waste)
  - Fix summary token dims 14-19 (after training validated)
  - Fix price token dim 15 (after training validated)
  - Optimize IntraDayEncoder day-loop (after training validated)
  - Fix config.d_ff unused bug (maintenance)
  - Clean up zombie processes (cosmetic, now 10)
  - Consider loss weight rebalancing / per-head LR (future experiment to unlock cosine schedule)
  - Consider disabling return head entirely (w_return=0.0) to reclaim 27% gradient budget (future experiment)
- **Validation patterns:**
  - torch.compile mode='reduce-overhead' causes OOM — NEVER use again
  - Per-epoch-only checkpointing inadequate for long epochs — RESOLVED
  - Cosine schedule flat over 566K steps — RESOLVED via constant LR
  - Multi-task loss weight sum 3.0 amplifies effective LR — compensated by lr=1e-4
  - Cosine decay triggers multi-task loss conflict — RESOLVED via constant LR
  - return_mae=NaN across ALL experiments — return head broken, not blocking CR
  - model.eval()/model.train() toggle during step-eval is a correctness risk — unit test must verify
- **Effective prompts:**
  - Detailed code change specs with exact locations (line numbers, function names) work well
  - Including "why" for each change helps dev prioritize
  - Sequencing instructions (implement -> test -> launch) prevent skipped prerequisites
  - Including verification steps with expected outputs catches bugs early
  - Kill-after-verification pattern for unit tests saves time
- **Watch items for next cycle:**
  - Has dev implemented subset eval every 2000 steps? Check train.py for eval_every_steps.
  - Has dev implemented step-based early stopping? Check train.py for step_patience_counter.
  - Did unit test pass? Check for step-eval log lines at batch 2000.
  - Has full training been launched? Check for run dir matching `run_*_full_constant_v1`.
  - If full training running: check first step-eval metrics (batch 2000, ~13 min in).
  - First step-eval: VL < 0.70, prec > 0.3, rec > 0.3, train loss declining.
  - return_mae=NaN still expected (known issue, not blocking).
  - Zombie process count (was 10).
  - Goal tracker is current as of cycle 6 — will need update after full training launches.
