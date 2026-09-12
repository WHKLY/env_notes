# 故障排查

## 找不到包

~~~bash
source /opt/ros/jazzy/setup.bash
source /home/xanter/ros2_ws/install/setup.bash
ros2 pkg prefix ardupilot_gz_bringup
~~~

仍失败则按构建文档重建。~/.bashrc 当前不会自动加载工作区。

## SDF URI 导致模型创建失败

表现为 sdformat_urdf 解析 package:// include 失败、robot_state_publisher 退出、ros_gz_sim create 等不到 robot_description。本机 robot.launch.py 已有 URI 修复。更新后复现时先检查：

~~~bash
git -C /home/xanter/Projects/ardupilot_gz diff --   ardupilot_gz_bringup/launch/robots/robot.launch.py
~~~

## SITL 等待 JSON

先看 Gazebo 世界、模型创建、ArduPilotPlugin，再核对端口和残留：

~~~bash
ss -lunp | grep -E ':9002|:2019|:14550|:14551'
ps -eo pid,comm,args | grep -E 'gz sim|arducopter|arduplane' | grep -v grep
~~~

## DDS 没初始化

正常日志包含 XRCE session 和 DDS: Initialization passed：

~~~bash
ros2 pkg prefix micro_ros_agent
ss -lunp | grep ':2019'
ros2 node list
ros2 topic list | grep '^/ap/'
~~~

重建后失败时检查 microxrceddsgen 是否仍为官方 v4.7.1，PATH 是否命中其他同名程序。

## 插件缺动态库

~~~bash
source /opt/ros/jazzy/setup.bash
source /home/xanter/ros2_ws/install/setup.bash
ldd /home/xanter/ros2_ws/install/ardupilot_gz_gazebo/lib/libArduPilotPlugin.so |   grep 'not found'
~~~

无输出表示依赖完整。

## 家目录出现 mav.tlog 或 terrain

它们是 MAVProxy/SITL 在当前工作目录写入的运行数据，不是 /usr/bin 污染。从 ~/ 启动就会落在家目录。请用运行手册的独立 run_dir。

## 其他已知现象

- ros_gz 覆盖提示：src 有源码，但 bridge/sim 当前来自 /opt；用 packages-select 构建即可。
- RViz 缺 x8_tga.jpg：目前只影响 Alti 一项纹理，不阻止物理、DDS 和起飞。
- QML/EGL 警告：场景仍刷新且话题正常时通常不影响仿真；黑屏再检查 GPU/渲染。
- Ctrl+C 后残留：在 GNOME Terminal 中对前台 launch 按 Ctrl+C，再核对进程。

## Skywalker X8 的 Jazzy 兼容项

Skywalker X8 原始 SDF 在 model 根下有 pose，Jazzy sdformat_urdf 不支持这种结构。供 robot_state_publisher 和 ros_gz_sim create 使用的临时 SDF 应移除根级 pose，出生位置改由 launch 的 x/y/z/R/P/Y 指定。

原模型没有 OdometryPublisher，因此仅配置 ros_gz_bridge 不会产生 /odometry。实验 launch 会向临时 SDF 注入 gz-sim-odometry-publisher-system。Collada 网格同时引用 x8_tga.jpg 和 x8.tga，实验包需同时安装两种纹理。
