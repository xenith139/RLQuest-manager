# Step 1: Ground Truth & State Assessment

**Timestamp:** 2026-03-27 21:28 UTC

**Delta from last check (20:35 UTC):** Nothing material changed. Dev still idle, GPU still idle, no full training launched. Goal tracker was updated and now accurately reflects reality.

---

### Dev: RUNNING — IDLE at prompt (~2.5 hours since unit test completed at 18:57)
- Session `claude-1-1774585502` active, 1 window
- Prompt showing `>` with bypass permissions on
- Last output was V5-Small unit test summary (same as last check)

### CPU: MOSTLY IDLE — load avg 4.06 on 20 cores (20% utilized)
- Same 3 zombie python processes (PIDs 102118, 102603, 896622) — no new processes
- No live Python processes running

### GPU: COMPLETELY IDLE — 0-1% utilization, 53MiB/24576MiB VRAM, 16W/260W
- Quadro RTX 6000, 24GB
- No processes on GPU
- Temperature 31C (idle)
- infoROM corruption warning (cosmetic)

### V5 Code: All present (unchanged)
- config.py, model.py, data_loader.py, train.py, prepare_tokens.py

### V5 Data: COMPLETE — 64/64 quarters, 11,411,714 samples (unchanged)

### V5 Training: Unit test COMPLETE. Full training NOT STARTED.
- Unit run: `run_20260327_185452_unit` — CR=0.0015, rank_corr=0.089, 679K params
- No new run directories created since last check

### Goal tracker discrepancy: NONE
- Goal tracker was updated (21:45 UTC stamp) and now accurately reflects reality: data complete, unit test passed, dev idle, GPU idle, full training not launched.
- Priority queue correctly identifies "Launch V5-Small full training" as #1 immediate action.

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
- V5 token status: `/home/ubuntu/workspace/RLQuest/firstrate_learning_v5/cache/tokens/status.json`
- V5 train progress: `/home/ubuntu/workspace/RLQuest/firstrate_learning_v5/train_progress.md`
- V5 unit results: `/home/ubuntu/workspace/RLQuest/firstrate_learning_v5/models/run_20260327_185452_unit/training_results.json`
- Goal tracker: `/home/ubuntu/workspace/RLQuest-manager/goal_tracker.md`

**Hardware:**
- nproc: 20 cores
- GPU: Quadro RTX 6000, 24GB VRAM, CUDA 12.4, Driver 535.288.01
- RAM: 64GB (58GB available)
- infoROM corrupted warning on GPU (cosmetic, not functional)

**V5 data stats (stable):**
- 64/64 quarters, 11,411,714 total samples
- Train: 40 quarters, 5,798,357 samples, 138 chunks
- Val: 12 quarters, 2,769,223 samples, 63 chunks
- Test: 12 quarters, 2,844,134 samples, 63 chunks
- Token dim: 20, max_tokens_per_day: 128, n_days: 5

**V5-Small unit test results (stable):**
- 679,498 params, compiled=true
- 50 batches, 1 epoch, 126 seconds
- CR=0.0015, rank_corr=0.089, return_corr=0.088, precision=0.7285, dir_acc=0.3779

**Known issues (from goal tracker):**
1. Summary token dims 14-19 all zeros (6/20 wasted)
2. Price token dim 15 always zero
3. config.d_ff unused by model (hardcoded d_model*4)
4. Feature scale mismatch on moneyness/mid_price_norm/intrinsic_value
5. Epoch time ~6-8 hours limits iteration speed

**Zombie processes:** 5 zombies (2 sh, 3 python3) — harmless but present
- tmux_view_claude.sh naming mismatch: looks for `claude_1` but session is `claude-1-1774585502`

**Last known state change:** Goal tracker updated to accurately reflect reality (between 20:35 and 21:45 UTC). No other changes.

**Critical observation:** GPU has been idle for ~2.5 hours since unit test completed. Full training should have been launched immediately after unit test success. This is the #1 priority action.

**Watch items for next cycle:**
- Has full training been launched? Check for new run dirs in V5 models/
- If training running: check GPU utilization, epoch progress, val loss trend
- Watch for overfitting after epoch 3-5 (val loss divergence)
- Monitor per-component loss magnitudes for conflicts
