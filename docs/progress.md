# Development Progress

Tracking key milestones and evaluation results for the spatial memory benchmark.

## 2026-02-16: Sentence-Transformer Embeddings

**Change**: Replaced hash-based text embeddings (32-dim random vectors) with
`sentence-transformers/all-MiniLM-L6-v2` (384-dim, L2-normalized). This makes
slot merging and query-time retrieval semantically meaningful.

**Setup**:
- Dataset: TUM freiburg1\_xyz (273 RGB frames, stride 5 = 55 processed)
- Platform: Jetson Orin Nano Super (7.4 GB RAM, CUDA 12.6)
- COLMAP 3.13 (source-built, GPU SIFT) for sparse reconstruction
- VLM captions from Qwen2-VL-2B (cached, generated on separate machine)
- 11 queries: 8 locate, 1 count, 2 attribute

### Results

| Metric | Hash Embeddings | Sentence Embeddings | Delta |
|---|---|---|---|
| retrieval@1 | 0.250 | **0.375** | +50% |
| retrieval@3 | 0.500 | 0.500 | -- |
| retrieval@5 | 0.500 | 0.500 | -- |
| MRR | 0.375 | **0.417** | +11% |
| answer\_accuracy | 0.333 | 0.333 | -- |
| memory\_slots | 24 | 16 | -8 |
| 3D map points | 15,422 | 15,456 | -- |
| query latency (mean) | 0.4 ms | 16 ms\* | -- |

\*First query includes model loading (~1.2 s); subsequent queries ~16 ms.

### Per-Query Breakdown

| Query | Type | Result |
|---|---|---|
| Locate the mouse | locate | hit@1 |
| Locate the man | locate | hit@1 |
| Where is the white mouse? | locate | hit@1 |
| Locate the keyboard | locate | hit@3 |
| Locate the black monitor | locate | miss |
| Locate the books | locate | miss |
| Where are the books? | locate | miss |
| Where is the office desk? | locate | miss |
| Are there any books on the desk? | count | correct |
| What is the shape of the monitor? | attribute | incorrect |
| How many computer monitors are there? | attribute | incorrect |

### Key Observations

1. **retrieval@1 improved 50%** (0.250 to 0.375) — real embeddings enable
   semantic matching between queries and slot descriptions.
2. **Slot count decreased** from 24 to 16. With real embeddings, semantically
   similar frames merge more aggressively. The `embedding_threshold` parameter
   (cosine similarity for merging) was raised from 0.70 to 0.95 to prevent
   total slot collapse in this single-room scene.
3. **Remaining misses** are driven by ambiguous references ("office desk"
   matches many slots) and near-duplicate slots for similar objects.
4. **Latency overhead** from the sentence-transformer is negligible after
   model warm-up on Jetson.

## 2026-02-16: COLMAP GPU on Jetson

Built COLMAP 3.13 from source with CUDA support on the Jetson Orin Nano.
GPU-accelerated SIFT feature extraction and matching. Registered 273/273 frames
with 15k+ 3D points. See `docs/colmap_gpu_jetson.md` for build instructions
and runtime analysis.

## 2026-02-16: Pipeline Persistence

Added save/load support for both the 3D map state (`.npz`) and scene memory
(JSON). This enables:
- Skipping COLMAP re-runs when iterating on memory or retrieval parameters
- Loading pre-built memory for fast evaluation
- Caching intermediate results across pipeline stages
