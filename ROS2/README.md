# 本机 ROS 2 仿真环境

> 快照日期：2026-09-13（Asia/Shanghai）
>
> 状态：ROS 2 Jazzy、Gazebo Harmonic、ArduPilot SITL、Micro XRCE-DDS Agent、ROS–Gazebo 桥和 RViz 已形成可工作的联合仿真链。Iris 多旋翼和 Alti Transition QuadPlane 均完成联合启动验证；Skywalker X8 普通固定翼已完成手动解锁、AUTO 起飞、绕场、自动降落和自动解除武装。

## 文档

| 文件 | 内容 |
|---|---|
| [01-environment.md](01-environment.md) | 系统版本、路径、ROS 包和源码快照 |
| [02-architecture.md](02-architecture.md) | 架构、数据流、模型和启动次序 |
| [03-build-and-fix.md](03-build-and-fix.md) | 构建、当前本机补丁和验证结果 |
| [04-runbook.md](04-runbook.md) | GNOME Terminal 启动、固定翼起飞和停止 |
| [05-interfaces.md](05-interfaces.md) | ROS 话题、服务、桥接和端口 |
| [06-troubleshooting.md](06-troubleshooting.md) | 已知问题和排查步骤 |
| [07-maintenance.md](07-maintenance.md) | 更新、日志、缓存和回归检查 |
| [tips.md](tips.md) | 本机实际踩坑、原因、修复和防复发规则 |

## 每个新终端先加载

当前 ~/.bashrc 没有自动加载这套环境：

~~~bash
source /opt/ros/jazzy/setup.bash
source /home/xanter/ros2_ws/install/setup.bash
~~~

直接使用 MAVProxy 时再执行：

~~~bash
source /home/xanter/venv-ardupilot/bin/activate
~~~

## 入口

~~~bash
# Iris
ros2 launch ardupilot_gz_bringup iris_runway.launch.py   rviz:=true use_gz_tf:=true use_instance_dir:=True

# Alti Transition QuadPlane
ros2 launch ardupilot_gz_bringup alti_transition_runway.launch.py   rviz:=true use_instance_dir:=True
~~~

日常启动请使用 [04-runbook.md](04-runbook.md) 的 GNOME Terminal 命令。它会创建独立运行目录，避免 mav.tlog、mav.parm、terrain 和飞行日志出现在家目录。

## 关键目录

| 用途 | 路径 |
|---|---|
| ROS 工作区 | /home/xanter/ros2_ws |
| 工作区安装 | /home/xanter/ros2_ws/install |
| ArduPilot | /home/xanter/ros2_ws/src/ardupilot |
| ardupilot_gz 实体仓库 | /home/xanter/Projects/ardupilot_gz |
| ardupilot_gz 工作区链接 | /home/xanter/ros2_ws/src/ardupilot_gz |
| MAVProxy 虚拟环境 | /home/xanter/venv-ardupilot |
| 推荐运行目录 | /home/xanter/sim_runs |

## 文档维护

本目录由上级 /home/xanter/Documents/env_notes 本地 Git 仓库管理。相关版本、路径或配置变化后，应更新对应文档并提交；仓库管理约定和历史记录见上级 [README.md](../README.md) 与 [CHANGELOG.md](../CHANGELOG.md)。
