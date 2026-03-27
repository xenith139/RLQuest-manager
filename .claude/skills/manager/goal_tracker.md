# Goal Tracker

Living document maintained by the manager. Updated during every "manage" command's LEARN phase.

## Current Goal

Close the gap to foresight upper bound by improving the backbone model. Two parallel tracks: (1) complete V4 training for a verified baseline, (2) design and implement V5 with temporal architecture improvements.

## Goal Status

**PARALLEL EXECUTION** — V4 ready to resume from archive weights (epoch 5, CR=0.0184). V5 design complete (model_v5_design.md). Next: launch V4 training (GPU) while dev implements V5 (code).

## Current Hypothesis

"V5's temporal architecture (5-day multi-surface attention + temporal diffs) will roughly double V4's captured return from ~1.84% to ~3.5% per 10-day period because the foresight analysis shows temporal momentum/z-score features are the strongest predictors of big moves, and V4's single-day snapshot cannot encode these patterns."

**Test**: Implement V5, train to convergence, compare captured return against V4 baseline.
**Success**: V5 captured return > 0.025 (35% better than V4's 0.0184).
**Failure**: V5 captured return < V4's 0.0184 after full training → investigate overfitting, data issues, or design flaws.
**Cost of being wrong**: ~2 weeks dev time + ~7 days GPU = significant but justified by 2x expected improvement.

## Constraint Analysis

**Current constraint**: Architecture — V4's single-day snapshot misses temporal patterns (momentum, z-scores, IV trend) that the foresight analysis identifies as the strongest predictive signals.

**Evidence**:
- Foresight analysis: `total_volume_mom_5d`, `put_call_volume_ratio_mom_5d`, `total_volume_zscore_20d` are top predictive features — all require multi-day comparison
- V4 gets 1 compressed price token with 6 scalars — cannot reconstruct temporal shape
- V2/V3 had 5-day temporal encoder with Conv1d and temporal diffs — V4 removed this entirely
- V4 still beat V3 (proving raw-token attention works) but has architectural ceiling

**Next constraint after this is resolved**: Training recipe optimization (loss weights, lr schedule, augmentation strategies for V5's larger architecture).

## Belief State

| Component | Confidence | Reasoning |
|-----------|-----------|-----------|
| V4 architecture reaches ~60% annualized | HIGH | Demonstrated CR=0.0184 in 5 epochs |
| V4 has a ceiling around ~60% annualized | MEDIUM-HIGH | Single-day snapshot can't encode temporal momentum — the strongest predictor |
| V5 temporal design will improve on V4 | MEDIUM | Grounded in foresight analysis + V2/V3 temporal evidence, but untested |
| V5 estimated ~125% annualized | LOW-MEDIUM | Rough estimate, many unknowns in implementation |
| Parallel V4+V5 is fastest path | HIGH | GPU and dev are independent resources, V4 gives baseline |

## Key Metrics — Model Comparison

| Version | Captured Return | P@5% | Rank Corr | Return Corr | Direction Acc | Params | Status |
|---------|----------------|------|-----------|-------------|---------------|--------|--------|
| V1 | 0.0064 | 0.339 | -0.004 | — | — | 49K | Archived |
| V2 | 0.0112 | 0.313 | 0.017 | — | 0.531 | 151K | Archived |
| V3 | 0.0147 | 0.334 | 0.015 | 0.013 | 0.505 | 136K | Baseline |
| V4 | 0.0184* | 0.401* | 0.035* | 0.035* | ~0.51* | 4.77M | *epoch 5, pre-infra. Needs full run. |
| V5 | — | — | — | — | — | ~5M | Design complete, implementation pending |

**Foresight upper bound:** 170% return in 50 days, 9.16 Sharpe, 74% daily win rate.

## Next Goals (priority order, with ETG)

1. **Launch V4 full training from archive** (GPU track — ~20hr) — gives verified baseline
2. **Implement V5 data preparation** (dev track — ~1 week, parallel with #1) — 5-day multi-surface tokens
3. **Implement V5 model + training** (dev track — ~1 week after #2)
4. **V5 smoke test → full training** (~7 days GPU)
5. **Compare V4 vs V5** — decide which backbone to use for portfolio model
6. **Portfolio model** — once backbone is satisfactory

## Data Pipeline Status

| Dataset | Status | Samples | Chunks |
|---------|--------|---------|--------|
| V4 tokens (train) | Complete | 5,798,357 | 138 |
| V4 tokens (val) | Complete | 2,769,223 | 63 |
| V4 tokens (test) | Complete | 2,844,134 | 63 |
| V5 tokens | Not started | — | — |

## Direction.md Priority Tracker

| Priority | Item | Status |
|----------|------|--------|
| P1 | Improve return magnitude prediction | V4 has return head. V5: asymmetric weighting + 5 quantiles. |
| P2 | Add price/momentum features | V4: 1 price token. V5: 5-day multi-surface + 20-dim tokens. |
| P3 | Contrastive learning for embeddings | V5 design includes contrastive loss. |
| P4 | Remove dead features | V4/V5 bypass via raw tokens. Done. |
| P5 | Calibration (temperature, smoothing) | V4 has label smoothing. V5: + confidence head. |
| P6 | Ensemble/SWA | Deferred until backbone stabilizes. |
