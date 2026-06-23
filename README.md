# robot_retargeter

English | [中文](README_zh.md)

A toolkit for retargeting human motion (SMPL-X) or source-robot motion onto
target humanoid robots, with support for side-by-side multi-robot visualization.

## Overview

The project provides an end-to-end motion retargeting pipeline with three stages:

1. **Replay / keypoint extraction**: Extract skeletal keypoints from SMPL-X
   motion (`.npz`) or source-robot motion (`.csv`).
2. **Retargeting**: Map the keypoints onto a target robot model via inverse
   kinematics ([mink](https://github.com/kevinzakka/mink) + MuJoCo) to produce
   robot motion.
3. **Visualization**: Play back the retargeted result for one or more target
   robots side by side.

Directory layout:

| Directory | Contents |
|---|---|
| `asset/` | Robot models (URDF/MJCF/mesh), SMPL-X models, skeleton |
| `config/` | Retargeting configs for robots and skeleton (YAML) |
| `dataset/` | Input motion data (SMPL-X `.npz` / LAFAN1 `.csv`, etc.) |
| `output_data/` | Output keypoints and robot motion |
| `scripts/` | Python pipeline scripts |
| `bash/` | One-command pipeline scripts |

## Installation

### Clone the repository

Large robot mesh/texture files (`*.stl`, `*.obj`, `*.dae`, `*.png`, `*.mtl`) are
committed directly in this repository. You can clone with plain Git.

```bash
# Clone with regular Git
git clone https://github.com/ccrpRepo/robot_retargeter.git
cd robot_retargeter
```

### Python environment

Requires Python >= 3.10 (developed on Python 3.11). A virtual environment
(conda / venv) is recommended.

```bash
# 1) Create and activate an environment (conda as an example)
conda create -n robot_retargeter python=3.11 -y
conda activate robot_retargeter

# 2) Install dependencies
pip install -e .
# or
pip install -r requirements.txt
```

Key dependencies: `mujoco`, `mink`, `numpy`, `torch`, `scipy`, `PyYAML`,
`tqdm`, `glfw`, `smplx`, `trimesh` (exact version floors are listed in
`setup.py` / `requirements.txt`).

### SMPL-X models (downloaded separately)

The SMPL-X model files are **not** included in this repository (they are
subject to their own license). To use the SMPL-X pipeline
(`retarget_from_smplx.sh`), download them yourself:

1. Register and download from the official site: [smplx model download](https://download.is.tue.mpg.de/download.php?domain=smplx&sfile=smplx_lockedhead_20230207.zip)
   (the "SMPL-X" models, `.npz` format).
2. Place the files under `asset/smplx/` so the layout looks like:

   ```
   asset/smplx/
     SMPLX_NEUTRAL.npz
     SMPLX_MALE.npz
     SMPLX_FEMALE.npz
   ```

> By downloading the models you agree to the SMPL-X license terms. This
> directory is git-ignored and will not be committed.

> Note: Large robot mesh/texture files (`*.stl`, `*.obj`, `*.dae`, `*.png`,
> `*.mtl`) are stored directly in this repository, so cloning may take longer.

### Motion datasets (AMASS / ACCAD)

The files in `dataset/ACCAD/` are a small set of open-source sample motions from
the **ACCAD** subset of the AMASS dataset (SMPL-X `.npz` format), provided only
to quickly try out the pipeline. If you need more motion data, download it
yourself from the AMASS website:

- AMASS website (register to download): https://amass.is.tue.mpg.de/
- ACCAD subset download page: https://amass.is.tue.mpg.de/download.php
  (select **ACCAD** in the list and download the `SMPL-X N` (neutral) format)

After downloading, unzip and place the `.npz` motion files under any directory
in `dataset/` (e.g. `dataset/ACCAD/`), then point `SMPL_MOTION_FILE` to the
desired file.

> By downloading you agree to the AMASS / ACCAD license terms; the data may only
> be used as permitted by those licenses.

## Usage

The `bash/` directory provides two one-command pipeline scripts that
automatically run all three stages (replay -> retarget -> visualize). By default
the scripts use the `python` from the currently active environment; you can
override it via `PYTHON_BIN`.

### 1) Retarget from SMPL-X motion

```bash
./bash/retarget_from_smplx.sh
```

Customize via environment variables:

| Variable | Default | Description |
|---|---|---|
| `SMPL_MOTION_FILE` | `dataset/ACCAD/Form_1_stageii.npz` | Input SMPL-X motion file |
| `VIS_ROBOTS` | `g1 h2 t800 r1` | Target robot list (space-separated, multiple allowed) |
| `KEYPOINTS_NAME` | derived from motion file name | Keypoints / output motion name |
| `SOURCE_FPS` | `120` | Source motion frame rate |
| `RENDER_FPS` | `30` | Visualization render frame rate |
| `PYTHON_BIN` | auto-detected | Python interpreter to use |

Example (custom robots and motion file):

```bash
VIS_ROBOTS="g1 hightorque_hi h2 t800" \
SMPL_MOTION_FILE="dataset/ACCAD/Form_1_stageii.npz" \
./bash/retarget_from_smplx.sh
```

### 2) Retarget from source-robot motion

Demo preview (click to open MP4):

[![retarget preview](retarget_from_g1_dance1_subject2_preview.gif)](retarget_from_g1_dance1_subject2.mp4)

Direct video link:
[retarget_from_g1_dance1_subject2.mp4](retarget_from_g1_dance1_subject2.mp4)

```bash
./bash/retarget_from_robot.sh
```

| Variable | Default | Description |
|---|---|---|
| `ROBOT_MOTION_FILE` | `dataset/lafan1_g1/dance1_subject2.csv` | Source robot motion file |
| `ORIGIN_ROBOT` | `g1` | Source robot name (provides the skeleton config) |
| `VIS_ROBOTS` | `h2 r1` | Target robot list (space-separated, multiple allowed) |
| `SOURCE_FPS` | `30` | Source motion frame rate |
| `RENDER_FPS` | `30` | Visualization render frame rate |
| `PYTHON_BIN` | auto-detected | Python interpreter to use |

Example:

```bash
VIS_ROBOTS="jaka_pi r1 booster_t1 hightorque_hi" \
ROBOT_MOTION_FILE="dataset/lafan1_g1/dance2_subject4.csv" \
ORIGIN_ROBOT="g1" \
SOURCE_FPS=30 \
RENDER_FPS=30 \
./bash/retarget_from_robot.sh
```

After a run finishes, the retargeted robot motion is saved under
`output_data/robot_motion/`.

#### Using the Bones Seed G1 Dataset

[Bones Studio](https://huggingface.co/datasets/bones-studio/seed) has released a
**Seed** motion dataset recorded on the G1 robot. Its raw format (root pose as
Euler angles in centimetres, joint values in degrees) differs from the LAFAN1
format required by this project (root pose as a quaternion in metres, joint
values in radians). Use `scripts/convert_bones_to_lafan1.py` to convert before
retargeting.

**Download the data**

```bash
# Download the G1 Seed dataset from Hugging Face
wget https://huggingface.co/datasets/bones-studio/seed/blob/main/g1.tar.gz -O g1.tar.gz
tar -xzf g1.tar.gz -C dataset/bones_g1_origin/
```

**Format conversion**

`convert_bones_to_lafan1.py` converts the raw G1 CSV files to LAFAN1-compatible
format:

| Argument | Default | Description |
|---|---|---|
| `--input-root` | `dataset/bones_g1_origin` | Directory containing the raw CSV files (batch mode) |
| `--output-root` | `dataset/bones_g1` | Output directory for converted CSV files |
| `--input-csv` | _(none)_ | Convert a single CSV file only |
| `--root-scale` | `0.01` | Root translation scale factor (cm → m) |

```bash
# Batch-convert all CSV files under dataset/bones_g1_origin/
python scripts/convert_bones_to_lafan1.py

# Convert a single file only
python scripts/convert_bones_to_lafan1.py \
    --input-csv dataset/bones_g1_origin/grab_walk_ff_180_001__A550_M.csv

# Custom input / output directories
python scripts/convert_bones_to_lafan1.py \
    --input-root dataset/bones_g1_origin \
    --output-root dataset/bones_g1
```

Once conversion is complete, pass the generated CSV directly to the retargeting
pipeline:

```bash
VIS_ROBOTS="jaka_pi h2 t800 pnd_adam" \
ROBOT_MOTION_FILE="dataset/bones_g1/grab_walk_ff_180_001__A550_M.csv" \
ORIGIN_ROBOT="g1" \
SOURCE_FPS=120 \
RENDER_FPS=30 \
./bash/retarget_from_robot.sh
```
