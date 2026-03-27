# V4 Model Improvements — Architecture Analysis & Enhancement Roadmap

## The Core Question: Does a Single-Day Snapshot Make Sense?

**Short answer: V4's single-day design is powerful but incomplete.** It achieves strong results (CR=0.0184, beating V3's 0.0147 after just 5 epochs), proving that raw options surface attention works. But it leaves significant performance on the table by not seeing temporal patterns — how the options surface evolved over recent days.

---

## What V4 Sees vs What It Misses

### What V4 Sees (Single-Day Snapshot)

On any given day, V4's transformer attends over up to 256 tokens representing the full options surface:

- **Every contract's Greeks** — delta, gamma, vega, theta capture the market's view of risk
- **Implied volatility** — the market's bet on future volatility per strike/expiry
- **Open interest and volume** — where money is positioned today
- **Bid-ask spreads** — liquidity and urgency of traders
- **Moneyness distribution** — how the smile/skew looks today
- **A single compressed price token** — 5/10/20-day returns and 5-day realized vol as 6 scalar values

The transformer can learn: "When ATM put volume is 3x normal AND IV skew is steep AND the stock just dropped 5% in 5 days → big move coming." This is powerful — it's why V4 beats V3 even with less temporal context.

### What V4 Misses (No Multi-Day Lookback)

V4 cannot see **how things changed** over time. The foresight analysis (`analyze_foresight.py`) found that the most predictive features involve **temporal comparison**:

| Predictive Feature | What It Measures | V4 Can See It? |
|---|---|---|
| `total_volume_mom_5d` | Is options volume accelerating? | No — only today's volume |
| `total_volume_zscore_20d` | Is today's volume extreme vs recent history? | No — no history to compare |
| `put_call_volume_ratio_mom_5d` | Is sentiment shifting from calls to puts? | No — only today's ratio |
| `put_call_volume_ratio_ema_10d` | Smoothed sentiment trend | No |
| IV curve change over 3 days | Is the smile steepening? | No — only today's IV |
| Volume surge on specific strikes | Did someone buy 10x normal volume on OTM calls this week? | Partially — today's volume, but not "vs yesterday" |

**The pattern**: Big moves are preceded by **changes** — momentum, z-scores, accelerations. A single snapshot shows the current state but not the trajectory. It's like reading a stock price without seeing the chart.

### How V1/V2/V3 Handle This

V1/V2/V3 use a **multi-day temporal encoder**:

```
V2/V3 Input: (batch, 5 days, 276 features)
                   │
                   ▼
Temporal diffs: day2-day1, day3-day2, etc. → captures changes
                   │
                   ▼
Causal Conv1d (kernel=3) → learns temporal patterns
                   │
                   ▼
Masked mean pooling → single representation
```

With 5 consecutive days × 276 features = 1,380 total values per stock, V2/V3 explicitly encode:
- How IV changed day-over-day
- Volume trends across the week
- Whether open interest is building or unwinding
- Momentum in all 276 features

V4 has ~3,072 values (256 × 12) but they're all from one day. More data per snapshot, less temporal context.

---

## V4's Strengths That Should Be Preserved

Before proposing changes, it's important to understand why V4 works so well despite the single-day limitation:

1. **Contract-level granularity**: V2/V3 aggregate options into 276 fixed features (e.g., "ATM call IV"). V4 sees every individual contract. The transformer can learn that "this specific OTM put at 0.8 moneyness, 45 DTE, with 10x normal volume" is a signal — V2/V3 would average this away.

2. **Variable-length sequences**: Different stocks have different numbers of options. V4 handles 10-contract stocks and 500-contract stocks equally. V2/V3 force everything into a fixed 276-dim grid.

3. **Continuous positional encoding**: Moneyness and DTE are encoded as continuous values, not discrete buckets. The transformer learns smooth functions over the options surface.

4. **Attention over the full surface**: Self-attention lets the model compare any contract to any other — "this call's IV vs that put's IV" — without feature engineering.

These are fundamental advantages. Any enhancement should **add temporal context without losing contract-level granularity**.

---

## Proposed Enhancements (Priority Order)

### Enhancement 1: Multi-Day Price Tokens (High Priority, Low Complexity)

**Problem**: The price token compresses 20 days of price history into 6 scalars (5d/10d/20d return, 5d vol, daily return, volume ratio). The transformer can't reconstruct temporal shape from these scalars.

**Solution**: Instead of one price token, add 3-5 price tokens representing consecutive days:

```
Current:  [price_token_today] [call_1] [call_2] ... [put_1] ...
Proposed: [price_t0] [price_t-1] [price_t-2] [call_1] [call_2] ... [put_1] ...
```

Each daily price token carries full 12-dim features:
- 5d/10d/20d returns as of that day
- Realized vol as of that day
- Volume ratio as of that day
- **New**: IV term structure summary (ATM 30d IV, ATM 60d IV if available from options)
- **New**: Put-call volume ratio as of that day
- **New**: Max unusual volume z-score across all contracts

The transformer can then attend between price tokens to learn: "price_t0 shows volume surge but price_t-2 was calm → something happened in the last 2 days."

**Impact on sequence length**: Uses 3-5 tokens out of 256. Minimal impact — reduce max_opts from 255 to 251.

**Implementation**: Modify `prepare_tokens.py` to compute price features for t-0, t-1, t-2. Add `token_age` to token_types or encode in positional encoding.

### Enhancement 2: Temporal Positional Encoding (High Priority, Low Complexity)

**Problem**: `ContinuousPositionalEncoding` only encodes moneyness + DTE. It has no concept of "when" a token's data is from.

**Solution**: Add a third dimension — `token_age` — to the positional encoding:

```python
# Current (2D):
pos = torch.stack([moneyness, dte], dim=-1)  # (B, L, 2)

# Proposed (3D):
pos = torch.stack([moneyness, dte, token_age], dim=-1)  # (B, L, 3)
# token_age: 0.0 for today's tokens, 0.33 for yesterday, 0.66 for 2 days ago, etc.
```

This enables the transformer to distinguish "today's ATM call" from "yesterday's ATM call" even if they have similar features. The model can learn temporal patterns directly through attention.

**Implementation**: Modify `ContinuousPositionalEncoding.proj` from 2→d_model to 3→d_model. Add `token_age` as a new field in the data preparation.

### Enhancement 3: Historical Options Summary Tokens (Medium Priority, Medium Complexity)

**Problem**: Even with multi-day price tokens, V4 doesn't see how individual options behaved in prior days. Was this OTM call's volume surging all week, or is today the first spike?

**Solution**: Add 2-3 "historical summary tokens" that aggregate prior days' options activity:

```
[price_t0] [price_t-1] [hist_summary_t-1] [hist_summary_t-2] [call_1] [call_2] ...
```

Each historical summary token (12-dim) encodes:
- Total options volume yesterday (log-scaled)
- Put-call volume ratio yesterday
- ATM implied vol yesterday
- 25-delta skew yesterday
- Most active strike moneyness yesterday
- Volume z-score vs 20-day avg
- OI change vs yesterday
- IV term structure slope
- Additional aggregated signals

These are not per-contract tokens (that would be too many) but per-day aggregate summaries. The transformer compares "yesterday's surface summary" to "today's individual contracts."

**Impact**: Uses 2-4 tokens. Combined with multi-day price tokens, total overhead is ~7 tokens out of 256 — still 249 contracts available.

### Enhancement 4: Utilize Unused Price Token Dimensions (Quick Win)

**Problem**: The price token has 12 dimensions but only uses 6 (indices 6-11 are zeros).

**Solution**: Fill the remaining 6 dimensions with high-value features:

| Dim | Current | Proposed |
|-----|---------|----------|
| 6 | Zero | 10-day realized volatility (annualized) |
| 7 | Zero | 5-day volume momentum (today_vol / 5d_avg_vol) |
| 8 | Zero | 10-day volume momentum |
| 9 | Zero | Price acceleration (5d_ret - prev_5d_ret) |
| 10 | Zero | Daily return 2 days ago (close[-2] vs close[-3]) |
| 11 | Zero | Daily return 3 days ago (close[-3] vs close[-4]) |

This is a **zero-cost improvement** — no architecture changes, no new tokens, just more information in the existing price token. The transformer already processes this token; we're just giving it more signal.

### Enhancement 5: Contrastive Embedding Loss (Medium Priority, Low Complexity)

**Problem**: The 64-dim backbone embedding is trained only by the prediction heads. It may not be maximally discriminative for the downstream portfolio model.

**Solution**: Add a contrastive loss term that explicitly shapes the embedding space:
- Pull embeddings closer for stocks with similar future returns
- Push apart embeddings for stocks with different future returns

```python
# In V4Loss:
# Big movers (|return| > 10%) should cluster together
# Small movers (|return| < 2%) should be far from big movers
loss_contrastive = cosine_similarity(big_embeddings) - cosine_similarity(big_vs_small)
```

**Why this helps the portfolio model**: The portfolio model receives 64-dim embeddings and must rank stocks. If big winners cluster in embedding space, the portfolio model's job is much easier.

This is Priority P3 from `direction.md` — "Improve embedding informativeness through contrastive learning."

### Enhancement 6: Asymmetric Return Weighting (Medium Priority, Low Complexity)

**Problem**: The Huber loss for return regression treats a +5% prediction error on a +40% stock the same as a +5% error on a +2% stock. But for portfolio construction, accurately predicting the magnitude of big winners matters far more.

**Solution**: Weight the return loss by the absolute size of the actual return:

```python
# Current:
loss_return = F.huber_loss(pred_return, y_return, delta=0.10)

# Proposed:
errors = F.huber_loss(pred_return, y_return, delta=0.10, reduction='none')
weights = 1.0 + 2.0 * (y_return.abs() > 0.10).float()  # 3x weight for big moves
loss_return = (weights * errors).mean()
```

**Why**: The foresight model gets 170% return by picking the very best stocks. Accurately distinguishing +10% from +40% stocks is where the portfolio value comes from. The model should allocate more learning capacity to getting these right.

### Enhancement 7: Transformer-in-Transformer for Multi-Day (Low Priority, High Complexity)

**Problem**: Even with multi-day tokens in the same sequence, the transformer treats them all as one flat set. There's no explicit hierarchical structure.

**Solution**: Two-level architecture:
- **Inner transformer**: Process each day's options surface independently → per-day embedding
- **Outer transformer**: Process the sequence of daily embeddings → temporal patterns

```
Day t:   [price, calls, puts] → inner_transformer → emb_t    (64-dim)
Day t-1: [price, calls, puts] → inner_transformer → emb_t-1  (64-dim)
Day t-2: [price, calls, puts] → inner_transformer → emb_t-2  (64-dim)

[emb_t, emb_t-1, emb_t-2] → outer_transformer → final_embedding (64-dim)
                                                    │
                                                    ▼
                                               Prediction heads
```

This is architecturally clean but significantly more complex — requires multi-day token data in each sample, 3x more data per sample, and a two-stage forward pass.

**Recommendation**: Defer this to V5. Enhancements 1-6 can be implemented within V4's current architecture and provide most of the benefit.

---

## Enhancement Impact Estimates

| Enhancement | Complexity | Data Change? | Architecture Change? | Expected Impact |
|---|---|---|---|---|
| 1. Multi-day price tokens | Low | Yes (prepare_tokens) | No (just more tokens) | High — restores temporal price context |
| 2. Temporal positional encoding | Low | Yes (add token_age) | Small (3→d_model proj) | Medium — helps distinguish temporal tokens |
| 3. Historical options summary | Medium | Yes (new summary tokens) | No | High — captures IV/volume momentum |
| 4. Fill price token dims | Trivial | Yes (prepare_tokens) | No | Medium — free information, zero cost |
| 5. Contrastive embedding loss | Low | No | Small (loss term) | Medium — better embeddings for portfolio |
| 6. Asymmetric return weighting | Trivial | No | Small (loss weights) | Low-Medium — better tail accuracy |
| 7. Transformer-in-Transformer | High | Yes (multi-day) | Major (new architecture) | High but risky — defer to V5 |

**Recommended implementation order**: 4 → 6 → 5 → 1 → 2 → 3 → 7

Start with zero-cost improvements (4, 6, 5), then add temporal context (1, 2, 3). Save the architectural overhaul (7) for V5.

---

## How Each Enhancement Bridges the Foresight Gap

The foresight model achieves 170% in 50 days because it knows the exact 10-day forward return. The backbone model must approximate this knowledge from observable signals. Here's how each enhancement helps:

```
Current V4:  captures ~1.84% per 10-day period (from single-day snapshot)
             ├── Contract-level attention: learns options surface structure
             └── Price context: 5/10/20-day returns (compressed)

Enhancement 4 (fill dims):     +0.1-0.2% estimated
  └── More price features in existing token (vol momentum, acceleration)

Enhancement 6 (asymmetric loss): +0.05-0.1% estimated
  └── Better tail accuracy → picks bigger winners

Enhancement 5 (contrastive):    +0.1-0.2% estimated
  └── Cleaner embedding space → portfolio model ranks better

Enhancement 1 (multi-day price): +0.2-0.5% estimated
  └── Restores temporal price context → sees momentum, mean reversion

Enhancement 2 (temporal PE):     +0.1-0.2% estimated (combined with #1)
  └── Transformer distinguishes today vs yesterday tokens

Enhancement 3 (options summary): +0.3-0.5% estimated
  └── Captures IV/volume momentum → the biggest predictive features

Total potential:  2.7-3.5% per 10-day period → ~90-130% annualized
                  (vs current 1.84% → ~58% annualized)
```

These are rough estimates. The actual impact depends on how well the transformer leverages the new information, overfitting risk (more features = more parameters), and whether the training recipe needs adjustment.

---

## Relationship to direction.md Priorities

| direction.md Priority | Enhancement | Status |
|---|---|---|
| P1: Improve return magnitude prediction | #6 (asymmetric loss), #5 (contrastive) | Ready to implement |
| P2: Add price/momentum features | #4 (fill dims), #1 (multi-day price), #3 (options summary) | Ready to implement |
| P3: Contrastive learning for embeddings | #5 (contrastive loss) | Ready to implement |
| P4: Remove dead features (24% zeroed) | N/A — V4 bypasses via raw tokens | Already solved |
| P5: Calibration (temperature, smoothing) | Label smoothing already in V4Loss (0.05) | Partially done |
| P6: Ensemble/SWA for stability | Not addressed yet | Defer to after V4 training stabilizes |

---

## Summary

V4's single-day snapshot design is **architecturally sound but temporally limited**. It proves that raw-token attention over the options surface is more powerful than hand-crafted features (beating V3 after 1 epoch). But it leaves ~40-60% of potential performance on the table by not seeing temporal patterns.

The highest-leverage improvements are:
1. **Fill the unused price token dimensions** (free performance)
2. **Add multi-day price tokens** (restore temporal context the transformer can attend over)
3. **Add historical options summary tokens** (capture IV/volume momentum — the strongest predictive signals from foresight analysis)

These can be implemented within V4's current architecture without a major redesign. The estimated combined impact is roughly doubling the captured return from ~1.84% to ~3.5% per 10-day period, or from ~58% to ~130% annualized.
