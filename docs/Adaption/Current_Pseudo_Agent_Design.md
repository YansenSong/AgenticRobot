# 当前“类 Agent”设计源码记录

## 1. 记录范围

本文只记录当前仓库中已经存在的、与“Agent 式任务理解和执行”有关的源码结构与行为。

记录范围主要包括：

- agentic_robot/agentOS/sandbox_test/long_horizon_text_runner.py
- agentic_robot/agentOS/holoagent_skills/
- agentic_robot/services/src/robot_bridge/
- agentic_robot/services/src/multi_robot_ctl/
- agentic_robot/core/src/navigation/nav_executor/
- agentic_robot/chatbot/g1/ 中与命令、信号和 ROS 桥接相关的代码

本文不描述未来改造方案，也不将目录名或说明文档当作运行时能力依据。

## 2. 当前设计的总体性质

当前项目存在“模型理解指令、选择预定义能力、调用机器人接口、等待执行结果”的链路，但没有一个统一、持续运行的 Agent Runtime。

从源码职责看，当前结构更接近：

~~~text
Skill 文件与独立请求脚本
        +
LLM 一次性生成受限任务 DAG
        +
确定性 DAG 调度器
        +
HTTP / ROS 机器人服务
        +
导航到点回调与聊天信号
~~~

其中，最接近 Agent 执行器的实现是 LongHorizonTextRunner；它是一个可直接运行的长指令 DAG 规划、验证和执行程序，而不是一个常驻的通用 Agent Runtime。

## 3. 模块组成

### 3.1 Skill 目录

路径：

~~~text
agentic_robot/agentOS/holoagent_skills/skills/
~~~

当前存在的 Skill 目录包括：

~~~text
arm-skill
rel-move-skill
robot-service
sem-nav-skill
workflow
~~~

每个 Skill 通常包含：

~~~text
SKILL.md
README.md
scripts/
assets/
~~~

可执行脚本承担的是具体 HTTP 请求构造与发送。例如：

- rel-move-skill/scripts/relative_move.py 向 /api/relative_nav 发送相对移动请求。
- sem-nav-skill/scripts/semantic_nav.py 向 /api/semantic_nav 发送语义导航请求。
- arm-skill/scripts/trigger_arm_skill.py 向 /api/arm/{skill} 发送动作请求。
- robot-service/scripts/service_request.py 可向指定 HTTP endpoint 发送请求。

Skill 列举逻辑位于：

~~~text
agentic_robot/agentOS/holoagent_skills/scripts/list_skills.py
~~~

该脚本扫描包含 SKILL.md 的目录并打印名称、描述及目录信息。当前源码中未见它将 Skill 自动转换为模型 tool schema、自动执行模型 tool call 或维护运行时 Tool Registry。

### 3.2 长指令 DAG 执行器

路径：

~~~text
agentic_robot/agentOS/sandbox_test/long_horizon_text_runner.py
~~~

类：

~~~text
LongHorizonTextRunner
~~~

该程序的主要职责如下：

1. 接收单机或多机的自然语言长指令。
2. 调用 OpenAI 或 Azure OpenAI 模型。
3. 要求模型一次性返回固定 JSON 格式的 DAG。
4. 校验 DAG 中的 robot_id、skill、target、depends_on 和依赖环。
5. 对 DAG 做虚拟执行校验。
6. 使用线程池按依赖关系执行已就绪的 DAG 节点。
7. 将规划、验证、执行结果和监控事件写入 YAML 文件。

当前 DAG 节点的核心字段是：

~~~json
{
  "id": "r11_nav_p1",
  "robot_id": 11,
  "skill": "navigation",
  "target": "one_point_1",
  "depends_on": []
}
~~~

当前源码中允许的 skill 是固定集合：

~~~text
navigation
arm
~~~

目标也由代码中的白名单限制。模型输出不是开放式的工具调用，而是受固定 JSON 格式、固定 skill 名称和固定 target 白名单约束的 DAG。

### 3.3 DAG 校验与调度

LongHorizonTextRunner 在接收到模型输出后执行以下确定性逻辑：

~~~text
LLM JSON
  -> 字段与白名单校验
  -> 检查单机/多机约束
  -> 检查依赖是否存在
  -> 检查 DAG 是否有环
  -> 虚拟执行验证
  -> 依赖满足的节点并发执行
  -> 成功记录 completed
  -> 失败记录 failed，并停止后续执行
~~~

真实执行阶段使用 ThreadPoolExecutor 并发运行无依赖冲突的节点。每个节点的执行分支由 skill 字段决定：

~~~text
navigation
  -> 控制中心 navigation HTTP 调用
  -> 轮询到点状态

arm
  -> 机器人 /api/arm/{target} HTTP 调用
  -> 固定等待时间
~~~

执行过程会记录 node_execution_started、node_execution_finished、navigation_wait_started 等监控事件，并写入 monitor.yaml、dag_plan.yaml、virtual_validation.yaml、execution_result.yaml。

### 3.4 机器人服务层

路径：

~~~text
agentic_robot/services/src/robot_bridge/
~~~

robot_bridge 根据 bridge_config.yaml 动态创建 HTTP 到 ROS topic 的映射。

当前配置中包含的典型接口包括：

~~~text
POST /api/navigation/{name}
POST /api/relative_nav
POST /api/semantic_nav
POST /api/arm/{skill}
POST /api/navigation/stop
~~~

robot_bridge 也支持 ROS 到 HTTP 的反向回调。当前配置订阅 waypoint_reached，并向控制中心发送机器人到点信息。

因此，robot_bridge 是当前任务执行链中的协议适配层，而不是任务规划器。

### 3.5 多机器人控制中心

路径：

~~~text
agentic_robot/services/src/multi_robot_ctl/
~~~

LongHorizonTextRunner 会向控制中心触发命名导航目标，并通过控制中心接口轮询机器人是否到达。

当前长指令执行器中的导航过程为：

~~~text
DAG navigation node
  -> 控制中心 /trigger/{target}?robot_id={id}
  -> robot_bridge
  -> chat_signal_pub
  -> nav_executor
  -> Nav2
  -> waypoint_reached
  -> robot_bridge 反向 HTTP 回调
  -> 控制中心到点状态
  -> LongHorizonTextRunner 轮询确认
~~~

### 3.6 导航执行器与执行反馈

路径：

~~~text
agentic_robot/core/src/navigation/nav_executor/nav_executor/pubpose.py
~~~

nav_executor 的主要职责包括：

- 订阅 chat_signal_pub。
- 根据机器人和地图的 signal registry 解析命名目标。
- 调用 Nav2 BasicNavigator。
- 在导航成功、卡住或语义导航结束时发布 waypoint_reached。

当前 waypoint_reached 是字符串 topic。其消息内容可能是航点名称、nav_finish 或 struck。

该 topic 同时被聊天模块、robot_bridge 反向回调以及原有 G1 动作链路使用。因此它是当前系统的执行结果/状态反馈通道，而不是专门的 Agent observation 接口。

### 3.7 Chatbot 命令与信号链路

路径：

~~~text
agentic_robot/chatbot/g1/
~~~

g1.py 实现语音输入、ASR、LLM、TTS 和队列处理。其控制行为中包含确定性 Hook：

~~~text
ASR 文本
  -> wake/sleep hook
  -> asr hook 匹配
  -> 命中时产生 response 与 signal
  -> 未命中时进行 LLM 处理
~~~

g1chat_node.py 将 g1.py 的文本队列转换为 ROS topic：

~~~text
user:/assistant: -> chat_qa_pub
location:        -> chat_loc_pub
signal:          -> chat_signal_pub
~~~

它还订阅 waypoint_reached，并将收到的内容放回聊天模块的 control_queue，用于到点后的播报控制。

当前 signal 以字符串编码，例如：

~~~text
signal:stop
signal:wave_under_head
signal:custom_one_point_1_0.0_0.0_0.0
~~~

## 4. 当前实际执行闭环

### 4.1 长指令 DAG 闭环

~~~text
自然语言长指令
  -> LongHorizonTextRunner 调用模型
  -> 模型一次性返回固定 JSON DAG
  -> 本地校验与虚拟验证
  -> DAG 调度器执行 HTTP 请求
  -> 机器人服务与导航系统执行
  -> 控制中心记录到点状态
  -> Runner 轮询结果
  -> 写入 YAML 监控与执行结果
~~~

这个闭环包含规划、执行和外部状态确认，但模型只参与 DAG 的一次性生成。

### 4.2 语音命令闭环

~~~text
麦克风
  -> ASR
  -> Hook 匹配或 LLM
  -> g1.py text_queue
  -> g1chat_node
  -> chat_signal_pub / chat_loc_pub
  -> nav_executor 或语义导航模块
  -> waypoint_reached
  -> g1chat_node
  -> g1.py control_queue
  -> TTS 播报
~~~

这个闭环包含确定性规则、聊天模型、ROS topic 和导航结果反馈。

## 5. 当前设计具备的 Agent 式特征

从现有源码可确认的特征包括：

- 使用模型把自然语言指令转成可执行结构。
- 使用 Skill 名称和 HTTP/ROS 接口表达预定义机器人能力。
- 将长任务表示为存在依赖关系的 DAG。
- 能够对无依赖冲突的任务并行执行。
- 能够校验 DAG 结构、节点依赖和目标白名单。
- 能够等待导航完成并将结果记录到监控文件。
- 能够通过 waypoint_reached 将执行结果传播给控制中心和聊天模块。
- 对一部分语音命令优先使用确定性 Hook，而不总是调用模型。

## 6. 当前设计不具备的统一 Agent Runtime 机制

从当前源码中未发现下列统一运行时机制：

### 6.1 中心化常驻 Agent

未发现一个长期维护任务状态、会话状态、机器人状态和工具状态的统一 Agent 进程。

LongHorizonTextRunner 是命令行式执行器；g1.py 是聊天与语音服务；robot_bridge 是协议桥接器；它们不是同一个统一 Runtime。

### 6.2 模型原生 Tool Calling

未发现将 Skill 自动注册为模型 tools/function schema，并接收模型 tool call 后自动分发执行的实现。

当前模型输出是 JSON DAG；Skill 脚本和 HTTP endpoint 由程序中的固定逻辑调用。

### 6.3 ReAct 式反复推理与行动

未发现如下循环：

~~~text
模型推理
  -> 选择工具
  -> 执行工具
  -> 获取 observation
  -> 将 observation 回传模型
  -> 模型决定继续、恢复、换工具或结束
~~~

当前 LongHorizonTextRunner 在执行失败时记录 failed 并停止后续执行；外部执行结果不会作为新的模型输入触发自动重规划。

### 6.4 通用 Observation 模型

当前存在 waypoint_reached、控制中心轮询、HTTP 成功/失败和 YAML 监控记录，但未发现统一的 Observation 类型、状态存储或面向模型的观察上下文。

### 6.5 通用 Tool 生命周期与策略控制

未发现统一处理下列内容的运行时层：

- Tool schema 注册和版本管理。
- Tool 调用授权与权限边界。
- 参数校验的统一入口。
- 重试、超时、取消和补偿策略。
- 多工具执行结果汇总。
- 工具错误恢复与重新规划。
- 长期记忆或任务状态持久化后恢复。

## 7. 当前设计的名称定位

仅依据当前源码，可将项目中的该部分描述为：

> 受限技能集上的 LLM DAG 规划与确定性机器人工作流执行系统。

其中：

~~~text
Skill 目录：能力说明与独立调用脚本
LongHorizonTextRunner：一次性 DAG 规划、校验、调度与监控
robot_bridge：HTTP / ROS 协议适配
nav_executor：导航执行与 waypoint_reached 反馈
g1.py / g1chat_node：语音、规则 Hook、LLM、TTS 与 ROS 信号桥接
~~~

它包含多个 Agent 式组件，但当前源码尚未形成“模型 + 循环 + 动态 tools + observation + 再规划”的统一 Agent Runtime。
