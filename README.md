<div align="center">

<img src="docs/assets/holoagent_logo_text.png" alt="HoloAgent Logo" width="420"/>

# AgenticRobot / HoloAgent 集成仓库

[项目主页](https://horizonrobotics.github.io/robot_lab/holoagent/) · [arXiv](https://arxiv.org/abs/2606.23565) · [数据集](https://huggingface.co/datasets/HorizonRobotics/fsrvln_datasets) · [FSR-VLN](https://horizonrobotics.github.io/robot_lab/fsr-vln)

</div>

> **上游项目归属说明**
>
> HoloAgent 框架、FSR-VLN 研究成果、论文、数据集、Logo 以及相关上游组件均属于其原始作者与组织。本仓库是围绕该机器人技术栈进行开发、适配与集成的代码工作区。在科研引用、二次分发或继续开发时，请保留原项目的论文引用、许可证以及第三方声明。

AgenticRobot 是一个面向 ROS 2 的智能机器人集成工作区，重点覆盖 **自然语言驱动的机器人技能、导航、感知、语义建图、多机器人服务以及不同机器人平台的控制适配**。

当前仓库将机器人无关的 HoloAgent 核心能力，与 Unitree、HexFellow 等具体机器人平台的硬件适配工作区组合在一起。

> 这是一个 **面向真实机器人部署** 的系统，而不是纯软件、无硬件即可运行的 Demo。代码成功编译只是部署的一部分；实际运行还需要对应的 ROS 驱动、传感器、标定文件、地图、模型权重、GPU 环境、网络配置以及机器人侧服务。

## 项目整体架构

```text
自然语言 / Agent 层
        │
        ▼
AgentOS / Skills / Chatbot
        │
        ▼
机器人服务与任务编排
        │
        ├─ 导航
        ├─ 感知
        ├─ 语义建图 / FSR-VLN
        ├─ HTTP ↔ ROS Bridge
        └─ 多机器人控制
        │
        ▼
机器人平台适配层
        ├─ Unitree
        └─ HexFellow
        │
        ▼
真实机器人硬件
```

## 核心能力

- **Agentic 机器人执行**：将自然语言任务转化为已注册的机器人技能，并根据运行反馈持续执行、恢复或重新规划。
- **技能系统**：在 AgentOS 下组织可复用的机器人 Skill、文档、示例、注册与调度逻辑。
- **机器人导航**：提供基于 ROS 2 Navigation2 的启动配置，以及语义导航、相对目标导航等能力。
- **机器人感知**：集成摄像头、GPU 推理与 ROS 感知节点。
- **三维空间 / 语义记忆**：为具身导航和长时任务提供空间表达、检索与语义建图能力。
- **FSR-VLN**：集成 Fast-and-Slow Vision-Language Navigation，通过层级多模态场景表示完成长距离视觉语言导航。
- **Robot Bridge**：通过 HTTP ↔ ROS 服务层向上层系统暴露机器人控制能力。
- **多机器人服务**：包含多机器人协同与远程控制示例。
- **多平台适配**：机器人无关核心与 Unitree、HexFellow 等硬件工作区分离维护。
- **容器与集成脚本**：提供面向既有部署环境的构建、启动、感知、录制和容器管理脚本。

## 仓库结构

```text
AgenticRobot/
├── agentic_robot/
│   ├── agentOS/            # Agent / Skill 编排与执行基础设施
│   ├── chatbot/            # 语音 / 自然语言交互组件
│   ├── core/               # 机器人无关的 ROS 2 工作区
│   ├── services/           # HTTP/ROS Bridge 与多机器人服务
│   ├── fsr_vln/            # 语义建图 / 视觉语言导航组件
│   ├── thirdparty/         # 仓库内集成的 ROS / C++ 第三方依赖
│   ├── tools/              # 建图与通用工具
│   ├── build.sh            # 分层构建脚本
│   └── clear.sh            # 构建产物清理脚本
├── robots/
│   ├── unitree/            # Unitree 平台专用工作区 / 控制包
│   └── hexfellow/          # HexFellow 平台专用工作区 / 接口
├── scripts/
│   ├── build.sh            # 根目录兼容构建入口
│   ├── container/          # 容器生命周期管理脚本
│   ├── intergation/        # 集成 / 工作流启动脚本
│   ├── perception/         # 感知模块启动脚本
│   └── recording/          # 数据录制脚本
├── docs/
│   ├── assets/             # 项目图片 / Demo 资源
│   └── user_guide/
│       └── Intruduction.md # 更详细的环境、架构和部署说明
├── .gitmodules
├── THIRD_PARTY_NOTICES.md
└── LICENSE
```

> 当前 `docs/user_guide/Intruduction.md` 中仍可能引用 `README_agent.md` 和 `README_workflow.md`，但这两个文件在当前 `main` 分支根目录中并不存在。因此，请优先以 `docs/user_guide/Intruduction.md`、实际脚本以及各 ROS 包自身文档作为当前版本的运行依据。

## 关键模块说明

### AgentOS

```text
agentic_robot/agentOS/
```

AgentOS 负责高层 Agent 执行与机器人 Skill 编排，包含技能注册、后台任务、沙箱 / 长流程执行等相关基础设施。

整体执行模式可概括为：

```text
自然语言任务
      │
      ▼
AgentOS
      │ 选择 / 组合 Skill
      ▼
技能执行
      │
      ▼
运行时反馈
      │
      └────────► 继续执行 / 恢复 / 重新规划
```

### Core ROS 2 工作区

```text
agentic_robot/core/
```

该目录主要存放与具体机器人型号无关的 ROS 2 功能。

当前仓库的重要模块包括：

- `nav_bringup`：Navigation2 启动与参数配置；
- `navigation`：导航执行器、语义目标和相对目标导航；
- `perception`：感知与推理相关 ROS 节点；
- `fast_livo`：FAST-LIVO 建图 / 重定位相关集成。

### Services

```text
agentic_robot/services/
```

用于对外提供机器人服务能力，主要包括：

- `robot_bridge`：基于 YAML 配置的 HTTP ↔ ROS Bridge；
- `multi_robot_ctl`：多机器人控制服务与示例。

### FSR-VLN

```text
agentic_robot/fsr_vln/
```

FSR-VLN 是上游 HoloAgent 体系中的视觉语言导航组件，通过层级多模态场景图与 Fast / Slow Reasoning 实现长距离空间推理和导航。

上游资料：

- 项目主页：https://horizonrobotics.github.io/robot_lab/fsr-vln
- arXiv：https://arxiv.org/abs/2509.13733

### 机器人平台工作区

```text
robots/unitree/
robots/hexfellow/
```

机器人驱动、执行器、传感器、控制接口与平台专属逻辑被单独维护，避免与机器人无关的 HoloAgent 核心代码强耦合。

因此：

> `agentic_robot/core` 编译成功，并不意味着某个具体机器人已经可以直接运行。

实际部署还需要对应平台的 SDK、ROS 驱动、网络、传感器、标定以及硬件配置。

## 环境基线

当前仓库文档主要面向 ROS 2 Humble 兼容的 Linux 环境，典型配置包括：

- Ubuntu 22.04 / ROS 2 Humble；
- `colcon`；
- `rosdep`；
- Python 3；
- CMake / C++ 编译工具链；
- 用于感知和语义建图的 CUDA GPU；
- 相机、IMU、LiDAR、机器人底盘和执行器对应的硬件驱动。

常见依赖包括：

- Sophus；
- GTSAM；
- PCL；
- OpenCV；
- Eigen；
- Ceres；
- `cv_bridge`；
- `image_transport`；
- Navigation2 相关依赖；
- 对应机器人 / 传感器 SDK。

## ROS 2 Humble 依赖准备

典型 ROS 2 Humble 环境可以先准备以下依赖：

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

TF 相关依赖：

```bash
sudo apt install -y 'ros-humble-tf2-*' ros-humble-tf-transformations
```

不同工作区、机器人和传感器的依赖并不完全相同。实际开发时，应优先根据具体 ROS 包的 `package.xml`、构建错误和对应硬件文档补充依赖，而不是无差别安装全部软件包。

## 克隆 Submodule

仓库中存在 Git Submodule 配置，首次克隆后建议执行：

```bash
git submodule update --init --recursive
```

当前 `.gitmodules` 中包含 `rpg_vikit-ros2` 等依赖，因此不要假设普通浅克隆已经包含全部第三方源码。

## 构建策略

该项目采用分层工作区结构，建议按照“底层依赖 → 通用核心 → 服务 → 机器人平台”的顺序构建：

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
指定机器人工作区
```

### 构建全部机器人无关工作区

```bash
bash agentic_robot/build.sh --workspace all
```

### 单独构建某一层

```bash
bash agentic_robot/build.sh --workspace thirdparty
bash agentic_robot/build.sh --workspace core
bash agentic_robot/build.sh --workspace services
```

### 构建指定 ROS 包

```bash
bash agentic_robot/build.sh --package nav_executor
```

也可以明确指定工作区：

```bash
bash agentic_robot/build.sh \
  --workspace core \
  --package perception
```

### 限制并行编译数

内存较小的机器建议降低并行度：

```bash
bash agentic_robot/build.sh --workspace core --jobs 2
```

### 根目录构建入口

仓库根目录保留兼容构建脚本：

```bash
bash scripts/build.sh
```

对于新开发，优先推荐使用各层自己的 `build.sh`，这样更容易区分机器人无关依赖和具体机器人平台依赖。

## 构建机器人平台工作区

通用层编译完成后，再选择实际使用的机器人平台。

### Unitree

```bash
bash robots/unitree/build.sh
```

构建指定包：

```bash
bash robots/unitree/build.sh --package g1_move
```

### HexFellow

```bash
bash robots/hexfellow/build.sh
```

构建指定包：

```bash
bash robots/hexfellow/build.sh --package <package-name>
```

除非部署场景确实需要同时支持多类机器人，否则不建议默认同时构建全部机器人平台工作区。

## Source 顺序

构建完成后，建议从通用层向机器人平台层依次加载环境：

```bash
source /opt/ros/humble/setup.bash
source agentic_robot/thirdparty/install/setup.bash
source agentic_robot/core/install/setup.bash
source agentic_robot/services/install/setup.bash
source robots/unitree/install/setup.bash
```

如果使用 HexFellow，则最后一行替换为：

```bash
source robots/hexfellow/install/setup.bash
```

如果同一个依赖同时存在于 `/opt/ros/...` 系统环境和 `agentic_robot/thirdparty` 中，ROS Overlay 顺序可能产生难以定位的问题。对于同一依赖族，尽量保持单一来源。

## 系统运行方式

本项目没有一个对所有硬件都适用的“启动全部模块”命令，因为运行流程取决于：

- 机器人型号；
- 传感器组合；
- 建图 / 定位模式；
- 模型路径；
- ROS Topic / Service；
- 网络拓扑；
- GPU 推理环境；
- 容器和部署方式。

首次部署建议先阅读：

```text
docs/user_guide/Intruduction.md
```

然后根据实际任务查看：

```text
scripts/container/
scripts/intergation/
scripts/perception/
scripts/recording/
agentic_robot/agentOS/
```

这些脚本主要用于已经准备好的机器人 / 容器环境，不应当被理解为无硬件即可运行的通用 Quick Start。

## Agentic 模式与 Workflow 模式

当前代码可以大致分为两类使用方式。

### Agentic 模式

```text
自然语言目标
     │
     ▼
AgentOS
     │
     ▼
注册的机器人 Skills
     │
     ▼
ROS / 服务执行
     │
     ▼
运行时反馈
```

适用于需要 Agent 根据实际任务动态选择、组合机器人能力的场景。

### Workflow 模式

```text
已知固定场景
     │
     ▼
预定义脚本
     │
     ▼
固定 Launch / Service 执行顺序
```

适用于调试、重复测试、系统 Bringup 和确定性部署流程。

由于此前文档提到的独立 Workflow Runbook 当前并不存在，请以实际脚本、Launch 文件和 ROS 包实现作为当前版本的运行依据。

## 运行时配置

部署相关参数应通过环境变量或本地配置文件提供，不建议将真实环境值硬编码进公共仓库。

当前系统常见变量包括：

```text
ROBOT_ID
CONTROL_URL
ROBOT_11_URL ... ROBOT_16_URL
EXPECTED_ROBOTS
HOLOAGENT_DATA_ROOT
```

LLM / 云服务可能使用：

```text
OPENAI_API_KEY
AZURE_OPENAI_API_KEY
AZURE_OPENAI_ENDPOINT
AZURE_OPENAI_DEPLOYMENT
```

语音 / Chatbot 模块可能还需要：

```text
CHATBOT_ARK_API_KEY
CHATBOT_ASR_APP_KEY
CHATBOT_ASR_ACCESS_KEY
CHATBOT_TTS_APP_KEY
CHATBOT_TTS_ACCESS_KEY
```

请勿提交真实 API Key、机器人密码、生产环境 Token 或私有网络凭据。

## 模型与数据

模型权重、地图、生成结果以及本地部署资产不保证存放在 Git 仓库中。

运行感知、语义建图或导航之前，应检查具体配置文件中要求的路径。

典型外部资源包括：

- 感知模型权重；
- 语义建图模型；
- 地图 / 点云数据；
- 相机、IMU、LiDAR 标定；
- 机器人平台配置；
- 数据集；
- ROS Bag。

> 编译成功并不代表运行时资源已经完整。缺少模型、地图或标定文件时，节点仍可能在启动阶段立即失败。

## 真机 Bringup 推荐顺序

建议从最底层开始逐步确认，每一步稳定后再继续向 Agent 层推进：

```text
1. 底盘 / 执行器能够独立控制
2. 传感器正常发布 Topic
3. TF 树正确
4. 建图 / 定位正常
5. 导航正常
6. 感知正常
7. Robot Bridge / Service 正常
8. 单个 Skill 可以独立执行
9. AgentOS 能够正确调用 Skill
10. 最后再接入自然语言任务
```

如果第 3 步 TF 树都不正确，此时优先调试 LLM 通常没有意义。

## 常见问题与排查

### 找不到 ROS 包

检查当前 Overlay 与 Source 顺序：

```bash
printenv AMENT_PREFIX_PATH
ros2 pkg list | grep <package>
```

### 编译成功，但 Launch 启动失败

重点检查运行时模型、地图、配置、驱动和环境变量。编译依赖完整并不等于运行时依赖完整。

### 第三方依赖冲突

如果 Navigation2、vikit 等包同时存在于系统 ROS 环境和 `agentic_robot/thirdparty` 中，请确认当前 Source 顺序与预期一致。

### Robot Bridge 可以收到请求，但机器人不执行

先绕过 HTTP 和 AgentOS，直接在 ROS 侧测试对应 Topic / Service / Action 是否能够正常控制机器人。

### Agent 能选择 Skill，但 Skill 执行失败

先将该 Skill 作为确定性的独立操作调通，再接入 Agentic 编排。

高层 Agent 应建立在可靠的底层原子能力之上，而不是用 Agent 掩盖底层执行问题。

## 真机安全说明

本仓库面向物理机器人。LLM 与高层 Agent 不应被当作最终安全层。

真实机器人部署建议至少保证：

- 保留独立、可立即触发的物理急停；
- 在 Agent 层以下实施确定性的速度、工作空间和执行器限制；
- 对远程控制接口进行身份认证；
- 尽可能隔离机器人控制网络；
- 记录高层命令、Skill 调用和实际执行结果；
- 新 Skill 首次测试时使用低速和受控环境；
- 高风险导航与操作任务保留人工监督。

Agent 可以决定“请求执行什么 Skill”，但机器人底层软件与硬件安全机制必须决定“哪些动作实际上被允许发生”。

## 科研资源

### HoloAgent

- 项目主页：https://horizonrobotics.github.io/robot_lab/holoagent/
- arXiv：https://arxiv.org/abs/2606.23565
- 数据集：https://huggingface.co/datasets/HorizonRobotics/fsrvln_datasets

### FSR-VLN

- 项目主页：https://horizonrobotics.github.io/robot_lab/fsr-vln
- arXiv：https://arxiv.org/abs/2509.13733

在论文、报告和公开科研成果中使用相关研究时，请引用 **原论文与原作者**。

## 第三方软件

本仓库集成或引用了多个开源项目及第三方依赖。请查看：

```text
THIRD_PARTY_NOTICES.md
.gitmodules
agentic_robot/thirdparty/
```

了解当前仓库中的第三方软件来源与集成关系。

上游 HoloAgent 项目中还涉及 OVO、HOV-SG、rerun、dimos、OpenClaw 等开源项目，使用时请同时遵守对应项目的许可证。

## License

本仓库根目录包含 `LICENSE` 文件。第三方源码、Submodule、模型、数据集和科研组件可能使用与根仓库不同的许可证。

在重新分发、商用或公开发布前，请同时确认：

- 根仓库 `LICENSE`；
- `THIRD_PARTY_NOTICES.md`；
- `.gitmodules` 中各上游仓库的许可证；
- 使用的模型与数据集许可证；
- HoloAgent / FSR-VLN 原项目的引用与授权要求。

---

如果你是第一次接触本仓库，建议按照以下顺序开始：

```text
1. 阅读本 README，理解项目分层
2. 阅读 docs/user_guide/Intruduction.md
3. 初始化 Git Submodule
4. 编译 thirdparty → core → services
5. 编译目标机器人工作区
6. 单独验证机器人、传感器、TF 和导航
7. 验证单个 Skill
8. 最后接入 AgentOS 和自然语言交互
```
