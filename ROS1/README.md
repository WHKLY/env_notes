# 本机 ROS 1 联合仿真环境

> 快照日期：2026-09-13（Asia/Shanghai）
>
> 当前阶段：可复现的 Noetic、Gazebo Classic 11、MAVROS 与主机 ArduPilot SITL 联合链已经建立；Iris 基线完成连接与归档验证，Zephyr 普通固定翼已完成实际人工解锁、自动起飞、绕场、降落和自动解除武装。

## 当前已确认

| 项目 | 状态 |
|---|---|
| 主机 | Ubuntu 24.04 |
| Docker Engine | 29.1.3，服务默认 disabled，测试后已恢复 inactive |
| 历史镜像 | osrf/ros:noetic-desktop-full-focal，摘要 sha256:f138c82f... |
| Docker Compose | 2.40.3 |
| 历史容器 | ros1_noetic，host network，保持停止 |
| 容器系统 | Ubuntu 20.04.6 LTS |
| ROS | Noetic，roslaunch 1.17.4 |
| Python | 3.8.10 |
| Gazebo | Classic 11.15.1 |
| ROS 集成 | gazebo_ros 已安装 |
| 飞控桥 | MAVROS 已安装 |
| ArduPilot | 主机 /home/xanter/Projects/ardupilot |
| 实验根目录 | /home/xanter/ros1simulation |

## 文档

- [01-environment.md](01-environment.md)：本机事实、路径和已知版本。
- [02-architecture.md](02-architecture.md)：推荐联合链与隔离边界。
- [03-runbook.md](03-runbook.md)：构建、启动、验证和停止顺序。
- [04-troubleshooting.md](04-troubleshooting.md)：当前已知风险和排查入口。
- [tips.md](tips.md)：本次构建和动态联调中实际遇到的问题。
- [../examples/ROS1/README.md](../examples/ROS1/README.md)：infra、Iris 基线与 Zephyr 全任务的可复现教程。


## 共享基础设施

`/home/xanter/ros1simulation/infra/noetic_gazebo11` 保存固定摘要 Dockerfile、Compose、入口脚本、legacy 插件 vendor 快照、GNOME Terminal 构建/doctor/shell 脚本及可再生成的 runtime 证据。派生镜像为 `xanter/ros1-noetic-gazebo11:20260913`。详细文件职责和复现步骤见 [ROS1 infra 示例](../examples/ROS1/00-noetic-gazebo11-infra.md)。

实验目录与 infra 分开：infra 是多个 ROS1 实验共享的软件栈，`ros1sim_*` 保存单个实验定义和每次 run。

## 目标链路

~~~text
主机 ArduPilot SITL
  ├── UDP 9002/9003 <-> 容器 Gazebo Classic 11 + legacy ArduPilotPlugin
  └── MAVLink UDP 14550 -> 容器 MAVROS -> ROS Noetic topics

容器 Gazebo -> gazebo_ros -> /clock、传感器、TF
容器 ROS 节点 -> rosbag、任务控制与指标
~~~

## 已验证基线

2026-09-13 的 run_20260913T011015+0800 已确认 Gazebo 时钟、Iris 模型、MAVROS connected、IMU、全局位置、本地位置和 rosbag。stop.sh 正常停止全部组件并自动写 DONE；测试未解锁。

派生镜像：xanter/ros1-noetic-gazebo11:20260913，镜像 ID sha256:f7b402cf06d7a39a86603eeb2cf0285017d6d4694d154de0900557fc4ee0a03b。

## 已验证的普通固定翼任务

`/home/xanter/ros1simulation/ros1sim_260913_zephyr_circuit` 使用 Zephyr 单推进器飞翼、ArduPlane `gazebo-zephyr` frame、Gazebo Classic 气动插件和 MAVROS；任务上传后等待操作者输入 `ARM`，随后自动起飞、飞左矩形航线并自动降落。

动态验证 `run_20260913T015818+0800` 已成功完成全流程。任务用时 290.19 秒，最高相对高度 60.35 m，接地后自动解除武装；rosbag 正常闭合为 192,488,211 字节，停止退出码为 0，所有进程与 Docker service/socket 均已清理。

## 维护

本目录由 /home/xanter/Documents/env_notes 本地 Git 仓库管理。完成一次环境变更或验证后更新文档并提交。
