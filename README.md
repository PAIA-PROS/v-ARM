# v-ARM

[![MuJoCo](https://img.shields.io/badge/MuJoCo-3.13-blue)](https://mujoco.org/) [![MLGame3D](https://img.shields.io/pypi/v/mlgame3d?label=MLGame3D
)](https://pypi.org/project/mlgame3d/)

v-ARM 是「行動操作機器人」（mobile manipulator）訓練環境：一支 4 軸機械手臂加平行夾爪，裝在麥克納姆輪（mecanum）移動底盤上。
自 **3.0.0** 起底層物理引擎由 Unity ML-Agents 改為 **MuJoCo**，並沿用同一套觀測、動作、規則與 PAIA Robot / MLGame3D 介面，
學生的 Blockly 積木與 Python 腳本不用改就能在兩個版本之間互換。

## 下載

[![Windows](https://custom-icon-badges.demolab.com/badge/Windows-3.0.1-blue?logo=windows)](https://github.com/PAIA-PROS/v-ARM/releases/download/3.0.1/v-ARM-win32-3.0.1.zip)
[![macOS](https://img.shields.io/badge/macOS-3.0.1-red?logo=apple)](https://github.com/PAIA-PROS/v-ARM/releases/download/3.0.1/v-ARM-darwin-universal-3.0.1.zip)

2.1.0 及更早的 Unity 版本仍可在 [Releases](https://github.com/PAIA-PROS/v-ARM/releases) 頁面下載。

## 3.0.0：底層改為 MuJoCo

3.0.0 是 v-ARM 第一個以 MuJoCo 物理引擎實作的版本，遊戲程式改為 Python（PyInstaller 打包），不再是 Unity build。

**不變的部分**

- PAIA Robot / MLGame3D 介面完全相同（ML-Agents gRPC 協定、behavior `Robot`、7 維連續動作、55 維觀測），`python -m mlgame3d … <path>/v-ARM` 的用法不變。
- 觀測欄位、動作定義、鍵盤鍵位、關節限位與 ROS2 `/cmd_vel` 慣例都與 Unity 版一致，學生的 Blockly 積木與 Python 腳本不用改。
- 底盤仍為運動學模擬；感知仍是幾何可見性判定（FOV、距離、視角），沒有真實影像 ArUco 偵測。

**改變的部分**

- **真實物理夾取**：夾爪靠指墊摩擦力把 30 mm 方塊夾住並抬起，放開就掉（Unity 版是運動學鎖定）。夾歪、夾太淺或鬆爪都會掉，`has_object` 代表兩指墊都確實接觸到方塊。
- **關卡**：提供 **Navigation** 與 **PickPlace** 兩關。PickPlace 取代原本的 Grasp 與 Place，在同一站把方塊夾起、放到放置標記（ArUco id 4）上並鬆爪。Align 與托盤六格放置暫不提供，相關觀測欄位（`dock_*`、`place_cell*_ee_local`、`ee_to_goal_ee_local`）保留但恆為 0。
- **Navigation 判定**：成功門檻改為車頭到進場點 `<= 0.25 m`（Unity 版 0.4 m）；從箭頭前方靠近（越過箭頭朝向且離箭頭 `< 0.4 m`）一律算 keepout 失敗（Unity 版預設關閉）；逾時算失敗。
- **觀測**：新增 `place_marker_visible`、`place_marker_ee_local`（放置標記）；`task_phase` 的 PickPlace 值為 4。
- **遊戲參數**精簡為 `trainingPhase`、`maxTimeSeconds`、`navMinDistanceFromBase`，Unity 版的 `navFieldSize*`、`navSpawnAroundStandoff`、`cbfFilterEnabled` 等不再提供。
- **場景**：固定 3 × 3 m 場地、30 cm 牆、取物站檯面；方塊頂面與放置標記使用真實 ArUco DICT_4X4_50 圖案。
- **視窗**：第三人稱追蹤視角 + 手臂相機子母畫面 + 狀態 HUD（流程列、倒數、距離／雷達／羅盤、夾爪狀態、成功徽章），`F5`–`F8` 切換顯示，滑鼠可旋轉／平移／縮放視角。

## 遊戲敘述

- 場地 3 × 3 m，四周有 30 cm 高的牆。
- **Navigation（導航）**：導航目標箭頭在場地內隨機生成，車子要開到箭頭後方的進場點。
- **PickPlace（取放）**：車子停在取物站前，檯面上有一顆 30 mm 方塊（頂面 ArUco DICT_4X4_50 id 10）與一張放置標記（ArUco id 4，24 mm）。用手臂相機找到兩者，夾起方塊放到標記上並鬆爪。
- 每回合有時間限制（預設 60 秒），逾時算失敗。
- 夾取是真實物理：夾爪靠摩擦力把方塊夾住，夾歪、夾太淺或鬆爪都會掉。

## 如何遊玩

執行時會開一個 MuJoCo 視窗：第三人稱視角跟著車子，右下角是手臂相機的子母畫面，上方與兩側是狀態 HUD。
鍵盤（視窗要有焦點）：

底盤（`action[0..2]`）：
- 前進 / 後退：`W` / `S`
- 橫移左 / 右：`A` / `D`（**Navigation 階段無效**）
- 左轉 / 右轉：`Q` / `E`

手臂（`action[3..6]`，絕對關節角，每步 ±1.5°）：
- 關節 1（J1）：`J`（−）/ `L`（＋）
- 關節 2（J2）：`K`（−）/ `I`（＋）
- 關節 3（J3）：`U`（−）/ `O`（＋）
- 夾爪（J4）：`F`（閉）/ `R`（開）

視窗：`F5` HUD 開關、`F6` 子母畫面左下 / 右下、`F7` 子母畫面開關、`F8` 影子開關；滑鼠左鍵拖曳旋轉、右鍵拖曳平移、滾輪縮放。

## 遊戲關卡

`trainingPhase` 可選 **Navigation** 與 **PickPlace**（成功需連續數個 decision 維持才算過關；decision 間隔 0.1 秒）。

- **Navigation（導航）**
  - 成功：車頭到「箭頭後方 0.4 m 的進場點」平面距離 `<= 0.25 m`，連續 3 個 decision。畫面上的綠色圓環就是這個範圍。
  - 失敗：從箭頭前方靠近（越過箭頭朝向且離箭頭 `< 0.4 m`，keepout）；逾時。
  - 只以位置判定，朝向只記錄不納入。
- **PickPlace（取放）**
  - 成功：曾經閉爪（夾過方塊）之後鬆爪、方塊不在手上，且方塊與放置標記的平面距離 `<= 0.03 m`，連續 5 個 decision。
  - 失敗：方塊掉到檯面下（低於檯面 3 cm）；逾時。

## 遊戲參數設定

由 MLGame3D 以 `-gp <key> <value>` 傳入（PAIA Robot 介面的參數表單就是這些）：

- `trainingPhase`：`Navigation` / `PickPlace`
- `maxTimeSeconds`：單回合最大時間（秒），預設 60
- `navMinDistanceFromBase`：導航目標離車身的最小生成距離（公尺），預設 0.3，上限 3.0

## Python 命令列

```bash
# 經 MLGame3D（與 PAIA Robot 相同路徑；最後一個參數是遊戲 app）
python -m mlgame3d -i ./scripts/student_pick_place.py -e 3 -gp trainingPhase PickPlace <path>/v-ARM

# Navigation 階段 PPO 訓練（HualienRobot/python/training，產出 nav.zip）
python -m mlgame3d -i ./scripts/train_nav.py -e 600 -ts 5 -dp 5 -ng -gp trainingPhase Navigation <path>/v-ARM
```

`-e` 回合數、`-ts` 倍速、`-dp` decision period（5 = 0.1 秒）、`-ng` 不開視窗。

`<path>/v-ARM` 是 PAIA Robot 安裝遊戲的位置：macOS 在 `~/Library/Application Support/PAIA Robot/games/v-ARM/v-ARM`，
Windows 在 `%APPDATA%\PAIA Robot\games\v-ARM\v-ARM`（mlagents 會自動接上 `.app` / `.exe`）。

## 觀察空間

完整定義見 `observation_structure.json`（55 維）。分兩層：

- **Structured snapshot**：命名欄位，供學生 Python / 積木與 reward shaping 使用。
- **Flattened 向量**：11 維，導航 PPO policy 的輸入。

### Structured snapshot

| key | 中文名稱 |
| --- | --- |
| `task_phase` | 任務相位（0 導航／4 取放） |
| `ee_to_target_ee_local` | 方塊相對末端執行器位置（ee-local 座標：x 前、y 左、z 上，公尺）。看不到時為上次的值，請自行 latch |
| `ee_to_goal_ee_local` | 目標格相對末端執行器位置（放置階段限定） |
| `target_visible` | 方塊 ArUco 是否在手臂相機視野內 |
| `is_caging` | 方塊是否已對正在夾爪指間 |
| `has_object` | 夾爪是否夾住方塊（兩指墊都接觸到方塊） |
| `distance_base_to_nav_target_planar` | 車頭到導航目標的平面距離 |
| `nav_yaw_error_deg` | 車身朝向與導航目標朝向的誤差（度） |
| `nav_along_track` | 沿導航目標軸的帶號距離（負 = 箭頭尾側） |
| `nav_cross_track` | 垂直導航目標軸的帶號偏移 |
| `nav_yaw_err_to_bearing_deg` | 車身朝向與「到目標方位」的誤差（度） |
| `nav_yaw_err_to_standoff_bearing_deg` | 車身朝向與「到進場點方位」的誤差（度） |
| `dock_marker_visible`、`dock_lateral`、`dock_forward`、`dock_yaw_err_deg` | 站台標記（Align 階段用，PickPlace 恆為 0） |
| `place_cell0_ee_local`～`place_cell5_ee_local` | 放置托盤六格中心相對末端（Placement 階段用） |
| `place_marker_visible` | 放置標記（ArUco id 4）是否在手臂相機視野內 |
| `place_marker_ee_local` | 放置標記中心相對末端執行器位置（ee-local，公尺） |
| `is_success` | 本回合判定成功 |
| `is_failed` | 本回合判定失敗 |

ee-local 向量可用積木「把末端座標向量換算成底座座標點」（Python：`ee_to_base`）換成底座座標，再用「伸向底座座標點的行動」（`reach_base`）做逆運動學。

### Flattened 向量（11 維，導航 policy 輸入）

| 索引 | key | 中文名稱 |
| --- | --- | --- |
| 0 | `base_vx_norm` | 底盤側向速度（正規化） |
| 1 | `base_vz_norm` | 底盤前向速度（正規化） |
| 2 | `base_yaw_rate_norm` | 底盤自轉角速度（正規化） |
| 3 | `prev_base_strafe_cmd_norm` | 上一步橫移命令 |
| 4 | `prev_base_forward_cmd_norm` | 上一步前進命令 |
| 5 | `prev_base_yaw_rate_cmd_norm` | 上一步自轉命令 |
| 6 | `nav_bearing_sin` | 到導航目標方位角 sin |
| 7 | `nav_bearing_cos` | 到導航目標方位角 cos |
| 8 | `nav_yaw_error_sin` | 對導航目標朝向偏航誤差 sin |
| 9 | `nav_yaw_error_cos` | 對導航目標朝向偏航誤差 cos |
| 10 | `nav_distance_norm` | 到導航目標平面距離（正規化） |

## 動作空間

7 維連續動作：

| 索引 | 意義 | 使用階段 |
| --- | --- | --- |
| `[0]` | 底盤前進速度（-1～1，+ 前） | Navigation |
| `[1]` | 底盤橫移速度（-1～1，+ 左；Navigation 階段忽略） | PickPlace |
| `[2]` | 底盤自轉速率（-1～1，+ 左轉） | Navigation |
| `[3]` | 關節 1（J1）絕對角度（度，0～180；90 = 正前） | PickPlace |
| `[4]` | 關節 2（J2）絕對角度（度，0～90；90 = 垂直） | PickPlace |
| `[5]` | 關節 3（J3）絕對角度（度，0～180） | PickPlace |
| `[6]` | 夾爪命令（`>0` 閉／`<=0` 開） | PickPlace |

- 底盤 `[0..2]` 對映 ROS2 `/cmd_vel`，模擬與真機同一慣例；Navigation 階段手臂動作被忽略，PickPlace 階段底盤不動。
- 手臂 `[3..5]` 是絕對關節角（度），伺服以有限速度追到目標；起始搬運姿為 (90, 90, 0)。
- 夾爪 `[6]`：閉合時指墊以有限力夾持，方塊靠摩擦力被拿起。
