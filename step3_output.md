## Step 3: Design Review — 2026-03-27 21:50 UTC
### Mode: NO CHANGE

No source files changed since last cycle (21:15 UTC). No new training runs launched. Previous deep analysis remains fully valid.

**Previous analysis summary (still current):**
- Model: 679K params, d_model=128, 2+1 layers, backbone=64, dropout=0.25
- Params/sample ratio: 0.117 -- MEDIUM overfit risk (5x V3's 0.023, 7x less than V4's 0.82)
- Token format: 20/20 contract dims populated, 14/20 summary, 19/20 price -- known zero-dim bugs are non-blocking
- Day mask: 90-94% with 5 valid days -- clean
- Loss: 7 components, total weight 3.15, 73% from core tasks -- acceptable
- Recipe: lr=3e-4, batch=512, dropout=0.25, cosine decay -- appropriate
- Data: 11.4M samples, time-based split (train 2010-2019, val 2020-2022, test 2023-2025)
- Epoch estimate: 6.3-8.4 hours
- Score formula dominated by classification (p_big * p_up * p_confidence) -- explains low unit test CR with high return_corr

### CRITICAL ISSUES: None blocking. Known issues (summary token zeros, price dim 15 bug, d_ff unused, feature scale mismatch) are all non-blocking and documented for future iteration.

### RECOMMENDATIONS: LAUNCH FULL TRAINING IMMEDIATELY. No further design review needed. GPU idle 2.5+ hours.

---

## Persistent Notes
(Cache for next cycle -- carry forward and update.)

- **File timestamps:** model.py=1774634991, config.py=1774637440, data_loader.py=1774637686, prepare_tokens.py=1774590272
- **Model:** 679,498 params, d_model=128, n_heads=4, intra_layers=2, temporal_layers=1, backbone_dim=64
- **Param breakdown:** intra_day=400,641 (59%), temporal=232,320 (34%), backbone=25,152 (3.7%), heads=~4.2K each (0.6% each)
- **Params/sample ratio:** 0.117 -- MEDIUM overfit risk
- **Token dims:** 20/20 contract dims populated; summary token dims 14-19 all zeros (6/20 wasted); price token dim 15 always zero (bug)
- **Sparse dims:** bid_ask_spread (dim 10) 97% zeros, log_ba_ratio (dim 19) 97% zeros
- **Feature scale mismatch:** moneyness [0,97], moneyness_peak_iv [-64,+53], mid_price_norm [0,50], intrinsic_value [0,96] vs typical [-10,10]
- **Day mask:** 89.8-93.6% with 5 valid days
- **Loss:** 7 components, weights=[1.0, 0.5, 0.4, 0.8, 0.2, 0.15, 0.1], sum=3.15, dominant=magnitude(1.0)+return(0.8)=57%
- **Recipe:** lr=3e-4, warmup=1000 steps, cosine decay to 0.1x, AdamW(beta2=0.98), weight_decay=0.01, grad_clip=1.0, dropout=0.25, AMP FP16
- **Data:** 11,411,714 samples, 264 chunks (train=138, val=63, test=63), split=50.8/24.3/24.9
- **Split:** train=2010q1-2019q4, val=2020q1-2022q4, test=2023q1-2025q4 (time-based, no leakage)
- **Epoch estimate:** 6.3-8.4 hours (~2.01s/batch measured, 11,324 batches/epoch)
- **Score formula:** p_big * p_up * p_confidence + quantile/return adjustments (classification-dominated)
- **Known issues:**
  1. Summary token dims 14-19 all zeros (6/20 wasted) -- non-blocking
  2. Price token dim 15 always zero (code skips pf[14] to pf[16]) -- non-blocking
  3. config.d_ff unused by model (hardcoded d_model*4) -- maintenance trap
  4. Feature scale mismatch on moneyness/mid_price_norm/intrinsic_value -- affects ~2% extreme tokens
  5. Epoch time 6-8 hours limits iteration speed
  6. IntraDayEncoder day-loop is sequential (potential 2-3x speedup if batched)
- **Training status:** Unit test complete (CR=0.0015, rank_corr=0.089, return_corr=0.088, 50 batches). Full training NOT STARTED.
- **Unit test signals:** return_corr=0.088 is 6x stronger than V3 final (0.0132) after only 50 batches -- suggests strong signal extraction potential
- **Baselines:** V3 CR=0.0147 (136K params, epoch 12), V1 CR=0.0192 (epoch 33), V4 CR=0.0031 (4.77M params, overfit)
- **Watch items for next cycle:**
  - Has full training been launched? Check for new run dirs in models/
  - If training running: GPU utilization, epoch progress, val loss trend, per-component loss magnitudes
  - Watch for overfitting after epoch 3-5 (val loss divergence)
  - If contrastive/confidence losses show erratic behavior, recommend disabling in next run
  - If overfitting: increase dropout to 0.30, weight_decay to 0.03
  - Monitor score vs return_corr relationship -- does CR improve as classification heads calibrate?
