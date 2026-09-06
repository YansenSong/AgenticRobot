# HoloAgent：从 Unitree G1 清理到 Ackermann 移动机器人的迁移计划

## 1. 目标与边界

本计划的第一目标，是将工程从“包含 G1 人形机器人实现”的状态，整理为“保留通用 Agentic Robot 能力、可接入 Ackermann 移动底盘”的状态。

这不是简单删除所有出现 G1 字样的文件，也不是立刻实现 Ackermann 驱动。正确的原则是：

1. 删除 G1 人形手臂、手势、全身动作及其全部上游入口。
2. 保留可迁移的系统边界、能力抽象、事件反馈、确定性控制和机器人配置分层。
3. 保留移动底盘所需的上层接口；在 Ackermann 实现接管前，不删除当前仍在承担接口职责的 G1 实现。
4. 不复制 G1 的厂商 SDK、动作 ID、FIFO 协议和经验参数；仅复用经源码验证的架构思想，必要时将通用部分重构为新实现。
5. 本阶段不把 robots/hexfellow 当作人形代码删除；它属于另一种移动平台，后续单独审查。

最终的判断标准不是“目录里没有 g1”，而是：

~~~text
Agent、聊天、导航、定位的活动执行路径
不再知道 Unitree、G1 手臂、手势或 Action ID；
机器人型号差异只存在于配置、状态适配器和底盘驱动层。
~~~

## 2. 从源码得到的可保留设计

### 2.1 能力 Skill 架构：保留架构，删除 Arm Skill

当前 arm-skill 的问题不是“Skill 机制”，而是其能力模型绑定了挥手、拥抱、击掌等人形手臂动作。其可执行脚本 trigger_arm_skill.py 本质是高层能力调用到 HTTP 服务的适配器。

应保留的目标边界：

~~~text
Agent
  -> capability skill
  -> robot service / ROS Action
  -> 导航、对接、巡检等子系统
  -> 底盘或传感器执行
~~~

Agent 不应直接面向转向角、CAN ID、串口报文或电机速度。未来应以能力作为工具边界，例如：

- navigation-skill：go_to_lab、go_to_waypoint
- docking-skill：dock_to_charger、undock
- patrol-skill：start_patrol、pause_patrol
- follow-skill：follow_target、stop_following
- return-home-skill：return_home
- stop-skill：stop_robot、cancel_navigation
- relocalize-skill：relocalize
- inspection-skill：start_inspection、capture_and_report

处理策略：

1. 先从 arm-skill 中提取可复用的 HTTP 请求骨架或将其重写为 capability request helper。
2. 删除 arm-skill 的名称、手势参数、README、示例和 API 路径。
3. 保留并修正通用 robot-service Skill，使其只描述当前实际支持的移动机器人能力。
4. 高优先级安全能力，例如 stop_robot，不应依赖普通的 LLM Tool 调用才生效。

当前源码对应：

- agentic_robot/agentOS/holoagent_skills/skills/arm-skill/scripts/trigger_arm_skill.py
- agentic_robot/agentOS/holoagent_skills/skills/robot-service/
- agentic_robot/services/src/robot_bridge/

### 2.2 确定性语音指令优先于 LLM：保留快速路径

g1.py 已经具备有价值的执行顺序：

~~~text
ASR
  -> wake/sleep hook 与 asr hook 匹配
  -> 命中：直接产生控制结果
  -> 未命中：再进入 LLM 语义理解或任务规划
~~~

这比让“停车”一类指令绕经 LLM 更适合移动机器人。后续应保留该规则优先级，但将当前 signal:xxx 字符串总线改为明确的命令模型。

第一批建议保留为确定性命令的语义：

- 停止、取消导航、继续
- 回家、回充、开始/暂停巡检
- 重新定位
- 进入/离开特定运行模式

目标接口：

~~~text
ASR
  -> Command Rule Matcher
  -> typed SafetyCommand / NavigationCommand / RobotCapability
  -> ROS Service 或 ROS Action
~~~

注意：

- 普通 stop_robot 可作为高优先级软件停车命令。
- emergency_stop 不能只依赖语音、网络或普通 ROS topic；最终必须具备底盘控制器/硬件安全链路。
- 当前 chat_signal_pub 可作为迁移期间兼容层，但不应成为最终的“字符串大杂烩总线”。

### 2.3 waypoint_reached：保留到点事件，删除到点挥手

导航执行器在导航成功后发布 waypoint_reached；robot_bridge 已可订阅该 topic 并向控制中心进行 HTTP 回调。这条“执行结果反馈”链路应保留。

应删除的旧链路：

~~~text
waypoint_reached
  -> g1_arm
  -> G1 手势 Action
~~~

应保留并演进为：

~~~text
Navigation Action Result / NavigationEvent
  -> Agent 状态机
  -> Chatbot 播报
  -> 控制中心
  -> 巡检、拍照、传感器采集等后续能力
~~~

过渡阶段可以保留 waypoint_reached（std_msgs/String），以免打断当前 robot_bridge 反向上报。最终建议采用带有 goal_id、capability、status、timestamp、message 的类型化结果消息，或直接消费 Nav2 Action 的结果。

### 2.4 算法代码与机器人配置分层：扩大现有 Fast-LIVO 模式

Fast-LIVO 中已有 mid360_online_reloc_g1.yaml 与 mid360_online_reloc_hexfellow.yaml，说明定位算法与机器人外参、地图、初始化参数已经部分解耦。应保留算法，扩大这种分层。

建议目标结构：

~~~text
config/
  common/
    navigation.yaml
    robot_service.yaml
    safety.yaml
  unitree_g1/                 # 迁移期存在，最终删除
    localization.yaml
    chassis.yaml
  hexfellow/                  # 本阶段不删除
    localization.yaml
  ackermann/
    localization.yaml
    lidar.yaml
    imu.yaml
    chassis.yaml
    navigation.yaml
    safety.yaml
~~~

迁移原则：

- Fast-LIVO、Nav2、感知算法代码不因 G1 命名而删除。
- G1 的外参、地图路径、初始位姿和传感器参数不能直接复制给 Ackermann。
- Ackermann 配置经过真实标定和运行验证后，才删除 G1 配置与 G1 启动入口。

### 2.5 /cmd_vel 到硬件驱动：保留边界，不复用 G1 协议

g1_move/getvel.cpp 从标准 geometry_msgs/Twist 的 /cmd_vel 接收上层速度；pubvel.cpp 才调用 Unitree G1 LocoClient。这种“上层不依赖厂商协议、最下层才接厂商驱动”的边界应保留。

Ackermann 的目标链路：

~~~text
Nav2 / capability executor
  -> /cmd_vel
  -> Command Validator
  -> Ackermann Kinematics
  -> Constraint Limiter
  -> Ackermann Chassis Driver
  -> CAN / Serial / Ethernet / 现有 ROS Driver
~~~

需要明确：

- Ackermann 不支持横向速度。现有 getvel.cpp 会透传 linear.y；新适配器必须拒绝、归零或诊断该值。
- angular.z 不能直接等同前轮转角，需要根据轮距、轴距、速度和曲率换算。
- 不能保留 /tmp/vel_fifo 作为最终架构，也不能复制 Unitree LocoClient 调用。

### 2.6 速度保护思想：保留为独立安全层，不复用参数

pubvel.cpp 中存在小角速度补偿与大转向时前进速度限制。这些数值是 G1 经验参数，不能用于 Ackermann；同时该程序没有可靠的命令超时归零机制。

Ackermann Base Adapter 必须拥有独立、可配置、可测试的约束层：

~~~text
最大前进速度
最大倒车速度
最大转向角
最大转向角速度
最大加速度与减速度
最小转弯半径
曲率限制
车速-转向联动限制
输入消息合法性检查
命令超时 -> 输出零车速
~~~

所有限值必须由 Ackermann 实车参数、轮胎附着、控制器能力和安全测试确定，不得沿用 G1 的 0.1、0.3、0.22 等常量。

## 3. 源码中的边界与分类

### 3.1 必须删除的人形能力实现及全部入口

| 范围 | 当前源码位置 | 处理 |
| --- | --- | --- |
| G1 手臂执行 | robots/unitree/src/g1_arm/ | 删除整个 ROS package |
| 手臂启动 | robots/unitree/scripts/run_armctl.sh | 删除 |
| Arm Skill | agentOS/holoagent_skills/skills/arm-skill/ | 提取通用请求骨架后删除整个目录 |
| Arm HTTP 入口 | robot_bridge/config/bridge_config.yaml 的 /api/arm/{skill} | 删除 |
| 导航手臂分发 | navigation/nav_executor/nav_executor/pubpose.py 的 arm_skill 和 arm_signal_pub | 删除 arm 分支与 publisher，保留导航分支 |
| 手势信号配置 | robots/unitree/config/signals_common.yaml | 删除 arm_skill schema 与所有手势 signal |
| 聊天动作 Hook | chatbot/g1/g1.json | 删除挥手、动作表演及无后端消费的旧动作 hook |
| 单机动作 Demo | chatbot/g1/g1chat_demo.py | 删除或改成导航-only Demo；不得保留 motion_tracking 动作白名单 |
| 多机动作 DAG | chatbot/g1/g1chat_demo_multirobot_dag.py | 删除 arm skill 分支、动作样例、HTTP 调用；或删除此 Demo |
| 长指令动作 DAG | agentOS/sandbox_test/long_horizon_text_runner.py | 删除 arm skill、动作目标、提示词、校验、执行函数和测试数据 |
| 通用服务 Skill 中的手臂声明 | robot-service/SKILL.md、assets/service_examples.sh、service_request.py | 删除 Arm API 声明与示例 |
| 旧映射 | holoagent_skills/scripts/sync_legacy_docs.py | 删除 ArmSkill 映射 |
| 已提交动作输出 | agentOS/sandbox_test/output/ | 删除或用导航-only 输出重新生成 |

清理后的约束是：任何活动源码、Skill 定义、配置、测试样例、已提交输出中都不能再产生 arm、gesture、motion_tracking、/api/arm、arm_signal_pub、G1ArmActionClient 或 /tmp/arm_fifo。

### 3.2 必须保留并去 G1 化的通用能力

| 范围 | 当前源码位置 | 处理 |
| --- | --- | --- |
| ASR、LLM、TTS、聊天队列 | chatbot/g1/g1.py 与 audio/ | 保留；改名为通用聊天模块；将字符串 signal 逐步替换为 typed command/event |
| ROS 聊天桥接 | chatbot/g1/g1chat_node.py | 保留 QA、位置、控制反馈；改名并改用新命令/事件接口 |
| HTTP/ROS Bridge | services/src/robot_bridge/ | 保留动态 topic/HTTP 与反向回调能力；删除 arm endpoint |
| waypoint 执行与反馈 | core/src/navigation/nav_executor/ | 保留导航、到点事件、控制中心上报；删除 arm dispatch；去掉默认 g1 配置 |
| Fast-LIVO 算法 | core/src/fast_livo/ | 保留；新增 Ackermann 参数和启动入口 |
| 感知算法 | core/src/perception/ | 保留；G1PerceptionNode、g1_perception.launch.py 等仅做重命名/参数化，不作为人形功能删除 |
| Skill 仓库机制 | agentOS/holoagent_skills/scripts/ | 保留目录扫描、校验和通用 Skill 架构 |

### 3.3 必须“替换后删除”的 Unitree 移动平台实现

| 当前实现 | 仍承担的职责 | 删除前条件 |
| --- | --- | --- |
| robots/unitree/src/g1_move/ | /cmd_vel 到 Unitree 底盘运动 | Ackermann Base Adapter 已接管且经实车验证 |
| robots/unitree/src/robot_odom/ | Unitree SportModeState 到 /robot_odom、/robot_imu | 新状态适配器和实际消费者重映射完成 |
| run_velctl.sh、run_ctl.sh、run_sensor_ctl.sh | G1 速度/手臂/传感器组合启动 | 新 launch 或 Ackermann 启动方式存在 |
| run_sensors.sh | 启动 Unitree robot_odom | 新状态源接管 |
| run_localization.sh | 启动 G1 Fast-LIVO 重定位配置 | Ackermann 定位配置可运行 |
| run_nav.sh | 启动 g1 Nav2 配置与旧 signals | Ackermann Nav2 和配置目录可运行 |
| init_env.sh | source Unitree workspace 与 chatbot/g1 环境 | 通用环境与新机器人环境已迁出 |

### 3.4 不应直接作为“人形代码”删除的内容

- robots/hexfellow：另一种移动平台；本轮不删除。
- agentic_robot/fsr_vln 中的 g1 数据集、场景、离线实验配置：先按数据来源和相机标定分类，不能仅凭名称删除算法。
- Fast-LIVO 中的 G1 注释、历史外参和 g1 评估名称：随配置迁移逐步清理，不删除算法。
- Nav2、感知中 G1 命名的 launch、类名、默认参数：保留能力，后续改为通用或 Ackermann 名称。

## 4. 当前源码中必须补齐的运行入口

仅删除 g1_arm 会使下列入口失效，因此它们必须纳入同一迁移清单：

~~~text
agentic_robot/agentOS/run_dameon/run_g1_background_daemon.py
robots/unitree/scripts/run_sensor_ctl.sh
robots/unitree/scripts/run_sensors.sh
robots/unitree/scripts/run_localization.sh
robots/unitree/scripts/run_nav.sh
robots/unitree/scripts/run_audio.sh
robots/unitree/scripts/run_sem_nav.sh
robots/unitree/scripts/run_nav_bridge.sh
scripts/intergation/run_holoagent_pipeline.sh
scripts/intergation/run_holoagent_running_docker.sh
scripts/intergation/kill_all.sh
scripts/audio/start_audio_ctl.sh
scripts/audio/start_audio_ctl_demo.sh
scripts/container/start_container.sh
scripts/build.sh
robots/unitree/build.sh
~~~

特别注意：

- run_sensor_ctl.sh 会启动 run_armctl.sh；它必须在第一阶段同步改造。
- run_g1_background_daemon.py 硬编码 robots/unitree/scripts；最终需改成通用后台服务启动器或 Ackermann 启动器。
- scripts/build.sh 与 robots/unitree/build.sh 仍引用 robots/g1，和实际目录 robots/unitree 不一致。该已有问题必须在最终构建清理时修复或废弃。
- scripts/container/start_container.sh 中包含 Unitree 主目录和 HoloMotion 挂载，必须在容器部署阶段重新审查。

## 5. 接口契约：不要凭名称假设消费者

当前状态 topic 的实际关系并不完全统一：

~~~text
robot_odom
  -> /robot_imu
  -> /robot_odom

Fast-LIVO LIVMapper
  <- /robot_odom
  <- 配置的 imu_topic（当前为 /livox/imu）

Nav2
  <- /odom
~~~

因此，不能只因为 /robot_imu 和 /robot_odom 是“标准消息类型”，就假设它们都是全部模块的统一契约。

在删除 robot_odom 之前，必须建立并验证下表：

| 生产者 | topic / action | 消费者 | Ackermann 替代来源 |
| --- | --- | --- | --- |
| 底盘编码器/控制器 | /odom | Nav2 等 | Ackermann chassis driver 或 state adapter |
| 底盘 IMU / LiDAR IMU | IMU topic | Fast-LIVO | 真实传感器驱动与标定 |
| 底盘里程计 | /robot_odom 或新约定 topic | Fast-LIVO wheel odom 相关路径 | Ackermann state adapter |
| 导航执行器 | waypoint_reached 或 NavigationEvent | Chatbot、Bridge、控制中心 | 保留并类型化 |

只有确认每个实际消费者完成重映射后，才能删除 Unitree SportModeState、/lf/odommodestate 和 robot_odom 包。

## 6. 推荐目标架构

~~~text
                  ┌──────────────────────────────┐
                  │            Agent             │
                  │  capability skills / planner │
                  └──────────────┬───────────────┘
                                 │
                 ┌───────────────┴────────────────┐
                 ▼                                ▼
       Deterministic Command Path          Capability Actions
       stop / cancel / relocalize          nav / dock / patrol / inspect
                 │                                │
                 └───────────────┬────────────────┘
                                 ▼
                     Typed ROS Command / Action
                                 │
              ┌──────────────────┴──────────────────┐
              ▼                                     ▼
         Navigation                            Robot Bridge
              │                                     │
              ▼                                     ▼
          /cmd_vel                       HTTP control-center callback
              │
              ▼
  Command Validator -> Ackermann Kinematics -> Constraint Limiter
              │
              ▼
       Ackermann Chassis Driver -> chassis controller

Navigation result
  -> NavigationEvent / temporary waypoint_reached
  -> Agent / Chatbot / control center / inspection workflow

IMU + wheel odom + LiDAR
  -> state adapter + Ackermann configuration
  -> Fast-LIVO / Nav2
~~~

## 7. 分阶段执行计划

### Phase 0：建立基线与接口清单

1. 不删除代码，先用 rg 和运行入口清点 G1、Unitree、arm、gesture、motion_tracking、/api/arm、arm_signal_pub 的全部活动引用。
2. 建立 topic、Service、Action 的生产者/消费者矩阵，特别是 /odom、/robot_odom、/robot_imu、/livox/imu、/cmd_vel、waypoint_reached。
3. 为清理后的行为建立最低验证用例：导航触发、停止、到点回调、聊天播报、控制中心回调。

### Phase 1：抽取通用能力接口，切断人形入口

1. 将 trigger_arm_skill.py 的通用请求模式迁入 capability helper 或 robot-service；不要保留 arm 名称。
2. 删除 arm-skill。
3. 删除 bridge_config.yaml 中的 /api/arm/{skill}。
4. 删除 robot-service 中 Arm API 的说明、示例和参数示例。
5. 删除 long_horizon_text_runner.py、g1chat_demo_multirobot_dag.py 中的 arm DAG 分支与提交输出。
6. 删除 g1chat_demo.py 中 motion_tracking 人形动作分支，或删除该 Demo。
7. 删除 g1.json 中 wave_under_head、watch_demo 等 G1 动作 Hook。
8. 删除 signals_common.yaml 中 arm_skill、挥手、握手等配置。
9. 从 nav_executor 删除 arm_signal_pub 与 arm_skill dispatch。

完成条件：上层 Agent、Chatbot、Bridge、导航配置均不能产生手臂动作请求。

### Phase 2：保留快速语音通道并类型化

1. 保留 g1.py 的 Hook 优先于 LLM 的处理顺序。
2. 将聊天目录、类名、节点名、配置文件从 g1 改为 robot_chat 或 common。
3. 将 signal:xxx 分类为：

   - SafetyCommand：stop_robot、cancel_navigation
   - NavigationCommand：go_to_named_goal、return_home
   - SystemCommand：relocalize、start_patrol、pause_patrol
   - NavigationEvent：succeeded、failed、cancelled、stuck

4. 新接口优先使用 ROS Service/Action 或类型化消息；保留 chat_signal_pub 仅作短期兼容。
5. 为 stop_robot 增加“不调用 LLM”的自动化验证。

### Phase 3：配置层去 G1 化

1. 新建 Ackermann 配置目录及 localization、lidar、imu、chassis、navigation、safety 配置。
2. 将 Fast-LIVO 的 mid360_online_reloc_g1.yaml 复制为 Ackermann 配置起点，但重新标定外参、地图、初始化参数和点云参数。
3. 新建 Ackermann Nav2 launch/parameter 文件；不再由 g1_navigation2_launch.py 作为默认入口。
4. 将 nav_executor 默认 robot_name 从 g1 改为显式参数或通用默认值，并将 signal registry 移到新机器人配置层。
5. 感知模块的 G1PerceptionNode、g1_perception.launch.py 等改为通用命名；不删除感知算法。

### Phase 4：实现 Ackermann Base Adapter

1. 保留 g1_move 直到新适配器能接管 /cmd_vel。
2. 实现 Command Validator、运动学转换、约束限制、底盘驱动和状态诊断。
3. 将 timeout -> zero speed 作为必备行为；对非法线速度、横向速度、曲率和转向速度给出明确拒绝或降级策略。
4. 在仿真、空载和受控实车阶段逐步验证后，删除 g1_move、run_velctl.sh 及 /tmp/vel_fifo。

### Phase 5：实现 Ackermann State Adapter

1. 接入真实底盘编码器、IMU、转向状态和必要的 TF。
2. 根据消费者矩阵提供 /odom、Fast-LIVO 所需 IMU 与 wheel odom。
3. 验证 Fast-LIVO、定位和 Nav2 后，删除 robot_odom、unitree_go、SportModeState、/lf/odommodestate。

### Phase 6：启动、构建、部署收口

1. 用通用或 Ackermann launch 替换 Unitree 脚本。
2. 更新后台 daemon、集成脚本、音频启动脚本、容器挂载、kill 脚本和构建分发脚本。
3. 修复或移除 scripts/build.sh 中 robots/g1 与实际 robots/unitree 的历史不一致。
4. 当没有活动引用后，删除 robots/unitree/、G1 chat 旧目录、G1 launch、G1 配置和旧环境目录。

## 8. 验收标准

### 人形能力已清除

- G1ArmActionClient、g1_arm、/tmp/arm_fifo、/api/arm、arm_signal_pub、arm-skill 在活动源码、配置、测试与提交输出中均为 0。
- Chatbot Demo、长指令 DAG、多机 DAG 不再接受或产生 arm、gesture、motion_tracking。
- signals_common 与 nav_executor 不再存在 arm_skill 分发。
- 删除 g1_arm 后，run_sensor_ctl、后台 daemon 和集成启动脚本不会尝试启动它。

### 通用能力被保留

- Agent 仍可通过 capability skill 调用导航等高层能力。
- “停止”能够在 ASR 命中 Hook 后直接进入高优先级命令路径，不调用 LLM。
- waypoint_reached 或其替代 NavigationEvent 能够使 Chatbot 和控制中心获知导航成功、失败或取消。
- robot_bridge 仍能提供导航/通用 HTTP 接口，并对控制中心发送到点回调。
- Fast-LIVO、导航、感知算法保留且不再依赖 G1 默认配置。

### Ackermann 替代完成后

- Base Adapter 对 /cmd_vel 执行有效性检查、Ackermann 运动学转换、车辆约束和命令超时停车。
- 新状态源满足已确认的 /odom、IMU、wheel odom 消费者。
- Ackermann 配置完成标定并成为默认启动路径。
- 所有构建、启动、守护、容器和集成入口不再引用 robots/unitree、robots/g1、Unitree SDK、G1 Chat 或 G1 launch。

## 9. 最终决策

本项目不应把 G1 代码“直接改名为 Ackermann”。应当：

~~~text
删除：G1 人形动作实现与其所有入口
保留：能力 Skill、确定性命令、执行结果事件、算法与配置分层、标准 ROS 边界
重写：Ackermann 底盘驱动、运动学、安全约束、状态适配器、机器人配置
最后删除：Unitree workspace、SDK 依赖、G1 启动与配置
~~~

这样得到的不是“删去一部分 G1 的工程”，而是一个以高层机器人能力为中心、能够长期承载 Ackermann 移动机器人的 Agentic Robot 基础。
