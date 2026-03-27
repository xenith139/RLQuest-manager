## Step 5: Action Plan — 2026-03-27 ~22:30 UTC

### Status: NEW PLAN (DELIVERY-FOCUSED — previous plans were correct but never delivered)

### All possible actions:
1. **Launch V5-Small full training (`--full --no-smoke`)** — PRIMARY. All prerequisites met for 3+ hours.
2. Launch with smoke test (`--full`) — wasteful, 18-24hr smoke test unnecessary after unit test passed.
3. Re-run unit test with more logging — zero value, unit test already passed.
4. Fix summary token zero dims — deferred to post-training.
5. Fix price token dim 15 bug — deferred to post-training.
6. Fix config.d_ff bug — zero impact currently.
7. Add feature clipping — deferred to post-training.
8. Optimize IntraDayEncoder day-loop — deferred to post-training.
9. Return to V3 — premature without V5 results.
10. Design V5-Medium — premature without V5-Small baseline.
11. **SEND THE PROMPT TO DEV VIA TMUX** — THIS is the actual bottleneck. Previous cycles wrote the prompt but never sent it.

### Resources: GPU=IDLE (0%, 3+ hours), CPU=IDLE (load ~4/20 cores), Dev=IDLE (at prompt 3+ hours)

### Chosen action(s):

**Action #11 then #1: Actually deliver the training launch prompt to dev via tmux send-keys.**

**Root cause of stagnation:** Steps 1-5 have been producing correct analysis and correct prompts for 3 consecutive cycles, but the prompt in `step5_output.md` was never physically sent to the dev agent's tmux session. The management loop writes to a file; nothing reads that file and sends it. The fix is to use `tmux send-keys` to deliver the prompt directly.

### Validation gate:
- [x] Unit test passed (CR=0.0015, rank_corr=0.089, return_corr=0.088)
- [x] GPU util >70% during unit test (86%)
- [x] AMP+FP16 enabled
- [x] torch.compile enabled (reduce-overhead mode)
- [x] Checkpoint/resume working (all files produced in unit test)
- [x] Per-run directory (timestamped)
- [x] Design doc complete
- [x] Data complete (64/64 quarters, 11.4M samples)
- [x] Chained to follow-up (monitor epoch 1, compare against baselines)
- [x] Success criteria defined
- [x] Failure criteria defined

**ALL GATES PASS. No blockers.**

### Parallel tracks:
| Resource | During Training | After Training |
|----------|----------------|----------------|
| GPU | V5-Small full training | Next experiment |
| CPU | Data loading | Token prep fixes |
| Dev | Monitor epoch 1 metrics | Evaluate results |

### Success criteria:
- **Minimum:** Test CR >= 0.0147, return_corr > 0.0132
- **Strong:** Test CR >= 0.0192, P@5% > 0.33
- **Exceptional:** Test CR > 0.025
- **Infrastructure:** GPU util >= 80%, no NaN/crash

### Failure criteria:
- **Mild:** CR < 0.0147 but return_corr > 0.02 — tune loss weights
- **Moderate:** CR < 0.005 and return_corr < 0.01 — check for collapse/overfitting
- **Severe:** NaN/OOM/crash — debug immediately
- **Worst case:** CR <= V4 levels (~0.003) — consider V3 revert

---

## Recommended Prompt

**DELIVERY METHOD: tmux send-keys to session `claude-1-1774585502`**

The prompt to deliver:

```
Launch V5-Small full training immediately. The unit test passed (CR=0.0015, rank_corr=0.089, return_corr=0.088, 679K params, GPU util 86%). Data is complete (64/64 quarters, 11.4M samples). GPU has been idle 3+ hours.

Run this command:
cd /home/ubuntu/workspace/RLQuest && python -m firstrate_learning_v5.train --full --no-smoke

Justification for --no-smoke: unit test validated full pipeline, epoch time is 6-8 hours making smoke test cost 18-24 hours.

After launching, monitor first 100 batches for NaN/Inf. After epoch 1 (~6-8 hours), report: val CR, rank_corr, return_corr, per-component losses, GPU util. Update train_progress.md. Compare final results against V3 (CR=0.0147) and V1 (CR=0.0192).
```

## Delivery Confirmation

**CRITICAL: This prompt MUST be sent via `tmux send-keys` to the dev session. Writing it to this file is NOT sufficient. The orchestrator or this step must execute the tmux command.**

## Persistent Notes
- **Actions taken history:**
  - Cycle 1 (~20:35 UTC): Wrote "launch training" prompt to step5_output.md. NOT DELIVERED to dev.
  - Cycle 2 (~22:00 UTC): Wrote improved "launch training" prompt to step5_output.md. NOT DELIVERED to dev.
  - Cycle 3 (~22:30 UTC): THIS CYCLE. Identified delivery as the bottleneck. Prompt MUST be sent via tmux send-keys.
- **Pending items:**
  - Fix summary token dims 14-19 (after training)
  - Fix price token dim 15 (after training)
  - Optimize IntraDayEncoder day-loop (after training)
  - Fix config.d_ff unused bug (maintenance)
- **Validation patterns:** All validation gates have passed for 3 consecutive cycles. The bottleneck is not validation — it is delivery.
- **Effective prompts:** Unknown — no prompt has been delivered yet. The prompt format is ready but untested.
- **Watch items for next cycle:**
  - VERIFY training is actually running (new run dir in models/, GPU util > 0%)
  - If running: monitor epoch 1 metrics
  - If NOT running: escalate — the management process cannot deliver prompts and needs human intervention
- **STAGNATION ALERT:** 3 cycles with identical recommendation, 0 state changes. If this cycle also fails to deliver, the management loop is fundamentally broken for prompt delivery.
