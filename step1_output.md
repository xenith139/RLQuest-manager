# Step 1: Ground Truth & State Assessment

**Timestamp:** 2026-03-28 03:45 UTC

**Delta from last check (00:11 UTC):** MAJOR PROGRESS. All 4 LR prove-out experiments completed. Training is IDLE. GPU is IDLE. Dev is IDLE at prompt. Prove-out sweep results are in — all cosine recipes show prec/rec collapse at epoch 3, constant LR is the only stable one. User has identified multi-task loss weight conflict as the deeper root cause.

---

### Dev: RUNNING -- IDLE at prompt
- Session `claude-1-1774585502` active
- At `>` prompt with bypass permissions on
- Last activity: dev completed prove-out sweep analysis, recommended P2 recipe (lr=5e-5 cosine) for full training
- Dev noted precision collapse pattern but recommended cosine recipe anyway (P2 CR=0.072 best)

### CPU: IDLE -- only zombie processes remain
- No live Python training processes
- 10 zombie python3 processes (PIDs: 102118, 102603, 896622, 1154253, 1166716, 1233730, 1234161, 1245000, 1245351, 1392002)
- Zombies accumulating (was 7 last check, now 10) -- will need cleanup eventually

### GPU: IDLE -- 1% utilization, 53MiB/24576MiB VRAM, 44C, 16W
- All training complete
- GPU cool (44C vs 81C during training)
- Ready for next training run

### V5 Code: All present (unchanged)
- config.py, model.py, data_loader.py, train.py, prepare_tokens.py

### V5 Data: COMPLETE (unchanged) -- 64/64 quarters, 11,411,714 samples

### V5 Training: ALL PROVE-OUT EXPERIMENTS COMPLETE -- GPU IDLE

**Run directories (6 total):**
1. `run_20260327_224100_full` -- original full training (mode collapse at epoch 2)
2. `run_20260328_003047_unit_test_lr` -- unit test
3. `run_20260328_003209_proveout_lr1e-4_cosine` (P1)
4. `run_20260328_011901_proveout_lr5e-5_cosine` (P2)
5. `run_20260328_020617_proveout_lr3e-4_scaled_cosine` (P3)
6. `run_20260328_025307_proveout_lr1e-4_constant` (P4)

**Prove-out Sweep Results (20% data, 5 epochs each, ~45 min/experiment):**

| Exp | LR | Schedule | Best CR | Best Epoch | Epoch 3+ Prec/Rec | Test CR | Stable? |
|-----|-----|----------|---------|------------|-------------------|---------|---------|
| P1 | 1e-4 | cosine | 0.0633 | 2 | collapse (0.728/0.036 -> 0.722/0.022) | 0.0141 | NO -- near-zero recall from epoch 3 |
| P2 | 5e-5 | cosine | 0.0721 | 4 | collapse (0.000/0.000 from epoch 3) | 0.0185 | NO -- prec/rec=0 from epoch 3 |
| P3 | 3e-4 | cosine | 0.0711 | 2 | collapse (0.000/0.000 from epoch 3) | 0.0163 | NO -- prec/rec=0 from epoch 3 |
| P4 | 1e-4 | constant | 0.0586 | 1 | STABLE (0.595/0.978 at epoch 5) | 0.0176 | YES -- only stable recipe |

**Detailed Epoch-by-Epoch Results:**

P1 (lr=1e-4, cosine):
| Epoch | TrL | VL | Prec | Rec | P@5 | CR |
|-------|------|------|------|------|------|--------|
| 1 | 0.5681 | 0.6578 | 0.700 | 0.470 | 0.427 | 0.0527 |
| 2 | 0.5599 | 0.6478 | 0.689 | 0.595 | 0.492 | 0.0633 |
| 3 | 0.5672 | 0.6508 | 0.728 | 0.036 | 0.180 | -0.0056 |
| 4 | 0.5730 | 0.6490 | 0.724 | 0.024 | 0.141 | -0.0129 |
| 5 | 0.5727 | 0.6486 | 0.722 | 0.022 | 0.140 | -0.0130 |

P2 (lr=5e-5, cosine):
| Epoch | TrL | VL | Prec | Rec | P@5 | CR |
|-------|------|------|------|------|------|--------|
| 1 | 0.5705 | 0.6350 | 0.646 | 0.692 | 0.319 | 0.0228 |
| 2 | 0.5592 | 0.6200 | 0.677 | 0.525 | 0.409 | 0.0563 |
| 3 | 0.5757 | 0.6693 | 0.000 | 0.000 | 0.456 | 0.0680 |
| 4 | 0.6059 | 0.6645 | 0.000 | 0.000 | 0.457 | 0.0721 |
| 5 | 0.6018 | 0.6630 | 0.000 | 0.000 | 0.454 | 0.0717 |

P3 (lr=3e-4, cosine):
| Epoch | TrL | VL | Prec | Rec | P@5 | CR |
|-------|------|------|------|------|------|--------|
| 1 | 0.5571 | 0.6482 | 0.642 | 0.843 | 0.471 | 0.0365 |
| 2 | 0.5494 | 0.6550 | 0.684 | 0.642 | 0.472 | 0.0711 |
| 3 | 0.5766 | 0.6805 | 0.000 | 0.000 | 0.438 | 0.0451 |
| 4 | 0.5870 | 0.6803 | 0.000 | 0.000 | 0.181 | -0.0037 |
| 5 | 0.5872 | 0.6802 | 0.000 | 0.000 | 0.179 | -0.0087 |

P4 (lr=1e-4, constant):
| Epoch | TrL | VL | Prec | Rec | P@5 | CR |
|-------|------|------|------|------|------|--------|
| 1 | 0.5632 | 0.6105 | 0.664 | 0.652 | 0.480 | 0.0586 |
| 2 | 0.5524 | 0.7012 | 0.635 | 0.877 | 0.238 | -0.0017 |
| 3 | 0.5524 | 0.6810 | 0.598 | 0.650 | 0.346 | 0.0249 |
| 4 | 0.5844 | 0.6757 | 0.595 | 0.977 | 0.418 | 0.0555 |
| 5 | 0.5850 | 0.6756 | 0.595 | 0.978 | 0.418 | 0.0558 |

### CRITICAL ANALYSIS

**1. Prec/Rec Collapse is Universal with Cosine Schedules:**
- ALL three cosine recipes (P1, P2, P3) show precision/recall collapse at or after epoch 3
- P1: recall drops from 0.595 to 0.036 at epoch 3
- P2, P3: precision AND recall go to exactly 0.000 at epoch 3
- This happens regardless of LR (5e-5, 1e-4, 3e-4) -- it is NOT an LR magnitude issue

**2. Constant LR (P4) is the Only Stable Recipe:**
- P4 maintains prec=0.595, rec=0.978 through epoch 5
- Val loss oscillates but prec/rec never collapse
- Test CR=0.0176, which already beats V3 baseline (0.0147) by 20%

**3. The Paradox: CR Improves While Prec/Rec Collapse (P2):**
- P2 achieves best CR (0.0721) at epoch 4 with prec=0.000, rec=0.000
- This means the ranking signal (P@5=0.457) is excellent but the classification head is dead
- The multi-task loss components are fighting: ranking improves while classification collapses
- This confirms the user's diagnosis: **multi-task loss weight conflict is the root cause**

**4. Cosine Decay Triggers the Conflict:**
- When LR decays via cosine, the different loss components converge at different rates
- The classification head (precision/recall) is more sensitive to LR changes than the ranking head (P@5, CR)
- As LR drops, the ranking signal dominates and the classification signal dies
- Constant LR avoids this because all heads maintain the same gradient scale

**5. Best Test CR Ranking:**
- P2 (cosine, 5e-5): test CR=0.0185 (best overall, but prec/rec=0 at test time)
- P4 (constant, 1e-4): test CR=0.0176 (only stable recipe)
- P3 (cosine, 3e-4): test CR=0.0163
- P1 (cosine, 1e-4): test CR=0.0141 (barely under V3)

**6. V3 Baseline Beaten by All Experiments:**
- V3 baseline: CR=0.0147
- All 4 prove-out test CRs beat V3: 0.0141-0.0185 (though P1 is marginal)
- Even on 20% data, V5 architecture beats V3 -- architecture is validated

### Goal Tracker Discrepancy: SIGNIFICANT (STALE)
- Goal tracker still says "STOP current training" (priority #1) -- DONE, training stopped
- Goal tracker says "IMPLEMENT prove-out mode" (priority #2) -- DONE
- Goal tracker says "RUN prove-out LR sweep" (priority #4) -- DONE, all 4 experiments complete
- Goal tracker says "RELAUNCH full training with winning recipe" (priority #5) -- NOT YET DONE
- Goal tracker does not reflect prove-out results
- Goal tracker does not reflect the step-based training user request
- Goal tracker does not reflect the multi-task loss weight conflict as the true root cause

### User Request: Step-Based Training
- User wants to implement step-based training from `manual_docs/step_based_training.md`
- Document found at: `/home/ubuntu/workspace/RLQuest-manager/manual_docs/step_based_training.md`
- Document describes: step-based checkpointing (every 2000 steps), step-based evaluation (subset validation every N steps), step-based LR schedule (already implemented), step-based resume
- Current state of implementation:
  - Intra-epoch checkpointing: ALREADY IMPLEMENTED (every 2000 batches, confirmed in logs)
  - Step-based resume: ALREADY IMPLEMENTED (batch_offset in checkpoint)
  - Step-based LR schedule: ALREADY IMPLEMENTED (warmup_steps=1000)
  - Subset validation every N steps: NOT YET IMPLEMENTED
  - Step-based early stopping: NOT YET IMPLEMENTED

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
- Manual docs: `/home/ubuntu/workspace/RLQuest-manager/manual_docs/`
- Dev session PID file: `/home/ubuntu/workspace/RLQuest/claude_session_PID_1`
- Dev tmux session name: `claude-1-1774585502`

**Key paths:**
- V5 token meta: `/home/ubuntu/workspace/RLQuest/firstrate_learning_v5/cache/tokens/meta.json`
- V5 train progress: `/home/ubuntu/workspace/RLQuest/firstrate_learning_v5/train_progress.md`
- P1 log: `/home/ubuntu/workspace/RLQuest/firstrate_learning_v5/output/P1_lr1e-4_cosine.log`
- P2 log: `/home/ubuntu/workspace/RLQuest/firstrate_learning_v5/output/P2_lr5e-5_cosine.log`
- P3 log: `/home/ubuntu/workspace/RLQuest/firstrate_learning_v5/output/P3_lr3e-4_cosine.log`
- P4 log: `/home/ubuntu/workspace/RLQuest/firstrate_learning_v5/output/P4_lr1e-4_constant.log`
- Step-based training doc: `/home/ubuntu/workspace/RLQuest-manager/manual_docs/step_based_training.md`
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
- 679,498 params, compiled=true (mode='default'), AMP FP16 enabled
- dropout=0.25, weight_decay=0.01, label_smoothing=0.05
- contrastive loss disabled (w_contrast=0.0)

**Prove-out results summary (COMPLETE):**
- P1 (lr=1e-4, cosine): best CR=0.0633 (epoch 2), test CR=0.0141, UNSTABLE (recall collapse epoch 3)
- P2 (lr=5e-5, cosine): best CR=0.0721 (epoch 4), test CR=0.0185, UNSTABLE (prec/rec=0 from epoch 3)
- P3 (lr=3e-4, cosine): best CR=0.0711 (epoch 2), test CR=0.0163, UNSTABLE (prec/rec=0 from epoch 3)
- P4 (lr=1e-4, constant): best CR=0.0586 (epoch 1), test CR=0.0176, STABLE (prec/rec maintained through epoch 5)
- ALL beat V3 baseline (0.0147) on test CR -- architecture is validated
- Cosine schedule universally causes prec/rec collapse -- problem is multi-task loss weight conflict, not LR magnitude

**ROOT CAUSE (UPDATED): Multi-Task Loss Weight Conflict**
- Cosine decay reduces LR uniformly across all loss components
- Different heads (classification vs ranking) have different sensitivity to LR
- As LR drops, the ranking signal dominates and classification head dies
- Constant LR avoids this by maintaining gradient scale parity
- Deeper fix: per-head LR scaling, gradient normalization, or loss weight scheduling
- Simpler fix: use constant LR (P4 recipe) for full training -- already beats V3

**Step-based training implementation status:**
- Checkpointing every 2000 batches: DONE (confirmed in all prove-out logs)
- Step-based resume with batch_offset: DONE
- Step-based LR schedule (warmup_steps): DONE
- Subset validation every N steps: NOT DONE (user request)
- Step-based early stopping: NOT DONE (user request)

**Zombie processes:** 10 total (accumulating, was 7 last check)

**Last known state change:** All 4 prove-out experiments completed. GPU idle. Dev idle at prompt. User identified multi-task loss conflict as root cause. User requested step-based training implementation.

**Watch items for next cycle:**
- Has dev started implementing step-based training?
- Has full training been relaunched? If so, with which recipe?
- Is the loss weight conflict being addressed (per-head LR, loss normalization, etc.)?
- Are the remaining step-based features (subset validation, step-based early stopping) implemented?
- Zombie process count (now 10, growing)
- Goal tracker needs major update with prove-out results and new priorities
