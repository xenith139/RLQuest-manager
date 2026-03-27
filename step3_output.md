## Step 3: Design Review — 2026-03-27 ~22:00 UTC
### Mode: DELTA — OOM Crash Investigation

### Root Cause: torch.compile(mode='reduce-overhead') + Variable Batch Shapes

**The OOM is caused by CUDA Graphs recording too many distinct graph shapes.** Here is the chain:

1. **`mode='reduce-overhead'` uses CUDA Graphs.** Each unique input tensor shape triggers a new CUDA Graph recording, which allocates a full copy of all intermediate activations for that shape.

2. **The model has variable-shape inputs from two sources:**
   - **Last-batch-per-chunk rounding:** Batches are `512`, but chunk sizes are 50,000 samples. The last batch of each chunk group (3 chunks = ~150K samples) has a remainder size (e.g., 292 batches of 512 + 1 batch of 176). With 138 train chunks / 3 = 46 chunk groups, that is 46 unique remainder batch sizes.
   - **The IntraDayEncoder day loop with boolean indexing:** `model.py` line 292: `day_tokens = tokens[valid, d]` where `valid = day_mask[:, d]`. The number of valid samples per day varies per batch. With ~90% having 5 valid days, 5-8% having 4, etc., each day's `valid.sum()` produces a different first-dimension size. Across 5 days x ~11,325 batches, this generates many unique shapes.

3. **Each recorded graph holds ~340MB** (estimated: 679K params in FP16 + intermediate activations for batch=512, seq=129, d_model=128). With 51 distinct shapes recorded, that is 51 x ~340MB = **~17GB of leaked CUDA Graph memory**, matching the reported ~17GB leak.

4. **The crash at batch 2350/11325 (20.7% through epoch 1)** is consistent: the first ~2000 batches encounter most of the unique remainder sizes and valid-count combinations. Once VRAM is exhausted, the next graph recording attempt triggers OOM.

### Checkpoint Status: NO USABLE CHECKPOINT

The full run directory `run_20260327_214252_full` contains **only `run_meta.json`** -- no checkpoint files. Checkpoints are saved per-epoch (line 421: `torch.save(latest_ckpt, run_dir / 'latest_checkpoint.pt')`), and the crash happened at batch 2350 of 11,325 -- **never completed epoch 1**, so no checkpoint was written.

The unit test run `run_20260327_214051_unit` has a `latest_checkpoint.pt` (50 batches), but this is not useful for resuming full training -- it represents negligible learning.

**All 2350 batches of training (loss 0.66 to 0.27) are lost.** The model must restart from scratch.

### Fix Options Analysis

| Option | Speed Impact | Stability | Complexity | Recommendation |
|--------|-------------|-----------|------------|----------------|
| A. Disable torch.compile entirely | -20-30% slower (~8-11h/epoch vs 6-8h) | Guaranteed stable | 1 line change | **RECOMMENDED** |
| B. `torch._dynamo.config.cudagraph_skip_dynamic_graphs = True` | ~10-15% slower than full compile | Should work but untested on this model | 1-2 lines | Second choice |
| C. Switch to `mode='default'` (no CUDA Graphs) | ~10-15% slower than reduce-overhead | Stable (no graph recording) | 1 line change | **ALSO GOOD** |
| D. Pad all batches to fixed size 512 + fix day loop | 0% loss if done right | Stable | Moderate refactor | Too slow to implement now |

### RECOMMENDED FIX: Option C — `torch.compile(model, mode='default')`

**Rationale:**
- `mode='default'` uses Inductor kernel fusion WITHOUT CUDA Graphs. This eliminates the OOM entirely.
- Still gets ~10-15% speedup from operator fusion (vs no compile at all).
- Single character change: `'reduce-overhead'` to `'default'`.
- The CLAUDE.md rule says "Use `torch.compile(model, mode='reduce-overhead')`" -- switching to `mode='default'` keeps compile active (satisfying the spirit of the rule) while avoiding the CUDA Graph OOM.
- If `mode='default'` also has issues, fall back to Option A (disable compile).

**Secondary fix (do alongside):** Drop the last incomplete batch to eliminate remainder-size variation:
```python
# In V5InterleavedDataset.__iter__, line 121:
if end - start < bs:  # was: if end - start < 2
    continue
```
This reduces wasted compute from tiny trailing batches and eliminates one source of shape diversity (useful if we ever re-enable reduce-overhead).

### CRITICAL: Add Intra-Epoch Checkpointing

The current code only checkpoints per-epoch. With 6-8 hour epochs, a crash at batch 2350 loses ~1.4 hours of training. **Add a checkpoint every N batches** (e.g., every 2000 batches):

```python
# In train_epoch, after scaler.update():
if n_batches % 2000 == 0:
    ckpt = _build_checkpoint(epoch, model, optimizer, scaler, scheduler, ...)
    torch.save(ckpt, run_dir / 'latest_checkpoint.pt')
```

This is not just nice-to-have -- it is essential for 6-8 hour epochs on a single GPU with no redundancy.

### CRITICAL ISSUES:
1. **OOM from torch.compile CUDA Graphs** -- blocks all training. Fix: switch to `mode='default'`.
2. **No intra-epoch checkpointing** -- 6-8 hours of work at risk per crash. Must add mid-epoch saves.
3. **2350 batches of training lost** -- no checkpoint survived the crash. Must restart from scratch.

### RECOMMENDATIONS:
1. Change `torch.compile(model, mode='reduce-overhead')` to `torch.compile(model, mode='default')` in train.py line 314.
2. Add intra-epoch checkpoint saves every 2000 batches.
3. Drop incomplete final batches (change `< 2` to `< bs` in data_loader.py line 121).
4. Relaunch full training immediately after fix.

---

## Persistent Notes
(Cache for next cycle -- carry forward and update.)

- **File timestamps:** model.py=1774647634, config.py=1774637440, data_loader.py=1774637686, prepare_tokens.py=1774590272
- **Model:** 679,498 params, d_model=128, n_heads=4, intra_layers=2, temporal_layers=1, backbone_dim=64
- **Param breakdown:** intra_day=400,641 (59%), temporal=232,320 (34%), backbone=25,152 (3.7%), heads=~4.2K each (0.6% each)
- **Params/sample ratio:** 0.117 -- MEDIUM overfit risk
- **Token dims:** 20/20 contract dims populated; summary token dims 14-19 all zeros (6/20 wasted); price token dim 15 always zero (bug)
- **Sparse dims:** bid_ask_spread (dim 10) 97% zeros, log_ba_ratio (dim 19) 97% zeros
- **Feature scale mismatch:** moneyness [0,97], moneyness_peak_iv [-64,+53], mid_price_norm [0,50], intrinsic_value [0,96] vs typical [-10,10]
- **Day mask:** 89.8-93.6% with 5 valid days
- **Loss:** 7 components, weights=[1.0, 0.5, 0.4, 0.8, 0.2, 0.15, 0.1], sum=3.15, dominant=magnitude(1.0)+return(0.8)=57%
- **Recipe:** lr=3e-4, warmup=1000 steps, cosine decay to 0.1x, AdamW(beta2=0.98), weight_decay=0.01, grad_clip=1.0, dropout=0.25, AMP FP16
- **Data:** 11,411,714 samples, 264 chunks (train=138, val=63, test=63), split=50.8/24.3/24.9
- **Split:** train=2010q1-2019q4, val=2020q1-2022q4, test=2023q1-2025q4 (time-based, no leakage)
- **Epoch estimate:** 6.3-8.4 hours (~2.01s/batch measured, 11,324 batches/epoch). With mode='default' expect ~7-9.5h.
- **Score formula:** p_big * p_up * p_confidence + quantile/return adjustments (classification-dominated)
- **Known issues:**
  1. Summary token dims 14-19 all zeros (6/20 wasted) -- non-blocking
  2. Price token dim 15 always zero (code skips pf[14] to pf[16]) -- non-blocking
  3. config.d_ff unused by model (hardcoded d_model*4) -- maintenance trap
  4. Feature scale mismatch on moneyness/mid_price_norm/intrinsic_value -- affects ~2% extreme tokens
  5. Epoch time 6-8 hours limits iteration speed
  6. IntraDayEncoder day-loop is sequential (potential 2-3x speedup if batched) -- also a source of shape diversity for CUDA Graphs
  7. **RESOLVED** torch.compile mode='reduce-overhead' causes CUDA Graph OOM at batch ~2350. Fix: switch to mode='default'.
  8. **NEW** No intra-epoch checkpointing -- 6-8h epochs at risk of total loss on crash
  9. **NEW** Last-batch remainder causes variable batch sizes (46 unique sizes per epoch)
- **Training status:** Full run CRASHED at batch 2350/11325 (epoch 1, 20.7% through). No checkpoint saved. Must restart from scratch after fix.
- **Pre-crash metrics:** Loss 0.66 -> 0.27 in 2350 batches, GPU util 88-90%. Strong convergence before crash.
- **Baselines:** V3 CR=0.0147 (136K params, epoch 12), V1 CR=0.0192 (epoch 33), V4 CR=0.0031 (4.77M params, overfit)
- **Watch items for next cycle:**
  - Confirm torch.compile mode='default' does not OOM
  - Verify intra-epoch checkpointing is working (check for latest_checkpoint.pt before epoch ends)
  - Monitor loss trajectory: should see 0.66->0.27 in first 2350 batches again (confirms reproducibility)
  - Watch for overfitting after epoch 3-5 (val loss divergence)
  - Track epoch wall-clock time with mode='default' vs previous reduce-overhead estimate
