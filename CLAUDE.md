# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

XLeRobot is an open-source, low-cost (~$660) dual-arm mobile household robot platform built on [LeRobot](https://github.com/huggingface/lerobot). It supports multiple base variants (3-omniwheel, 2-wheel differential, 4-wheel mecanum), various teleoperation modes (keyboard, Xbox controller, Nintendo Switch Joy-Con, VR headset), simulation (MuJoCo, ManiSkill), and web-based remote control.

## Architecture

The repo is organized into six top-level subsystems:

- **`software/`** — Core robot control stack. Built as an extension of LeRobot's robot/teleoperator/dataset framework.
- **`hardware/`** — CAD/STL files, BOM, and assembly guides for the physical robot.
- **`simulation/`** — MuJoCo and ManiSkill simulation environments with teleoperation and RL support.
- **`XLeVR/`** — VR teleoperation using WebSockets + HTTPS (Meta Quest 3 browser-based). Based on telegrip.
- **`web_control/`** — FastAPI + Socket.IO backend and React/TypeScript/Vite frontend for remote robot control.
- **`docs/`** — Sphinx-based documentation source, published at xlerobot.readthedocs.io.

### Robot Class Hierarchy

All robot classes live under `software/src/robots/` and extend LeRobot's `Robot` base. Each variant has:

- A **main robot class** (`xlerobot.py`) — direct hardware control over serial buses (Feetech STS3215 servos)
- A **host class** (`*_host.py`) — receives commands via ZMQ (runs on the robot's onboard computer)
- A **client class** (`*_client.py`) — sends commands via ZMQ (runs on a remote laptop)
- A **config dataclass** — registered with LeRobot's config system via `@RobotConfig.register_subclass`

The three robot variants differ only in their mobile base kinematics:

| Variant | Robot name | `software/src/robots/` | Base kinematics |
|---|---|---|---|
| 3-omniwheel | `xlerobot` | `xlerobot/` | Omnidirectional (x, y, theta) |
| 2-wheel diff | `xlerobot_2wheels` | `xlerobot_2wheels/` | Differential drive (x, theta only) |
| 4-wheel mecanum | `xlerobot` | `xlerobot_mecanum/` | Mecanum X-layout (x, y, theta) |

Each robot uses two `FeetechMotorsBus` serial buses (`/dev/ttyACM0` and `/dev/ttyACM1` by default). Bus1 handles the left arm (6 joints) + head (2 motors). Bus2 handles the right arm (6 joints) + base wheels. Arms use position control; base wheels use velocity control.

### Key LeRobot Integration Points

This repo **requires** the `lerobot` Python package to be installed (not vendored). It plugs into LeRobot's framework at these extension points:
- Custom `Robot` subclass with registration via `RobotConfig.register_subclass`
- Custom `Teleoperator` subclass for VR teleop (`software/src/teleporators/xlerobot_vr/`)
- Dataset recording via LeRobot's `lerobot-record` CLI, using `software/src/record.py`
- Camera configs supporting OpenCV USB cameras and Intel RealSense RGB-D cameras

### Simulation Layer (`simulation/`)

- **`mujoco/`** — Standalone MuJoCo simulation with keyboard control. Uses `mujoco_viewer` for rendering. Single file (`xlerobot_mujoco.py`) containing the `XLeRobotController` class.
- **`Maniskill/`** — ManiSkill-based RL-ready simulation. Includes:
  - `agents/xlerobot/` — Agent definitions registered with ManiSkill's agent system
  - `envs/scenes/` — Custom scene environments
  - `examples/` — Demo scripts for EE control with keyboard, Xbox, and VR
  - `run_xlerobot_sim.py` / `run_xlerobot_sim_host.py` — Entry points for local and host-mode simulation

### VR Teleoperation (`XLeVR/`)

VR teleop uses a browser-based WebSocket approach for Meta Quest 3 (no app install needed). Architecture:
- `XLeVR/xlevr/inputs/vr_ws_server.py` — WebSocket server that receives VR controller data
- `XLeVR/xlevr/config.py` — Configuration (ports, SSL, robot ports) loaded from YAML
- `XLeVR/vr_monitor.py` — Python client that reads VR goal poses for controlling the real robot
- Data flows: VR Headset Browser → WebSocket → VR Monitor → Robot via LeRobot teleop framework

### Web Control (`web_control/`)

Two-tier client-server architecture:
- **Backend** (`web_control/server/`) — FastAPI + python-socketio. `main.py` is the entry point. `core/remote_core.py` abstracts communication with different robot hosts (real robot, MuJoCo sim, ManiSkill sim) over ZMQ. Includes rate limiting and throttling per client.
- **Frontend** (`web_control/client/`) — React + TypeScript + Vite + Tailwind CSS. Key components: `RobotVideoCanvas.tsx` (video streaming display), `BottomControlConsole.tsx` (movement controls), `MultiTabPanel.tsx` (UI layout). Uses Socket.IO for real-time communication.

### Joy-Con Robotics (`software/joyconrobotics/`)

Library for interfacing with Nintendo Switch Joy-Con controllers over Bluetooth. Core classes: `JoyconRobotics` (main controller interface), `JoyCon` (individual Joy-Con), `Gyro` (IMU data), `Event` (button/analog events). Used by teleop examples in `software/examples/`.

### Vision

- `software/test_yolo.py` and the YOLO-based examples in `software/examples/` use OpenCV + YOLO for real-time object detection and tracking
- The YOLO pipeline enables end-effector control that follows detected objects

## Common Commands

This project has no automated build, lint, or test framework. All verification is done by running examples against real hardware or simulation.

**Documentation build:**
```bash
pip install -e .[docs]
cd docs && make html
```

**Run examples (real robot):**
```bash
cd software/examples
python 4_xlerobot_teleop_keyboard.py      # Keyboard teleop (3-omniwheel)
python 4_xlerobot_2wheels_teleop_keyboard.py  # Keyboard teleop (2-wheel)
python 5_xlerobot_teleop_xbox.py          # Xbox controller teleop
python 7_xlerobot_teleop_joycon.py        # Joy-Con controller teleop
python 8_xlerobot_teleop_vr.py            # VR headset teleop
```

**Simulation:**
```bash
# MuJoCo
python simulation/mujoco/xlerobot_mujoco.py

# ManiSkill
python simulation/Maniskill/run_xlerobot_sim.py
```

**VR monitor:**
```bash
cd XLeVR && python vr_monitor.py
```

**Web control:**
```bash
# Server
cd web_control/server
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env   # Edit ROBOT_HOST, ports
python main.py

# Client
cd web_control/client
npm install
cp .env.example .env   # Edit VITE_SERVER_HOST, VITE_SERVER_PORT
npm run dev            # Dev server at http://localhost:5173
```

## Environment & Configuration

- **Python**: 3.9+ (ReadTheDocs uses 3.9; web_control requires 3.11+)
- **Node.js**: 22+ (for web_control client)
- **Hardware**: Feetech STS3215 servos (2 buses via USB serial), optional Intel RealSense RGB-D cameras, Raspberry Pi for onboard compute
- **Robot configurations**: Dataclass-based, registered via `@RobotConfig.register_subclass("name")`. Serial ports default to `/dev/ttyACM0` and `/dev/ttyACM1`.
- **Calibration**: Stored per-robot in `~/.cache/lerobot/calibration/<robot-type>/`. Handled interactively at first connect or restored from file.
- **Cameras**: Configured through LeRobot's `CameraConfig` system. Supports OpenCV USB cameras and RealSense RGB-D cameras. Camera configs are empty by default; uncomment in the config to enable.

## Key Dependencies

- `lerobot` — HuggingFace's robot learning framework (provides `Robot`, `Teleoperator`, `FeetechMotorsBus`, dataset recording, etc.)
- `mujoco` + `mujoco_viewer` — Physics simulation
- `mani_skill` + `gymnasium` + `sapien` — RL simulation environment
- `fastapi` + `uvicorn` + `python-socketio` — Web control backend
- `joyconrobotics` — Custom Joy-Con interface library (included in repo)
- `opencv-python` + `ultralytics` (YOLO) — Computer vision
- `numpy`, `torch` — Numerical computing
