# 联合仿真示例

本目录把已经实际运行过的三个实验整理为可复现教程。阅读起点是假定本机的基础 ROS1 和 ROS2 环境已经构建好，但尚未创建具体实验。文档解释实验为什么这样分层、改了哪些代码、每个文件承担什么责任，以及怎样用运行证据判断成功。

## 建议学习顺序

认识联合仿真应从较少变量的链路验证开始，再增加任务控制与飞行闭环：

1. [ROS1 基础设施](ROS1/00-noetic-gazebo11-infra.md)：先理解为什么 Noetic 与 Gazebo Classic 放进容器，以及主机 SITL 如何跨容器通信。
2. [ROS1 Iris 基线](ROS1/01-iris-setuptest.md)：只验证物理、FDM、MAVLink、MAVROS、ROS 话题和 rosbag，不解锁。
3. [ROS1 Zephyr 构建](ROS1/02-zephyr-circuit-build.md)与[运行](ROS1/03-zephyr-circuit-run.md)：在已验证链路上增加普通固定翼模型转换、人工解锁、AUTO 航线和自动降落。
4. [ROS2 Skywalker 构建](ROS2/01-skywalker-circuit-build.md)与[运行](ROS2/02-skywalker-circuit-run.md)：理解 Gazebo Harmonic、JSON FDM、XRCE-DDS 和 ros_gz_bridge 的对应实现。

## 三个已验证示例

| 链路 | 实验 | 目的 | 成功运行 |
|---|---|---|---|
| ROS1 | Iris / ArduCopter | 最小联合链基线，不解锁 | `run_20260913T011015+0800` |
| ROS1 | Zephyr / ArduPlane | 人工解锁后自动起飞、绕场、降落 | `run_20260913T015818+0800` |
| ROS2 | Skywalker X8 / ArduPlane | DDS 与 ros_gz 下的同类固定翼闭环 | `run_20260912T235453+0800` |

实验定义分别保存在 `/home/xanter/ros1simulation` 和 `/home/xanter/ros2simulation`。`runs/` 中的内容是运行证据，不是模板源码。复现实验时总是创建新的 `run_时间戳`，不要覆盖上述成功 run。

## 共同判断方法

一个联合仿真至少有四条需要独立验证的链：

1. 物理链：Gazebo 已创建模型并持续输出状态。
2. 飞控链：ArduPilotPlugin 与 SITL 已交换 FDM 和执行器数据。
3. ROS 链：ROS1 的 MAVROS 或 ROS2 的 XRCE-DDS/bridge 已有真实消息。
4. 记录链：rosbag、日志、任务遥测与结束标记正常收尾。

看到 Gazebo 窗口只证明 GUI 启动。完整结果还要检查 `manifest.yaml`、关键就绪标记、`metrics/result.json`、闭合的 bag，以及停止后没有残留进程。

## 环境隔离

ROS1 使用容器内 Noetic 与 Gazebo Classic；ROS2 使用主机 Jazzy 与 Gazebo Harmonic。不要在同一个 shell 同时 source 两套 ROS 环境。所有可视进程使用 GNOME Terminal；SITL、MAVProxy 和 bag 的工作目录必须指向本次 run，避免 `mav.tlog`、`terrain` 和 `eeprom.bin` 落入家目录。
