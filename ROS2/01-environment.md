# 环境与版本

快照日期：2026-09-12。

## 系统和工具链

| 项目 | 当前值 |
|---|---|
| 系统 | Ubuntu 24.04.5 LTS |
| 内核 / 架构 | 7.0.0-31-generic / x86_64 |
| ROS 2 | Jazzy，ROS_VERSION=2 |
| Gazebo | Gazebo Sim 8.11.0，Harmonic 系列 |
| Python / colcon-core | 3.12.3 / 0.21.1 |
| MAVProxy / pymavlink | 1.8.74 / 2.4.49 |
| CMake / GCC | 3.28.3 / 13.3.0 |
| Java | OpenJDK 17.0.20 |
| Micro XRCE-DDS Gen | 官方 v4.7.1，f5b96777802b |

RMW_IMPLEMENTATION 未固定，使用 Jazzy 默认实现。系统没有独立 gradle 命令；DDS 生成器使用源码仓库的 Gradle Wrapper。microxrceddsgen 的 CLI 可能显示 version: null，这是上游打包元数据现象，本机源码标签和提交已核对。

## 程序位置

| 程序 | 路径 |
|---|---|
| ros2 | /opt/ros/jazzy/bin/ros2 |
| gz | /opt/ros/jazzy/opt/gz_tools_vendor/bin/gz |
| colcon | /usr/bin/colcon |
| arducopter | /home/xanter/ros2_ws/install/ardupilot_sitl/bin/arducopter |
| arduplane | /home/xanter/ros2_ws/install/ardupilot_sitl/bin/arduplane |
| micro_ros_agent | /home/xanter/ros2_ws/install/micro_ros_agent/lib/micro_ros_agent/micro_ros_agent |
| microxrceddsgen | /home/xanter/.local/bin/microxrceddsgen |
| mavproxy.py | /home/xanter/venv-ardupilot/bin/mavproxy.py |

## 包来源

工作区安装：ardupilot_msgs、ardupilot_sitl、ardupilot_gazebo、ardupilot_gz_application、ardupilot_gz_description、ardupilot_gz_gazebo、ardupilot_gz_bringup、ardupilot_sitl_models、micro_ros_agent、sdformat_urdf。

系统 Jazzy 安装：ros_gz_sim 1.0.24、ros_gz_bridge 1.0.24。src 中虽有 ros_gz 源码，当前这两个运行包来自 /opt/ros/jazzy；colcon 可能给出 underlay 覆盖提示。

## 源码快照

| 仓库 | 分支/标签 | 提交 | 状态 |
|---|---|---|---|
| ardupilot | master | b832113b10aa | 干净 |
| ardupilot_gz | main | 8df4dc1726e3 | 有一个本机 URI 修复 |
| ardupilot_gazebo | ros2 | cc0290d964df | 干净 |
| ardupilot_sitl_models | main | d5d6017367a5 | 干净 |
| microxrcedds_gen | v4.7.1 detached | f5b96777802b | 干净 |
| ros_gz | jazzy | 0fa70cb7c15f | 干净 |

ArduPilot describe：ArduPilot-4.6.0-beta1-8540-gb832113b10。

## 快速核对

~~~bash
source /opt/ros/jazzy/setup.bash
source /home/xanter/ros2_ws/install/setup.bash
ros2 pkg prefix ardupilot_gz_bringup
ros2 pkg prefix micro_ros_agent
ros2 pkg prefix ros_gz_bridge
gz sim --versions
command -v arduplane
~~~

前两个应位于工作区 install，ros_gz_bridge 应位于 /opt/ros/jazzy。第二次 source 同时加入 Gazebo models、worlds、系统插件和 SDF 资源路径。
