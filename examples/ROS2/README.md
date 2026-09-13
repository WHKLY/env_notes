# ROS2 Skywalker 示例学习路径

ROS2 示例直接运行在 Ubuntu 24.04 主机。系统层 `/opt/ros/jazzy` 提供 ROS 2 Jazzy、Gazebo Harmonic 和 ros_gz；`/home/xanter/ros2_ws/install` 提供 ArduPilot ROS、SITL、模型与 Micro XRCE-DDS Agent；实验自己的 `.colcon/install` 只提供 `setuptest_bringup`。

先阅读[构建与代码改动](01-skywalker-circuit-build.md)，理解三层 overlay、临时 SDF 和 DDS/bridge 的职责；再按[运行与归档](02-skywalker-circuit-run.md)完成实际任务。

数据链如下：

~~~text
Gazebo Harmonic <-> ArduPilotPlugin <-> JSON UDP 9002 <-> ArduPlane SITL
Gazebo Transport -> ros_gz_bridge -> /clock /imu /odometry /tf
ArduPlane DDS client <-> UDP 2019 <-> Micro XRCE-DDS Agent -> /ap/*
MAVProxy -> UDP 14551 -> pymavlink 任务控制器
~~~

每个新 shell 必须先 source Jazzy，再 source公共工作区；实验启动脚本还会 source 私有 overlay。不要从家目录直接启动 SITL。
