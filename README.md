# Kinematics + CNN Road-Following (ROS2 + Gazebo)

> [![Watch the demo](https://img.youtube.com/vi/WNtQVkMYbj0/hqdefault.jpg)](https://www.youtube.com/watch?v=WNtQVkMYbj0)
>
>**Demo video:** https://www.youtube.com/watch?v=WNtQVkMYbj0

---

## Overview

This project has **two parts**:

1. **Kinematics (Part 1):** derive and reason about **homogeneous transformation matrices** for a planar robot setup (analysis-focused).
2. **Road-following robot (Part 2):** build a **vision-based** road follower in **ROS2 + Gazebo** by training a small **CNN (LeNet-style)** on labeled camera images (**left / forward / right**), then deploying it as a ROS2 node that publishes `/cmd_vel`.

---

## Key highlights

- Built a full **data → training → deployment** pipeline: Gazebo simulation → image dataset → CNN training → real-time inference ROS2 node.
- Implemented a 3-class **behavior classifier** (`left`, `forward`, `right`) and mapped predictions to **Twist** commands for autonomous control.
- Evaluated robustness by modifying the simulation environment and observing generalization behavior (dataset shift).

---

## Table of Contents

- [Overview](#overview)
- [Key highlights (resume-friendly)](#key-highlights-resume-friendly)
- [What this project does](#what-this-project-does)
- [Approach](#approach)
  - [Part 1 — Kinematics](#part-1--kinematics)
  - [Part 2 — Road World + CNN Road Follower](#part-2--road-world--cnn-road-follower)
- [Results summary](#results-summary)
- [Technologies](#technologies)
- [How to run](#how-to-run)
  - [1) Setup](#1-setup)
  - [2) Build the ROS2 workspace](#2-build-the-ros2-workspace)
  - [3) Collect training images (manual driving)](#3-collect-training-images-manual-driving)
  - [4) Train the CNN model](#4-train-the-cnn-model)
  - [5) Quick offline evaluation](#5-quick-offline-evaluation)
  - [6) Run autonomous driving in Gazebo](#6-run-autonomous-driving-in-gazebo)
- [Learnings](#learnings)
- [Resume bullets (copy/paste)](#resume-bullets-copypaste)
- [Key parameters](#key-parameters)
- [Troubleshooting](#troubleshooting)

---

## What this project does

- Launches a **Gazebo road world** and spawns a robot with a forward-facing camera.
- Captures images while **manually driving**, storing them into labeled folders:
  - `left/`, `forward/`, `right/`
- Trains a CNN that classifies an image into one of the 3 driving labels.
- Deploys the trained model inside a ROS2 node that:
  - subscribes to the camera image topic
  - runs inference on each frame
  - publishes `geometry_msgs/Twist` on `/cmd_vel` to steer the robot

---

## Approach

### Part 1 — Kinematics
- Uses **homogeneous transforms** (rotation + translation) to derive transformation matrices for planar motion.
- Results are documented in the Project 2 report.  
  *(No runtime component required for Part 1.)*

### Part 2 — Road World + CNN Road Follower

#### 1) Data collection (ROS2)
- A ROS2 node captures camera frames and saves them under a chosen dataset directory.
- Each frame is labeled based on the current manual driving command:
  - `left`, `forward`, or `right`

Dataset output structure:
```
trainImages/
  left/
  forward/
  right/
```

#### 2) Model training (Python + Keras)
- Images are loaded from the dataset folders and preprocessed:
  - resized to **28×28**
  - normalized to **[0, 1]**
- Labels are encoded into 3 classes and split into train/test sets.
- A **LeNet-style CNN** is trained and saved as:
  - `road-follower.keras`

Typical training settings used in the report:
- Epochs: **100**
- Batch size: **32**
- Optimizer: **Adam**, learning rate **1e-3**
- Output: **3-way softmax** (`left`, `forward`, `right`)

#### 3) Deployment (ROS2 inference → control)
- A ROS2 node loads `road-follower.keras`.
- For each incoming camera frame:
  - preprocess → predict label
  - map label → control:
    - `forward`: +`x_vel`, `theta = 0`
    - `left/right`: +`x_vel` with ±`theta_vel`
  - publish `Twist` on `/cmd_vel`

#### 4) Generalization testing
- The Gazebo world was modified with additional objects (e.g., a vehicle, marker) to test robustness under **visual distribution shift**.

---

## Results summary

Three model variants were compared in the report:

- **Baseline:** worked in the clean environment but is more sensitive to novel objects and can veer off course.
- **v2 (deeper CNN):** increased depth/filters; overall performed worse in this setup.
- **v3 (dropout + categorical cross-entropy):** best overall stability and recovery behavior (most robust line following).

---

## Technologies

- **VMWare Workstation Pro with Ubuntu 22.04** (running on a **Windows PC**)
- **ROS2 with Gazebo for Simulations**
- **Python 3**
- **TensorFlow / Keras** (CNN training + inference)
- **OpenCV (cv2)** + **cv_bridge** (ROS image conversion and preprocessing)
- **Inverse Kinematics**
- Python utilities used in the pipeline:
  - NumPy
  - scikit-learn (train/test split, one-hot encoding)
  - imutils (image path handling)

---

## How to run

> Notes:
> - Commands assume a ROS2 workspace containing this project’s packages/scripts.
> - The report references `cpmr_ch6` for launch files/nodes. If your package name differs, replace it accordingly.

### 1) Setup
On Ubuntu 22.04 inside VMware:
- Install ROS2 (commonly **Humble** for Ubuntu 22.04)
- Install Gazebo + ROS2 Gazebo integration packages
- Ensure `cv_bridge` is installed
- Install Python ML dependencies (TensorFlow/Keras, OpenCV, etc.)

### 2) Build the ROS2 workspace

```bash
source /opt/ros/humble/setup.bash
cd <your_ws>
colcon build --symlink-install
source install/setup.bash
```

### 3) Collect training images (manual driving)

Launch the road world + data collection:

```bash
ros2 launch cpmr_ch6 drive_by_road.launch.py
```

This generates labeled images into an output directory such as:

```
output/
  left/
  forward/
  right/
```

If your training scripts expect `trainImages/`, rename/copy:

```bash
cp -r output trainImages
```

### 4) Train the CNN model

From the folder containing your training script(s):

```bash
python3 road-follower.py
# or
python3 road-follower-v3.py
```

Expected artifact:

```
road-follower.keras
```

### 5) Quick offline evaluation

```bash
python3 road-follower-test.py
```

This loads `road-follower.keras`, predicts labels for dataset images, and reports mismatches.

### 6) Run autonomous driving in Gazebo

```bash
ros2 launch cpmr_ch6 auto_drive_by_road.launch.py
```

If your node supports parameters (recommended), you can run it directly:

```bash
ros2 run cpmr_ch6 auto_drive_by_road --ros-args   -p model:=/absolute/path/to/road-follower.keras   -p x_vel:=0.2   -p theta_vel:=0.2   -p image_size:=28
```

---

## Learnings

- **ROS2 perception pipeline:** subscribing to camera topics, converting ROS images using `cv_bridge`, and applying consistent preprocessing.
- **Control mapping:** translating a discrete classifier output into stable velocity commands (`Twist`) and tuning `x_vel` / `theta_vel`.
- **Dataset quality matters:** class balance, lighting/texture variation, and camera viewpoint changes strongly affect performance.
- **Generalization is hard:** even in simulation, adding new objects can create distribution shift; dropout and better loss choices improved robustness.

---

## Overall

- Built a ROS2 + Gazebo autonomous road-following system by training a 3-class CNN (left/forward/right) and deploying real-time inference to publish `geometry_msgs/Twist` on `/cmd_vel`.
- Implemented an end-to-end ML workflow: automated dataset capture in simulation, image preprocessing (28×28 normalization), CNN training (Keras), and offline validation.
- Improved robustness against simulation changes through iterative model design (dropout + categorical cross-entropy) and environment-based generalization testing.

---

## Key parameters

Common ROS2 driving-node parameters (based on the report):

- `image` (string): camera topic (e.g., `/mycamera/image_raw`)
- `cmd` (string): command topic (typically `/cmd_vel`)
- `model` (string): path/name of the `.keras` model
- `x_vel` (float): forward speed
- `theta_vel` (float): turning speed
- `image_size` (int): input size (28)

---

## Troubleshooting

- **Model loads but robot doesn’t move**
  - Confirm the node publishes to the same `/cmd_vel` your robot listens to.
  - Verify the camera topic matches what the node subscribes to.

- **Robot oscillates / jitters**
  - Reduce `theta_vel` and/or `x_vel`.
  - Increase dataset diversity (lighting, viewpoints) and retrain.

- **Poor performance after adding objects**
  - Collect additional images in the modified environment and retrain.
  - Prefer the v3 model (dropout) for better robustness.

- **cv_bridge errors**
  - Ensure the correct ROS2 `cv_bridge` package is installed for your ROS distribution.
  - Confirm expected image encoding (commonly `bgr8`).
