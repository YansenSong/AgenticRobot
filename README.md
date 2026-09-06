<div align="center">

<img src="docs/assets/holoagent_logo_text.png" alt="HoloAgent Logo" width="420"/>

# AgenticRobot / HoloAgent Integration Repository

[Project Page](https://horizonrobotics.github.io/robot_lab/holoagent/) · [arXiv](https://arxiv.org/abs/2606.23565) · [Dataset](https://huggingface.co/datasets/HorizonRobotics/fsrvln_datasets) · [FSR-VLN](https://horizonrobotics.github.io/robot_lab/fsr-vln)

</div>

> **Upstream attribution**
>
> The HoloAgent framework, FSR-VLN research, papers, datasets, logos, and upstream components are the work of their original authors and organizations. This repository is a code/integration workspace around that robotics stack. When using the research in publications or redistribution, preserve the original project citations, licenses, and third-party notices.

AgenticRobot is a ROS 2 robotics workspace for **natural-language-driven robot skills, navigation, perception, semantic mapping, multi-robot services, and robot-specific control adapters**. The current repository combines robot-independent HoloAgent components with hardware-specific workspaces for platforms such as Unitree and HexFellow.

This is a **real-robot-oriented stack**, not a hardware-free demo. Building the code is only one part of setup: practical deployment also requires the appropriate ROS drivers, sensors, calibration, maps, model weights, GPU environment, network configuration, and robot services.

## What is in this repository

```text
Language / Agent layer
        │
        ▼
AgentOS / Skills / Chatbot
        │
        ▼
Robot services & orchestration
        │
        ├─ navigation
        ├─ perception
        ├─ semantic mapping / FSR-VLN
        ├─ HTTP ↔ ROS bridge
        └─ multi-robot control
        │
        ▼
Robot-specific adapters
        ├─ Unitree
        └─ HexFellow
        │
        ▼
Real robot hardware
```

## Main capabilities

- **Agentic robot execution**: convert high-level language goals into registered robot skills and monitored execution steps.
- **Skill system**: reusable skill documentation, examples, registration, and orchestration under AgentOS.
- **Navigation**: ROS 2 Navigation2-based bringup plus semantic / relative navigation components.
- **Perception**: camera / GPU inference integration and ROS perception nodes.
- **3D spatial / semantic mapping**: mapping and retrieval components used by embodied navigation.
- **FSR-VLN**: fast-and-slow vision-language navigation with hierarchical multimodal scene representations.
- **Robot bridge**: HTTP-to-ROS service layer for robot control and integration.
- **Multi-robot control**: service examples for coordinating multiple robots.
- **Robot-specific workspaces**: Unitree and HexFellow adapters kept separate from robot-independent core code.
- **Container / integration scripts**: helpers for prepared deployment environments.

## Repository layout

```text
AgenticRobot/
├── agentic_robot/
│   ├── agentOS/            # Agent / skill orchestration infrastructure
│   ├── chatbot/            # Speech / language interaction components
│   ├── core/               # Robot-independent ROS 2 workspace
│   ├── services/           # HTTP/ROS bridge and multi-robot services
│   ├── fsr_vln/            # Semantic mapping / VLN components
│   ├── thirdparty/         # Vendored ROS / C++ dependencies
│   ├── tools/              # Mapping and utility tools
│   ├── build.sh            # Layer-aware build helper
│   └── clear.sh            # Build cleanup helper
├── robots/
│   ├── unitree/            # Unitree-specific workspace / control packages
│   └── hexfellow/          # HexFellow-specific workspace / interfaces
├── scripts/
│   ├── build.sh            # Root compatibility build dispatcher
│   ├── container/          # Container lifecycle helpers
│   ├── intergation/        # Integration / workflow helpers
│   ├── perception/         # Perception startup helpers
│   └── recording/          # Recording helpers
├── docs/
│   ├── assets/             # Project figures / demo assets
│   └── user_guide/
│       └── Intruduction.md # Detailed setup and architecture guide
├── .gitmodules
├── THIRD_PARTY_NOTICES.md
└── LICENSE
```

> Note: the current user guide contains references to `README_agent.md` and `README_workflow.md`, but those files are not present at the repository root in the current `main` branch. Use `docs/user_guide/Intruduction.md` plus the actual scripts / package documentation as the source of truth until those runbooks are restored or replaced.

## Important components

### AgentOS

```text
agentic_robot/agentOS/
```

Contains the agent-side execution infrastructure, including reusable robot skills and background / sandbox-oriented helpers.

The intended pattern is roughly:

```text
natural-language task
        │
        ▼
AgentOS
        │ select / compose skills
        ▼
skill execution
        │
        ▼
runtime feedback
        │
        └────────► re-plan / continue / recover
```

### Core ROS 2 workspace

```text
agentic_robot/core/
```

Robot-independent functionality lives here. Important package groups described by the repository include:

- `nav_bringup`: Navigation2 launch and configuration;
- `navigation`: navigation executors and semantic / relative goals;
- `perception`: perception and inference ROS nodes;
- `fast_livo`: mapping / relocalization integration.

### Services

```text
agentic_robot/services/
```

Includes service-layer integrations such as:

- `robot_bridge`: YAML-driven HTTP ↔ ROS bridge;
- `multi_robot_ctl`: multi-robot control services / examples.

### FSR-VLN

```text
agentic_robot/fsr_vln/
```

FSR-VLN is the vision-language navigation component associated with the upstream research project. It combines semantic scene representations with fast / slow reasoning for long-range navigation tasks.

Upstream research resources:

- Project: https://horizonrobotics.github.io/robot_lab/fsr-vln
- arXiv: https://arxiv.org/abs/2509.13733

### Robot workspaces

```text
robots/unitree/
robots/hexfellow/
```

Robot-specific drivers, control packages, sensors and adapters are intentionally separated from the generic HoloAgent layer.

This separation matters: building `agentic_robot/core` successfully does not mean a selected robot workspace is fully configured for hardware.

## Environment baseline

The repository documentation targets a ROS 2 Humble-compatible Linux environment, typically:

- Ubuntu 22.04 / ROS 2 Humble-compatible tooling;
- `colcon`;
- `rosdep`;
- Python 3;
- CMake / C++ toolchain;
- CUDA-capable GPU for perception / semantic mapping workflows;
- hardware-specific drivers for camera, IMU, LiDAR, actuators and robot base.

Common dependencies mentioned by the current stack include:

- Sophus;
- GTSAM;
- PCL;
- OpenCV;
- Eigen;
- Ceres;
- `cv_bridge`;
- `image_transport`;
- Navigation2 dependencies;
- robot / sensor SDKs as required by the selected platform.

## ROS Humble dependency preparation

A typical prepared ROS 2 Humble environment needs packages similar to:

```bash
sudo apt update

sudo apt install -y \
  ros-humble-sophus \
  ros-humble-gtsam \
  libboost-all-dev \
  libtbb-dev \
  libpcl-dev \
  ros-humble-pcl-conversions \
  ros-humble-pcl-msgs \
  libopencv-dev \
  ros-humble-cv-bridge \
  ros-humble-image-transport \
  ros-humble-bondcpp \
  ros-humble-test-msgs \
  ros-humble-behaviortree-cpp-v3 \
  ros-humble-diagnostic-updater \
  ros-humble-rviz2 \
  ros-humble-angles \
  libgraphicsmagick++1-dev \
  libunwind-dev \
  libgoogle-glog-dev \
  libceres-dev \
  libxtensor-dev \
  libxsimd-dev \
  libompl-dev \
  libnanoflann-dev \
  tmux git wget curl python3-pip
```

TF-related packages used by the existing guide:

```bash
sudo apt install -y 'ros-humble-tf2-*' ros-humble-tf-transformations
```

Exact dependencies vary by workspace and robot. Prefer package-specific build errors / `package.xml` over blindly installing every possible ROS package.

## Clone submodules

The repository currently contains Git submodule configuration. After cloning:

```bash
git submodule update --init --recursive
```

One currently declared submodule is `rpg_vikit-ros2`; do not assume a shallow source checkout contains all third-party code.

## Build strategy

The project is layered. Build from generic dependencies toward the selected robot workspace.

```text
thirdparty
   │
   ▼
core
   │
   ▼
services
   │
   ▼
selected robot workspace
```

### Build all robot-independent layers

```bash
bash agentic_robot/build.sh --workspace all
```

### Build one layer

```bash
bash agentic_robot/build.sh --workspace thirdparty
bash agentic_robot/build.sh --workspace core
bash agentic_robot/build.sh --workspace services
```

### Build a specific package

```bash
bash agentic_robot/build.sh --package nav_executor
```

or constrain the workspace explicitly:

```bash
bash agentic_robot/build.sh \
  --workspace core \
  --package perception
```

### Limit parallel jobs

On memory-constrained machines:

```bash
bash agentic_robot/build.sh --workspace core --jobs 2
```

### Root build script

The root helper remains available as a compatibility dispatcher:

```bash
bash scripts/build.sh
```

For new development, the layer-specific build scripts are easier to reason about because they keep generic and robot-specific dependencies separate.

## Build the robot-specific workspace

After the generic layers build successfully, choose one hardware workspace.

### Unitree

```bash
bash robots/unitree/build.sh
```

Specific package example:

```bash
bash robots/unitree/build.sh --package g1_move
```

### HexFellow

```bash
bash robots/hexfellow/build.sh
```

Specific package:

```bash
bash robots/hexfellow/build.sh --package <package-name>
```

Do not build both robot trees by default unless the deployment genuinely needs both sets of drivers / interfaces.

## Source order

After building, source from generic to specific:

```bash
source /opt/ros/humble/setup.bash
source agentic_robot/thirdparty/install/setup.bash
source agentic_robot/core/install/setup.bash
source agentic_robot/services/install/setup.bash
source robots/unitree/install/setup.bash
```

or replace the final line with HexFellow:

```bash
source robots/hexfellow/install/setup.bash
```

Mixing a system-installed ROS package and a vendored version of the same package in one environment can cause confusing overlay behavior. Prefer one source for each dependency family.

## Running the system

There is no single universal “run everything” command because the stack depends on:

- selected robot;
- sensor suite;
- mapping / localization mode;
- model paths;
- service URLs;
- network topology;
- perception GPU environment;
- deployment-specific containers.

Start with:

```text
docs/user_guide/Intruduction.md
```

Then inspect the relevant orchestration helpers under:

```text
scripts/container/
scripts/intergation/
scripts/perception/
scripts/recording/
agentic_robot/agentOS/
```

Treat those scripts as deployment helpers, not as portable hardware-free examples.

## Agentic vs workflow-style operation

The current codebase supports two broad operating styles.

### Agentic mode

```text
natural-language goal
       │
       ▼
AgentOS
       │
       ▼
registered skills
       │
       ▼
ROS / service execution
       │
       ▼
runtime feedback
```

Use this when the task should be dynamically composed from available robot capabilities.

### Workflow mode

```text
known scenario
    │
    ▼
predefined scripts
    │
    ▼
known launch / service sequence
```

Use this for debugging repeatable pipelines and deployment bringup where deterministic orchestration is more useful than language-driven composition.

Because the previously referenced standalone workflow runbooks are not currently present, use the actual scripts and package launch files when determining the current runnable sequence.

## Runtime configuration

Deployment-specific values should remain in environment variables or local config files rather than being committed as universal defaults.

Common variables documented by the current stack include:

```text
ROBOT_ID
CONTROL_URL
ROBOT_11_URL ... ROBOT_16_URL
EXPECTED_ROBOTS
HOLOAGENT_DATA_ROOT
```

LLM / cloud integration may require:

```text
OPENAI_API_KEY
AZURE_OPENAI_API_KEY
AZURE_OPENAI_ENDPOINT
AZURE_OPENAI_DEPLOYMENT
```

Voice / chatbot integration may require values such as:

```text
CHATBOT_ARK_API_KEY
CHATBOT_ASR_APP_KEY
CHATBOT_ASR_ACCESS_KEY
CHATBOT_TTS_APP_KEY
CHATBOT_TTS_ACCESS_KEY
```

Do not commit real API keys, robot passwords, private IP credentials, or production service tokens.

## Models and data

Large model weights, maps, generated outputs and deployment-local assets are intentionally not guaranteed to live in Git.

Before running perception, mapping or semantic navigation, verify the exact package configuration for expected paths.

Typical external assets include:

- perception model weights;
- semantic mapping models;
- map / point-cloud data;
- calibration files;
- robot-specific configuration;
- datasets;
- recorded ROS bags.

A successful build with missing assets can still fail immediately at runtime.

## Real-robot bringup checklist

Before trying an Agent task, validate the robot stack bottom-up.

```text
1. Base robot / actuator control
2. Sensors publishing expected topics
3. TF tree
4. Localization / mapping
5. Navigation
6. Perception
7. Robot bridge / services
8. Individual skills
9. AgentOS orchestration
10. Natural-language task
```

If step 3 is broken, debugging the LLM is usually wasted effort.

## Debugging suggestions

### Package not found

Check source order and overlay state:

```bash
printenv AMENT_PREFIX_PATH
ros2 pkg list | grep <package>
```

### Build succeeds but launch fails

Look for missing runtime assets, environment variables or hardware drivers. Build dependencies and runtime dependencies are not always the same.

### Third-party package conflict

If the same Navigation2 / vikit package is available both from `/opt/ros/...` and `agentic_robot/thirdparty`, ensure the intended overlay wins consistently.

### Robot bridge works but robot does not move

Verify the ROS-side service / topic manually before involving HTTP or AgentOS.

### Agent selects a skill but execution fails

Debug the skill as a standalone deterministic operation first. Agentic composition should sit on top of reliable primitive skills.

## Safety

This repository targets physical robots. Language models and high-level agents should not be treated as the final safety layer.

For real hardware:

- retain an independent physical emergency stop;
- enforce deterministic velocity / workspace / actuator limits below the Agent layer;
- authenticate remote control endpoints;
- isolate robot control networks where possible;
- audit high-level commands and resulting skill execution;
- test new skills at low speed in controlled environments;
- require human supervision for hazardous manipulation / navigation tasks.

The Agent may decide *what* skill to request; deterministic robot software and hardware safety mechanisms must still constrain *what is allowed to happen*.

## Research resources

### HoloAgent

- Project: https://horizonrobotics.github.io/robot_lab/holoagent/
- arXiv: https://arxiv.org/abs/2606.23565
- Dataset: https://huggingface.co/datasets/HorizonRobotics/fsrvln_datasets

### FSR-VLN

- Project: https://horizonrobotics.github.io/robot_lab/fsr-vln
- arXiv: https://arxiv.org/abs/2509.13733

When using these components in academic work, cite the **original publications and authors**.

## Third-party software

This repository incorporates or references multiple open-source projects and vendored dependencies. See:

```text
THIRD_PARTY_NOTICES.md
.gitmodules
agentic_robot/thirdparty/
```

for current integration context.

Examples mentioned by the upstream project include OVO, HOV-SG, rerun, dimos, OpenClaw and ROS ecosystem components.

## License

The repository root includes an Apache License 2.0 `LICENSE` file. Third-party components may have their own licenses; those terms continue to apply independently. Preserve upstream notices and attribution when redistributing the combined system.
