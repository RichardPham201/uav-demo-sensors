# 🚁 UAV X-Vision: Hệ Thống Huấn Luyện Học Tăng Cường Cho UAV

Dự án **UAV X-Vision** cung cấp một hệ thống huấn luyện toàn diện cho máy bay không người lái (UAV - dựa trên mẫu khung *Iris* từ *Pegasus Simulation*) sử dụng Học tăng cường (Reinforcement Learning - RL). Được xây dựng trên nền tảng **Isaac Lab** và framework **RSL-RL**, hệ thống tập trung vào các tác vụ điều hướng, bám sát mục tiêu di động, hạ cánh, và tích hợp AI Computer Vision để nhận diện đối tượng/cảnh báo nguy hiểm (cháy nổ) trong các môi trường phức tạp.

## 🌟 Các tính năng chính

* **Đa dạng tác vụ (Multi-task):** Hỗ trợ Navigation (Bay đến đích), Tracking (Nhận diện & Bám mục tiêu di động/Random Walk), và Landing (Hạ cánh).
* **Kiến trúc Sim-to-Real & AI Perception:** Sử dụng Ground-Truth để huấn luyện cực nhanh trong Simulation. Hỗ trợ quy trình Deploy thực tế bằng cách ghép nối với model AI (YOLO/RT-DETR) để nhận diện người, xe, và đám cháy từ camera.
* **Curriculum Learning:** Huấn luyện mô hình từ dễ đến khó qua các giai đoạn (Stages).
* **Mô phỏng cảm biến chuẩn công nghiệp:** Hỗ trợ cấu hình raycaster và giả lập cho camera Intel RealSense D455 (RGB/Depth/IMU) và LiDAR Hesai XT32 SD10 (Point Cloud 360 độ).
* **Song song hóa cao (High Parallelism):** Chạy đồng thời 2048 - 4096 môi trường trực tiếp trên GPU qua PyTorch Tensor API (không dùng ROS 2 trong luồng train) để tối ưu hóa quá trình thu thập dữ liệu bằng thuật toán **PPO (Proximal Policy Optimization)**.
* **Hỗ trợ Deploy & Visualize:** Khả năng xuất mô hình đã huấn luyện sang định dạng **ONNX** và Bridge dữ liệu ra ROS 2 (Jazzy/Humble) để visualize trên Foxglove.

## 📁 Cấu trúc dự án

Hệ thống được tổ chức theo kiến trúc module hóa cao, chia tách rõ ràng giữa Environment, Markov Decision Process (MDP), và các Tasks:

```text
uav/
├── assets/                 # Chứa các model 3D (UAV Iris, môi trường, vật cản)
├── envs/                   # Cấu hình môi trường cốt lõi
│   ├── env_cfg.py          # Định nghĩa robot Iris, prim paths, cảm biến (Raycaster/Camera)
│   ├── observations/       # Xử lý dữ liệu RGB, Depth, LiDAR (Raycaster), Ground-Truth
│   │   ├── rgb_observation.py # dữ liệu camera sensor RGB UAV nhìn thấy
│   │   ├── thermal_observation.py # dữ liệu camera sensor nhiệt UAV nhìn thấy
│   │   ├── lidar_observation.py # dữ liệu lidar sensor UAV nhìn thấy (1D Vector)
│   │   └── observation.py # mix sensor observations
│   ├── rewards/            # Các hàm reward chung (tránh vật cản, phạt crash...)
│   │   ├── step_penalty.py  # Phạt theo thời gian sống (Living penalty)
│   │   ├── velocity_control_reward.py # thưởng control velocity
│   │   ├── crashing_penalty.py # phạt khi drone crash
│   │   ├── smoothness_penalty.py # Phạt bẻ lái gắt, chuyển động thừa của motor
│   │   └── obstacle_avoidance_reward.py # thưởng tránh vật cản/tránh vùng cháy (fire hazard)
│   ├── actions/            # Quản lý Action (Dual mode: Low-level Thrust hoặc High-level PID)
│   │   ├── action_manager.py # action cho drone điều khiển velocity
│   ├── events/             # Xử lý sự kiện (Reset env khi bắt đầu/kết thúc episode)
│   │   ├── on_episode_reset.py # reset environment khi episode kết thúc
│   │   └── on_episode_start.py # reset environment khi episode bắt đầu
├── tasks/                  # Các tác vụ huấn luyện chính
│   ├── __init__.py         # Đăng ký tasks với Isaac Lab
│   ├── navigation_task/    # Tác vụ bay đến đích
│   │   ├── navigation_task.py # task cho drone bay đến đích
│   │   ├── mdp/
│   │       ├── actions/
│   │       │   └── navigation_action.py # action cho drone bay đến đích
│   │       ├── envs/
│   │       │   └── navigation_env.py # environment cho drone bay đến đích
│   │       ├── observations/
│   │       │   └── navigation_observation.py # observation cho drone bay đến đích
│   │       └── rewards/
│   │           ├── progress_reward.py   # Thưởng thu hẹp khoảng cách tới đích
│   │           ├── reached_goal_reward.py # Thưởng khi bay đến đích thành công
│   │           └── navigation_reward.py # Tệp tổng hợp cấu hình reward
│   ├── tracking_object_task/ # Tác vụ bám sát mục tiêu
│   │   └── tracking_object_task.py # task cho drone bám sát mục tiêu
│   │   └── mdp/
│   │       ├── actions/
│   │       │   └── tracking_object_action.py # action cho drone bám sát mục tiêu
│   │       ├── envs/
│   │       │   └── tracking_object_env.py # environment cho drone bám sát mục tiêu
│   │       ├── observations/
│   │       │   └── tracking_object_observation.py # observation cho drone bám sát mục tiêu
│   │       └── rewards/
│   │           └── target_tracking_reward.py # thưởng tracking target
│   │           └── keep_target_in_view_reward.py # thưởng giữ target trong tầm nhìn
│   └── landing_task/       # Tác vụ hạ cánh
│       └── landing_task.py # task cho drone hạ cánh
│       └── mdp/
│           ├── actions/
│           │   └── landing_action.py # action cho drone hạ cánh
│           ├── envs/
│           │   └── landing_env.py # environment cho drone hạ cánh
│           ├── observations/
│           │   └── landing_observation.py # observation cho drone hạ cánh
│           └── rewards/
│               └── landing_reward.py # reward cho drone hạ cánh
├── scripts/                # Entry points để chạy dự án
│   ├── train.py            # Script huấn luyện chính
│   ├── play.py             # Script chạy mô hình đã huấn luyện (Có thể dùng ROS 2 Foxglove)
│   └── benchmark.py        # Đánh giá hiệu suất mô hình
├── logs/                   # Chứa checkpoint và logs của RSL-RL (logs/rsl_rl/{project}/{date})
└── utils/                  # Chứa các file config (drone_config.py) và script utility

```

## 🧠 Luồng hoạt động & Kiến trúc hệ thống

1. **Entry Point (`scripts/train.py`):** Tiếp nhận tham số CLI, khởi tạo không gian Isaac Sim thông qua `AppLauncher` và gọi quá trình huấn luyện.
2. **Orchestration:** Hệ thống tự động quản lý vòng đời huấn luyện, chuyển đổi giữa các Stage của Curriculum Learning, lưu checkpoint và cấu hình lại tọa độ spawn drone và target.
3. **Core Logic (Tensor-based):** Kế thừa từ `ManagerBasedRLEnvCfg` của Isaac Lab. Quản lý tính toán vật lý, ActionManager, ObservationManager trực tiếp trên VRAM GPU thông qua PyTorch Tensor, đảm bảo tốc độ mô phỏng hàng chục nghìn FPS.
4. **Deploy Pipeline (Real-world):** Ở pha thực tế, dữ liệu từ camera D455 đi qua mạng AI (như YOLO) để sinh ra vector ngữ cảnh (Tọa độ 3D của mục tiêu). Vector này cùng dữ liệu Point Cloud được đưa vào mạng MLP (ONNX model) để xuất tín hiệu điều khiển.

## 🚀 Phương pháp huấn luyện (Curriculum Learning) - Với task drone tracking

Hệ thống sử dụng mạng MLP (3 lớp ẩn x 128 units, ELU activation) và huấn luyện qua lộ trình 4 giai đoạn để UAV thích nghi dần với độ khó của môi trường:

| Giai đoạn | Tên Stage | Mục tiêu huấn luyện | Độ khó |
| --- | --- | --- | --- |
| **Stage 1** | **Hover** | Giữ độ cao ổn định tại một điểm cố định. | Rất dễ |
| **Stage 2** | **Track Static** | Bay đến và giữ khoảng cách với mục tiêu đứng yên. | Dễ |
| **Stage 3** | **Track Moving** | Bám theo mục tiêu di chuyển ngẫu nhiên (Random Walk). | Trung bình |
| **Stage 4** | **Domain Rand.** | Thêm nhiễu môi trường (sương mù, gió, ánh sáng). | Chuyên gia |

## 🚀 Phương pháp huấn luyện (Curriculum Learning) - Với task drone navigation

Hệ thống sử dụng mạng MLP (3 lớp ẩn x 128 units, ELU activation) và huấn luyện qua lộ trình 2 giai đoạn để UAV thích nghi dần với độ khó của môi trường:

| Giai đoạn | Tên Stage | Mục tiêu huấn luyện | Độ khó |
| --- | --- | --- | --- |
| **Stage 1** | **Hover** | Giữ độ cao ổn định tại một điểm cố định. | Rất dễ |
| **Stage 2** | **Track Static** | Bay đến điểm target | Dễ |

## ⚙️ Cấu hình kỹ thuật (Config)

### 1. Thông số Robot Iris & Motor Prims

*(Thông số vật lý chi tiết sẽ được đưa vào `utils/drone_config.py` để tiện tùy chỉnh khi nhận hardware)*

* **Model:** Iris (Pegasus Sim Assets). Trọng lượng tham khảo: ~1.2kg - 1.5kg.
* **Cơ chế điều khiển (Action Space):**
* *Pha 1 (Low-level Control):* RL điều khiển trực tiếp xung PWM/lực đẩy `[-1, 1]` của 4 rotor.
* *Pha 2 (High-level Control - Tùy chọn sau):* Output lệnh Roll, Pitch, Yaw, Thrust cho bộ điều khiển PID (Pixhawk/PX4).


* **Prims điều khiển:**
* `/Root/World/iris/rotor0` đến `/Root/World/iris/rotor3`



### 2. Không gian trạng thái (Observation Space)

Do mạng MLP của RSL-RL không xử lý tốt Point Cloud thô dạng 3D, hệ thống sử dụng kỹ thuật:

* **Ground-Truth (Trong Sim):** Lấy trực tiếp tọa độ tuyệt đối `[x,y,z]` của mục tiêu để mạng nhanh hội tụ. (Khi Deploy sẽ thay bằng đầu ra của AI Model).
* **RayCaster 1D Vector:** Giả lập LiDAR và Depth Camera bằng cách bắn tia raycaster, trả về một vector mảng 1D (ví dụ 108 chiều) đo khoảng cách đến vật cản.
* **Altitude/Optical Flow giả lập:** Cung cấp trục Z tuyệt đối để bù đắp sai số drift của IMU.

### 3. Cảm biến (Sensors) & ROS 2 Bridge

Cảm biến chuẩn công nghiệp được mô phỏng chính xác và hỗ trợ publish qua ROS 2 (Chỉ dùng lúc Deploy/Visualize, **không dùng lúc Train** để tránh nghẽn cổ chai CPU):

* **Hesai PandarXT-32 SD10 (LiDAR 360 - 10Hz):**
* Prim: `/Root/World/iris/body/Root/XT32_SD10/PandarXT_32_10hz`
* Topic: `/point_cloud`


* **Intel RealSense D455 (RGB Camera - OV9782 Color):**
* Prim: `/Root/World/iris/body/rsd455/RSD455/Camera_OmniVision_OV9782_Color`
* Topic: `/rgb`


* **Intel RealSense D455 (IMU Sensor):**
* Prim: `/Root/World/iris/body/rsd455/RSD455/Imu_Sensor`
* Topic: `/imu`



### 4. Thiết kế phần thưởng (Reward Shaping)

Hệ thống sử dụng các custom reward linh hoạt tùy vào Task:

* `target_tracking_reward`: Khuyến khích UAV nằm trên vùng không gian của mục tiêu.
* `keep_target_in_view_reward`: Đảm bảo mục tiêu luôn nằm trong khung hình camera trước.
* `fire_avoidance_penalty`: Phạt cực nặng nếu UAV bay vào bán kính < 3m của khu vực có dán nhãn ngữ nghĩa (Semantic Label) là `hazard_fire`. (Nhận diện lửa ngoài thực tế sẽ do mô hình AI đảm nhận).
* `obstacle_avoidance_reward` / `crashing_penalty`: Thưởng duy trì khoảng cách an toàn, phạt khi va chạm.
* `smoothness_penalty`: Phạt sự thay đổi Action đột ngột của motor.

### 5. Cấu hình Scene

* **Engine:** Tối ưu hóa cho Isaac Sim phiên bản 5.1.0 / Isaac Lab.
* **World Scene:** `/root/IsaacLab/uav/source_v2/assets/custom/Factory_drone_lidar_rgb.usd`
* **Reset Scenario:**
* UAV và Target sinh ngẫu nhiên trong vùng Collision-free zones (Bán kính 1x1m cho UAV, 2x2m cho Target). Khoảng cách an toàn tối thiểu ban đầu là 5m.
* Target di chuyển ngẫu nhiên (Random Walk) bằng plugin/asset có sẵn trong Isaac Sim.
* Timeout 30 giây cho mỗi Episode.



## 🛠️ Các tính năng huấn luyện nâng cao

Hệ thống hỗ trợ nhiều tính năng linh hoạt để tối ưu hóa quá trình huấn luyện và triển khai:

### 1. Chế độ huấn luyện (Training Modes)

* **Large Environments:** Khả năng mở rộng lên hàng ngàn môi trường song song (`--num_envs`).
* **Fabric GPU:** Tùy chọn sử dụng (`--disable_fabric=0`) để render nhanh trên GPU.
* **Stage-based & Resume Training:** Hỗ trợ `--stage` và `--resume` từ checkpoint.
* **Observation Noise:** Thêm Gaussian Noise vào IMU/Raycaster để tăng tính Robustness cho mô hình (Sim-to-Real).

### 2. Xuất mô hình (Export Model)

* **ONNX Export:** Chuyển đổi mô hình MLP sang **ONNX** để chạy trên Jetson Orin / TensorRT.

### 3. Giám sát và Lưu trữ (Logging & Recording)

* Tích hợp **TensorBoard / WandB**.
* Ghi luồng Video (`--video`) phục vụ báo cáo.

## 💻 Hướng dẫn chạy dự án

Sử dụng môi trường ảo có cài đặt Isaac Lab và chạy các lệnh sau từ thư mục gốc của dự án.

**1. Huấn luyện toàn bộ lộ trình (Curriculum Learning):**

```bash
conda activate env_isaaclab && OMNI_KIT_ALLOW_ROOT=1 ISAACSIM_PATH=/isaacsim LIVESTREAM=1 PUBLIC_IP= OMNI_KIT_ALLOW_FABRIC_RENDER=0 ./isaaclab.sh -p uav/source_v2/scripts/train.py --task UAV-Tracking-v2 --num_envs 1 --enable_camera

conda activate env_isaaclab && OMNI_KIT_ALLOW_ROOT=1 ISAACSIM_PATH=/isaacsim LIVESTREAM=1 PUBLIC_IP= OMNI_KIT_ALLOW_FABRIC_RENDER=0 ./isaaclab.sh -p uav/source_v2/scripts/train.py --task UAV-Navigation-v2 --num_envs 1 --enable_camera

conda activate env_isaaclab && OMNI_KIT_ALLOW_ROOT=1 ISAACSIM_PATH=/isaacsim LIVESTREAM=1 PUBLIC_IP= OMNI_KIT_ALLOW_FABRIC_RENDER=0 ./isaaclab.sh -p uav/source_v2/scripts/train.py --task UAV-Landing-v2 --num_envs 1 --enable_camera


```

**2. Visualize sensors data (Pha Kiểm thử / Deploy):**
Kết nối ra Foxglove trên PC Client.

```bash
ROS_DISTRO=jazzy

sudo apt install ros-$ROS_DISTRO-foxglove-bridge

ros2 launch foxglove_bridge foxglove_bridge_launch.xml address:=0.0.0.0 port:=8765

ros2 topic list
# Kết quả mong đợi: /imu, /rgb, /point_cloud

```

**3. Xem UAV đã huấn luyện (Play):**

```bash
./isaaclab.sh -p scripts/play.py --task tracking_object_task

```
