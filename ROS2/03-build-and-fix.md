# 构建、修复与验证

## 源码布局

- /home/xanter/ros2_ws/src/ardupilot：ArduPilot 和 SITL。
- /home/xanter/Projects/ardupilot_gz：ardupilot_gz 实体仓库。
- /home/xanter/ros2_ws/src/ardupilot_gz：指向上项的符号链接。
- /home/xanter/ros2_ws/src/ardupilot_gazebo：Gazebo 插件和资源。
- /home/xanter/ros2_ws/src/ardupilot_sitl_models：SITL 模型。
- /home/xanter/ros2_ws/src/microxrcedds_gen：官方 v4.7.1。
- /home/xanter/ros2_ws/src/ros_gz：源码快照；当前运行桥来自 /opt。

## 推荐构建

~~~bash
cd /home/xanter/ros2_ws
source /opt/ros/jazzy/setup.bash
test -f install/setup.bash && source install/setup.bash
colcon build --symlink-install --packages-select   ardupilot_msgs ardupilot_sitl ardupilot_gazebo   ardupilot_gz_application ardupilot_gz_description   ardupilot_gz_gazebo ardupilot_sitl_models   micro_ros_agent sdformat_urdf ardupilot_gz_bringup
source install/setup.bash
~~~

只修改 launch 时：

~~~bash
cd /home/xanter/ros2_ws
source /opt/ros/jazzy/setup.bash
source install/setup.bash
colcon build --symlink-install --packages-select ardupilot_gz_bringup
source install/setup.bash
~~~

当前保留的 build 目录均成功完成。以前中断的 ros_gz_bridge 本地缓存已清理，系统桥仍正常工作。

## DDS 生成器

ArduPilot DDS 代码生成必须使用兼容版本。本机为官方 v4.7.1、Java 17，并使用仓库自带 ./gradlew。不要使用来源不明的同名生成器。

~~~bash
command -v microxrceddsgen
git -C /home/xanter/ros2_ws/src/microxrcedds_gen describe --tags --always
java -version
~~~

## 本机补丁

文件：/home/xanter/Projects/ardupilot_gz/ardupilot_gz_bringup/launch/robots/robot.launch.py

Jazzy 的 sdformat_urdf 解析含 package:// URI 的 SDF include 时，sdf::findFile() 回调曾返回空路径，导致 robot_state_publisher 退出和实体创建失败。

本机修复在读取 SDF 后将 package://包名/... 转成相应 ament share 目录的 file:// 绝对 URI，再传给 robot_state_publisher。代码新增 re 和 resolve_package_uris()。这是 ardupilot_gz 当前唯一已知未提交修改，更新或重置前必须保存：

~~~bash
git -C /home/xanter/Projects/ardupilot_gz status --short
git -C /home/xanter/Projects/ardupilot_gz diff --   ardupilot_gz_bringup/launch/robots/robot.launch.py
~~~

## 已验证

- Gazebo 创建实体，robot_state_publisher 和 ArduPilotPlugin 正常。
- SITL 收到 JSON FDM。
- XRCE-DDS 会话建立，DDS: Initialization passed。
- /clock、/imu、/odometry 有消息。
- /ap/status、/ap/pose/filtered 有消息。
- Alti 在 GUIDED 解锁、起飞，达到约 20.07 m 相对高度。
- ardupilot_sitl 相关测试曾通过 24 项。

ardupilot_gz_bringup 运行通过。flake8 和 pep257 对多个上游 launch 文件仍报既有风格问题，这不是联合仿真功能失败。
