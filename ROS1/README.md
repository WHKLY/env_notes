# 本机 ROS 1 联合仿真环境

> 快照日期：2026-09-13（Asia/Shanghai）
>
> 当前阶段：可复现的 Noetic、Gazebo Classic 11、MAVROS 与主机 ArduPilot SITL 联合链已经建立；Iris 基线完成动态连接、话题记录和一键停止归档验证，尚未进行解锁与起飞。

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

旧文档 [ROS1_Noetic_Docker_Environment.md](../../ROS1_Noetic_Docker_Environment.md) 保留为历史参考；与本目录冲突时以本目录的重新验证结果为准。

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

## 已准备的普通固定翼任务

`/home/xanter/ros1simulation/ros1sim_260913_zephyr_circuit` 已完成静态构建。任务使用 Zephyr 单推进器飞翼、ArduPlane `gazebo-zephyr` frame、Gazebo Classic 气动插件和 MAVROS；任务上传后等待操作者输入 `ARM`，随后自动起飞、飞左矩形航线并自动降落。

该任务截至 2026-09-13 尚未动态启动。首次实际运行必须检查 Classic 气动插件加载、滑跑方向、舵面响应和降落结果，不能仅凭静态检查标记为成功。

## 维护

本目录由 /home/xanter/Documents/env_notes 本地 Git 仓库管理。完成一次环境变更或验证后更新文档并提交。
