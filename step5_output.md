## Step 5: Action Plan — 2026-03-28 00:25 UTC

### Status: NEW PLAN — Stop training, implement prove-out mode, fix cosine schedule, run LR sweep

### All possible actions:
1. **STOP current training** — epoch 3 still running (batch ~4600/11325), train loss rising (0.5924), wasting GPU time (BLOCKING, IMMEDIATE)
2. **Implement --prove-out flag** — 20% data, 5 epochs, scaled cosine, --lr override, --tag arg (~30 lines of code)
3. **Fix cosine schedule to scale to actual run length** — currently total_steps = max_epochs * batches_per_epoch = 566K, makes cosine flat for first 10 epochs (ROOT CAUSE of mode collapse)
4. **Run LR sweep in prove-out mode** — P1: lr=1e-4 cosine, P2: lr=5e-5 cosine, P3: lr=3e-4 scaled cosine, P4: lr=1e-4 constant (~90 min each)
5. **Relaunch full training with winning recipe** — only after sweep identifies stable recipe
6. Fix summary token dims 14-19 (deferred)
7. Fix price token dim 15 (deferred)
8. Optimize IntraDayEncoder day-loop (deferred)
9. Fix config.d_ff unused (deferred)
10. Clean up zombie processes (cosmetic)
11. Design V5-Medium (premature)

### Resources: GPU=BUSY (training epoch 3, WASTING TIME), CPU=BUSY (training), Dev=IDLE (at prompt)

### Chosen action(s):

**Actions #1, #2, #3, #4 in sequence: Stop training -> Implement prove-out + fix cosine -> Run LR sweep.**

**Reasoning:** The constraint is TRAINING RECIPE. The root cause is identified with mathematical precision: cosine schedule over 566K steps = 99.6% of peak LR at epoch 2, combined with loss weights summing to 3.0 = effective LR ~9e-4 in single-task terms. Current training (epoch 3, batch ~4600) continues the degradation pattern (train loss 0.5924, rising). Every minute of continued training is wasted GPU time — best model from epoch 1 is already saved.

The prove-out approach (20% data, 5 epochs, ~90 min/experiment) enables testing 4 LR configurations in ~6 hours vs ~20 hours wasted on patience epochs with the broken recipe. The key insight is that mode collapse is visible by epoch 2 batch-level metrics (train loss rising), so prove-out will reproduce the diagnostic signal.

**Mandatory sequencing:**
1. STOP current training FIRST (reclaim GPU)
2. Implement prove-out mode AND fix cosine schedule (code changes while GPU idle)
3. Run LR sweep P1-P4 in prove-out mode
4. Only relaunch full training with the winning recipe

### Validation gate:

**Implementation prerequisites:**
- [ ] Current training stopped (kill PID 1245000)
- [ ] Best model from epoch 1 preserved (best_model.pt, best_model_epoch1.pt) — VERIFY after stop
- [ ] --prove-out flag added to argparse (mutually exclusive with --unit-test, --smoke-test, --full)
- [ ] --lr override argument added (float, optional)
- [ ] --tag argument added for run dir naming
- [ ] data_loader.py: max_chunks parameter added to V5InterleavedDataset and V5SequentialDataset
- [ ] prove-out: 28 train chunks, 13 val chunks, 5 epochs, warmup=200 steps
- [ ] CRITICAL: cosine total_steps computed from ACTUAL run length (max_epochs * actual_batches_per_epoch), NOT from full-data calculation
- [ ] Run dir named run_TIMESTAMP_proveout_{tag}

**Cosine schedule fix (applies to ALL modes, not just prove-out):**
- [ ] total_steps = max_epochs * batches_per_epoch where batches_per_epoch reflects ACTUAL data size
- [ ] Verify: in prove-out mode, total_steps = 5 * 2265 = 11,325 (NOT 566,250)
- [ ] Verify: cosine_mult at epoch 2 of prove-out = cos(pi * 2/5) = ~0.31 (meaningful decay, not 0.996)

**Unit test before sweep:**
- [ ] prove-out mode runs 1 epoch without error
- [ ] Batch count matches ~2265 (20% of 11,325)
- [ ] LR at end of epoch 1 shows meaningful decay from peak

**All actions:**
- [x] Chained to follow-up: sweep results -> pick winner -> relaunch full with winning recipe
- [x] Success criteria defined (below)
- [x] Failure criteria defined (below)

### Parallel tracks:
| Resource | During Code Changes | During Sweep | After Sweep |
|----------|-------------------|-------------|-------------|
| GPU | Idle (code only) | Prove-out experiments P1-P4 | Full training with winning recipe |
| CPU | Idle | Data loading (20%) | Data loading (full) |
| Dev | Implement prove-out + cosine fix | Launch P1, review, launch P2, etc. | Monitor epoch 1-2 of full run |

### Success criteria:
- **Stop:** PID 1245000 killed, best_model.pt and best_model_epoch1.pt still intact
- **Implementation:** prove-out runs 1 epoch in ~15 min, batch count ~2265, cosine decays meaningfully
- **Sweep:** At least one LR shows DECLINING or FLAT train loss across all 5 prove-out epochs. Val loss does not spike >10%. Precision/recall stay nonzero through epoch 5.
- **Key signal:** Train loss trajectory. Must decline or stay flat. Any rise in epoch 2+ = recipe still broken.

### Failure criteria:
- **All 4 LRs still show mode collapse in prove-out:** Multi-task loss conflict is the primary cause, not LR. Escalate to loss weight normalization experiment (normalize weights to sum to 1.0).
- **Prove-out behavior does not transfer to full data:** If winning recipe collapses on full data, the 20% subset has different dynamics. Fall back to iterating on full data with reduced patience (patience=3).
- **Implementation takes >30 min:** Simplify — just manually limit chunks by editing data_loader.py temporarily.

---

## Recommended Prompt

```
STOP the current training run immediately. It is wasting GPU time — mode collapsed at epoch 2 and train loss has been rising since (now 0.5924 at epoch 3 batch 4600). Best model from epoch 1 is already saved.

Kill PID 1245000:
```bash
kill 1245000
```

Verify best model is preserved:
```bash
ls -la /home/ubuntu/workspace/RLQuest/firstrate_learning_v5/models/run_20260327_224100_full/best_model*.pt
```

Then implement these changes to enable rapid LR experimentation:

## 1. Fix the cosine schedule (ROOT CAUSE of mode collapse)

The cosine schedule computes total_steps = max_epochs * batches_per_epoch = 50 * 11325 = 566,250. This means at epoch 2 the LR is still 99.6% of peak — effectively NO decay. Combined with loss weights summing to 3.0 (effective LR ~9e-4), this causes the model to overshoot the minimum found in epoch 1.

The fix: total_steps must be computed AFTER we know actual batches_per_epoch (from the data loader). This is already the case in the code (line 361), but prove-out mode will have fewer batches per epoch, so the schedule will naturally scale. Just ensure total_steps uses the real batches_per_epoch from the actual data loaded.

## 2. Add prove-out mode to train.py

Add these arguments to the argparser:
- `--prove-out` in the mutually exclusive group (with --unit-test, --smoke-test, --full)
- `--lr` (float, optional) — override learning rate for any mode
- `--tag` (str, optional) — suffix for run directory name

When `--prove-out` is set:
- `max_epochs = 5`
- `warmup_steps = 200`
- Pass `max_chunks=28` to the train data loader (20% of 138 train chunks)
- Pass `max_chunks=13` to the val data loader (20% of 63 val chunks)
- Disable early stopping (set patience = max_epochs + 1)
- Run dir name: `run_TIMESTAMP_proveout_{tag}`
- When `--lr` is provided, override the config learning rate

## 3. Add max_chunks support to data_loader.py

In `V5InterleavedDataset.__init__` and `V5SequentialDataset.__init__`:
- Add `max_chunks: int = None` parameter
- After building the chunk file list, if max_chunks is set, truncate to `chunks[:max_chunks]`
- This limits data while keeping the same chunked loading logic

## 4. Run the LR sweep

After implementing, run these experiments sequentially. Each takes ~90 min (5 epochs * ~15 min/epoch + validation):

```bash
# P1: lr=1e-4 with cosine (3x reduction from current)
cd /home/ubuntu/workspace/RLQuest && python -m firstrate_learning_v5.train --prove-out --lr 1e-4 --tag lr1e-4_cosine

# P2: lr=5e-5 with cosine (6x reduction)
cd /home/ubuntu/workspace/RLQuest && python -m firstrate_learning_v5.train --prove-out --lr 5e-5 --tag lr5e-5_cosine

# P3: lr=3e-4 with cosine (same lr, but now cosine actually decays over 5 epochs instead of 50)
cd /home/ubuntu/workspace/RLQuest && python -m firstrate_learning_v5.train --prove-out --lr 3e-4 --tag lr3e-4_scaled_cosine

# P4: lr=1e-4 with constant schedule (no cosine decay at all — tests if cosine is needed)
cd /home/ubuntu/workspace/RLQuest && python -m firstrate_learning_v5.train --prove-out --lr 1e-4 --tag lr1e-4_constant --constant-lr
```

For P4, also add a `--constant-lr` flag that makes the LR schedule return 1.0 after warmup (no cosine decay).

**Start with P1 (lr=1e-4 cosine).** If P1 shows declining train loss across all 5 epochs, you can skip P2 and go straight to full training with that recipe.

## What to look for in each experiment

The KEY diagnostic is the train loss trajectory across epochs:
- GOOD: train loss declines or stays flat across all 5 epochs
- BAD: train loss rises in epoch 2+ (same pattern as current run)

Also check:
- Val loss: should not spike >10% between epochs
- Precision/recall: should stay nonzero
- P@5: should not collapse

After completing the sweep (or after P1 if it clearly works), report:
1. Train loss per epoch for each experiment
2. Val loss per epoch for each experiment
3. Which recipe showed the most stable training
4. Recommendation for full training launch

## New training progression going forward

unit-test -> prove-out (20% data, 5 epochs) -> smoke-test (if needed) -> full training

Never launch full training without first validating the recipe in prove-out mode. This saves 10-20 hours per failed recipe.
```

---

## Persistent Notes
- **Actions taken history:**
  - Cycle 1 (~20:35 UTC): Wrote "launch training" prompt. NOT DELIVERED to dev.
  - Cycle 2 (~22:00 UTC): Wrote improved "launch training" prompt. NOT DELIVERED to dev.
  - Cycle 3 (~22:30 UTC): Identified delivery bottleneck. Training WAS launched independently.
  - Cycle 4 (~22:35 UTC): Training crashed at batch 2350 (OOM). Analyzed root cause. Planned 3 fixes.
  - Cycle 5 (~22:45 UTC): Planned OOM fix + relaunch. Fixes were applied, training relaunched successfully.
  - Cycle 6 (~00:25 UTC): THIS CYCLE. Training completed 2 epochs + running epoch 3. Epoch 1 good (CR=0.0138), epoch 2+ mode collapse. Root cause: cosine over 566K steps = flat LR + loss weights 3.0 = effective LR 9e-4. Planning stop + prove-out + sweep.
- **Pending items:**
  - Fix summary token dims 14-19 (after training recipe solved)
  - Fix price token dim 15 (after training recipe solved)
  - Optimize IntraDayEncoder day-loop (after training recipe solved)
  - Fix config.d_ff unused bug (maintenance)
  - Clean up zombie processes (cosmetic, now 7 total)
  - Consider normalizing loss weights to sum to 1.0 (if LR sweep alone doesn't fix collapse)
- **Validation patterns:**
  - torch.compile mode='reduce-overhead' causes OOM with variable shapes — RESOLVED, never use again
  - Per-epoch-only checkpointing inadequate for 6-8 hour epochs — RESOLVED, intra-epoch saves added
  - Cosine schedule over max_epochs*full_batches is functionally flat for early epochs — ROOT CAUSE, being fixed
  - Multi-task loss weight sum amplifies effective LR — must account for in LR selection
  - V4 had same recipe failure (lr=3e-4, cosine over max_epochs) — cross-version pattern matching is critical
- **Effective prompts:**
  - Detailed fix prompts with exact line numbers and before/after code work well
  - Verification checklists ensure dev confirms fixes
  - Including "reason" for each change helps dev prioritize correctly
  - Sequencing instructions (do X THEN Y) prevent dev from skipping prerequisites
- **Watch items for next cycle:**
  - Has current training been stopped? Check PID 1245000 is dead.
  - Has prove-out mode been implemented? Check for --prove-out in train.py argparse.
  - Has --lr override been added? Check argparse.
  - Has max_chunks been added to data_loader.py? Check V5InterleavedDataset init.
  - Is cosine schedule scaling correctly for prove-out? total_steps should = 5 * ~2265 = ~11,325 for prove-out.
  - Has P1 (lr=1e-4 cosine) been launched? Check for proveout run dirs.
  - If P1 completed: does train loss decline across 5 epochs? This is the key signal.
  - Best model from epoch 1 PRESERVED (best_model.pt, best_model_epoch1.pt) — safety net.
  - New progression: unit-test -> prove-out -> smoke-test -> full. Enforce this going forward.
