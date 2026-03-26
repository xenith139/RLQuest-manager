# RLQuest Project Goals

## Core Thesis

A perfect foresight model proves profitable trades exist in every market regime (170% return in 50 days, 9.16 Sharpe). Modern transformer architectures trained on large raw data can learn subtle patterns — slight volume changes, implied volatility shifts, put-call ratio anomalies — that predict large moves. The goal is to close the gap between current model performance and the foresight upper bound.

## Architecture

- **Backbone (FirstRate Learning)** — per-stock feature extractor. Processes raw options chain + price data as token sequences. Identifies patterns for individual stocks only, not across stocks. Must be pre-trained and trained on sequence-based data following the training recipe in `direction.md`.

- **V4 Transformer** — current backbone iteration. 6-layer, 8-head attention model (d_model=256) that takes raw options contracts as variable-length token sequences (max 256). No hand-crafted features — lets the model learn directly from moneyness, DTE, bid/ask, volume, open interest. Predicts: big move probability, direction, return quantiles, return magnitude.

- **Portfolio Model (downstream)** — uses the backbone as a frozen feature extractor to learn trading signals across all stocks. Designs buy/sell signals for variable-size universes (e.g., train on 2K stocks, inference on 5K). Handles cross-sectional ranking that the backbone cannot.

## Immediate Goals

- Improve V4 backbone performance to match/exceed V3 captured return (0.0147) with better return correlation
- Complete V4 token preparation and full training cycle
- Iterate on training recipe: loss weights, learning rate schedule, data augmentation
- Evaluate against V1/V2/V3 on standard metrics (P@5%, captured return, rank correlation, direction accuracy)
- Progress toward foresight benchmark quality of stock selection
