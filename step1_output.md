# Step 1: Ground Truth & State Assessment

**Timestamp:** 2026-03-27 22:17 UTC

**Delta from last check (21:28 UTC):** MAJOR CHANGE. Full training was launched, ran for ~19 minutes through batch 2350 of epoch 1, then CRASHED with CUDA OOM. GPU is now idle. Two failed training attempts occurred.

---

### Dev: RUNNING -- IDLE at prompt
- Session `claude-1-1774585502` active
- Prompt showing `>` with bypass permissions on
- Last visible output references training PID 1166716 and notes loss dropping from 0.66 to 0.55
- Dev appears unaware training has crashed (no error handling visible in pane)

### CPU: IDLE -- 5 zombie python processes (no live python)
- Zombie PIDs: 102118, 102603, 896622, 1154253, 1166716
- PIDs 1154253 and 1166716 are NEW since last check -- these are from the two training attempts
- PID 1166716 was the full training process that OOM'd

### GPU: COMPLETELY IDLE -- 0% utilization, 53MiB/24576MiB VRAM, 15W
- No processes on GPU
- Temperature 32C (idle)
- All 23.58 GiB that was in use has been freed after crash

### V5 Code: All present (unchanged)
- config.py, model.py, data_loader.py, train.py, prepare_tokens.py

### V5 Data: COMPLETE (unchanged) -- 64/64 quarters, 11,411,714 samples

### V5 Training: CRASHED -- CUDA OOM at epoch 1, batch 2350 of 11,325

**Crash Timeline:**
1. **Run 1 (21:35:31):** Launched, crashed at batch 200 with CUDA assertion error (`input_val >= zero && input_val <= one` in BCE loss). This was the BCE-without-logits bug. Run dir: `run_20260327_213531_full` (NOT in models/ -- was cleaned up or never created properly)
2. **Run 2 (21:40:51):** Unit test re-run to verify BCE-with-logits fix. Completed successfully. Run dir: `run_20260327_214051_unit`
3. **Run 3 (21:42:52):** Full training launched with BCE fix. Ran successfully through 2350 batches (~20.7% of epoch 1) over ~19 minutes. CRASHED at 22:02 UTC.

**OOM Crash Details:**
- Crashed during `scaler.scale(loss).backward()` at batch 2350
- Error: `torch.OutOfMemoryError: CUDA out of memory. Tried to allocate 2.00 MiB`
- At crash: 23.58 GiB of 23.64 GiB used (GPU nearly 100% full)
- PyTorch allocated: 6.22 GiB + 1.26 GiB in CUDA Graphs private pools + 31.80 MiB reserved
- **Critical:** 23.58 GiB total process usage vs 6.22 GiB PyTorch allocation = ~17.36 GiB unaccounted, likely CUDA Graph recordings
- Root cause: 51 distinct CUDAGraph sizes recorded (warned in log). Each new graph recording consumes GPU memory. Over 2350 batches, the accumulation of graph recordings exhausted the 24GB VRAM.

**Loss trajectory before crash (good -- model was learning):**
- Batch 50: 0.6596
- Batch 200: 0.5917
- Batch 500: 0.4082
- Batch 1000: 0.4032
- Batch 1500: 0.3375
- Batch 2000: 0.3174
- Batch 2350: 0.2742 (last recorded)

**Key insight:** The 17.36 GiB gap between total GPU usage (23.58 GiB) and PyTorch tensor allocation (6.22 GiB) strongly implicates CUDA Graphs. The log explicitly warned about 51 distinct sizes being recorded. `torch.compile` with default settings creates CUDAGraphs for each unique input shape, and these are never freed during training.

### Goal tracker discrepancy: SIGNIFICANT
- Goal tracker says "Full training NOT YET LAUNCHED" -- WRONG, it was launched AND crashed
- Goal tracker priority #1 "Launch V5-Small full training" -- STALE, needs to be "Fix OOM and relaunch"
- Goal tracker does not reflect the two training attempts or the BCE fix that was applied

---

## Persistent Notes

**Project structure (stable):**
- RLQuest workspace: `/home/ubuntu/workspace/RLQuest/`
- V5 code: `/home/ubuntu/workspace/RLQuest/firstrate_learning_v5/`
- V5 tokens: `/home/ubuntu/workspace/RLQuest/firstrate_learning_v5/cache/tokens/`
- V5 models: `/home/ubuntu/workspace/RLQuest/firstrate_learning_v5/models/`
- V4 models: `/home/ubuntu/workspace/RLQuest/firstrate_learning_v4/models/`
- Manager workspace: `/home/ubuntu/workspace/RLQuest-manager/`
- Research docs: `/home/ubuntu/workspace/RLQuest-manager/.claude/skills/manager/research/`
- Dev session PID file: `/home/ubuntu/workspace/RLQuest/claude_session_PID_1`
- Dev tmux session name: `claude-1-1774585502`

**Key paths:**
- V5 token meta: `/home/ubuntu/workspace/RLQuest/firstrate_learning_v5/cache/tokens/meta.json`
- V5 train progress: `/home/ubuntu/workspace/RLQuest/firstrate_learning_v5/train_progress.md`
- Training log (crashed): `/home/ubuntu/workspace/RLQuest/firstrate_learning_v5/output/train_v5_full_20260327_214250.log`
- Earlier crash log: `/home/ubuntu/workspace/RLQuest/firstrate_learning_v5/output/train_v5_full_20260327_213530.log`
- Goal tracker: `/home/ubuntu/workspace/RLQuest-manager/goal_tracker.md`

**Hardware:**
- nproc: 20 cores
- GPU: Quadro RTX 6000, 24GB VRAM, CUDA 12.4, Driver 535.288.01
- RAM: 64GB (58GB available)
- infoROM corrupted warning on GPU (cosmetic, not functional)

**V5 data stats (stable):**
- 64/64 quarters, 11,411,714 total samples
- Train: 40 quarters, 5,798,357 samples, 138 chunks (11,325 batches at batch_size=512)
- Val: 12 quarters, 2,769,223 samples, 63 chunks
- Test: 12 quarters, 2,844,134 samples, 63 chunks

**V5 training config (from run_meta.json):**
- d_model=128, n_heads=4, intra_layers=2, temporal_layers=1, d_ff=512
- backbone_dim=64, lr=3e-4, batch_size=512, patience=15, max_epochs=50
- 679,498 params, compiled=true, AMP FP16 enabled

**OOM root cause analysis:**
- CUDA Graphs from torch.compile recorded 51 distinct input shapes
- Each shape requires a separate graph recording that consumes GPU memory
- Over 2350 batches, graph memory grew from ~6 GiB to 23.58 GiB (the entire GPU)
- PyTorch only tracked 6.22 GiB as "allocated" -- the rest was in CUDA Graph private pools
- FIX OPTIONS (in order of preference):
  1. Set `torch._inductor.config.triton.cudagraph_skip_dynamic_graphs=True` -- skip CUDA graphs for variable-size inputs
  2. Pad inputs to fixed sizes to reduce distinct shape count
  3. Disable torch.compile entirely (loses ~10-20% performance)
  4. Set `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True` (helps fragmentation, may not fix root cause)
  5. Reduce batch_size from 512 to 256 (halves base memory, but graphs still accumulate)

**Training was working well before crash:**
- Loss dropped from 0.66 to 0.27 over 2350 batches (steady decline, good learning)
- GPU util 88-90% (healthy)
- ~20 seconds per 50 batches, so full epoch would take ~75 min (much faster than estimated 6-8 hours)
- No signs of divergence or instability in the loss curve

**Zombie processes:** 5 zombies (PIDs 102118, 102603, 896622, 1154253, 1166716)

**Last known state change:** Training launched, ran 2350/11325 batches of epoch 1 with good loss trajectory (0.66->0.27), then CUDA OOM crash at 22:02 UTC due to CUDA Graph memory accumulation.

**Watch items for next cycle:**
- Has OOM fix been applied? Check train.py for cudagraph config changes
- Has training been relaunched? Check for new run dirs
- If training running: monitor GPU memory usage trend (should stay flat, not grow)
- BCE fix was applied between run 1 and run 3 -- verify it persists
- train_progress.md is stale (says "RUNNING" at step 2/52) -- should be updated
