# 演示模式运行指南

本文介绍如何以**演示模式（Demo mode）**运行本仓库。在该模式下，通过预定义脚本和已知机器人工作流进行演示、验证和操作员辅助执行。

## 适用范围

演示模式适用于以下场景：

- 脚本化演示；
- 集成验证；
- 操作员辅助的机器人执行；
- 可重复的展示场景。

与智能体模式不同，演示模式不以动态技能规划作为主要控制路径。

## 常用工作流脚本

- `scripts/intergation/run_holoagent_workflow.sh`
- `scripts/intergation/run_holoagent_running_docker.sh`
- `scripts/intergation/kill_all.sh`
- `scripts/audio/start_audio_ctl_demo.sh`

## 典型工作流

1. 启动所需容器或运行环境。
2. 启动感知、导航和机器人侧服务。
3. 启动所需的音频或聊天机器人组件。
4. 运行演示启动脚本。
5. 监控日志，完成后使用清理工具停止所有进程。

## 示例命令

启动演示系统：

```bash
bash scripts/intergation/run_holoagent_workflow.sh
```

在已运行的 Docker 环境中启动演示系统：

```bash
bash scripts/intergation/run_holoagent_running_docker.sh
```

停止所有集成进程：

```bash
bash scripts/intergation/kill_all.sh
```

## 注意事项

- 演示脚本与运行环境紧密相关，可能依赖固定的机器人 IP、地图和硬件设备。
- 部分演示要求在启动前准备好音频设备、GPU 推理环境和机器人侧 ROS 节点。
- 如果需要动态技能规划和长时任务执行，请使用智能体模式。

## 相关文档

- [`README.md`](README.md)：仓库概览；
- [`README_agent.md`](README_agent.md)：智能体模式运行指南。
