# Long-Running Script Optimization Guide

Reference guide for the manager to evaluate whether a long-running script in RLQuest is properly optimized. Based on patterns from the project's production-grade prepare scripts (`prepare_data.py`, `prepare_chunks.py`, `prepare_chunks_pt.py`, `prepare_tokens.py`, `prepare_norm_*.py`).

## Optimization Checklist

When the manager detects dev is running a long-running task, evaluate the script against each category below. If the script fails multiple categories, instruct dev to stop the run and optimize before resuming.

---

### 1. Parallelization

**Required for any script processing large datasets.**

| Technique | When to Use | Reference Implementation |
|-----------|------------|--------------------------|
| `multiprocessing.Pool` / `ProcessPoolExecutor` | CPU-bound work (feature computation, data transforms) | `prepare_data.py`: `mp.Pool(processes=num_workers)` with `imap_unordered()` |
| `ThreadPoolExecutor` | I/O-bound work (file loading, network) | `prepare_chunks.py`: `LOAD_WORKERS = 2` threads for quarter loading |
| Dynamic chunk sizing | When distributing work across workers | `prepare_data.py`: `chunk_size = max(50, num_files // (num_workers * 4))` |
| Worker count from CLI | Allow tuning per machine | `--workers` flag, default `mp.cpu_count()` or 8 |

**Red flags:**
- Sequential processing of files/symbols/quarters when they're independent
- Single-threaded I/O on many small files
- No `--workers` CLI option

**Prompt to dev:**
> "This script processes [N items] sequentially. Can you add multiprocessing with ProcessPoolExecutor? Use `imap_unordered` for CPU-bound work or ThreadPoolExecutor for I/O-bound loading. Reference `prepare_data.py` pattern."

---

### 2. GPU Acceleration

**Check if any steps could benefit from GPU.**

| Candidate Steps | GPU Approach |
|----------------|--------------|
| Large matrix operations | `torch.cuda` tensors or `cupy` |
| Feature normalization at scale | Batch GPU normalize |
| Distance/similarity computations | GPU-accelerated scipy alternatives |

Note: The current RLQuest prepare scripts use CPU-only for data prep, deferring GPU to training. This is acceptable when I/O or serialization dominates, but large numerical transforms should be evaluated.

**Prompt to dev:**
> "Are there large matrix operations or numerical transforms in this script that could run on GPU? Check if moving [specific step] to torch.cuda or cupy would reduce runtime."

---

### 3. Caching & Serialization

**Every intermediate result should be cached to disk.**

| Technique | When to Use | Reference |
|-----------|------------|-----------|
| Memory-mapped numpy (`mmap_mode='r'`) | Large arrays read multiple times | `prepare_data.py`: `np.load(path, mmap_mode='r')` for 8766x4030 price array |
| Compressed numpy (`.npz_compressed`) | Final output arrays, moderate size | `prepare_price_features.py`: `np.savez_compressed(path, features=..., ...)` |
| Zstandard compression (`.zst`) | Large serialized objects, need fast decompression | `prepare_chunks.py`: `zstd.ZstdCompressor(level=1, threads=8)` — 3-5x faster than gzip |
| PyTorch save + zstd | Tensor data with mixed types | `prepare_chunks.py`: `torch.save(chunk, buf)` then zstd compress |

**Red flags:**
- Using `gzip` or `pickle` without compression for large files (zstd is faster)
- Recomputing values that could be cached to `.npy`/`.npz`
- Loading entire large files into memory when `mmap_mode='r'` would work
- No intermediate file caching between pipeline stages

**Prompt to dev:**
> "This script recomputes [X] every run. Can you cache it as a compressed numpy file (`.npz_compressed`) or use zstd for torch tensors? Also, switch any `np.load()` of large arrays to `mmap_mode='r'`."

---

### 4. Resumability & Checkpointing

**Any script that runs >5 minutes MUST be resumable.**

| Pattern | Implementation | Reference |
|---------|---------------|-----------|
| Status JSON file | Track completed splits/quarters/phases | `prepare_chunks.py`: `STATUS_PATH = CHUNKS_DIR / "prepare_status.json"` |
| Skip-if-complete check | Check meta file before processing a unit | `prepare_data.py`: `if is_quarter_complete(quarter_dir): return cached` |
| Per-batch progress saving | Save after each batch/symbol group | `prepare_chunks_pt.py`: `SYMBOLS_PER_BATCH = 50`, saves progress after each |
| `--reset` flag | Allow forced fresh start | CLI flag to clear status and reprocess |

**Red flags:**
- No status/checkpoint file — crash = restart from zero
- Processing all items even when some are already done
- No `--reset` or `--force` flag for fresh runs

**Prompt to dev:**
> "This script has no checkpointing. If it crashes at 80%, everything is lost. Add a status JSON that tracks completed units and skip them on restart. Reference `prepare_chunks.py` pattern with `prepare_status.json`."

---

### 5. Memory Optimization

**Critical for large datasets.**

| Technique | When to Use | Reference |
|-----------|------------|-----------|
| `dtype=np.float32` | All feature arrays (50% savings vs float64) | All prepare scripts |
| `dtype=np.int16` | Age/index arrays with small ranges | `prepare_chunks.py`: `ages_out` |
| `dtype=bool` | Mask arrays | `prepare_chunks_pt.py`: `mask_out` |
| Chunked processing | When full dataset exceeds RAM | `prepare_chunks.py`: `CHUNK_SIZE = 50_000` |
| Explicit `del` + gc | After large intermediate arrays | `prepare_tokens.py`: `del date_contracts, sym_samples` |
| Streaming/generators | Per-file or per-line processing | `prepare_tokens.py`: line-by-line file reading |
| Welford's algorithm | Computing mean/std without materializing all data | `prepare_chunks.py`, `prepare_norm_*.py` |

**Red flags:**
- Using `float64` when `float32` suffices
- Loading entire dataset into a single array
- No chunking strategy for large outputs
- Computing mean/std by loading all data first (use Welford instead)

**Prompt to dev:**
> "Check memory usage. Are arrays using float32? Is data loaded in chunks or all at once? For normalization stats, use Welford's online algorithm to avoid materializing the full dataset. Reference `prepare_chunks.py` Welford implementation."

---

### 6. Vectorized Computation

**No Python loops over large arrays.**

| Pattern | Reference |
|---------|-----------|
| Vectorized sliding windows via fancy indexing | `prepare_chunks.py`: `row_idx = pos[:, None] - offsets[None, :]` |
| `np.lexsort` for multi-key sorting | `prepare_chunks.py`: `np.lexsort((sym_ids, date_ints))` |
| Broadcast operations for feature engineering | `prepare_price_features.py`: vectorized returns across all symbols |
| `np.where` / `np.clip` for conditional logic | `prepare_norm_fmp.py`: `std = np.where(std < 1e-8, 1.0, std)` |

**Red flags:**
- `for` loops iterating over rows/samples in numpy arrays
- Per-element conditional logic instead of `np.where`
- Python-level sorting instead of `np.lexsort`/`np.argsort`

**Prompt to dev:**
> "This script uses Python loops over [N] samples. Convert to vectorized numpy operations. Reference `prepare_chunks.py` sliding window pattern using fancy indexing."

---

### 7. Progress Monitoring

**All long-running scripts must report progress.**

| Feature | Reference |
|---------|-----------|
| Periodic logging with rate & ETA | `prepare_data.py`: `ProgressTracker` class with rate/ETA |
| tqdm progress bars | `prepare_chunks_pt.py`: `tqdm(quarters, desc="...")` |
| System resource reporting | `prepare_data.py`: CPU cores, total RAM at startup |
| Per-unit timing | All scripts: log elapsed time per quarter/batch |

**Red flags:**
- No progress output — impossible to estimate completion
- No rate/ETA calculation
- No system resource logging at startup

---

## Manager Decision: Wait or Stop & Optimize?

After evaluating the script against the checklist above:

### WAIT (let it run) if:
- Script passes all categories OR is in `verified_scripts.json` with matching runtime behavior
- Script is >90% complete (not worth restarting)
- Only minor optimizations possible (< 20% improvement estimate)

### STOP & OPTIMIZE if:
- Script fails 3+ categories from the checklist
- No parallelization on a multi-hour task
- No checkpointing (crash = full restart)
- Using float64, gzip, Python loops on large data
- Estimated optimization would save >50% runtime

### Prompt template for stopping:
> "The script [name] has been running for [time] and appears to lack [missing optimizations]. I recommend stopping the current run and optimizing: [specific recommendations]. After optimization, the script can resume from the last checkpoint (if checkpointing exists) or restart faster. Reference the patterns in `prepare_chunks.py` and `prepare_data.py` for production-grade implementations."

---

## Runtime Verification Commands

Use these to validate script performance while running:

```bash
# CPU utilization — all cores should be active if parallelized
htop -p $(pgrep -f "script_name")

# Memory usage
ps aux --sort=-%mem | head -5

# GPU utilization (if applicable)
nvidia-smi

# Disk I/O
iostat -x 1 3

# Process-specific memory
cat /proc/$(pgrep -f "script_name")/status | grep -i vmrss
```
