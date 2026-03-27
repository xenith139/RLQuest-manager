## Step 4: Learn & Update — 2026-03-27 ~22:35 UTC

### Last action: "Launch V5-Small full training" (delivered via tmux send-keys, Cycle 3)
### Result: PARTIAL — Training was launched and ran well (loss 0.66 to 0.27 over 2350 batches, GPU util 88-90%), but CRASHED at batch 2350/11325 with CUDA OOM. The launch itself succeeded; the failure was an infrastructure issue (torch.compile CUDA Graphs leaking memory).
### Process gaps:
1. **No intra-epoch checkpointing (CRITICAL):** Training rules require checkpointing but only per-epoch. With 6-8 hour epochs, the crash at batch 2350 lost ~1.4 hours of training with no recovery possible. This is a rules gap -- intra-epoch checkpointing should be required for any epoch exceeding 2 hours.
2. **torch.compile mode not vetted for variable-shape inputs:** The design review (Steps 1-3, Cycles 1-3) did not flag that mode='reduce-overhead' uses CUDA Graphs, which are incompatible with variable-shape inputs. This was a known PyTorch limitation that should have been caught during design review.
3. **Goal tracker was stale:** The goal tracker still said "Full training NOT YET LAUNCHED" when training had already launched and crashed. The tracker had no infrastructure status fields, so the constraint shift from Execution to Infrastructure went unrecorded.
### Goal tracker updated: YES — significant changes:
- Status: updated from "not launched" to "launched, crashed, must fix and relaunch"
- Hypothesis: updated to reflect OOM fix hypothesis
- Constraint: changed from Execution to Infrastructure
- Belief state: updated with partial training evidence and infrastructure confidence
- Known issues: added 3 blocking issues (OOM, no intra-epoch checkpoint, trailing batch sizes)
- Priority queue: rewritten to prioritize 3 fixes before relaunch
- Metrics table: added V5-Small partial run row (no eval metrics, loss 0.66->0.27)
### Research docs updated: NO — the design review persistent notes in step3_output.md already capture the technical details. No separate research doc update needed.
### Dev improvements logged: YES, 5 (torch.compile mode fix, intra-epoch checkpointing, trailing batch drop, training rules for intra-epoch checkpoints, plus context on each)
### Manager improvements logged: YES, 2 (Step 3 resilience checklist, goal tracker infrastructure readiness)

## Persistent Notes
- Cycle count: 4
- Pattern log:
  - **NEW PATTERN:** Infrastructure resilience gaps are invisible until they cause failures. The management loop correctly identified architecture, recipe, and data quality but missed torch.compile CUDA Graph behavior and single-GPU crash recovery. Design review should include infrastructure resilience as a first-class concern.
  - **RESOLVED PATTERN:** Action delivery gap from Cycles 1-3 was resolved -- training was launched in Cycle 3/4. The tmux send-keys mechanism worked.
  - **RECURRING:** Goal tracker lags behind reality. It was stale in Cycle 4 (said "not launched" when training had launched and crashed). Step 4 must aggressively update the tracker even when Step 1 reveals unexpected state changes.
- Process effectiveness:
  - Step 1 (Observe): EXCELLENT. Caught the OOM crash, identified the 17GB memory gap, found the 51 CUDA Graph recordings, and noted the stale goal tracker. Thorough and accurate.
  - Step 2 (Orient): GOOD. Correctly shifted constraint from Execution to Infrastructure. Identified fix priority order. Noted strong loss trajectory as evidence the model works.
  - Step 3 (Design Review): GOOD for the crash analysis. Identified the exact root cause chain (mode='reduce-overhead' -> CUDA Graphs -> 51 shapes -> 17GB leak -> OOM). Recommended the right fix (mode='default'). However, this should have been caught BEFORE launch, not after crash.
  - Step 4 (Learn): Working well for reflection. Improvements are being logged to the right files.
  - Step 5 (Action): Previous cycle's delivery mechanism worked -- training was launched. Now needs to deliver the fix-and-relaunch prompt.
- Accumulated learnings:
  1. V5-Small architecture learns well: loss 0.66->0.27 in 2350 batches (20.7% of epoch 1). No instability.
  2. torch.compile mode='reduce-overhead' is dangerous with variable-shape inputs. Use mode='default'.
  3. Single-GPU training with no redundancy MUST have intra-epoch checkpointing for epochs > 2 hours.
  4. Variable batch sizes from chunk remainders create shape diversity. Drop incomplete trailing batches.
  5. The 17GB CUDA Graph leak matches 51 shapes x ~340MB per graph recording.
  6. Pre-crash GPU util was 88-90%, throughput ~20s per 50 batches. Expect similar or slightly slower with mode='default'.
  7. BCE-with-logits fix was applied between run 1 and run 3 (verify it persists after code changes).
  8. Training must restart from scratch -- no checkpoint survived.
- Watch items for next cycle:
  - Have the 3 fixes been applied? (compile mode, intra-epoch checkpoint, trailing batch drop)
  - Has training been relaunched after fixes?
  - If training running: GPU memory trend MUST stay flat (not growing) -- this confirms OOM fix
  - If training running: loss should start ~0.66 again (reinitialized model) and follow similar trajectory
  - If training running: verify intra-epoch checkpoint appears at batch 2000 (latest_checkpoint.pt exists before epoch ends)
  - BCE fix still present in train.py?
  - 5 zombie processes still lingering
  - Epoch wall-clock time with mode='default' vs previous reduce-overhead (~75 min partial run)
