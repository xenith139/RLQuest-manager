## Step 2: Gap & Constraint — 2026-03-28 03:50 UTC

### Constraint SHIFTED: Training Recipe (mode collapse / LR) -> Execution (implement step-based training + relaunch with stable recipe)

Architecture is VALIDATED. All 4 prove-outs beat V3 baseline. The training recipe constraint is partially resolved: we know constant LR works (P4 stable through epoch 5), and we know the root cause of collapse (multi-task loss weight conflict under cosine decay). The bottleneck is now **execution**: implementing the remaining step-based training features and relaunching full training with the winning recipe.

### Best performance: P2 (lr=5e-5, cosine) CR=0.0721 val / 0.0185 test — but UNSTABLE (prec/rec=0). P4 (lr=1e-4, constant) CR=0.0586 val / 0.0176 test — STABLE, prec/rec maintained.
### Target: CR >= 0.0147 with return_corr > 0.0132 (V3 baseline)
### Gap: ALREADY EXCEEDED on 20% data. P4 test CR=0.0176, which is 1.20x V3 baseline. Full-data training should improve further. Gap is CLOSED on prove-out; need to confirm on full training.
### Gap trend: MAJOR IMPROVEMENT. Was 0.94x last cycle (epoch 1 only). Now 1.20x on prove-out test data. Architecture validated — V5 beats V3 even on 20% data with stable recipe.
### Constraint: Execution — implement step-based training features and relaunch full training
### Constraint changed from last cycle: YES. Was Training Recipe (LR/scheduler). Now Execution. Recipe is known (constant LR, 1e-4). Two features remain unimplemented.
### Hypothesis: "Implementing subset validation every N steps and step-based early stopping, then launching full training with constant lr=1e-4, will produce a stable model that exceeds V3 CR=0.0147 on full test data, because (a) P4 already beats V3 on 20% data with stable prec/rec, and (b) step-based eval will catch any divergence within ~2 hours instead of waiting 22 hours per epoch."
### Success criteria: Full training with constant lr=1e-4 achieves test CR > 0.0176 (P4 prove-out baseline) with prec > 0.5 and rec > 0.5, and subset validation detects trends within 5000 steps.
### Failure criteria: Full training with constant LR shows different dynamics than prove-out (collapse reappears at full-data scale), or step-based features introduce bugs that corrupt training.

### What Needs to Happen (prioritized):

**1. Implement subset validation every N steps (NOT DONE)**
- From step_based_training.md: run 10% of val data every 5000 steps
- Gives mid-epoch signal on val loss trend — critical for detecting collapse early
- On full data (11,325 batches/epoch), this means ~2 eval checkpoints per epoch before full eval
- Each subset eval on 10% val data: ~6 chunks of 63 = ~270K samples, should take ~2-3 min

**2. Implement step-based early stopping (NOT DONE)**
- Convert patience from epochs to evaluation cycles
- If using eval every 5000 steps: patience=15 epochs becomes patience=15 eval cycles = 75,000 steps (~6.6 epochs)
- More responsive: catches plateau within hours, not days

**3. Relaunch full training with P4 recipe (constant lr=1e-4)**
- Already proven stable on 20% data through 5 epochs
- Expected: better CR on full data due to more gradient steps per epoch
- Step-based features provide safety net for early detection of problems

**4. Consider loss weight tuning as follow-up (OPTIONAL, FUTURE)**
- Root cause of cosine collapse: multi-task loss weight conflict
- Per-head LR scaling or gradient normalization could unlock cosine schedule
- But this is optimization beyond the immediate goal — constant LR already works

---

## Persistent Notes
- Previous constraint: Training Recipe (lr=3e-4 causes mode collapse; cosine decay causes multi-task loss conflict) — RESOLVED via P4 constant LR recipe
- Current constraint: Execution (implement remaining step-based features, relaunch full training)
- Previous gap: 0.94x of target (V5 epoch 1 CR=0.0138 vs V3 0.0147)
- Current gap: EXCEEDED on prove-out. P4 test CR=0.0176 = 1.20x V3 baseline. Need full-data confirmation.
- Hypothesis history:
  - Cycle 1-3: "Launch full training, expect CR >= 0.0147" (blocked by execution)
  - Cycle 4: "Fix CUDA Graph OOM and relaunch" (CONFIRMED — OOM fixed)
  - Cycle 5: "Implement prove-out mode for rapid LR/scheduler iteration" (CONFIRMED — all 4 prove-outs complete)
  - Cycle 6 (current): "Implement step-based training + relaunch full training with constant lr=1e-4"
- Key evidence accumulated:
  - Architecture VALIDATED: all 4 prove-outs beat V3 on test CR (0.0141-0.0185 vs 0.0147)
  - P4 (constant lr=1e-4): ONLY stable recipe. Prec=0.595, Rec=0.978 at epoch 5. Test CR=0.0176.
  - Root cause identified: multi-task loss weight conflict under cosine decay (not LR magnitude)
  - Cosine decay universally kills prec/rec by epoch 3 regardless of LR
  - P2 paradox: CR=0.0721 with prec/rec=0 means ranking head excellent but classification head dead
  - V3 baseline: CR=0.0147, rank_corr=0.0152, return_corr=0.0132 (epoch 12)
  - V1 best ever: CR=0.0192 (epoch 33)
  - Full-data epoch 1 (lr=3e-4): CR=0.0138, strong but collapsed by epoch 2
  - Step-based training: checkpointing (2000 steps) and resume DONE. Subset val and step-based early stopping NOT DONE.
  - Step-based training doc: `/home/ubuntu/workspace/RLQuest-manager/manual_docs/step_based_training.md`
  - GPU: Quadro RTX 6000, 24GB. Full epoch: ~22 hours at full data scale (per doc), ~75 min at 20% scale.
  - Zombie processes: 10 (cosmetic, accumulating)
- Watch items for next cycle:
  - Has dev implemented subset validation every N steps?
  - Has dev implemented step-based early stopping?
  - Has full training been relaunched with P4 recipe (constant lr=1e-4)?
  - If training running: check step-level val loss trend for stability
  - Monitor prec/rec through first 5000 steps — confirm no collapse with constant LR at full scale
  - Goal tracker still stale — needs update with prove-out results and new priorities
