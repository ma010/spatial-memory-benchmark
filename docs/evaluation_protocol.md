# Evaluation Protocol

1. **Sampling**: select object-centric prompts per clip.
2. **Memory snapshot**: freeze the memory after processing available frames.
3. **Retrieval**: parse queries and rank candidate slots.
4. **Metrics**:
   - Retrieval@k (k in {1, 3, 5})
   - MRR
   - Spatial IoU between predicted and GT 3D extents (if provided)
   - Temporal continuity (ID switches per minute)
   - Memory footprint (serialized size)
5. **Reporting**: average per scene, then across the dataset.

This shell documents the protocol only; implementation details live outside
this repository.

Query types may include locate, attribute, and count prompts. Exact query
formatting will be finalized with the dataset release.
