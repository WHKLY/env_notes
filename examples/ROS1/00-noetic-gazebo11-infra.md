# ROS1 Noetic 与 Gazebo Classic 基础设施

## 1. 为什么单独建立 infra

Ubuntu 24.04 主机没有 ROS Noetic，直接安装会破坏发行版与 ABI 边界。基础设施把 Ubuntu 20.04、Noetic、Gazebo Classic 11、gazebo_ros、MAVROS 和 legacy ArduPilotPlugin 固定在镜像中；主机只运行当前 ArduPilot SITL。目录为：

~~~text
/home/xanter/ros1simulation/infra/noetic_gazebo11/
├── Dockerfile
├── compose.yaml
├── ros_entrypoint.sh
├── VENDOR_COMMIT
├── vendor/ardupilot_gazebo_classic/
├── scripts/
└── runtime/
~~~

`runtime/` 保存构建和 doctor 的本机证据，属于可再生成内容。

## 2. 固定输入与可复现边界

`Dockerfile` 的基础镜像使用完整摘要：

~~~dockerfile
FROM osrf/ros:noetic-desktop-full-focal@sha256:f138c82f326f179e8510eb03f40ee7ccaf16538e5577ab8550af74c4e12616de
~~~

`VENDOR_COMMIT` 固定 legacy 插件为 `ardupilot_gazebo` 的 `origin/gazebo11` 提交 `e72ecf5916a04736a2bc920e3b577ad784c221b5`。源码导出到 `vendor/ardupilot_gazebo_classic`，镜像内构建成：

~~~text
/opt/ardupilot_gazebo_classic/build/libArduPilotPlugin.so
~~~

这两个固定点分别约束操作系统/ROS 软件包和 Gazebo Classic 插件源码。实验 manifest 还会记录主机 ArduPilot 提交，从而把运行结果对应到完整输入。

## 3. Dockerfile 增加的内容

基础镜像不能依赖历史容器的可写层。`Dockerfile` 明确安装：

~~~dockerfile
geographiclib-tools
ros-noetic-mavros
ros-noetic-mavros-extras
~~~

随后运行 MAVROS 自带的 `install_geographiclib_datasets.sh`。镜像构建阶段还执行 CMake、编译 legacy 插件，并检查 `libArduPilotPlugin.so` 确实存在。最后建立 UID/GID 1000 的 `xanter` 用户，使挂载目录中的日志不会变成 root 所有。

## 4. Compose 的职责

`compose.yaml` 定义镜像 `xanter/ros1-noetic-gazebo11:20260913`，并做四件事：

- `network_mode: host`：让容器 Gazebo/MAVROS 与主机 SITL 直接使用 UDP 9002/9003/14550；
- 挂载 `/home/xanter/ros1simulation`、只读 ArduPilot 源码、X11 socket 和只读 Xauthority；
- 设置 `LIBGL_ALWAYS_SOFTWARE=1`，绕开 Focal Mesa 不识别本机 Raptor Lake iGPU 的问题；
- 设置本地 `ROS1_GAZEBO_MODEL_DATABASE_URI`，阻止 Gazebo 等待已经退役的在线模型库。

X11 使用只读授权文件，不执行 `xhost +`。当前用户不加入 docker 组，sudo 只在可见 GNOME Terminal 中输入。

## 5. entrypoint 为什么要再次设置模型库 URI

source `/opt/ros/noetic/setup.bash` 时，Gazebo 包可能恢复默认在线数据库地址。`ros_entrypoint.sh` 在 source 之后重新赋值：

~~~bash
export GAZEBO_MODEL_DATABASE_URI="${ROS1_GAZEBO_MODEL_DATABASE_URI-}"
~~~

它只在历史 `/home/xanter/ros1_ws/devel` 的路径仍然有效时加载该 overlay。旧的 `/root/ros1_ws` 固化产物不会被盲目复用。

## 6. 构建与检查

Docker 默认 inactive/disabled。先在可见终端按需启动：

~~~bash
gnome-terminal --wait --title="启动 Docker" -- bash -lc \
  'sudo systemctl start docker'
~~~

然后：

~~~bash
cd /home/xanter/ros1simulation/infra/noetic_gazebo11
./scripts/build-visible.sh
./scripts/doctor-visible.sh
~~~

两个脚本都在 GNOME Terminal 中运行。成功时 `runtime/build.status` 和 `runtime/doctor.status` 均为 `0`。doctor 应确认 Ubuntu 20.04.6、Noetic、Gazebo 11.15.1、gazebo_ros 2.9.3、MAVROS 1.20.1、插件依赖完整，以及容器 UID/GID 为 1000。 当前成功构建的镜像 ID 为 `sha256:f7b402cf06d7a39a86603eeb2cf0285017d6d4694d154de0900557fc4ee0a03b`。

需要交互检查时运行：

~~~bash
./scripts/shell-visible.sh
~~~

## 7. 服务收尾

实验结束且没有容器后恢复默认状态：

~~~bash
gnome-terminal --wait --title="停止 Docker" -- bash -lc \
  'sudo systemctl stop docker.service docker.socket'
~~~

应得到 `systemctl is-active docker.service` 和 `docker.socket` 均为 `inactive`。Zephyr 的启动脚本会记录 `DOCKER_STARTED_BY_RUN`，在确由该 run 启动 Docker 时自动恢复；Iris 基线需要操作者显式完成此步骤。
