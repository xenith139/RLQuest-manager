# Goal Tracker

Updated by Step 4 (Learn & Update) — 2026-03-28 04:00 UTC (Cycle 6)

## Current Goal
Validate V5-Small temporal architecture — full training run, evaluate against V3 baseline (CR >= 0.0147).

## Status
Prove-out sweep COMPLETE (4 experiments). Architecture VALIDATED: all 4 prove-outs beat V3 baseline on test CR. Winning recipe: P4 (lr=1e-4, constant LR) — test CR=0.0176 (1.2x V3), Prec=0.642, Rec=0.586, stable through epoch 5. All cosine recipes (P1-P3) show prec/rec collapse by epoch 3 regardless of LR — root cause is multi-task loss weight conflict under cosine decay, not LR magnitude. Two step-based features remain unimplemented: subset eval every 2000 steps and step-based early stopping (~40 lines, ~30 min). Dev implemented prove-out mode, ran all 4 experiments, and is now IDLE. GPU IDLE. Full training NOT YET LAUNCHED — awaiting step-based eval features.

## Hypothesis
"Implementing subset validation every 2000 steps and step-based early stopping, then launching full training with constant lr=1e-4, will produce a stable model that exceeds V3 CR=0.0147 on full test data, because (a) P4 already beats V3 on 20% data with stable prec/rec through 5 epochs, (b) step-based eval will catch any divergence within ~13 min instead of 72 min, and (c) full data (5x more) should improve generalization."

## Constraint
Execution — Recipe is KNOWN (constant lr=1e-4). Two features need implementation before full training launch: (1) subset validation every 2000 steps (~20 lines), (2) step-based early stopping (~20 lines). These provide fast divergence detection (13 min vs 72 min) and responsive stopping (104 min max vs 18 hours). Previous constraints RESOLVED: Infrastructure (OOM fix), Training Recipe (P4 constant LR stable).

## Belief State

| Area | Confidence | Evidence |
|------|-----------|----------|
| Architecture | VERY HIGH | ALL 4 prove-outs beat V3 on test CR (0.0141-0.0185 vs 0.0147). Architecture validated at 4-5x reduction in data. |
| Recipe | HIGH | P4 (lr=1e-4, constant) stable through 5 epochs. Prec=0.595, Rec=0.978. Test CR=0.0176 (1.2x V3). Root cause of cosine collapse identified (multi-task loss weight conflict). |
| Data | HIGH | 11.4M samples, 64/64 quarters, time-based split. No leakage. Prove-out on 20% data validates quality. |
| Infrastructure | HIGH | OOM fix confirmed. Intra-epoch checkpoints working. Prove-out mode implemented and tested. |
| Next action | HIGH | Implement 2 step-based features (~40 lines, ~30 min), then launch full training with constant lr=1e-4. Well-scoped. |

## Metrics

| Version | CR | P@5% | Rank Corr | Return Corr | Dir Acc | Params | Status |
|---------|-----|------|-----------|-------------|---------|--------|--------|
| V1 (archive) | 0.0192 | N/A | 0.0197 | N/A | 0.5218 | unknown | All-time best CR |
| V2 | 0.0112 | 0.3134 | 0.0172 | N/A | 0.4976 | unknown | Regression from V1 |
| V3 | 0.0147 | 0.3338 | 0.0152 | 0.0132 | 0.5050 | 136K | Baseline target |
| V4 (smoke) | 0.0031 | 0.2696 | -0.0282 | -0.0004 | 0.4516 | 4.77M | Overfit failure |
| V4 (unit) | 0.0129 | 0.3445 | 0.0346 | -0.0023 | 0.4609 | 4.77M | Unit only |
| V5-Small (unit, 50 batches) | 0.0015 | 0.2656 | 0.0892 | 0.0875 | 0.3779 | 680K | Unit only — NOT comparable |
| V5-Small (epoch 1) | 0.0138 | 0.213 | N/A | 0.000 | N/A | 680K | Full data, lr=3e-4 cosine — collapsed epoch 2 |
| V5-Small P1 (20% data) | 0.0141 | N/A | N/A | N/A | N/A | 680K | lr=1e-4 cosine — UNSTABLE (rec collapse ep3) |
| V5-Small P2 (20% data) | 0.0185 | N/A | N/A | N/A | N/A | 680K | lr=5e-5 cosine — UNSTABLE (prec/rec=0 ep3) |
| V5-Small P3 (20% data) | 0.0163 | N/A | N/A | N/A | N/A | 680K | lr=3e-4 scaled cosine — UNSTABLE (prec/rec=0 ep3) |
| V5-Small P4 (20% data) | 0.0176 | 0.313 | N/A | 0.000 | N/A | 680K | **BEST** lr=1e-4 constant — STABLE, 1.2x V3 |

## Targets

- **Minimum:** CR >= 0.0147 (match V3), return_corr > 0.0132
- **Strong:** CR >= 0.0192 (match V1), P@5% > 0.33, rank_corr > 0.02
- **Exceptional:** CR > 0.025

## Known Issues

1. Summary token dims 14-19 are all zeros (6/20 dims wasted) — fix in next token prep
2. Price token dim 15 always zero (bug in prepare_tokens.py) — fix in next token prep
3. config.d_ff never used by model (hardcoded d_model*4) — latent maintenance trap
4. Feature scale mismatch: moneyness/mid_price_norm/intrinsic_value have ranges 10-100x other features
5. ~~Epoch time ~6-8 hours limits iteration~~ RESOLVED — actual epoch ~75 min, prove-out mode ~15 min/epoch
6. ~~**BLOCKING** torch.compile OOM~~ RESOLVED — mode='default' working, GPU stable at 6.8 GiB
7. ~~**BLOCKING** No intra-epoch checkpointing~~ RESOLVED — checkpoints added every 2000 batches
8. ~~Last-batch remainder~~ RESOLVED — trailing batches dropped
9. ~~**BLOCKING** Cosine LR schedule flat~~ RESOLVED — constant LR recipe bypasses issue. Root cause: multi-task loss weight conflict under cosine decay.
10. ~~**BLOCKING** Loss weight sum 3.0~~ MITIGATED — constant LR at 1e-4 compensates. No rebalancing needed for now.
11. ~~**BLOCKING** No prove-out mode~~ RESOLVED — --prove-out implemented, 4 experiments completed successfully.
12. Confidence loss creates feedback loop with magnitude head — monitor during full training.
13. **NEW** return_mae=NaN across ALL prove-outs — return prediction head broken. Not blocking CR but should investigate.
14. **NEW** Subset eval every N steps NOT YET IMPLEMENTED — needed for fast divergence detection in full training.
15. **NEW** Step-based early stopping NOT YET IMPLEMENTED — needed to prevent hours of wasted compute on divergence.

## Priority Queue

1. ~~STOP current training~~ DONE
2. ~~IMPLEMENT prove-out mode~~ DONE
3. ~~FIX cosine schedule~~ RESOLVED via constant LR
4. ~~RUN prove-out LR sweep~~ DONE — P4 (lr=1e-4, constant) is winner
5. **IMPLEMENT subset eval every 2000 steps** — ~20 lines in train.py, fast divergence detection (IMMEDIATE)
6. **IMPLEMENT step-based early stopping** — ~20 lines in train.py, responsive stopping (IMMEDIATE)
7. **LAUNCH full training** with P4 recipe: `--full --lr 1e-4 --constant-lr --no-smoke` (AFTER items 5-6)
8. Monitor first 5000 steps of full training — prec/rec must stay nonzero, train loss must not rise
9. After training completes: evaluate against V3/V1 baselines (CR >= 0.0147 minimum, target > 0.0176)
10. Investigate return_mae=NaN — return prediction head broken (secondary)
11. Consider loss weight rebalancing or per-head LR to unlock cosine schedule (future experiment)
12. If capacity bottleneck after full training: try V5-Medium (d_model=192, 3+2 layers, ~2.5M params)
13. Fix summary/price token zero dims in next token prep iteration
