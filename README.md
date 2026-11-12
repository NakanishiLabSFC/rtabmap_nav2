# rtabmap_nav2 — TB3 × D435i × RTAB-Map(VSLAM) × Nav2

TurtleBot3 (burger) に RealSense D435i を載せて、**RTAB-Map の Visual SLAM で地図作成・自己位置推定**し、
**Nav2 で自律移動**させる構成の引き継ぎ資料です。
ローカルコストマップには **Spatio-Temporal Voxel Layer (STVL)** を使っていて、カメラの点群から障害物をボクセルで出します。

> 作業時期：2025-12-01 〜 2026-01-05（最終的な設定は `param/nav2_param_scan6.yaml`）
> 環境：Ubuntu 22.04 / ROS 2 Humble / TurtleBot3 burger / RealSense D435i

---

## いまどこまでできているか

| 項目 | 状態 | メモ |
|---|---|---|
| D435i の起動（RGB + aligned depth） | ✅ できた | |
| RTAB-Map で地図作成（mapping） | ✅ できた | DB は `~/maps/new_session_*.db`。大きいのでリポジトリには入れていない |
| RTAB-Map の地図で自己位置推定（localization） | ✅ できた | `map -> odom` は RTAB-Map が出す |
| TB3 の TF / odom を使う | ✅ できた | TB3 bringup が `odom -> base_footprint -> base_link` を出す |
| カメラのTF | ✅ できた | `base_link -> camera_link` を `ros2 run` の static TF で出す |
| Nav2 で RTAB-Map の地図を使う | ✅ できた | global costmap が `/rtabmap/grid_prob_map` を直接読む（map_server / AMCL なし） |
| STVL でボクセルを出す | ✅ できた | 入力は `/rtabmap/local_grid_obstacle` |
| RViz からゴールを送って走らせる | ✅ できた | 2D Goal Pose |
| オドメトリの整理 | ⚠️ 未整理 | TB3 odom と visual odom が二重になっている（Issue 参照） |

---

## 構成

```
 [TB3 ラズパイ]                          [PC]
 turtlebot3_bringup ──/odom, TF───┐
   odom -> base_footprint         │   static_transform_publisher
   base_footprint -> base_link    │     base_link -> camera_link
                                  │
 /cmd_vel <───────────────────────┼── Nav2 (navigation_launch)
                                  │     ├ global: static_layer(/rtabmap/grid_prob_map) + obstacle + inflation
                                  │     └ local : STVL(/rtabmap/local_grid_obstacle) + inflation  ← ボクセル
                                  │
                                  │   RTAB-Map  → map -> odom, /rtabmap/grid_prob_map, /rtabmap/local_grid_obstacle
                                  │
                                  └── realsense2_camera (D435i)
```

TFツリー：`map → odom → base_footprint → base_link → camera_link → camera_*_frame`
（当時の実物：`docs/tf/tf_tree_2025-12-16_with_camera.pdf`）

---

## 必要なパッケージ

```bash
sudo apt install ros-humble-turtlebot3* ros-humble-navigation2 ros-humble-nav2-bringup \
  ros-humble-rtabmap-ros ros-humble-realsense2-camera ros-humble-spatio-temporal-voxel-layer
```

## 環境設定（全ターミナル共通）

PC と TB3 で **`ROS_DOMAIN_ID` を揃える**こと。

```bash
source /opt/ros/humble/setup.bash
export TURTLEBOT3_MODEL=burger
export ROS_DOMAIN_ID=30
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
# 共有メモリを切って UDP のみにする（PC⇔TB3 の通信が詰まる対策）
export FASTDDS_DEFAULT_PROFILES_FILE=<このリポジトリ>/param/fastdds_no_shm.xml
```

---

## 動かし方（ターミナル6つ）

### 1. TB3 本体（TB3 のラズパイで）
```bash
ros2 launch turtlebot3_bringup robot.launch.py
```

### 2. カメラの TF（PC）
カメラは base_link から前 0.10m・上 0.15m に付けていた。
```bash
ros2 run tf2_ros static_transform_publisher 0.10 0 0.15 0 0 0 base_link camera_link
```

### 3. D435i（PC）
```bash
ros2 launch realsense2_camera rs_launch.py device_type:=d435i align_depth.enable:=true publish_tf:=true \
  depth_module.profile:=848x480x30 rgb_camera.profile:=848x480x30
```

### 4. RTAB-Map（PC）
**地図で自己位置推定（Nav2 で走らせるときはこっち）**
```bash
ros2 launch rtabmap_launch rtabmap.launch.py \
  database_path:=$HOME/maps/new_session_202615.db localization:=true \
  publish_tf:=true frame_id:=base_footprint odom_frame_id:=odom map_frame_id:=map \
  subscribe_depth:=true subscribe_rgbd:=false \
  rgb_topic:=/camera/color/image_raw \
  depth_topic:=/camera/aligned_depth_to_color/image_raw \
  camera_info_topic:=/camera/color/camera_info \
  approx_sync:=true rtabmap_viz:=true \
  Grid/FromDepth:=true Grid/RangeMax:=4.0 \
  publish_map:=true publish_prob_map:=true publish_obstacles:=true use_sim_time:=false
```

**地図を新しく作るとき**：上のコマンドを `localization:=false args:=--delete_db_on_start` にして、`database_path` を新しいファイル名にする（指定したDBは消えるので注意）。
別ターミナルで `ros2 run turtlebot3_teleop teleop_keyboard` で走らせて、終わったら Ctrl+C で DB が保存される。

> ⚠️ 当時は `rgbd_odometry:=false` も付けていたが、これは存在しない引数で無視される（Issue 参照）。

### 5. Nav2（PC）
地図は RTAB-Map から来るので、map_server なしの `navigation_launch.py` を使う。
```bash
ros2 launch nav2_bringup navigation_launch.py \
  params_file:=<このリポジトリ>/param/nav2_param_scan6.yaml use_sim_time:=false
```

### 6. RViz（PC）
```bash
ros2 launch nav2_bringup rviz_launch.py
```
- 2D Goal Pose でゴールを送ると走る
- ボクセルを見るには Add → By topic → `/local_costmap/voxel_grid`（PointCloud2）

---

## ファイル

```
param/nav2_param_scan6.yaml   Nav2 のパラメータ（最終版）
param/fastdds_no_shm.xml      FastDDS を UDP のみにする設定
docs/tf/                      当時の TF ツリー（view_frames の出力）
docs/logs/                    2026-01-05 の RTAB-Map localization 起動ログ
docs/issues/                  次にやるとよさそうなこと（GitHub の Issue と同じ内容）
```

### nav2_param_scan6.yaml のポイント
- `local_costmap`：`voxel_layer` → **`stvl_layer`**。`voxel_decay: 1.0` で見えなくなった障害物は1秒で消える（残像対策）
- `local_costmap` の入力：`/rtabmap/local_grid_obstacle`（PointCloud2）。LiDAR の `/scan` はコメントアウト
- `global_costmap` の `static_layer`：`map_topic: /rtabmap/grid_prob_map`
- controller：MPPI

---

## ハマりどころ
- **`TF of received image for camera 0 ... is not set!`** → 2 の static TF の起動忘れか、TB3 の TF が届いていない（DOMAIN_ID / ネットワーク）
- **`Did not receive data since 5 seconds!`** → カメラのトピックが来ていない。`ros2 topic hz /camera/color/image_raw` で確認
- RTAB-Map の DB は数百MB〜数GBあるので Git に入れない（`.gitignore` 済み）
