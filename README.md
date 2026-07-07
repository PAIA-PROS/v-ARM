# v-ARM

[![Unity](https://img.shields.io/badge/Unity-%23000000.svg?logo=unity&logoColor=white)](#) [![MLGame3D](https://img.shields.io/pypi/v/mlgame3d?label=MLGame3D
)](https://pypi.org/project/mlgame3d/)

v-ARM 是一個以 Unity ML-Agents 製作的「行動操作機器人」（mobile manipulator）訓練環境：一支 4 軸機械手臂安裝在麥克納姆輪（mecanum）移動底盤上。完整任務鏈為 Navigation（導航到站台）→ Align（對位）→ Grasp（抓取）→ Place（放置到托盤格），每個階段可獨立遊玩訓練。

## 下載

[![Windows](https://custom-icon-badges.demolab.com/badge/Windows-2.0.0-blue?logo=windows)](https://github.com/PAIA-PROS/v-ARM/releases/download/2.0.0/v-ARM-win32-2.0.0.zip)
[![macOS](https://img.shields.io/badge/macOS-2.0.0-red?logo=apple)](https://github.com/PAIA-PROS/v-ARM/releases/download/2.0.0/v-ARM-darwin-universal-2.0.0.zip)

## 遊戲敘述

- 導航目標（箭頭／站台）在場地內隨機生成；目標方塊生成在取物站檯面上，放置目標為托盤六格之一。
- 需在時間限制內達成當前關卡（`trainingPhase`）的成功條件。
- 失敗條件依關卡而異（撞到障礙、夾住前撞動方塊、方塊掉出檯面、放錯目標格、逾時等），詳見「遊戲關卡」。

## 如何遊玩

- 用 Unity 開啟專案並進入 Play Mode（將 `RobotArmAgent → Behavior Parameters → Behavior Type` 設為 `Heuristic`）。
- 使用鍵盤控制（Heuristic 模式，鍵位對映 ROS2 `/cmd_vel` 與絕對關節角）：

  底盤（`action[0..2]`）：
  - 前進 / 後退：`W` / `S`（linear.x，+前）
  - 橫移左 / 右：`A` / `D`（linear.y，+左；**僅非 Navigation 階段有效**，見下方動作空間說明）
  - 左轉 / 右轉：`Q` / `E`（angular.z，+左轉 / CCW）

  手臂（`action[3..6]`，絕對關節角，每步 ±1.5°）：
  - 關節 1（J1）：`J`（−）/ `L`（＋）
  - 關節 2（J2）：`K`（−）/ `I`（＋）
  - 關節 3（J3）：`U`（−）/ `O`（＋）
  - 夾爪（J4）：`F`（閉，link4 = 25°）/ `R`（開，link4 = 90°）

- 依當前階段達成 Navigation、Align、Grasp 或 Place 的成功條件。

## 遊戲關卡

四個關卡階段：**Navigation（導航）、Align（對位）、Grasp（抓取）、Place（放置）**。成功／失敗判定見 `PhaseRuleSet`（成功需連續數幀維持才算過站；括號內為程式預設值）。

- **Navigation（導航）**
  - 成功：車頭到 standoff 判定點的平面距離 `<= navDistanceThreshold`（0.4 m）。連續 `navSuccessThreshold`（3）幀。
  - 註：僅以「位置」判定成功；朝向（yaw）與 tail-corridor 只記錄不納入，最終對齊交給實體 parking controller。
  - 失敗：車身撞到 `NavObstacleHazard`；（選用）車頭進入前方 keepout 扇形硬失敗（`navFailOnKeepoutEntry`，預設關）。
- **Grasp（抓取）**
  - 成功：已穩定夾住方塊（`has_object`）**且**手臂到達固定搬運姿態 `graspSuccessPoseDeg`（J1,J2,J3 = 90,90,0）、每關節 `±graspPoseToleranceDeg`（±5°）。連續 `graspSuccessThreshold`（5）幀。
  - 失敗：夾住前撞動方塊（水平位移超過 `graspBumpDisplacementThreshold`，0.02 m）；或方塊掉到檯面下。
- **Align（對位）**
  - 成功：站台標記（dock marker）可見，且三軸同時進容差——縱深 `dock_forward` 進 `alignTargetForward`（0.10 m）`±alignForwardToleranceM`（0.05 m）、側向 `|dock_lateral| <= alignLateralToleranceM`（0.05 m）、朝向 `|dock_yaw_err_deg| <= alignYawToleranceDeg`（10°）。連續 `alignSuccessThreshold`（10）幀。
  - 判定訊號（`dock_forward` / `dock_lateral` / `dock_yaw_err_deg`）與真機 `student_dock` 部署腳本同一套。
- **Place（放置）**
  - 成功：已放手（`!isGripperClosed`）**且**方塊與目標格距離 `<= placeDistanceThreshold`（0.015 m）。連續 `placeSuccessThreshold`（1）幀。
  - 失敗：放手落定（約 0.5 s）後仍未進目標格（進錯格 / 掉格外）。

## 遊戲參數設定

可透過 `GameParametersManager` 動態調整的參數鍵（由 MLGame3D 以 `-gp <key> <value>` 經 SideChannel 傳入）。鍵名以 `GameParametersManager.cs` 的 `*Key` 常數為準。

通用：

- `trainingPhase` / `TrainingPhase`：關卡階段（`Navigation` / `Grasp` / `Place` / `Align`）
- `maxTimeSeconds` / `m_MaxTimeSeconds`：單回合最大時間（秒）

Navigation 場地與目標生成：

- `navFieldSizeX` / `navFieldSizeZ`：導航場地尺寸（公尺）
- `navTargetYawMinDeg` / `navTargetYawMaxDeg`：導航目標（箭頭）朝向隨機範圍（度）
- `navSpawnAroundStandoff`：是否環繞 standoff 站位生成目標（bool）
- `navSpawnStandoffRadius`：環繞 standoff 生成的半徑（公尺）
- `navMinDistanceFromBase`：目標離底盤的最小距離（公尺）
- `navForceFrontSpawn`：是否強制在底盤前方生成（bool）
- `cbfFilterEnabled`：導航 keepout CBF safety filter 開關（>=0.5 啟用；eval/部署用，不送則不作用）

## Python 命令列

透過 MLGame3D harness 訓練與測試。RL 現況為 **nav-only**；`python/training/scripts/` 內容：

- `train_nav.py`（Nav PPO 訓練）、`eval_nav.py`（Nav 評估）
- `student_dock.py` / `student_grasp.py` / `student_place.py`：Align / Grasp / Place 單站腳本（感知-only 參考實作，介面同部署宿主）
- `student.py`：最小 MLPlay 範本
- 診斷工具：`drive_test.py`、`probe_reach.py`、`strafe_gate.py`、`student_arm_j1_test.py`

指令從 `python/training/` 目錄執行（先 `source .venv/bin/activate`）。

```bash
# Navigation 階段訓練（產出 nav.zip）
python -m mlgame3d -i ./scripts/train_nav.py -e 600 -ts 5 \
  -gp trainingPhase Navigation
```

- `-gp trainingPhase` 的合法值對應 `PhaseRuleSet.TrainingPhase` enum；nav 訓練用 `Navigation`，model 檔名自動推導為 `nav.zip`。
- 場地／生成參數一律走 `-gp` CLI 傳（鍵名見上方「遊戲參數設定」），不寫進腳本。

## 觀察空間

完整定義見 `Assets/Resources/observation_structure.json`。觀察分為兩層：

- **Structured snapshot**：命名欄位，供學生 Python FSM、eval 與 reward shaping 使用（非 PPO 輸入）。
- **Flattened 向量**：11 維

### Structured snapshot（逐項中文名稱）

| key                                            | 中文名稱                                                                                                             |
| ---------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| `task_phase`                                   | 任務相位（0 導航／1 抓取／2 放置／3 對位 Align）                                                                     |
| `ee_to_target_ee_local`                        | 方塊相對末端執行器位置（ee-local 座標）                                                                              |
| `ee_to_goal_ee_local`                          | 目標格相對末端執行器位置（放置階段限定）                                                                             |
| `target_visible`                               | 方塊 ArUco 是否可見                                                                                                  |
| `is_caging`                                    | 方塊是否已對正夾在夾爪指間                                                                                           |
| `has_object`                                   | 夾爪是否持有方塊                                                                                                     |
| `distance_base_to_nav_target_planar`           | 底盤到導航目標的平面距離                                                                                             |
| `nav_yaw_error_deg`                            | 對導航目標朝向的偏航誤差（度）                                                                                       |
| `nav_along_track`                              | 沿軌距離（沿導航朝向軸，負=在箭頭尾側）                                                                              |
| `nav_cross_track`                              | 橫軌距離（偏離中心線的垂直距離）                                                                                     |
| `nav_yaw_err_to_bearing_deg`                   | 對「到目標方位」的偏航誤差（度）                                                                                     |
| `nav_yaw_err_to_standoff_bearing_deg`          | 對「到虛擬 standoff 站位方位」的偏航誤差（度）                                                                       |
| `dock_marker_visible`                          | 對位站台標記是否可見                                                                                                 |
| `dock_lateral`                                 | 站台標記側向偏移（m，對正→0）                                                                                        |
| `dock_forward`                                 | 站台標記前方深度（m，→ standoff 距離）                                                                               |
| `dock_yaw_err_deg`                             | 車頭對站台標記的偏航誤差（度，正對→0）                                                                               |
| `place_cell0_ee_local`～`place_cell5_ee_local` | 放置托盤六格中心相對末端（ee-local；0-2 前排、3-5 後排；放置階段限定。用 `ee_to_base(observations['place_cellN_ee_local'])` 換算 base 座標） |
| `is_success`                                   | 本回合判定成功                                                                                                       |
| `is_failed`                                    | 本回合判定失敗（依關卡而異：撞障礙／撞動方塊／掉檯面／逾時等）                                                                          |

### Flattened 向量（11 維，PPO 導航 policy 輸入）

| 索引 | key                           | 中文名稱                                           |
| ---- | ----------------------------- | -------------------------------------------------- |
| 0    | `base_vx_norm`                | 底盤側向速度（正規化，base-local，來自里程計）     |
| 1    | `base_vz_norm`                | 底盤前向速度（正規化，base-local）                 |
| 2    | `base_yaw_rate_norm`          | 底盤自轉角速度（正規化，度/秒）                    |
| 3    | `prev_base_strafe_cmd_norm`   | 上一步底盤橫移命令（正規化）                       |
| 4    | `prev_base_forward_cmd_norm`  | 上一步底盤前進命令（正規化）                       |
| 5    | `prev_base_yaw_rate_cmd_norm` | 上一步底盤自轉命令（正規化）                       |
| 6    | `nav_bearing_sin`             | 到導航目標方位角 sin（抓取/放置階段遮罩為0）       |
| 7    | `nav_bearing_cos`             | 到導航目標方位角 cos（抓取/放置階段遮罩為0）       |
| 8    | `nav_yaw_error_sin`           | 對導航目標朝向偏航誤差 sin（抓取/放置階段遮罩為0） |
| 9    | `nav_yaw_error_cos`           | 對導航目標朝向偏航誤差 cos（抓取/放置階段遮罩為0） |
| 10   | `nav_distance_norm`           | 到導航目標平面距離（正規化，抓取/放置階段遮罩為0） |

## 動作空間

7 維連續動作（`ContinuousActions[0..6]`）。各代理只使用與自己階段相關的維度（手臂代理把 `[0..2]` 歸零、導航代理把 `[3..6]` 歸零）。

| 索引  | 意義                                       | 使用者                |
| ----- | ------------------------------------------ | --------------------- |
| `[0]` | 底盤 linear.x（前進，+前）                 | Navigation            |
| `[1]` | 底盤 linear.y（橫移，+左，mecanum strafe） | Navigation / 其他階段 |
| `[2]` | 底盤 angular.z（自轉，+左轉 / CCW）        | Navigation            |
| `[3]` | 手臂關節 1（J1）絕對角度（度）             | Grasp / Place         |
| `[4]` | 手臂關節 2（J2）絕對角度（度）             | Grasp / Place         |
| `[5]` | 手臂關節 3（J3）絕對角度（度）             | Grasp / Place         |
| `[6]` | 夾爪命令（`>0` = 閉 / `≤0` = 開）          | Grasp / Place         |

- 底盤 `[0..2]` 對映 ROS2 `/cmd_vel`（Twist），sim ↔ 真機同一慣例。
- 手臂 `[3..5]` 為**絕對關節角（度）**，`RobotArmDriver` 直收，無 `[-1,1]` 正規化來回。關節限位：J1 `0~180°`、J2 `0~90°`、J3 `0~180°`。
- 夾爪 `[6]`：`+1` = 閉（link4 = **25°**）、`−1` = 開（link4 = **90°**）。

### 橫移（strafe）階段策略

- **Navigation 階段禁止橫移**：底盤為車輛式（只前後 + 自轉），`action[1]`（linear.y）一律忽略。
- **其他階段（Grasp / Place / Align）開放橫移**：可使用麥克納姆橫移。
- 可用 `RobotArmAgent → allowBaseStrafe` 覆寫：設為 `true` 則連 Navigation 階段也開放橫移（除錯用）。此開關同時作用於 policy 與 Heuristic 的 `A` / `D`。
