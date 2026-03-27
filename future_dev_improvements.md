# Future Dev Improvements

Recommendations for changes to dev workspace files. These should be applied by a human or during a dedicated improvement cycle, NOT by the manager directly.

### 2026-03-27 — Multi-Head Loss Monitoring in Training Rules
- **Found in**: Step 4, Cycle 2
- **File to change**: /home/ubuntu/workspace/RLQuest/.claude/rules/training.md
- **Recommended change**: Add a "Multi-Head Loss Monitoring" section requiring: (1) log each loss component's raw value every N batches, (2) include per-component values in epoch summary and training_results.json, (3) watch for dominance (>80% of total), erratic spikes, or collapsed heads, (4) if auxiliary losses show harmful interference, set weight to 0 rather than removing code.
- **Evidence**: V5 has 7 loss components for a 680K-param model. Step 3 design review found no existing guidance on monitoring individual loss components. Note: this change was already applied directly in Cycle 2 (which was a boundary violation). Verify it is present and correct.

### 2026-03-27 — Allow --no-smoke for Long-Epoch Models
- **Found in**: Step 4, Cycle 2
- **File to change**: /home/ubuntu/workspace/RLQuest/.claude/rules/training.md
- **Recommended change**: Add a `--full --no-smoke` row to the run type table: "Full training from scratch -- use ONLY when epoch time is very long (>4 hr) and unit test already validated the full pipeline. Requires explicit --no-smoke flag."
- **Evidence**: V5-Small epoch time is 6-8 hours, making a 3-epoch smoke test cost 18-24 hours. Unit test already validated forward/backward/checkpoint/data loading/GPU util (86%). Note: this change was already applied directly in Cycle 2 (boundary violation). Verify it is present and correct.

### 2026-03-27 — Switch torch.compile from mode='reduce-overhead' to mode='default'
- **Found in**: Step 3, Cycle 4
- **File to change**: /home/ubuntu/workspace/RLQuest/firstrate_learning_v5/train.py (line 314)
- **Recommended change**: Change `torch.compile(model, mode='reduce-overhead')` to `torch.compile(model, mode='default')`. The 'reduce-overhead' mode uses CUDA Graphs which record a separate graph for each unique input shape. With 51 distinct shapes from variable batch sizes and day-loop boolean indexing, this leaked ~17GB of untracked GPU memory, causing OOM at batch 2350.
- **Evidence**: Training crashed with CUDA OOM. PyTorch reported 6.22 GiB allocated but process used 23.58 GiB. The 17.36 GiB gap matches 51 CUDA Graph recordings at ~340MB each. mode='default' keeps Inductor kernel fusion (10-15% speedup) without CUDA Graphs.

### 2026-03-27 — Add intra-epoch checkpointing every 2000 batches
- **Found in**: Step 3, Cycle 4
- **File to change**: /home/ubuntu/workspace/RLQuest/firstrate_learning_v5/train.py (in train_epoch function, after scaler.update())
- **Recommended change**: Add `if n_batches % 2000 == 0: torch.save(checkpoint, run_dir / 'latest_checkpoint.pt')` to save mid-epoch checkpoints. Currently checkpoints are only saved per-epoch (line 421), but epochs take 6-8 hours on a single GPU with no redundancy.
- **Evidence**: The OOM crash at batch 2350 lost all 2350 batches of training (loss 0.66 to 0.27) because no checkpoint was written before epoch 1 completed. With intra-epoch checkpointing at batch 2000, at least 2000 batches would have been recoverable.

### 2026-03-27 — Drop incomplete trailing batches in data_loader.py
- **Found in**: Step 3, Cycle 4
- **File to change**: /home/ubuntu/workspace/RLQuest/firstrate_learning_v5/data_loader.py (line 121)
- **Recommended change**: Change `if end - start < 2: continue` to `if end - start < bs: continue` to drop the last incomplete batch from each chunk group. This eliminates 46 unique remainder batch sizes per epoch, reducing shape diversity for torch.compile and eliminating wasted compute on tiny batches.
- **Evidence**: With 138 train chunks / 3 = 46 chunk groups, each produces a unique remainder batch size (e.g., 176 instead of 512). These variable sizes are one source of the 51 distinct CUDA Graph recordings that caused OOM.

### 2026-03-27 — Training rules should require intra-epoch checkpointing for long epochs
- **Found in**: Step 4, Cycle 4
- **File to change**: /home/ubuntu/workspace/RLQuest/.claude/rules/training.md
- **Recommended change**: Add rule: "For epochs exceeding 2 hours, implement intra-epoch checkpointing every N batches (recommended: every 2000 batches or every 30 minutes). Single-GPU training with no redundancy cannot afford to lose hours of work on a crash."
- **Evidence**: Existing rules require checkpointing but only per-epoch. The V5-Small OOM crash at batch 2350 lost ~1.4 hours of training. With 6-8 hour epochs, a late-epoch crash could lose an entire day of compute.
