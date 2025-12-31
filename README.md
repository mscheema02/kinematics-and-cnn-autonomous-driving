# Kinematics + CNN Road-Following (ROS2 + Gazebo)

[![Watch the demo](https://img.youtube.com/vi/WNtQVkMYbj0/hqdefault.jpg)](https://www.youtube.com/watch?v=WNtQVkMYbj0)

**Demo video:** https://www.youtube.com/watch?v=WNtQVkMYbj0

This project contains **two parts**:

1. **Kinematics (Part 1):** analytical work deriving homogeneous transformation matrices for a planar robot setup. I used the **Kinova Gen3 Lite**, a 6 degree of freedom robotic arm availabe at Lassonde School of Engineering to test movements.
2. **Road-following robot (Part 2):** a **vision-based lane/road follower** in **ROS2 + Gazebo**, trained with a small **CNN (LeNet-style)** on labeled camera images (**left / forward / right**) and then deployed to drive the robot autonomously.

---

## Table of Contents

- [What this project does](#what-this-project-does)
- [Approach](#approach)
  - [Part 1 — Kinematics](#part-1--kinematics)
  - [Part 2 — Road World + CNN Road Follower](#part-2--road-world--cnn-road-follower)
- [Technologies](#technologies)
- [How to run](#how-to-run)
  - [1) Setup](#1-setup)
  - [2) Build the ROS2 workspace](#2-build-the-ros2-workspace)
  - [3) Collect training images (manual driving)](#3-collect-training-images-manual-driving)
  - [4) Train the CNN model](#4-train-the-cnn-model)
  - [5) Quick offline evaluation](#5-quick-offline-evaluation)
  - [6) Run autonomous driving in Gazebo](#6-run-autonomous-driving-in-gazebo)
- [Model versions + results summary](#model-versions--results-summary)
- [Key parameters](#key-parameters)
- [Troubleshooting](#troubleshooting)

---

## What this project does

- Builds/runs a **Gazebo road world** and spawns a robot with a forward-facing camera.
- Captures images while **manually driving** and writes them to labeled folders:
  - `left/`, `forward/`, `right/`
- Trains a CNN that classifies an image into one of the 3 driving labels.
- Deploys that trained model inside a ROS2 node that:
  - Subscribes to the camera image topic
  - Runs inference on each frame
  - Publishes `Twist` commands on `/cmd_vel` to steer the robot

---

## Approach

### Part 1 — Kinematics
- Uses **homogeneous transforms** (rotation + translation) to derive transformation matrices for planar motion.
- Results are documented in the report PDF for Project 2.  
  *(No runtime component required for Part 1.)*

### Part 2 — Road World + CNN Road Follower

#### 1) Data collection
- A ROS2 node captures camera frames and saves them under a chosen `output/` directory.
- Each frame is labeled based on the current manual driving command:
  - `left`, `forward`, or `right`

#### 2) Model training
- Images are loaded from the dataset folders and preprocessed:
  - resized to **28×28**
  - converted to arrays
  - normalized to **[0, 1]**
- Labels are encoded into 3 classes and split into train/test sets.
- A **LeNet-style CNN** is trained and saved as:
  - `road-follower.keras`

Typical training settings used:
- Epochs: **100**
- Batch size: **32**
- Optimizer: **Adam**, learning rate **1e-3**
- Output: **3-way softmax** (`left`, `forward`, `right`)

#### 3) Deployment (autonomous driving)
- A ROS2 node loads `road-follower.keras`.
- For each incoming camera frame:
  - run prediction → choose label id
  - convert label into a steering command:
    - `forward`: positive `x_vel`, zero `theta`
    - `left/right`: positive `x_vel` with ±`theta_vel`
  - publish `geometry_msgs/Twist` to `/cmd_vel`

#### 4) Generalization testing
- The Gazebo environment was modified with additional objects (e.g., a vehicle, ArUco marker) to test how well the model generalizes beyond the training distribution.

---

## Technologies

- **VMWare Workstation Pro with Ubuntu 22.04** (running on a **Windows PC**)
- **ROS2 with Gazebo for Simulations**
- **Python 3**
- **TensorFlow / Keras** (CNN training + inference)
- **OpenCV (cv2)** + **cv_bridge** (ROS image conversion and preprocessing)
- **Inverse Kinematics**
- Common Python utilities used in the pipeline:
  - NumPy
  - scikit-learn (train/test split, one-hot encoding)
  - imutils (image path handling)

---

## How to run

> Notes:
> - Commands below assume you already have a ROS2 workspace that contains this project’s packages/scripts.
> - Package/launch/script names may vary depending on how you organized your repo — the report references `cpmr_ch6` for launch files and nodes. If your package name differs, replace it accordingly.

### 1) Setup
On Ubuntu 22.04 inside VMware:
- Install ROS2 (commonly **Humble** for Ubuntu 22.04)
- Install Gazebo + ROS2 Gazebo integration packages
- Ensure `cv_bridge` is installed for ROS2 image conversion
- Install Python ML dependencies (TensorFlow/Keras, OpenCV, etc.)

### 2) Build the ROS2 workspace
From your ROS2 workspace root:

```bash
source /opt/ros/humble/setup.bash
colcon build --symlink-install
source install/setup.bash
```

### 3) Collect training images (manual driving)

Launch the Gazebo road world + the data-collection node:

```bash
ros2 launch cpmr_ch6 drive_by_road.launch.py
```

This mode is used to generate labeled images into an output directory such as:

```
output/
  left/
  forward/
  right/
```

If your training scripts expect `trainImages/`, copy or rename:

```bash
cp -r output trainImages
```

> Tip: If you’re unsure about keybinds for manual driving / labeling, open the `drive_by_road.py` script and check how key codes map to left/forward/right.

### 4) Train the CNN model

From the folder containing your training script(s):

```bash
python3 road-follower.py
# or
python3 road-follower-v3.py
```

Expected output artifact:

```
road-follower.keras
```

### 5) Quick offline evaluation

To sanity-check predictions against your dataset folders:

```bash
python3 road-follower-test.py
```

This loads `road-follower.keras`, runs predictions on dataset images, and reports mismatches (when the predicted label does not match the folder name).

### 6) Run autonomous driving in Gazebo

Launch the autonomous-driving setup:

```bash
ros2 launch cpmr_ch6 auto_drive_by_road.launch.py
```

The autonomous node loads the model (default `road-follower.keras`) and publishes `/cmd_vel` based on camera inference.

If you need to explicitly set the model path (recommended), run the node directly with parameters:

```bash
ros2 run cpmr_ch6 auto_drive_by_road --ros-args \
  -p model:=/absolute/path/to/road-follower.keras \
  -p x_vel:=0.2 \
  -p theta_vel:=0.2 \
  -p image_size:=28
```

---

## Model versions + results summary

The report compares three model variants:

- **Baseline (original / pre-implemented)**  
  Works in the clean environment but is more sensitive to novel objects and can veer off course.

- **v2 (deeper CNN)**  
  Changes included:
  - increased conv layers (2 → 4)
  - filters: 32, 64, 128, 256
  - kernel size: (5×5 → 3×3)
  - added an extra dense layer  
  Overall: worse than baseline in this setup.

- **v3 (dropout + loss update)** — **best overall**  
  Changes included:
  - dropout(0.25) after pooling layers
  - dropout(0.5) after dense layer
  - categorical cross-entropy loss for 3-class classification  
  Overall: most robust line following and best recovery behavior.

---

## Key parameters

These are commonly used parameters in the ROS2 driving nodes (based on the report’s code):

- `image` (string): camera topic (e.g., `/mycamera/image_raw`)
- `cmd` (string): command topic (typically `/cmd_vel`)
- `model` (string): path/name of the `.keras` model
- `x_vel` (float): forward speed
- `theta_vel` (float): turning speed
- `image_size` (int): input size (28)

---

## Troubleshooting

- **Model loads but robot doesn’t move**
  - Confirm the node is publishing on the same `/cmd_vel` topic your robot listens to.
  - Verify the camera topic name matches what the node subscribes to.

- **Robot oscillates / “jitters”**
  - Reduce `theta_vel` slightly, or reduce `x_vel`.
  - Ensure lighting/textures in Gazebo are not drastically different from training images.

- **Bad generalization in modified worlds**
  - Collect additional training images in the modified environment.
  - Prefer the v3 model (dropout) and consider increasing dataset diversity.

- **cv_bridge errors**
  - Ensure the correct ROS2 `cv_bridge` package is installed for your ROS distribution.
  - Confirm image encoding matches expected (commonly `"bgr8"`).

---
