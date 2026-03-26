# Manager & Developer Improvements Log

Living document updated continuously during all developer interactions and monitoring. Identifies gaps and potential improvements to manager skills, developer skills, rules, and workflows.

---

## 2026-03-26 — V4 Training Launch Monitoring

### Observations

**What went well:**
- Dev immediately read CLAUDE.md and rules/training.md after receiving prompt — rules auto-loaded correctly
- Dev implemented all 8 mandatory requirements in order without skipping any
- GPU timing instrumentation added correctly — util% logged every 50 batches (reference from trade_learning worked)
- AMP+FP16 was already partially in V4 train.py (had `use_amp` config flag) — dev kept it enabled
- Smoke test shows excellent GPU util (95%) and low data loading overhead (5.4%)
- ChunkLoader caching warmup visible: data latency dropped from 259ms → 26ms as cache warmed

**Potential improvements identified:**

1. **Manager prompt length** — The send prompt was very long (single block of text). Dev handled it well, but for complex multi-step tasks, consider breaking into numbered phases sent sequentially (Phase 1: implement, Phase 2: smoke test, Phase 3: full run). This would reduce context overload and make checkpoint tracking cleaner.

2. **V4 train.py already had partial AMP** — The `use_amp` config flag existed but the manager prompt said "MANDATORY NO EXCEPTIONS" as if it was missing. The manager should check the current state of the target script before crafting the prompt to avoid redundant instructions. Could add a pre-send step: "Read the target script to identify what's already implemented."

3. **ProgressTracker wasn't in V3 or V4 train.py** — Only V1 had it. The rules/training.md now mandates it, but this gap existed for 2+ versions. The developer skill `details_long-running-progress.md` should explicitly list training scripts as requiring ProgressTracker, not just data pipeline scripts.

4. **GPU timing instrumentation wasn't in any training script except trade_learning** — The pattern existed but wasn't propagated to firstrate_learning versions. Consider adding a skill or rule that requires GPU timing in ALL scripts that use CUDA, not just training scripts.

5. **Smoke test warmup period** — GPU util started at 67% (batch 50) and climbed to 95% (batch 500). The manager's 70% threshold could cause a false alarm if checked only at the start. Consider: "GPU util must be >70% after warmup (batch 100+), not at first batch."

6. **ChunkInterleavedDataset vs ChunkIterableDataset** — The manager prompt specified both, but it's unclear if dev used ChunkInterleavedDataset for training specifically (vs just ChunkIterableDataset for everything). The manager should verify this specific pattern in a milestone check.

7. **Log file naming** — Dev ran smoke test with `2>&1` inline instead of the mandated `nohup ... > script.log 2>&1 &` pattern. This is acceptable for smoke tests (short, interactive), but the rules don't distinguish between smoke test execution (interactive, inline) and full run execution (nohup, background). Consider clarifying in rules.

### Improvement Candidates

| Area | Current Gap | Proposed Fix | Priority |
|------|-------------|--------------|----------|
| Manager pre-send | Doesn't check existing code before crafting prompt | Add step: read target files to identify what's already done | Medium |
| Smoke test warmup | 70% threshold checked from batch 0 | Amend: measure after warmup (batch 100+) | Low |
| ProgressTracker in training | Only V1 had it, V3/V4 didn't | Already fixed in rules — monitor compliance | Done |
| GPU timing in training | Only trade_learning had it | Already fixed in rules — monitor compliance | Done |
| Prompt structure | Long single block | Consider phased prompts for complex tasks | Medium |
| Smoke test vs full run rules | Rules don't distinguish inline vs nohup | Clarify in rules: smoke test can be inline | Low |
| Smoke test timeout | Smoke test hit 10m timeout in Claude, had to background | Consider: smoke test for training should use nohup too if >5 min. Or adjust timeout expectations. 3-epoch smoke test with 11.4M samples takes ~15-20 min. | Medium |
| Sleep-based polling | Dev used `sleep 1800` to wait for background task — wasteful | Suggest: dev should use shorter poll intervals or check progress file | Low |
| Smoke test duration | 3-epoch smoke test on 11.4M samples takes ~60 min, not "quick" | Consider: smoke test should limit data (e.g., 2 train chunks, not all 138), or use --smoke-test to cap batches/epoch | High |
| Manager monitoring cost | Manager has been polling for 1h+ during smoke test — many token-consuming checks | For known long waits, manager should use only `ps` checks until process exits, then do one full pane capture | Medium |
| Checkpoint/resume gap | V4 train.py saves only model weights — no optimizer/scaler/scheduler/patience. Resume is impossible. V3 same issue. Only V1 and trade_learning have proper resume. | Now fixed in rules. Manager must verify checkpoint contents and --resume flag before allowing full training. | Critical — fixed |
| Every-epoch save missing | All versions only save on best val improvement. If training runs 10 epochs with no improvement then crashes, 10 epochs of training state is lost. | Now required in rules: `latest_checkpoint.pt` saved every epoch. | Critical — fixed |
