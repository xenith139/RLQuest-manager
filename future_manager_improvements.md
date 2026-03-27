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
