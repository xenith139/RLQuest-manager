## Step 3: Design Review — 2026-03-28 00:17 UTC
### Mode: DELTA — Mode Collapse Root Cause + Prove-Out Design

Files changed since last cycle: model.py (1774651156, was 1774647634), data_loader.py (1774650708, was 1774637686), train.py (1774651165, was prior). config.py and prepare_tokens.py unchanged.

---

### 3.1 ROOT CAUSE: Why Mode Collapse Happened

**The LR schedule is effectively FLAT for the first 10+ epochs.** This is the primary cause.

Mathematical proof from `train.py` lines 363-367:

```
warmup_steps = 1000
batches_per_epoch = 5,798,357 // 512 = 11,325
total_steps = 50 * 11,325 = 566,250

lr_schedule(step):
  if step < 1000: return step/1000          # warmup
  progress = (step - 1000) / 565,250        # cosine phase
  return max(0.1, 0.5 * (1 + cos(pi * progress)))
```

**Effective LR at key points:**
| Point | Step | Progress | Cosine Mult | Effective LR |
|-------|------|----------|-------------|-------------|
| End warmup | 1,000 | 0.000 | 1.000 | 3.00e-4 |
| End epoch 1 | 11,325 | 0.0183 | 0.999 | 2.997e-4 |
| End epoch 2 | 22,650 | 0.0383 | 0.996 | 2.989e-4 |
| End epoch 5 | 56,625 | 0.0984 | 0.985 | 2.955e-4 |
| End epoch 10 | 113,250 | 0.1985 | 0.942 | 2.826e-4 |
| Epoch where LR halves | ~180 | 0.500 | 0.500 | 1.50e-4 |

**The LR at epoch 2 is 99.6% of its peak value.** The cosine schedule is designed for 50 epochs (566K steps), so it barely decays at all in the first 5 epochs. The model finds a good minimum in epoch 1, then epoch 2 runs at essentially the same LR and overshoots that minimum. The rising train loss (0.5508 -> 0.5894) is the signature of LR-induced oscillation around or departure from the minimum.

**V4 confirmation:** V4 used identical lr=3e-4, identical cosine schedule over max_epochs (with 2000 warmup steps), and also peaked at epoch 1 then degraded. Same root cause.

### 3.2 Multi-Task Loss Conflict Analysis

**6 active loss components** (contrastive disabled, w_contrast=0.0):

| Component | Weight | Type | Gradient Direction |
|-----------|--------|------|-------------------|
| magnitude (focal BCE) | 1.0 | Classification: "is this a big move?" | Push p_big toward y_big |
| direction (BCE) | 0.5 | Classification: "up or down?" | Push p_up toward sign(return) |
| quantile (asymmetric) | 0.4 | Regression: predict return quantiles | Push quantiles toward y_return |
| return (Huber, 3x big moves) | 0.8 | Regression: predict exact return | Push pred_return toward y_return |
| confidence (BCE) | 0.2 | Meta: "was magnitude prediction correct?" | Push confidence toward correctness |
| embedding variance | 0.1 | Regularizer: prevent collapse | Push embeddings apart |

**Conflict analysis:**
- magnitude + direction are correlated (big moves have clearer direction) — LOW conflict
- quantile + return both target y_return — LOW conflict, they cooperate
- confidence depends on magnitude's correctness — creates a FEEDBACK LOOP. If magnitude head gets worse in epoch 2, confidence targets change, creating additional instability. Weight is only 0.2 so impact is moderate.
- embedding variance is a regularizer that opposes the classification heads' desire to cluster embeddings — MILD conflict, but weight is only 0.1.

**Verdict: Loss conflict is a secondary contributor, not the primary cause.** The confidence feedback loop could amplify instability once the LR causes the magnitude head to oscillate, but the core problem is the flat LR schedule. The losses sum to weight 3.0, which effectively makes the gradient magnitude 3x what a single-loss model would see — this compounds the too-high LR problem.

**The effective LR is really 3e-4 * ~3.0 (sum of loss weights) in gradient magnitude terms.** A single-task model at lr=3e-4 would have 3x smaller gradient updates than this 6-task model.

### 3.3 Should LR be Reduced from 3e-4?

**YES, strongly.** Three converging lines of evidence:

1. **V5 mode collapse at epoch 2** with lr=3e-4, cosine barely decaying
2. **V4 same pattern** with lr=3e-4 — peaked epoch 1, degraded after
3. **Effective gradient scale is ~3x** due to multi-task loss weights summing to 3.0

**Recommended LR range to test:** 5e-5 to 1e-4.

Rationale:
- Multi-task effective multiplier of ~3x means lr=3e-4 is effectively lr=9e-4 in gradient magnitude
- Standard transformer LR for 680K params is 1e-4 to 3e-4 for SINGLE-task
- Dividing by ~3x for multi-task: target range is 3e-5 to 1e-4
- Conservative starting point: 1e-4 (3x reduction from current)
- Aggressive option: 5e-5 (6x reduction)

**Alternative: keep lr=3e-4 but add aggressive decay.** ReduceLROnPlateau with factor=0.5 and patience=1 would halve the LR after epoch 1's performance isn't beaten — this is closer to what the model needs. But this is harder to tune in prove-out mode.

### 3.4 Prove-Out Mode Design

**What `--prove-out` should do:**

```
python -m firstrate_learning_v5.train --prove-out [--lr 1e-4] [--tag sweep1]
```

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Training data | 20% of chunks (28 of 138) | ~2,265 batches/epoch |
| Validation data | 20% of val chunks (13 of 63) | Proportional reduction |
| Max epochs | 5 | Enough to see collapse (happens at epoch 2) |
| Early stopping | Disabled | Run all 5 epochs regardless |
| Batch logging | Every 50 batches (already done) | Key diagnostic |
| Epoch-level validation | YES (full validation on reduced set) | Need val loss to detect overshoot |
| LR schedule | Scale warmup to 200 steps (proportional to 20% data) | Keep same warmup fraction |
| Total steps for cosine | 5 * 2,265 = 11,325 | Cosine over prove-out length, not 50 epochs |
| Run dir | `run_TIMESTAMP_proveout` | Separate from full runs |
| Tag | User-provided string for identification | e.g., `lr1e-4_cosine` |

**Critical design choice: Scale the cosine schedule to prove-out length.** If we keep total_steps = 50 * 11,325 = 566,250 but only run 5 * 2,265 = 11,325 steps, the cosine is again flat (same problem). The prove-out schedule must be: total_steps = max_epochs * batches_per_epoch with PROVE-OUT values, so the cosine actually decays meaningfully across 5 epochs.

**Implementation approach (minimal changes to train.py):**

1. Add `--prove-out` flag to argparse (mutually exclusive with other modes)
2. When prove-out:
   - `max_epochs = 5`
   - Pass `max_chunks=28` to a modified `build_v5_loaders` (or `V5InterleavedDataset`)
   - `warmup_steps = 200`
   - Compute `total_steps` from actual prove-out batches, not full data
   - Disable early stopping (patience = max_epochs + 1)
3. Add `--lr` override argument (float, optional)
4. Add `--tag` argument for run dir naming

**Changes to data_loader.py:**
- `V5InterleavedDataset.__init__` needs a `max_chunks` parameter
- In `__iter__`, truncate `order` to `order[:max_chunks]` after shuffle
- Similarly for val: `V5SequentialDataset.__init__` needs `max_chunks`

**Estimated change: ~30 lines across train.py + data_loader.py.**

### 3.5 Iteration Speed Analysis

**Current full training:**
- ~140ms/batch GPU time (from latest log)
- 11,325 batches/epoch
- ~26 min/epoch (much faster than previous 6-8h estimate — mode='default' is working well)
- Wait — recalculating: 11,325 * 0.2s (data+gpu) = 2,265s = ~38 min/epoch. Log shows batch 2500 at 00:16 from epoch start ~00:00, so ~16 min for 2500 batches. Full epoch: 16 * (11,325/2500) = ~72 min/epoch.

**Actually: epoch 3 started around 00:00 UTC (after epoch 2 ended at ~23:50). At 00:16, batch 2500. Rate: 2500 batches in 16 min = 156 batches/min. Full epoch: 11,325 / 156 = 72.6 min/epoch.** This matches the ~75 min estimate.

**Prove-out mode (20% data):**
- 2,265 batches/epoch
- At 156 batches/min: 2,265 / 156 = **14.5 min/epoch**
- 5 epochs: **~73 min per experiment** (including val)
- Val on 13 chunks: ~5 min * 5 epochs = ~25 min. Total: ~98 min per experiment.
- Reduce val to skip or subsample: ~73 min per experiment.

**Experiments per day:**
- 1 experiment = ~90 min (with reduced validation)
- 24 hours / 90 min = **16 experiments/day** (if fully automated)
- Realistic with human review: **6-8 experiments/day** (review results, decide next lr, launch)
- Minimum useful sweep: 4 experiments (lr = {5e-5, 1e-4, 2e-4, 3e-4 with cosine scaled}) = **6 hours**

**Compared to current approach:**
- Full training with patience=15: 75 min * 16 epochs (1 good + 15 patience) = **20 hours wasted**
- Prove-out sweep of 4 LRs: **6 hours** to find the right recipe, then 1 full run

**Speed-up: 3.3x faster to find the right recipe with prove-out vs trial-and-error on full data.**

### 3.6 Recommended Prove-Out Sweep

Run in this order (each ~90 min):

| Experiment | LR | Schedule | Warmup | Hypothesis |
|------------|-----|---------|--------|-----------|
| P1 | 1e-4 | Cosine over 5 epochs | 200 steps | 3x reduction fixes overshoot |
| P2 | 5e-5 | Cosine over 5 epochs | 200 steps | 6x reduction for extra stability |
| P3 | 3e-4 | Cosine over 5 epochs | 200 steps | Same LR but fast decay (tests if decay alone fixes it) |
| P4 | 1e-4 | Constant (no decay) | 200 steps | Tests if cosine is even needed |

**Success signal:** Train loss DECLINES or stays flat across all 5 epochs. Val loss does not spike >10%. Precision/recall stay nonzero.

**Failure signal:** Train loss rises in epoch 2+ (same as current).

**After sweep:** Take the winning recipe and launch full training with cosine schedule scaled to full 50 epochs. Monitor epoch 2 carefully.

---

### CRITICAL ISSUES:
1. **LR schedule is effectively flat for first 10 epochs** — cosine decay over 566K steps barely moves in the first 11K steps. This is the root cause of mode collapse. The model overshoots the epoch-1 minimum at full LR.
2. **Training is still running (epoch 3, batch ~2500) and wasting GPU time** — train loss continues rising (0.5923 at batch 2500, worse than epoch 2's 0.5894 final). Every minute of continued training is wasted compute.
3. **Multi-task loss weights sum to 3.0**, effectively tripling the gradient magnitude vs a single-task model. At lr=3e-4 the effective learning rate is ~9e-4 in single-task equivalent terms.

### RECOMMENDATIONS:
1. **STOP current training immediately** — it is wasting GPU time. Best model from epoch 1 is safely preserved.
2. **Implement prove-out mode** (~30 lines of code changes) with `--prove-out`, `--lr`, `--tag` flags.
3. **Run 4-experiment LR sweep** (P1-P4 above), ~6 hours total.
4. **Scale cosine schedule to actual run length** in prove-out mode — this is critical, otherwise the schedule is flat again.
5. **Consider reducing loss weight sum** — normalize weights so they sum to 1.0 instead of 3.0, then lr=3e-4 might work. But this is a secondary experiment after the LR sweep.

---

## Persistent Notes
(Cache for next cycle — carry forward and update.)

- **File timestamps:** model.py=1774651156, config.py=1774637440, data_loader.py=1774650708, prepare_tokens.py=1774590272, train.py=1774651165
- **Model:** 679,498 params, d_model=128, n_heads=4, intra_layers=2, temporal_layers=1, backbone_dim=64
- **Param breakdown:** intra_day=400,641 (59%), temporal=232,320 (34%), backbone=25,152 (3.7%), heads=~4.2K each (0.6% each)
- **Params/sample ratio:** 0.117 — MEDIUM overfit risk
- **Token dims:** 20/20 contract dims populated; summary token dims 14-19 all zeros (6/20 wasted); price token dim 15 always zero (bug)
- **Sparse dims:** bid_ask_spread (dim 10) 97% zeros, log_ba_ratio (dim 19) 97% zeros
- **Feature scale mismatch:** moneyness [0,97], moneyness_peak_iv [-64,+53], mid_price_norm [0,50], intrinsic_value [0,96] vs typical [-10,10]
- **Day mask:** 89.8-93.6% with 5 valid days
- **Loss:** 6 active components (contrastive disabled), weights=[1.0, 0.5, 0.4, 0.8, 0.2, 0.1], sum=3.0, dominant=magnitude(1.0)+return(0.8)=60%
- **Loss conflict:** confidence head creates feedback loop with magnitude head — amplifies instability. Embedding variance mildly opposes clustering. Overall: secondary to LR problem.
- **Recipe:** lr=3e-4, warmup=1000 steps, cosine decay to 0.1x over 566K steps, AdamW(beta2=0.98), weight_decay=0.01, grad_clip=1.0, dropout=0.25, AMP FP16
- **LR SCHEDULE BUG:** Cosine decay over 50 epochs means LR is 99.6% of peak at epoch 2, 98.5% at epoch 5. Effectively NO decay during the critical early epochs. This is the root cause of mode collapse.
- **Effective LR analysis:** Loss weights sum to 3.0, so effective single-task-equivalent LR is ~9e-4. Standard transformer LR for 680K params is 1e-4 to 3e-4. Model is 3-9x over the recommended range.
- **Data:** 11,411,714 samples, 264 chunks (train=138, val=63, test=63), split=50.8/24.3/24.9
- **Split:** train=2010q1-2019q4, val=2020q1-2022q4, test=2023q1-2025q4 (time-based, no leakage)
- **Epoch timing:** ~72-75 min/epoch on full data (156 batches/min). Prove-out (20%): ~14.5 min/epoch.
- **Score formula:** p_big * p_up * p_confidence + quantile/return adjustments (classification-dominated)
- **Known issues:**
  1. Summary token dims 14-19 all zeros (6/20 wasted) — non-blocking
  2. Price token dim 15 always zero (code skips pf[14] to pf[16]) — non-blocking
  3. config.d_ff unused by model (hardcoded d_model*4) — maintenance trap
  4. Feature scale mismatch on moneyness/mid_price_norm/intrinsic_value — affects ~2% extreme tokens
  5. **RESOLVED** torch.compile mode='reduce-overhead' causes CUDA Graph OOM. Fix: mode='default' applied.
  6. IntraDayEncoder day-loop is sequential (potential 2-3x speedup if batched)
  7. **RESOLVED** No intra-epoch checkpointing — added every 2000 batches.
  8. **ROOT CAUSE FOUND** LR schedule effectively flat for first 10 epochs. Cosine decay spread over 566K steps. Need to either (a) reduce LR to 5e-5 to 1e-4, or (b) scale cosine to shorter horizon, or (c) use ReduceLROnPlateau.
  9. **NEW** Multi-task loss weights sum to 3.0 — effectively 3x the learning rate in gradient magnitude. Consider normalizing weights to sum to 1.0.
  10. **NEW** Confidence loss creates feedback loop: if magnitude head degrades, confidence targets change, amplifying instability.
- **Training status:** Epoch 3 in progress (batch ~2500/11325), train loss rising (0.5923), continuing degradation. SHOULD BE STOPPED.
- **Best result:** Epoch 1: TrL=0.5508, VL=0.6111, Prec=0.591, Rec=0.881, CR=0.0138, P@5=0.213. Best model preserved.
- **Baselines:** V3 CR=0.0147 (136K params, epoch 12), V1 CR=0.0192 (epoch 33), V4 CR=0.0031 (overfit at epoch 1-2, same lr=3e-4)
- **V4 pattern match:** V4 used identical lr=3e-4 with cosine over max_epochs, also peaked epoch 1 and degraded. Same root cause.
- **Prove-out design:** --prove-out flag, 28 train chunks, 13 val chunks, 5 epochs, warmup=200, cosine scaled to prove-out length, --lr and --tag args. ~30 lines of changes.
- **Iteration speed:** Prove-out: ~90 min/experiment. 4-experiment sweep: ~6 hours. Full training with patience: ~20 hours wasted. Prove-out is 3.3x faster.
- **Watch items for next cycle:**
  - Has current training been stopped?
  - Has prove-out mode been implemented?
  - If prove-out running: check train loss trajectory across 5 epochs for each LR tested
  - Key signal: train loss should decline or stay flat. Any rise = recipe broken.
  - P1 (lr=1e-4) is highest-priority experiment — if it works, go straight to full training
  - Epoch 1 best model PRESERVED at best_model.pt and best_model_epoch1.pt
  - If prove-out approach is rejected: minimum viable fix is lr=1e-4 + relaunch full training, monitor epoch 2
