# ROS1 联合仿真踩坑记录

> 最后更新：2026-09-13（Asia/Shanghai）

## 历史容器状态不等于基础镜像内容

历史 ros1_noetic 容器能找到 MAVROS，但同一基础镜像新建的容器不能。原因是 MAVROS 后来装在旧容器的可写层中。

可复现镜像必须在 Dockerfile 中明确安装 ros-noetic-mavros、ros-noetic-mavros-extras、geographiclib-tools 和 GeographicLib 数据集，不能把旧容器当成构建定义。

## Docker Compose 最初没有安装

系统装有 Docker Engine 29.1.3，但没有 Compose 子命令。2026-09-13 安装 docker-compose-v2 2.40.3。脚本使用 docker compose，不使用旧的 docker-compose 命令。

## ros1_ws 的历史 build/devel 不可直接复用

旧容器以 root 身份把 /root/ros1_ws 映射到宿主 ~/ros1_ws，生成文件属于 root，且 devel 中固化了 /root/ros1_ws。新的派生镜像使用 UID/GID 1000；实验优先采用自包含配置，不继续增量使用旧 build/devel。

## 空的 GAZEBO_MODEL_DATABASE_URI 会被恢复

gazebo_ros 的 gzserver 包装脚本会保存数据库 URI，若值为空，则在重新 source Gazebo setup 后恢复 http://models.gazebosim.org。该地址会让 world 加载阻塞。

本机使用非空本地 URI：

~~~text
file:///opt/ardupilot_gazebo_classic/models
~~~

因此 Gazebo 会快速提示没有 database.config，然后只使用本地模型。这条提示符合预期。

## legacy world 依赖远程 sun

旧 iris world 使用 model://sun。为避免模型库访问，实验 world 改为内联 directional light。实验所需资源必须本地自包含。

## legacy Iris 引用了不存在的云台

gazebo11 分支的 iris_with_ardupilot/model.sdf 引用 gimbal_small_2d，但该分支没有提供对应模型，导致 Unable to find uri 和 LoadJoint Failed。

实验使用本地 Iris 派生模型，移除缺失云台 include 与 iris_gimbal_mount joint，保留 Iris 本体、动力学插件和 ArduPilotPlugin。

## Focal Mesa 不识别 Raptor Lake Intel GPU

容器内旧 Mesa 对主机 Intel 0xa788 PCI ID 报错，无法创建 iris DRI screen。本机基线设置 LIBGL_ALWAYS_SOFTWARE=1，使用 llvmpipe 运行 Gazebo Classic GUI。该场景足以做基础联调；复杂视觉仿真再考虑 NVIDIA Container Toolkit。

## sim_vehicle.py 会优先选择 Xterm

ArduPilot 的 run_in_terminal_window.sh 在 DISPLAY 存在且能找到 xterm 时，会先于 gnome-terminal 选择 Xterm。

启动脚本显式设置 SITL_RITW_TERMINAL 为 gnome-terminal，最终测试未出现 Xterm。不要依赖终端自动探测顺序。

## MAVROS connected 早于 local_position 就绪

MAVROS 收到 heartbeat 后很快显示 connected: True，但 /mavros/local_position/pose 要等 EKF 初始化并设置原点。一次性采集会得到空文件。

自动采集应持续重试 model_states、IMU 和 local_position，直到三者都有消息或达到明确超时。

## 结束外层终端不保证所有容器子进程退出

早期停止测试中 rosmaster 和 gzclient 曾短暂残留。当前 stop.sh 记录容器名与终端进程组，由 Docker 向容器发送停止信号；容器内先给 rosbag SIGINT，再停止 roslaunch，随后停止主机 SITL，最后检查残留并自动 finalize。

## 已验证结果

最终回归：

~~~text
run_20260913T011015+0800
~~~

确认：

- Gazebo /clock 有数据；
- model_states 包含 ground_plane 和 iris_demo；
- MAVROS 1.20.1 connected；
- IMU、global position 和 local position 有数据；
- ArduCopter V4.8.0-dev 显示 ArduPilot Ready；
- 无 Xterm；
- stop.sh 返回 0，写 DONE；
- rosbag v2 正常收尾，时长 154 秒、55.3 MB、310405 条消息；
- 停止后无 Gazebo、ROS、MAVROS、SITL 残留；
- Docker 服务恢复为 inactive。

该验证没有解锁或起飞。

## Zephyr 上游模型不能原样用于 Gazebo Classic

本机可用的 Zephyr 模型来自 `ardupilot_gazebo` 的 Gazebo Sim 历史分支，包含 `gz-sim-lift-drag-system` 等系统插件。ROS1 Noetic 链运行 Gazebo Classic 11，不能直接加载这些插件。

`ros1sim_260913_zephyr_circuit` 固定模型来源提交 `8f3970a3d2bf0a5c4f283b969b39853e36716774`，保留几何、惯性和气动参数，把七个气动系统转换为 Classic `libLiftDragPlugin.so`，移除 Gazebo Sim 的 JointStatePublisher/ApplyJointForce，再接入 legacy `libArduPilotPlugin.so`。

## 视觉网格不适合直接作为地面碰撞体

Zephyr 第一次动态运行 `run_20260913T015218+0800` 未能解锁。飞机在跑道上持续高速抖动，MAVLink `VIBRATION` 的 z 轴达到约 58 m/s²，IMU clipping 累积到数千，ArduPlane 反复报告 `PreArm: Accels inconsistent`。

原因是细致的机腹三角网格和向下延伸的桨叶碰撞体直接参与地面接触。修复方法是把机身碰撞体替换为简单 box，增加机鼻、左翼和右翼三点低摩擦球形滑橇，移除不参与飞行动力学的桨叶碰撞体，并把模型初始高度调整到 0.155 m。修复后静止角速度约 0.0004 rad/s，振动约 0.003 m/s²，IMU clipping 为 0。

第二次动态运行 `run_20260913T015818+0800` 完成人工解锁、AUTO 起飞、四边航线、自动进近、接地和自动解除武装。结果为 `landed_and_auto_disarmed`，用时 290.19 秒，最高相对高度 60.35 m；闭合 rosbag 为 192,488,211 字节，`stop.status` 为 0，停止后 Docker service/socket 均为 inactive。

## 用户启动脚本不能假定安装了 ripgrep

Codex 工具环境能调用 `rg`，但用户的普通交互 shell 中不一定安装或暴露该命令。固定翼实验最初因此在 `check.sh` 的最后两项检查退出，尚未启动 Docker。用户入口脚本已改用系统自带 `grep`；静态验证还要使用最小系统 PATH 重跑，避免把代理工具环境误当成本机用户环境。
