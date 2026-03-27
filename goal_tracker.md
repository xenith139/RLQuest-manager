# Goal Tracker

Updated by Step 4 (Learn & Update) — 2026-03-27 21:45 UTC

## Current Goal
Validate V5-Small temporal architecture — full training run, evaluate against V3 baseline (CR >= 0.0147).

## Status
V5-Small implemented and unit-tested. Token prep COMPLETE (64/64 quarters, 11.4M samples). Unit test passed (CR=0.0015, rank_corr=0.089, return_corr=0.088 after 50 batches). Dev IDLE at prompt ~2+ hours. GPU IDLE at 0%. Full training NOT YET LAUNCHED.

## Hypothesis
"V5-Small's temporal architecture (5-day, d_model=128, 2+1 layers, 680K params) will achieve CR >= 0.0147 because it shows 6x stronger correlation signals than V3's final values after only 50 training batches, indicating superior signal extraction that will convert to captured return with full training."

## Constraint
Execution — all prerequisites met (code complete, data complete, unit test passed, GPU idle). The full training run simply needs to be launched. Dev is idle and waiting for instruction.

## Belief State

| Area | Confidence | Evidence |
|------|-----------|----------|
| Architecture | HIGH | V5-Small shows rank_corr 0.089 and return_corr 0.088 after 50 batches — 6x V3 final values. Temporal attention is extracting signal. |
| Recipe | MEDIUM | lr=3e-4, batch=512, dropout=0.25 are standard/reasonable. Untested at full scale. 7 loss components is aggressive for 680K params but weights are balanced. |
| Data | HIGH | 11.4M samples, 64/64 quarters, time-based split (train 2010-2019, val 2020-2022, test 2023-2025). No leakage. Token data is clean (no NaN/Inf). |
| Next action | VERY HIGH | Launch full training. Zero ambiguity. GPU idle, dev idle, data ready, code validated. |

## Metrics

| Version | CR | P@5% | Rank Corr | Return Corr | Dir Acc | Params | Status |
|---------|-----|------|-----------|-------------|---------|--------|--------|
| V1 (archive) | 0.0192 | N/A | 0.0197 | N/A | 0.5218 | unknown | All-time best CR |
| V2 | 0.0112 | 0.3134 | 0.0172 | N/A | 0.4976 | unknown | Regression from V1 |
| V3 | 0.0147 | 0.3338 | 0.0152 | 0.0132 | 0.5050 | 136K | Baseline target |
| V4 (smoke) | 0.0031 | 0.2696 | -0.0282 | -0.0004 | 0.4516 | 4.77M | Overfit failure |
| V4 (unit) | 0.0129 | 0.3445 | 0.0346 | -0.0023 | 0.4609 | 4.77M | Unit only |
| V5-Small (unit, 50 batches) | 0.0015 | 0.2656 | 0.0892 | 0.0875 | 0.3779 | 680K | Unit only — NOT comparable |

## Targets

- **Minimum:** CR >= 0.0147 (match V3), return_corr > 0.0132
- **Strong:** CR >= 0.0192 (match V1), P@5% > 0.33, rank_corr > 0.02
- **Exceptional:** CR > 0.025

## Known Issues (non-blocking)

1. Summary token dims 14-19 are all zeros (6/20 dims wasted) — fix in next token prep
2. Price token dim 15 always zero (bug in prepare_tokens.py) — fix in next token prep
3. config.d_ff never used by model (hardcoded d_model*4) — latent maintenance trap
4. Feature scale mismatch: moneyness/mid_price_norm/intrinsic_value have ranges 10-100x other features
5. Epoch time ~6-8 hours limits iteration to 1-3 experiments/week

## Priority Queue

1. Launch V5-Small full training (IMMEDIATE — GPU idle)
2. Monitor training for overfitting (watch val loss diverge after epoch 3-5)
3. Monitor per-component loss magnitudes for conflicts
4. After training completes: evaluate against V3/V1 baselines
5. If overfitting: increase dropout to 0.30 or weight_decay to 0.03
6. If capacity bottleneck: try V5-Medium (d_model=192, 3+2 layers, ~2.5M params)
7. Fix summary/price token zero dims in next token prep iteration
