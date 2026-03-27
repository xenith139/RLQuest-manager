## Step 5: Action Plan — 2026-03-27 ~22:45 UTC

### Status: NEW PLAN — Fix OOM crash, add intra-epoch checkpointing, relaunch training

### All possible actions:
1. **Fix torch.compile OOM** — change mode='reduce-overhead' to mode='default' in train.py line 314 (BLOCKING)
2. **Add intra-epoch checkpointing** — save checkpoint every 2000 batches during train_epoch (BLOCKING — previous crash lost 2350 batches)
3. **Drop incomplete trailing batches** — change `< 2` to `< bs` in data_loader.py line 121 (reduces shape diversity)
4. **Relaunch V5-Small full training** — after all 3 fixes applied (PRIMARY ACTION)
5. Fix summary token zero dims — deferred to post-training
6. Fix price token dim 15 bug — deferred to post-training
7. Fix config.d_ff unused bug — deferred
8. Optimize IntraDayEncoder day-loop — deferred to post-training
9. Design V5-Medium — premature without V5-Small results
10. Clean up zombie processes — cosmetic, low priority

### Resources: GPU=IDLE (0%, idle since crash ~22:02 UTC), CPU=IDLE, Dev=IDLE (at prompt)

### Chosen action(s):

**Actions #1, #2, #3, then #4: Apply all 3 fixes, then relaunch full training.**

**Reasoning:** The constraint is Infrastructure (CUDA OOM). All 3 fixes are mandatory before relaunch per orchestrator directive. The previous crash lost 2350 batches (~1.4 hours) because there was no intra-epoch checkpoint. Fix 2 (intra-epoch checkpointing) is non-negotiable — it prevents catastrophic loss of training progress if any future crash occurs during the 6-8 hour epochs.

**Fix details:**

**FIX 1 — torch.compile mode (train.py line 314):**
- Current: `model = torch.compile(model, mode='reduce-overhead')`
- Change to: `model = torch.compile(model, mode='default')`
- Why: `mode='reduce-overhead'` uses CUDA Graphs which recorded 51 distinct shapes, leaking ~17GB. `mode='default'` keeps kernel fusion (10-15% speedup) without CUDA Graphs.

**FIX 2 — Intra-epoch checkpointing (train.py, train_epoch function + training loop):**
- The `train_epoch` function currently returns only after processing all batches. It has no access to checkpoint-building dependencies (optimizer state, scheduler state, etc.).
- Approach: Add a checkpoint callback or pass checkpoint dependencies into train_epoch. Every 2000 batches, save `latest_checkpoint.pt` with a `batch_offset` field so resume knows where in the epoch to continue.
- Minimal approach: Move the training loop inline in the main training function (lines 394-397) rather than calling train_epoch, OR pass a save callback to train_epoch.
- The checkpoint must include: epoch, batch_offset (n_batches), model_state_dict, optimizer_state_dict, scaler_state_dict, scheduler_state_dict, best_val_score, patience_counter, history, n_params, compiled.

**FIX 3 — Drop trailing batches (data_loader.py line 121):**
- Current: `if end - start < 2: continue`
- Change to: `if end - start < bs: continue`
- Why: Eliminates 46 unique remainder batch sizes, reducing shape diversity for torch.compile.

### Validation gate:

**Training prerequisites:**
- [x] Unit test passed (CR=0.0015, rank_corr=0.089, return_corr=0.088)
- [x] GPU util >70% during unit test (86%)
- [x] AMP+FP16 enabled
- [x] torch.compile enabled (will be mode='default' after fix)
- [MUST VERIFY] Checkpoint/resume working with intra-epoch checkpoint fields
- [x] Per-run directory (timestamped)
- [x] Design doc complete
- [x] Data complete (64/64 quarters, 11.4M samples)

**Fix-specific validation:**
- [MUST VERIFY] mode='default' compiles without error
- [MUST VERIFY] Intra-epoch checkpoint saves at batch 2000 (check file exists before epoch ends)
- [MUST VERIFY] GPU memory stays flat (not growing) after first ~500 batches
- [MUST VERIFY] Trailing batch drop works (no batches < 512 in logs)

**All actions:**
- [x] Chained to follow-up (monitor epoch 1 completion, check val CR against baselines)
- [x] Success criteria defined
- [x] Failure criteria defined

### Parallel tracks:
| Resource | During Fixes | During Training | After Epoch 1 |
|----------|-------------|----------------|----------------|
| GPU | Idle (code changes only) | V5-Small full training | Continue training |
| CPU | Idle | Data loading | Data loading |
| Dev | Apply 3 fixes | Monitor first 500 batches for OOM, verify checkpoint at batch 2000 | Report val metrics |

### Success criteria:
- **Fix validation:** Training runs past batch 2500 without OOM (past the previous crash point)
- **Checkpoint validation:** `latest_checkpoint.pt` exists and is updated at batch 2000 (before epoch ends)
- **Memory validation:** GPU memory usage is stable (not growing) between batch 1000 and batch 5000
- **Training minimum:** Test CR >= 0.0147, return_corr > 0.0132 (match V3)
- **Training strong:** Test CR >= 0.0192, P@5% > 0.33 (match V1)

### Failure criteria:
- **OOM recurs with mode='default':** Fall back to disabling torch.compile entirely (`compiled=False`)
- **Intra-epoch checkpoint corrupt or missing:** Fix checkpoint logic before continuing
- **Loss diverges or NaN:** Check BCE-with-logits fix still present, check gradient clipping
- **Training completes but CR < 0.005:** Recipe issue, not infra — investigate loss component weights

---

## Recommended Prompt

**IMPORTANT: All 3 fixes MUST be applied BEFORE launching training. The previous crash lost 2350 batches because there was no intra-epoch checkpoint. Do NOT relaunch without intra-epoch checkpointing.**

```
The V5-Small full training CRASHED at batch 2350/11325 with CUDA OOM. torch.compile mode='reduce-overhead' leaked ~17GB via CUDA Graphs. No checkpoint survived — training must restart from scratch after applying these 3 mandatory fixes.

Apply ALL 3 fixes before relaunching:

**FIX 1 — torch.compile mode (train.py line 314):**
Change: `model = torch.compile(model, mode='reduce-overhead')`
To:     `model = torch.compile(model, mode='default')`
Reason: mode='default' keeps kernel fusion without CUDA Graphs. Eliminates the OOM.

**FIX 2 — Intra-epoch checkpointing (train.py, CRITICAL):**
Add checkpoint saves every 2000 batches during training. The previous crash lost 2350 batches (~1.4 hours) because checkpoints were only saved per-epoch. With 11,325 batches per epoch (6-8 hours), intra-epoch checkpointing is essential.

Implementation: In train_epoch (or by restructuring the training loop), every 2000 batches save a checkpoint to `run_dir / 'latest_checkpoint.pt'` containing all 9 fields: epoch, model_state_dict, optimizer_state_dict, scaler_state_dict, scheduler_state_dict, best_val_score, patience_counter, history, n_params, compiled. Include a `batch_offset` field so training can resume mid-epoch.

The checkpoint resume logic (lines 348-369) should also handle the batch_offset field — when resuming mid-epoch, skip batches up to batch_offset.

**FIX 3 — Drop incomplete trailing batches (data_loader.py line 121):**
Change: `if end - start < 2:`
To:     `if end - start < bs:`
Reason: Eliminates 46 unique remainder batch sizes that contribute to shape diversity.

After applying all 3 fixes, relaunch full training:
```
cd /home/ubuntu/workspace/RLQuest && python -m firstrate_learning_v5.train --full --no-smoke
```

**Verification checklist after launch:**
1. Confirm mode='default' in the training log (no "51 distinct CUDA graph sizes" warning)
2. Watch GPU memory at batch 500 and batch 1500 — must be stable, not growing
3. Verify `latest_checkpoint.pt` exists in the run directory at batch 2000 (before epoch ends)
4. Confirm training passes batch 2350 (the previous crash point) without OOM
5. After epoch 1 (~7-9h): report val CR, rank_corr, return_corr, per-component losses
6. Compare against V3 baseline (CR=0.0147) and V1 best (CR=0.0192)
7. Update train_progress.md with results
```

---

## Persistent Notes
- **Actions taken history:**
  - Cycle 1 (~20:35 UTC): Wrote "launch training" prompt. NOT DELIVERED to dev.
  - Cycle 2 (~22:00 UTC): Wrote improved "launch training" prompt. NOT DELIVERED to dev.
  - Cycle 3 (~22:30 UTC): Identified delivery bottleneck. Training WAS launched (by dev independently or by tmux delivery).
  - Cycle 4 (~22:35 UTC): Training crashed at batch 2350. Analyzed OOM root cause. Updated goal tracker. Identified 3 mandatory fixes.
  - Cycle 5 (~22:45 UTC): THIS CYCLE. Planning fix-and-relaunch. All 3 fixes mandatory before relaunch.
- **Pending items:**
  - Fix summary token dims 14-19 (after training completes)
  - Fix price token dim 15 (after training completes)
  - Optimize IntraDayEncoder day-loop (after training completes)
  - Fix config.d_ff unused bug (maintenance)
  - Clean up 5 zombie processes (cosmetic)
- **Validation patterns:**
  - torch.compile mode='reduce-overhead' is dangerous with variable-shape inputs — NEVER use again without fixed input shapes
  - Per-epoch-only checkpointing is inadequate for 6-8 hour epochs — always add intra-epoch saves
  - Previous validation gates missed infrastructure resilience (OOM, checkpoint frequency) — now explicitly checking
- **Effective prompts:**
  - Detailed fix prompts with exact line numbers and before/after code work well
  - Verification checklists ensure dev confirms fixes before moving on
  - Including the "reason" for each fix helps dev understand priority and not skip steps
- **Watch items for next cycle:**
  - Have all 3 fixes been applied? Check train.py for mode='default' and intra-epoch checkpoint logic, data_loader.py for `< bs`
  - Has training been relaunched? Check for new run dirs in models/
  - If training running: GPU memory trend MUST stay flat (key OOM-fix confirmation)
  - If training running: verify intra-epoch checkpoint appears at batch 2000
  - If training running: loss should start ~0.66 and follow similar trajectory to pre-crash run
  - BCE-with-logits fix still present?
  - Epoch wall-clock time with mode='default' (expect ~7-9.5h vs ~6.25h with reduce-overhead)
