# Manager Linear Question Checklist

Every manage cycle, the manager answers these questions **in order, one by one**. No question may be skipped. Each answer informs the next question. If any answer reveals a problem, the manager addresses it before moving forward.

---

## Phase 1: What Is Actually Happening Right Now?

1. **Is the dev session running?** If not, start it and resume the latest conversation.

2. **What is dev currently doing?** (Capture tmux pane — is dev idle, coding, running a script, waiting?)

3. **What processes are running on CPU right now?** (`ps aux` — any training, data prep, or pipeline scripts?)

4. **What is the GPU doing right now?** (`nvidia-smi` — idle, training, or other?)

5. **What files exist on disk that tell me the current state?** (Check: model run dirs, checkpoint files, token cache dirs, log files, status.json files. Do NOT trust goal_tracker.md — verify filesystem first.)

6. **Does the filesystem state match what goal_tracker.md says?** If not, which is correct? Update goal_tracker.md to match reality.

---

## Phase 2: Where Are We Relative to the Goal?

7. **What is the current best model performance?** (Read the latest training results — captured return, P@5%, rank correlation. Which model version? Which epoch?)

8. **What is the target performance?** (Read goals.md — what captured return / annualized return are we aiming for?)

9. **How big is the gap between current and target?** (Quantify: current CR vs target CR. Is it 2x? 5x? 10x?)

10. **Is the gap shrinking, stable, or growing?** (Compare to last cycle's metrics. Are we making progress or stalled?)

---

## Phase 3: What Is Blocking Progress?

11. **What is the single biggest constraint preventing us from closing the gap?** (Choose ONE from: architecture limitation / training recipe / data quality / data quantity / infrastructure / knowledge gap / nothing is blocking — we just need to execute)

12. **How do I know this is the real constraint and not a symptom?** (What evidence supports this? Could there be a deeper constraint I'm missing?)

13. **Has the constraint changed since the last cycle?** (Did new information emerge — training results, overfitting, architecture analysis — that shifts what the bottleneck is?)

---

## Phase 4: Is the Current Plan Still the Right Plan?

14. **What is the current hypothesis we're testing?** (State it explicitly: "We believe X will improve Y because Z.")

15. **Has any new evidence confirmed or weakened this hypothesis?** (Training metrics, overfitting patterns, architecture analysis, user feedback?)

16. **Is there a faster way to test this hypothesis?** (Could we use a smaller model? Fewer epochs? A subset of data? A simpler experiment that answers the same question?)

17. **Are we building the right thing, or are we building the thing we already started just because we started it?** (Sunk cost check: if we were starting fresh today with everything we know now, would we make the same choice?)

---

## Phase 5: Is the Design Sufficient?

18. **Is the model size appropriate for the training data?** (Compute params/sample ratio. Compare to models that overfit (V4: 0.82) and models that didn't (V3: 0.023). Where does the current design fall? Is overfitting likely?)

19. **How long will one training epoch take?** (Estimate from model size, data size, batch size, GPU speed. Is this fast enough to iterate 3-4 experiments per week? Or are we stuck at 1 experiment per week?)

20. **Could we start with a smaller version and scale up after validating?** (A smaller model trains faster, overfits less, and answers "does the approach work?" sooner. Always prefer small-first unless there's a specific reason for large.)

21. **Is the data format correct for this model design?** (Does the input format match what the model expects? Are the labels computed correctly? Is lookback window appropriate for the prediction horizon?)

22. **Is the data preparation sufficiently optimized?** (Has a performance test been run on representative data — not just tiny test quarters? Is CPU utilization multi-core? Is throughput reasonable compared to previous versions?)

23. **Is there a complete design document in research/ for what we're about to implement?** (Token spec, architecture, loss function, training recipe, data format — all present? If not, the constraint is a knowledge gap — investigate before implementing.)

---

## Phase 6: What Should We Do Next?

24. **What are ALL possible actions right now?** (List everything: operational tasks, strategic investigations, meta improvements, things that can run in parallel. Don't just pick the next item on the list.)

25. **Which resources are idle right now?** (GPU, CPU, dev — check all three. Any idle resource with available work is waste.)

26. **Can any actions run in parallel?** (GPU training + dev coding? CPU data prep + dev implementing model? Two independent investigations?)

27. **Which action has the highest expected value?** (Consider: expected improvement, cost in time, risk of failure, information value. Cheap investigations before expensive executions.)

28. **Is the highest-value action addressing the actual constraint, or a non-constraint?** (If the constraint is architecture but the action is hyperparameter tuning — wrong priority.)

---

## Phase 7: Is This Action Ready to Execute?

29. **If this is a data pipeline task: has the performance been validated?** (Smoke test on representative data? CPU multi-core verified? Memory stable? Throughput measured and extrapolated? If ANY is unchecked — validate first, don't launch the full run.)

30. **If this is a training task: are all prerequisites met?** (Unit test passed? Smoke test with GPU util >70%? AMP+FP16 enabled? torch.compile? Checkpoint/resume working? Per-run directory? If ANY is unchecked — fix first.)

31. **If this is an implementation task: is the design document complete?** (All specs present in research/? If not — investigate and write the design first.)

32. **Does this action chain to a follow-up?** (After this task completes, what should dev do immediately? Include it in the prompt. Never leave dev idle without the next task.)

33. **What are the success criteria?** (What specific metric or output confirms this action worked?)

34. **What are the failure criteria?** (What would tell us to stop and change approach? Define this BEFORE starting, not after.)

---

## Phase 8: After Taking Action

35. **Did the last action produce the expected result?** (Check metrics, outputs, dev's response. Did it match success criteria?)

36. **Did I miss anything in this cycle?** (Any question I should have asked but didn't? Any information I had but didn't use?)

37. **Should the goal_tracker be updated?** (New metrics, changed constraint, updated hypothesis, shifted beliefs?)

38. **Are all resources still utilized?** (After this action completes — is GPU busy? CPU busy? Dev busy? If anything went idle, what's next for that resource?)

---

## Quick Reference: The Questions That Caught Past Failures

| Failure | Question That Would Have Caught It |
|---------|-----------------------------------|
| Launched V4 training without architecture review | #17: "Are we building the right thing?" #16: "Is there a faster way?" |
| V4 missing checkpoint/resume | #30: "Are all training prerequisites met?" |
| V5 token prep launched without perf validation | #29: "Has performance been validated on representative data?" |
| Dev idle 10 hours while GPU trained | #25: "Which resources are idle?" #32: "Does this chain to a follow-up?" |
| Accepted ~1-2hr estimate without evidence | #22: "Is data prep sufficiently optimized?" #29: Smoke test on representative data? |
| V5-Large design accepted without questioning size | #18: "Is model size appropriate?" #19: "How long per epoch?" #20: "Could we start smaller?" |
| Manager didn't ask about V4 architecture before training | #14: "What hypothesis are we testing?" #15: "Has evidence changed?" |
| Smoke test ran on tiny quarters only | #29: "Representative data?" — must include large quarters |
| No design doc before implementation | #23: "Complete design document in research/?" |
| Optimized non-constraint (GPU util) while architecture was the bottleneck | #28: "Is this addressing the actual constraint?" |
