## Step 2: Gap & Constraint — 2026-03-27 22:20 UTC

### Constraint SHIFTED: Execution -> Infrastructure (CUDA OOM crash)

Previous cycle's Execution constraint was resolved — training was launched. But it crashed at batch 2350/11325 (20.7% of epoch 1) due to CUDA OOM from torch.compile CUDA Graph memory accumulation. Constraint has shifted from "just launch it" to "fix the memory leak and relaunch."

### Best performance: V1 (archive) epoch 33 CR=0.0192
### V5 partial run (crashed): Loss 0.66 -> 0.27 over 2350 batches. No CR/eval metrics available (crash before epoch completion). Loss trajectory was excellent — steady decline, no instability.
### Baseline target: V3 epoch 12 CR=0.0147, return_corr=0.0132
### Target: CR >= 0.0147 with return_corr > 0.0132
### Gap: Still unmeasurable — no completed epoch means no validation metrics. However, the strong loss trajectory (0.66->0.27 in 20% of epoch 1) is highly encouraging. Model was clearly learning.
### Gap trend: Directionally positive but formally unchanged — we now have evidence the V5 architecture learns well, but still no CR number to compare against target.
### Constraint: Infrastructure — CUDA Graph memory leak from torch.compile
### Constraint changed from last cycle: YES. Was Execution (training not launched), now Infrastructure (training launched but crashes due to OOM). The Execution constraint was resolved — dev launched training. The new blocker is a torch.compile CUDA Graphs issue that recorded 51 distinct input shapes, leaking ~17GB of untracked GPU memory until OOM at 23.58/24GB.
### Hypothesis: "Disabling CUDA Graphs in torch.compile (via cudagraph_skip_dynamic_graphs=True or compile mode='reduce-overhead' removal) will allow full training to complete, achieving CR >= 0.0147, because the model was learning well (loss 0.27 at crash) and the OOM is purely a memory management issue, not a model or data problem."
### Success criteria: Training completes all epochs without OOM, GPU memory stays flat (not growing), test CR >= 0.0147
### Failure criteria: OOM recurs after fix (try next fix option); training completes but CR < 0.005 (recipe issue, not infra)

### Root cause detail:
- torch.compile with CUDA Graphs records a separate graph for each unique input tensor shape
- V5 data has variable-length sequences producing 51 distinct shapes over 2350 batches
- Each graph recording consumes GPU memory that PyTorch doesn't track as "allocated"
- Result: PyTorch reports 6.22 GiB allocated, but process actually uses 23.58 GiB (17.36 GiB in CUDA Graph private pools)
- This is a known PyTorch issue with variable-length inputs + torch.compile

### Fix priority (for dev directive):
1. **Best:** `torch._inductor.config.triton.cudagraph_skip_dynamic_graphs = True` — keeps compile benefits, skips graphs for variable shapes
2. **Good:** Pad all inputs to fixed sizes to reduce distinct shape count to ~1-3
3. **Safe fallback:** Disable torch.compile entirely (`compiled=False` in config) — loses ~10-20% throughput but guarantees no graph leak
4. **Complementary:** `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True` — helps fragmentation, use alongside option 1

### Urgency: HIGH
- GPU is idle (0% util, 53MiB VRAM, 15W)
- Dev is idle at prompt, appears unaware of crash
- Fix is straightforward (1-line config change for option 1)
- Pre-crash training speed was ~75 min/epoch (much faster than estimated 6-8 hours), so full 50-epoch training with patience=15 could complete in a few hours once relaunched

---

## Persistent Notes
- Previous constraint: Execution (training not launched) — RESOLVED by launching training
- Current constraint: Infrastructure (CUDA OOM from torch.compile CUDA Graphs)
- Previous gap: Unmeasurable (no training run)
- Current gap: Still unmeasurable (crash before epoch 1 completion), but loss trajectory 0.66->0.27 is strong signal
- Hypothesis history:
  - Cycle 1-3: "Launch full training, expect CR >= 0.0147 based on strong early correlation signals"
  - Cycle 4 (current): "Fix CUDA Graph OOM and relaunch — model learns well, OOM is purely infra"
- Key evidence accumulated:
  - V5 partial training: loss 0.66->0.27 over 2350 batches (20.7% of epoch 1) — excellent learning
  - V5 GPU util during training: 88-90% (healthy), ~20s per 50 batches
  - V5 OOM at batch 2350: 51 CUDA Graph shapes, 17.36 GiB untracked memory
  - BCE fix was applied between run 1 and run 3 (verify it persists)
  - V5-Small unit test: 679K params, CR=0.0015, rank_corr=0.089, return_corr=0.088 (50 batches)
  - V3 baseline: 136K params, CR=0.0147, rank_corr=0.0152, return_corr=0.0132 (full training, epoch 12)
  - V1 best ever: CR=0.0192, rank_corr=0.0197, dir_acc=0.5218 (full training, epoch 33)
  - V4 was a regression: 4.77M params, CR=0.0031, negative correlations. Overparameterized.
  - V5 data: 64/64 quarters, 11.4M samples, complete and verified
  - GPU: Quadro RTX 6000, 24GB, currently IDLE
  - Known data issues: 6/20 summary token dims are zeros, price token dim 15 always zero
- Watch items for next cycle:
  - Has OOM fix been applied? Check train.py for cudagraph config or compile changes
  - Has training been relaunched? Check for new run dirs in models/
  - If training running: monitor GPU memory trend (MUST stay flat, not growing — this is the key OOM indicator)
  - If training running: verify loss continues from similar starting point (model reinitializes, so loss should start ~0.66 again)
  - If epoch 1 completes: check val metrics for first real CR/correlation numbers
  - BCE fix still present in train.py?
  - 5 zombie processes still lingering (cosmetic but worth cleaning eventually)
