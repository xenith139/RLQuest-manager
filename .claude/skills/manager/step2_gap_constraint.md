# Step 2: Gap Analysis & Constraint Identification

**Purpose**: Understand where we are vs where we need to be, and identify the single biggest bottleneck blocking progress.

## Instructions

Using the ground truth from Step 1, answer these questions with evidence.

### 2.1 Current Performance
- What is the best model performance achieved so far? (Which version, which epoch, what metrics?)
- Read the latest training results or logs to get: captured return, P@5%, rank correlation, return correlation, direction accuracy.
- If no model has been trained yet for the current architecture: state that explicitly.

### 2.2 Target Performance
- Read `goals.md` — what is the target?
- What captured return per 10-day period would reach the target annualized return?
- What does the foresight upper bound look like? (170% in 50 days = ~3.4% per 10-day period)

### 2.3 Gap
- Current CR vs target CR — quantify the ratio (e.g., 1.5x, 2x, 5x improvement needed).
- Is the gap shrinking cycle over cycle? (Compare to previous training results if available.)
- At the current rate of progress, when would we reach the target? (If unknown, say so.)

### 2.4 Constraint Identification
Choose ONE primary constraint from:
- **Architecture**: Model design is the ceiling — no amount of training/tuning will fix it.
- **Training recipe**: Architecture is fine but hyperparameters, loss weights, lr schedule need work.
- **Data quality**: Architecture + recipe are fine but data has issues (format, labels, features, noise).
- **Data quantity**: Need more data or different data splits.
- **Infrastructure**: Tooling, checkpointing, performance, pipeline issues blocking progress.
- **Knowledge gap**: We don't know enough to make the right decision — need investigation.
- **Execution**: Everything is ready, just need to run it.

For the chosen constraint, provide:
- **Evidence**: What specifically tells you this is the constraint?
- **Why not the others**: Why is this the bottleneck and not something else?
- **Has it changed since last cycle?**

### 2.5 Hypothesis
State explicitly:
- "I believe [action] will improve [metric] from [current] to [target] because [reasoning]."
- "This can be tested by [experiment]."
- "Success looks like: [specific criteria]."
- "Failure looks like: [specific criteria]. If it fails, the next step is [pivot]."
- "Cost of being wrong: [time/resources wasted]."

## Output Format

```
## Step 2: Gap & Constraint

### Best performance: [version] [epoch] CR=[value] P@5%=[value]
### Target: CR > [value] (~[X]% annualized)
### Gap: [current] vs [target] = [ratio]x improvement needed
### Constraint: [category] — [one-sentence description]
### Evidence: [what proves this is the bottleneck]
### Hypothesis: "[full statement]"
### Success criteria: [specific metrics]
### Failure criteria: [what triggers a pivot]
```
