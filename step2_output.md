## Step 2: Gap & Constraint — 2026-03-28 00:11 UTC

### Constraint SHIFTED: Infrastructure (OOM) -> Training Recipe (mode collapse / LR overshoot)

OOM fix confirmed working (GPU memory stable at 6.8 GiB). Training completed epoch 1 successfully with strong results (CR=0.0138, near V3 baseline of 0.0147). However, epoch 2 showed catastrophic mode collapse: train loss ROSE (0.5508->0.5894), val loss spiked 15.7%, precision/recall collapsed to 0.000. Epoch 3 continuing same degradation. This is NOT overfitting — it is LR overshooting or multi-task loss conflict destroying the minimum found in epoch 1.

### Best performance: V5-Small epoch 1 CR=0.0138, Prec=0.591, Rec=0.881, P@5=0.213
### Target: CR >= 0.0147 with return_corr > 0.0132 (V3 baseline)
### Gap: 0.0138 / 0.0147 = 0.94x — only 6% improvement needed. Gap is now MEASURABLE and SMALL.
### Gap trend: MAJOR IMPROVEMENT. Was unmeasurable (no completed epoch). Now within striking distance of target. Epoch 1 alone nearly matches V3. The problem is not model capability — it is training stability beyond epoch 1.
### Constraint: Training Recipe — learning rate causes mode collapse after epoch 1
### Constraint changed from last cycle: YES. Was Infrastructure (CUDA OOM). Now Training Recipe (LR/scheduler). Infrastructure constraint fully resolved.
### Hypothesis: "Implementing a prove-out mode (20% data, 5 epochs, ~15 min total) will accelerate LR/scheduler iteration by 10x, enabling rapid discovery of a stable training recipe that sustains epoch 1's CR=0.0138 performance across multiple epochs, because the mode collapse signature (rising train loss) would be visible within 2 epochs on reduced data."
### Success criteria: Prove-out run completes in <20 min, reproduces the mode collapse at current lr=3e-4, and identifies a lr/schedule that shows declining or stable train loss across 5 epochs
### Failure criteria: Prove-out runs do not reproduce full-data behavior (reduced data has fundamentally different dynamics), wasting time on non-transferable experiments

### Detailed Analysis:

**Why prove-out mode makes sense NOW:**
1. The mode collapse is visible by epoch 2 batch-level metrics (train loss rising within the epoch). With 20% data, each epoch takes ~15 min instead of ~75 min. Five epochs in ~75 min instead of ~6.25 hours.
2. The key diagnostic (train loss trajectory) does not require the full dataset to be informative. If lr=3e-4 overshoots on full data, it will almost certainly overshoot on 20% data (possibly even faster due to fewer gradient steps per epoch).
3. Multiple experiments can be run sequentially in a few hours: lr=3e-4, lr=1e-4, lr=5e-5, lr=3e-4 with cosine decay, etc.
4. Current approach (full data, patience=15) would waste ~17 hours of GPU time on 14 more failing epochs before early stopping triggers.

**Implementation approach (for dev directive):**
- Add a `--prove-out` flag to train.py that:
  - Uses 20% of training chunks (28 of 138 chunks, ~2265 batches/epoch)
  - Runs 5 epochs max (no early stopping patience)
  - Logs batch-level loss every 50 batches (already done)
  - Skips full validation (only log train loss trajectory — the diagnostic we need)
  - OR: faster alternative — just pass `--max-chunks 28 --max-epochs 5` if the data loader supports chunk limiting
- Alternatively, the simplest implementation: create a small token subset by symlinking only 28 of 138 train chunk files

**Immediate action priority:**
1. STOP current training (epoch 3+ is wasting GPU time, best model already saved from epoch 1)
2. Implement prove-out mode OR manually limit data
3. Run prove-out sweep: test lr in {1e-4, 5e-5, 3e-4 with cosine decay, 3e-4 with warmup+decay}
4. Whichever lr/schedule shows stable or declining train loss across 5 prove-out epochs, use that for full run
5. Relaunch full training with winning recipe

**Risk assessment of prove-out approach:**
- LOW risk: epoch 1 best model is preserved regardless of what we do next
- The 20% subset may have slightly different batch statistics, but the mode collapse is so dramatic (train loss rising 7% in epoch 2) that any recipe that fixes it on 20% data will very likely fix it on full data
- Cost of being wrong: ~75 min of GPU time on a prove-out sweep that doesn't transfer. This is still much less than the ~17 hours of wasted patience epochs.

**Alternative to prove-out (also valid):**
- Just reduce lr to 1e-4 and relaunch full training directly. This is simpler but slower to iterate if 1e-4 also fails.
- Add cosine annealing scheduler and relaunch. Good odds of working but no fast feedback loop.

---

## Persistent Notes
- Previous constraint: Infrastructure (CUDA OOM from torch.compile) — RESOLVED by switching to mode='default'
- Current constraint: Training Recipe (lr=3e-4 causes mode collapse after epoch 1)
- Previous gap: Unmeasurable (no completed epoch)
- Current gap: 0.94x of target (CR=0.0138 vs target 0.0147) — NEARLY THERE
- Hypothesis history:
  - Cycle 1-3: "Launch full training, expect CR >= 0.0147" (blocked by execution)
  - Cycle 4: "Fix CUDA Graph OOM and relaunch" (CONFIRMED — OOM fixed, training ran)
  - Cycle 5 (current): "Implement prove-out mode for rapid LR/scheduler iteration to fix mode collapse"
- Key evidence accumulated:
  - V5 epoch 1: CR=0.0138, Prec=0.591, Rec=0.881, P@5=0.213, TrL=0.5508, VL=0.6111 — STRONG
  - V5 epoch 2: MODE COLLAPSE. TrL=0.5894 (ROSE), VL=0.7073 (+15.7%), Prec/Rec=0.000
  - V5 epoch 3: Continuing degradation, TrL rising within epoch (0.585->0.588 at batch 1000)
  - Mode collapse is NOT overfitting (train loss rises too) — it is LR overshoot or loss conflict
  - P@5 improved 0.213->0.251 despite collapse — ranking signal preserved, calibration destroyed
  - LR barely decayed epoch 1->2: 3.00e-4 -> 2.99e-4 (essentially no schedule effect)
  - Best model (epoch 1) safely preserved in best_model.pt and best_model_epoch1.pt
  - V5 training speed: ~75 min/epoch on full data. Prove-out (20%) would be ~15 min/epoch.
  - V5 data: 138 train chunks, 11,325 batches/epoch at batch_size=512
  - GPU: Quadro RTX 6000, 24GB, memory stable at 6.8 GiB (OOM fix working)
  - V3 baseline: CR=0.0147, rank_corr=0.0152, return_corr=0.0132 (epoch 12)
  - V1 best ever: CR=0.0192 (epoch 33)
  - V4 was a regression: CR=0.0031 (overparameterized)
  - Known data issues: 6/20 summary token dims zeros, price dim 15 always zero
  - BCE fix applied and confirmed working
  - 7 zombie processes (cosmetic)
- Watch items for next cycle:
  - Has dev stopped the current (wasteful) training run?
  - Has prove-out mode been implemented or data subset created?
  - If prove-out running: what is train loss trajectory across 5 epochs for each lr tested?
  - Key signal: train loss should DECLINE or stay flat across prove-out epochs. Any rise = recipe still broken.
  - Once winning recipe identified: full training relaunched with new lr/schedule?
  - Epoch 1 best model PRESERVED — this is our safety net regardless of experiments
  - If dev takes alternative approach (just change lr and relaunch full): monitor epoch 2 for same collapse pattern
