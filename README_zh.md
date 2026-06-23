# robot_retargeter

中文 | [English](README.md)

将人体动作（SMPL-X）或源机器人动作重定向到目标人形机器人，并支持多机器人并排可视化的工具集。

## 简介

本项目提供一条完整的动作重定向流水线，主要包含三步：

1. **回放 / 提取关键点**：从 SMPL-X 动作（`.npz`）或源机器人动作（`.csv`）中提取骨骼关键点。
2. **重定向（Retarget）**：基于逆运动学（[mink](https://github.com/kevinzakka/mink) + MuJoCo）把关键点映射到目标机器人模型上，生成机器人动作。
3. **可视化**：将一个或多个目标机器人的重定向结果并排播放查看。

目录结构概览：

| 目录 | 内容 |
|---|---|
| `asset/` | 机器人模型（URDF/MJCF/mesh）、SMPL-X 模型、骨架 |
| `config/` | 机器人与骨架的重定向配置（YAML） |
| `dataset/` | 输入动作数据（SMPL-X `.npz` / LAFAN1 `.csv` 等） |
| `output_data/` | 输出的关键点与机器人动作 |
| `scripts/` | Python 流水线脚本 |
| `bash/` | 一键运行的流水线脚本 |

## 安装
### 克隆仓库

仓库中的较大机器人 mesh / 纹理文件（`*.stl`、`*.obj`、`*.dae`、`*.png`、`*.mtl`）直接随仓库提交，使用普通 Git 即可克隆。

```bash
# 使用普通 Git 克隆
git clone https://github.com/ccrpRepo/robot_retargeter.git
cd robot_retargeter
```

### Python 环境
需要 Python ≥ 3.10（开发环境为 Python 3.11）。建议使用虚拟环境（conda / venv）。

```bash
# 1) 创建并激活环境（以 conda 为例）
conda create -n robot_retargeter python=3.11 -y
conda activate robot_retargeter

# 2) 安装依赖
pip install -e .
# 或
pip install -r requirements.txt
```

主要依赖：`mujoco`、`mink`、`numpy`、`torch`、`scipy`、`PyYAML`、`tqdm`、`glfw`、`smplx`、`trimesh`（精确版本下限见 `setup.py` / `requirements.txt`）。

### SMPL-X 模型（需单独下载）

SMPL-X 模型文件**不包含**在本仓库中（受其自身许可协议约束）。若要使用 SMPL-X 流水线（`retarget_from_smplx.sh`），请自行下载：

1. 在官网注册并下载：[smplx model download](https://download.is.tue.mpg.de/download.php?domain=smplx&sfile=smplx_lockedhead_20230207.zip) （下载 "SMPL-X" 模型，`.npz` 格式）。
2. 将文件放到 `asset/smplx/` 目录下，结构如下：

   ```
   asset/smplx/
     SMPLX_NEUTRAL.npz
     SMPLX_MALE.npz
     SMPLX_FEMALE.npz
   ```

> 下载即表示你同意 SMPL-X 的许可协议。该目录已被 git 忽略，不会被提交。

> 注意：较大的机器人 mesh / 纹理文件（`*.stl`、`*.obj`、`*.dae`、`*.png`、`*.mtl`）已直接存储在仓库中，因此首次克隆耗时可能更长。

### 动作数据集（AMASS）

`dataset/ACCAD/` 中提供的是 AMASS 数据集里 **ACCAD** 子集的少量开源示例动作（SMPL-X `.npz` 格式），仅用于快速体验流水线。若需要更多动作数据，请自行从 AMASS 官网下载：

- AMASS 官网（注册后下载）：https://amass.is.tue.mpg.de/

下载后解压，将 `.npz` 动作文件放到 `dataset/` 下任意目录（例如 `dataset/ACCAD/`），再通过 `SMPL_MOTION_FILE` 指向对应文件即可。

> 下载即表示你同意 AMASS 的许可协议，相关数据仅可用于其许可允许的用途。

## 运行

`bash/` 目录提供了两个一键流水线脚本，会自动完成「关键映射点生成 → 重定向 → 可视化」三步。脚本默认使用当前激活环境中的 `python`，也可用 `PYTHON_BIN` 指定解释器。

### 1) 从 SMPL-X 动作重定向

```bash
./bash/retarget_from_smplx.sh
```

可通过环境变量自定义参数：

| 变量 | 默认值 | 说明 |
|---|---|---|
| `SMPL_MOTION_FILE` | `dataset/ACCAD/Form_1_stageii.npz` | 输入的 SMPL-X 动作文件 |
| `VIS_ROBOTS` | `g1 h2 t800 r1` | 目标机器人列表（空格分隔，支持多个） |
| `KEYPOINTS_NAME` | 由动作文件名自动推导 | 关键点 / 输出动作名称 |
| `SOURCE_FPS` | `120` | 源动作帧率 |
| `RENDER_FPS` | `30` | 可视化渲染帧率 |
| `PYTHON_BIN` | 自动检测 | 指定 Python 解释器 |

示例（自定义机器人与动作文件）：

```bash
VIS_ROBOTS="g1 jaka_pi h2 t800" \
SMPL_MOTION_FILE="dataset/ACCAD/Form_1_stageii.npz" \
./bash/retarget_from_smplx.sh
```

### 2) 从源机器人动作重定向

演示预览（点击 GIF 打开 MP4）：

[![retarget 预览](retarget_from_g1_dance1_subject2_preview.gif)](retarget_from_g1_dance1_subject2.mp4)

视频直链：
[retarget_from_g1_dance1_subject2.mp4](retarget_from_g1_dance1_subject2.mp4)

```bash
./bash/retarget_from_robot.sh
```

| 变量 | 默认值 | 说明 |
|---|---|---|
| `ROBOT_MOTION_FILE` | `dataset/lafan1_g1/dance1_subject2.csv` | 源机器人动作文件 |
| `ORIGIN_ROBOT` | `g1` | 源机器人名称（提供骨架配置） |
| `VIS_ROBOTS` | `h2 r1` | 目标机器人列表（空格分隔，支持多个） |
| `SOURCE_FPS` | `30` | 源动作帧率 |
| `RENDER_FPS` | `30` | 可视化渲染帧率 |
| `PYTHON_BIN` | 自动检测 | 指定 Python 解释器 |

示例：

```bash
VIS_ROBOTS="jaka_pi h2 t800 pnd_adam" \
ROBOT_MOTION_FILE="dataset/bones_g1/grab_walk_ff_180_001__A550_M.csv" \
ORIGIN_ROBOT="g1" \
SOURCE_FPS=120 \
RENDER_FPS=30 \
./bash/retarget_from_robot.sh
```

运行结束后，重定向得到的机器人动作会保存在 `output_data/robot_motion/` 下。

#### 使用 Bones Seed G1 数据集

[Bones Studio](https://huggingface.co/datasets/bones-studio/seed) 发布了以 G1 机器人为载体的 **Seed** 动作数据集，其原始格式（根节点使用欧拉角/厘米单位，关节值为角度）与本项目所需的 LAFAN1 格式（根节点使用四元数/米单位，关节值为弧度）不同，需要先用 `scripts/convert_bones_to_lafan1.py` 进行转换。

**数据下载**

```bash
# 从 Hugging Face 下载 G1 Seed 数据集
wget https://huggingface.co/datasets/bones-studio/seed/blob/main/g1.tar.gz -O g1.tar.gz
tar -xzf g1.tar.gz -C dataset/bones_g1_origin/
```

**格式转换**

`convert_bones_to_lafan1.py` 将原始 G1 CSV 转换为 LAFAN1 兼容格式：

| 参数 | 默认值 | 说明 |
|---|---|---|
| `--input-root` | `dataset/bones_g1_origin` | 原始 CSV 所在目录（批量转换） |
| `--output-root` | `dataset/bones_g1` | 转换后 CSV 输出目录 |
| `--input-csv` | 无 | 仅转换单个 CSV 文件 |
| `--root-scale` | `0.01` | 根节点位移缩放系数（cm→m） |

```bash
# 批量转换 dataset/bones_g1_origin/ 下所有 CSV 文件
python scripts/convert_bones_to_lafan1.py

# 仅转换单个文件
python scripts/convert_bones_to_lafan1.py \
    --input-csv dataset/bones_g1_origin/grab_walk_ff_180_001__A550_M.csv

# 自定义输入 / 输出目录
python scripts/convert_bones_to_lafan1.py \
    --input-root dataset/bones_g1_origin \
    --output-root dataset/bones_g1
```

转换完成后，即可将生成的 CSV 作为源动作传入重定向流水线：

```bash
VIS_ROBOTS="jaka_pi h2 t800 pnd_adam" \
ROBOT_MOTION_FILE="dataset/bones_g1/grab_walk_ff_180_001__A550_M.csv" \
ORIGIN_ROBOT="g1" \
SOURCE_FPS=120 \
RENDER_FPS=30 \
./bash/retarget_from_robot.sh
```
