# Step 3: Design & Architecture Review

**Purpose**: Deep investigation of whether the current design (model, data, tokens, training recipe) is actually correct and optimized for the goal. NOT a surface-level checkbox — genuine analysis.

## Instructions

For each section, READ THE ACTUAL CODE and data, don't rely on descriptions. Use bash to inspect files, sample data, check configs.

### 3.1 Model Architecture Assessment

Read the model code (`model.py`, `config.py`):
- What are the actual dimensions? (d_model, n_heads, n_layers, backbone_dim)
- How many parameters? Compute params/sample ratio. Compare to V3 (0.023, no overfit) and V4 (0.82, overfit epoch 2). Where does current model fall?
- Is the model likely to overfit? If params/sample > 0.3, flag HIGH risk.
- Is the model large enough to learn the target patterns? If params < 500K, flag capacity concern.
- How long will one epoch take? Estimate from model size, data size, batch size. Is this fast enough for 3+ experiments/week?

### 3.2 Data Token Format Assessment

Read the data preparation code (`prepare_tokens.py`, `config.py`):
- What does each token dimension represent? Are all 20 dims populated or are some zeros?
- **Sample actual data**: Load one chunk and inspect token values. Are they in reasonable ranges? Any NaN/inf?
- Check day_mask distribution: what % of samples have all 5 days valid vs only 1-2? Are sparse samples (1-2 days) polluting training?
- Is the surface summary token capturing meaningful aggregates? Check a few values.
- Is the price context token fully populated? Check all 20 dims.
- Are the new features (IV percentile, volume percentile, IV/realized vol, etc.) correctly computed? Spot-check against raw data if possible.

### 3.3 Lookback & Prediction Horizon

- Current lookback: how many days of options data does the model see?
- Current prediction horizon: how many days forward for the label?
- Is the lookback sufficient for the strongest predictive signals? (Foresight analysis showed 5d/10d/20d momentum matters most.)
- Does the price context token compensate for limited options lookback? (It carries 20-day aggregate returns.)
- Would extending lookback from 5 to 7 or 10 days significantly help? What's the cost (data size, memory, training time)?

### 3.4 Loss Function Assessment

Read the loss function code:
- How many loss components? What are the weights?
- For a ~1M param model, is 7 loss components too many? (Risk: conflicting gradients, noisy optimization.)
- Which loss component dominates? (Compute approximate magnitude of each.)
- Should any components be disabled for the first training run to simplify optimization?
- Is the contrastive loss appropriate at this model size, or should it be added later after a baseline?

### 3.5 Training Recipe Assessment

Read the training code (`train.py`, `config.py`):
- Learning rate, scheduler, warmup steps — appropriate for model size?
- Batch size — does it fit in GPU memory with this model? Is it optimal?
- Dropout — appropriate for params/sample ratio?
- Is there anything in the training code that could cause issues? (Wrong data shapes, missing normalization, incorrect loss computation?)

### 3.6 Data Preparation Quality

Check the token preparation output:
- `cat meta.json` — total samples, chunks per split.
- Is the train/val/test split reasonable? (Time-based? Random? Leakage risk?)
- Throughput achieved — is it acceptable for iteration?
- Any warnings or errors in the preparation log?

## Output Format

```
## Step 3: Design Review

### Model: [params]M params, [ratio] params/sample — [LOW/MEDIUM/HIGH] overfit risk
### Capacity: [sufficient/insufficient] for temporal pattern learning
### Epoch estimate: ~[N] hours → [N] experiments/week
### Token format: [N]/20 dims populated, [issues found / clean]
### Day mask: [X]% samples with 5 valid days, [X]% with <3
### Lookback: [N] days options + [N] days price context — [sufficient/needs extension]
### Loss: [N] components — [appropriate/too complex for model size]
### Recipe: lr=[X], batch=[X], dropout=[X] — [appropriate/needs adjustment]
### Data quality: [N]M samples, [N] chunks — [good/issues found]
### CRITICAL ISSUES: [list any blockers or high-risk items]
### RECOMMENDATIONS: [specific changes before training]
```
