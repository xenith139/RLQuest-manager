# Manager Review — 2026-03-26 (Updated)

## Observation

Dev is idle at prompt. Background task running:
- **Script:** `firstrate_learning_v4/prepare_tokens.py` (PID 1912098)
- **Progress:** Quarter 7/40 train (+ 12 val + 12 test = 64 total), ~715K samples
- **ETA reported by dev:** 3-4 hours
- **Runtime behavior:**
  - CPU: 98.2% on **1 core** — 20 cores available, 19 idle
  - Memory: 21.4GB RSS (33.4% of 62GB) — dev reported 5.3GB earlier, significant growth
  - No GPU usage (N/A for this task)

## Decision Tree Outcome

```
ACTIVE CONVERSATION → RUNNING A SCRIPT → LONG TASK (hours+)
  → Not in verified_scripts.json
  → Investigate optimization urgently
  → No pre-run validation cycle was performed (no smoke test, no perf test)
  → Script fails multiple optimization categories → STOP & OPTIMIZE
```

## Optimization Failures (per long_running_script_guide.md)

| Category | Status | Detail |
|----------|--------|--------|
| 1. Parallelization | FAIL | `ThreadPoolExecutor` imported but never used. Single-core processing on 20-core machine. ~19x potential speedup wasted. |
| 2. GPU | N/A | Data prep task, CPU-appropriate |
| 3. Caching/Serialization | UNKNOWN | Need to verify zstd vs gzip, mmap usage |
| 4. Resumability | UNKNOWN | Need to verify status.json checkpointing |
| 5. Memory | WARNING | 21.4GB and growing. Dev reported 5.3GB — discrepancy suggests accumulation without cleanup |
| 6. Vectorization | UNKNOWN | Need script review |
| 7. Progress monitoring | PARTIAL | Has logging but no ProgressTracker `_progress.md` |
| 8. Pre-run validation | FAIL | No smoke test was run. No performance test was run. Full dataset launched directly without validation cycle. |

**Verdict: STOP & OPTIMIZE** — Single-core bottleneck + no pre-run validation. With 20 cores, parallelizing could reduce 3-4 hours to ~15-20 minutes. The script must go through the mandatory smoke test → performance test → iterate cycle before any full run.

## Recommended Prompt to Dev

```
STOP the current prepare_tokens.py run (kill PID 1912098). It is running single-threaded (98.2% on 1 of 20 cores) with a 3-4 hour ETA, and was launched without the mandatory pre-run validation cycle.

Optimize prepare_tokens.py with these changes:

1. **Add `--smoke-test` flag** — must run on 1-2 quarters, ~100 symbols, complete in <2 min. This is now a project requirement for all data pipeline scripts.

2. **Parallelization (critical):** ThreadPoolExecutor is imported but never used. Implement ProcessPoolExecutor for CPU-bound file processing across quarters — 20 cores available. Use `imap_unordered` with dynamic chunk sizing: `max(50, num_files // (num_workers * 4))`. Add `--workers` CLI flag.

3. **Memory:** RSS is 21.4GB and growing (you reported 5.3GB). Check for accumulated data structures not being freed. Use explicit `del` after processing each quarter. Consider mmap for price data reads.

4. **ProgressTracker:** Add `from progress.progress import ProgressTracker` for live `_progress.md` output.

5. **Resumability:** Add `status.json` checkpointing so completed quarters are skipped on restart.

6. **Compression:** Verify using zstd (not gzip) for output files.

After implementing, follow the MANDATORY pre-run validation cycle:

  a) Run `--smoke-test` — verify correctness and output format
  b) Run on ~5 quarters — measure CPU% (must show multi-core), memory (must be stable), throughput rate
  c) Iterate: optimize → smoke test → perf test at least 2-3 times
  d) Verify checkpointing works (kill and restart, confirm resume)
  e) Only then launch the full run

Reference implementations: `firstrate_learning/prepare_chunks.py` and `firstrate_learning/prepare_data.py`.

Check CLAUDE.md and .claude/rules/data-pipeline.md for the full project standards — these have been updated with pre-run validation requirements.
```

## Config Changes

Rules already updated in this session:
- `RLQuest/CLAUDE.md` — added "Pre-Run Validation — MANDATORY" section
- `RLQuest/.claude/rules/data-pipeline.md` — added pre-run validation cycle requirement
- `manager/decision_tree.md` — added "ABOUT TO RUN A NEW SCRIPT" branch and pre-run expectation
- `manager/long_running_script_guide.md` — added §8: Pre-Run Validation cycle

No further config changes needed. Dev should now see these rules automatically.
