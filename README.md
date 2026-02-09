# v-ARM

[![Unity](https://img.shields.io/badge/Unity-%23000000.svg?logo=unity&logoColor=white)](#) [![MLGame3D](https://img.shields.io/pypi/v/mlgame3d?label=MLGame3D
)](https://pypi.org/project/mlgame3d/)

v-ARM 是一個以 Unity ML-Agents 製作的四軸機械手臂操作遊戲／訓練環境。玩家或代理需要在有限時間內，控制手臂在扇形工作區中完成目標方塊的 Grasp、Place 任務。

## 下載

[![Windows](https://custom-icon-badges.demolab.com/badge/Windows-1.0.0-blue?logo=windows)](https://github.com/PAIA-PROS/v-ARM/releases/download/1.0.0/v-ARM-win32-1.0.0.zip)
[![macOS](https://img.shields.io/badge/macOS-1.0.0-red?logo=apple)](https://github.com/PAIA-PROS/v-ARM/releases/download/1.0.0/v-ARM-darwin-universal-1.0.0.zip)

## 遊戲敘述

- 目標方塊與目標點會在工作區（或指定的生成範圍）中隨機生成。
- 手臂需在工作區內完成指定關卡目標，並在時間限制內取得成功。
- 若目標或末端執行器落在工作區外、或低於地面高度，會判定為失敗。

## 如何遊玩

- 用 Unity 開啟專案並進入 Play Mode。
- 使用鍵盤控制手臂（Heuristic 模式）：
  - 關節 1：`Q` / `A`
  - 關節 2：`W` / `S`
  - 關節 3：`E` / `D`
  - 夾爪（關節 4）：`F`（關）/ `R`（開）
- 依關卡目標達成 Grasp 或 Place 的成功條件。

## 遊戲關卡

- Grasp
  - 若 `requireGripperClosedForGrasp` 為真，夾爪需閉合。
  - 方塊高度相對初始位置提升 `>= graspLiftHeightDelta`。
  - 連續達成 `graspSuccessThreshold` 次視為成功。
- Place
  - 方塊與目標點距離 `<= placeDistanceThreshold`。
  - 若 `requireReleaseForPlace` 為真，需鬆開夾爪。
  - 連續達成 `placeSuccessThreshold` 次視為成功。

## 遊戲參數設定

可透過 `GameParametersManager` 動態調整的參數鍵（例如透過 SideChannel 傳入）：

- `trainingPhase` / `TrainingPhase`：關卡階段（`Reach` / `Grasp` / `Place`）
- `maxTimeSeconds` / `m_MaxTimeSeconds`：單回合最大時間（秒）
- `minRadius` / `maxRadius`：工作區半徑範圍（公尺）
- `minAngleDeg` / `maxAngleDeg`：工作區扇形角度範圍（度）
- `spawnMinRadius` / `spawnMaxRadius`：生成範圍半徑（公尺）
- `spawnMinAngleDeg` / `spawnMaxAngleDeg`：生成範圍角度（度）
- `minTargetGoalDistance`：方塊與目標點最小距離（公尺）

## Python 命令列

### 第一階段 Reach 訓練
```bash
python -m mlgame3d -i ./scripts/robot_arm_training_p1.py -e 600 -ts 5 -gp minRadius 1.1 -gp maxRadius 1.8 -gp minAngleDeg 180 -gp mazAngleDeg 360 spawnMinRadius 1.3 -gp spawnMaxRadius 1.5 -gp spawnMinAngleDeg 210 -gp spawnMaxAngleDeg 330 -gp maxTimeSeconds 10 -gp trainingPhase Reach
```

### 第一階段 Grasp 訓練
```bash
python -m mlgame3d -i ./scripts/robot_arm_training_p1.py -e 600 -ts 5 -gp trainingPhase Grasp
```
## 觀察空間

完整定義見 `Assets/Resources/observation_structure.json`。主要觀察量如下：

- 位置/向量
  - `cube_position`（Vector3）
  - `cube_initial_position`（Vector3）
  - `goal_position`（Vector3）
  - `end_effector_position`（Vector3）
  - `vector_from_end_effector_to_target`（Vector3）
  - `vector_from_target_to_goal`（Vector3）
  - `vector_from_end_effector_to_goal`（Vector3）
  - `vector_from_base_to_cube_planar`（Vector3）
  - `FK_direction_unitVector`（Vector3）
- 距離
  - `distance_from_end_effector_to_target`（float）
  - `distance_from_target_to_goal`（float）
  - `distance_from_end_effector_to_goal`（float）
  - `distance_from_base_to_cube_planar`（float）
  - `normalized_distance_from_base_to_cube_planar`（float）
- 關節狀態（皆為正規化值）
  - `joint_1_angle_normalized` / `joint_2_angle_normalized` / `joint_3_angle_normalized` / `joint_4_angle_normalized`
  - `joint_1_delta_angle_normalized` / `joint_2_delta_angle_normalized` / `joint_3_delta_angle_normalized` / `joint_4_delta_angle_normalized`
  - `joint_1_commanded_normalized` / `joint_2_commanded_normalized` / `joint_3_commanded_normalized` / `joint_4_commanded_normalized`
- 狀態旗標
  - `is_gripper_closed`（bool）
  - `is_success`（bool）
  - `is_failed`（bool）
- 扁平化向量
  - `flattened`（Vector，`vector_size: 26`，用於 PPO 訓練）

## 動作空間

- 4 維連續動作（`ContinuousActions[0..3]`），每維範圍 `[-1, 1]`。
- 關節 1~3：以 `maxDeltaDeg`（預設 5 度）做單步增量控制。
- 關節 4（夾爪）：以絕對角度控制，依關節最小/最大角度映射（0~50 度）。
- 在 Reach 階段，夾爪角度會被鎖定為 `reachGripperAngle`。
