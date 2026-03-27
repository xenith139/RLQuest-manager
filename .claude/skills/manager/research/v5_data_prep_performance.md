# V5 Data Preparation — Performance Analysis

## What We Know

**V4 baseline** (optimized, 16 workers, verified):
- Late quarters (2024+, ~6000 files each): 85-103 files/sec
- Full 64-quarter run: ~47 minutes
- Per-sample: parse CSV, compute 12-dim tokens, sort by moneyness, pad to 256, save zstd chunk

**V5 additional work per sample:**
- 5-day sliding window (not just 1 day) — must load 5 consecutive days of options data
- 8 new token dimensions: IV percentile, volume percentile, OI percentile, IV/realized vol ratio, moneyness distance from peak IV, expiry bucket, intrinsic value ratio, log bid-ask ratio
- Surface summary token per day (14-dim aggregate: ATM IV, skew, volume concentration, etc.)
- Price context token expanded from 6 to 20 dimensions
- Chunk format: (n, 5, 128, 20) vs V4's (n, 256, 12) — 5x more tokens per sample

**V5 unit test result** (3 tiny early quarters):
- 880 train + 2,473 val + 2,572 test = 5,925 samples in 34 seconds
- ~174 samples/sec — but early quarters are tiny (~100-200 files), not representative

## What We Don't Know

1. **Throughput on large quarters** — 2024+ quarters have ~6,000 files each. The 5-day window means each file's data must be cross-referenced with 4 prior days. Is this O(n) or O(n²)?

2. **IV percentile computation** — ranking all contracts by IV per stock-day requires sorting. Is this vectorized (numpy argsort) or Python loops? If looped across thousands of contracts × thousands of dates → potentially very slow.

3. **Memory under load** — 5 days × 128 tokens × 20 dims × float32 = 51.2 KB per sample. 50,000 samples per chunk = 2.5 GB per chunk buffer. Is this hitting memory limits with 16 workers?

4. **Disk I/O pattern** — V4 reads one CSV file per sample. V5 needs data from 5 consecutive days, which may mean reading 5 files or doing lookback joins. Is this sequential or random access?

5. **Actual parallelization** — V5 uses ProcessPoolExecutor like V4, but the 5-day window creates dependencies between dates within a quarter. Can workers still process symbols independently?

## Estimated Performance

**Conservative estimate** (5x slowdown from V4):
- V4: 85-103 files/sec → V5: 17-21 files/sec
- Full run: 47 min × 5 = ~4 hours

**Optimistic estimate** (2-3x slowdown — 5-day window is cached efficiently):
- V5: 30-50 files/sec
- Full run: ~1.5-2.5 hours

**Pessimistic estimate** (10x slowdown — IV percentile not vectorized, memory pressure):
- V5: 8-10 files/sec
- Full run: ~8+ hours

## What Must Be Validated (Smoke Test)

Before launching full run, the smoke test MUST measure:

1. **Throughput on large quarters** (2024_q3, 2024_q4 with ~5,800 files each)
   - Acceptable: >30 files/sec
   - Unacceptable: <15 files/sec → profile and optimize

2. **CPU utilization** — all 16+ workers active?
   - `ps aux | grep prepare_tokens` should show multiple processes at 80%+ CPU

3. **Memory per worker** — stable or growing?
   - Each worker processing 5-day windows with large contract counts could accumulate memory

4. **Specific bottleneck identification** — if slow, WHERE is the time spent?
   - CSV parsing? (same as V4, should be fast)
   - 5-day window assembly? (new, could be slow if not cached)
   - IV/volume percentile computation? (new, sorting per stock-day)
   - Surface summary computation? (new, aggregate statistics)
   - Chunk compression/writing? (same as V4, should be fast)

## Optimization Opportunities (if smoke test shows slow performance)

| Bottleneck | Optimization | Expected Speedup |
|-----------|-------------|-----------------|
| IV percentile ranking | `np.argsort` vectorized, not Python sort | 5-10x on this step |
| 5-day window assembly | Pre-sort dates, cache price lookups, mmap prices | 2-3x |
| Surface summary | Vectorize ATM/skew computation with numpy fancy indexing | 2-3x |
| Memory pressure | Reduce chunk_size from 50K to 25K, explicit `del` | Prevents OOM |
| Disk I/O | Ensure price data is mmap'd (not loaded per worker) | 2x on I/O |

## Actual Performance (measured during full run)

| Quarter Size | Files | Time | Files/sec | Status |
|-------------|-------|------|-----------|--------|
| Small (2010) | ~100 | 2.6s | ~38 | Startup overhead dominates |
| Medium (2011-2012) | 3400-4000 | 13-27s | 140-250 | **Faster than V4 (85-103)** |
| Large (2024+) | ~6000 | TBD | TBD | Not yet reached |

**Key finding**: The smoke test reported ~12 files/sec which was misleading — tiny quarters (100 files) are dominated by worker startup overhead, not actual processing speed. On real-sized quarters (3000-4000 files), V5 runs at 140-250 files/sec, significantly faster than V4.

**Manager process gap**: The smoke test was not representative. It only ran 5 small quarters. Future VALIDATE checks must require the smoke test to include at least 1-2 large quarters (2024+) to get meaningful throughput numbers.

## Decision

The full run is performing well at 140-250 files/sec on medium quarters. No optimization needed at this scale. However, throughput on large quarters (2024+, ~6000 files) still needs verification — the 5-day window assembly may slow down as contract counts increase.
