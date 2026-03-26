# Goal Tracker

Living document maintained by the manager. Updated during every "manage" command.

## Current Goal

V4 full training — token preparation completed (11.4M samples), next step is to launch V4 transformer training and evaluate against V3 baseline.

## Goal Status

**READY TO START** — V4 token prep complete. Waiting for training launch.

## Key Metrics — Model Comparison

| Version | Captured Return | P@5% | Rank Corr | Return Corr | Direction Acc | Params | Status |
|---------|----------------|------|-----------|-------------|---------------|--------|--------|
| V1 | 0.0064 | 0.339 | -0.004 | — | — | 49K | Archived |
| V2 | 0.0112 | 0.313 | 0.017 | — | 0.531 | 151K | Archived |
| V3 | 0.0147 | 0.334 | 0.015 | 0.013 | 0.505 | 136K | Current best |
| V4 | — | — | — | — | — | 4.77M | Token prep done, training pending |

**Foresight upper bound:** 170% return in 50 days, 9.16 Sharpe, 74% daily win rate.

## V4 Target Thresholds

To justify V4's complexity (4.77M params vs V3's 136K), it must exceed V3 on:
- Captured return > 0.0147
- Return correlation > 0.013
- P@5% > 0.334

If V4 underperforms after full training, investigate: overfitting, learning rate, loss weights, data quality.

## Next Goals (priority order)

1. **Launch V4 full training** — use ChunkLoader + IterableDataset, monitor loss/metrics
2. **Evaluate V4** — compare against V3 on standard metrics, analyze per-quarter stability
3. **Iterate V4 training recipe** — adjust loss weights, lr schedule, augmentation per direction.md
4. **Portfolio model** — once backbone is satisfactory, begin cross-stock portfolio model

## Data Pipeline Status

| Dataset | Status | Samples | Chunks |
|---------|--------|---------|--------|
| V4 tokens (train) | Complete | 5,798,357 | 138 |
| V4 tokens (val) | Complete | 2,769,223 | 63 |
| V4 tokens (test) | Complete | 2,844,134 | 63 |
| V4 total | Complete | 11,411,714 | 264 |

## Direction.md Priority Tracker

From `firstrate_learning/direction.md`:

| Priority | Item | Status |
|----------|------|--------|
| P1 | Improve return magnitude prediction (direct regression head) | V3 implemented, V4 has return head |
| P2 | Add price/momentum features (5-day returns, vol, IV ratio) | V4 uses raw tokens instead |
| P3 | Improve embedding informativeness (contrastive learning) | Not started |
| P4 | Remove dead features (24% zeroed) | V4 bypasses via raw tokens |
| P5 | Calibration (temperature scaling, label smoothing) | Not started |
| P6 | Ensemble/SWA for stability | Not started |
