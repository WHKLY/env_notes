# ROS2 Skywalker X8：构建与代码改动

## 1. 实验边界

实验位于 `/home/xanter/ros2simulation/ros2sim_260912_setuptest`。它假定公共 `/home/xanter/ros2_ws` 已经构建，只在 `.colcon/` 下构建私有 `setuptest_bringup` 包，不修改公共 install。

~~~text
src/setuptest_bringup/
├── package.xml
├── setup.py
├── launch/skywalker_setuptest.launch.py
├── config/
└── models/skywalker_x8/
config/                 受维护的参数、bridge 和记录话题
missions/circuit.json   相对航线
scripts/                构建、检查、启动、解锁、控制和归档
runs/                   每次运行证据
~~~

## 2. 公共 ardupilot_gz 的 Jazzy URI 补丁

文件位置：`/home/xanter/Projects/ardupilot_gz/ardupilot_gz_bringup/launch/robots/robot.launch.py`。当前提交为 `8df4dc1726e37504e6fc8b952d02e554cfa3176f`，工作树有这一处未提交修改。

补丁新增 `import re` 和：

~~~python
def resolve_package_uris(sdf: str) -> str:
    def replace_uri(match: re.Match) -> str:
        package_name = match.group(1)
        return f"file://{get_package_share_directory(package_name)}"
    return re.sub(r"package://([^/\s<]+)", replace_uri, sdf)
~~~

`launch_state_pub_with_bridge()` 读取 SDF 后调用 `robot_desc = resolve_package_uris(robot_desc)`。原因是 Jazzy 的 `sdformat_urdf` 在这一链路中不能可靠解析 `package://`；先用 ament package index 转为绝对 `file://`，robot_state_publisher 才能生成描述并让 create 节点继续。

更新、reset 或 checkout `ardupilot_gz` 前必须保存这处 diff。

## 3. Skywalker 临时 SDF 转换

核心文件：`src/setuptest_bringup/launch/skywalker_setuptest.launch.py`。

`create_robot()` 从 `ardupilot_sitl_models` 读取公共 Skywalker SDF，但不覆盖原件。运行时依次做四项派生：

1. 把 `model://skywalker_x8` 改为 package URI；
2. 把四个 DAE 网格 URI 指向实验自包含的 meshes；
3. 删除根级 `<pose>0 0 0.246 ...</pose>`，因为 Jazzy `sdformat_urdf` 不接受 model-level pose；
4. 在 `</model>` 前注入 `gz-sim-odometry-publisher-system`，生成 bridge 所需的 Gazebo odometry 话题。

派生内容写到临时 `.sdf`，出生位姿由 launch 参数 `x/y/z/R/P/Y` 指定，其中 `z=0.25`、航向 90°。这种做法保留公共模型原件，也把兼容处理限制在实验内。

## 4. SITL 参数与 DDS

launch 组合四份默认参数：

~~~text
skywalker_x8.param
dds_udp.parm
dds_use_ns.parm
setup_plane.param
~~~

前一份提供机型参数，中间两份启用 UDP 2019 的 DDS client 和 `/ap` 命名空间，最后一份是实验叠加：落地 5 秒后解除武装、50 m 起飞高度、10 m 开始平飞、35 m 航点半径。

`robot.launch.py` 的参数选择 `command=arduplane`、`model=json`、`synthetic_clock=True`、`use_dds_agent=True` 和 `use_instance_dir=True`。因此一个 launch 同时带起 Gazebo、SITL、Micro XRCE-DDS Agent、bridge 和 RViz。

## 5. bridge 与资源安装

`config/skywalker_bridge.yaml` 桥接 `/clock`、joint_states、odometry、Gazebo TF 和 IMU。bridge 只能转发现有 Gazebo Transport 话题，所以前述 OdometryPublisher 注入不可省略。 `src/setuptest_bringup/config/setup_plane.param` 和 `skywalker_bridge.yaml` 是指向实验根 `config/` 的符号链接；launch 从包 share 读取，维护者只修改根目录原件。

Skywalker DAE 同时引用 `x8_tga.jpg` 和 `x8.tga`。`setup.py` 把 meshes 与两种纹理都安装进 `setuptest_bringup` share，避免 Gazebo 与 RViz 使用不同解析上下文时丢资源。

## 6. 航线与控制责任

`missions/circuit.json` 定义与 ROS1 Zephyr 相同的相对几何。`scripts/mission_controller.py` 通过 UDP 14551 获取 HOME 和航向，生成 HOME、TAKEOFF、三个巡航 WP、进近 WP、LAND 共七项任务，写 `MISSION_READY` 后等待实际 armed，再切 AUTO。

`scripts/arm.sh` 才是人工授权点。操作者输入 `ARM` 后，它检查 `/ap/arm_motors` 服务，依次调用：

~~~bash
ros2 service call /ap/prearm_check std_srvs/srv/Trigger
ros2 service call /ap/arm_motors ardupilot_msgs/srv/ArmMotors "{arm: true}"
~~~

控制器不调用解锁服务。DDS 完成人工解锁，MAVLink 控制器负责上传任务、切 AUTO 与记录飞行结果。

## 7. 构建和静态检查

~~~bash
cd /home/xanter/ros2simulation/ros2sim_260912_setuptest
./scripts/build.sh
./scripts/check.sh
~~~

`build.sh` 按 Jazzy、公共工作区的顺序加载环境，并把私有包构建到 `.colcon/{build,install,log}`。`check.sh` 验证 Bash、Python、JSON、包前缀、launch 参数以及公共 SITL/模型文件。

手工核对三层环境：

~~~bash
source /opt/ros/jazzy/setup.bash
source /home/xanter/ros2_ws/install/setup.bash
source .colcon/install/setup.bash
ros2 pkg prefix ardupilot_gz_bringup
ros2 pkg prefix ros_gz_bridge
ros2 pkg prefix setuptest_bringup
~~~

预期前缀分别位于 `/home/xanter/ros2_ws/install`、`/opt/ros/jazzy` 和本实验 `.colcon/install`。
