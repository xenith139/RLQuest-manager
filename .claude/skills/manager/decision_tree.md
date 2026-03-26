# Manager Decision Tree

High-level decision tree for the manager when monitoring dev (Claude tmux session 1).

## Decision Flow

```
[Check Dev Session]
        │
        ├── NOT RUNNING ──► Start session ──► Resume latest conversation
        │
        └── RUNNING ──► Capture pane output
                │
                ├── FRESH/CLEAR PROMPT ──► Resume latest conversation
                │
                └── ACTIVE CONVERSATION ──► Analyze what dev is doing
                        │
                        ├── IDLE (waiting at prompt) ──► Report status to user
                        │
                        ├── RUNNING A SCRIPT/TASK ──► Evaluate duration & complexity
                        │       │
                        │       ├── SHORT TASK (< 1 min expected) ──► Wait for completion
                        │       │
                        │       ├── MEDIUM TASK (1-30 min) ──► Check optimization
                        │       │       │
                        │       │       └── See: long_running_script_guide.md
                        │       │
                        │       └── LONG TASK (30+ min / hours) ──► Investigate optimization urgently
                        │               │
                        │               └── See: long_running_script_guide.md
                        │
                        ├── ABOUT TO RUN A NEW SCRIPT ──► Verify pre-run validation
                        │       │
                        │       ├── DATA PIPELINE (prepare_*/process_*):
                        │       │       ├── Was smoke test run? ──► If NO: stop, require it
                        │       │       ├── Was CPU multi-core verified? ──► If NO: stop
                        │       │       ├── Was memory stable? ──► If NO: stop
                        │       │       └── All passed? ──► Allow full run
                        │       │               See: long_running_script_guide.md §8
                        │       │
                        │       └── TRAINING SCRIPT (train*.py):
                        │               ├── Was --smoke-test run (3 epochs)? ──► If NO: stop
                        │               ├── Was GPU util measured (must be >70%)? ──► If NO: stop
                        │               ├── Was data/gpu timing logged? ──► If NO: add instrumentation
                        │               ├── Is ProgressTracker implemented? ──► If NO: add it
                        │               ├── Is ChunkLoader used (not MapDataset)? ──► If NO: fix
                        │               ├── Is AMP + FP16 enabled? ──► MANDATORY, if NO: enable immediately
                        │               ├── Is torch.compile used (transformers)? ──► If NO: enable
                        │               ├── Does it create per-run directory (run_YYYYMMDD_type/)? ──► If NO: add
                        │               ├── Does it save latest_checkpoint.pt EVERY epoch in run dir? ──► If NO: add
                        │               ├── Does checkpoint include optimizer+scaler+scheduler state? ──► If NO: fix
                        │               ├── Does it support --resume from latest run dir? ──► If NO: add
                        │               ├── Does it support --unit-test (<2min, 50 batches)? ──► If NO: add
                        │               └── All passed? ──► Allow full training
                        │                       See: training_evaluation_guide.md §Before Training
                        │
                        ├── IN CONVERSATION (responding/thinking) ──► Do not interrupt, report status
                        │
                        └── (ALWAYS) EVALUATE WORK ALIGNMENT ──► Goal tracking
                                │
                                ├── IDENTIFY what dev is working on (from pane output)
                                │       What files edited? What scripts running? Conversation topic?
                                │
                                ├── MAP TO PROJECT GOALS (from goal_tracker.md + goals.md)
                                │       │
                                │       ├── V4 TOKEN PREP ──► Data pipeline goal
                                │       ├── V4 TRAINING ──► Backbone training goal
                                │       │       └── See: training_evaluation_guide.md
                                │       ├── V4 EVALUATION ──► Model comparison goal
                                │       │       └── See: training_evaluation_guide.md
                                │       ├── V3/V2 COMPARISON ──► Benchmarking goal
                                │       ├── PORTFOLIO MODEL ──► Downstream model goal
                                │       ├── INFRASTRUCTURE / TOOLING ──► Support goal
                                │       └── UNKNOWN / OFF-TRACK ──► Investigate, may need redirect
                                │
                                ├── EVALUATE GOAL PROGRESS
                                │       │
                                │       ├── Training in progress?
                                │       │       Check loss curves, val metrics, GPU util
                                │       │       Compare to V3 baseline thresholds
                                │       │       See: training_evaluation_guide.md
                                │       │
                                │       ├── Data pipeline running?
                                │       │       Check progress through quarters, output quality
                                │       │       Check status.json completeness
                                │       │
                                │       └── Evaluation/testing?
                                │               Check metrics vs previous versions
                                │               Compute foresight gap reduction
                                │
                                ├── UPDATE goal_tracker.md
                                │       Update current goal, status, metrics, next goals
                                │
                                └── PLAN NEXT GOAL (if dev is idle or current goal complete)
                                        │
                                        ├── Current goal complete? ──► Identify next priority
                                        │       from direction.md and goal_tracker.md
                                        ├── Current goal stalled? ──► Suggest alternative approach
                                        └── Current goal failing? ──► Recommend pivot or debugging
```

## Long-Running Script Detection

When dev reports a script is running, evaluate the expected duration:

| Duration | Action |
|----------|--------|
| < 1 minute | Let it run, no intervention needed |
| 1-30 minutes | Question optimization — is this the best it can be? Refer to `long_running_script_guide.md` |
| 30+ minutes | Urgently investigate optimization. Likely candidate for parallelization, GPU, or caching improvements. Refer to `long_running_script_guide.md` |
| Hours+ | Strong signal the script may be suboptimal. Check `verified_scripts.json` first — if not already verified, instruct dev to stop and optimize before resuming |

## Script Verification Tracking

The manager maintains a registry of verified scripts in:
**`/home/ubuntu/workspace/RLQuest-manager/.claude/skills/manager/verified_scripts.json`**

Before investigating a long-running script:
1. Check `verified_scripts.json` to see if this script has already been verified as fully optimized.
2. If verified, **do not blindly trust the previous result** — also check current runtime behavior:
   - CPU/GPU utilization (`htop`, `nvidia-smi`)
   - Memory usage (`free -h`, process RSS)
   - I/O throughput (is disk the bottleneck?)
3. If the runtime behavior matches expectations from the verified entry, let it run.
4. If behavior deviates (higher memory, lower CPU utilization, slower throughput), re-investigate.

## Pre-Run Validation Expectation

The manager expects dev to follow this cycle BEFORE any full-scale run:

1. **Smoke test** (`--smoke-test`) on 1-2 quarters — must pass
2. **Performance test** on ~5 quarters — measure CPU, memory, throughput
3. **Iterate 2-3 times** — optimize → test → measure → optimize
4. **Full run only** when multi-core active, memory stable, checkpointing verified

If the manager observes dev launching a full run without evidence of this cycle:
- Check tmux history for smoke test runs
- Check if `--smoke-test` flag exists in the script
- If no validation was done, instruct dev to stop and run the cycle first

See: `long_running_script_guide.md` §8 for full details.

## When to Interrupt Dev

**DO interrupt** if:
- Script is running for hours with no parallelization
- Single-core CPU usage when multi-core is available
- No caching/checkpointing (a crash means starting over)
- Loading full datasets into memory when mmap is possible
- Using gzip when zstd would be faster
- Dev launched full run without smoke test / performance validation cycle
- Python loops over large arrays instead of vectorized numpy
- Dev skips a checkpoint expectation from the manager's instructions
- Dev ignores or misinterprets a manager correction

**Intervention severity levels:**

| Severity | Trigger | Action |
|----------|---------|--------|
| **Minor** | Wrong order, slightly off approach | Brief nudge via tmux: "Note: please also [step]" |
| **Major** | Skipped critical step (smoke test, parallelization) | Firm correction: "STOP. You skipped [step]. This is mandatory." |
| **Critical** | About to overwrite data without backup, full run without validation | Immediate stop: "STOP IMMEDIATELY. Do not proceed." |

After any intervention: wait 10-15s, verify dev acknowledged. If ignored, escalate. If ignored twice, report to user and pause.

## Active Monitoring Integration

During the "send" command monitoring loop, the manager MUST apply this decision tree on **every check**, not just the initial assessment. Specifically:

1. On each monitoring check, evaluate dev's current activity against:
   - The checkpoint expectations list (from `manager.md`)
   - The compliance review branch (Unexpected Dev Behavior section below)
   - The long_running_script_guide.md optimization checklist
2. If a violation is detected, intervene immediately using the severity levels above
3. If a config gap is found (rule missing/vague), fix the RLQuest `.claude/` files immediately
4. Log all interventions to `manager.md`

**DO NOT interrupt** if:
- Script is in `verified_scripts.json` AND runtime behavior matches
- Dev is actively debugging or mid-conversation about the script
- Script is nearly complete (>90% progress)

---

## Unexpected Dev Behavior — Compliance Review

When the manager observes dev NOT following expected standards (from rules, skills, or CLAUDE.md), trigger this decision branch:

```
[Dev output shows unexpected behavior]
        │
        ├── IDENTIFY THE VIOLATION
        │       What standard was not followed?
        │       (e.g., no parallelization, gzip instead of zstd, no ProgressTracker,
        │        sys.path hacks, silent try-except, no checkpointing, float64, etc.)
        │
        ├── DIAGNOSE WHY DEV DIDN'T COMPLY
        │       │
        │       ├── Read RLQuest/.claude/ config that should have enforced this:
        │       │       • CLAUDE.md — is the rule stated clearly?
        │       │       • .claude/rules/ — does a path-scoped rule cover this file?
        │       │       • .claude/skills/all/ — does the relevant details_*.md exist?
        │       │
        │       ├── Possible root causes:
        │       │       │
        │       │       ├── RULE MISSING — the standard isn't in CLAUDE.md or rules/
        │       │       │       └── Fix: Add the rule to appropriate file
        │       │       │
        │       │       ├── RULE EXISTS BUT WRONG PATH SCOPE — rule file has paths
        │       │       │   that don't match the file dev was editing
        │       │       │       └── Fix: Update paths glob in rule frontmatter
        │       │       │
        │       │       ├── RULE TOO VAGUE — rule exists but wording is ambiguous,
        │       │       │   dev interpreted it differently
        │       │       │       └── Fix: Make directive more specific with concrete examples
        │       │       │
        │       │       ├── RULE CONTRADICTS ANOTHER — two rules give conflicting guidance
        │       │       │       └── Fix: Resolve conflict, update both files
        │       │       │
        │       │       ├── SKILL DETAIL NOT LOADED — the detailed pattern is only in
        │       │       │   skills/all/details_*.md (on-demand), not in rules/ (auto-loaded)
        │       │       │       └── Fix: Extract key directive into rules/ so it auto-loads
        │       │       │
        │       │       └── DEV CONTEXT ISSUE — dev was in a conversation context where
        │       │           the rule was compressed/truncated from context window
        │       │               └── Fix: Ensure critical rules are in CLAUDE.md (always loaded)
        │       │                   and rules/ (auto-loaded), not only in skills (on-demand)
        │
        ├── FIX THE CONFIGURATION
        │       │
        │       ├── Update the relevant file(s) in RLQuest/:
        │       │       • CLAUDE.md — add/clarify the mandate
        │       │       • .claude/rules/*.md — add/update path-scoped rule
        │       │       • .claude/skills/all/details_*.md — add/update reference examples
        │       │
        │       └── Verify the fix:
        │               • Rule path globs match the target files
        │               • Wording is directive (MUST/NEVER), not suggestive
        │               • Concrete examples of correct vs incorrect approach
        │
        └── CORRECT DEV'S CURRENT WORK
                │
                ├── Send dev a message via tmux_send_claude.sh explaining:
                │       1. What the violation is
                │       2. What the correct approach should be
                │       3. Reference the specific rule/guide
                │
                └── Monitor dev's response to confirm compliance
```

### Compliance Review Procedure

1. **Detect**: While reviewing dev's pane output, compare behavior against:
   - `long_running_script_guide.md` — optimization checklist
   - `RLQuest/CLAUDE.md` — project mandates
   - `RLQuest/.claude/rules/data-pipeline.md` — data processing standards
   - `RLQuest/.claude/rules/training.md` — training standards
   - `RLQuest/.claude/rules/general.md` — universal standards

2. **Read the config**: Before assuming dev is wrong, read the actual config files in `RLQuest/.claude/` to verify the rule exists, is correctly scoped, and is clearly worded.

3. **Root cause**: Determine if the failure is in dev's behavior or in the configuration:
   - If config is missing/vague → fix the config first, then correct dev
   - If config is correct and clear → correct dev, note the pattern for future monitoring

4. **Fix config files**: Update the appropriate file(s) in `/home/ubuntu/workspace/RLQuest/`:
   - `CLAUDE.md` for universal mandates (always loaded)
   - `.claude/rules/*.md` for path-scoped auto-loaded rules
   - `.claude/skills/all/details_*.md` for detailed reference patterns

5. **Correct dev**: Use `tmux_send_claude.sh 1 "message"` to inform dev of the correct approach.

6. **Log**: Record what was found and fixed so the same gap doesn't recur. If a new pattern emerges that isn't covered by any existing guide, create or update the relevant guide.

### Examples of Violations to Watch For

| Observed Behavior | Expected Standard | Config to Check |
|---|---|---|
| Dev writes `for row in data:` loop over 100K+ rows | Vectorized numpy operations | `rules/data-pipeline.md` §1, `CLAUDE.md` §Data Processing |
| Dev uses `gzip` for new chunk files | Zstandard `.pt.zst` | `rules/data-pipeline.md` §3, `details_data-processing.md` §4 |
| Dev runs script without ProgressTracker | ProgressTracker required >30s | `rules/general.md` §Long-Running, `details_long-running-progress.md` |
| Dev uses `python3` instead of venv | `firstrate_learning/.venv/bin/python` | `CLAUDE.md` §Python Environment, `rules/general.md` §1 |
| Dev commits without double-confirm | Double-confirm required | `rules/general.md` §Safety, `details_coding-agent-actions.md` |
| Dev uses `sys.path.insert` | Package imports via `pip install -e` | `CLAUDE.md` §Python Environment, `details_python-imports.md` |
| Dev stores normalized data in chunks | Runtime normalization only | `rules/data-pipeline.md` §6, `rules/training.md` §Normalization |
| No `status.json` checkpointing | Resumable for scripts >5 min | `rules/data-pipeline.md` §5, `details_data-processing.md` §8 |
| Silent `try-except` that continues | Fail-fast-loud, explicit errors | `rules/general.md` §Error Handling, `details_fail-fast-loud.md` |
| Training without AMP/FP16 | AMP + FP16 is mandatory for all training | `rules/training.md` §Mixed Precision, `CLAUDE.md` §Training |
| Training without torch.compile | torch.compile mandatory for transformers | `rules/training.md` §torch.compile |
| Training without GPU timing instrumentation | Must log data_ms, gpu_ms, util% | `rules/training.md` §GPU Utilization |
| Training without per-run directory | Must create models/run_YYYYMMDD_type/ per run | `rules/training.md` §Per-Run Directory |
| Training without every-epoch checkpoint | Must save latest_checkpoint.pt every epoch in run dir | `rules/training.md` §Checkpointing & Resume |
| Checkpoint missing optimizer/scaler/scheduler state | All state must be saved for proper resume | `rules/training.md` §Checkpoint Contents |
| Training without --resume flag | Must support resume from latest run dir | `rules/training.md` §Resume from Checkpoint |
| Training without --unit-test flag | Must support <2min unit test (1 epoch, 50 batches) | `rules/training.md` §Run Types |
| Model files from different runs mixed in one folder | Each run must have isolated directory | `rules/training.md` §Per-Run Directory |
