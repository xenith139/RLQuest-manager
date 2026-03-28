## Step 4: Learn & Update — 2026-03-28 04:00 UTC (Cycle 6)

### Last action: "Stop training, implement prove-out mode, fix cosine schedule, run 4-experiment LR sweep" (Cycle 5)
### Result: SUCCESS — All 4 prove-out experiments completed. Architecture validated (all beat V3). Winning recipe identified: P4 (lr=1e-4, constant LR), test CR=0.0176 (1.2x V3), stable prec/rec through 5 epochs. Dev implemented prove-out mode, --lr override, --constant-lr, max_chunks in data_loader, and ran all 4 experiments sequentially.
### Process gaps:
1. **return_mae=NaN not flagged earlier:** ALL prove-outs show NaN return_mae and 0.0 return_corr. The return prediction head (loss weight=0.8, second highest) is broken and consuming gradient budget without producing useful signal. This was noted but not escalated — a broken head using 27% of gradient budget could be degrading training quality.
2. **Step-based eval features not implemented before prove-out sweep:** The step_based_training.md spec was identified but only 3 of 5 items were already implemented. The remaining 2 (subset eval, step-based early stopping) were not prioritized alongside prove-out implementation. They should have been built concurrently since they share the same train.py file.
3. **No process gap in prove-out execution:** The sweep was well-designed, all 4 experiments ran cleanly, and the results were unambiguous. The OODA process worked well this cycle.
### Goal tracker updated: YES — major changes:
- Status: updated from mode collapse / prove-out pending to prove-out COMPLETE, full training pending
- Hypothesis: updated to step-based features + full training with constant lr=1e-4
- Constraint: changed from Training Recipe to Execution
- Belief state: Architecture raised to VERY HIGH, Recipe raised to HIGH (was LOW)
- Metrics: added P1-P4 prove-out rows, relabeled epoch 1 row
- Known issues: marked items 9-11 as RESOLVED, added 3 new items (return_mae NaN, subset eval, step-based early stopping)
- Priority queue: marked items 1-4 as DONE, added items 5-6 (step-based features), renumbered
### Research docs updated: NO — step3_output.md persistent notes already contain full prove-out analysis.
### Dev improvements logged: YES, 2 new items (return_mae NaN investigation, subset eval + step-based early stopping implementation)
### Manager improvements logged: YES, 1 new item (Step 3 should cross-check all loss heads for NaN/degenerate output)

## Persistent Notes
- Cycle count: 6
- Pattern log:
  - **NEW PATTERN: Broken loss heads silently waste gradient budget.** return_mae=NaN means the return head (weight=0.8) produces no useful gradient but still consumes 27% of the effective gradient budget. Multi-task models should validate that all loss heads produce meaningful gradients, not just that the primary metric improves.
  - **CONTINUING PATTERN: Cosine schedule interacts poorly with multi-task loss weights.** Now confirmed across 3 experiments (P1-P3) at different LRs. The root cause is differential sensitivity of classification vs ranking heads to LR decay, not LR magnitude. Constant LR is the workaround; proper fix is per-head LR or gradient normalization.
  - **RESOLVED PATTERN: Goal tracker lag.** Goal tracker was stale for 2 cycles. Now updated comprehensively with prove-out results, new priorities, resolved items. Must keep current each cycle.
  - **CONTINUING PATTERN: Feature implementation can be parallelized.** Step-based eval features could have been implemented alongside prove-out mode. When multiple code changes target the same file, batch them.
- Process effectiveness:
  - Step 1 (Observe): EXCELLENT. Comprehensive state assessment with all 4 prove-out results tabulated. Correctly identified user's step-based training request. Flagged zombie accumulation.
  - Step 2 (Orient): EXCELLENT. Correctly shifted constraint from Training Recipe to Execution. Computed gap closure (1.2x V3). Identified the 2 missing features with time estimates.
  - Step 3 (Design Review): EXCELLENT. Detailed implementation design for both missing features (~40 lines). Validated constant LR recipe for full training. Computed iteration speed improvement (5.5x faster feedback). Correctly flagged return_mae=NaN as non-blocking but noteworthy.
  - Step 4 (Learn): Working well. Catching NaN gradient waste pattern. Goal tracker now current.
  - Step 5 (Action): Previous cycle's action (prove-out sweep) was executed successfully and completely by dev. Prompt quality was good — dev implemented all requested features and ran all 4 experiments.
- Accumulated learnings:
  1. V5-Small architecture is VALIDATED: all 4 prove-outs beat V3 on test CR (0.0141-0.0185 vs 0.0147)
  2. Winning recipe: lr=1e-4, constant (no cosine). Test CR=0.0176 (1.2x V3). Stable through 5 epochs.
  3. Cosine decay universally kills prec/rec by epoch 3 regardless of LR (tested at 5e-5, 1e-4, 3e-4)
  4. Root cause: multi-task loss weight conflict under cosine decay. Different heads have different LR sensitivity.
  5. P2 paradox: best CR (0.0721) with prec/rec=0 means ranking head excellent but classification head dead
  6. return_mae=NaN across all experiments — return head broken, consuming 27% gradient budget uselessly
  7. Constant LR avoids the multi-task conflict by maintaining gradient scale parity across heads
  8. Prove-out mode (20% data, 5 epochs, ~15 min/epoch) enables rapid recipe validation
  9. Step-based eval (every 2000 steps) would provide 5.5x faster divergence detection vs epoch-based
  10. V3 baseline: CR=0.0147. V1 best: CR=0.0192. P4 on 20% data: CR=0.0176. Target: exceed on full data.
  11. Full training command: `python -m firstrate_learning_v5.train --full --lr 1e-4 --constant-lr --no-smoke`
  12. Full epoch timing: ~72 min at 156 batches/min. Step eval every 2000 steps = signal every ~13 min.
- Watch items for next cycle:
  - Has dev implemented subset eval every 2000 steps? Check train.py for eval_every_steps parameter.
  - Has dev implemented step-based early stopping? Check train.py for step-based patience logic.
  - Has full training been launched with P4 recipe (constant lr=1e-4)?
  - If full training running: check step-eval metrics at first 2000-step checkpoint for stability
  - Monitor prec/rec through first 5000 steps — confirm no collapse at full data scale
  - return_mae=NaN — has it been investigated?
  - Zombie process count (was 10, growing)
  - Goal tracker is now current — verify it stays current next cycle
