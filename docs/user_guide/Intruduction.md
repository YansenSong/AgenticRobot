HoloAgent 智能机器人系统

HoloAgent 智能机器人系统是一套基于 ROS 2 的机器人软件栈，支持自然语言驱动的机器人技能、导航、感知、语义建图以及面向具体机器人的控制适配器。

当前仓库支持两种运行方式：

- **智能体模式（Agentic mode）**：通过自然语言任务注册并组合机器人技能。
- **工作流模式（Workflow mode）**：使用预定义脚本编排已知工作流，便于调试。

本项目面向真实机器人部署环境。目前，无需硬件的快速开始流程和公开模型分发流程仍在准备中。

## 仓库目录结构

```text
agentic_robot_system/
├── agentic_robot/
│   ├── agentOS/
│   │   ├── holoagent_skills/ # 技能注册表、单技能文档、示例和增删改查工具
│   │   ├── run_dameon/       # 后台守护进程辅助工具
│   │   └── sandbox_test/     # 长时任务和集成测试脚本
│   ├── core/                 # 与机器人无关的 ROS 2 工作区
│   ├── services/             # HTTP/ROS 桥接和多机器人控制服务
│   ├── thirdparty/           # core 使用的第三方 ROS/C++ 依赖源码
│   ├── fsr_vln/              # 语义建图和检索 API
│   └── tools/                # 建图和通用工具箱
├── robots/
│   ├── unitree/              # Unitree 机器人专用 ROS 2 工作区和脚本
│   └── hexfellow/            # HexFellow 机器人专用 ROS 2 工作区和脚本
├── scripts/
│   ├── build.sh              # 工作区构建辅助脚本
│   ├── container/            # 容器生命周期辅助脚本
│   ├── intergation/          # 集成/工作流启动辅助脚本
│   ├── perception/           # 感知启动辅助脚本
│   └── recording/            # 数据录制辅助脚本
├── README_agent.md           # 智能体模式操作指南
└── README_workflow.md        # 工作流模式操作指南
```

## 主要组件

- `agentic_robot/core/src/nav_bringup`：Navigation2 启动文件和参数。
- `agentic_robot/core/src/navigation`：导航执行器以及语义/相对目标相关软件包。
- `agentic_robot/core/src/perception`：感知 ROS 节点和 GPU 推理集成。
- `agentic_robot/core/src/fast_livo`：基于 FAST-LIVO 的建图和重定位集成。
- `agentic_robot/services/src/robot_bridge`：基于 YAML 配置的 HTTP 到 ROS 桥接器。
- `agentic_robot/services/src/multi_robot_ctl`：多机器人 HTTP 控制示例。
- `robots/unitree/src`：Unitree 机器人机械臂和运动控制软件包。
- `robots/hexfellow/src`：HexFellow 相机检测、升降控制和接口软件包。

## 环境要求

预期的基础环境包括：

- 支持 ROS 2 Humble 工具链的 Ubuntu 系统
- `colcon`、`rosdep` 以及标准 ROS 2 构建工具
- 用于 ROS 2 Python 软件包和服务工具的 Python 3
- 用于感知和语义建图工作流的支持 CUDA 的 GPU
- 所选机器人、相机、IMU、LiDAR 和执行器对应的硬件驱动

重要的外部依赖包括 `livox_ros_driver2`、GTSAM、PCL、OpenCV、Eigen、`cv_bridge` 和 `image_transport`。部分工作流还需要 ZED、Unitree、HexFellow、OpenAI/Azure OpenAI、模型权重以及本地地图/数据资源。

模型权重、生成文件、构建产物和本地部署配置不会被纳入版本控制。

## ROS Humble Docker 依赖配置

当前经过测试的配置以 ROS 2 Humble Docker 镜像为基础，在构建本仓库前安装以下系统依赖和驱动依赖。

```bash
apt update

# Sophus
apt install -y ros-humble-sophus

# GTSAM
apt install -y cmake libboost-all-dev libtbb-dev
apt install -y ros-humble-gtsam

# PCL
apt install -y libpcl-dev ros-humble-pcl-conversions ros-humble-pcl-msgs

# OpenCV 和 ROS 图像桥接
apt install -y libopencv-dev ros-humble-cv-bridge ros-humble-image-transport

# vendored thirdparty 工作区使用的 Navigation2 依赖
apt install -y \
  ros-humble-bondcpp ros-humble-test-msgs \
  ros-humble-behaviortree-cpp-v3 ros-humble-diagnostic-updater \
  libgraphicsmagick++1-dev ros-humble-rviz2 ros-humble-angles \
  libunwind-dev libgoogle-glog-dev libceres-dev \
  libxtensor-dev libxsimd-dev libompl-dev libnanoflann-dev

apt install -y ros-humble-tf2-* ros-humble-tf-transformations
apt install -y --only-upgrade ros-humble-geometry-msgs

# 常用工具
apt install -y tmux git wget curl python3-pip
```

## 构建

构建主工作区：

```bash
bash scripts/build.sh
```

通过辅助脚本构建指定的软件包组：

```bash
bash agentic_robot/build.sh -p robot_bridge
```

`agentic_robot/core` 不是完全独立的工作区。它依赖当前放置在 `agentic_robot/thirdparty` 下的部分软件包，包括 `nav2_simple_commander`、`nav2_bringup` 等 Navigation2 软件包，以及 `vikit_common`、`vikit_ros` 等 `rpg_vikit-ros2` 软件包。

正常的构建顺序按层划分：

1. 准备 ROS、系统和驱动依赖。
2. 构建非机器人专用的 `agentic_robot` 层：`thirdparty` -> `core` -> `services`。
3. 构建一个机器人专用工作区：`robots/unitree` 或 `robots/hexfellow`。

先构建非机器人专用层：

```bash
bash agentic_robot/build.sh --workspace all
```

在处理单个工作区时，也可以构建范围更小的非机器人目标：

```bash
bash agentic_robot/build.sh --workspace thirdparty
bash agentic_robot/build.sh --workspace core
bash agentic_robot/build.sh --workspace services
```

然后构建选定的机器人工作区：

```bash
bash robots/unitree/build.sh
# 或
bash robots/hexfellow/build.sh
```

构建单个非机器人软件包：

```bash
bash agentic_robot/build.sh --package nav_executor
bash agentic_robot/build.sh --workspace core --package perception
```

构建单个机器人软件包：

```bash
bash robots/unitree/build.sh --package g1_move
bash robots/hexfellow/build.sh --package <pkg>
```

在配置较低的机器上限制并行度：

```bash
bash agentic_robot/build.sh --workspace core --jobs 2
bash robots/unitree/build.sh --jobs 2
```

根目录下的 `scripts/build.sh` 仍作为旧命令的兼容分发器，例如支持 `bash scripts/build.sh --workspace core` 或 `bash scripts/build.sh --workspace unitree`。新工作流优先使用分层构建脚本，因为它们可以将与机器人无关的依赖和机器人专用依赖分开管理。

构建完成后，按照从通用到具体的顺序加载工作区：

```bash
source agentic_robot/thirdparty/install/setup.bash
source agentic_robot/core/install/setup.bash
source agentic_robot/services/install/setup.bash      # 如果构建了 robot_bridge
source robots/unitree/install/setup.bash              # 或 robots/hexfellow/install/setup.bash
```

可以使用系统 ROS 软件包替代 vendored thirdparty 软件包，但在同一个环境中应避免为同一组依赖混用两种来源。

## 运行

详细的操作流程仍请参考现有操作指南：

- 智能体模式：[README_agent.md](README_agent.md)
- 工作流模式：[README_workflow.md](README_workflow.md)

仓库中的启动/编排辅助工具主要位于 `scripts/container/`、`scripts/intergation/`、`scripts/perception/` 和 `scripts/audio/`。这些脚本假设机器人/容器环境和硬件相关服务已经准备就绪，并不是无需硬件的快速开始方案。

## 配置

运行时配置应通过环境变量或本地配置文件提供，不要提交默认配置。常见变量包括：

- `ROBOT_ID`
- `CONTROL_URL`
- `ROBOT_11_URL` 至 `ROBOT_16_URL`
- `EXPECTED_ROBOTS`
- `OPENAI_API_KEY`
- `AZURE_OPENAI_API_KEY`
- `AZURE_OPENAI_ENDPOINT`
- `AZURE_OPENAI_DEPLOYMENT`
- `HOLOAGENT_DATA_ROOT`

如果需要聊天机器人的语音交互，还需配置：

- `CHATBOT_ARK_API_KEY`
- `CHATBOT_ASR_APP_KEY`
- `CHATBOT_ASR_ACCESS_KEY`
- `CHATBOT_TTS_APP_KEY`
- `CHATBOT_TTS_ACCESS_KEY`

本地模型路径、地图路径、机器人 IP 地址和服务 URL 都应视为部署配置。

## 模型与数据

模型文件和数据集不存储在本仓库中。在运行感知或语义建图前，请将所需资源放置到相关配置文件指定的路径，或通过本地配置覆盖这些路径。

在公开发布前，模型分发方案、校验和以及许可证信息仍需最终确定。

## 入口

- 智能体工作流：参见 [README_agent.md](README_agent.md)
- 预定义工作流：参见 [README_workflow.md](README_workflow.md)
- 技能系统：参见 [agentic_robot/agentOS/holoagent_skills/README.md](agentic_robot/agentOS/holoagent_skills/README.md)
- AgentOS 概览：参见 [agentic_robot/agentOS/README.md](agentic_robot/agentOS/README.md)

## 第三方代码

本仓库包含 vendored 的第三方源代码。当前的声明列表和待处理的许可证审查事项请参阅 [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)。

已知后续事项：在公开发布前，解决 FAST-LIVO 的 README 与软件包元数据之间存在的许可证信息不一致问题。

## 安全注意事项

本软件可以控制真实机器人和执行器。请仅在受控环境中运行，并配备适当的急停流程、速度限制和人员监督。不得将 LLM、视觉和语义输出视为安全关键决策。

## 许可证

除非某个文件、软件包或第三方目录另有说明，本仓库自有代码采用 Apache License 2.0。详情请参阅 [LICENSE](LICENSE) 和 [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)。

## 说明

- 许多工作流假设机器人侧服务、地图和模型资源已经存在。
- 第三方目录以 vendored 方式引入，应与项目维护的文档分开管理。
- 部分脚本与内部部署拓扑和机器人 IP 规划紧密耦合。

## 相关基础模型

部分 HoloAgent 演示依赖 HorizonRobotics 的基础模型。下面列出两个直接相关的开源项目。

### HoloBrain

[HoloBrain](https://horizonrobotics.github.io/robot_lab/holobrain/) 是一个面向通用具身操作的基础模型，用于 HexFellow 移动操作演示，并通过显式的具身先验（包括相机参数和运动学描述）支持异构机器人。更多信息请参阅其[开源实现](https://github.com/HorizonRobotics/RoboOrchardLab/tree/master/projects/holobrain)。

### HoloMotion

[HoloMotion](https://horizonrobotics.github.io/robot_lab/holomotion/) 是一个面向全身人形机器人控制的基础模型，用于 G1 相关演示中的稳健全身运动跟踪，并提供涵盖运动数据、训练、评估和真实机器人部署的端到端流程。更多信息请参阅其[开源仓库](https://github.com/HorizonRobotics/HoloMotion)。
