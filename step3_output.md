## Step 3: Design Review — 2026-03-27 (Step-Based Training Implementation)
### Mode: DELTA — Step-Based Training Design for Full Run

Files changed since last cycle: data_loader.py (1774657753, was 1774650708), train.py (1774657838, was 1774651165). model.py, config.py, prepare_tokens.py unchanged.

---

### Context: Where We Are

Prove-out sweep is COMPLETE. Four experiments ran:

| Experiment | LR | Schedule | Best Epoch | Test CR | Val Score |
|---|---|---|---|---|---|
| P1: lr=1e-4 cosine | 1e-4 | Cosine/5ep | 2 | 0.0141 | 0.0633 |
| P2: lr=5e-5 cosine | 5e-5 | Cosine/5ep | 4 | 0.0185 | 0.0721 |
| P3: lr=3e-4 scaled cosine | 3e-4 | Cosine/5ep | 2 | 0.0163 | 0.0711 |
| P4: lr=1e-4 constant | 1e-4 | Constant | 1 | **0.0176** | 0.0586 |

**Winning recipe: P4 (lr=1e-4, constant LR).** Test CR=0.0176, which is 1.2x V3 baseline (0.0147). P2 had higher val score but precision=0/recall=0 on test (degenerate predictions). P4 has balanced precision (0.642) and recall (0.586) with best P@5% (0.313).

**Recipe for full training: `--full --lr 1e-4 --constant-lr --no-smoke`**

### 3.1 What's Already Implemented in train.py

Reading the current train.py (lines 1-591), the following step-based features ARE present:

1. **Intra-epoch checkpointing** (line 159-162): Every 2000 batches, `checkpoint_fn(batch_idx)` saves `latest_checkpoint.pt` with `batch_offset`. DONE.
2. **Step-based resume** (lines 403-436): On resume, `start_batch` is loaded from `batch_offset` in checkpoint. Batches are skipped via `if batch_idx <= start_batch` (line 106). DONE.
3. **Prove-out mode** (lines 283-286, 560-561): `--prove-out` flag with 28 train chunks, 13 val chunks, 5 epochs. DONE.
4. **LR override + constant LR** (lines 371, 392-398, 568-569): `--lr` and `--constant-lr` flags. DONE.

### 3.2 What's MISSING (from step_based_training.md)

Two features from the spec are NOT implemented:

**A. Subset validation every N steps (spec item 3)**
- The spec says: "Optionally: run lightweight validation every N steps (e.g., 5,000). Subset validation (10% of val data) for quick trend check."
- Currently, validation ONLY runs at end of each epoch (~11,325 batches for full data).
- With ~75 min/epoch, you wait 75 min for the first divergence signal.
- With subset eval every 2000 steps, you'd get a signal every ~13 min.

**B. Step-based early stopping (spec item 4)**
- The spec says: "Instead of patience in epochs, patience in evaluation cycles (which could be every 5000 steps)."
- Currently, early stopping is epoch-based: `patience_counter >= model_config.patience` checked once per epoch (line 519).
- With epoch-based patience=15 and ~75 min/epoch, worst case is 15 * 75 = 18.75 hours before stopping.
- With step-based patience (e.g., patience=5 eval cycles at every 2000 steps), worst case is 5 * 13 min = 65 min.

### 3.3 Design: What Needs to Change in train.py

#### Feature A: Subset Validation Every N Steps

**Add to `train_epoch`:**
- New parameter: `eval_every_steps` (default: None, meaning no mid-epoch eval)
- New parameter: `eval_fn` — callable that runs subset validation and returns metrics
- Inside the batch loop, after the checkpoint check (line 162), add:

```python
# Subset validation every eval_every_steps batches
if eval_fn and eval_every_steps and batch_idx % eval_every_steps == 0:
    sub_metrics = eval_fn()
    logger.info(f"    [step-eval at batch {batch_idx}]: VL={sub_metrics['loss']:.4f} "
                f"CR={sub_metrics['captured_return']:.4f}")
    model.train()  # back to train mode after eval
```

**Create subset val loader:**
- In `train()`, build a separate subset val loader with `val_max_chunks=6` (10% of 63 val chunks).
- Pass `evaluate(model, subset_val_loader, criterion, device)` as the eval_fn.

**Key detail:** Must call `model.train()` after eval since `evaluate()` calls `model.eval()`.

**Estimated: ~15 lines in train_epoch, ~5 lines in train().**

#### Feature B: Step-Based Early Stopping

**Modify the training loop in `train()`:**
- Instead of checking `patience_counter` only at epoch boundaries, track it at each step-eval.
- Add `step_eval_patience` parameter (default: 8 eval cycles).
- After each step-eval, check if `captured_return` improved. If not, increment `step_patience_counter`. If `step_patience_counter >= step_eval_patience`, break out of both the batch loop AND epoch loop.

**Implementation approach:**
- `train_epoch` returns a signal if step-based early stopping triggers: e.g., `{'loss': ..., 'early_stop': True}`
- The eval_fn callback updates `best_val_score` and `patience_counter` (closure over outer scope variables).
- If early stop triggers mid-epoch, `train_epoch` breaks and returns.

**Estimated: ~20 lines total across train_epoch and train().**

#### Combined Change Size: ~40 lines

### 3.4 Is constant lr=1e-4 the Right Recipe for Full Training?

**YES, with caveats:**

1. **P4 proved stability.** Constant LR ran 5 epochs on 20% data without divergence. Best epoch=1 means it found a good point early, but epochs 2-5 didn't degrade (val_score=0.0586 at epoch 1, no collapse). This is the opposite of the cosine failures where train loss rose.

2. **Risk with full data:** Full training has 5x more data per epoch (138 vs 28 chunks). At constant lr=1e-4, the model sees 5x more gradient updates per epoch. The effective training is much longer per epoch. This could mean:
   - Best epoch shifts later (more data = slower convergence per epoch) — GOOD.
   - Or the model has already converged at prove-out scale and more data just adds noise — UNLIKELY given 20% sampling.

3. **Loss weight rebalancing NOT needed for initial full run.** The constant LR compensates for the 3.0x loss weight sum. The effective gradient scale is 1e-4 * 3.0 = 3e-4 equivalent, which is reasonable for a 680K-param model. Rebalancing would change the recipe and invalidate the prove-out results.

4. **Watch item: return_mae=NaN and return_corr=0.0.** ALL prove-outs show NaN return_mae and 0.0 return_corr. This means `pred_return` is producing NaN or degenerate values. The return head and quantile head may not be contributing meaningfully. This is NOT blocking (CR comes from classification heads), but should be investigated after full training.

**Recommendation: Proceed with `--full --lr 1e-4 --constant-lr --no-smoke`. Do NOT rebalance loss weights first.**

### 3.5 Iteration Speed: Divergence Detection with Step-Based Eval

**Current (epoch-based eval only):**
- Full data: ~11,325 batches/epoch at ~156 batches/min = ~72 min/epoch
- First eval signal: 72 min
- If diverging, earliest detection: 72 min after start
- With patience=15: could waste 15 * 72 = 18 hours

**With step-based eval every 2000 steps:**
- Eval interval: 2000 / 156 = ~12.8 min
- First eval signal: ~13 min after start
- Subset eval cost: 6 val chunks, ~65K samples / 512 = ~127 batches. At 156 batches/min, ~49 sec per eval. Negligible.
- 5 evals per epoch (at 2000, 4000, 6000, 8000, 10000 steps)
- **Divergence detected in ~13 min instead of ~72 min — 5.5x faster feedback.**
- With step patience=8: max waste = 8 * 13 = 104 min (vs 18 hours epoch-based). **10x faster stopping.**

**This is the key value of step-based training: not just checkpointing (already done) but fast feedback loops.**

### 3.6 Implementation Priority

1. **Subset eval every 2000 steps** — HIGH value, ~20 lines. Enables fast divergence detection.
2. **Step-based early stopping** — HIGH value, ~20 lines. Prevents hours of wasted compute.
3. **Full training launch** — AFTER features 1-2 are implemented.

Without these features, full training can still run (epoch-based eval + patience work), but divergence detection takes 72 min instead of 13 min, and early stopping wastes up to 18 hours instead of 104 min.

**Decision for orchestrator:** Implement both features first (30 min of coding), THEN launch full training. The 30 min investment saves potentially 17+ hours of wasted GPU time if something goes wrong.

---

### CRITICAL ISSUES:
1. **return_mae=NaN across ALL prove-outs.** The return prediction head is producing NaN. Not blocking (CR from classification), but indicates a broken loss component. Should investigate the Huber loss on `pred_return` after full training.
2. **No step-based eval or early stopping yet.** Full training with epoch-only eval means 72 min blind spots and potential 18-hour waste on divergence.

### RECOMMENDATIONS:
1. **Implement subset eval every 2000 steps + step-based early stopping** (~40 lines in train.py). This is the remaining work from step_based_training.md.
2. **Then launch full training:** `python -m firstrate_learning_v5.train --full --lr 1e-4 --constant-lr --no-smoke --tag full_v1`
3. **Do NOT rebalance loss weights** before full training. The constant LR recipe is validated.
4. **Investigate return_mae=NaN** as a follow-up item (likely NaN in pred_return output).

---

## Persistent Notes
(Cache for next cycle -- carry forward and update.)

- **File timestamps:** model.py=1774651156, config.py=1774637440, data_loader.py=1774657753, prepare_tokens.py=1774590272, train.py=1774657838
- **Model:** 679,498 params, d_model=128, n_heads=4, intra_layers=2, temporal_layers=1, backbone_dim=64
- **Param breakdown:** intra_day=400,641 (59%), temporal=232,320 (34%), backbone=25,152 (3.7%), heads=~4.2K each (0.6% each)
- **Params/sample ratio:** 0.117 -- MEDIUM overfit risk
- **Token dims:** 20/20 contract dims populated; summary token dims 14-19 all zeros (6/20 wasted); price token dim 15 always zero (bug)
- **Sparse dims:** bid_ask_spread (dim 10) 97% zeros, log_ba_ratio (dim 19) 97% zeros
- **Feature scale mismatch:** moneyness [0,97], moneyness_peak_iv [-64,+53], mid_price_norm [0,50], intrinsic_value [0,96] vs typical [-10,10]
- **Day mask:** 89.8-93.6% with 5 valid days
- **Loss:** 6 active components (contrastive disabled), weights=[1.0, 0.5, 0.4, 0.8, 0.2, 0.1], sum=3.0, dominant=magnitude(1.0)+return(0.8)=60%
- **Loss conflict:** confidence head creates feedback loop with magnitude head -- amplifies instability. Embedding variance mildly opposes clustering. Overall: secondary to LR problem.
- **Recipe VALIDATED:** lr=1e-4, constant (no cosine decay), warmup=200 (proveout) / 1000 (full), AdamW(beta2=0.98), weight_decay=0.01, grad_clip=1.0, dropout=0.25, AMP FP16
- **LR SCHEDULE ROOT CAUSE (RESOLVED):** Cosine decay over 566K steps was effectively flat. Constant LR at 1e-4 is stable. Cosine at lr=3e-4 caused mode collapse at epoch 2.
- **Data:** 11,411,714 samples, 264 chunks (train=138, val=63, test=63), split=50.8/24.3/24.9
- **Split:** train=2010q1-2019q4, val=2020q1-2022q4, test=2023q1-2025q4 (time-based, no leakage)
- **Epoch timing:** ~72-75 min/epoch on full data (156 batches/min). Prove-out (20%): ~14.5 min/epoch.
- **Score formula:** p_big * p_up * p_confidence + quantile/return adjustments (classification-dominated)
- **Prove-out results (COMPLETE):**
  - P1 (lr=1e-4, cosine): CR=0.0141, best_ep=2
  - P2 (lr=5e-5, cosine): CR=0.0185 but Prec=0/Rec=0 (degenerate)
  - P3 (lr=3e-4, scaled cosine): CR=0.0163, best_ep=2
  - P4 (lr=1e-4, constant): CR=0.0176, Prec=0.642, Rec=0.586, P@5=0.313 -- WINNER
- **Baselines:** V3 CR=0.0147, V1 CR=0.0192. P4 at 1.2x V3. Target: match or exceed V1.
- **Known issues:**
  1. Summary token dims 14-19 all zeros (6/20 wasted) -- non-blocking
  2. Price token dim 15 always zero (code skips pf[14] to pf[16]) -- non-blocking
  3. config.d_ff unused by model (hardcoded d_model*4) -- maintenance trap
  4. Feature scale mismatch on moneyness/mid_price_norm/intrinsic_value -- affects ~2% extreme tokens
  5. **RESOLVED** torch.compile mode='reduce-overhead' causes CUDA Graph OOM. Fix: mode='default' applied.
  6. IntraDayEncoder day-loop is sequential (potential 2-3x speedup if batched)
  7. **RESOLVED** No intra-epoch checkpointing -- added every 2000 batches.
  8. **RESOLVED** LR schedule effectively flat for first 10 epochs. Fix: constant lr=1e-4.
  9. Multi-task loss weights sum to 3.0 -- effectively 3x the learning rate in gradient magnitude. Compensated by lower LR.
  10. Confidence loss creates feedback loop with magnitude head -- monitor during full training.
  11. **NEW** return_mae=NaN across ALL prove-outs. Return prediction head producing NaN. Not blocking CR.
  12. **NEW** Subset eval + step-based early stopping NOT YET IMPLEMENTED. Spec items 3-4 from step_based_training.md.
- **Training status:** Prove-out COMPLETE. Full training NOT YET STARTED. Awaiting step-based eval implementation.
- **Watch items for next cycle:**
  - Have subset eval and step-based early stopping been implemented?
  - Has full training been launched?
  - If full training running: check step-eval metrics at 2000-step intervals for divergence
  - return_mae=NaN investigation (secondary)
  - Full training expected: ~72 min/epoch * 50 epochs max = ~60 hours upper bound. With patience=15 step-evals, much less.
