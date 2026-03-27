# V5 Design Review — Critical Questions

## 1. Is the Model Size Right? (Overfitting vs Capacity)

### The Evidence

| Model | Params | Train Samples | Params/Sample | Data per Sample | Overfit? |
|-------|--------|---------------|---------------|-----------------|----------|
| V2 | 151K | 5.8M | 0.026 | 1,380 values (5d × 276) | No (38 epochs) |
| V3 | 136K | 5.8M | 0.023 | 1,260 values (5d × 210 + price) | No (38 epochs) |
| V4 | 4.77M | 5.8M | 0.82 | 3,072 values (256 × 12) | **YES (epoch 2)** |
| V5 | ~5M | ~5.8M* | 0.86 | 12,800 values (5 × 128 × 20) | **HIGH RISK** |

*V5 has more total samples (~11M) but many are from the same stocks on different days — effective independent samples may be closer to V4's 5.8M.

### The Problem

**V5 has almost the same params/sample ratio as V4 (0.86 vs 0.82).** V4 overfit catastrophically by epoch 2. V5 has 4x more data per sample (richer tokens), which helps — but the params/sample ratio is the primary overfitting predictor. V2/V3 had 0.023-0.026 params/sample and never overfit.

### The Risk

V5 at ~5M params may overfit just like V4. The 5-day temporal context gives the model more structure to learn from (reducing overfitting somewhat), but the fundamental imbalance remains: too many parameters for the number of independent training examples.

### Recommendation: Consider a Smaller V5

**Option A: V5-Small (recommended first attempt)**
- IntraDayEncoder: 2 layers instead of 4, d_model=128 instead of 256
- TemporalEncoder: 1 layer instead of 2
- backbone_dim: 64 instead of 128
- Estimated params: ~800K-1.2M
- Params/sample ratio: ~0.14-0.21 (between V3's 0.023 and V4's 0.82)
- Training time: ~2-3 hr/epoch instead of ~7 (faster iteration)

**Option B: V5-Medium**
- IntraDayEncoder: 3 layers, d_model=192
- TemporalEncoder: 2 layers
- backbone_dim: 96
- Estimated params: ~2.5M
- Params/sample ratio: ~0.43

**Option C: V5-Large (current design)**
- As designed: 4+2 layers, d_model=256, backbone=128
- ~5M params
- Params/sample ratio: ~0.86
- Highest risk of overfitting

**The smart approach**: Start with V5-Small. If it doesn't overfit and metrics plateau → scale up to V5-Medium. This is 2-3x faster per training run, meaning we can iterate on the training recipe 2-3x more experiments in the same GPU time.

---

## 2. Is the 5-Day Lookback Sufficient?

### What the Foresight Analysis Shows

The most predictive temporal features use 5-day and 10-day windows:
- `total_volume_mom_5d` — 5-day momentum
- `total_volume_mom_10d` — 10-day momentum
- `total_volume_zscore_20d` — 20-day z-score
- `put_call_volume_ratio_mom_5d` — 5-day sentiment shift

### Assessment

**5 days captures the 5-day momentum signals but misses the 10-day and 20-day patterns.** However:
- The price context token already carries 20-day return information (as scalar features)
- The 20-day z-score needs history but it can be computed as a feature in the price/summary token (which we do)
- Extending to 10+ days would 2x the data per sample, 2x memory, 2x training time
- Diminishing returns: most options action happens in the last 3-5 days before a move

**Verdict**: 5 days is a reasonable starting point. The price context token compensates for the missing 10/20-day lookback at the aggregate level. If V5-Small shows that temporal patterns improve metrics, we can extend to 7-10 days in a later iteration.

---

## 3. Is the Token Format Correct?

### What's Good
- 20-dim tokens capture more per contract than V4's 12 (IV percentile, volume percentile, IV/realized vol — all valuable)
- Surface summary token aggregates the full surface per day (ATM IV, skew, concentration)
- Price context token has 20 dimensions of historical price/volume features
- 128 tokens per day covers most stocks (few have >126 active contracts)

### What's Questionable

**a) Token Labeling (y_return, y_big)**
- y_return = 10-day forward return from day t
- y_big = 1.0 if |y_return| > 5%
- This is the same as V4 — no change. It's standard and correct.
- However: the model sees 5 days of history but predicts from only day t. Days t-4 through t-1 contribute to the embedding but the label is anchored to day t's close price. This is correct — the temporal encoder should learn "what happened in the last 5 days that predicts what happens in the NEXT 10 days from today."

**b) 128 Tokens Per Day — Is It Enough?**
- Most stocks: 20-80 active contracts → fits easily
- Liquid names (AAPL, SPY): 300-500+ contracts → truncated to 126 after price+summary tokens
- The priority selection (by volume then moneyness) keeps the most informative contracts
- **Risk**: For very liquid stocks, we lose OTM contracts that might carry signal (unusual activity on far OTM options is a classic big-move predictor)
- **Mitigation**: The surface summary token captures aggregate statistics even from contracts that didn't make the cut

**c) 5-Day Window Assembly — Day Alignment**
- Does the data correctly handle holidays/weekends? (Trading days only, not calendar days)
- Does the data handle stocks that didn't trade on all 5 days? (day_mask handles this)
- Are the 5 days truly consecutive TRADING days? → Need to verify in prepare_tokens.py

---

## 4. Will Training Be Fast Enough to Iterate?

### Training Time Estimates

| Config | Params | Tokens/Sample | Batch | Est. Time/Epoch | Epochs to Converge | Total |
|--------|--------|---------------|-------|-----------------|-------------------|-------|
| V5-Large (current) | ~5M | 5×128=640 | 256 | ~7 hr | 15-25 | 4-7 days |
| V5-Medium | ~2.5M | 5×128=640 | 384 | ~4 hr | 10-20 | 2-3 days |
| **V5-Small** | **~1M** | **5×128=640** | **512** | **~2 hr** | **10-15** | **1-1.5 days** |

**V5-Small gives results in 1-1.5 days.** V5-Large takes 4-7 days. If V5-Large overfits (likely given V4's history), those 4-7 days are wasted.

**The iteration cycle matters more than single-run quality:**
- V5-Small: 3-4 experiments per week (fast feedback)
- V5-Large: 1 experiment per week (slow, expensive failures)

At this stage of research (first V5 training), **fast iteration dominates**.

---

## 5. Summary: What Should Change Before Training

| Issue | Current V5 Design | Recommended Change | Priority |
|-------|-------------------|-------------------|----------|
| Model too large | ~5M params, 0.86 params/sample | **Start with V5-Small: ~1M params, d_model=128, 2+1 layers** | HIGH |
| Training time | ~7 hr/epoch | V5-Small: ~2 hr/epoch → 3x faster iteration | HIGH |
| Overfitting risk | Same ratio as V4 which overfit epoch 2 | V5-Small has 0.17 ratio → much safer | HIGH |
| 5-day lookback | 5 days | Keep 5 days for now — sufficient for first attempt | OK |
| Token format | 20-dim, 128/day | Keep — well designed | OK |
| Labels | y_return, y_big (same as V4) | Keep — standard, correct | OK |
| Loss function | 7 components | Consider simplifying for V5-Small: drop contrastive + confidence initially | MEDIUM |

### Proposed V5-Small Config

```python
# V5-Small — fast iteration, low overfitting risk
d_model = 128          # (was 256)
n_heads = 4            # (was 8)
n_layers_intra = 2     # (was 4)
n_layers_temporal = 1  # (was 2)
d_ff = 512             # (was 1024)
backbone_dim = 64      # (was 128)
dropout = 0.25         # (was 0.2, slightly higher for smaller model)
batch_size = 512       # (was 256, larger batch OK with smaller model)

# Simplified loss for first attempt:
loss = focal + direction_BCE + asymmetric_quantile + asymmetric_return
# Add contrastive + confidence AFTER baseline is established
```

**Estimated params**: ~1M
**Estimated training**: ~2 hr/epoch, 10-15 epochs, ~1-1.5 days total
**Params/sample**: ~0.17 (between V3's 0.023 and V4's 0.82 — safe zone)

---

## 6. The Manager's Process Failure

These questions should have been asked during ORIENT before the manager recommended implementing V5-Large. The manager:
1. Accepted the V5 design document at face value without questioning model size vs overfitting risk
2. Didn't compare V5's params/sample ratio to V4's (which already overfit)
3. Didn't estimate training time per epoch or iteration speed
4. Didn't consider a smaller first attempt to validate the temporal hypothesis faster

**The ORIENT phase should always ask**: "Is this design the fastest path to answering our hypothesis, or are we over-engineering before we have evidence?" Starting small and scaling up is always faster than starting large and discovering it overfits.
