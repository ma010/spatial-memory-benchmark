# Running COLMAP with GPU on NVIDIA Jetson

> Tested on Jetson Orin Nano Super (JetPack 6, CUDA 12.6, L4T R36.4).
> Applies broadly to Orin/Xavier/TX2 family devices.

## Problem

The apt-packaged COLMAP on Jetson segfaults when GPU SIFT is enabled.
The root cause is that COLMAP's SiftGPU library requires an OpenGL/GLX
context, but Jetson's L4T display stack uses EGL — there is no GLX.

## Fix: Build COLMAP from Source

Build with `CUDA_ENABLED=ON` and `GUI_ENABLED=OFF`. This forces SiftGPU
onto a pure CUDA path that does not require OpenGL.

```bash
git clone https://github.com/colmap/colmap.git && cd colmap
git checkout 3.13.0  # must be >= 3.11 for CUDA SIFT fix

mkdir build && cd build
cmake .. -GNinja \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_CUDA_COMPILER=/usr/local/cuda/bin/nvcc \
  -DCMAKE_CUDA_ARCHITECTURES="87" \
  -DCUDA_ENABLED=ON \
  -DGUI_ENABLED=OFF \
  -DTESTS_ENABLED=OFF \
  -DCGAL_ENABLED=OFF
ninja && sudo ninja install
```

Set `CMAKE_CUDA_ARCHITECTURES` to match your Jetson: Orin = `87`,
Xavier = `72`, TX2 = `62`.

## COLMAP 3.13 CLI Flag Renames

COLMAP 3.10+ renamed SIFT-related flags. Update any wrapper scripts:

| Old (3.7) | New (3.10+) |
|---|---|
| `--SiftExtraction.use_gpu` | `--FeatureExtraction.use_gpu` |
| `--SiftMatching.use_gpu` | `--FeatureMatching.use_gpu` |

## Multi-Model Selection

COLMAP's mapper can produce multiple sparse sub-models in `sparse/`
(e.g. `0/`, `1/`, `2/`). Model `0/` is not always the best — it may
be a degenerate initialization with very few frames. When selecting
which model to use, pick the one with the largest `images.bin` file,
which corresponds to the most registered cameras.

## Pipeline Stages: What Uses CUDA

| Stage | Compute | CUDA | Typical Runtime | CPU | RAM |
|---|---|---|---|---|---|
| Feature Extraction | SiftGPU (CUDA) | Yes | ~30 s / 273 frames | Low | ~400 MB |
| Feature Matching | CUDA kernels | Yes | ~15 s (sequential, overlap=10) | Low | ~400 MB |
| Mapper (SfM) | Ceres BA, PnP | No | ~50 min / 273 frames | ~380% (4 cores) | ~490 MB |
| Dense Reconstruction | CUDA | Yes | N/A (not used here) | — | — |

The mapper is CPU-bound and dominates total runtime (~98%). GPU
acceleration mainly reduces extraction and matching time, which matters
more as frame count grows (matching is quadratic in the exhaustive
case).

## References

- [COLMAP FAQ](https://colmap.github.io/faq.html)
- [COLMAP 3.11 Release Notes](https://github.com/colmap/colmap/releases/tag/3.11.0) — CUDA SIFT fix
- [Issue #2295](https://github.com/colmap/colmap/issues/2295) — GPU not used on aarch64
- [PR #2151](https://github.com/colmap/colmap/pull/2151) — SiftGPU with GUI_ENABLED=OFF
