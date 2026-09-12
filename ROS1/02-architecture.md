# 架构与边界

## 推荐方案

ROS1 使用完整的 Ubuntu 20.04 容器，容器内运行 Noetic、Gazebo Classic 11、gazebo_ros、MAVROS 和 ROS1 应用节点。ArduPilot SITL 运行在 Ubuntu 24.04 主机。两侧通过 host network 上的 UDP 通信。

这样可以保持 Noetic 与 Ubuntu 20.04 ABI 一致，也不会把 Gazebo Classic 库安装到 Ubuntu 24.04 主机。

## 数据流

~~~text
Gazebo Classic 11
  ├── legacy ArduPilotPlugin <-> UDP 9002/9003 <-> ArduPilot SITL
  └── gazebo_ros -> /clock、/gazebo、传感器和 TF

ArduPilot SITL/MAVProxy -> UDP 14550 -> MAVROS
MAVROS -> /mavros/... -> ROS1 控制、感知和记录节点
~~~

## 与 ROS2 链的关系

ROS1 与 ROS2 是两套并列实验链：

- ROS1：Noetic + Gazebo Classic 11 + gazebo_ros + MAVROS。
- ROS2：Jazzy + Gazebo Harmonic + ros_gz + ArduPilot DDS。

不要在同一个 shell 同时 source Noetic 与 Jazzy。不要让 Classic 的 GAZEBO_* 环境变量污染 Harmonic 的 GZ_* 环境变量。

## 版本策略

Noetic 和 Gazebo Classic 已属于遗留软件。基础镜像、插件提交和 apt 包版本必须记录并固定。现有镜像摘要作为第一版基线；在确认能完整构建和运行前，不删除历史镜像与容器。

## 图形与权限

- 通过 X11 socket 和只读 Xauthority 文件显示 Gazebo GUI。
- 不使用 xhost +。
- 容器用户采用宿主 UID/GID 1000。
- /dev/dri 映射给容器，先验证 Intel/DRI 渲染；需要 NVIDIA 专用运行时再单独增加。
- Docker 服务按需启动，不设置开机自启。
