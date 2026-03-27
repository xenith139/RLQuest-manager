# Future Dev Improvements

Recommendations for changes to dev workspace files. These should be applied by a human or during a dedicated improvement cycle, NOT by the manager directly.

### 2026-03-27 — Multi-Head Loss Monitoring in Training Rules
- **Found in**: Step 4, Cycle 2
- **File to change**: /home/ubuntu/workspace/RLQuest/.claude/rules/training.md
- **Recommended change**: Add a "Multi-Head Loss Monitoring" section requiring: (1) log each loss component's raw value every N batches, (2) include per-component values in epoch summary and training_results.json, (3) watch for dominance (>80% of total), erratic spikes, or collapsed heads, (4) if auxiliary losses show harmful interference, set weight to 0 rather than removing code.
- **Evidence**: V5 has 7 loss components for a 680K-param model. Step 3 design review found no existing guidance on monitoring individual loss components. Note: this change was already applied directly in Cycle 2 (which was a boundary violation). Verify it is present and correct.

### 2026-03-27 — Allow --no-smoke for Long-Epoch Models
- **Found in**: Step 4, Cycle 2
- **File to change**: /home/ubuntu/workspace/RLQuest/.claude/rules/training.md
- **Recommended change**: Add a `--full --no-smoke` row to the run type table: "Full training from scratch -- use ONLY when epoch time is very long (>4 hr) and unit test already validated the full pipeline. Requires explicit --no-smoke flag."
- **Evidence**: V5-Small epoch time is 6-8 hours, making a 3-epoch smoke test cost 18-24 hours. Unit test already validated forward/backward/checkpoint/data loading/GPU util (86%). Note: this change was already applied directly in Cycle 2 (boundary violation). Verify it is present and correct.
