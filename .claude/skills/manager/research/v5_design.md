# V5 Architecture Design — Temporal Options Surface Transformer

## Design Philosophy

V5 is a ground-up redesign that combines V4's strengths (raw contract-level attention, continuous positional encoding, variable-length sequences) with V2/V3's strengths (multi-day temporal context, temporal differencing) and adds everything that was missing from all versions: multi-day options evolution, hierarchical attention, contrastive embedding training, and a training recipe explicitly designed to maximize captured return.

**Core insight from all prior versions**: V4 proved that raw-token attention over the options surface beats hand-crafted features. V2/V3 proved that temporal context matters. The foresight analysis proved that momentum and z-scores of options activity are the strongest predictive signals. V5 fuses all three.

---

## Architecture Overview

```
                     ┌─────────────────────────────────────────┐
                     │        V5: Temporal Options Surface      │
                     │             Transformer (TOST)           │
                     └─────────────────────────────────────────┘

Input: 5 consecutive days of raw options chain + price data per stock

Day t-4:  [price_token] [contract_1] [contract_2] ... [contract_N]  (up to 128 tokens)
Day t-3:  [price_token] [contract_1] [contract_2] ... [contract_N]  (up to 128 tokens)
Day t-2:  [price_token] [contract_1] [contract_2] ... [contract_N]  (up to 128 tokens)
Day t-1:  [price_token] [contract_1] [contract_2] ... [contract_N]  (up to 128 tokens)
Day t:    [price_token] [contract_1] [contract_2] ... [contract_N]  (up to 128 tokens)

                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │   Stage 1: Intra-Day Encoder   │
                    │   (Per-Day Surface Attention)   │
                    │                               │
                    │   4-layer transformer per day   │
                    │   Shared weights across days    │
                    │   [CLS] pooling → 256-dim      │
                    └───────────────────────────────┘
                                    │
                            5 × (256-dim) daily embeddings
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │   Stage 2: Temporal Encoder     │
                    │   (Cross-Day Attention)         │
                    │                               │
                    │   2-layer temporal transformer  │
                    │   + temporal diffs (velocity)   │
                    │   + temporal positional enc     │
                    │   [CLS] pooling → 256-dim      │
                    └───────────────────────────────┘
                                    │
                              256-dim fused embedding
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │   Stage 3: Prediction Heads    │
                    │                               │
                    │   backbone → 128-dim           │
                    │                               │
                    │   magnitude_head → p_big       │
                    │   direction_head → p_up        │
                    │   quantile_head → 5 quantiles  │
                    │   return_head → pred_return    │
                    │   regime_head → regime_embed   │
                    │                               │
                    │   Composite score              │
                    └───────────────────────────────┘
```

---

## Stage 1: Intra-Day Surface Encoder

### Purpose

Process a single day's options surface — all contracts visible on that day — into a rich per-day embedding. This is conceptually similar to V4's transformer but operates on each day independently, with shared weights.

### Input: Per-Day Token Sequence

Each day has up to **128 tokens** (reduced from V4's 256 to fit 5 days in memory):

**Token types:**
- 1 **price context token** (full 20-dim — see below)
- 1 **surface summary token** (aggregate options statistics)
- Up to 126 **option contract tokens** (20-dim each — expanded from V4's 12)

### Expanded Token Dimensionality: 20-dim

V4 used 12 dimensions per token. V5 expands to **20 dimensions** to encode more signal per contract:

| Dim | Feature | Source | Why New/Changed |
|-----|---------|--------|-----------------|
| 0 | Moneyness | strike / spot | Same as V4 |
| 1 | Is Call | 1.0 / 0.0 | Same as V4 |
| 2 | DTE normalized | days_to_expiry / 365 | Same as V4 |
| 3 | Log IV | log1p(iv) | Same as V4 |
| 4 | Log OI | log1p(open_interest) | Same as V4 |
| 5 | Log Volume | log1p(volume) | Same as V4 |
| 6 | Delta | clipped [-1, 1] | Same as V4 |
| 7 | Gamma | clipped [-10, 10] | Same as V4 |
| 8 | Vega | clipped [-10, 10] | Same as V4 |
| 9 | Theta | clipped [-10, 10] | Same as V4 |
| 10 | Bid-Ask Spread | (ask-bid)/max(mid, 0.01) | Same as V4 |
| 11 | Mid Price Norm | mid / spot | Same as V4 |
| 12 | **IV Percentile** | rank(iv) / n_contracts | **NEW**: Where does this contract's IV sit relative to all contracts today? Captures smile shape. |
| 13 | **Volume Percentile** | rank(volume) / n_contracts | **NEW**: Relative volume ranking — identifies unusual activity. |
| 14 | **OI Percentile** | rank(oi) / n_contracts | **NEW**: Relative positioning concentration. |
| 15 | **IV / Realized Vol** | iv / realized_vol_20d | **NEW**: The volatility risk premium for this contract. High ratio = options are expensive relative to actual movement. |
| 16 | **Moneyness Distance from Peak IV** | moneyness - moneyness_of_max_iv | **NEW**: Position relative to the IV smile peak. Captures skew structure. |
| 17 | **Expiry Bucket** | 0=weekly, 0.33=monthly, 0.66=quarterly, 1.0=LEAPS | **NEW**: Discretized maturity category. Weekly options behave differently from LEAPS. |
| 18 | **Intrinsic Value Ratio** | max(0, spot-strike)/spot for calls | **NEW**: How much of the option price is intrinsic vs time value. |
| 19 | **Log Bid-Ask Ratio** | log(ask/max(bid, 0.01)) | **NEW**: Asymmetric liquidity measure — wide ratio means low conviction. |

### Price Context Token (20-dim, fully utilized)

| Dim | Feature | Lookback |
|-----|---------|----------|
| 0 | 1-day return | 1 day |
| 1 | 3-day return | 3 days |
| 2 | 5-day return | 5 days |
| 3 | 10-day return | 10 days |
| 4 | 20-day return | 20 days |
| 5 | 5-day realized vol (annualized) | 5 days |
| 6 | 10-day realized vol | 10 days |
| 7 | 20-day realized vol | 20 days |
| 8 | Volume ratio (today / 20d avg) | 20 days |
| 9 | Volume momentum (5d avg / 20d avg) | 20 days |
| 10 | Price acceleration (5d_ret - prev_5d_ret) | 10 days |
| 11 | High-low range (normalized) | 1 day |
| 12 | Gap (open vs prev close, normalized) | 1 day |
| 13 | RSI-like (up_moves / total_moves, 14d) | 14 days |
| 14 | ATM IV (30-day interpolated) | Today |
| 15 | IV term structure slope (60d IV - 30d IV) | Today |
| 16 | Put-call volume ratio | Today |
| 17 | Put-call OI ratio | Today |
| 18 | Total options volume (log) | Today |
| 19 | Max volume z-score across all contracts | Today + 20d history |

### Surface Summary Token (20-dim, new in V5)

An aggregate token that summarizes the entire options surface for the day. Captures structure that individual contracts can't express alone:

| Dim | Feature |
|-----|---------|
| 0 | Number of active contracts (log-scaled) |
| 1 | Call/put contract count ratio |
| 2 | ATM IV (closest to moneyness=1.0) |
| 3 | 25-delta put IV (OTM put skew) |
| 4 | 25-delta call IV (OTM call skew) |
| 5 | Skew: 25d put IV - 25d call IV |
| 6 | Term structure: weighted avg DTE |
| 7 | Volume-weighted moneyness (where is the action?) |
| 8 | OI-weighted moneyness (where is the positioning?) |
| 9 | Total volume / total OI (turnover rate) |
| 10 | Max single-contract volume / total volume (concentration) |
| 11 | Bid-ask spread median (market-wide liquidity) |
| 12 | IV dispersion (std of IV across contracts) |
| 13 | Volume Herfindahl index (concentration measure) |
| 14-19 | Reserved for learned features or sector/market context |

### Intra-Day Transformer Architecture

```python
class IntraDayEncoder(nn.Module):
    """Process one day's options surface into a single embedding."""

    def __init__(self, token_dim=20, d_model=256, n_heads=8, n_layers=4, max_seq=128):
        self.token_proj = nn.Linear(token_dim, d_model)
        self.pos_enc = ContinuousPositionalEncoding3D(d_model)  # moneyness + DTE + IV_percentile
        self.type_embed = nn.Embedding(5, d_model)  # 0=CLS, 1=price, 2=summary, 3=call, 4=put
        self.cls_token = nn.Parameter(torch.randn(1, 1, d_model) * 0.02)
        self.blocks = nn.ModuleList([
            TransformerBlock(d_model, n_heads, d_model * 4, dropout=0.15)
            for _ in range(n_layers)
        ])
        self.norm = nn.LayerNorm(d_model)

    def forward(self, tokens, token_types, mask):
        # tokens: (B, L, 20), token_types: (B, L), mask: (B, L)
        x = self.token_proj(tokens)                         # (B, L, 256)
        x = x + self.pos_enc(tokens[:,:,0], tokens[:,:,2], tokens[:,:,12])  # moneyness, DTE, IV_pctile
        x = x + self.type_embed(token_types)

        # Prepend [CLS]
        cls = self.cls_token.expand(x.size(0), -1, -1)
        x = torch.cat([cls, x], dim=1)

        # 4 transformer blocks
        for block in self.blocks:
            x = block(x, key_padding_mask=...)

        x = self.norm(x)
        return x[:, 0, :]  # [CLS] output: (B, 256)
```

**Key differences from V4:**
- **4 layers instead of 6** — lighter per day since we stack 5 days
- **3D positional encoding** — moneyness + DTE + IV percentile (V4 used only moneyness + DTE)
- **5 token types** — CLS, price, surface summary, call, put (V4 had 4)
- **128 max tokens per day** instead of 256 — accommodates 5-day stack
- **Shared weights** across all 5 days — same encoder, applied 5 times

---

## Stage 2: Temporal Encoder

### Purpose

Process the sequence of 5 daily embeddings to learn temporal patterns — how the options surface evolved over the lookback window. This is where V5 captures what V4 completely missed: momentum, trend changes, volatility regime shifts.

### Architecture

```python
class TemporalEncoder(nn.Module):
    """Cross-day attention over daily surface embeddings."""

    def __init__(self, d_model=256, n_heads=8, n_layers=2, n_days=5):
        self.day_proj = nn.Linear(d_model, d_model)
        self.temporal_pe = nn.Embedding(n_days, d_model)  # learned per-day position
        self.diff_proj = nn.Linear(d_model, d_model)      # project temporal diffs
        self.cls_token = nn.Parameter(torch.randn(1, 1, d_model) * 0.02)
        self.blocks = nn.ModuleList([
            TransformerBlock(d_model, n_heads, d_model * 4, dropout=0.15)
            for _ in range(n_layers)
        ])
        self.norm = nn.LayerNorm(d_model)

    def forward(self, day_embeddings, day_mask):
        # day_embeddings: (B, 5, 256) — from IntraDayEncoder applied 5 times
        # day_mask: (B, 5) — which days have valid data

        B, T, D = day_embeddings.shape

        # Temporal positional encoding (learned, day 0=oldest, day 4=today)
        positions = torch.arange(T, device=day_embeddings.device)
        x = self.day_proj(day_embeddings) + self.temporal_pe(positions)

        # Temporal diffs: velocity of surface change
        diffs = torch.zeros_like(x)
        diffs[:, 1:, :] = x[:, 1:, :] - x[:, :-1, :]
        x = x + self.diff_proj(diffs)  # Add velocity information

        # Prepend temporal [CLS]
        cls = self.cls_token.expand(B, -1, -1)
        x = torch.cat([cls, x], dim=1)  # (B, 6, 256)

        # 2 transformer blocks — cross-day attention
        for block in self.blocks:
            x = block(x, key_padding_mask=...)

        x = self.norm(x)
        return x[:, 0, :]  # temporal [CLS]: (B, 256)
```

### What the Temporal Encoder Learns

The 2-layer cross-day transformer can learn:

- **"IV spiked yesterday but contracts are calm today"** → false alarm, no big move
- **"Volume has been building for 3 consecutive days on OTM calls"** → accumulation signal
- **"The surface embedding changed dramatically between day t-2 and t-1"** → something happened
- **"Today's surface looks like day t-3 but different from yesterday"** → mean reversion pattern
- **"The daily embedding is drifting consistently in one direction"** → momentum

The temporal diffs explicitly encode **velocity** — how fast things are changing. The transformer over diffs can learn **acceleration** — whether the rate of change is increasing or decreasing.

### Why 5 Days?

- **V2/V3 used 5-day lookback** and showed it captures useful temporal patterns (causal Conv1d over 5 timesteps)
- **5 trading days = 1 week** — captures weekly patterns (Monday effect, Friday positioning)
- **More than 5 days** adds marginal value but significantly increases memory and compute (5 × 128 = 640 total tokens is already large)
- **The price context tokens already carry 20-day history** — the temporal encoder adds 5-day granular options evolution on top of that

---

## Stage 3: Prediction Heads

### Backbone Projection

```python
class BackboneProjection(nn.Module):
    def __init__(self, d_model=256, backbone_dim=128):
        self.proj = nn.Sequential(
            nn.Linear(d_model, d_model),
            nn.LayerNorm(d_model),
            nn.GELU(),
            nn.Dropout(0.15),
            nn.Linear(d_model, backbone_dim),
            nn.LayerNorm(backbone_dim),
        )

    def forward(self, x):
        return self.proj(x)  # (B, 128)
```

**128-dim backbone** (doubled from V4's 64) — the downstream portfolio model receives a richer per-stock representation. This is justified because V5 encodes significantly more information (5 days × full options surface + temporal dynamics).

### Prediction Heads

```python
# From 128-dim backbone embedding:

magnitude_head:  128 → 64 → 1 → sigmoid       → p_big (>5% move probability)
direction_head:  128 → 64 → 1 → sigmoid       → p_up (positive return probability)
quantile_head:   128 → 64 → 5                  → quantiles [0.50, 0.75, 0.90, 0.95, 0.99]
return_head:     128 → 64 → 1                  → pred_return (raw 10-day return)
confidence_head: 128 → 64 → 1 → sigmoid       → p_confidence (model's confidence)
```

**New heads vs V4:**
- **5 quantiles** instead of 3 — adds median (0.50) and extreme tail (0.99). The 99th percentile is critical for identifying +30-40% movers that the foresight model captures.
- **Confidence head** — the model predicts how certain it is. Low confidence predictions get down-weighted in the score. This is a form of learned uncertainty quantification.

### Composite Score (Improved)

```python
# V4 score: p_big * p_up + 0.2 * sigmoid(q95) + 0.1 * tanh(pred_return)
# V5 score: more sophisticated, learned weighting

score = (
    p_big * p_up * p_confidence              # base: big × up × confident
    + 0.15 * sigmoid(quantiles[:, 3])        # 95th percentile
    + 0.10 * sigmoid(quantiles[:, 4])        # 99th percentile (extreme upside)
    + 0.10 * tanh(pred_return)               # magnitude prediction
    - 0.05 * relu(quantiles[:, 0])           # penalize high median (already priced in)
)
```

The score now explicitly values:
- **Extreme tail prediction** (99th percentile) — the biggest winners
- **Confidence gating** — uncertain predictions contribute less
- **Median penalty** — a stock predicted to have +3% median return but +40% 99th percentile is more interesting than one with +10% median (which is probably already priced in)

---

## V5 Loss Function

### Multi-Task Loss with Contrastive and Calibration Components

```python
class V5Loss(nn.Module):
    def __init__(self):
        # Task weights
        self.w_mag = 1.0           # magnitude (focal)
        self.w_dir = 0.5           # direction (BCE)
        self.w_quant = 0.4         # quantile (pinball, 5 quantiles)
        self.w_ret = 0.8           # return regression (asymmetric Huber)
        self.w_conf = 0.2          # confidence calibration
        self.w_contrast = 0.15     # contrastive embedding
        self.w_emb_var = 0.1       # embedding variance regularization

    def forward(self, outputs, y_return, y_big, embedding):
        # 1. Focal loss for magnitude (same as V4, with label smoothing)
        loss_mag = focal_loss(outputs['p_big'], y_big, alpha=0.5, gamma=2.0, smoothing=0.05)

        # 2. Direction BCE
        y_up = (y_return > 0).float()
        loss_dir = F.binary_cross_entropy(outputs['p_up'], y_up)

        # 3. Quantile regression (5 quantiles, asymmetric weights)
        taus = [0.50, 0.75, 0.90, 0.95, 0.99]
        tau_weights = [0.5, 1.0, 1.5, 2.5, 4.0]  # Heavily weight upper tail
        loss_quant = asymmetric_pinball(outputs['quantiles'], y_return, taus, tau_weights)

        # 4. Asymmetric return regression (weight big moves 3x)
        errors = F.huber_loss(outputs['pred_return'], y_return, delta=0.10, reduction='none')
        move_weight = torch.where(y_return.abs() > 0.10, 3.0, 1.0)
        loss_ret = (move_weight * errors).mean()

        # 5. Confidence calibration loss
        # Confidence should be high when the model is correct, low when wrong
        correct = ((outputs['p_big'] > 0.5).float() == y_big).float()
        loss_conf = F.binary_cross_entropy(outputs['p_confidence'], correct.detach())

        # 6. Contrastive embedding loss
        loss_contrast = contrastive_embedding_loss(embedding, y_return)

        # 7. Embedding variance (prevent collapse)
        emb_var = embedding.var(dim=0).mean()
        loss_emb_var = F.relu(1.0 - emb_var)

        total = (
            self.w_mag * loss_mag +
            self.w_dir * loss_dir +
            self.w_quant * loss_quant +
            self.w_ret * loss_ret +
            self.w_conf * loss_conf +
            self.w_contrast * loss_contrast +
            self.w_emb_var * loss_emb_var
        )

        return total, {
            'mag': loss_mag.item(), 'dir': loss_dir.item(),
            'quant': loss_quant.item(), 'ret': loss_ret.item(),
            'conf': loss_conf.item(), 'contrast': loss_contrast.item(),
            'emb_var': emb_var.item(),
        }
```

### Contrastive Embedding Loss (New)

```python
def contrastive_embedding_loss(embedding, y_return, temperature=0.1):
    """
    Pull embeddings of similar-return stocks together,
    push embeddings of different-return stocks apart.

    Uses return magnitude buckets:
      big_up:   return > +10%
      small:    |return| < 2%
      big_down: return < -10%
    """
    B = embedding.size(0)

    big_up = y_return > 0.10
    big_down = y_return < -0.10
    small = y_return.abs() < 0.02

    # Normalized embeddings for cosine similarity
    z = F.normalize(embedding, dim=1)

    # Similarity matrix
    sim = torch.mm(z, z.t()) / temperature  # (B, B)

    # Positive pairs: same return bucket
    pos_mask = (big_up.unsqueeze(0) & big_up.unsqueeze(1)) | \
               (big_down.unsqueeze(0) & big_down.unsqueeze(1)) | \
               (small.unsqueeze(0) & small.unsqueeze(1))
    pos_mask.fill_diagonal_(False)

    # Negative pairs: different return buckets
    neg_mask = (big_up.unsqueeze(0) & small.unsqueeze(1)) | \
               (small.unsqueeze(0) & big_up.unsqueeze(1)) | \
               (big_up.unsqueeze(0) & big_down.unsqueeze(1))

    if pos_mask.sum() > 0 and neg_mask.sum() > 0:
        pos_sim = sim[pos_mask].mean()
        neg_sim = sim[neg_mask].mean()
        return -pos_sim + neg_sim  # maximize pos similarity, minimize neg similarity

    return torch.tensor(0.0, device=embedding.device)
```

### Asymmetric Quantile Loss (Enhanced)

```python
def asymmetric_pinball(pred_quantiles, y_return, taus, weights):
    """
    Pinball loss with per-quantile weights.
    Upper quantiles (0.95, 0.99) get 2.5x and 4x weight
    because accurately predicting extreme upside is most valuable.
    """
    total = 0
    for i, (tau, w) in enumerate(zip(taus, weights)):
        errors = y_return - pred_quantiles[:, i]
        loss = torch.where(
            errors >= 0,
            tau * errors,
            (tau - 1) * errors
        )
        total += w * loss.mean()
    return total / sum(weights)
```

---

## Data Preparation: V5 Token Format

### Multi-Day Sample Structure

Each training sample contains **5 consecutive days** of options data for one stock:

```python
sample = {
    # Per-day token sequences (5 days × up to 128 tokens × 20 features)
    'tokens': np.zeros((5, 128, 20), dtype=np.float32),
    'token_types': np.zeros((5, 128), dtype=np.int64),
    'masks': np.zeros((5, 128), dtype=bool),
    'day_mask': np.zeros(5, dtype=bool),  # which days have valid data

    # Labels (same as V4)
    'y_return': float,   # 10-day forward return from day t
    'y_big': float,      # 1.0 if |y_return| > 5%

    # Metadata
    'date_int': int,     # day t
    'symbol_id': int,
}
```

### Handling Missing Days

Not all stocks have options data every day. The `day_mask` field indicates which of the 5 days have valid data:

```python
day_mask = [True, True, False, True, True]  # day t-2 has no data
# The temporal encoder uses the mask to skip missing days
# Intra-day encoder only processes days where day_mask=True
```

This is handled naturally by the attention mask — the temporal transformer simply doesn't attend to missing days.

### Token Selection Per Day (128 tokens)

With 128 tokens per day instead of V4's 256, we need smarter selection:

1. Always include: price token (1) + surface summary token (1) = 2 fixed tokens
2. Remaining 126 slots for option contracts
3. **Priority selection** (if >126 contracts):
   - Keep all contracts with volume > 0 (active today)
   - From remaining, keep by moneyness proximity to ATM
   - Within same moneyness band, prefer shorter DTE
4. Sort by moneyness (ATM-centric, same as V4)

Most stocks have <126 contracts with volume, so truncation is rare. For liquid names like AAPL with 500+ contracts, the selection focuses on where the action is.

---

## Training Recipe

### Optimizer & Schedule

```python
optimizer = torch.optim.AdamW(
    model.parameters(),
    lr=2e-4,          # Slightly lower than V4 (3e-4) due to larger model
    weight_decay=0.02, # Slightly higher regularization
    betas=(0.9, 0.98),
)

# Cosine warmup with step-level scheduling
warmup_steps = 3000     # More warmup for larger model
total_steps = max_epochs * batches_per_epoch

def lr_schedule(step):
    if step < warmup_steps:
        return step / warmup_steps
    progress = (step - warmup_steps) / (total_steps - warmup_steps)
    return max(0.05, 0.5 * (1 + math.cos(math.pi * progress)))  # Floor at 5% of peak
```

### Training Phases (Curriculum)

V5 uses a **two-phase training curriculum**:

**Phase 1: Surface Learning (epochs 1-10)**
- Train the intra-day encoder to learn options surface structure
- Freeze the temporal encoder (or use reduced learning rate: 0.1x)
- Loss: magnitude + direction + quantile + return (no contrastive yet)
- Purpose: Let the per-day encoder learn good surface representations before adding temporal complexity

**Phase 2: Temporal Integration (epochs 11-50)**
- Unfreeze the temporal encoder (full learning rate)
- Add contrastive loss and confidence loss
- Gradually increase contrastive weight from 0 → 0.15 over 5 epochs
- Purpose: Learn temporal patterns on top of stable surface representations

This prevents the temporal encoder from dominating early training when the surface representations are still noisy.

### Batch Construction

```python
batch_size = 256  # Reduced from V4's 512 due to 5x more data per sample

# Each sample: 5 days × 128 tokens × 20 features = 12,800 floats
# Batch: 256 × 12,800 = 3.3M floats ≈ 13 MB per batch (FP32) ≈ 6.5 MB (FP16)
# Plus attention matrices: 256 × 5 × (128+1)^2 × 8 heads ≈ manageable on RTX 6000 (24GB)
```

### Data Augmentation

New for V5 — augmentations that the options domain supports:

1. **Lookback jitter**: Randomly shift the 5-day window by 0-2 days (e.g., use t-6 to t-2 instead of t-4 to t). Requires forward horizon adjustment.

2. **Contract dropout**: Randomly drop 10-20% of option contracts per day. Forces the model to be robust to sparse data.

3. **Day dropout**: Randomly mask 1 of the 5 lookback days (set day_mask=False). The temporal encoder must handle variable-length history.

4. **Noise injection**: Add small Gaussian noise (σ=0.01) to continuous features. Prevents overfitting to exact values.

---

## Model Parameters & Compute Estimate

### Parameter Count

```
IntraDayEncoder (shared, applied 5x):
  token_proj:        20 × 256 = 5,120
  pos_enc:           3 × 256 = 768 (+ scale)
  type_embed:        5 × 256 = 1,280
  cls_token:         256
  4 transformer blocks:
    each: 256×256×3 (QKV) + 256×256 (out) + 256×1024 + 1024×256 + norms
    ≈ 790K per block × 4 = 3,160K
  Total IntraDayEncoder: ~3.17M

TemporalEncoder:
  day_proj:          256 × 256 = 65,536
  temporal_pe:       5 × 256 = 1,280
  diff_proj:         256 × 256 = 65,536
  cls_token:         256
  2 transformer blocks: 790K × 2 = 1,580K
  Total TemporalEncoder: ~1.71M

BackboneProjection:
  256 → 256 → 128 + norms: ~100K

Prediction Heads:
  5 heads × (128 → 64 → 1-5): ~50K

TOTAL: ~5.03M parameters
```

Comparable to V4's 4.77M. The parameter budget shifted from 6 deep layers on one day to 4 layers × 5 days + 2 temporal layers.

### Memory Estimate (RTX 6000, 24GB)

```
Batch of 256 samples:
  Input tensors: 256 × 5 × 128 × 20 × 4 bytes = 131 MB (FP32), 66 MB (FP16)

IntraDayEncoder forward (per day):
  Attention: 256 × (129)² × 8 × 2 bytes ≈ 68 MB per day × 5 = 340 MB
  Activations for backprop: ~200 MB per day × 5 = 1,000 MB

TemporalEncoder forward:
  Attention: 256 × 6² × 8 × 2 bytes = negligible

Total peak: ~2.5 GB forward + ~2.5 GB gradients + 1 GB optimizer = ~6 GB
Plus model weights: ~20 MB
Plus ChunkLoader cache: ~2 GB

Total: ~8.5 GB — fits comfortably in RTX 6000's 24 GB
```

With FP16 (AMP), peak memory drops to ~5 GB, leaving room for larger batches if needed.

### Throughput Estimate

```
Per sample: IntraDayEncoder × 5 + TemporalEncoder × 1
  ≈ 5 × (4-layer transformer on 128 tokens) + 2-layer on 5 tokens
  ≈ 5 × 0.8ms + 0.1ms = 4.1ms per sample (FP16, compiled)

Batch of 256: 4.1ms × 256 / GPU_efficiency ≈ 1.5s per batch

5.8M train samples / 256 = 22,656 batches/epoch
22,656 × 1.5s = 34,000s ≈ 9.4 hours per epoch

With torch.compile + flash attention: ~6-7 hours per epoch
```

This is slower than V4 (~1.5 hours/epoch) because of 5x more data per sample. But V5 extracts far more signal per epoch, so it should converge in fewer epochs. Estimated total: 15-25 epochs × 7 hours = 4-7 days.

---

## V5 vs All Previous Versions

| Aspect | V1/V2 | V3 | V4 | **V5** |
|--------|-------|----|----|--------|
| Input | 5d × 276 features | 5d × 210 features + price | 1d × 256 tokens × 12 | **5d × 128 tokens × 20** |
| Temporal | Conv1d + diffs | Conv1d + diffs | None (1 price token) | **Transformer + diffs** |
| Contract granularity | Fixed 22-strike grid | Fixed grid | Individual contracts | **Individual contracts** |
| Positional encoding | None | None | Moneyness + DTE | **Moneyness + DTE + IV rank** |
| Options surface | Flattened to 276-dim | Flattened, dead cols removed | Full surface, attention | **Full surface, attention, per-day** |
| Temporal awareness | 5-day causal conv | 5-day causal conv | 6 scalars in price token | **5-day cross-attention + diffs** |
| Embedding dim | 32 | 32 | 64 | **128** |
| Parameters | 49K-151K | 136K | 4.77M | **~5M** |
| Quantiles | 3 (75, 90, 95) | 3 | 3 | **5 (50, 75, 90, 95, 99)** |
| Contrastive loss | No | No (var reg only) | No | **Yes** |
| Confidence head | No | No | No | **Yes** |
| Data augmentation | No | No | No | **Yes** (contract dropout, day dropout, noise) |
| Training curriculum | No | No | No | **Yes** (surface → temporal phases) |
| Asymmetric return loss | No | No | No | **Yes** (3x weight on big moves) |

---

## Expected Performance

### Conservative Estimate

Based on the enhancement impact analysis from `model_v4_improvements.md`, applied cumulatively:

```
V4 baseline:           1.84% per 10-day period (epoch 5)

+ Multi-day temporal:  +0.5% (the biggest single improvement — momentum/trend signals)
+ Expanded tokens:     +0.2% (more features per contract, surface summary)
+ Contrastive loss:    +0.2% (better embedding discrimination)
+ Asymmetric return:   +0.1% (better tail prediction)
+ 99th quantile:       +0.1% (extreme winner identification)
+ Confidence gating:   +0.1% (reduce noise from uncertain predictions)
+ Data augmentation:   +0.1% (better generalization)
+ Curriculum training: +0.1% (stable optimization path)

V5 estimate:           ~3.3% per 10-day period
                       → ~125% annualized (compounded)
```

### Optimistic Estimate

If the temporal encoder captures options momentum patterns as effectively as the foresight analysis suggests:

```
V5 potential:          ~4-5% per 10-day period
                       → ~170-240% annualized
```

This would bring the backbone signal within range of the portfolio model achieving foresight-like stock selection quality.

### What Would Close the Remaining Gap to Foresight

Even V5 won't reach foresight performance because:
1. **Foresight uses 2 stocks** — extreme concentration. V5's metrics are on top 10%.
2. **Foresight has zero noise** — knows the exact return. V5 must estimate from noisy signals.
3. **Foresight rebalances optimally** — only trades when improvement > 2%. V5's portfolio model must learn this.

The remaining gap (V5 ~3-5% vs foresight ~3.4% per period) is primarily in the **portfolio model's job** — cross-sectional ranking and position sizing. V5's backbone gets the per-stock signal close to what's theoretically achievable; the portfolio model must learn to concentrate on the very best opportunities.

---

## Implementation Roadmap

### Phase 1: Data Preparation (1-2 weeks)

1. Create `firstrate_learning_v5/prepare_tokens.py`:
   - 5-day sliding window over stock-dates
   - 20-dim token construction (12 from V4 + 8 new features)
   - Surface summary token computation per day
   - Price context token with all 20 dimensions
   - 128 max tokens per day, priority selection
   - Multi-day chunk format: (n, 5, 128, 20) + masks
   - All V4's optimizations: parallel workers, zstd, status.json, ProgressTracker

2. Validate: unit test on 1-2 quarters, verify chunk correctness

### Phase 2: Model Implementation (1 week)

1. Create `firstrate_learning_v5/model.py`:
   - ContinuousPositionalEncoding3D
   - IntraDayEncoder (4-layer shared transformer)
   - TemporalEncoder (2-layer cross-day transformer)
   - BackboneProjection (256 → 128)
   - Prediction heads (5 outputs including confidence)
   - V5Loss with contrastive and calibration terms

2. Create `firstrate_learning_v5/data_loader.py`:
   - ChunkLoader adapted for multi-day samples
   - InterleavedDataset for training (regime mixing)
   - SequentialDataset for eval

### Phase 3: Training (1-2 weeks)

1. Unit test → smoke test → full training progression
2. Phase 1 curriculum: 10 epochs surface learning
3. Phase 2 curriculum: 15-40 epochs temporal integration
4. Evaluate against V3/V4 baselines
5. Iterate on loss weights, learning rate, augmentation

### Phase 4: Portfolio Integration (after backbone training)

1. Extract 128-dim embeddings + predictions for all stock-days
2. Feed to portfolio model for cross-sectional ranking
3. Backtest full system (backbone + portfolio)
