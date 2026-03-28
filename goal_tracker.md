# Goal Tracker

Updated by Step 4 (Learn & Update) — 2026-03-28 00:20 UTC (Cycle 5)

## Current Goal
Validate V5-Small temporal architecture — full training run, evaluate against V3 baseline (CR >= 0.0147).

## Status
V5-Small full training relaunched after OOM fix (mode='default'). Epoch 1 completed successfully: CR=0.0138, Prec=0.591, Rec=0.881 — within 6% of V3 baseline. MODE COLLAPSE at epoch 2: train loss ROSE (0.5508->0.5894), val loss spiked 15.7% (0.6111->0.7073), precision/recall collapsed to 0.000. Epoch 3 running with continued degradation. Root cause identified: cosine LR schedule spread over 566K steps = effectively flat for first 10 epochs. Combined with loss weights summing to 3.0, effective LR is ~9e-4 (3x too high). Best model from epoch 1 preserved. STOP training, implement prove-out mode for rapid LR/recipe sweeps.

## Hypothesis
"Implementing prove-out mode (20% data, 5 epochs, ~90 min/experiment) and sweeping LR in {5e-5, 1e-4, 2e-4, 3e-4 with scaled cosine} will identify a training recipe where train loss declines or stays flat across all 5 epochs, because the mode collapse is caused by lr=3e-4 with effectively flat cosine schedule (effective LR ~9e-4 due to loss weight sum of 3.0), and reducing LR or scaling cosine decay to the actual run length will prevent overshooting the epoch-1 minimum."

## Constraint
Training Recipe — LR overshooting causes mode collapse after epoch 1. Cosine schedule spread over 566K steps barely decays (99.6% of peak at epoch 2). Six loss components with weights summing to 3.0 effectively triple the gradient magnitude, making lr=3e-4 behave like ~9e-4 in single-task terms. Fix: implement prove-out mode for rapid LR/schedule sweeps, then apply winning recipe to full training. Infrastructure constraint (OOM) is RESOLVED.

## Belief State

| Area | Confidence | Evidence |
|------|-----------|----------|
| Architecture | HIGH | Epoch 1 CR=0.0138 (94% of V3 target) with Prec=0.591, Rec=0.881 — model learns well. Collapse is recipe, not architecture. |
| Recipe | LOW | lr=3e-4 causes mode collapse at epoch 2. Cosine schedule flat (99.6% at epoch 2). Loss weight sum 3.0 = effective LR ~9e-4. Need LR sweep via prove-out. |
| Data | HIGH | 11.4M samples, 64/64 quarters, time-based split. No leakage. Epoch 1 results validate data quality. |
| Infrastructure | HIGH | OOM fix confirmed (GPU stable at 6.8 GiB). mode='default' working. Intra-epoch checkpoints added. |
| Next action | HIGH | Stop training, implement prove-out mode, run LR sweep. Well-defined plan with clear success criteria. |

## Metrics

| Version | CR | P@5% | Rank Corr | Return Corr | Dir Acc | Params | Status |
|---------|-----|------|-----------|-------------|---------|--------|--------|
| V1 (archive) | 0.0192 | N/A | 0.0197 | N/A | 0.5218 | unknown | All-time best CR |
| V2 | 0.0112 | 0.3134 | 0.0172 | N/A | 0.4976 | unknown | Regression from V1 |
| V3 | 0.0147 | 0.3338 | 0.0152 | 0.0132 | 0.5050 | 136K | Baseline target |
| V4 (smoke) | 0.0031 | 0.2696 | -0.0282 | -0.0004 | 0.4516 | 4.77M | Overfit failure |
| V4 (unit) | 0.0129 | 0.3445 | 0.0346 | -0.0023 | 0.4609 | 4.77M | Unit only |
| V5-Small (unit, 50 batches) | 0.0015 | 0.2656 | 0.0892 | 0.0875 | 0.3779 | 680K | Unit only — NOT comparable |
| V5-Small (epoch 1) | 0.0138 | 0.213 | N/A | 0.000 | N/A | 680K | *BEST V5* — 94% of V3 target |
| V5-Small (epoch 2) | 0.0114 | 0.251 | N/A | 0.000 | N/A | 680K | MODE COLLAPSE — Prec/Rec=0.000 |

## Targets

- **Minimum:** CR >= 0.0147 (match V3), return_corr > 0.0132
- **Strong:** CR >= 0.0192 (match V1), P@5% > 0.33, rank_corr > 0.02
- **Exceptional:** CR > 0.025

## Known Issues

1. Summary token dims 14-19 are all zeros (6/20 dims wasted) — fix in next token prep
2. Price token dim 15 always zero (bug in prepare_tokens.py) — fix in next token prep
3. config.d_ff never used by model (hardcoded d_model*4) — latent maintenance trap
4. Feature scale mismatch: moneyness/mid_price_norm/intrinsic_value have ranges 10-100x other features
5. ~~Epoch time ~6-8 hours limits iteration~~ Actual epoch time ~75 min. Prove-out mode needed for fast sweeps.
6. ~~**BLOCKING** torch.compile OOM~~ RESOLVED — mode='default' working, GPU stable at 6.8 GiB
7. ~~**BLOCKING** No intra-epoch checkpointing~~ RESOLVED — checkpoints added every 2000 batches
8. ~~Last-batch remainder~~ RESOLVED — trailing batches dropped
9. **BLOCKING** Cosine LR schedule spread over 566K steps = flat for first 10 epochs. Must scale to actual run length.
10. **BLOCKING** Loss weight sum 3.0 makes effective LR ~3x nominal. Consider normalizing to 1.0 or reducing LR proportionally.
11. **BLOCKING** No prove-out mode — cannot iterate on recipe quickly. Need --prove-out flag (20% data, 5 epochs, scaled cosine).
12. Confidence loss creates feedback loop with magnitude head — amplifies instability once model starts oscillating.

## Priority Queue

1. **STOP current training** — epoch 3+ is wasting GPU time, best model preserved from epoch 1 (IMMEDIATE)
2. **IMPLEMENT prove-out mode** — --prove-out flag: 28/138 train chunks, 13/63 val chunks, 5 epochs, cosine scaled to prove-out length, --lr and --tag args (~30 lines) (IMMEDIATE)
3. **FIX cosine schedule** — total_steps must use actual run length, not max_epochs * full_data_batches (IMMEDIATE)
4. **RUN prove-out LR sweep** — P1: lr=1e-4 cosine, P2: lr=5e-5 cosine, P3: lr=3e-4 scaled cosine, P4: lr=1e-4 constant (~6 hours total)
5. **RELAUNCH full training** with winning recipe from sweep
6. Monitor epoch 2 of full training — train loss must NOT rise
7. After training completes: evaluate against V3/V1 baselines (CR >= 0.0147 minimum)
8. Consider normalizing loss weights to sum to 1.0 (secondary experiment)
9. If capacity bottleneck after recipe fix: try V5-Medium (d_model=192, 3+2 layers, ~2.5M params)
10. Fix summary/price token zero dims in next token prep iteration
