<div align="center">
  <video src="https://github.com/user-attachments/assets/2d0d30b7-14ad-4b86-9eb6-ea422a811269" 
    width="100%" 
    style="max-width: 900px; border-radius: 10px;" 
    autoplay 
    muted 
    loop 
    playsinline>
  </video>
</div>


# 🚁 UAV X-Vision: Reinforcement Learning Framework for Autonomous UAVs

## 🌟 Overview

**UAV X-Vision** is a comprehensive training system for Unmanned Aerial Vehicles (UAVs) based on the *Iris* frame (Pegasus Simulation). Built on **NVIDIA Isaac Lab** and the **RSL-RL** framework, this project focuses on high-performance Reinforcement Learning (RL) for complex tasks including autonomous navigation, dynamic object tracking, and precision landing.

The system integrates **AI Computer Vision** (YOLO/RT-DETR) for object detection and fire hazard identification, bridging the gap between simulation and real-world deployment (Sim-to-Real).

## 🚀 Key Features

*   **Multi-Task Support:** Specialized environments for **Navigation** (Point-to-Point), **Tracking** (Moving Targets/Random Walk), and **Landing**.
*   **High-Fidelity Sensor Simulation:** 
    *   **LiDAR:** Hesai XT32 SD10 (360° Point Cloud).
    *   **Camera:** Intel RealSense D455 (RGB/Depth/IMU).
*   **Massive Parallelism:** Trains 2048+ environments simultaneously on GPU using PyTorch Tensor API, achieving tens of thousands of FPS.
*   **Curriculum Learning:** Multi-stage training pipelines to gradually increase environment complexity.
*   **Sim-to-Real Pipeline:** Export models to **ONNX** for deployment on Edge AI hardware (Jetson Orin) and ROS 2 (Jazzy/Humble) integration for visualization via Foxglove.

---

## 📁 Project Structure

The project follows a modular architecture separating Environments, MDP (Markov Decision Process) logic, and specific Tasks:

```text
uav/
├── assets/                 # 3D Models (UAV Iris, Environments, Obstacles)
├── envs/                   # Core Environment Configurations
│   ├── env_cfg.py          # Robot, Prim paths, and Sensor definitions
│   ├── observations/       # Processing RGB, Depth, and LiDAR data
│   ├── rewards/            # Global reward functions (Avoidance, Smoothness)
│   └── actions/            # Action Manager (Velocity/Thrust control)
├── tasks/                  # Main RL Tasks
│   ├── navigation_task/    # Point-to-Point Navigation
│   ├── tracking_task/      # Dynamic Object Tracking
│   └── landing_task/       # Precision Landing
├── scripts/                # Entry points (train.py, play.py, benchmark.py)
└── utils/                  # Drone & Hardware configurations

```

---

## 🧠 Training Strategy: Curriculum Learning

We utilize an MLP architecture (3 hidden layers x 128 units) trained across multiple stages to ensure robust policy convergence.

### Tracking Task Roadmap

| Stage | Name | Objective | Difficulty |
| --- | --- | --- | --- |
| **Stage 1** | **Hover** | Maintain altitude at a fixed point. | Beginner |
| **Stage 2** | **Track Static** | Reach and stay near a stationary target. | Easy |
| **Stage 3** | **Track Moving** | Follow a target with Random Walk behavior. | Intermediate |
| **Stage 4** | **Domain Rand.** | Add environmental noise (Wind, Fog, Lighting). | Expert |

---

## ⚙️ Technical Specifications

### 1. Sensor Integration (ROS 2 Bridge)

The system supports industry-standard sensors for real-world parity:

* **LiDAR (Hesai XT32):** `/point_cloud` topic at 10Hz.
* **Camera (RealSense D455):** `/rgb` and `/imu` topics.
* **Note:** ROS 2 bridge is used for **inference/visualization only**, not during training to avoid CPU bottlenecks.

### 2. Reward Shaping

* `target_tracking_reward`: Encourages the UAV to stay within the target's vicinity.
* `fire_avoidance_penalty`: Heavy penalty for entering a 3m radius of `hazard_fire` semantic labels.
* `smoothness_penalty`: Reduces erratic motor behavior for longer hardware lifespan.

---

## 🛠️ Installation & Usage

### Training

Run the curriculum training for specific tasks:

```bash
# Activate environment
conda activate env_isaaclab

# Train Navigation Task
./isaaclab.sh -p uav/source_v2/scripts/train.py --task UAV-Navigation-v2 --num_envs 2048 --enable_camera

# Train Tracking Task
./isaaclab.sh -p uav/source_v2/scripts/train.py --task UAV-Tracking-v2 --num_envs 2048

```

### Visualization (Foxglove)

To visualize sensor data, launch the Foxglove bridge:

```bash
ros2 launch foxglove_bridge foxglove_bridge_launch.xml address:=0.0.0.0 port:=8765

```

### Playback

Test your trained model:

```bash
./isaaclab.sh -p scripts/play.py --task UAV-Tracking-v2

```

---

## 📡 Deployment (Sim-to-Real)

The trained MLP models can be exported to **ONNX** format. In real-world scenarios, the visual context vector (Target 3D coordinates) is provided by a **YOLO/RT-DETR** model, which is then fed into the RL policy to output control signals.

---

*Developed for AI-Native UAV Transformation Strategy.*

```
