# Future Manager Improvements

Recommendations for changes to manager process files (.claude/skills/manager/step*.md). These should be applied by a human or during a dedicated improvement cycle, NOT by the manager directly.

### 2026-03-27 — [MANAGER] Action plans must include task chaining
- **Found in**: Step 4, Cycle 2
- **Step file to change**: step5_action_plan.md (or equivalent Step 5 instructions)
- **Recommended change**: Require every action plan to include explicit "On success: [next immediate action]" and "On failure: [diagnostic steps]" sections. The dev prompt should never end without a chained next step.
- **Evidence**: After unit test succeeded at 18:57 UTC, dev sat idle for 3+ hours because no follow-up command was queued. The action plan ended at "run unit test" with no chaining instruction.

### 2026-03-27 — [MANAGER] Action delivery verification mechanism
- **Found in**: Step 4, Cycle 3
- **Step file to change**: step5_action_plan.md (or equivalent Step 5 instructions) and SKILL.md orchestration logic
- **Recommended change**: After Step 5 generates the dev prompt, there must be an explicit "deliver prompt to dev" step that sends the prompt to the dev tmux session (e.g., via tmux send-keys) and then verifies delivery (e.g., checking that the dev agent is no longer idle). Without this, the management loop generates correct plans that never execute.
- **Evidence**: The management loop has now completed 3 consecutive cycles all concluding "launch training immediately," but training has not been launched. The Step 5 prompt is written to step5_output.md but never delivered. Two full OODA loops produced zero state change because the action output was not connected to the action executor.

### 2026-03-27 — [MANAGER] Stagnation detection and escalation
- **Found in**: Step 4, Cycle 3
- **Step file to change**: step4_learn_update.md
- **Recommended change**: Add a stagnation check: if the same constraint persists for 3+ consecutive cycles with no state change, Step 4 should flag this as STAGNATION and recommend escalation (e.g., alert the human operator, attempt direct action delivery, or change approach). Currently, the loop can spin indefinitely generating the same recommendation without progress.
- **Evidence**: Constraint has been "Execution" for 3 cycles. Status has been "dev idle, GPU idle, launch training" for 3 cycles. No state change between cycles.

### 2026-03-27 — [MANAGER] Step 3 design review should flag missing resilience patterns
- **Found in**: Step 4, Cycle 4
- **Step file to change**: step3_design_review.md
- **Recommended change**: Add a "Resilience Checklist" to Step 3's review template: (1) Does training have intra-epoch checkpointing for epochs > 2 hours? (2) Does the code handle variable input shapes gracefully with torch.compile? (3) Is there a GPU memory monitoring/alerting mechanism? (4) Can training resume from the last checkpoint after any crash? Step 3 should flag missing resilience items as CRITICAL, not just architectural or recipe issues.
- **Evidence**: The Step 3 design review (Cycles 1-3) reviewed architecture, recipe, data, and loss components thoroughly but did not flag the absence of intra-epoch checkpointing or the CUDA Graph risk from variable shapes. These were infrastructure resilience gaps, not design gaps, but they caused the most impactful failure so far (losing 2350 batches of training).

### 2026-03-27 — [MANAGER] Goal tracker should track infrastructure readiness separately
- **Found in**: Step 4, Cycle 4
- **Step file to change**: step4_learn_update.md (goal tracker update instructions)
- **Recommended change**: Add an "Infrastructure Readiness" section to the goal tracker template that tracks: torch.compile mode, checkpointing strategy (per-epoch vs intra-epoch), GPU memory headroom, known crash risks, and recovery plan. Currently the goal tracker focuses on model/data/recipe readiness but not infrastructure resilience.
- **Evidence**: The goal tracker said "Full training NOT YET LAUNCHED" and constraint was "Execution" when in reality training had been launched, crashed, and the constraint had shifted to Infrastructure. The tracker had no fields for infrastructure status, so this gap went unrecorded until Step 1 of the next cycle discovered it.

### 2026-03-28 — [MANAGER] Add Iteration Speed to Step 3 Design Review
- **Found in**: User request during cycle 5
- **Step file to change**: step3_design_review.md
- **Recommended change**: Add section "3.7 Iteration Speed Assessment" with questions:
  - How long does one epoch/prove-out take?
  - How many experiments can be run per day?
  - Is there a fast prove-out mode for quick hypothesis testing?
  - Does the cosine/LR schedule scale correctly for shorter runs?
  - Can mode collapse / overfitting be detected within 2-3 prove-out epochs?
- **Evidence**: Mode collapse was not detected until 90 min into full training. With prove-out mode analysis in Step 3, the manager would have recommended prove-out sweeps before committing to a full run.

### 2026-03-28 — [MANAGER] Updated Training Progression in Step 5 Validation
- **Found in**: User request
- **Step file to change**: step5_action_plan.md
- **Recommended change**: Update the training validation gate progression to: `unit-test → prove-out (LR sweep) → smoke-test → full`. Prove-out must pass before smoke test. Smoke test must pass before full. Never skip prove-out for new recipes.
- **Evidence**: V5-Small went straight from unit test to full training, skipping any fast recipe validation. Mode collapse discovered 90 min later.

### 2026-03-28 — [MANAGER] Step 3 Should Cross-Check All Loss Heads for NaN/Degenerate Output
- **Found in**: Step 4, Cycle 6
- **Step file to change**: step3_design_review.md
- **Recommended change**: Add to Step 3 design review checklist: "For each loss head, verify that test metrics show non-NaN, non-zero values. If any head produces NaN (e.g., return_mae=NaN), flag as a WARNING even if not blocking the primary metric." Step 3 noted the NaN but classified it as non-blocking. A broken loss head consuming gradient budget (weight=0.8) should be flagged more prominently.
- **Evidence**: return_mae=NaN across ALL 4 prove-outs. The return head has weight=0.8 (second highest loss component) but produces no useful gradient signal. This wastes ~27% of the effective gradient budget (0.8/3.0). Step 3 correctly identified this but did not escalate it as a potential training efficiency issue.
