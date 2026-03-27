## Step 2: Gap & Constraint — 2026-03-27 21:30 UTC

### Constraint unchanged, hypothesis unchanged, gap unmeasurable (same as last cycle).

The situation is identical to the previous cycle (20:45 UTC). Nothing has changed in 45 minutes: no training launched, GPU still at 0%, dev still idle at prompt for ~2.5 hours.

### Best performance: V1 (archive) epoch 33 CR=0.0192
### Baseline target: V3 epoch 12 CR=0.0147, return_corr=0.0132
### Target: CR >= 0.0147 with return_corr > 0.0132
### Gap: Cannot measure — V5 has only 50 training batches (unit test). V5 unit test CR=0.0015 is not a valid performance number. Early correlation signals (rank_corr=0.089, return_corr=0.088) are 6x stronger than V3 final values, suggesting high potential.
### Gap trend: Stable/unknown — no new data since last cycle. Cannot assess whether gap is shrinking until full training completes.
### Constraint: Execution — all prerequisites met, full training simply has not been launched
### Constraint changed from last cycle: No. Was Execution at 20:45, still Execution now. (Prior to that it was Data — V5 token prep incomplete. That was resolved.)
### Hypothesis: "Launching V5-Small full training will achieve CR >= 0.0147 because the temporal architecture shows 6x stronger correlation signals than V3 after only 50 batches, indicating superior signal extraction that will convert to captured return with full training."
### Success criteria: Test CR >= 0.0147, return_corr > 0.0132, training completes without failure
### Failure criteria: Test CR < 0.005 at convergence with return_corr < 0.01 triggers recipe investigation; CR at V4 levels (~0.003) with negative correlations triggers architecture pivot

### Urgency note: GPU has now been idle ~2.5 hours. Every minute of idle GPU is wasted capacity. The ONLY action needed is to instruct the dev agent to launch full training. No further analysis, investigation, or preparation is required.

---

## Persistent Notes
- Previous constraint: Execution (unchanged for 2 cycles; before that it was Data — token prep)
- Previous gap: Unmeasurable (no full V5 training run exists)
- Hypothesis history: Hypothesis has been stable since first formulation — "launch full training, expect CR >= 0.0147 based on strong early correlation signals." No evidence has arrived to update this because training hasn't been run.
- Key evidence accumulated:
  - V5-Small unit test: 679K params, CR=0.0015, rank_corr=0.089, return_corr=0.088 (50 batches)
  - V3 baseline: 136K params, CR=0.0147, rank_corr=0.0152, return_corr=0.0132 (full training, epoch 12)
  - V1 best ever: CR=0.0192, rank_corr=0.0197, dir_acc=0.5218 (full training, epoch 33)
  - V4 was a regression: 4.77M params, CR=0.0031 (smoke test), negative correlations. Overparameterized.
  - V5 data: 64/64 quarters, 11.4M samples, complete and verified
  - GPU: Quadro RTX 6000, 24GB, idle since 18:57 UTC
  - Known data issues: 6/20 summary token dims are zeros, price token dim 15 always zero — not blocking but may limit ceiling
- Watch items for next cycle:
  - Has full training been launched? (Check for new run dirs in models/)
  - If training running: GPU utilization, epoch count, val loss trend, any NaN/crash
  - If training complete: compare test CR, rank_corr, return_corr, P@5% against V3 baseline
  - Watch for overfitting after epoch 3-5 (val loss divergence from train loss)
  - Estimated full training time: 2-4 hours based on V4 timing scaled for smaller V5 model
