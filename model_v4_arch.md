# V4 Model Architecture — Options Surface Transformer

## Overview

V4 is a transformer-based model that processes **raw options chain data** directly as variable-length token sequences. Unlike V1-V3 which use hand-crafted features, V4 lets the transformer learn patterns directly from raw contract-level data — moneyness, Greeks, implied volatility, volume, open interest.

The core hypothesis: a transformer with attention over the full options surface can detect subtle patterns (IV skew shifts, unusual volume at specific strikes, put-call ratio anomalies) that hand-crafted features miss.

---

## Data Pipeline: Raw CSV → Token Sequences

### Source Data

- **Options data**: Raw CSV files from `firstrate_processing/historic/`, one file per symbol per quarter
- **Price data**: Daily OHLCV from `fmp_processing/cache/historic/daily_prices/price_data.npz` (8,766 symbols × 4,025 dates)
- **Split index**: Train/val/test quarter assignments from `firstrate_learning/cache/index.json`

### Raw CSV Format (16 columns per contract)

```
trade_date, strike, expiry_date, type(c/p), bid, ask, mid, iv, dte, open_interest, volume, delta, gamma, vega, theta, rho
```

### Tokenization: CSV → 12-Dimensional Token

Each option contract becomes one **12-dimensional float vector**:

| Dim | Feature | Computation | Why |
|-----|---------|-------------|-----|
| 0 | Moneyness | strike / spot_price | Normalizes across different stock prices |
| 1 | Is Call | 1.0 if call, 0.0 if put | Option type indicator |
| 2 | DTE (normalized) | (expiry - trade_date).days / 365 | Time to expiration, scale-invariant |
| 3 | Log IV | log1p(max(iv, 0)) | Implied volatility, log-normalized |
| 4 | Log OI | log1p(max(open_interest, 0)) | Open interest, log-normalized |
| 5 | Log Volume | log1p(max(volume, 0)) | Trading volume, log-normalized |
| 6 | Delta | clipped [-1, 1] | First-order price sensitivity |
| 7 | Gamma | clipped [-10, 10] | Second-order price sensitivity |
| 8 | Vega | clipped [-10, 10] | Volatility sensitivity |
| 9 | Theta | clipped [-10, 10] | Time decay |
| 10 | Bid-Ask Spread | (ask - bid) / max(mid, 0.01), clipped [0, 5] | Liquidity indicator |
| 11 | Mid Price Norm | mid / spot_price | Normalized option price |

### Price Context Token (prepended)

A single **price context token** is prepended at position 0 of every sequence with recent price/volume history:

| Dim | Feature | Lookback |
|-----|---------|----------|
| 0 | 5-day return | (close[-1] - close[-6]) / close[-6] |
| 1 | 10-day return | (close[-1] - close[-11]) / close[-11] |
| 2 | 20-day return | (close[-1] - close[-21]) / close[-21] |
| 3 | 5-day realized volatility | std(daily_returns[-6:]) × √252 (annualized) |
| 4 | Daily return | (close[-1] - close[-2]) / close[-2] |
| 5 | Volume ratio | today_volume / mean(previous_volumes) |
| 6-11 | Zero-padded | — |

All values clipped to [-5, 5].

### Lookback Window

- **Options data**: Single day snapshot — all contracts available on that trade date. No multi-day lookback of options.
- **Price context**: Uses **20 trading days** (~1 month) of historical price data for returns and volatility features in the price token.
- **Forward horizon**: 10 trading days (~2 weeks) for return targets.

### Sequence Construction

For each stock-day:

1. Parse all option contracts from the CSV for that date
2. Filter: skip if fewer than 10 contracts
3. Sort contracts by **moneyness** (distance from ATM, ascending) — ATM-centric ordering
4. Truncate to 255 contracts (keep closest to ATM if more)
5. Prepend the price context token → total sequence length up to **256 tokens**
6. Pad shorter sequences with zeros

```
[Price Token] [ATM Call] [ATM Put] [Near OTM Call] [Near OTM Put] ... [Far OTM] [PAD] [PAD]
  position 0    pos 1      pos 2       pos 3           pos 4             ...       255
  type=1        type=2     type=3      type=2          type=3                     type=0
```

### Labels (Targets)

- **y_return**: Forward 10-day return = (price[t+10] - price[t]) / price[t]
- **y_big**: Binary — 1.0 if |y_return| > 5%, else 0.0

### Lookahead Prevention

- Spot price uses **previous day's close** (not today's) to avoid lookahead
- Forward return uses today's close vs close 10 days later

---

## Chunk Storage Format

Sequences are stored in compressed chunks (50,000 samples each):

```
tokens:      (n, 256, 12)  float32   — option contract tokens
token_types: (n, 256)      int64     — 0=pad, 1=price, 2=call, 3=put
mask:        (n, 256)      bool      — True=valid, False=padding
y_return:    (n,)          float32   — 10-day forward return
y_big:       (n,)          float32   — binary big move indicator
date_ints:   (n,)          int32     — days since epoch
seq_lens:    (n,)          int16     — actual sequence length before padding
```

Compressed with **zstandard level 3** → `.pt.zst` files.

**Total dataset**: 11,411,714 samples across 264 chunks (train: 5.8M/138, val: 2.8M/63, test: 2.8M/63).

---

## Model Architecture: OptionsSurfaceTransformer

```
Input: (batch=512, seq_len=256, token_dim=12)
  │
  ▼
Linear Projection: 12 → d_model=256
  │
  ▼
+ Continuous Positional Encoding (from moneyness col 0 + DTE col 2)
  │   Uses sinusoidal functions, not learned embeddings
  │   Handles any moneyness/DTE value (continuous, not discrete)
  │
+ Token Type Embedding (0=CLS, 1=price, 2=call, 3=put)
  │
  ▼
Prepend Learnable [CLS] Token → seq_len becomes 257
  │
  ▼
┌──────────────────────────────┐
│   Transformer Block × 6      │
│                              │
│   Pre-LayerNorm              │
│   Multi-Head Self-Attention  │
│     8 heads, d_model=256     │
│     SDPA (flash attention)   │
│   + Residual                 │
│                              │
│   Pre-LayerNorm              │
│   FFN: 256 → 1024 → 256     │
│     GELU activation          │
│   + Residual                 │
│   Dropout 0.2                │
└──────────────────────────────┘
  │
  ▼
Final LayerNorm
  │
  ▼
Extract [CLS] token output (position 0) → (batch, 256)
  │
  ▼
Project to backbone_dim=64 → (batch, 64)
  │   This is the per-stock embedding for downstream portfolio model
  │
  ▼
┌──────────────────────────────────────────────────────┐
│   Prediction Heads (all from backbone_dim=64)        │
│                                                      │
│   magnitude_head: 64 → 1 → sigmoid     → p_big      │
│   direction_head: 64 → 1 → sigmoid     → p_up       │
│   quantile_head:  64 → 3               → quantiles   │
│       (predicts 75th, 90th, 95th percentile returns) │
│   return_head:    64 → 1               → pred_return │
│       (raw return magnitude prediction)              │
└──────────────────────────────────────────────────────┘
  │
  ▼
Composite Score: p_big × p_up + 0.2 × sigmoid(q95) + 0.1 × tanh(pred_return)
  │   Used for ranking stocks — higher score = more likely big upward move
```

### Key Design Choices

- **Continuous positional encoding**: Moneyness and DTE are continuous values (not buckets). Sinusoidal encoding naturally handles any value range, unlike learned embeddings which need discrete positions.
- **[CLS] token pooling**: Following BERT, a learnable [CLS] token aggregates information from all options contracts via self-attention. This becomes the per-stock representation.
- **backbone_dim=64**: Compressed embedding for the downstream portfolio model. The transformer learns to distill the full options surface into 64 features.
- **Pre-norm**: LayerNorm before attention/FFN (not after). Better gradient flow for deep transformers.
- **Flash attention**: Uses PyTorch native `scaled_dot_product_attention` which auto-selects the fastest backend (flash, memory-efficient, or math).

### Parameters

- **Total**: 4,769,287 (4.77M) — 35x larger than V3 (136K)
- d_model=256, n_heads=8, n_layers=6, d_ff=1024, backbone_dim=64

---

## Training Recipe

### Loss Function (V4Loss)

Multi-task loss combining four objectives:

| Component | Weight | Loss Type | Target |
|-----------|--------|-----------|--------|
| Magnitude | 1.0 | Focal loss (α=0.5, γ=2.0) + label smoothing (0.05) | p_big: probability of >5% move |
| Direction | 0.5 | Binary cross-entropy | p_up: probability of positive return |
| Quantile | 0.3 | Pinball loss (quantiles 0.75, 0.90, 0.95) | Upper quantiles of return distribution |
| Return | 0.5 | Huber loss (δ=0.10) | Raw forward 10-day return |

**Why focal loss for magnitude**: Big moves (>5%) are rare (~15% of samples). Focal loss down-weights easy negatives and focuses on hard-to-classify near-threshold cases.

**Why quantile regression**: Captures the shape of the return distribution, not just the mean. The 95th percentile prediction identifies potential outlier returns.

### Optimizer & Schedule

- **AdamW**: lr=3e-4, weight_decay=0.01, betas=(0.9, 0.98)
- **Cosine warmup**: Linear warmup for 2,000 steps, then cosine decay to 10% of peak lr
- **Gradient clipping**: max_norm=1.0

### Performance Optimizations

- **AMP FP16**: `torch.amp.autocast('cuda', dtype=torch.float16)` + GradScaler
- **torch.compile**: `mode='reduce-overhead'` for kernel fusion — 20-30% speedup
- **ChunkInterleavedDataset**: Pools 10 chunks (~550MB), globally shuffles for regime mixing
- **GPU timing**: Logs data_ms vs gpu_ms every 50 batches, targeting >70% GPU util

### Early Stopping

- Patience: 15 epochs without improvement in captured return
- Checkpoints: `latest_checkpoint.pt` every epoch (full state), `best_model.pt` on val improvement

---

## Data Flow Summary

```
Raw CSV options files (firstrate_processing/historic/)
  + Daily price data (fmp_processing/cache/historic/daily_prices/)
        │
        ▼  prepare_tokens.py (parallel, 16 workers)
        │
  Parse CSV → 12-dim tokens per contract
  Sort by moneyness, truncate to 255
  Prepend price context token (5/10/20-day returns, vol, volume)
  Compute labels: y_return (10-day fwd), y_big (>5%)
        │
        ▼
  .pt.zst chunks (50K samples each, zstd compressed)
  11.4M total samples across 264 chunks
        │
        ▼  data_loader.py (ChunkLoader + IterableDataset)
        │
  Training: ChunkInterleavedDataset (10-chunk buffer, global shuffle)
  Val/Test: ChunkSequentialDataset (streaming, no shuffle)
  DataLoader: num_workers=0, batch_size=None (pre-batched at 512)
        │
        ▼  train.py (AMP FP16, torch.compile, GPU timing)
        │
  OptionsSurfaceTransformer: 6-layer, 8-head, d_model=256
  [CLS] token → backbone_dim=64 → 4 prediction heads
  V4Loss: focal + direction + quantile + return regression
  AdamW + cosine warmup, patience=15
        │
        ▼
  Per-run directory: models/run_YYYYMMDD_type/
    best_model.pt, latest_checkpoint.pt, run_meta.json, training_results.json
```

---

## V4 vs V3 Comparison

| Aspect | V3 | V4 |
|--------|----|----|
| Input | Hand-crafted features (fixed grid) | Raw option contracts (variable-length tokens) |
| Architecture | 3-layer backbone + heads | 6-layer transformer + [CLS] + heads |
| Parameters | 136K | 4.77M (35x) |
| Feature engineering | Extensive (flatten, aggregate) | Minimal (normalize, log-transform) |
| Lookback (options) | Multiple days stacked | Single day snapshot |
| Lookback (price) | Embedded in features | 20 days via price context token |
| Positional encoding | None (fixed feature order) | Continuous (moneyness + DTE) |
| Attention | None | Self-attention over full options surface |
| Best captured return | 0.0147 (38 epochs) | 0.0184 (5 epochs, pre-infrastructure) |

The key insight: V4 exceeded V3's best result **after just 1 epoch** (CR=0.0186, +26%), suggesting the raw-token transformer approach is fundamentally more capable.
