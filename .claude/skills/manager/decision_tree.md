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
                        └── IN CONVERSATION (responding/thinking) ──► Do not interrupt, report status
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

## When to Interrupt Dev

**DO interrupt** if:
- Script is running for hours with no parallelization
- Single-core CPU usage when multi-core is available
- No caching/checkpointing (a crash means starting over)
- Loading full datasets into memory when mmap is possible
- Using gzip when zstd would be faster
- Python loops over large arrays instead of vectorized numpy

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
