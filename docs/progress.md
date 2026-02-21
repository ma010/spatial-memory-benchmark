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

## 2026-02-21: Geometry Ablation and Cross-Platform Validation

Two controlled experiments to isolate the effect of 3D geometry on retrieval,
and to verify results are reproducible across platforms.

### Experiment 1: Does 3D geometry help retrieval?

Same scene and config, two conditions: skip-mapping (embeddings only) vs full
COLMAP reconstruction (Mac, COLMAP 3.12.6 CPU, 15,622 sparse points).

| Metric | No geometry | With geometry | Delta |
|---|---|---|---|
| retrieval@1 | 0.375 | 0.375 | 0 |
| retrieval@3 | 0.500 | 0.500 | 0 |
| retrieval@5 | **0.625** | 0.500 | −0.125 |
| MRR | **0.469** | 0.417 | −0.052 |
| memory slots | 24 | 16 | −8 |
| 3D map points | — | 15,622 | — |

**Finding: 3D geometry does not improve retrieval on this scene.** Spatial
proximity causes more aggressive slot merging (24 → 16 slots), which reduces
candidate diversity and hurts R@5 and MRR. The spatial reranking benefits
(depth scores, density radius) do not compensate for the loss of slot
granularity.

### Experiment 2: Mac vs Jetson (same conditions)

Both platforms run the full pipeline with geometry and sentence-transformer
embeddings. Mac uses COLMAP 3.12.6 (CPU, Homebrew); Jetson uses COLMAP 3.13
(source-built, GPU SIFT).

| Metric | Mac (COLMAP 3.12 CPU) | Jetson (COLMAP 3.13 GPU) |
|---|---|---|
| retrieval@1 | 0.375 | 0.375 |
| retrieval@3 | 0.500 | 0.500 |
| retrieval@5 | 0.500 | 0.500 |
| MRR | 0.417 | 0.417 |
| memory slots | 16 | 16 |
| 3D map points | 15,622 | 15,456 |

**Finding: Results are platform-independent.** Identical retrieval metrics
despite different COLMAP versions, SIFT backends (CPU vs GPU), and slightly
different point counts. The retrieval ceiling on this small desk scene is
determined by slot granularity and query difficulty, not hardware.

### Summary across all runs

| Run | Embedding | Geometry | R@1 | R@5 | MRR | Slots |
|---|---|---|---|---|---|---|
| Jetson (sentence-transformer, Feb 16) | all-MiniLM-L6-v2 | 15,456 pts | 0.375 | 0.500 | 0.417 | 16 |
| Mac (no geometry, Feb 20) | all-MiniLM-L6-v2 | none | 0.375 | **0.625** | **0.469** | 24 |
| Mac (with geometry, Feb 21) | all-MiniLM-L6-v2 | 15,622 pts | 0.375 | 0.500 | 0.417 | 16 |

The no-geometry run (embeddings only, 24 slots) currently produces the best
retrieval quality. Improving geometry-based retrieval without sacrificing slot
granularity is an open problem.

## 2026-02-16: Pipeline Persistence

Added save/load support for both the 3D map state (`.npz`) and scene memory
(JSON). This enables:
- Skipping COLMAP re-runs when iterating on memory or retrieval parameters
- Loading pre-built memory for fast evaluation
- Caching intermediate results across pipeline stages
