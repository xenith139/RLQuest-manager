# Step 2: Gap Analysis & Constraint Identification

**Timestamp:** 2026-03-27 20:45 UTC

---

## 2.1 Current Performance

### Historical Model Performance Summary (all full training runs, test set metrics)

| Version | Params | Best Epoch | CR | P@5% | Rank Corr | Return Corr | Dir Acc |
|---------|--------|------------|------|------|-----------|-------------|---------|
| V1 (archive) | unknown | 33 | 0.0192 | N/A | 0.0197 | N/A | 0.5218 |
| V2 (firstrate_learning) | unknown | 12 | 0.0112 | 0.3134 | 0.0172 | N/A | 0.4976 |
| V3 | 135,622 | 12 | 0.0147 | 0.3338 | 0.0152 | 0.0132 | 0.5050 |
| V4 (smoke test only) | 4,769,287 | 2 | 0.0031 | 0.2696 | -0.0282 | -0.0004 | 0.4516 |
| V4 (unit test) | 4,769,287 | 1 | 0.0129 | 0.3445 | 0.0346 | -0.0023 | 0.4609 |
| V5 (unit test, 50 batches) | 679,498 | 1 | 0.0015 | 0.2656 | 0.0892 | 0.0875 | 0.3779 |

### Best fully-trained model: V1 (archive)
- **CR = 0.0192**, rank_corr = 0.0197, direction_acc = 0.5218, best epoch 33

### Best model goals.md references as baseline: V3
- **CR = 0.0147**, P@5% = 0.3338, rank_corr = 0.0152, return_corr = 0.0132, direction_acc = 0.5050, best epoch 12, 135K params

### Current architecture (V5-Small): unit test only, NOT fully trained
- **CR = 0.0015** from 50 training batches (1 epoch on a tiny subset). This number is essentially meaningless for performance comparison. It only shows the model can learn and gradients flow.
- However, the early signals are notable: rank_corr = 0.0892 and return_corr = 0.0875 after only 50 batches are **5.9x and 6.6x higher** than V3's final fully-trained values (0.0152 and 0.0132 respectively). This suggests the temporal architecture is picking up correlation signal much faster than previous versions.
- The low CR (0.0015) despite high correlation likely means the model hasn't yet learned to calibrate its predictions for profitable thresholding -- it correlates with returns but doesn't yet identify the right stocks to buy. This is expected at 50 batches.

### V4: A regression from V3
- V4 full smoke test: CR = 0.0031, which is **4.7x worse than V3** (0.0147) and **6.2x worse than V1** (0.0192).
- V4 has negative return correlation (-0.0004) and negative rank correlation (-0.0282) in the smoke test. The model is anti-correlated.
- V4 unit test showed better CR (0.0129) but still negative return_corr (-0.0023). This is a concerning pattern: the model gets some captured return through the classification head but has no return prediction signal.
- V4 was a 4.77M parameter model -- 35x larger than V3 (136K) -- which likely overfit on limited data. V5-Small's 680K params is a deliberate correction.

---

## 2.2 Target Performance

### goals.md targets:
1. **Primary target:** Match or exceed V3 captured return of 0.0147 with better return correlation.
2. **Stretch target:** Progress toward foresight benchmark (170% return in 50 days, 9.16 Sharpe).
3. **Additional goals:** Iterate on training recipe, evaluate against V1/V2/V3 on standard metrics.

### Foresight upper bound calculation:
- 170% in 50 trading days = compounding rate of (1 + 1.70)^(1/50) - 1 = ~2.0% per day
- Per 10-day period: (2.70)^(10/50) - 1 = ~22.0% per 10-day period
- Annualized (252 trading days): extraordinary -- this is the theoretical ceiling, not a realistic near-term target.

### Realistic near-term target:
- CR >= 0.0147 (matching V3)
- Return correlation > 0.0132 (V3's level) -- goals.md specifically says "with better return correlation"
- P@5% > 0.3338 (V3's level)
- Stretch: CR >= 0.0192 (matching V1, the all-time best)

### What CR = 0.0147 means in annualized terms:
- CR is captured return per evaluation period. If the evaluation periods are 10-day windows, CR = 0.0147 means ~1.47% per 10-day period.
- Annualized: (1.0147)^(25.2) - 1 = ~44.6% annualized return (rough estimate, 252/10 = 25.2 periods per year).
- This is a meaningful signal but still far from the foresight upper bound.

---

## 2.3 Gap

### Current vs Target (using V3 as baseline target):

**No gap can be measured yet for V5** because no full training run has been completed. The V5 unit test (50 batches) is not a valid performance measurement.

### What we can infer from unit test signals:

| Metric | V5 (50 batches) | V3 (full, epoch 12) | V5 direction | Notes |
|--------|-----------------|---------------------|--------------|-------|
| CR | 0.0015 | 0.0147 | 9.8x below target | Expected -- 50 batches vs full training |
| Rank Corr | 0.0892 | 0.0152 | 5.9x ABOVE target | Very encouraging early signal |
| Return Corr | 0.0875 | 0.0132 | 6.6x ABOVE target | Very encouraging early signal |
| P@5% | 0.2656 | 0.3338 | 1.3x below target | Expected -- needs more training |
| Dir Acc | 0.3779 | 0.5050 | 1.3x below target | Expected -- needs calibration |

### Interpretation of the gap pattern:
The V5 architecture shows dramatically stronger correlation signals (rank_corr, return_corr) after only 50 batches compared to V3's final trained values. However, the decision metrics (CR, P@5%, direction accuracy) are still below V3 because:
1. 50 batches is nowhere near convergence
2. The model hasn't learned to threshold predictions for profitability
3. Classification heads need more epochs to calibrate

This pattern (high correlation, low decision quality) is expected early in training and is actually the ideal pattern to see. It means the temporal architecture is extracting genuine signal from the data -- it just needs full training to convert that signal into profitable decisions.

### Is the gap shrinking cycle over cycle?
- V1 -> V2: CR regressed (0.0192 -> 0.0112), gap widened
- V2 -> V3: CR improved (0.0112 -> 0.0147), gap narrowed
- V3 -> V4: CR regressed dramatically (0.0147 -> 0.0031 smoke test), gap widened. V4 was an overparameterized failure.
- V4 -> V5: **Cannot determine yet.** V5 unit test is not comparable. Early correlation signals are the strongest ever seen, but CR requires full training to evaluate.

### At current rate of progress, when would we reach the target?
**Unknown.** We have never completed a full V5 training run. The earliest we can have a valid comparison is after a full training run completes (estimated 2-4 hours on the available GPU based on V4 timing and the smaller V5 model).

---

## 2.4 Constraint Identification

### Primary Constraint: **Execution**

Everything is ready, the full training run just needs to be launched.

### Evidence:

1. **V5 code is complete:** config.py, model.py, data_loader.py, train.py, prepare_tokens.py all present and recently modified (March 27).

2. **Token data is complete:** 64/64 quarters prepared, 11,411,714 total samples (train: 5.8M, val: 2.8M, test: 2.8M). meta.json confirms all splits are ready.

3. **Unit test passed:** V5-Small ran successfully with 679K params, compiled model, 86% GPU utilization. Training pipeline works end-to-end.

4. **GPU is completely idle:** Quadro RTX 6000 at 0% utilization, 53MB/24576MB VRAM. No processes running.

5. **Dev agent is idle:** Has been sitting at the prompt for ~1.5 hours since completing the unit test at 18:57.

6. **No one has sent the command to start full training.** The dev session is waiting for instruction. The GPU is waiting. The data is waiting. The code is waiting. The only missing piece is executing the command.

### Why not the others:

- **Architecture:** Not the constraint. The V5-Small temporal architecture shows the strongest early correlation signals of any version (rank_corr 0.0892, return_corr 0.0875 after 50 batches). The architecture redesign (680K params vs V4's 4.77M) is a reasonable correction. We cannot know if architecture is limiting until we actually train it fully.

- **Training recipe:** Not the constraint yet. We have a valid recipe (lr=0.0003, batch_size=512, patience=15, d_model=128, n_heads=4, intra_layers=2, temporal_layers=1, d_ff=512, backbone_dim=64). It may need tuning, but we cannot know until we see full training results. Premature optimization of the recipe without a baseline full run would be wasted effort.

- **Data quality:** Not the constraint. The token preparation completed successfully. 11.4M samples across 64 quarters is substantial. V3 trained on similar data (similar sample counts in the archive results) and achieved CR=0.0147. Data quality issues would manifest during training -- we need to see training curves first.

- **Data quantity:** Not the constraint. 5.8M training samples is comparable to what V1 (the best performer at CR=0.0192) and V3 used.

- **Infrastructure:** Not the constraint. GPU is available and working. Unit test proved the full pipeline runs. 86% GPU utilization during unit test shows good hardware usage. No disk space issues evident. CUDA 12.4 and driver 535.288.01 are functional.

- **Knowledge gap:** Not the primary constraint. We have clear next steps. The V5 design is documented. The only "knowledge gap" is not knowing V5-Small's full training performance, which can only be resolved by running the training. That makes this an execution problem, not a knowledge problem.

### Has the constraint changed since last cycle?

Yes. The previous constraint (per goal_tracker.md) was **Data -- V5 token prep incomplete (~47/64 quarters)**. That constraint has been fully resolved: all 64/64 quarters are complete. The constraint has shifted from data preparation to execution. The goal tracker is stale and does not reflect this change.

---

## 2.5 Hypothesis

### Primary hypothesis:

**"I believe launching the V5-Small full training run will improve captured return from 0.0015 (50-batch unit test) to >= 0.0147 (V3 baseline) because the V5 temporal architecture already shows 5.9x stronger rank correlation and 6.6x stronger return correlation than V3's final values after only 50 training batches, indicating the architecture has superior signal extraction capability that will translate to higher captured return once the model trains to convergence over the full 5.8M training samples."**

### This can be tested by:
Running the full V5-Small training with current config (lr=0.0003, batch_size=512, patience=15, max epochs uncapped) on the complete token dataset. The training should be launched immediately on the idle GPU by instructing the dev agent.

### Success looks like:
- **Minimum success:** V5-Small test CR >= 0.0147 (matching V3), with return_corr > 0.0132
- **Strong success:** V5-Small test CR >= 0.0192 (matching V1, all-time best), P@5% > 0.33, rank_corr > 0.02
- **Exceptional success:** V5-Small test CR > 0.025, demonstrating the temporal architecture is categorically better
- Training completes without infrastructure failures
- GPU utilization stays >= 80% throughout training

### Failure looks like:
- **Mild failure:** V5-Small CR < 0.0147 but return_corr > 0.02. Interpretation: architecture captures signal but decision heads need recipe tuning. Next step: adjust loss weights, try different learning rate schedules, increase model capacity (V5-Medium).
- **Moderate failure:** V5-Small CR < 0.005 and return_corr < 0.01 at convergence. Interpretation: the promising unit test signals didn't scale to full training. Next step: investigate if the model overfit, check training curves for collapse, consider data augmentation or regularization changes.
- **Severe failure:** Training crashes, NaN losses, or GPU OOM. Interpretation: infrastructure or code issue. Next step: debug, fix, relaunch.
- **Worst case:** V5-Small CR <= V4 levels (~0.003) with negative correlations. Interpretation: the V5 architecture is not an improvement. Next step: return to V3 architecture as baseline, investigate what V3 does right that V4/V5 don't.

### Cost of being wrong:
- **Time:** 2-4 hours of GPU time for a full training run. This is minimal cost.
- **Resources:** No additional data prep needed. No code changes needed. Just GPU hours.
- **Opportunity cost:** Nearly zero -- the GPU is currently sitting completely idle at 0% utilization. Running the training has zero opportunity cost because the alternative is continued idleness.
- **Information value:** Even if the run fails to meet targets, we gain critical information about V5-Small's training dynamics, convergence behavior, and ceiling that cannot be obtained any other way. This makes the run valuable regardless of outcome.

---

## Summary

### Best performance: V1 (archive) epoch 33, CR=0.0192, rank_corr=0.0197, dir_acc=0.5218
### Baseline target: V3 epoch 12, CR=0.0147, P@5%=0.3338, return_corr=0.0132
### Target: CR >= 0.0147 (~45% annualized) with return_corr > 0.0132
### Gap: V5 unit test CR=0.0015 vs target CR=0.0147, but this comparison is invalid -- V5 has only trained 50 batches. Correlation signals (rank 0.089, return 0.088) are 6x stronger than V3's final values, suggesting the architecture has untapped potential.
### Constraint: Execution -- all prerequisites met (code complete, data complete, unit test passed, GPU idle), full training simply has not been launched. Dev agent has been idle 1.5+ hours.
### Evidence: GPU at 0% utilization, dev session waiting at prompt, 64/64 token quarters complete, unit test passed at 18:57, no instruction sent since.
### Hypothesis: "Launching V5-Small full training will achieve CR >= 0.0147 because the temporal architecture shows 6x stronger correlation signals than V3 after only 50 batches, indicating superior signal extraction that will convert to captured return with full training."
### Success criteria: Test CR >= 0.0147, return_corr > 0.0132, training completes without failure
### Failure criteria: Test CR < 0.005 at convergence with return_corr < 0.01 triggers recipe investigation; CR at V4 levels (~0.003) with negative correlations triggers architecture pivot back to V3 baseline
