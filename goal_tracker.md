# Goal Tracker

Updated by Step 4 (Learn & Update) — 2026-03-27 22:30 UTC (Cycle 4)

## Current Goal
Validate V5-Small temporal architecture — full training run, evaluate against V3 baseline (CR >= 0.0147).

## Status
V5-Small full training LAUNCHED but CRASHED at batch 2350/11325 (20.7% of epoch 1) with CUDA OOM. Loss trajectory was excellent (0.66 to 0.27) before crash. Root cause: torch.compile mode='reduce-overhead' recorded 51 CUDA Graph shapes, leaking ~17GB of untracked GPU memory. No checkpoint survived (only per-epoch checkpoints, crash before epoch 1 ended). All 2350 batches of training lost. Dev idle at prompt, GPU idle. Fixes identified: switch torch.compile to mode='default', add intra-epoch checkpointing every 2000 batches, drop incomplete trailing batches.

## Hypothesis
"Disabling CUDA Graphs in torch.compile (mode='default' instead of 'reduce-overhead') will allow V5-Small full training to complete and achieve CR >= 0.0147, because the model was learning well (loss 0.66 to 0.27 before crash) and the OOM is purely a CUDA Graph memory management issue, not a model or data problem."

## Constraint
Infrastructure — CUDA OOM from torch.compile CUDA Graphs. Training was launched and ran well (loss 0.66 to 0.27) but crashed at batch 2350 due to 51 CUDA Graph shape recordings consuming ~17GB. Fix: switch torch.compile to mode='default' (eliminates CUDA Graphs while keeping kernel fusion). Secondary: add intra-epoch checkpointing (prevent losing hours of training on crash). Dev idle, GPU idle, awaiting fixes and relaunch.

## Belief State

| Area | Confidence | Evidence |
|------|-----------|----------|
| Architecture | HIGH | V5-Small loss 0.66->0.27 over 2350 batches (20.7% of epoch 1) — strong learning signal. 6x stronger correlations than V3 after 50 batches in unit test. |
| Recipe | MEDIUM-HIGH | lr=3e-4, batch=512, dropout=0.25 producing steady loss decline. No instability/divergence observed in 2350 batches. |
| Data | HIGH | 11.4M samples, 64/64 quarters, time-based split. No leakage. Token data clean. Variable batch sizes from chunk remainders (46 unique sizes). |
| Infrastructure | MEDIUM | torch.compile mode='reduce-overhead' causes OOM via CUDA Graphs. Fix identified (mode='default'). No intra-epoch checkpointing means crashes lose hours of work. |
| Next action | VERY HIGH | Apply 3 fixes (compile mode, intra-epoch checkpoint, trailing batch drop) and relaunch. Straightforward. |

## Metrics

| Version | CR | P@5% | Rank Corr | Return Corr | Dir Acc | Params | Status |
|---------|-----|------|-----------|-------------|---------|--------|--------|
| V1 (archive) | 0.0192 | N/A | 0.0197 | N/A | 0.5218 | unknown | All-time best CR |
| V2 | 0.0112 | 0.3134 | 0.0172 | N/A | 0.4976 | unknown | Regression from V1 |
| V3 | 0.0147 | 0.3338 | 0.0152 | 0.0132 | 0.5050 | 136K | Baseline target |
| V4 (smoke) | 0.0031 | 0.2696 | -0.0282 | -0.0004 | 0.4516 | 4.77M | Overfit failure |
| V4 (unit) | 0.0129 | 0.3445 | 0.0346 | -0.0023 | 0.4609 | 4.77M | Unit only |
| V5-Small (unit, 50 batches) | 0.0015 | 0.2656 | 0.0892 | 0.0875 | 0.3779 | 680K | Unit only — NOT comparable |
| V5-Small (partial, 2350 batches) | N/A | N/A | N/A | N/A | N/A | 680K | CRASHED at batch 2350/11325, loss 0.66->0.27, no eval metrics |

## Targets

- **Minimum:** CR >= 0.0147 (match V3), return_corr > 0.0132
- **Strong:** CR >= 0.0192 (match V1), P@5% > 0.33, rank_corr > 0.02
- **Exceptional:** CR > 0.025

## Known Issues

1. Summary token dims 14-19 are all zeros (6/20 dims wasted) — fix in next token prep
2. Price token dim 15 always zero (bug in prepare_tokens.py) — fix in next token prep
3. config.d_ff never used by model (hardcoded d_model*4) — latent maintenance trap
4. Feature scale mismatch: moneyness/mid_price_norm/intrinsic_value have ranges 10-100x other features
5. Epoch time ~6-8 hours limits iteration to 1-3 experiments/week
6. **BLOCKING** torch.compile mode='reduce-overhead' causes CUDA OOM at batch ~2350 via CUDA Graph memory leak — FIX: switch to mode='default'
7. **BLOCKING** No intra-epoch checkpointing — 6-8 hour epochs at risk of total loss on crash — FIX: save every 2000 batches
8. Last-batch remainder produces 46 unique batch sizes per epoch — FIX: drop incomplete trailing batches (change `< 2` to `< bs`)

## Priority Queue

1. **FIX OOM:** Change torch.compile mode='reduce-overhead' to mode='default' in train.py line 314 (IMMEDIATE)
2. **ADD INTRA-EPOCH CHECKPOINTING:** Save checkpoint every 2000 batches in train.py (IMMEDIATE)
3. **DROP TRAILING BATCHES:** Change `< 2` to `< bs` in data_loader.py line 121 (IMMEDIATE)
4. **RELAUNCH** V5-Small full training after fixes applied (IMMEDIATE — GPU idle)
5. Monitor GPU memory trend after relaunch — must stay flat, not growing (confirms OOM fix)
6. Monitor training for overfitting (watch val loss diverge after epoch 3-5)
7. Monitor per-component loss magnitudes for conflicts
8. After training completes: evaluate against V3/V1 baselines
9. If overfitting: increase dropout to 0.30 or weight_decay to 0.03
10. If capacity bottleneck: try V5-Medium (d_model=192, 3+2 layers, ~2.5M params)
11. Fix summary/price token zero dims in next token prep iteration
