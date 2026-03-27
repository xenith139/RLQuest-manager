# Current Status — 2026-03-27

## What's Running

**V5 data preparation (FULL)** — running on CPU with 18 parallel workers (PID 896622).
- This is for **V5** (Temporal Options Surface Transformer), NOT V4.
- Processing all 64 quarters of raw options data into 5-day multi-surface token format.
- Output: (n, 5, 128, 20) chunks with zstd compression.
- Smoke test validated: ~12 files/sec throughput, 34s on 5 representative quarters.
- Estimated full run time: unknown until we see large quarter throughput.
- Log: `firstrate_learning_v5/output/prepare_tokens_full.log`

**V5 model implementation** — dev is writing model.py + data_loader.py from `research/v5_design.md`.
- This is code implementation, not training.
- Runs in parallel with token prep (dev writes code while CPU prepares data).

## What's NOT Running

- **No training** — V5 model not implemented yet, V5 tokens not ready yet. Training will start after both complete.
- **V4 training** — stopped. Overfitting confirmed (CR peaked epoch 1 at 0.0187, collapsed to 0.0017 by epoch 6). V4 is done.
- **GPU** — idle (0%, 53MB). No GPU work available until V5 model + data are ready.

## V4 vs V5

- **V4**: Complete. Best result: CR=0.0187 (epoch 1). Overfits immediately — architectural ceiling from single-day snapshot.
- **V5**: In progress. Design complete, data prep running, model being implemented. Adds 5-day temporal lookback to address V4's limitation.
