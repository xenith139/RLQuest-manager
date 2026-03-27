## Step 6: Execute & Monitor — 2026-03-27 ~22:50 UTC

### Prompt sent: 3 mandatory OOM fixes (torch.compile mode, intra-epoch checkpointing, trailing batch drop) + relaunch training

### Dev response:
Dev received prompt and executed all 3 fixes in correct order:
1. FIX 1 (torch.compile): Changed mode='reduce-overhead' to mode='default' in train.py line 314
2. FIX 2 (intra-epoch checkpointing): Added checkpoint_fn callback every 2000 batches in train_epoch, added batch_offset to _build_checkpoint and resume logic, added start_batch parameter for mid-epoch resume
3. FIX 3 (trailing batches): Changed `< 2` to `< bs` in data_loader.py lines 121 and 150
4. Ran unit test — passed (TrL=0.5739, CR=-0.0294, RetCorr=0.126)
5. Launched full training — PID 1233730

HOWEVER: Training NaN'd at batch 300. Dev self-detected and applied additional fixes:
- Disabled contrastive loss (w_contrast = 0.0) as the most unstable component
- Added NaN guard in loss forward (model.py)
- Added redundant NaN skip in train_epoch (was already present)
- Ran second unit test — passed (Loss=0.5951, CR=0.0224, RetCorr=0.1044)
- Relaunched full training — PID 1245000, log: train_v5_full_20260327_224058.log

### Checkpoints:
- [x] Fix 1: torch.compile mode='default' applied
- [x] Fix 2: Intra-epoch checkpointing every 2000 batches implemented
- [x] Fix 3: Drop trailing batches `< bs` applied (both lines)
- [x] Unit test passed (2 rounds: first to verify 3 fixes, second after NaN fix)
- [x] Full training launched (PID 1245000)
- [x] GPU memory stable: 6819 MiB at batch 400, 1600, AND 2600 — zero growth
- [x] Intra-epoch checkpoint written at batch 2000 (verified: batch_offset=2000, epoch=0, all fields present)
- [x] Training passed batch 2350 (old OOM crash point) without OOM
- [x] Loss healthy and decreasing: 0.6379 -> 0.5807 -> 0.5610 -> 0.5587 (no NaN)
- [ ] Epoch 1 completed (in progress, ~batch 2600 of ~11325 as of 22:50 UTC)
- [ ] Epoch 1 val metrics reported

### Interventions: 0 interventions needed
Dev self-detected the NaN issue after first relaunch and fixed it independently. All 3 mandatory fixes were applied without needing correction.

### Final status: TASK IN PROGRESS — training running, all fixes verified, past all critical milestones

### Dev idle after completion: Yes — dev is idle at prompt. Training running as background process (nohup).

### Training details:
- **PID**: 1245000
- **Log**: firstrate_learning_v5/output/train_v5_full_20260327_224058.log
- **Run dir**: firstrate_learning_v5/models/run_20260327_224100_full
- **Model**: 679,498 params, d=128, intra=2, temporal=1, heads=4
- **Data**: train=5.8M, val=2.8M, test=2.8M samples
- **Memory**: 6.8 GB / 24.6 GB GPU (stable, no growth)
- **Batch rate**: ~7 seconds per 50 batches (~0.14s/batch)
- **Epoch estimate**: ~11325 batches * 0.14s = ~26 min per epoch (much faster than previous run due to mode='default' avoiding CUDA Graph overhead)
- **Next checkpoint**: batch 4000 (~22:62 UTC)
- **Epoch 1 ETA**: ~23:10-23:20 UTC

### Additional fixes applied by dev (beyond the 3 mandatory):
- Disabled contrastive loss (w_contrast=0.0) to fix NaN divergence at batch 300
- Added NaN guard in loss forward (model.py)
- Note: contrastive loss was causing NaN with FP16+mode='default'; this may need investigation before re-enabling

## Persistent Notes
- **Dev behavior patterns:**
  - Dev follows instructions well — applied all 3 fixes in correct order, ran unit test before launch
  - Dev self-detects and fixes new issues (NaN divergence) without needing intervention
  - Dev uses informative verification tables (good for checking compliance)
  - Dev still uses `rm -rf models/run_*` during unit tests (deletes evidence of previous runs)
  - Dev correctly identified contrastive loss as the NaN source and disabled it
- **Effective prompt patterns:**
  - Detailed fix prompts with exact line numbers and before/after code work perfectly
  - Including "CRITICAL" labels and reasons for each fix ensures dev doesn't skip steps
  - Verification checklists in prompts drive dev to verify after launch
  - Providing 3 numbered fixes with clear labels led to clean sequential execution
- **Intervention history:**
  - Cycle 5 (this cycle): 0 interventions needed — dev was fully self-sufficient
  - Cycle 4 (previous): 1 critical intervention — had to interrupt dev's 2-hour sleep to report crash
- **Monitoring observations:**
  - torch.compile mode='default' is MUCH faster than mode='reduce-overhead': ~0.14s/batch vs ~0.4s/batch
  - GPU memory is rock solid at 6819 MiB with mode='default' (was growing to 23.6 GB with reduce-overhead)
  - Intra-epoch checkpoint at batch 2000 saved successfully (8.2 MB, contains batch_offset field)
  - NaN divergence at batch 300 was caused by contrastive loss + FP16 + mode='default'; disabling w_contrast fixed it
  - Loss trajectory with contrastive disabled: 0.64 -> 0.56 in 2600 batches (slower decline than previous run)
- **Watch items for next cycle:**
  - Epoch 1 should complete around 23:10-23:20 UTC — check val CR against V3 baseline (0.0147) and V1 best (0.0192)
  - Next intra-epoch checkpoint at batch 4000 — verify it's updated
  - Contrastive loss is disabled — this may hurt final metrics. Consider re-enabling with lower weight (0.05) if base training is stable
  - Monitor for loss plateau — without contrastive loss, model may converge to different local minimum
  - IMPORTANT: If dev starts another long sleep for monitoring, intervene to use shorter polling intervals
  - After epoch 1: check if checkpoint has batch_offset=0 (end-of-epoch save)
