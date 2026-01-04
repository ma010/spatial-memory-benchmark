# Data Card

## Summary
This benchmark targets short egocentric clips with associated camera poses and
lightweight scene geometry. This public shell does not ship any data.

## Availability
Planned. Access and licensing terms are TBD.

## Collection (Planned)
- Source: opt-in indoor captures.
- Sensors: RGB video with pose estimates (IMU or SLAM-derived).
- Pre-processing: frame extraction, pose smoothing, and manifest generation.

## Recommended Splits
- `demo`: tiny sanity-check subset (TBD).
- `benchmark`: evaluation subset (TBD).

## Ethical Considerations
Avoid capturing or releasing sensitive personal details. Data access will follow
a consent-based process and may include additional restrictions.
