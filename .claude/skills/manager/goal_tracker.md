# Goal Tracker

Updated by Step 4 (Learn & Update) on every manage cycle.

## Current Goal
Validate V5-Small temporal architecture — train and evaluate against V4 baseline.

## Status
V5-Small implemented (config, model, data_loader, train.py). Unit test passed. Token prep running. Awaiting data completion to start training.

## Hypothesis
"V5-Small's temporal architecture (5-day, d_model=128, 2+1 layers, ~1M params) will improve on V4's CR=0.0187 without overfitting."

## Constraint
Data — V5 token prep incomplete (~47/64 quarters as of last check).

## Metrics

| Version | CR | P@5% | Rank Corr | Params | Status |
|---------|-----|------|-----------|--------|--------|
| V3 | 0.0147 | 0.334 | 0.015 | 136K | Baseline |
| V4 | 0.0187 (ep1) | 0.404 | 0.035 | 4.77M | Overfits ep2+ |
| V5-Small | — | — | — | ~1M | Training pending |
