# Training Evaluation Guide

How the manager evaluates model training progress and decides next actions.

## Before Training — Pre-Run Validation (MANDATORY)

Training scripts follow the same mandatory validation cycle as data pipelines. The manager MUST verify this cycle was performed before allowing a full training run.

### Required Cycle

```
[Implement/Modify Training Script]
        │
        ▼
[Smoke Test] — --smoke-test flag (3 epochs, full data)
        │
        ├── Loss NaN or not decreasing → Fix and retry
        │
        └── Loss decreasing → Measure performance
                │
                ▼
[Performance Test] — measure GPU util, data loading, memory
        │
        ├── GPU util < 70% → Tune ChunkLoader (increase preload_ahead/threads)
        ├── Data loading dominates → Fix data pipeline bottleneck
        ├── Memory OOM → Reduce buffer_chunks or batch_size
        │
        └── ALL PASS → Ready for full training
                │
                ▼
[Full Training] — nohup, ProgressTracker, monitor
```

### Performance Measurements During Smoke Test

```bash
# While smoke test is running:

# GPU utilization — must be >70%
nvidia-smi --query-gpu=utilization.gpu,memory.used,memory.total --format=csv -l 5

# Check training log for timing instrumentation
tail -f train.log | grep "util\|data.*ms\|gpu.*ms"

# Expected output pattern (from trade_learning):
# data 2.1ms/batch | gpu 8.3ms/batch | GPU util: 80%
```

### What the Manager Checks

| Metric | Acceptable | Unacceptable | Action if Unacceptable |
|--------|-----------|--------------|------------------------|
| GPU utilization | >70% | <50% | Tune ChunkLoader: increase `preload_ahead_count` or `preload_threads` |
| Data loading time | <30% of total batch time | >50% of total | Data pipeline bottleneck — increase I/O threads, check disk |
| Memory | Stable, no OOM | OOM or growing | Reduce `buffer_chunks`, `batch_size`, or check for leaks |
| Loss after 3 epochs | Decreasing | NaN or flat | Check lr, data loading, model init |
| ProgressTracker | `_progress.md` exists | Missing | Require dev to add ProgressTracker |
| torch.compile | Enabled for transformers | Disabled without justification | Enable for 20-30% speedup |
| Mixed precision (AMP+FP16) | Always enabled | Disabled | MANDATORY — enable AMP + GradScaler immediately |
| Per-run directory | `models/run_YYYYMMDD_type/` created per run | Files mixed in one folder | MANDATORY — each run gets its own dir |
| run_meta.json | Written in run dir with type, config, timestamps | Missing | MANDATORY — write at start, update on completion |
| Every-epoch checkpoint | `latest_checkpoint.pt` written every epoch in run dir | Only on best val or missing | MANDATORY — add immediately |
| Checkpoint contents | All 9 required fields present | Missing optimizer/scaler/scheduler state | Fix before full run — broken resume |
| --resume flag | Finds latest run dir, restores full state | Missing or only restores model weights | MANDATORY — add before full run |
| --unit-test flag | 1 epoch, 50 batches, <2 min, validates all components | Missing | MANDATORY — fastest validation cycle |
| Training progression | unit → smoke → full (--resume) | Full run from scratch ignoring smoke weights | MANDATORY — full run must --resume from smoke checkpoint |

### ChunkLoader Tuning Reference

From V1-V4 implementations, these parameters control GPU feed rate:

| Parameter | Default | Increase if GPU starving | Reference |
|-----------|---------|--------------------------|-----------|
| `preload_ahead_count` | 15 | 20-30 | Chunks pre-submitted for decompression |
| `prefetch_chunks` | 20 | 30-40 | LRU cache size in RAM |
| `preload_threads` | 4-8 | 8-16 | Decompression thread pool workers |
| `buffer_chunks` (training) | 10 | 15-20 | Interleaved pool size (~55MB/chunk) |

### Proven Patterns from V1-V4

| Pattern | V1 | V3 | V4 | Trade | Manager Expects |
|---------|----|----|----|----|-----------------|
| ChunkLoader + IterableDataset | ✓ | ✓ | ✓ | ✓ | Required |
| ChunkInterleavedDataset (train) | ✓ | ✓ | ✓ | ✓ | Required for training |
| GPU timing instrumentation | ✗ | ✗ | ✗ | ✓ | Required — log data_ms, gpu_ms, util% |
| Mixed precision (AMP+FP16) | avail | avail | ✓ | ✗ | MANDATORY — always enabled, no exceptions |
| torch.compile | ✗ | ✗ | ✓ | ✗ | Required for transformer models |
| ProgressTracker | ✓ | ✗ | ✗ | ✗ | Required for all training scripts |
| --smoke-test flag | ✓ | ✓ | ✓ | ✓ | Required |
| Gradient clipping | ✓ | ✓ | ✓ | ✓ | Required (max_norm=1.0) |
| Per-run directory | ✗ | ✗ | ✓ (new) | ✗ | MANDATORY — models/run_YYYYMMDD_type/ |
| Every-epoch checkpoint | ✗ | ✗ | ✓ (new) | ✗ | MANDATORY — latest_checkpoint.pt in run dir |
| Full checkpoint contents | ✓ (model+opt+scaler) | partial | ✓ (new, all 9 fields) | ✓ (model+opt) | MANDATORY — all 9 fields |
| --resume flag | ✓ | ✗ | ✓ (new) | ✓ | MANDATORY — restore full training state |
| --unit-test flag | ✗ | ✗ | ✓ (new) | ✗ | MANDATORY — <2min component validation |

**Known gaps (remaining):**
- V1, V3, Trade: no per-run directories (files mixed)
- V1: saves only on best val (not every epoch)
- V3: no resume, no per-run dir
- V4: NOW FIXED — has per-run dirs, every-epoch save, full resume, unit test

## During Training — What to Monitor

### Loss Curves

Check the training log for loss trajectory:

```bash
# Check latest loss values
tail -50 firstrate_learning_v4/train.log | grep -i "loss\|epoch"

# Or check progress file
cat firstrate_learning_v4/train_progress.md
```

| Signal | Meaning | Action |
|--------|---------|--------|
| Train loss decreasing, val loss decreasing | Healthy training | Continue |
| Train loss decreasing, val loss flat | Beginning to overfit | Watch closely, may need early stop soon |
| Train loss decreasing, val loss increasing | Overfitting | Stop training, reduce model capacity or add regularization |
| Both losses flat | Training stalled | Adjust learning rate, check data loading |
| Loss NaN or exploding | Training diverged | Stop, reduce lr, check for data issues |

### Key Metrics to Track Per Epoch

| Metric | What It Measures | V3 Baseline | Target for V4 |
|--------|-----------------|-------------|---------------|
| Captured Return | Quality of top predictions for portfolio | 0.0147 | > 0.0147 |
| P@5% | Precision in top 5% of predictions | 0.334 | > 0.334 |
| Rank Correlation | Overall ranking quality | 0.015 | > 0.015 |
| Return Correlation | Predicted vs actual return correlation | 0.013 | > 0.013 |
| Direction Accuracy | Up/down prediction accuracy | 0.505 | > 0.531 (V2 level) |
| Return MAE | Return prediction error | 0.078 | < 0.078 |

### Resource Utilization

```bash
# GPU utilization — should be >70% during training
nvidia-smi --query-gpu=utilization.gpu,memory.used,memory.total --format=csv -l 5

# Check if ChunkLoader is keeping GPU fed
# If GPU util < 50%, increase preload_ahead_count or preload_threads
```

## After Training — Evaluation Protocol

### Step 1: Compare Against All Versions

Produce the standard comparison table:

```
| Metric          | V1      | V2     | V3     | V4     |
|-----------------|---------|--------|--------|--------|
| Captured Return | 0.0064  | 0.0112 | 0.0147 | ???    |
| P@5%            | 0.339   | 0.313  | 0.334  | ???    |
| Rank Corr       | -0.004  | 0.017  | 0.015  | ???    |
| Return Corr     | —       | —      | 0.013  | ???    |
| Direction Acc   | —       | 0.531  | 0.505  | ???    |
| Params          | 49K     | 151K   | 136K   | 4.77M  |
```

### Step 2: Evaluate Against Thresholds

| Result | Decision |
|--------|----------|
| V4 exceeds V3 on captured return AND rank corr | Success — V4 is new best backbone. Proceed to portfolio model. |
| V4 exceeds V3 on some metrics but not all | Partial success — iterate training recipe (loss weights, lr, augmentation) |
| V4 matches V3 but with 35x more params | Inefficient — the complexity isn't justified. Try reducing model size. |
| V4 underperforms V3 | Investigate: overfitting? data quality? architecture mismatch? |

### Step 3: Foresight Gap Analysis

The foresight benchmark achieves 170% return in 50 days. Calculate how much closer V4 gets:

```
Foresight gap = (foresight_captured_return - model_captured_return) / foresight_captured_return
```

Track this gap shrinking across versions. If a new version doesn't reduce the gap, question the approach.

### Step 4: Per-Quarter Stability

Good models should perform consistently across quarters, not just on average:
- Check captured return per quarter — high variance = unstable
- Check if performance degrades on recent quarters (distribution shift)
- Compare early quarters (small universe) vs late quarters (large universe)

## Decision Framework: What Next?

After evaluating V4 training results:

```
[V4 Training Complete]
        │
        ├── EXCEEDS V3 on key metrics
        │       │
        │       ├── By large margin (>30% improvement) → V4 is new backbone
        │       │       → Proceed to portfolio model development
        │       │       → Update goal_tracker.md
        │       │
        │       └── By small margin (<30%) → V4 promising but iterate
        │               → Try: adjust loss weights per direction.md
        │               → Try: learning rate schedule changes
        │               → Try: data augmentation
        │               → Re-train and re-evaluate
        │
        ├── MATCHES V3 (within noise)
        │       │
        │       └── V4 uses 35x more params for same result
        │               → Try reducing V4 model size (fewer layers/heads)
        │               → Try: contrastive learning (direction.md P3)
        │               → If still no improvement → consider V3 + enhancements
        │
        └── UNDERPERFORMS V3
                │
                ├── Check for overfitting (train vs val gap)
                ├── Check data quality (token preparation correctness)
                ├── Check training recipe (lr too high? wrong loss weights?)
                ├── Check architecture (is raw-token approach viable?)
                │
                └── If fundamental issue → pivot to V3 + direction.md priorities
                        (P3: contrastive learning, P5: calibration, P6: ensemble)
```

## Manager Actions During Training

| Training Phase | Manager Action |
|----------------|---------------|
| First 1-3 epochs | Check loss is decreasing, no NaN. GPU util >70%. Full pane check. |
| Early training (epochs 3-10) | Check val metrics improving. Compare to V3 baseline. Lightweight checks. |
| Mid training (epochs 10-30) | Watch for overfitting. Check per-epoch improvement rate. Is it plateauing? |
| Late training (epochs 30-50) | Check patience counter. Early stopping imminent? Final metrics trend. |
| Training complete | Full evaluation protocol. Update goal_tracker.md. Recommend next step. |
