# Step 3: Design & Architecture Review

**Purpose**: Deep investigation of whether the current design is correct and optimized. NOT surface-level — read actual code, sample actual data.

## Persistence

**FIRST**: Read your own previous output from `/home/ubuntu/workspace/RLQuest-manager/step3_output.md` if it exists. This contains your previous deep analysis of the model, tokens, loss, and recipe — plus persistent notes about code structure, param counts, token distributions, and known issues.

**CRITICAL DELTA LOGIC**: Check if source files have changed since your last analysis:
```bash
stat -c '%Y %n' /home/ubuntu/workspace/RLQuest/firstrate_learning_v5/{model.py,config.py,data_loader.py,prepare_tokens.py} 2>/dev/null
```
Compare timestamps to those recorded in your persistent notes.

- **If no files changed AND no new training results**: Reuse your persistent notes. Write a brief "No changes since last cycle. Previous analysis still valid." + carry forward persistent notes. This should take <1 minute.
- **If files changed**: Re-investigate ONLY the changed files. Reuse persistent notes for unchanged parts.
- **If new training results exist**: Analyze the new metrics against your previous predictions.
- **If no previous output exists**: Full deep investigation (first run).

## Instructions (Full Investigation — only when needed)

### 3.1 Model Architecture
Read `model.py` and `config.py`:
- Actual dimensions (d_model, n_heads, n_layers, backbone_dim).
- Param count. Params/sample ratio. Compare to V3 (0.023) and V4 (0.82).
- Overfit risk assessment.
- Epoch time estimate.

### 3.2 Data Token Format
- Load one chunk with python. Inspect token values: min/max per dim, NaN/inf count, zero-dim check.
- Day mask distribution: % with 5 valid days, % with <3.
- Surface summary token: populated or zeros?
- Price context token: all 20 dims populated?

### 3.3 Lookback & Prediction Horizon
- Days of options lookback vs prediction horizon.
- Price context compensation for limited lookback.

### 3.4 Loss Function
- Number of components, weights, magnitudes.
- Appropriate for model size?
- Conflicting gradients risk?

### 3.5 Training Recipe
- lr, scheduler, warmup, batch size, dropout.
- Appropriate for model size and data?

### 3.6 Data Quality
- Total samples, chunks, split ratios.
- Time-based split? Leakage risk?

## Output Format

Write to `/home/ubuntu/workspace/RLQuest-manager/step3_output.md` (overwrite):

```
## Step 3: Design Review — [date/time]
### Mode: [FULL / DELTA / NO CHANGE]

[Analysis findings — full or delta depending on what changed]

### CRITICAL ISSUES: [blockers or "none"]
### RECOMMENDATIONS: [changes needed or "none — proceed"]

## Persistent Notes
(Carry forward + update. These are the "cache" for next cycle.)
- File timestamps: model.py=[ts], config.py=[ts], data_loader.py=[ts], prepare_tokens.py=[ts]
- Model: [params] params, d_model=[N], [N]+[N] layers, backbone=[N]
- Params/sample ratio: [value] — [LOW/MEDIUM/HIGH risk]
- Token dims: [N]/20 populated, zeros at dims [list]
- Day mask: [X]% with 5 valid days
- Loss: [N] components, weights=[list], dominant=[which]
- Recipe: lr=[X], batch=[X], dropout=[X], warmup=[X]
- Data: [N]M samples, [N] chunks, split=[description]
- Known issues: [numbered list]
- Epoch estimate: ~[N] hours
- Training status: [not started / epoch N / metrics summary]
- Watch items for next cycle: [specific checks]
```
