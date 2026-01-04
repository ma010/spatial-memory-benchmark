# Memory Schema (High Level)

A spatial memory store combines geometry, object slots, and text metadata:

```text
SceneMemory
├── poses: List[Pose]
├── points: Nx3 array (or splats)
├── slots: List[ObjectSlot]
└── attributes: Dict[str, AttributeStats]
```

Each `ObjectSlot` includes:
- `slot_id`: stable integer identifier
- `position`: 3D centroid in world coordinates
- `embedding`: unit-length feature vector
- `attributes`: label -> score pairs
- `evidence`: references to supporting frames and captions

This schema is illustrative and may evolve alongside the benchmark release.
Fields and representations are intentionally high level and may vary by
submission.
