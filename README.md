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

# 🚀 UAV X-Vision: Reinforcement Learning Training System for Autonomous UAVs

## 🌟 Key Features

* **Multi-task Capabilities:** Supports **Navigation** (Point-to-Point), **Target Tracking** (Dynamic/Random Walk), and **Precision Landing**.
* **Sim-to-Real Architecture & AI Perception:** Leverages Ground-Truth data for accelerated training in simulation. Supports real-world deployment by integrating AI models (YOLO/RT-DETR) for detecting people, vehicles, and fire hazards from camera feeds.
* **Curriculum Learning:** Implements a staged training approach, progressing from basic hovering to complex environmental navigation.
* **Industry-Standard Sensor Simulation:** High-fidelity simulation of the **Intel RealSense D455** (RGB/Depth/IMU) and **Hesai XT32 SD10 LiDAR** (360° Point Cloud).
* **High-Performance Parallelism:** Simultaneously runs 2,048 – 4,096 environments directly on the GPU via PyTorch Tensor API. Optimized data collection using **Proximal Policy Optimization (PPO)**.
* **Deployment & Visualization Ready:** Supports exporting trained models to **ONNX** format and bridging data to **ROS 2 (Jazzy/Humble)** for visualization via Foxglove.

## 📁 Project Structure

The project features a highly modular architecture, clearly separating Environments, Markov Decision Processes (MDP), and specific Tasks:

```text
uav/
├── assets/                 # 3D models (Iris UAV, environments, obstacles)
├── envs/                   # Core environment configurations
│   ├── env_cfg.py          # Definitions for Iris robot, prim paths, and sensors
│   ├── observations/       # Processing RGB, Depth, LiDAR, and Ground-Truth data
│   │   ├── rgb_observation.py      # UAV RGB camera sensor data
│   │   ├── thermal_observation.py  # UAV thermal camera sensor data
│   │   ├── lidar_observation.py    # LiDAR sensor data (1D Vector)
│   │   └── observation.py          # Sensor fusion and mixed observations
│   ├── rewards/            # General reward functions (Obstacle avoidance, crash penalties)
│   │   ├── step_penalty.py         # Living penalty based on time steps
│   │   ├── velocity_control_reward.# Velocity control optimization
│   │   ├── crashing_penalty.py     # Penalty for UAV collisions
│   │   ├── smoothness_penalty.py   # Penalty for abrupt maneuvers/motor jitters
│   │   └── obstacle_avoidance_reward.py # Reward for avoiding hazards (fire, walls)
│   ├── actions/            # Action Management (Low-level Thrust or High-level PID)
│   │   └── action_manager.py       # Velocity-based drone control actions
│   └── events/             # Event handling (Environment resets/initialization)
│       ├── on_episode_reset.py     # Reset logic at the end of an episode
│       └── on_episode_start.py     # Initialization logic at the start of an episode
├── tasks/                  # Primary training tasks
│   ├── navigation_task/    # Goal-oriented flight task
│   ├── tracking_object_task/# Moving target pursuit task
│   └── landing_task/       # Precision landing task
├── scripts/                # Entry points for project execution
│   ├── train.py            # Main training script
│   ├── play.py             # Inference script for trained models (ROS 2/Foxglove support)
│   └── benchmark.py        # Model performance evaluation
├── logs/                   # RSL-RL checkpoints and logging directory
└── utils/                  # Utility scripts and hardware configurations (drone_config.py)

```

## 🧠 System Architecture & Workflow

1. **Entry Point (`scripts/train.py`):** Handles CLI parameters, initializes the Isaac Sim environment via `AppLauncher`, and triggers the training loop.
2. **Orchestration:** Automatically manages the training lifecycle, including Curriculum Learning stage transitions, checkpoint saving, and dynamic spawning of UAVs/targets.
3. **Core Logic (Tensor-based):** Built on Isaac Lab's `ManagerBasedRLEnvCfg`. All physics calculations, actions, and observations are processed directly on GPU VRAM using PyTorch Tensors to ensure simulation speeds of tens of thousands of FPS.
4. **Deployment Pipeline:** In real-world deployment, D455 camera data is processed through an AI vision network (e.g., YOLO) to generate a context vector (3D target coordinates). This vector, combined with Point Cloud data, is fed into an MLP (ONNX model) to generate control signals.

## 🚀 Training Methodology: Curriculum Learning

### 1. UAV Tracking Task

The system utilizes an MLP network (3 hidden layers x 128 units, ELU activation) through a 4-stage roadmap:

| Stage | Title | Objective | Difficulty |
| --- | --- | --- | --- |
| **Stage 1** | **Hover** | Maintain stable altitude at a fixed coordinate. | Very Easy |
| **Stage 2** | **Track Static** | Approach and maintain distance from a stationary target. | Easy |
| **Stage 3** | **Track Moving** | Pursue a target moving with Random Walk behavior. | Medium |
| **Stage 4** | **Domain Rand.** | Add environmental noise (Fog, Wind, Lighting). | Expert |

### 2. UAV Navigation Task

A 2-stage roadmap for autonomous navigation:

| Stage | Title | Objective | Difficulty |
| --- | --- | --- | --- |
| **Stage 1** | **Hover** | Maintain stable altitude at a fixed coordinate. | Very Easy |
| **Stage 2** | **Navigate** | Fly autonomously to a designated target point. | Easy |

## ⚙️ Technical Configuration

### 1. Iris Robot & Motor Prims

* **Model:** Iris (Pegasus Sim Assets). Estimated weight: 1.2kg - 1.5kg.
* **Control Mechanisms:**
* *Low-level:* Direct control of PWM/Thrust `[-1, 1]` for 4 rotors.
* *High-level:* Outputting Roll, Pitch, Yaw, and Thrust commands for PID controllers (Pixhawk/PX4).


* **Control Prims:** `/Root/World/iris/rotor0` to `rotor3`.

### 2. Observation Space

* **Ground-Truth (Simulation):** Direct absolute `[x, y, z]` coordinates for fast convergence.
* **RayCaster 1D Vector:** Simulates LiDAR/Depth sensors by returning a 1D array (e.g., 108 dimensions) measuring obstacle distances.
* **Altitude/Optical Flow:** Provides absolute Z-axis data to compensate for IMU drift.

### 3. Sensors & ROS 2 Bridge (Deploy/Visualize only)

* **Hesai PandarXT-32 SD10:** Topic `/point_cloud`.
* **Intel RealSense D455:** Topics `/rgb` and `/imu`.

## 💻 Getting Started

**Training the models:**

```bash
# Tracking Task
./isaaclab.sh -p uav/source_v2/scripts/train.py --task UAV-Tracking-v2 --num_envs 2048

# Navigation Task
./isaaclab.sh -p uav/source_v2/scripts/train.py --task UAV-Navigation-v2 --num_envs 2048

# Landing Task
./isaaclab.sh -p uav/source_v2/scripts/train.py --task UAV-Landing-v2 --num_envs 2048

```

**Testing the trained model (Play):**

```bash
./isaaclab.sh -p scripts/play.py --task tracking_object_task

```
