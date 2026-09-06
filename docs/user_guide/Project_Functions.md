# HoloAgent 项目功能介绍

## 1. 项目定位

HoloAgent 是一套基于 ROS 2 的具身智能机器人软件栈，将自然语言交互、任务规划、空间记忆、视觉感知、导航和机器人底层控制连接起来，用于真实机器人上的长时任务执行。

项目目前支持两种运行方式：

- **智能体模式（Agentic mode）**：由语言模型理解自然语言任务，选择并组合已注册的机器人技能。
- **工作流模式（Workflow mode）**：通过预定义脚本编排固定流程，适合调试、集成验证和演示。

项目整体介绍见 [`README.md`](../../README.md) 和 [`Intruduction.md`](Intruduction.md)。

## 2. 主要功能

### 2.1 自然语言和语音交互

Unitree G1 配套聊天模块实现了完整的语音交互链路：

```text
麦克风录音 → ASR 语音识别 → LLM 任务理解 → TTS 语音合成 → 扬声器播放
```

主要能力包括：

- 语音唤醒和休眠；
- 中文/英文对话配置；
- 使用豆包 ASR、LLM 和 TTS 服务；
- 将用户指令解析为问答、语义导航、相对移动或机器人动作；
- 通过 ROS topic 将解析结果发送给导航和动作执行节点。

主要实现位于 [`agentic_robot/chatbot/g1/g1.py`](../../agentic_robot/chatbot/g1/g1.py) 和 [`g1chat_demo.py`](../../agentic_robot/chatbot/g1/g1chat_demo.py)。

### 2.2 AgentOS 任务规划与技能编排

AgentOS 负责把自然语言任务转换为可执行技能流程，支持：

- 技能注册和技能元数据管理；
- 技能的创建、删除、查看、校验和同步；
- 长文本任务拆解为 DAG；
- 任务节点依赖关系管理；
- 单机器人串行任务；
- 多机器人并行和串行协作；
- 执行前虚拟验证；
- 执行过程监控和结果写入 YAML 文件；
- 后台守护进程维护机器人侧服务。

当前内置技能包括：

- `sem-nav-skill`：楼层、房间、物体级别的语义导航；
- `rel-move-skill`：相对位置和相对角度移动；
- `arm-skill`：预定义机械臂或全身动作；
- `robot-service`：通用 HTTP 服务调用；
- `workflow`：多步骤工作流组合。

相关代码见 [`agentOS/README.md`](../../agentic_robot/agentOS/README.md) 和 [`long_horizon_text_runner.py`](../../agentic_robot/agentOS/sandbox_test/long_horizon_text_runner.py)。

### 2.3 普通导航和航点导航

项目使用 Nav2 执行机器人导航，支持：

- 地图坐标目标导航；
- 预定义航点导航；
- 多航点连续导航；
- 导航任务停止和取消；
- 导航到点反馈；
- 按机器人和地图加载不同的 signal 配置；
- 导航完成后触发机械臂或其他动作。

导航执行节点接收 `object_pose` 和 `chat_signal_pub`，并通过 `waypoint_reached` 发布执行反馈。相关实现见 [`navigation/README.md`](../../agentic_robot/core/src/navigation/README.md) 和 [`pubpose.py`](../../agentic_robot/core/src/navigation/nav_executor/nav_executor/pubpose.py)。

### 2.4 语义导航

语义导航允许用户通过自然语言描述目标位置，例如：

```text
1楼，茶水间，咖啡机
```

执行流程为：

```text
/chat_loc_pub
    ↓
semantic_goal_node
    ↓
FSR-VLN 查询楼层、房间和物体
    ↓
生成地图坐标目标
    ↓
/object_pose
    ↓
Nav2 导航
```

语义导航节点位于 [`semantic_goal_node.py`](../../agentic_robot/core/src/navigation/semantic_goal/semantic_goal/semantic_goal_node.py)。

### 2.5 相对导航

项目支持以下格式的相对移动命令：

```text
forward,left,degrees
```

例如：

```text
1.0,0.5,30
```

含义为向前移动 1 米、向左移动 0.5 米、左转 30 度。系统会读取当前 `map -> base_link` TF，将机器人坐标系下的相对目标转换为地图坐标系下的绝对目标，再交给 Nav2 执行。

相关实现见 [`relative_goal_node.py`](../../agentic_robot/core/src/navigation/relative_goal/relative_goal/relative_goal_node.py)。

### 2.6 3D 建图、定位和重定位

项目集成 FAST-LIVO，支持激光、视觉和 IMU 融合定位建图，主要包括：

- 离线三维建图；
- 在线 LiDAR-视觉-惯性里程计；
- Unitree G1 在线重定位；
- HexFellow 在线重定位；
- 多会话地图融合或增量建图；
- 点云地图保存和后处理。

相关代码位于 [`agentic_robot/core/src/fast_livo`](../../agentic_robot/core/src/fast_livo)。

### 2.7 3D 空间记忆和 FSR-VLN 语义检索

FSR-VLN 是项目的空间记忆和视觉语言导航模块，主要流程为：

```text
RGB-D 数据
    ↓
实例建图和目标跟踪
    ↓
OVO 语义地图
    ↓
HMSG 分层多模态场景图
    ↓
楼层/房间/物体查询
    ↓
目标地图坐标
```

它可以：

- 保存楼层、房间、物体、视角和导航关系；
- 根据自然语言查询目标物体；
- 使用 CLIP 等视觉语言特征进行检索；
- 使用可选的 GPT/VLM 进行更慢但更精细的推理；
- 返回目标物体在场景图坐标和地图坐标中的位置；
- 支持离线场景建图、图结构可视化和查询评估；
- 提供 ROS 在线语义建图入口。

主要接口见 [`fsr_vln/api.py`](../../agentic_robot/fsr_vln/api.py)，建图入口见 [`run_holoagent_mapping.py`](../../agentic_robot/fsr_vln/run_holoagent_mapping.py)。

### 2.8 视觉感知和三维目标检测

感知模块以 ZED RGB-D 相机为输入，提供：

- RGB 图像和深度图同步；
- YOLO-E 开放词汇二维目标检测；
- 文本提示词或无提示词检测；
- 目标框、类别、置信度、掩码和跟踪信息；
- 根据深度图生成目标三维点云；
- 计算目标中心、范围和空间位置；
- 发布二维检测可视化图像；
- 发布三维点云和 MarkerArray；
- 通过 Redis 将图像转发到独立 GPU 推理环境。

主要实现见 [`perception/__init__.py`](../../agentic_robot/core/src/perception/perception/__init__.py) 及其 `modules`、`detectors` 子目录。

### 2.9 Unitree G1 机器人控制

Unitree G1 适配层支持：

- 通过速度指令控制机器人底盘移动；
- 对速度、角速度和持续时间进行限制；
- 发布和获取机器人里程计/IMU 信息；
- 执行预定义手臂和全身动作；
- 挥手、击掌、握手、拥抱、比心、举手等交互动作；
- `clamp` 夹取动作；
- `release_arm` 释放机械臂动作；
- 通过 FIFO 将 ROS 指令转发给 Unitree SDK；
- 动作结束后自动执行释放动作以降低安全风险。

动作映射主要位于 [`getcmd.cpp`](../../robots/unitree/src/g1_arm/src/getcmd.cpp)，底盘速度控制位于 [`pubmove.cpp`](../../robots/unitree/src/g1_move/src/pubmove.cpp)。

### 2.10 HexFellow 机器人工作流

HexFellow 目录提供机器人运行脚本、Nav2 参数和地图 signal 配置，支持：

- 启动硬件和底盘相关服务；
- 启动 LiDAR/定位和导航流程；
- 执行地图航点任务；
- 取洗衣篮、放洗衣篮和叠衣服等多航点任务示例；
- 导航完成后调用外部 HoloBrain 容器服务。

例如，`sh3_520` 地图配置中包含取篮子、放篮子和叠衣服的连续航点，见 [`signals.yaml`](../../robots/hexfellow/config/maps/sh3_520/signals.yaml)。

### 2.11 HTTP/ROS 桥接和多机器人控制

`robot_bridge` 将 HTTP 请求映射为 ROS topic，当前配置了以下接口：

- `POST /api/navigation/{name}`：触发命名导航 signal；
- `POST /api/relative_nav`：发送相对导航命令；
- `POST /api/semantic_nav`：发送语义导航命令；
- `POST /api/arm/{skill}`：触发预定义动作；
- `POST /api/navigation/stop`：停止导航；
- `GET /health`：服务健康检查。

它还可以订阅 `waypoint_reached`，将机器人到点状态反向上报给控制中心。实现见 [`robot_bridge_node.py`](../../agentic_robot/services/src/robot_bridge/robot_bridge/robot_bridge_node.py) 和 [`bridge_config.yaml`](../../agentic_robot/services/src/robot_bridge/config/bridge_config.yaml)。

多机器人控制中心可以向指定机器人或全部机器人发送导航、停止等指令，并记录各机器人到点状态，见 [`fastapi_control.py`](../../agentic_robot/services/src/multi_robot_ctl/fastapi_control.py)。

### 2.12 地图、数据和运维工具

项目还提供：

- PCD、PLY 和点云格式转换；
- 点云关键帧提取；
- 二维栅格地图生成；
- RGB-D 图像投影和深度图生成；
- 地图可视化和人工设置初始位姿；
- 传感器数据录制；
- Docker、tmux 和多进程启动脚本；
- 感知、音频、导航和集成流程启动脚本。

## 3. 典型使用场景

### 场景一：语音寻找物体

```text
用户：“去一楼茶水间找咖啡机”
    ↓
G1 语音识别
    ↓
LLM 提取 floor / room / object
    ↓
FSR-VLN 查询咖啡机位置
    ↓
生成地图目标点
    ↓
Nav2 导航到目标
```

### 场景二：长时单机器人任务

```text
去点位 1 → 挥手 → 去点位 2 → 击掌 → 去点位 3 → 再见
```

AgentOS 可以把上述指令拆成有依赖关系的导航和动作节点，逐步执行并保存执行结果。

### 场景三：多机器人协同

```text
11 号和 12 号机器人并行前往点位 1
    ↓
各自执行不同动作
    ↓
全部完成后继续前往点位 2
```

该流程可以由多机器人 DAG 和控制中心共同完成。

### 场景四：HexFellow 多航点任务

```text
前往洗衣篮 → 取篮子 → 导航到放置点 → 放置 → 前往叠衣服位置
```

当前仓库主要提供导航和工作流编排，具体抓取、放置和全身操作依赖机器人侧驱动或外部模型服务。

## 4. 当前边界和注意事项

### 4.1 需要真实部署环境

项目面向真实机器人部署，不是纯软件模拟器。通常需要：

- Ubuntu 和 ROS 2 Humble；
- `colcon`、`rosdep` 等构建工具；
- CUDA GPU；
- 相机、IMU、LiDAR 和机器人驱动；
- 已准备好的地图、模型权重和数据集；
- LLM、ASR、TTS 服务的 API 配置。

### 4.2 当前不是完整的通用操作系统

当前机械臂能力以预定义动作和有限的 `clamp` 动作为主，并不等同于已经实现了任意物体识别、抓取、搬运和放置的通用移动操作系统。

README 中列出的 HoloBrain 通用灵巧操作、HoloMotion 全身移动操作、主动探索 Agent-Loop，以及纯视觉自主建图/定位/导航仍属于未完成或后续版本能力，见 [`README.md`](../../README.md)。

### 4.3 桥接接口需要部署前核对

当前 `robot_bridge` 的 HTTP 到 ROS topic 映射已经实现，但 service/action 类型在代码中仍是预留接口。部分 HTTP 示例脚本的请求字段也需要与 `bridge_config.yaml` 统一后再用于实际部署。

### 4.4 安全要求

该项目能够控制真实机器人和执行器。应在受控环境中运行，配置急停、速度限制和人员监督，不能将 LLM、视觉或语义检索结果直接视为安全关键决策。

## 5. 总结

当前 HoloAgent 最完整的能力组合是：

> **Unitree G1/HexFellow 机器人 + ROS 2 Nav2 + FAST-LIVO 定位建图 + FSR-VLN 语义空间查询 + AgentOS 技能编排 + 语音/HTTP 控制。**

它适合用于实现自然语言驱动的机器人导航、空间目标查找、预定义动作交互、长时任务编排、多机器人协同，以及面向真实机器人的建图、感知和导航研究。
