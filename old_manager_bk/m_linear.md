# Manager Linear Analysis — 2026-03-27

## Phase 1: What Is Actually Happening Right Now?

**Q1: Is the dev session running?**
Yes. SESSION_RUNNING: claude-1-1774585502.

**Q2: What is dev currently doing?**
Dev is idle at prompt. Just completed: V5 config update to V5-Small (d_model=128, 2+1 layers), train.py implementation, and unit test — all passed. GPU util 86%, all 9 checkpoint fields, AMP+FP16 enabled, torch.compile enabled.

**Q3: What processes are running on CPU right now?**
V5 token preparation running (PID 896622 + 4 workers at 68-78% CPU, 13-16GB each). 47/64 quarters complete. 20 cores available.

**Q4: What is the GPU doing right now?**
Idle — 0%, 53MB/24GB. No training running.

**Q5: What files exist on disk?**
- V5 tokens: 47/64 quarters done, incomplete
- V5 code: __init__.py, config.py, model.py, data_loader.py, train.py — ALL exist
- V5 config: d_model=128, n_heads=4, backbone_dim=64, batch_size=512 (V5-Small ✓)
- V5 unit test run dir exists with all 5 files + 9 checkpoint fields
- V4 best: epoch 1 CR=0.0187, P@5%=0.404
- Research docs: v5_design.md, v5_design_review.md, v5_data_prep_performance.md, side.md

**Q6: Does filesystem match goal_tracker.md?**
No — goal_tracker says "Next: update config to V5-Small, implement train.py, unit test." Reality: all three are DONE. Dev moved faster than expected. Goal tracker is stale — needs update.

---

## Phase 2: Where Are We Relative to the Goal?

**Q7: What is the current best model performance?**
V4 epoch 1: CR=0.0187, P@5%=0.404, RetCorr=0.0175, Prec=0.627. Overfits by epoch 2. V5 not trained yet.

**Q8: What is the target performance?**
From goals.md: close gap to foresight. Realistic target: CR > 0.025 (~100% annualized). Stretch: CR > 0.035 (~125% annualized).

**Q9: How big is the gap?**
Current CR 0.0187 vs target 0.025 → need 34% improvement. Vs stretch 0.035 → need 87% improvement.

**Q10: Is the gap shrinking?**
Unknown — V5 not yet trained. V4 was a dead end (overfits). We shifted from V4 to V5 architecture. No V5 training data yet. Gap is static until V5 trains.

---

## Phase 3: What Is Blocking Progress?

**Q11: What is the single biggest constraint?**
**DATA** — V5 token preparation is 47/64 quarters (73% done). All code is ready (config, model, loader, train, unit test passed). Training can't start until data finishes. Estimated ~15 min remaining.

**Q12: How do I know this is the real constraint?**
All code is implemented and validated. train.py unit test passed with GPU util 86%. The only thing missing is complete token data. Once data finishes → training can start immediately.

**Q13: Has the constraint changed since last cycle?**
Yes — last cycle the constraint was "train.py not implemented." Dev completed it. Constraint shifted to data completion (which is in progress and will resolve itself in ~15 min).

---

## Phase 4: Is the Current Plan Still the Right Plan?

**Q14: What is the current hypothesis?**
"V5-Small's temporal architecture (5-day, 2+1 layers, ~1M params) will improve on V4's CR=0.0187 without overfitting, because: (a) temporal lookback captures momentum patterns V4 can't see, (b) smaller model (0.17 params/sample vs V4's 0.82) reduces overfitting risk, (c) 2hr/epoch enables 3-4 experiments/week."

**Q15: Has any new evidence confirmed or weakened this hypothesis?**
Strengthened: V5-Small unit test passed with GPU util 86% and loss=0.6262 (finite, no NaN). Model processes 5-day data correctly. torch.compile works. No evidence against the hypothesis.

**Q16: Is there a faster way to test this hypothesis?**
No — V5-Small IS the fast path. ~1M params, ~2hr/epoch, 1-1.5 days to convergence. This is already 3x faster than V5-Large would have been.

**Q17: Are we building the right thing, or continuing from inertia?**
Right thing. V5-Small addresses: (a) V4's overfitting (smaller model), (b) V4's temporal blindness (5-day lookback), (c) iteration speed (2hr vs 7hr/epoch). The pivot from V5-Large to V5-Small was the correct strategic decision.

---

## Phase 5: Is the Design Sufficient?

**Q18: Is the model size appropriate?**
Yes. V5-Small: ~1M params / ~5.8M samples = 0.17 params/sample. V3 (0.023, no overfit) < V5-Small (0.17) < V4 (0.82, overfit epoch 2). Safe zone.

**Q19: How long will one training epoch take?**
Estimated ~2 hours based on V5-Small config (d_model=128, batch_size=512, ~5.8M samples). Unit test showed 86% GPU util — good throughput. This allows 3-4 experiments per week.

**Q20: Could we start even smaller?**
Possible but unnecessary. V5-Small at ~1M params is already 5x smaller than V5-Large. Going smaller (e.g., d_model=64) risks too little capacity to learn temporal patterns. V5-Small is the right starting point.

**Q21: Is the data format correct?**
Yes. (n, 5, 128, 20) tokens verified. Labels: y_return (10-day forward), y_big (>5% threshold). 5-day lookback for 10-day prediction is reasonable. Unit test validated forward pass on real chunks.

**Q22: Is data preparation sufficiently optimized?**
Yes. Measured 125-250 files/sec on medium quarters (faster than V4). 47/64 done in ~45 min. Throughput acceptable. See `research/v5_data_prep_performance.md`.

**Q23: Is there a complete design document?**
Yes. `research/v5_design.md` (full architecture), `research/v5_design_review.md` (V5-Small recommendation with params/sample analysis). Both complete.

---

## Phase 6: What Should We Do Next?

**Q24: What are ALL possible actions right now?**
1. Wait for data prep to finish (~15 min), then run V5 smoke test (3 epochs, ~6hr)
2. Wait for data, then run V5 full training directly (skip smoke test)
3. Investigate further — review V5 model code quality, loss function weights
4. Update goal_tracker and research docs
5. Do nothing — CPU is busy, dev just finished, GPU can't train yet

**Q25: Which resources are idle?**
- CPU: BUSY (token prep 47/64)
- GPU: IDLE (0%) — but no training possible until data finishes
- Dev: IDLE — just completed all tasks

**Q26: Can any actions run in parallel?**
Dev could review/improve V5 code while data finishes. But all code is already implemented and unit-tested. Dev has nothing productive to do until data completes. This is acceptable — data finishes in ~15 min.

**Q27: Which action has the highest expected value?**
Wait ~15 min for data, then run V5 smoke test (3 epochs). This validates the temporal hypothesis with real metrics. Cost: ~6hr GPU. Information value: HIGH (first V5 training results).

**Q28: Is this addressing the constraint?**
Yes — the constraint is data completion, which resolves itself. The next action (smoke test) directly tests the hypothesis.

---

## Phase 7: Is This Action Ready to Execute?

**Q29: Data pipeline task validation?**
N/A — data prep is already running and nearly complete. No new pipeline task.

**Q30: Training task prerequisites?**
- Unit test passed? ✓ (GPU 86%, loss=0.6262, all 9 checkpoint fields)
- Smoke test with GPU util >70%? NOT YET — this IS the smoke test we're recommending
- AMP+FP16? ✓ (confirmed in unit test)
- torch.compile? ✓
- Checkpoint/resume? ✓ (9 fields, per-run dir)
- Design doc? ✓ (research/v5_design.md + v5_design_review.md)
- **All prerequisites for smoke test are met.** Smoke test IS the next validation step.

**Q31: Design doc complete?**
Yes. ✓

**Q32: Does this chain to a follow-up?**
Yes — after smoke test: if metrics > V4 baseline (CR > 0.0187) → launch full training. If worse → investigate (overfitting? data issue? architecture?).

**Q33: Success criteria?**
- V5-Small smoke test (3 epochs): loss decreasing, no NaN
- GPU util >70% sustained
- CR after 3 epochs: ideally > 0.01 (showing signal, even if not converged)
- No overfitting in 3 epochs (V4 overfit by epoch 2 — V5-Small should not)

**Q34: Failure criteria?**
- Loss NaN or exploding → check data, lr, model init
- GPU util <50% → tune ChunkLoader for 5-day format
- CR negative or zero after 3 epochs → architecture may not work, investigate
- Overfitting by epoch 2 (like V4) → model still too large, reduce further

---

## Phase 8: After Taking Action

**Q35-Q38: Deferred** — will evaluate after smoke test completes.

---

## Plan

1. **Wait ~15 min** for V5 token preparation to complete (47/64 → 64/64)
2. **Launch V5-Small smoke test** (3 epochs) with nohup once data is ready
3. **Monitor**: GPU util, loss trajectory, CR per epoch
4. **Evaluate**: Compare epoch 1-3 metrics against V4 baseline (CR=0.0187)
5. **Decision after smoke test**: if CR > 0.0187 → launch full training. If not → investigate.

## Recommended Prompt

```
V5-Small is fully implemented and unit-tested. Token prep is nearly complete (47/64 quarters). When token prep finishes:

1. Verify data complete: cat firstrate_learning_v5/cache/tokens/meta.json — should show all splits with n_chunks and n_samples.

2. Clean up old unit test run: rm -rf firstrate_learning_v5/models/run_*_unit

3. Launch V5-Small smoke test (3 epochs):
   nohup firstrate_learning/.venv/bin/python -u -m firstrate_learning_v5.train --smoke-test > firstrate_learning_v5/output/train_v5_smoke.log 2>&1 &

4. Verify it started: check PID, tail log for first batch output. Confirm GPU util >70% after batch 100.

5. Monitor every epoch: report val metrics (captured return, P@5%, return corr, precision, recall).

6. After 3 epochs complete, report the full comparison table vs V4:
   | Metric | V4 (ep1) | V5-Small (ep1) | V5-Small (ep3) |

7. Do NOT launch --full yet. Report results first so we can evaluate.

Success: loss decreasing, no NaN, GPU util >70%, CR > 0 after epoch 1.
Failure: NaN, GPU <50%, CR negative → stop and report.
```
