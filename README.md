# GT-Arc
Cobot
# 🤖 Cobot DM CAN Controller

> 6-DOF collaborative robot controller for **DM4310 / DM4340 CAN motors**  
> AR4-style Python GUI · ROS2 hardware interface · Web controller · Collision detection

[![ROS2](https://img.shields.io/badge/ROS2-Humble%20%7C%20Jazzy-blue)](https://docs.ros.org)
[![Python](https://img.shields.io/badge/Python-3.10%2B-blue)](https://python.org)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)
[![CAN](https://img.shields.io/badge/Protocol-MIT%20CAN%20Mode-orange)](https://github.com/gridar27/cobot-dm-can-controller)

---

## 📋 Table of Contents

- [Overview](#overview)
- [Hardware](#hardware)
- [Wiring Diagram](#wiring-diagram)
- [Repository Structure](#repository-structure)
- [Quick Start](#quick-start)
- [AR4 Python GUI](#ar4-python-gui)
- [ROS2 Hardware Interface](#ros2-hardware-interface)
- [Web Controller](#web-controller)
- [CAN MIT Mode Protocol](#can-mit-mode-protocol)
- [Motor Tuning Guide](#motor-tuning-guide)
- [Collision Detection](#collision-detection)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)

---

## Overview

This project replaces the original **Annin Robotics AR4** Teensy-based serial communication with direct **CAN MIT mode** commands to DM-series brushless motors. It provides three control interfaces:

| Interface | File | Description |
|---|---|---|
| Python GUI | `ar4_ui/AR4.py` | Modified AR4 HMI — joint jog, teach & playback, collision monitor |
| ROS2 | `ros2_control/` | `ros2_control` hardware interface — works with MoveIt2 |
| Web | `web/cobot_controller.html` | Browser-based controller via rosbridge WebSocket |

---

## Hardware

### Bill of Materials

| Component | Qty | Model | Notes |
|---|---|---|---|
| Base / Shoulder / Elbow motors | 3 | **DM4340** | High torque, 100W |
| Forearm / Wrist / Gripper motors | 4 | **DM4310** | Compact, 50W |
| CAN interface | 1 | USB-CAN adapter or PCIe CAN card | e.g. CANable, PCAN-USB |
| Controller PC | 1 | Any Linux PC | Ubuntu 22.04 recommended |
| 24V PSU | 1 | 10A minimum | Powers all motors |
| CAN bus termination resistors | 2 | 120Ω | One at each end of bus |

### Motor Specifications

| Spec | DM4340 | DM4310 |
|---|---|---|
| Rated Power | 100W | 50W |
| Peak Torque | 18 Nm | 10 Nm |
| Max Speed | 300 RPM | 400 RPM |
| Bus Voltage | 24V DC | 24V DC |
| Protocol | CAN 2.0A (MIT mode) | CAN 2.0A (MIT mode) |
| CAN Bitrate | 1 Mbps | 1 Mbps |

---

## Wiring Diagram

```
                        24V DC Power Supply
                        +──────────────────+
                        │                  │
                       (+)                (-)
                        │                  │
         ┌──────────────┴──────────────────┴──────────────┐
         │                  Power Bus                      │
         └──┬──────────┬──────────┬──────────┬────────────┘
            │          │          │          │
          [M1]        [M2]       [M3]       [M4..M7]
         DM4340      DM4340     DM4340     DM4310
         Base        Shoulder   Elbow      Forearm/Wrists/Gripper
         ID=1        ID=2       ID=3       ID=4,5,6,7

CAN Bus Wiring (Daisy Chain):

  PC/USB-CAN ──── [120Ω] ──── M1 ──── M2 ──── M3 ──── M4 ──── M5 ──── M6 ──── M7 ──── [120Ω]
                 CANH (yellow)  ──────────────────────────────────────────────────────
                 CANL (green)   ──────────────────────────────────────────────────────
                 GND  (black)   ──────────────────────────────────────────────────────

Motor CAN Connector Pinout (DM4310 / DM4340):
  Pin 1 → CANH
  Pin 2 → CANL
  Pin 3 → GND (signal ground, connect to PC GND)
  Pin 4 → +24V
  Pin 5 → GND (power)

⚠️  IMPORTANT:
  - Use twisted pair wire for CAN (CANH + CANL)
  - Keep CAN cable away from motor power cables
  - Both ends of the bus MUST have 120Ω termination resistors
  - Connect signal GND between all motors and PC
```

### Motor CAN ID Assignment

Each motor must have a unique CAN ID. Set via the motor's configuration tool before use:

| Joint | Label | Motor | CAN ID | Profile |
|---|---|---|---|---|
| joint_1 | Base | DM4340 | **1** | Kp=30, Kd=4.5 |
| joint_2 | Shoulder | DM4340 | **2** | Kp=30, Kd=4.5 |
| joint_3 | Elbow | DM4340 | **3** | Kp=30, Kd=4.5 |
| joint_4 | Forearm | DM4310 | **4** | Kp=10, Kd=1.5 |
| joint_5 | Wrist 1 | DM4310 | **5** | Kp=10, Kd=1.5 |
| joint_6 | Wrist 2 | DM4310 | **6** | Kp=10, Kd=1.5 |
| finger_left_joint | Gripper | DM4310 | **7** | Kp=8, Kd=1.2 |

---

## Repository Structure

```
cobot-dm-can-controller/
│
├── ar4_ui/                          # Python GUI (AR4 HMI modified)
│   ├── AR4.py                       # Main application entry point
│   ├── dm_can_comms.py              # CAN comms (replaces AR4 serial)
│   └── requirements.txt
│
├── ros2_control/                    # ROS2 hardware interface package
│   ├── dm_can_hardware_interface.cpp
│   ├── dm_can_hardware_interface.hpp
│   ├── dm_can_hardware_interface.xml
│   ├── CMakeLists.txt
│   ├── controllers.yaml
│   ├── ros2_control.urdf.xacro
│   ├── dm_can_bringup.launch.py
│   └── real_robot_driver_fixed.py   # Standalone ROS2 driver node
│
├── web/                             # Web-based controller
│   ├── cobot_controller.html        # Full controller (rosbridge)
│   ├── cobot_dashboard.html         # Dashboard with torque monitor
│   └── cobot_joint_control.html     # Joint control only
│
├── docs/
│   └── README.md                    # This file
│
└── LICENSE
```

---

## Quick Start

### 1. CAN Interface Setup

```bash
# Bring up CAN interface at 1 Mbps
sudo ip link set can0 up type can bitrate 1000000

# Verify it's up
ip link show can0

# Monitor CAN traffic (useful for debugging)
candump can0

# Set CAN interface persistent across reboots (optional)
sudo nano /etc/network/interfaces
# Add:
# auto can0
# iface can0 inet manual
#   pre-up /sbin/ip link set can0 up type can bitrate 1000000
#   down /sbin/ip link set can0 down
```

### 2. Verify Motor Communication

```bash
# Should see frames from all 7 motors after enabling
candump can0

# Quick enable test
python3 -c "
import can, time
bus = can.interface.Bus(channel='can0', bustype='socketcan')
enable = [0xFF,0xFF,0xFF,0xFF,0xFF,0xFF,0xFF,0xFC]
for mid in range(1, 8):
    bus.send(can.Message(arbitration_id=mid, data=enable, is_extended_id=False))
    time.sleep(0.1)
    print(f'Motor {mid} enabled')
bus.shutdown()
"
```

### 3. Run the Python GUI

```bash
cd ar4_ui
pip install -r requirements.txt
python AR4.py
```

1. In the **Settings** tab → select `can0` → click **Connect & Enable Motors**
2. Switch to **Joint Control** tab → use sliders or jog buttons
3. Switch to **Teach & Playback** → save waypoints and run programs

---

## AR4 Python GUI

The GUI is built with Python + Tkinter. It has 4 tabs:

### Joint Control Tab
- **Slider per joint** — drag to set position
- **Jog buttons** `«  −  +  »` with selectable step size (0.5° / 1° / 5° / 10°)
- **ZERO button** per joint — returns to 0 rad
- **Live torque bar** per joint — color coded green → amber → red
- Live readout in both **degrees** and **radians**

### Teach & Playback Tab
- Name and **save waypoints** at current position
- **Auto-record mode** — captures a waypoint every 2 seconds while active
- **Playback** with adjustable speed (10% – 200%)
- **Save/Load programs** as JSON files
- Double-click any waypoint to jump to it

### Collision Monitor Tab
- Live torque bar for all 7 motors
- **Per-joint threshold** (Nm) — editable spinbox
- **E-STOP button** — immediate safe stop
- **Event log** with timestamps

### Settings Tab
- CAN interface selector
- Connect / Disconnect buttons
- Tuning tips and CAN setup commands

---

## ROS2 Hardware Interface

### Prerequisites

```bash
# ROS2 Humble or Jazzy
sudo apt install ros-$ROS_DISTRO-ros2-control
sudo apt install ros-$ROS_DISTRO-ros2-controllers
sudo apt install ros-$ROS_DISTRO-moveit
sudo apt install ros-$ROS_DISTRO-rosbridge-suite
```

### Build

```bash
cd ~/ros2_ws/src
git clone https://github.com/gridar27/cobot-dm-can-controller.git

# Copy ros2_control package
cp -r cobot-dm-can-controller/ros2_control dm_can_hardware_interface

cd ~/ros2_ws
colcon build --packages-select dm_can_hardware_interface
source install/setup.bash
```

### Launch

```bash
# Bring up CAN first
sudo ip link set can0 up type can bitrate 1000000

# Full bringup (MoveIt + rosbridge + hardware interface)
ros2 launch dm_can_hardware_interface dm_can_bringup.launch.py

# Or use the standalone Python driver (no MoveIt needed)
python3 ros2_control/real_robot_driver_fixed.py
```

### Architecture

```
MoveIt2
  └─► JointTrajectoryController  (ros2_control)
        └─► DmCanHardwareInterface  (this package)
              └─► SocketCAN (can0)
                    └─► Motors 1–7 (MIT mode)
```

### URDF Integration

Add to your robot URDF:

```xml
<xacro:include filename="$(find dm_can_hardware_interface)/urdf/ros2_control.urdf.xacro"/>
```

---

## Web Controller

The web controller connects to your robot via **rosbridge WebSocket**.

### Setup

```bash
# Start rosbridge
ros2 launch rosbridge_server rosbridge_websocket_launch.xml

# Serve the web files
cd web
python3 -m http.server 8080

# Open in browser
# http://localhost:8080/cobot_controller.html
```

### Features

- **Joint Control** — sliders + jog buttons for all 7 joints
- **Gripper** — dedicated open/close panel with visual feedback
- **Teach & Playback** — waypoint recording and execution
- **Collision Monitor** — live torque display with threshold controls
- **Auto-reconnect** — reconnects to rosbridge automatically on drop

---

## CAN MIT Mode Protocol

The DM motors use MIT's open-source motor controller protocol.

### Command Frame (PC → Motor, 8 bytes)

```
Byte 0-1:  Position  (uint16, range ±12.5 rad)
Byte 2-3:  Velocity  (uint12, range ±30 rad/s)
Byte 3-4:  Kp        (uint12, range 0–500)
Byte 5-6:  Kd        (uint12, range 0–5)
Byte 6-7:  Torque FF (uint12, range ±10 Nm)
```

### Feedback Frame (Motor → PC, 8 bytes)

```
Byte 0:    Motor ID
Byte 1-2:  Position  (uint16)
Byte 3-4:  Velocity  (uint12)
Byte 4-6:  Torque    (uint12)
```

### Special Commands

| Command | Bytes |
|---|---|
| Enable motor | `FF FF FF FF FF FF FF FC` |
| Disable motor | `FF FF FF FF FF FF FF FD` |
| Set zero position | `FF FF FF FF FF FF FF FE` |

### Float ↔ Uint Conversion

```python
def float_to_uint(x, x_min, x_max, bits):
    return int((x - x_min) / (x_max - x_min) * ((1 << bits) - 1))

def uint_to_float(x_int, x_min, x_max, bits):
    return x_int * (x_max - x_min) / ((1 << bits) - 1) + x_min
```

---

## Motor Tuning Guide

The MIT mode PD controller applies torque as:

```
τ = Kp × (target_pos − actual_pos) + Kd × (0 − actual_vel)
```

### Fixing Springback / Oscillation

The most common issue — motor oscillates back and forth around the target.

**Cause:** Kd too low relative to Kp (underdamped system)

**Rule of thumb:** `Kd ≈ 10–15% of Kp` for critical damping

```
# Current tuned values (anti-springback):
DM4340 (Base/Shoulder/Elbow):  Kp=30.0, Kd=4.5  (Kd = 15% of Kp)
DM4310 (Forearm/Wrist):        Kp=10.0, Kd=1.5  (Kd = 15% of Kp)
DM4310 (Gripper):              Kp= 8.0, Kd=1.2  (Kd = 15% of Kp)
```

**Tuning procedure:**

| Symptom | Fix |
|---|---|
| Oscillates back and forth | Increase Kd by 20% |
| Arm droops under gravity | Increase Kp by 5.0 steps |
| Jerky / twitchy motion | Decrease Kp by 20% |
| Slow to reach target | Increase Kp by 10% |
| Overshoots and settles slowly | Increase Kd by 10% |

Edit `MOTOR_CONFIGS` in `ar4_ui/dm_can_comms.py`:

```python
MOTOR_CONFIGS: List[MotorConfig] = [
    MotorConfig(1, "joint_1", "Base",    kp=30.0, kd=4.5),  # ← edit here
    ...
]
```

### Gripper Screw Pitch

The gripper motor converts linear position (meters) to rotary (radians).
You must set the correct lead screw pitch:

```python
# In dm_can_comms.py:
SCREW_PITCH_MM = 4.0   # ← measure your actual lead screw pitch in mm
```

To find your pitch:
1. Mark the screw and count revolutions for 10mm of travel
2. `pitch = 10mm / revolutions`

### Startup Zero-Snap Prevention

The driver reads actual motor positions on startup before sending any commands, preventing the arm from jumping to 0 rad when the node starts. This is handled by the warmup loop in `enable_motors()`.

---

## Collision Detection

Collision detection works by monitoring motor torque feedback against configurable thresholds.

### How It Works

```
Every control cycle:
  read torque from CAN feedback frame
  if |torque| > threshold[joint]:
      trigger emergency_stop()
      fire collision_callbacks
```

### Emergency Stop Response

On collision, the driver switches to **pure damping mode**:
```
Kp = 0      (no position stiffness — won't fight the collision)
Kd = 3.0    (high damping — absorbs energy smoothly)
```

This is safer than a hard stop — the arm yields to the force instead of rigidly resisting.

### Default Thresholds

```python
# Set 30–40% above normal operating torque
joint_1 (Base):     8.0 Nm
joint_2 (Shoulder): 7.0 Nm
joint_3 (Elbow):    5.0 Nm
joint_4 (Forearm):  3.0 Nm
joint_5 (Wrist 1):  2.5 Nm
joint_6 (Wrist 2):  2.0 Nm
Gripper:            1.0 Nm
```

### Calibrating Thresholds

1. Run the robot in free air with no payload
2. Watch the torque bars in the Collision Monitor tab
3. Note the **peak torque** per joint during normal motion
4. Set threshold = `peak_torque × 1.35` (35% margin)
5. Test with a slow deliberate soft collision to verify

---

## Troubleshooting

### CAN Interface Not Found

```
❌ 'can0' not found
```

```bash
# Bring up the interface
sudo ip link set can0 up type can bitrate 1000000

# If using USB-CAN adapter, check it's detected
lsusb
dmesg | grep can
```

### Motors Not Responding

```bash
# Check CAN traffic — should see frames when motors are powered
candump can0

# No frames = wiring issue
# Check: power, termination resistors, CAN H/L polarity
```

### Springback on Startup

The arm jumps to a position when the driver starts.

**Fix:** The warmup gate in `enable_motors()` waits for real feedback before sending commands. If it still jumps:

```python
# Increase warmup time in dm_can_comms.py
time.sleep(0.5)   # was 0.3 — give more time for feedback frames
```

### rosbridge Connection Refused

```bash
# Check rosbridge is running
ros2 node list | grep rosbridge

# Start it manually
ros2 launch rosbridge_server rosbridge_websocket_launch.xml

# Check port
ss -tlnp | grep 9090

# Firewall
sudo ufw allow 9090
```

### Gripper Not Moving

The gripper requires correct screw pitch calibration:

```bash
# Add debug logging in dm_can_comms.py
# In _feedback_loop, add:
logger.info(f"Gripper raw pos: {pos:.4f} m = {pos*1000:.1f} mm")
```

If the value looks wrong, adjust `SCREW_PITCH_MM`.

### High CPU Usage

The 100 Hz control loop can be demanding on slow systems:

```python
# Reduce to 50 Hz in dm_can_comms.py
interval = 0.02   # was 0.01
```

---

## Contributing

Pull requests welcome. Please open an issue first for major changes.

**Areas that need work:**
- Forward/inverse kinematics integration
- Trajectory smoothing (currently step-to-step)
- Gravity compensation torque feedforward
- URDF for the specific arm geometry

---

## License

MIT License — see [LICENSE](LICENSE)

---

## Author

**gridar27** — [gridar27@gmail.com](mailto:gridar27@gmail.com)

Built on top of the [Annin Robotics AR4](https://github.com/Chris-Annin/Annin-Robot-Project) open-source project by Chris Annin.
