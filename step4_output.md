## Step 4: Learn & Update — 2026-03-28 00:20 UTC (Cycle 5)

### Last action: "Apply 3 OOM fixes (compile mode, intra-epoch checkpoint, trailing batch drop) and relaunch full training" (Cycle 4)
### Result: PARTIAL SUCCESS — All 3 fixes were applied and training relaunched successfully. OOM is resolved (GPU stable at 6.8 GiB). Epoch 1 completed with strong results (CR=0.0138, 94% of V3 target). However, mode collapse at epoch 2: train loss ROSE (0.5508->0.5894), val loss spiked 15.7%, precision/recall collapsed to 0.000. Epoch 3 continuing same degradation. The infrastructure fixes worked perfectly; the new blocker is the training recipe (LR too high with flat cosine schedule).
### Process gaps:
1. **Cosine schedule not reviewed for effective decay rate:** Step 3 design reviews (Cycles 1-4) reviewed architecture, loss components, data, and infrastructure but never computed the actual LR at epoch 2. The cosine schedule was spread over 566K steps, making it 99.6% of peak at epoch 2 — effectively flat. This should have been caught by computing LR at key epoch boundaries.
2. **Loss weight sum not factored into effective LR:** The 6 active losses sum to weight 3.0, effectively tripling gradient magnitude. At lr=3e-4, the model operates at ~9e-4 effective LR in single-task terms. This interaction between multi-task weights and LR was not analyzed before launching full training.
3. **V4 pattern not recognized earlier:** V4 used identical lr=3e-4 with cosine over max_epochs and also peaked at epoch 1 then degraded. This pattern match should have triggered a red flag before V5 launched with the same LR.
### Goal tracker updated: YES — significant changes:
- Status: updated from OOM crash to mode collapse / LR overshoot
- Hypothesis: updated to prove-out mode LR sweep
- Constraint: changed from Infrastructure to Training Recipe
- Belief state: Recipe confidence dropped to LOW, Infrastructure raised to HIGH
- Metrics: added V5-Small epoch 1 (CR=0.0138 BEST) and epoch 2 (MODE COLLAPSE) rows
- Known issues: marked OOM/checkpoint/trailing-batch as RESOLVED, added 4 new blocking issues (cosine schedule, loss weights, prove-out mode, confidence feedback loop)
- Priority queue: rewritten for prove-out implementation and LR sweep
### Research docs updated: NO — step3_output.md persistent notes already contain the full technical analysis of LR schedule, loss weights, and prove-out design.
### Dev improvements logged: NO — all relevant items (prove-out mode, cosine schedule fix, iteration speed requirement) were already logged by the orchestrator in the current cycle. No new items discovered.
### Manager improvements logged: NO — iteration speed in design review and updated training progression were already logged by the orchestrator. No new items discovered.

## Persistent Notes
- Cycle count: 5
- Pattern log:
  - **NEW PATTERN: LR schedule interactions with multi-task loss weights are invisible without explicit computation.** The cosine schedule looked correct in code (warmup + cosine decay) but was functionally flat for the relevant epoch range. Loss weight sums amplify this silently. Step 3 design review must compute effective LR at key epoch boundaries and factor in loss weight scaling.
  - **NEW PATTERN: V4 and V5 share the same recipe failure mode.** Both used lr=3e-4 with cosine over max_epochs and both collapsed after epoch 1. This was a missed cross-version pattern match. When launching a new version, compare its recipe parameters against previous failures.
  - **CONTINUING PATTERN: Goal tracker lags behind reality.** Cycle 4's tracker still referenced OOM as the constraint when the real constraint shifted to training recipe. Now updated. Step 4 must aggressively update the tracker.
  - **RESOLVED PATTERN: Infrastructure resilience gaps.** OOM fix, intra-epoch checkpointing, and trailing batch drops all applied and working. Infrastructure is now stable.
- Process effectiveness:
  - Step 1 (Observe): EXCELLENT. Caught mode collapse at epoch 2 with detailed batch-level analysis. Correctly identified rising train loss as the diagnostic signal (not classic overfitting).
  - Step 2 (Orient): EXCELLENT. Correctly shifted constraint from Infrastructure to Training Recipe. Computed the 6% gap to V3 baseline. Proposed prove-out mode with clear cost/benefit analysis.
  - Step 3 (Design Review): EXCELLENT. Computed exact LR at each epoch boundary, proving the cosine schedule is flat. Identified loss weight sum as a gradient magnitude multiplier. Designed prove-out mode with implementation details. Produced a concrete 4-experiment sweep plan.
  - Step 4 (Learn): Working well. Catching process gaps (cosine schedule not reviewed, V4 pattern not matched).
  - Step 5 (Action): Previous cycle's action (OOM fixes + relaunch) was executed successfully by dev. Delivery mechanism working.
- Accumulated learnings:
  1. V5-Small architecture is validated: epoch 1 CR=0.0138, Prec=0.591, Rec=0.881 — 94% of V3 target
  2. The problem is NOT the model, it is the training recipe (LR + schedule)
  3. Cosine schedule must be computed at epoch boundaries, not just inspected visually
  4. Loss weight sum (3.0) effectively triples the learning rate in gradient magnitude
  5. Effective single-task-equivalent LR at epoch 2 was ~9e-4 (3e-4 * 3.0 loss weight sum * 0.996 cosine)
  6. Mode collapse signature: train loss rises (not just val loss), precision/recall collapse to exactly 0.000
  7. P@5 can improve (0.213->0.251) even during mode collapse — ranking signal preserved, calibration destroyed
  8. V4 had the same failure pattern with the same lr=3e-4 — cross-version pattern matching is valuable
  9. Prove-out mode (20% data, 5 epochs) enables ~90 min experiments vs 75 min/epoch for full data
  10. Best model from epoch 1 is safely preserved (best_model.pt and best_model_epoch1.pt)
  11. Infrastructure is now stable: OOM fixed, intra-epoch checkpoints added, GPU memory flat at 6.8 GiB
  12. Epoch wall-clock: ~75 min on full data at 156 batches/min. Prove-out: ~15 min/epoch.
- Watch items for next cycle:
  - Has current (wasteful) training been stopped?
  - Has prove-out mode been implemented? Check for --prove-out flag in train.py argparse
  - Is cosine schedule scaled to actual run length (not 566K steps)?
  - Has LR sweep started? Which experiment (P1-P4) is running or completed?
  - Key signal in prove-out: train loss should DECLINE or stay flat across all 5 epochs. Any rise = recipe still broken.
  - If P1 (lr=1e-4) shows stable training, proceed directly to full run with lr=1e-4 + scaled cosine
  - Epoch 1 best model PRESERVED — this is the safety net regardless of experiments
  - Dev awareness: does dev understand the root cause (flat cosine + loss weight scaling)?
