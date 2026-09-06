# 智能体模式运行指南

本文介绍如何以**智能体模式（Agentic mode）**运行本仓库。在该模式下，语言模型负责规划任务，并调度已注册的机器人技能。

## 适用范围

智能体模式适用于以下场景：

- 自然语言任务拆解；
- 技能选择与执行；
- 单机器人或多机器人编排；
- 长时任务验证和试运行测试。

相关实现主要位于 `agentic_robot/agentOS/`。

## 关键目录

- `agentic_robot/agentOS/holoagent_skills/`：技能注册表、技能元数据、示例和增删改查工具；
- `agentic_robot/agentOS/run_dameon/`：后台守护进程辅助工具；
- `agentic_robot/agentOS/sandbox_test/`：长时任务规划和试运行脚本；
- `agentic_robot/services/src/robot_bridge/`：机器人侧 HTTP 到 ROS 桥接器；
- `agentic_robot/services/src/multi_robot_ctl/`：多机器人控制中心示例。

## 典型启动流程

1. 准备 ROS 2 环境并构建所需软件包。
2. 启动机器人侧 ROS 节点和硬件适配器。
3. 在每台机器人上启动 `robot_bridge`。
4. 如果需要多机器人编排，启动多机器人控制中心。
5. 配置 LLM 凭据和机器人服务端点环境变量。
6. 运行智能体模式测试或集成入口。

## 环境准备

环境变量示例：

```bash
export GPT_PROVIDER=azure
export AZURE_OPENAI_API_KEY=your_key
export AZURE_OPENAI_ENDPOINT=https://your-endpoint.openai.azure.com/
export AZURE_OPENAI_API_VERSION=2024-02-15-preview
export AZURE_OPENAI_DEPLOYMENT=your_deployment_name

export ROBOT_11_URL=http://192.168.124.101:8000
export ROBOT_12_URL=http://192.168.124.102:8000
export ROBOT_13_URL=http://192.168.124.103:8000
export MULTI_ROBOT_CONTROL_CENTER_URL=http://127.0.0.1:8080
```

## 推荐入口

### 单机器人长时任务试运行

```bash
python3 agentic_robot/agentOS/sandbox_test/test_single_robot_long_instruction.py
```

### 多机器人长时任务试运行

```bash
python3 agentic_robot/agentOS/sandbox_test/test_multi_robot_long_instruction.py
```

### 集成演示启动器

```bash
bash scripts/intergation/run_holoagent_pipeline.sh
```

## 相关文档

- [`README.md`](README.md)：仓库概览；
- [`README_demo.md`](README_demo.md)：演示模式运行指南；
- [`agentic_robot/agentOS/README.md`](agentic_robot/agentOS/README.md)：AgentOS 概览；
- [`agentic_robot/agentOS/holoagent_skills/README.md`](agentic_robot/agentOS/holoagent_skills/README.md)：技能系统文档。
