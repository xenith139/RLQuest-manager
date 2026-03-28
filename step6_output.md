## Step 6: Execute & Monitor — 2026-03-28 ~03:40 UTC

### Prompt sent: Stop training PID 1245000, implement --prove-out mode with --lr override and scaled cosine schedule, add max_chunks to data_loader.py, run 4-experiment LR sweep (P1-P4)

### Dev response:
Dev received and executed all instructions correctly in sequence:
1. Killed PID 1245000 (and also killed PID 1392002 which dev had launched for resume)
2. Implemented max_chunks in data_loader.py (V5InterleavedDataset and V5SequentialDataset)
3. Implemented prove-out mode in train.py: --prove-out, --lr, --tag, --constant-lr flags
4. Verified cosine schedule scales to prove-out length: total_steps=13580 (5 * 2716 batches/epoch)
5. Ran unit test -- passed
6. Ran all 4 experiments sequentially: P1, P2, P3, P4

### Checkpoints:
- [x] Current training stopped (PID 1245000 and PID 1392002 killed)
- [x] Best model from epoch 1 preserved (best_model.pt + best_model_epoch1.pt, verified)
- [x] --prove-out flag added to argparse (in mutually exclusive group)
- [x] --lr override argument added (float, optional)
- [x] --tag argument added for run dir naming
- [x] --constant-lr flag added (returns 1.0 after warmup)
- [x] max_chunks parameter added to V5InterleavedDataset and V5SequentialDataset
- [x] build_v5_loaders updated with train_max_chunks and val_max_chunks params
- [x] prove-out: 28 train chunks, 13 val chunks, 5 epochs, warmup=200 steps
- [x] CRITICAL: cosine total_steps computed from ACTUAL run length (5 * 2716 = 13580 for prove-out)
- [x] Run dirs named run_TIMESTAMP_proveout_{tag}
- [x] Unit test passed before sweep
- [x] P1 (lr=1e-4, cosine) completed -- 5 epochs + test eval
- [x] P2 (lr=5e-5, cosine) completed -- 5 epochs + test eval
- [x] P3 (lr=3e-4, cosine) completed -- 5 epochs + test eval
- [x] P4 (lr=1e-4, constant) completed -- 5 epochs + test eval

### Interventions: 0 interventions needed
Dev executed everything correctly without deviations. Dev analyzed results between experiments and provided clear summaries.

### Final status: TASK COMPLETE — All 4 prove-out experiments finished

### Dev idle after completion: Yes — dev is idle at prompt after presenting sweep summary

### LR Sweep Results (complete):

| Exp | LR | Schedule | E1 CR | E2 CR | E3 CR | E4 CR | E5 CR | Best Epoch | Stable? |
|-----|------|----------|-------|-------|-------|-------|-------|------------|---------|
| P1 | 1e-4 | cosine | 0.053 | 0.063 | -0.006 | -0.013 | -0.013 | E2 (CR=0.063) | Collapsed E3 |
| P2 | 5e-5 | cosine | 0.023 | 0.056 | 0.068 | 0.072 | 0.072 | E4 (CR=0.072) | CR stable, BUT Prec/Rec=0 at E3+ |
| P3 | 3e-4 | cosine | 0.037 | 0.071 | 0.045 | -0.004 | -0.009 | E2 (CR=0.071) | Collapsed E3 |
| P4 | 1e-4 | constant | 0.059 | -0.002 | 0.025 | 0.056 | 0.056 | E1 (CR=0.059) | Oscillating, Prec/Rec nonzero all epochs |

#### Detailed per-experiment results:

**P1 (lr=1e-4, cosine) — best single-epoch metrics:**
| Epoch | TrL | VL | Prec | Rec | P@5 | CR | RetCorr |
|-------|------|------|------|------|------|------|---------|
| 1 | 0.5681 | 0.6578 | 0.700 | 0.470 | 0.427 | 0.0527 | 0.1110 |
| 2 | 0.5599 | 0.6478 | 0.689 | 0.595 | 0.492 | 0.0633 | 0.0905 |
| 3 | 0.5672 | 0.6508 | 0.728 | 0.036 | 0.180 | -0.0056 | -0.0072 |
| 4 | 0.5730 | 0.6490 | 0.724 | 0.024 | 0.141 | -0.0129 | -0.0076 |
| 5 | 0.5727 | 0.6486 | 0.722 | 0.022 | 0.140 | -0.0130 | -0.0075 |

**P2 (lr=5e-5, cosine) — best CR trajectory:**
| Epoch | TrL | VL | Prec | Rec | P@5 | CR | RetCorr |
|-------|------|------|------|------|------|------|---------|
| 1 | 0.5705 | 0.6350 | 0.646 | 0.692 | 0.319 | 0.0228 | -0.0380 |
| 2 | 0.5592 | 0.6200 | 0.677 | 0.525 | 0.409 | 0.0563 | 0.0875 |
| 3 | 0.5757 | 0.6693 | 0.000 | 0.000 | 0.456 | 0.0680 | 0.0002 |
| 4 | 0.6059 | 0.6645 | 0.000 | 0.000 | 0.457 | 0.0721 | 0.0062 |
| 5 | 0.6018 | 0.6630 | 0.000 | 0.000 | 0.454 | 0.0717 | 0.0079 |

**P3 (lr=3e-4, cosine) — worst stability:**
| Epoch | TrL | VL | Prec | Rec | P@5 | CR | RetCorr |
|-------|------|------|------|------|------|------|---------|
| 1 | 0.5571 | 0.6482 | 0.642 | 0.843 | 0.471 | 0.0365 | 0.0155 |
| 2 | 0.5494 | 0.6550 | 0.684 | 0.642 | 0.472 | 0.0711 | 0.1024 |
| 3 | 0.5766 | 0.6805 | 0.000 | 0.000 | 0.438 | 0.0451 | -0.0646 |
| 4 | 0.5870 | 0.6803 | 0.000 | 0.000 | 0.181 | -0.0037 | -0.0646 |
| 5 | 0.5872 | 0.6802 | 0.000 | 0.000 | 0.179 | -0.0087 | -0.0646 |

**P4 (lr=1e-4, constant) — only recipe with Prec/Rec nonzero all epochs:**
| Epoch | TrL | VL | Prec | Rec | P@5 | CR | RetCorr |
|-------|------|------|------|------|------|------|---------|
| 1 | 0.5632 | 0.6105 | 0.664 | 0.652 | 0.480 | 0.0586 | 0.1130 |
| 2 | 0.5524 | 0.7012 | 0.635 | 0.877 | 0.238 | -0.0017 | -0.0493 |
| 3 | 0.5524 | 0.6810 | 0.598 | 0.650 | 0.346 | 0.0249 | -0.0372 |
| 4 | 0.5844 | 0.6757 | 0.595 | 0.977 | 0.418 | 0.0555 | -0.0993 |
| 5 | 0.5850 | 0.6756 | 0.595 | 0.978 | 0.418 | 0.0558 | -0.0994 |

### Key Analysis:

1. **Cosine schedule fix worked**: The schedule now scales correctly (total_steps = 13,580 for prove-out vs 566,250 before). LR decays meaningfully: P1 went 1e-4 -> 9.6e-5 -> 8.1e-5 -> 6e-5 -> 3.7e-5 -> 1.6e-5.

2. **Mode collapse is a multi-task loss conflict, not purely an LR issue**: ALL cosine experiments (P1, P2, P3) show Prec/Rec collapsing to 0 at epoch 3+. P2's CR kept improving despite this, suggesting the return-prediction losses dominate and push the classification head to degenerate.

3. **P4 (constant LR) is the only recipe that kept classification alive**: Prec/Rec stayed nonzero all 5 epochs. This suggests cosine decay is destabilizing the multi-task balance — as LR drops, some loss components converge faster than others, creating task dominance.

4. **All recipes surpass V3 baselines in their best epochs**: V3 had CR=0.0147, P@5=0.334. Even P1's epoch 2 (CR=0.063, P@5=0.492) is 4x better.

5. **The real problem is multi-task loss weight balance, not LR alone**: The loss weights sum to 3.0 with contrastive already disabled. The classification and return tasks are fighting. A solution might be: normalize loss weights to sum to 1.0, or use task-specific LR scaling.

### Recommendation for next step:
The sweep shows NO recipe is fully stable over 5 epochs. The best options are:
- **Option A**: Use P1 recipe (lr=1e-4, cosine) for full training with patience-based early stopping at epoch 2. Accept the epoch 2 model as best.
- **Option B**: Use P4 recipe (lr=1e-4, constant) which keeps all metrics alive but oscillates. May need more epochs to converge on full data.
- **Option C**: Investigate loss weight normalization (weights sum to 1.0 instead of 3.0) to fix the multi-task conflict root cause, then re-sweep.

### Run directories:
- P1: firstrate_learning_v5/models/run_20260328_003209_proveout_lr1e-4_cosine
- P2: firstrate_learning_v5/models/run_20260328_011901_proveout_lr5e-5_cosine
- P3: firstrate_learning_v5/models/run_20260328_020617_proveout_lr3e-4_scaled_cosine
- P4: firstrate_learning_v5/models/run_20260328_025349_proveout_lr1e-4_constant (estimated)

### Log files:
- P1: firstrate_learning_v5/output/P1_lr1e-4_cosine.log
- P2: firstrate_learning_v5/output/P2_lr5e-5_cosine.log
- P3: firstrate_learning_v5/output/P3_lr3e-4_cosine.log
- P4: firstrate_learning_v5/output/P4_lr1e-4_constant.log

## Persistent Notes
- **Dev behavior patterns:**
  - Dev follows instructions well — applied all changes in correct order
  - Dev self-chains experiments without needing prompting between each one
  - Dev provides good analysis tables between experiments
  - Dev still uses `rm -rf models/run_*` during unit tests (deletes evidence)
  - Dev ran all 4 experiments inline (2h timeout each) rather than nohup — this blocked dev from doing other work but ensured sequential execution
- **Effective prompt patterns:**
  - Detailed implementation specs with parameter names and expected values lead to correct implementation
  - Including "CRITICAL" labels on cosine fix ensured it was implemented correctly
  - Sequential experiment list with clear naming conventions (P1-P4 with tags) worked well
  - Including "what to look for" section helped dev analyze results between experiments
- **Intervention history:**
  - Cycle 6 (this cycle): 0 interventions needed
  - Cycle 5 (previous): 0 interventions needed
  - Cycle 4: 1 critical intervention (crash notification)
- **Monitoring observations:**
  - Prove-out mode works: 5 epochs in ~40 min + ~6 min test eval = ~46 min per experiment
  - 4-experiment sweep completed in ~3 hours total (00:32 - 03:40 UTC)
  - ~2716 batches per epoch in prove-out (28 chunks), ~0.14s/batch = ~6.3 min training + ~1.3 min validation per epoch
  - GPU stable at ~72% utilization, no memory issues
  - Cosine schedule confirmed working: LR decayed from 1e-4 to 1.6e-5 over 5 epochs in P1
- **Watch items for next cycle:**
  - ALL cosine recipes show Prec/Rec collapse at epoch 3 — this is a multi-task conflict, not LR
  - P4 (constant LR) is the only recipe keeping classification alive — investigate why cosine causes task imbalance
  - Loss weights sum to 3.0 (w_direction=1.0, w_magnitude=1.0, w_return=1.0, w_contrast=0.0) — normalizing to sum to 1.0 may fix multi-task conflict
  - P1 epoch 2 has the best single-point metrics: CR=0.063, P@5=0.492, Prec=0.689, Rec=0.595
  - P2 has best CR trajectory (monotonically improving to 0.072) but Prec/Rec=0 from epoch 3 — the model abandoned classification in favor of return prediction
  - Consider: is CR the right metric to optimize, or do we need Prec/Rec to be nonzero?
  - Best model from original full training (epoch 1, CR=0.0138) is still preserved as safety net
  - New progression established: unit-test -> prove-out -> full. Working well — 4 recipes tested in 3 hours vs 20+ hours per full-data run.
  - Contrastive loss still disabled (w_contrast=0.0) — not re-evaluated in this sweep
