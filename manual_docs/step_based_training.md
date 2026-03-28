# Step-Based Training Design

## Why Steps Instead of Epochs

Modern LLMs (GPT, LLaMA, Claude) and large-scale models are trained in **steps** (individual gradient updates), not epochs (full passes through the dataset). This is the correct paradigm for RLQuest's V5 and future models.

### Problems with Epoch-Based Training

1. **Long epochs hide progress.** V5-Small has ~11,325 batches per epoch at ~7 seconds each = ~22 hours per epoch. If training crashes at batch 9,000, you lose 17.5 hours of work with per-epoch checkpointing only. Intra-epoch checkpoints at every 2,000 batches reduce max loss to ~4 hours.

2. **Evaluation is too infrequent.** Waiting 22 hours to see the first validation metrics is wasteful. If the model is diverging, you want to know at step 2,000, not after step 11,325.

3. **Learning rate schedules should be step-based.** Cosine warmup over N steps is more precise than epoch-based warmup. The V5 config already does this (`warmup_steps=1000`), which is correct.

4. **Logging and monitoring are step-based.** GPU util, loss, and gradient norms are already logged every 50 batches. Extending this to checkpointing and evaluation is natural.

5. **Resumability is finer-grained.** A step-based checkpoint at batch 8,000 loses only the batches since 8,000. An epoch-based checkpoint at epoch 0 loses everything.

## Step-Based Training Architecture

### Checkpointing

```
Every N steps (default: 2000):
  Save latest_checkpoint.pt with:
    - step (global step count, not epoch)
    - batch_offset (position within current epoch)
    - model_state_dict
    - optimizer_state_dict
    - scaler_state_dict
    - scheduler_state_dict (step-level scheduler)
    - best_val_score
    - patience_counter
    - history
    - config
```

### Evaluation

```
Every E steps (default: every epoch, ~11,325 steps):
  Run full validation
  Compute all metrics: CR, P@5%, rank_corr, return_corr, direction_acc
  Save best model if improved
  Log to training results

Optionally: run lightweight validation every N steps (e.g., 5,000):
  Subset validation (10% of val data) for quick trend check
  Log: val_loss_subset, approximate metrics
  This gives a midpoint signal without the cost of full eval
```

### Learning Rate Schedule

```
Step-based cosine warmup (already correct in V5):
  warmup_steps: 1000
  total_steps: max_epochs * batches_per_epoch

  def lr_schedule(step):
      if step < warmup_steps:
          return step / warmup_steps  # linear warmup
      progress = (step - warmup_steps) / (total_steps - warmup_steps)
      return max(0.05, 0.5 * (1 + cos(pi * progress)))  # cosine decay
```

### Logging

```
Every 50 steps: loss, GPU util, data_ms, gpu_ms (already correct)
Every 500 steps: per-component loss magnitudes (magnitude, direction, quantile, return, etc.)
Every 2000 steps: checkpoint saved
Every epoch: full validation + metrics + best model check
```

### Resume

```
Resume from step N:
  Load checkpoint → restore model, optimizer, scaler, scheduler
  Set start_step = checkpoint['step'] + 1
  Set batch_offset = checkpoint['batch_offset']
  Skip batches until reaching batch_offset within the current epoch
  Continue training from exact position
```

## Benefits for RLQuest

| Aspect | Epoch-Based | Step-Based |
|--------|------------|------------|
| Max training loss on crash | 22 hours (full epoch) | 4 hours (2000 steps) |
| First validation signal | 22 hours | 22 hours (full), ~10 hours (subset eval at 5000 steps) |
| Checkpoint granularity | 1 per epoch | 1 per 2000 steps (~5 per epoch) |
| Resume precision | Start of epoch | Within 2000 steps of crash point |
| LR schedule | Epoch boundaries (coarse) | Per-step (already correct) |
| Monitoring | Per-batch loss (already correct) | Same + periodic eval |

## Implementation Priority

1. **Intra-epoch checkpointing** — already implemented (every 2000 batches) after the OOM fix
2. **Step-based resume** — already implemented (batch_offset field in checkpoint)
3. **Subset validation every N steps** — not yet implemented, add in next iteration
4. **Step-based early stopping** — instead of patience in epochs, patience in evaluation cycles (which could be every 5000 steps)
