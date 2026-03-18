```markdown
# bvh-to-smplh

Reliable utilities to convert BVH motion data into SMPL-H style NPZ files, postprocess them (grounding and stabilization), and export NPZ back to BVH for DCC/game-engine visualization.

This repository contains multiple pipeline variants. The sections below describe the currently available scripts and their real behavior.

## What This Repo Does

1. BVH -> NPZ conversion for training/inference pipelines.
2. Optional motion cleanup and stabilization (smoothing, grounding, foot lock).
3. NPZ -> BVH conversion for Blender/Unity/Unreal workflows.
4. MP4 visualization and debugging helpers.

## Project Layout

### motion/

- `bvh_loader.py`:
    Recursive BVH parser (hierarchy, channels, End Site blocks, frame matrix).
- `bvh_to_smplh.py`:
    BVH body-joint mapping to SMPL-body outputs (`global_orient`, `body_pose`, `transl`).
- `smpl_fk.py`:
    22-joint forward kinematics used by stabilization tools.
- `visualizer.py`:
    Renders motion to MP4 with a simple 3D skeleton + ground grid.
- `production_exporter.py`:
    One-off production conversion script with hardcoded defaults.

### scripts/

- `smplh_processor.py`:
    Main single-file BVH -> NPZ processor (52-joint SMPL-H style output).
- `batch_processor.py`:
    Batch BVH -> NPZ wrapper around `smplh_processor.py`, then calls renderer.
- `batch_npz_to_bvh.py`:
    Main batch NPZ -> BVH exporter using a hardcoded raw-skeleton definition.
- `debug_frame0.py`:
    Prints frame-0 channel values for key joints.
- `inspect_skeleton.py`:
    Prints hierarchy/offset tree from BVH header.

### scripts/utils/

- `npz_to_bvh.py`:
    Template-based NPZ -> BVH exporter (uses source BVH header/channel map).
- `ground_real.py`:
    Ground correction using minimum FK height.
- `ground_robust.py`:
    Ground correction using percentile-based floor detection (outlier resistant).
- `ground_physics.py`:
    Physics-inspired grounding using capsule-radius contact assumptions.
- `postprocess_foot_lock.py`:
    Translation-only deterministic foot-lock stabilization.
- `postprocess_smplh_stabilize.py`:
    CLI wrapper to apply foot-lock stabilization to NPZ files.

## Folder Structure

```text
bvh-to-smplh/
|-- .git/
|-- .gitignore
|-- LICENSE
|-- README.md
|-- requirements.txt
|-- motion/
|   |-- __init__.py
|   |-- bvh_loader.py
|   |-- bvh_to_smplh.py
|   |-- production_exporter.py
|   |-- smpl_fk.py
|   `-- visualizer.py
`-- scripts/
    |-- batch_npz_to_bvh.py
    |-- batch_processor.py
    |-- debug_frame0.py
    |-- inspect_skeleton.py
    |-- smplh_processor.py
    `-- utils/
        |-- ground_physics.py
        |-- ground_real.py
        |-- ground_robust.py
        |-- npz_to_bvh.py
        |-- postprocess_foot_lock.py
        `-- postprocess_smplh_stabilize.py
```

## Install

Use Python 3.10+ recommended.

```bash
pip install numpy scipy matplotlib tqdm
```

Optional (for `ground_physics.py`):

```bash
pip install pybullet
```

## Quick Start

### 1) Single BVH -> NPZ

```bash
python -m scripts.smplh_processor --input path/to/input.bvh --output path/to/output.npz
```

### 2) Batch BVH -> NPZ

Edit paths inside `scripts/batch_processor.py`, then run:

```bash
python -m scripts.batch_processor
```

### 3) Batch NPZ -> BVH

```bash
python -m scripts.batch_npz_to_bvh --input_dir path/to/npz_dir --output_dir path/to/bvh_dir
```

### 4) Render MP4 Preview

```bash
python -m motion.visualizer --npy path/to/motion.npz --bvh path/to/template_or_source.bvh --out path/to/preview.mp4
```

### 5) Optional Grounding / Stabilization

Robust floor grounding:

```bash
python -m scripts.utils.ground_robust --npy path/to/in.npz --bvh path/to/source.bvh --out path/to/grounded.npz
```

Foot-lock stabilization:

```bash
python -m scripts.utils.postprocess_smplh_stabilize path/to/in.npz
```

## Data Format Notes

Different scripts read/write slightly different NPZ schemas. Most common keys are:

- `poses`: either `(T, J, 3)` rotvec or flattened `(T, J*3)`.
- `trans` or `transl`: root translation `(T, 3)`.
- `joint_names`: joint name list used to map `poses` columns.
- Some utils also expect `global_orient`, `body_pose`, and `fps`.

When chaining scripts, verify expected keys and reshape requirements.

## Coordinate / Scale Behavior

- BVH source translations are frequently converted from centimeters to meters.
- Different scripts apply different root-fix conventions (`+90` or `-90` around X, and matrix basis transforms).
- Grounding scripts assume Z-up for floor alignment in their FK calculations.

Keep conversion and export scripts consistent within one workflow to avoid double rotation or axis mismatch.

## Known Caveats

1. `requirements.txt` is currently empty.
2. Several scripts contain hardcoded Windows paths intended for local production runs.
3. Multiple conversion variants exist; outputs differ slightly depending on chosen path.
4. `postprocess_smplh_stabilize.py` import path may need adjustment if run as-is in some environments.

## Recommended Workflow

For most users:

1. Run `scripts/smplh_processor.py` (or `scripts/batch_processor.py`) to create NPZ.
2. Apply `scripts/utils/ground_robust.py` if character floats/sinks.
3. Apply `scripts/utils/postprocess_smplh_stabilize.py` to reduce visible foot sliding.
4. Export with `scripts/batch_npz_to_bvh.py` for DCC playback.

## License

This project is licensed under the MIT License. See the LICENSE file for details.

## Acknowledgment

This project is based on work from:
https://github.com/smaameri/mocap-cleaning

I contributed to the original pipeline and have reorganized, improved, and extended it here for clarity, usability, and independent use.