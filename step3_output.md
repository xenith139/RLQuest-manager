# Step 3: Design & Architecture Review

**Timestamp:** 2026-03-27 21:15 UTC

---

## 3.1 Model Architecture Assessment

### Actual Code Inspection

**Source:** `/home/ubuntu/workspace/RLQuest/firstrate_learning_v5/model.py` and `config.py`

**Config (V5-Small):**
```
d_model = 128       (model.py default = 256)
n_heads = 4         (model.py default = 8)
intra_layers = 2    (model.py default = 4)
temporal_layers = 1  (model.py default = 2)
d_ff = 512          (model.py default = 1024) -- NOTE: config d_ff is NEVER USED (see bug below)
dropout = 0.25      (model.py default = 0.15)
backbone_dim = 64   (model.py default = 128)
batch_size = 512
```

**Head dimension:** d_model / n_heads = 128 / 4 = 32. Valid, divisible.

### Actual Parameter Count (computed by loading model)

```
Total parameters: 679,498
Trainable parameters: 679,498
Total MB (float32): 2.6 MB

=== PARAMETER BREAKDOWN ===
  intra_day:       400,641 (59.0%)
  temporal:        232,320 (34.2%)
  backbone:         25,152 (3.7%)
  magnitude_head:    4,225 (0.6%)
  direction_head:    4,225 (0.6%)
  quantile_head:     4,485 (0.7%)
  return_head:       4,225 (0.6%)
  confidence_head:   4,225 (0.6%)

=== INTRA-DAY ENCODER BREAKDOWN ===
  token_proj:   2,688
  pos_enc:        385
  type_embed:     640
  blocks:     396,544  (2 transformer blocks)
  norm:           256

=== TEMPORAL ENCODER BREAKDOWN ===
  day_proj:    16,512
  temporal_pe:    640
  diff_proj:   16,512
  blocks:     198,272  (1 transformer block)
  norm:           256
```

**Key observation:** 59% of parameters are in the IntraDayEncoder (shared across 5 days). The temporal encoder is 34%. The prediction heads are tiny at ~3%.

### Overfit Risk Analysis

```
Params:              679,498
Train samples:     5,798,357
Params/sample ratio:   0.117

V3 ratio (no overfit, 38 epochs): 0.023
V4 ratio (overfit epoch 2):       0.82
V5-Small ratio:                   0.117
```

**MEDIUM overfit risk.** The ratio is 5x V3's safe zone but 7x less than V4's catastrophic zone. V5-Small is in the moderate range. With dropout=0.25 (higher than V3/V4) and weight_decay=0.01, overfitting is possible but not likely to be as severe as V4. V3 trained for 38 epochs without overfitting; V5-Small should get at least 5-10 epochs before overfitting manifests.

### Capacity Assessment

At 679K params, the model has sufficient capacity for temporal pattern learning:
- V3 achieved CR=0.0147 with only 136K params
- V5-Small is 5x larger than V3, with a more expressive architecture (attention vs linear)
- The IntraDayEncoder processes 128 tokens per day through 2 transformer layers -- sufficient to capture surface structure
- The TemporalEncoder sees 5 daily embeddings through 1 transformer layer -- minimal but functional for temporal patterns
- **Capacity is sufficient.** The 5 prediction heads are small (4K params each) which is appropriate for the backbone_dim=64 they receive.

### Epoch Time Estimate

From the unit test:
- 50 batches ran in ~2:14 (134 seconds)
- Measured: data=281ms/batch, gpu=1729ms/batch => ~2.01s/batch total
- GPU utilization: 86%

Full epoch calculation:
```
Train samples: 5,798,357
Batch size: 512
Batches per epoch: 11,324

At 2.01 s/batch (unit test measured):  6.3 hours/epoch
At 2.50 s/batch (conservative):        7.9 hours/epoch
At 2.68 s/batch (pessimistic):         8.4 hours/epoch
```

**Full training estimate (patience=15, likely 10-15 epochs):**
- Optimistic: 63 hours (2.6 days)
- Conservative: 95 hours (4.0 days)
- Pessimistic: 126 hours (5.3 days)

**Experiments per week: 1.3 to 2.7** -- this is on the slow side for rapid iteration.

### Latent Bug: config.d_ff is Never Used

The `config.py` has `d_ff: int = 512`, but the model code NEVER reads it. Both `IntraDayEncoder` and `TemporalEncoder` hardcode `d_model * 4` as the feedforward dimension:

```python
# IntraDayEncoder line 89:
TransformerBlock(d_model, n_heads, d_model * 4, dropout)

# TemporalEncoder line 154:
TransformerBlock(d_model, n_heads, d_model * 4, dropout)
```

In the current config, d_model=128 so d_model*4=512, which happens to equal config.d_ff=512. So there is NO functional impact now. But if someone changes d_ff in config expecting the model to respect it, nothing will happen. This is a maintenance trap.

**Severity: LOW** (no functional impact currently).

---

## 3.2 Data Token Format Assessment

### Actual Data Inspection

**Loaded chunk:** `chunk_0000.pt.zst` (50,000 samples, early time period ~2010) and `chunk_0070.pt.zst` (50,000 samples, middle time period)

```
tokens shape: (50000, 5, 128, 20) float32
token_types shape: (50000, 5, 128) int64
mask shape: (50000, 5, 128) bool
day_mask shape: (50000, 5) bool
y_return shape: (50000,) float32
y_big shape: (50000,) float32
```

### NaN/Inf Check

```
NaN count: 0
Inf count: 0
```

**CLEAN.** No NaN or Inf values in the token data. The `nan_to_num` cleanup in prepare_tokens.py is working correctly.

### Per-Dimension Analysis (Contract Tokens Only, chunk_0070)

```
Dim                   Name        Min        Max       Mean   Zeros%
  0              moneyness     0.0000    96.7742     1.2931     0.0%
  1                is_call     0.0000     1.0000     0.5000    50.0%  (expected: ~50% calls)
  2               dte_norm     0.0000     2.3041     0.2935     1.3%
  3                 log_iv     0.0000     1.6093     0.1388    52.4%  (** HIGH zeros **)
  4                 log_oi     0.0000    13.6891     1.8851    52.6%  (** HIGH zeros **)
  5             log_volume     0.0000    12.5818     0.9860    56.7%  (** HIGH zeros **)
  6                  delta    -1.0000     1.0000     0.0210     5.7%
  7                  gamma   -10.0000    10.0000     0.0493    20.1%
  8                   vega   -10.0000    10.0000     0.0546    22.4%
  9                  theta   -10.0000    10.0000    -0.0178    12.2%
 10         bid_ask_spread     0.0000     5.0000     0.0046    96.8%  (** EXTREMELY sparse **)
 11         mid_price_norm     0.0000    50.3226     0.2768     0.0%
 12          iv_percentile     0.0000     1.0000     0.5442     0.8%
 13      volume_percentile     0.0000     1.0000     0.5367     0.8%
 14          oi_percentile     0.0000     1.0000     0.5380     0.8%
 15      iv_realized_ratio     0.0000    10.0000     0.6966    52.4%  (zeros match log_iv)
 16      moneyness_peak_iv   -63.6201    52.8701     0.2027     7.9%
 17          expiry_bucket     0.0000     1.0000     0.6523     7.8%
 18        intrinsic_value     0.0000    95.7742     0.2928    50.0%
 19           log_ba_ratio     0.0000     5.0000     0.0162    96.9%  (** EXTREMELY sparse **)
```

### Populated Dimensions Assessment

**All 20 dims are populated for contract tokens** -- none are all-zeros. However:

1. **Dims 3, 4, 5, 15 have ~52% zeros:** These correspond to IV, OI, volume, and IV/realized_vol. The zeros come from contracts with zero IV and zero open interest/volume. This is expected for illiquid far-OTM options. The `log1p` transform means zero IV maps to log1p(0)=0. This is a valid encoding (zero = no information).

2. **Dims 10 and 19 have 96-97% zeros:** bid_ask_spread and log_ba_ratio. These are nearly all zero because the calculation `(asks - bids) / max(mids, 0.01)` yields zero when bid=ask (which is common) and the log ratio is also zero when bid=ask. These dims carry almost no information for most tokens but spike for illiquid contracts. The model may struggle to learn from such sparse signals.

3. **Dim 16 (moneyness_peak_iv) has extreme range [-63, +52]:** This is because moneyness itself can be very far from 1.0 (deep OTM), and the peak IV moneyness is subtracted. The extreme values are physically meaningful (very deep OTM contracts relative to peak IV). However, values of +-50 are far outside the typical [-5, 5] range of other features -- this creates a scale mismatch that the linear projection must handle.

4. **Dim 11 (mid_price_norm) and dim 18 (intrinsic_value) have extreme maxes (50+, 95+):** Deep ITM contracts can have very high normalized prices. Again, a scale mismatch with other dims.

### CRITICAL FINDING: Summary Token Has 6 Zero Dimensions

**Surface summary tokens (type 1) only populate 14 out of 20 dimensions.** Dims 14-19 are ALL ZEROS:

```
=== SURFACE SUMMARY TOKEN ANALYSIS (4,750 samples) ===
  Dim 14: min=  0.0000 max=  0.0000 mean=  0.0000 zeros=100.0%
  Dim 15: min=  0.0000 max=  0.0000 mean=  0.0000 zeros=100.0%
  Dim 16: min=  0.0000 max=  0.0000 mean=  0.0000 zeros=100.0%
  Dim 17: min=  0.0000 max=  0.0000 mean=  0.0000 zeros=100.0%
  Dim 18: min=  0.0000 max=  0.0000 mean=  0.0000 zeros=100.0%
  Dim 19: min=  0.0000 max=  0.0000 mean=  0.0000 zeros=100.0%
```

This is because `prepare_tokens.py` only sets `summary[0]` through `summary[13]` in the summary token builder. The remaining 6 dims are never assigned.

**Impact:** MEDIUM. The summary token appears in every valid day at position 1. The model sees it as a 14-dim signal padded with 6 zeros. The type embedding (type=1) tells the model this is a summary token, but the zeros waste 30% of the token's information capacity. With 128 tokens per day, the summary token is ~1.6% of input -- not critical, but it could be better utilized.

### FINDING: Price Context Token Dim 15 is Always Zero

```
  Dim 15: min=  0.0000 max=  0.0000 mean=  0.0000 zeros=100.0%
```

In `prepare_tokens.py`, the price context token skips from `pf[14]` (median IV) to `pf[16]` (put/call volume ratio). Dim 15 is never assigned.

**Impact:** LOW. One wasted dimension out of 20 in the price token.

### Price Context Token Analysis (4,750 samples)

```
  Dim  0 (1d return):      min=-0.0797 max= 0.0993 mean= 0.0027  zeros=2.5%
  Dim  1 (3d return):      min=-0.1301 max= 0.2336 mean= 0.0095  zeros=0.9%
  Dim  2 (5d return):      min=-0.1233 max= 0.2426 mean= 0.0159  zeros=0.6%
  Dim  3 (10d return):     min=-0.1708 max= 0.2707 mean= 0.0290  zeros=0.3%
  Dim  4 (20d return):     min=-0.2957 max= 0.3960 mean= 0.0413  zeros=0.2%
  Dim  5 (5d rvol):        min= 0.0213 max= 0.9333 mean= 0.2353  zeros=0.0%
  Dim  6 (10d rvol):       min= 0.0526 max= 0.8339 mean= 0.2662  zeros=0.0%
  Dim  7 (20d rvol):       min= 0.0787 max= 0.7594 mean= 0.2914  zeros=0.0%
  Dim  8 (vol ratio):      min= 0.1317 max= 5.0000 mean= 0.9783  zeros=0.0%
  Dim  9 (vol momentum):   min= 0.2666 max= 4.3813 mean= 1.0030  zeros=0.0%
  Dim 10 (price accel):    min=-0.3137 max= 0.2695 mean= 0.0028  zeros=0.0%
  Dim 11 (high-low range): min= 0.0038 max= 0.1742 mean= 0.0256  zeros=0.0%
  Dim 12 (gap):            min=-0.0700 max= 0.0539 mean= 0.0004  zeros=7.4%
  Dim 13 (RSI-like):       min= 0.1298 max= 0.9729 mean= 0.5894  zeros=0.0%
  Dim 14 (median IV):      min= 0.1555 max= 1.0087 mean= 0.3462  zeros=0.0%
  Dim 15 (EMPTY):          zeros=100.0%  ** BUG **
  Dim 16 (PC vol ratio):   min= 0.0000 max= 5.0000 mean= 0.9574  zeros=20.4%
  Dim 17 (PC OI ratio):    min= 0.0828 max= 3.2721 mean= 0.7716  zeros=0.0%
  Dim 18 (total opt vol):  min= 0.0000 max= 5.0000 mean= 4.1571  zeros=8.8%
  Dim 19 (max vol zscore): min= 0.0000 max= 5.0000 mean= 4.5072  zeros=8.8%
```

**Price context token: 19/20 dims populated (dim 15 is a bug).** All other dims show reasonable ranges and meaningful distributions. The price context token successfully carries 1d/3d/5d/10d/20d returns, realized volatility at multiple horizons, volume features, and options summary stats.

**Specific concern:** Dims 18 and 19 (total options volume and max volume z-score) have very high means (4.15, 4.51) because log1p(total_vol) for liquid stocks is naturally large. These are clipped to [-5, 5]. This creates a compressed distribution near the upper bound for liquid stocks.

### Token Type Distribution

```
  Type 0 (price_context):     237,291 (1.4%)
  Type 1 (surface_summary):   237,291 (1.4%)
  Type 2 (call_contract):   8,101,465 (48.6%)
  Type 3 (put_contract):    8,100,443 (48.6%)
```

Good balance between calls and puts. Price and summary tokens are ~1.4% each of all valid tokens -- they are a small fraction of the sequence but carry high-density aggregate information.

### Tokens Per Valid Day

```
Valid tokens per day: min=24, max=128, mean=74.6, median=62
  p10: 30
  p25: 42
  p50: 62
  p75: 118
  p90: 128
  p95: 128
  p99: 128
```

Most stocks have 60-120 tokens per day. ~25% of days hit the 128-token cap (these are liquid stocks like AAPL, TSLA, SPY where contracts are truncated to the 126 nearest-ATM). The truncation keeps the most informative contracts.

---

## 3.3 Day Mask Distribution

### Actual Data

```
=== chunk_0000 (early era, N=50,000) ===
  1 valid days:    1,271 (2.5%)
  2 valid days:    1,271 (2.5%)
  3 valid days:    1,271 (2.5%)
  4 valid days:    1,270 (2.5%)
  5 valid days:   44,917 (89.8%)

=== chunk_0070 (middle era, N=50,000) ===
  1 valid days:      809 (1.6%)
  2 valid days:      809 (1.6%)
  3 valid days:      809 (1.6%)
  4 valid days:      808 (1.6%)
  5 valid days:   46,765 (93.5%)

=== chunk_0130 (late era, N=50,000) ===
  1 valid days:      796 (1.6%)
  2 valid days:      796 (1.6%)
  3 valid days:      796 (1.6%)
  4 valid days:      796 (1.6%)
  5 valid days:   46,816 (93.6%)
```

**89.8%-93.6% of samples have all 5 valid days.** Only ~6-10% have fewer than 5 days. These sparse samples come from the beginning of each stock's options data history (first few days in the dataset have insufficient lookback).

The distribution is almost perfectly uniform for 1-4 days, suggesting these are exactly the first 4 samples for each stock entering the dataset.

**Assessment:** The day_mask distribution is GOOD. The sparse samples (1-4 days) are a small minority and the model correctly handles them via day_mask attention masking. No filtering needed.

---

## 3.4 Lookback & Prediction Horizon

### Current Setup
- **Options lookback:** 5 trading days of raw options chain data
- **Price context lookback:** 20 trading days via scalar features (1d/3d/5d/10d/20d returns, realized vol)
- **Prediction horizon:** 10 trading days forward
- **Big move threshold:** |return| > 5%

### Assessment

The 5-day options lookback captures the most recent trading week of options activity. The foresight analysis identified 5d and 10d momentum as the strongest signals. The 5-day raw lookback captures the 5d pattern directly through temporal attention. The 10d and 20d patterns are captured indirectly through the price context token (which carries 10d and 20d returns).

**The price context token effectively compensates for the limited 5-day options lookback.** It carries:
- 1d, 3d, 5d, 10d, 20d stock returns (catching multi-scale momentum)
- 5d, 10d, 20d realized volatility (catching volatility regime)
- Volume ratio and volume momentum (catching participation changes)
- RSI-like indicator (catching mean reversion signals)
- Options summary (median IV, P/C ratios)

**Extending to 7 or 10 days:**
- Memory cost: (7/5) to (10/5) = 1.4x to 2x more data per sample
- Training time: proportional increase (1.4x to 2x longer per epoch)
- Benefit: direct attention over 7-10 days of options chain evolution (vs scalar aggregates)
- **Not recommended for first training run.** Validate the 5-day architecture first.

**Verdict: 5 days options + 20 days price context is sufficient for the first full training run.**

---

## 3.5 Label Distribution

```
=== chunk_0000 (early era ~2010) ===
  y_return: mean=0.0382, std=0.0671
  y_big: 41.2% positive
  y_return > 0: 75.7%  ** STRONGLY POSITIVE BIAS (bull market) **

=== chunk_0070 (middle era ~2014-2015) ===
  y_return: mean=-0.0121, std=0.0938
  y_big: 44.3%
  y_return > 0: 44.6%

  > 0.05 (big up): 17.3%
  < -0.05 (big down): 27.0%
  abs < 0.02 (small): 26.3%
  abs > 0.10 (very big): 18.9%
```

**Key observations:**

1. **Label distribution varies dramatically across time periods.** Early chunks (2010 era, post-crisis recovery) have 75.7% positive returns. Middle chunks have 44.6% positive. The model must learn to handle changing market regimes across the training set.

2. **y_big rate is ~41-44%.** Nearly half of all 10-day periods have >5% absolute moves. This makes the big move classification task somewhat balanced (not extremely imbalanced).

3. **Contrastive loss bucket distribution (chunk_0070):** |return| > 10% (big up or big down): 18.9%, |return| < 2% (small): 26.3%. The contrastive loss requires pairs within these buckets. With batch_size=512: ~97 "very big" samples and ~135 "small" samples per batch. This should provide enough pairs for meaningful contrastive learning.

---

## 3.6 Loss Function Assessment

### Actual Loss Components and Weights

```
Component                Weight    Type
1. Magnitude (focal)      1.0     Focal BCE with label smoothing (gamma=2, alpha=0.5)
2. Direction               0.5     Binary cross-entropy
3. Quantile               0.4     Asymmetric pinball loss (5 quantiles, weighted)
4. Return (Huber)          0.8     Huber loss (delta=0.10), 3x weight on big moves
5. Confidence              0.2     BCE (self-calibration)
6. Contrastive             0.15    Cosine similarity between same-bucket embeddings
7. Embedding variance      0.1     ReLU penalty if embedding variance < 1.0
                          ----
Total weight sum:          3.15
```

### Analysis: 7 Components for a 680K-Param Model

**This is aggressive.** 7 loss components create 7 gradient signals that the model must balance. For a 680K-param model with only 5 prediction heads (sharing a 64-dim backbone), the risk is:

1. **Conflicting gradients:** The magnitude head wants to predict |return| > 5%, but the return head wants to predict the actual return value. The direction head wants p(return > 0). These three tasks can conflict: a stock with return = +3% is NOT a big move but IS positive direction. The model must learn a nuanced backbone representation that satisfies all three.

2. **Dominant component:** The magnitude loss (weight 1.0) and return loss (weight 0.8) together account for 57% of the total loss. The direction loss (0.5) adds 16%. So 73% of the gradient comes from three related-but-different regression/classification tasks on the same return variable.

3. **Contrastive loss (weight 0.15):** At batch_size=512, the contrastive loss computes a 512x512 similarity matrix on the 64-dim backbone embeddings. This has O(batch_size^2) compute but the weight is small (0.15 / 3.15 = 4.8% of total). The signal is likely too weak to meaningfully shape the embedding space early in training. However, it should not cause harm at this weight.

4. **Embedding variance (weight 0.1):** This regularizer prevents embedding collapse (all samples mapping to the same point). At weight 0.1 (3.2% of total), it is a weak safety net. It uses ReLU(1 - var) so it only fires when variance drops below 1.0. Reasonable.

5. **Confidence calibration (weight 0.2):** This trains p_confidence to predict whether the magnitude prediction is correct. It's a meta-prediction that depends on the magnitude head being reasonable first. Early in training, the magnitude head is random, so the "correct" label is also random. This creates a potentially noisy gradient signal early on that resolves as training progresses.

### Recommendation

The v5_design_review.md recommended simplifying the loss for the first run:

> "Consider simplifying for V5-Small: drop contrastive + confidence initially"

This is a reasonable recommendation. For the FIRST full training run, I recommend:

**Option A (conservative, recommended):** Keep all 7 components as-is. The weights are already low for the speculative components (contrastive=0.15, confidence=0.2, emb_var=0.1 = total 14.3% of loss). The risk of harmful interference is low, and removing them would require a code change that delays the training launch.

**Option B (if we need to change config before launch):** Set w_contrast=0 and w_conf=0. This removes 11% of the loss and simplifies to 5 components, all focused on the primary prediction task.

**Decision: Proceed with Option A (keep all 7) for the first run.** The dev should NOT be delayed for loss simplification. The GPU is idle. Launch now, analyze training curves for loss component conflicts.

---

## 3.7 Score Formula Analysis

```python
score = (
    p_big * p_up * p_confidence          # [0, 1]
    + 0.15 * sigmoid(quantiles[:, 3])    # [0, 0.15] (95th percentile)
    + 0.10 * sigmoid(quantiles[:, 4])    # [0, 0.10] (99th percentile)
    + 0.10 * tanh(pred_return)           # [-0.10, 0.10]
    - 0.05 * relu(quantiles[:, 0])       # [-0.05, 0] (median penalty)
)
```

**The score is dominated by the classification term `p_big * p_up * p_confidence` (range [0, 1]).** The quantile and return terms add at most 0.35 on a [0, 1] base. This means:

- The **captured return metric** (top 10% by score) is primarily driven by classification confidence
- The **return_corr metric** (correlation of pred_return with actual) is independent of score
- This explains the unit test pattern: high return_corr (0.088) but low CR (0.0015) -- the return prediction head learned some signal quickly, but the classification heads (which dominate score) hadn't converged yet

**This is not a bug** -- it's a deliberate design choice. The score reflects "probability of big upward move, weighted by confidence and tail quantiles." As training progresses and p_big/p_up/p_confidence calibrate, the CR should improve.

---

## 3.8 Training Recipe Assessment

### Learning Rate and Schedule

```
lr: 3e-4
warmup_steps: 1000
schedule: cosine decay with linear warmup (min LR = 0.1 * base)
optimizer: AdamW (betas=0.9, 0.98)
weight_decay: 0.01
grad_clip: 1.0
```

**Assessment:** lr=3e-4 is standard for transformer models of this size. The warmup of 1000 steps corresponds to ~1000 * 512 / 5.8M = 8.8% of one epoch. This is within the typical 5-15% warmup range. Cosine decay to 10% of base LR is standard. AdamW with beta2=0.98 (instead of default 0.999) provides slightly faster adaptation to gradient statistics -- reasonable for financial data with changing regimes.

**Concern:** weight_decay=0.01 is standard for large models but may be too weak for V5-Small with its medium overfit risk. Consider increasing to 0.03-0.05 if overfitting is observed.

### Batch Size

**batch_size=512** with 5-day sequences of 128 tokens each. Memory estimate per batch:

```
tokens: 512 * 5 * 128 * 20 * 4 bytes = 52 MB
token_types: 512 * 5 * 128 * 8 bytes = 2.6 MB
mask: 512 * 5 * 128 * 1 byte = 0.3 MB
Activations (estimated): ~200-400 MB (attention maps, intermediate)
Model params: 2.6 MB
Total VRAM estimate: ~300-500 MB
GPU: Quadro RTX 6000 with 24 GB VRAM
```

**batch_size=512 fits easily.** Could potentially increase to 1024 or 2048 for faster training, but the unit test showed 86% GPU utilization at batch_size=512, suggesting the GPU is already well-utilized. Increasing batch size would reduce the number of optimizer steps per epoch, which may hurt convergence.

### Dropout

**dropout=0.25** applied in TransformerBlock feedforward layers and attention. This is higher than the model default of 0.15, which was intentionally increased for V5-Small per the design review recommendation.

With a params/sample ratio of 0.117, dropout=0.25 is appropriate. If no overfitting is observed, could reduce to 0.15-0.20 for a second run.

### AMP (Automatic Mixed Precision)

**Enabled with FP16.** The loss computation is done in FP32 (outputs are cast with `outputs_f32 = {k: v.float() for k, v in outputs.items()}`). GradScaler is enabled. This is correct -- AMP FP16 gives ~2x throughput on Turing architecture (RTX 6000 has tensor cores).

**Concern:** The contrastive loss computes a batch_size x batch_size similarity matrix. At batch_size=512, this is a 512x512 matrix -- small enough that FP16 precision should be fine.

### Potential Issues in Training Code

1. **torch.compile(mode='reduce-overhead'):** This mode is appropriate for repeated identical tensor shapes (which V5 has -- fixed batch size, fixed sequence length). The first few batches will be slow (compilation) but subsequent batches benefit from kernel fusion.

2. **Data loading:** The V5InterleavedDataset loads 3 chunks at a time (~150K samples), shuffles, and yields batches. With 138 train chunks, this means 46 chunk-groups per epoch. Each chunk load requires decompression. The unit test showed data=281ms/batch, which is only 14% of total batch time. Data loading is not a bottleneck.

3. **IntraDayEncoder loop:** The forward pass loops over 5 days sequentially. This means the IntraDayEncoder processes one day at a time, not in parallel. With batch_size=512, each day processes at most 512 samples through 2 transformer layers on 128 tokens. This is 5 sequential GPU operations where they could potentially be batched into one. **This is a potential 2-3x throughput optimization if training is too slow**, but is not a correctness issue.

4. **val_score = captured_return:** The early stopping patience is based on captured_return (top 10% by score). This means the model selects the best epoch based on how well the score ranking identifies high-return stocks. If the score formula is dominated by classification (as analyzed above), early stopping may not select the epoch with the best return prediction. However, CR is the ultimate business metric, so this alignment is correct.

---

## 3.9 Data Preparation Quality

### Meta

```json
{
  "splits": {
    "train": {"n_chunks": 138, "n_samples": 5798357},
    "val":   {"n_chunks": 63,  "n_samples": 2769223},
    "test":  {"n_chunks": 63,  "n_samples": 2844134}
  },
  "token_dim": 20,
  "max_tokens_per_day": 128,
  "n_days": 5
}
```

- **Total: 11,411,714 samples** across 264 chunks
- **Split: 50.8% train / 24.3% val / 24.9% test**
- All 64/64 quarters processed successfully

### Train/Val/Test Split

```
Train (40 quarters): 2010_q1 through 2019_q4  (10 years)
Val   (12 quarters): 2020_q1 through 2022_q4  (3 years)
Test  (12 quarters): 2023_q1 through 2025_q4  (3 years)
```

**TIME-BASED split.** No data leakage. Train on historical data, validate on the next period, test on the most recent period. This is the correct methodology for financial data.

**Concern:** The val set includes COVID (2020) and the 2022 bear market. The test set includes 2023-2025 (recovery + bull market). Performance on val vs test may differ significantly due to regime change. This is realistic but means the model must generalize across regimes.

### Val/Test Size Concern

The val and test sets are each ~24% of total data (2.8M samples each). This is LARGE relative to the training set. In financial ML, having 3 years of val and 3 years of test is excellent for robust evaluation. However, it means the training set is only 5.8M samples (vs the full 11.4M). If overfitting becomes an issue, there's room to expand training by using walk-forward validation (though this requires code changes).

---

## 3.10 Moneyness Distribution and Feature Scale Mismatches

The moneyness distribution for contract tokens:
```
[0.0, 0.5):    10.2%  (deep OTM puts)
[0.5, 0.8):    15.2%  (OTM puts)
[0.8, 0.95):   13.9%  (near OTM)
[0.95, 1.05):  19.7%  (ATM)
[1.05, 1.2):   12.9%  (near OTM calls)
[1.2, 2.0):    20.7%  (OTM calls)
[2.0, 5.0):     5.8%  (deep OTM calls)
[5.0, 100.0):   1.7%  (extreme OTM)
```

Good ATM coverage (19.7% within 5% of ATM). The distribution is roughly symmetric around ATM, which is expected.

**Feature scale mismatches across dimensions:**
- Most features: range [-1, 5] or [-10, 10]
- moneyness: range [0, 97] -- can be huge for deep OTM
- moneyness_peak_iv: range [-64, +53] -- very large
- mid_price_norm: range [0, 50] -- high for deep ITM
- intrinsic_value: range [0, 96] -- high for deep ITM

The linear projection (`token_proj: 20 -> 128`) must learn to handle these scale differences. With only 2,688 parameters, this is a lightweight projection. The extreme values (moneyness > 5, mid_price_norm > 10) affect <2% of tokens, so the projection will primarily optimize for the bulk of the distribution. The extreme values may receive poor representation.

**Recommendation for future iteration:** Consider clipping moneyness to [0.2, 5.0] and mid_price_norm to [0, 10] in token preparation. This would lose some information about extreme OTM contracts but would normalize the feature scale.

---

## Summary

### Model: 0.680M params, 0.117 params/sample -- MEDIUM overfit risk
### Capacity: SUFFICIENT for temporal pattern learning (5x V3 which achieved CR=0.0147)
### Epoch estimate: ~6.3-8.4 hours -- ~1.3-2.7 experiments/week (slow but acceptable)
### Token format: 20/20 dims populated for contracts, 14/20 for summary tokens, 19/20 for price tokens -- ISSUES FOUND
### Day mask: 89.8-93.6% samples with 5 valid days, 6.4-10.2% with <5 -- CLEAN
### Lookback: 5 days options + 20 days price context -- SUFFICIENT for first run
### Loss: 7 components -- ACCEPTABLE (aggressive but weights are reasonable, 73% from core tasks)
### Recipe: lr=3e-4, batch=512, dropout=0.25 -- APPROPRIATE for model size and overfit risk
### Data quality: 11.4M samples, 264 chunks, time-based split -- GOOD, no leakage
### Split: train 2010-2019, val 2020-2022, test 2023-2025 -- CORRECT time-based methodology

### CRITICAL ISSUES:

1. **Summary token dims 14-19 are all zeros (6/20 dims wasted).** The `prepare_tokens.py` summary token builder only populates 14 dimensions. This wastes 30% of the summary token's capacity. NOT A BLOCKER -- the model can learn from 14 dims -- but should be fixed in a future token prep run.

2. **Price token dim 15 is always zero (bug).** Code skips from `pf[14]` to `pf[16]`. Minor bug, not a blocker.

3. **Config d_ff is never used by the model.** IntraDayEncoder and TemporalEncoder both hardcode `d_model * 4` for feedforward dimension. Currently harmless (128*4=512=config.d_ff) but a latent maintenance trap.

4. **Feature scale mismatch.** Moneyness, mid_price_norm, intrinsic_value, and moneyness_peak_iv have ranges 10-100x larger than other features. The linear projection handles this implicitly but extreme values (2% of tokens) may get poor representation.

5. **Epoch time is 6-8 hours.** This limits iteration speed to 1-3 experiments per week. Acceptable for the first full run but will become a bottleneck if recipe tuning is needed.

### RECOMMENDATIONS (in priority order):

1. **LAUNCH FULL TRAINING IMMEDIATELY.** No blocking issues found. The GPU has been idle for 2+ hours. The data is ready. The model is validated. Every hour of delay is wasted compute time. The bugs found (zero dims, d_ff mismatch) are non-blocking and can be addressed in the next iteration.

2. **Monitor for overfitting.** At params/sample = 0.117, watch for val loss diverging from train loss after epoch 3-5. If val captured_return peaks before epoch 5, the model is overfitting and we should consider:
   - Increasing dropout to 0.30
   - Increasing weight_decay to 0.03
   - Reducing model size (V5-Tiny: d_model=96, intra_layers=1)

3. **Monitor loss component magnitudes.** During training, check the per-component loss values logged in the loss dict. If contrastive or confidence losses dominate or show erratic behavior, they should be disabled in the next run.

4. **For the NEXT token prep run (after this training completes):** Fill summary token dims 14-19 with additional aggregate features (e.g., volume-weighted delta, gamma-weighted OI, term structure slope, skew kurtosis, max single-contract volume fraction, call volume near ATM ratio). Fix price token dim 15.

5. **For the NEXT training run:** Consider the IntraDayEncoder day-loop optimization (batch all 5 days together) for a potential 2-3x throughput improvement.

6. **For future training recipes:** If overfitting is observed, try V5-Tiny (d_model=96, intra_layers=1, temporal_layers=1, backbone_dim=48) at ~300K params. If capacity is the bottleneck (val metrics plateau without overfitting), try V5-Medium (d_model=192, intra_layers=3, temporal_layers=2, backbone_dim=96) at ~2.5M params.
