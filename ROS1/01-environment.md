# 环境事实

## 主机

- Ubuntu 24.04。
- Docker CLI 与 daemon 版本 29.1.3。
- Docker Compose v2：2.40.3+ds1-0ubuntu1~24.04.1。
- docker.service 默认 inactive、disabled，需要按需手动启动。
- 当前用户 xanter 不属于 docker 组，因此使用可见终端执行 sudo docker。
- 图形会话为 X11，DISPLAY=:1，Xauthority 位于 /run/user/1000/gdm/Xauthority。
- GPU 为 Intel i915 与 NVIDIA；/dev/dri 可用于容器图形加速。
- 主机没有 /opt/ros/noetic，也没有 roscore 命令。
- ROS2 Jazzy 提供的 Gazebo Harmonic 8.11.0 只有加载 Jazzy 环境后才进入 PATH。

## 历史容器

- 镜像：osrf/ros:noetic-desktop-full-focal。
- 镜像 ID：f138c82f326f。
- 镜像摘要：sha256:f138c82f326f179e8510eb03f40ee7ccaf16538e5577ab8550af74c4e12616de。
- 容器：ros1_noetic。
- 创建时间：2026-09-11 03:26（Asia/Shanghai 附近）。
- 网络：host。
- 挂载：/home/xanter/ros1_ws -> /root/ros1_ws，可写。
- 容器进程用户：root。
- 系统：Ubuntu 20.04.6 LTS。
- ROS：Noetic；roslaunch 1.17.4。
- Python：3.8.10。
- Gazebo Classic：11.15.1。
- MAVROS 与 gazebo_ros 均能被 rospack 找到。

## 工作区

/home/xanter/ros1_ws 当前没有用户 ROS 包，只有 catkin 的 CMakeLists.txt。历史 build 和 devel 由 root 创建，内部固化 /root/ros1_ws 路径，不能作为新的宿主机用户工作区直接复用。

新基础设施应以 UID/GID 1000 运行容器。历史 build/devel 在处理前保留，新的实验优先使用自包含 catkin overlay。

## ArduPilot 与 Gazebo 插件

- ArduPilot：/home/xanter/Projects/ardupilot。
- 2026-09-13 检查提交：b832113b10aa4e639889668a66d4b5c611944152。
- 工作树当时干净，分支 master。
- sim_vehicle.py：/home/xanter/Projects/ardupilot/Tools/autotest/sim_vehicle.py。
- 现代插件工作树：/home/xanter/ros2_ws/src/ardupilot_gazebo，分支 ros2，用于 Gazebo Harmonic。
- 本地 legacy gazebo11 分支固定提交：e72ecf5916a04736a2bc920e3b577ad784c221b5。
- legacy Iris 插件端口：FDM 输入 9002，输出 9003。
- legacy 分支较旧，必须以实际 SITL/FDM 测试判断当前兼容性。


## 派生镜像

- 名称：xanter/ros1-noetic-gazebo11:20260913。
- 2026-09-13 最终镜像 ID：sha256:f7b402cf06d7a39a86603eeb2cf0285017d6d4694d154de0900557fc4ee0a03b。
- 大小字段：1205142539 bytes。
- 显式增加 MAVROS 1.20.1、MAVROS extras、GeographicLib 工具与数据。
- legacy ArduPilotPlugin 动态库依赖检查通过。
- 容器运行用户 UID/GID 均为 1000。


## 当前目录布局

- `/home/xanter/ros1simulation/infra/noetic_gazebo11`：共享镜像定义、Compose、entrypoint、固定 legacy vendor、可见构建与 doctor 脚本。
- `/home/xanter/ros1simulation/ros1sim_260913_iris_setuptest`：ArduCopter/Iris 最小联合链基线。
- `/home/xanter/ros1simulation/ros1sim_260913_zephyr_circuit`：ArduPlane/Zephyr 人工解锁与 AUTO 全任务。
- `/home/xanter/ros1simulation/AGENTS.md`：目录命名、运行隔离、权限和验证规则。

三个层次分别代表公共基础设施、低风险连通性实验和完整飞行任务。具体复现见 `env_notes/examples/ROS1/`。
